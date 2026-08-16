# Example: Auth API

Description of an authentication service (login / logout / refresh / session) using DocScript.
This example applies most of the syntax rules defined in [`SPEC.md`](../SPEC.md) in a real-world scenario: object chains, parameters, output, error/side effect, control structures, nesting, and async functions.

```docscript
c(AuthService)(secret_key:str, token_ttl:int)-creates a new auth service with a signing secret and a token lifetime

c(AuthService)f(login)(username:str, password:str)->Token-authenticates a user and returns an access token
c(AuthService)f(login)!-raises InvalidCredentials if the username or password is wrong

c(AuthService)f(login)v(user)->User-the authenticated user object once login succeeds

if use c(AuthService)f(login) ->
    if use f(hash_password) ->
        if use f(compare_hash) ->
            return v(token)

async c(AuthService)f(logout)(token:str)->bool-invalidates an active token
c(AuthService)f(logout)!-returns false without effect if the token was already invalid

c(AuthService)f(refresh)(refresh_token:str)->Token-exchanges a valid refresh token for a new access token
c(AuthService)f(refresh)!-raises TokenExpired if the refresh_token has expired
c(AuthService)f(refresh)!-shuts down the current session and requires re-login

c(Session)(user:User, token:Token)-represents an active logged-in session
c(Session)v(is_active)->bool-whether this session is still valid
c(Session)f(extend)(minutes:int)->none-extends the session expiry time

for s in v(active_sessions) -> if use c(Session)v(is_active) -> c(Session)f(extend)(minutes:15)

while use f(server_running) -> c(AuthService)f(cleanup_expired_tokens)-removes expired tokens every cycle
```

## Notes

- `c(AuthService)f(login)v(user)` is an example of the composition/output relationship in a chain (section 3.1 in SPEC): after calling `login`, the `user` value becomes available.
- The nested `if` block near the beginning of the example uses the indent-based form because it has more than one level of nesting (section 7.2 in SPEC).
- Two separate `!-` scenarios are written for `f(refresh)`, following the "one `!-` per statement" limitation (section 8.1 in SPEC).
