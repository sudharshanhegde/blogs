# Authentication in SSH

When we type `ssh user@server`, the server does not directly ask for a password. SSH has a full negotiation protocol for authentication. The server advertises which methods it will accept, and the client tries them in preferred order. Some methods require multiple rounds; some can partially succeed requiring a second method after the first. The client can try and fail multiple times before getting rejected.

## What Happens on the Wire

After TCP connection, key exchange, and encryption are established, the client enters the authentication phase. The session is encrypted but not yet authenticated -> the server does not know who the client is yet.

### `ssh_userauth_none()`

The client sends an authentication request with method `"none"`. The server responds with `SSH_MSG_USERAUTH_FAILURE` and includes a list of authentication methods it accepts.

```
Client -> SSH_MSG_USERAUTH_REQUEST  method="none"
Server -> SSH_MSG_USERAUTH_FAILURE  methods="publickey,keyboard-interactive,password"
```

We get a comma-separated list; libssh reads it and returns a bitmask:

```c
int methods = ssh_userauth_list(session, NULL);

if (methods & SSH_AUTH_METHOD_PUBLICKEY)   { /* server accepts pubkey */ }
if (methods & SSH_AUTH_METHOD_INTERACTIVE) { /* keyboard-interactive */ }
if (methods & SSH_AUTH_METHOD_PASSWORD)    { /* password */ }
if (methods & SSH_AUTH_METHOD_GSSAPI_MIC)  { /* Kerberos/GSSAPI */ }
if (methods & SSH_AUTH_METHOD_HOSTBASED)   { /* host-based auth */ }
```

Some old servers skip this and return `SSH_AUTH_SUCCESS` for `none`, meaning no authentication is required at all. So `ssh_userauth_none()` is called first and its return value is checked before trying anything else.

### Trying Methods in Order

The client tries each method the server advertised, stopping when one succeeds. Order matters -> public key is preferred because it is non-interactive; password is last because it requires user input.

```
Client -> SSH_MSG_USERAUTH_REQUEST  method="publickey"  key=<public key>
Server -> SSH_MSG_USERAUTH_SUCCESS
```

```
Client -> SSH_MSG_USERAUTH_REQUEST  method="publickey"  key=<public key>
Server -> SSH_MSG_USERAUTH_FAILURE  methods="password"  partial=false
```

`partial=false` means this was not even a partial success -> the method was rejected outright.

### Partial Authentication (Multi-Factor)

Some servers require two methods. After the first succeeds, the server sends:

```
Server -> SSH_MSG_USERAUTH_FAILURE  methods="keyboard-interactive"  partial=true
```

`partial=true` means "that method worked, and I need one more." The client must now also succeed at `keyboard-interactive`.

This is how SSH implements two-factor auth:

- `publickey` + OTP
- `GSSAPI` + password

libssh returns `SSH_AUTH_PARTIAL` from any auth call when this happens:

```c
int rc = ssh_userauth_publickey_auto(session, NULL, NULL);

if (rc == SSH_AUTH_PARTIAL) {
    /* first factor done, server wants more */
    rc = authenticate_kbdint(session, NULL);
}
```

If we only check for `SSH_AUTH_SUCCESS` and return on anything else, multi-factor will silently fail.

---

## Authentication Methods

### Public Key

The client says "I have this public key, will you accept it?" The server checks its `authorized_keys` file. If the public key is there, the server says "prove you have the corresponding private key." The client signs a challenge with the private key and sends the signature. The server verifies.

```
Client -> USERAUTH_REQUEST  method="publickey"  has_sig=false  pubkey=<key>
Server -> SSH_MSG_USERAUTH_PK_OK       ← "yes, I'd accept that key"

Client -> USERAUTH_REQUEST  method="publickey"  has_sig=true  pubkey=<key>  sig=<signature>
Server -> SSH_MSG_USERAUTH_SUCCESS
```

```c
/* try all keys in ~/.ssh/ automatically */
int rc = ssh_userauth_publickey_auto(session, NULL, NULL);

/* or try a specific key */
ssh_key key;
ssh_pki_import_privkey_file("~/.ssh/id_ed25519", passphrase, NULL, NULL, &key);
int rc = ssh_userauth_publickey(session, NULL, key);
ssh_key_free(key);
```

`ssh_userauth_publickey_auto()` tries every key in `~/.ssh/` -> `id_ed25519`, `id_rsa`, `id_ecdsa`, etc. -> against the server. This is what OpenSSH does by default.

**PKCS#11:** A PKCS#11 provider is a shared library (`.so` file) that lets us use hardware tokens (smart cards, YubiKeys) as if they were key files. The private key never leaves the hardware. We load the provider, ask it for a list of keys, and use those keys for authentication. libssh has partial support for this via `SSH_OPTIONS_PROXYCOMMAND` workarounds; adding proper PKCS#11 support via the `-I` flag is one of the things this project will be implementing.

### Keyboard-Interactive

This is **not** the same as password auth. It is a flexible challenge-response protocol. The server sends a list of prompts (any number of them). The client displays each prompt, collects an answer, and sends all answers back. The server decides if they are correct.

Why not just use password auth for passwords? Because keyboard-interactive can be used for OTPs, Duo push, TOTP codes, or any arbitrary challenge. The server controls the prompts entirely.

```
Server -> SSH_MSG_USERAUTH_INFO_REQUEST
    name="Password"
    instruction=""
    prompts=["Password: " (echo=false), "Verification code: " (echo=true)]

Client -> SSH_MSG_USERAUTH_INFO_RESPONSE
    answers=["hunter2", "123456"]

Server -> SSH_MSG_USERAUTH_SUCCESS
```

The libssh API:

```c
int rc = ssh_userauth_kbdint(session, NULL, NULL);  /* starts the exchange */

while (rc == SSH_AUTH_INFO) {
    int n = ssh_userauth_kbdint_getnprompts(session);
    for (int i = 0; i < n; i++) {
        char echo;
        const char *prompt = ssh_userauth_kbdint_getprompt(session, i, &echo);
        char answer[128];
        /* display prompt, collect answer from user */
        ssh_userauth_kbdint_setanswer(session, i, answer);
    }
    rc = ssh_userauth_kbdint(session, NULL, NULL);  /* send answers, get next round */
}
/* rc is now SSH_AUTH_SUCCESS, SSH_AUTH_DENIED, or SSH_AUTH_PARTIAL */
```

### Password

The client sends the password in plaintext -> but inside the encrypted SSH channel, so it is safe on the wire.

```c
int rc = ssh_userauth_password(session, NULL, "my_password");
```

This is tried last because:

1. If the server has your public key, no password is needed.
2. Keyboard-interactive is more flexible and should be preferred.
3. Password is the fallback.

### GSSAPI (Kerberos)

No passwords, no keys -> a Kerberos ticket proves identity. Transparent to the user if properly configured.

```c
int rc = ssh_userauth_gssapi(session);
```

---

## `PreferredAuthentications`

OpenSSH's default order is:

```
gssapi-with-mic,hostbased,publickey,keyboard-interactive,password
```

The `PreferredAuthentications` config option lets users change this order. This is currently `SSH_OPTIONS_UNSUPPORTED` in libssh -> it is silently ignored and the order is always hardcoded.

**Plan for implementation:** Parse the comma-separated string, build an ordered list, and iterate through it during authentication.

```c
char *methods[] = {"publickey", "keyboard-interactive", "password", NULL};

for (int i = 0; methods[i] != NULL; i++) {
    if (strcmp(methods[i], "publickey") == 0) {
        rc = ssh_userauth_publickey_auto(session, NULL, NULL);
    } else if (strcmp(methods[i], "keyboard-interactive") == 0) {
        rc = authenticate_kbdint(session);
    } else if (strcmp(methods[i], "password") == 0) {
        rc = ssh_userauth_password(session, NULL, password);
    }
    if (rc == SSH_AUTH_SUCCESS) break;
    if (rc == SSH_AUTH_PARTIAL) continue;  /* need another method */
}
```

### `BatchMode` -> Disabling Interactive Prompts

When running SSH in a script:

```sh
ssh server "backup.sh" < /dev/null
```

There is no user at a terminal. If the server asks for a password, nobody can type it -> the script hangs forever.

`BatchMode yes` tells SSH: if any authentication method would require user input, skip it entirely. Only try methods that can succeed without a prompt -> `publickey` and `GSSAPI`. If all non-interactive methods fail, return an error immediately instead of prompting.

**Plan for implementation:**

```c
bool batch_mode = session->opts.batch_mode;

for (int i = 0; methods[i] != NULL; i++) {
    if (strcmp(methods[i], "password") == 0 && batch_mode) {
        continue;  /* skip password in batch mode */
    }
    if (strcmp(methods[i], "keyboard-interactive") == 0 && batch_mode) {
        continue;  /* skip kbdint in batch mode */
    }
    /* ... try the method ... */
}
```

This is also `SSH_OPTIONS_UNSUPPORTED` -> this project will implement it.

### `NumberOfPasswordPrompts`

By default, OpenSSH asks for a password up to 3 times before giving up. `NumberOfPasswordPrompts 0` disables password auth entirely. `NumberOfPasswordPrompts 1` asks once and gives up immediately on failure.

**Plan for implementation:**

```c
int max_prompts = session->opts.num_password_prompts;  /* default: 3 */
int attempts = 0;

while (attempts < max_prompts) {
    char password[128];
    ssh_getpass("Password: ", password, sizeof(password), 0, 0);
    rc = ssh_userauth_password(session, NULL, password);
    memset(password, 0, sizeof(password));  /* zero it immediately */

    if (rc == SSH_AUTH_SUCCESS) break;
    if (rc == SSH_AUTH_DENIED) {
        attempts++;
        continue;
    }
    break;  /* error or partial */
}
```

This is also `SSH_OPTIONS_UNSUPPORTED` -> this project will implement it.

---

## Putting It All Together

```c
int authenticate(ssh_session session, struct cli_opts *opts) {
    int rc;

    /* Step 1: none auth -> get list of accepted methods */
    rc = ssh_userauth_none(session, NULL);
    if (rc == SSH_AUTH_SUCCESS) return SSH_AUTH_SUCCESS;  /* server accepts anyone */
    if (rc == SSH_AUTH_ERROR)   return SSH_AUTH_ERROR;

    int server_methods = ssh_userauth_list(session, NULL);

    /* Step 2: get preferred order from config, default if not set */
    char **order = parse_preferred_auths(opts->preferred_auths);

    /* Step 3: iterate */
    for (int i = 0; order[i] != NULL; i++) {
        /* skip if server doesn't support this method */
        if (!server_supports(server_methods, order[i])) continue;

        /* skip interactive methods in batch mode */
        if (opts->batch_mode && is_interactive(order[i])) continue;

        rc = try_method(session, opts, order[i]);

        if (rc == SSH_AUTH_SUCCESS) return SSH_AUTH_SUCCESS;
        if (rc == SSH_AUTH_PARTIAL) continue;  /* need second factor */
        /* SSH_AUTH_DENIED: try next method */
    }

    fprintf(stderr, "ssh: all authentication methods failed\n");
    return SSH_AUTH_ERROR;
}
```

This is what `example/authentication.c` approximates but does not fully implement -> it has no `PreferredAuthentications` ordering, no `BatchMode`, no `NumberOfPasswordPrompts`, and does not handle `SSH_AUTH_PARTIAL` properly in all paths. This project will address all of these gaps.
