# OAuth login for functions

The [of-watchdog](/architecture/watchdog/) can add browser login to a function using an OAuth 2.0 or OpenID Connect (OIDC) provider. It runs the Authorization Code flow with PKCE on your function's behalf, keeps the resulting session in a signed cookie, and validates that cookie before forwarding each request to your function.

Your function does not have to implement the login flow itself: it can just read the verified session cookie and make its own authorization decisions based on the contents.

## How it works

On every request the watchdog checks for a valid session cookie:

* No cookie - the request is forwarded to the function unchanged, so public pages work without signing in.
* Valid cookie - the request is forwarded with the cookie intact, so the function can decode it to read the user's ID and access tokens.
* Invalid or expired cookie - the request is rejected with HTTP 401.

The function can start the login flow by sending the visitor to the watchdog's `GET /auth/login` endpoint, for example from a "Sign in" link. The watchdog then runs the authorization-code exchange with the configured provider.

On success it issues a JWT with the provider's tokens embedded and sets it as a session cookie (default name `of_session`), then sends the visitor back to the function.

The watchdog serves the following routes beneath the function's `oauth_base_url`:

| Method and path suffix | Purpose |
| ---------------------- | ------- |
| `GET /auth/login` | Start the login flow and redirect the browser to the provider |
| `GET /auth/callback` | Handle the provider's redirect and set the session cookie |
| `POST /auth/logout` | Clear the session cookie |

To sign in, direct the browser to `{oauth_base_url}/auth/login`. To sign out, submit a `POST` request to `{oauth_base_url}/auth/logout`, using a form or JavaScript request.

## Enabling OAuth

To enable OAuth for a function, register an OAuth application with your provider. The watchdog is configured through environment variables on the function.

When registering the client, set `{oauth_base_url}/auth/callback` as the redirect (callback) URI.

Point the watchdog at your provider in one of two ways:

* **OIDC (recommended)** - set `oauth_issuer_url` to the provider's issuer URL. The watchdog discovers the authorization, token and keys endpoints and validates ID tokens automatically. Works with common OIDC providers such as Keycloak, Okta, Google and Microsoft Entra ID.
* **Plain OAuth** - set `oauth_authorization_endpoint` and `oauth_token_endpoint` to the provider's authorization and token exchange URLs. Use this with providers that do not support OIDC discovery, such as a GitHub OAuth App.

Each function also needs a random 32-byte key to sign the session cookie. Generate one with the faas-cli and store it as an OpenFaaS secret, then reference it via `oauth_signing_key`:

```bash
faas-cli secret generate | faas-cli secret create profile-signing-key \
  --from-file=/dev/stdin
```

### Core configuration

| Option | Usage |
| ------ | ----- |
| `oauth_enabled` | Set to `true` to enable OAuth/OIDC login and session validation. Default: `false`. |
| `oauth_base_url` | The function's public URL, including any path, e.g. `https://gateway.example.com/function/profile`. Used to build the callback URL, the cookie path and the session issuer/audience. |
| `oauth_client_id` | The client ID registered with your provider. |
| `oauth_client_secret` | Name of an [OpenFaaS secret](/reference/secrets/) containing the client secret. Omit for public clients without a secret. PKCE is used with or without a client secret. |
| `oauth_issuer_url` | URL of the OIDC provider's issuer. The watchdog discovers the authorization, token and keys endpoints and validates ID tokens automatically. Use for OIDC providers. |
| `oauth_authorization_endpoint` | The provider's authorization URL. Use together with `oauth_token_endpoint` for providers without OIDC discovery. |
| `oauth_token_endpoint` | The provider's token exchange URL. Use together with `oauth_authorization_endpoint` for providers without OIDC discovery. |
| `oauth_signing_key` | Name of an [OpenFaaS secret](/reference/secrets/) containing a base64-encoded random 32-byte key used to sign session cookies. |

### Advanced configuration

| Option | Usage |
| ------ | ----- |
| `oauth_scopes` | Space- or comma-separated scopes. Default: `openid`. OIDC always includes `openid`. |
| `oauth_cookie_name` | Name of the session cookie your function reads to identify the user. Default: `of_session`. Must be a valid cookie name and differ from the login cookie name. |
| `oauth_login_cookie_name` | Internal state cookie for the login flow. Default: `of_login`. Must be a valid cookie name and differ from the session cookie name. |
| `oauth_login_redirect` | Destination after successful login. Defaults to `oauth_base_url`. |
| `oauth_logout_redirect` | Destination after logout. Defaults to `<oauth_base_url>/auth/login`. |
| `oauth_error_redirect` | Optional destination for login failures. When unset, the watchdog returns an HTTP error. |
| `oauth_session_default_ttl` | Session lifetime when the provider supplies no expiry. Default: `1h`. Accepts a Golang duration. |
| `oauth_session_ttl` | Optional override for the session JWT and cookie lifetime, even beyond provider token expiry. Does not refresh or extend the embedded token's validity. When unset, the ID token expiry, OAuth `expires_in`, or the default lifetime is used. Accepts a Golang duration. |
| `oauth_allow_http` | Allow HTTP provider endpoints, discovery and redirects for development. Default: `false` (HTTPS required). |
| `oauth_token_auth_method` | Client-secret authentication method: `client_secret_basic` (default) or `client_secret_post`. Unused without a client secret. |

### Reading the session in your function

After a successful login the watchdog forwards the session cookie with every request. Read the cookie (default name `of_session`) from the request in your handler.

The cookie is a signed JWT that embeds the original tokens issued by the provider in a `value` claim:

```json
{
  "value": {
    "id_token": "...",
    "access_token": "..."
  }
}
```

The original provider claims, or the issued token, can be read from the `value` and used to make authorization decisions in your handler, for example to allow or deny access.

The watchdog verifies the session JWT's signature, issuer, audience and expiry before the request reaches your function, so the function does not have to validate it.

## Examples

### Walkthrough

Walk through enabling OAuth login for a simple function.

1. **Scaffold the function**

    Create the function from the `golang-middleware` template:

    ```bash
    faas-cli new profile --lang golang-middleware
    ```

    This creates a `stack.yaml` and a `profile/` folder with a `handler.go` you can edit.

2. **Register an OAuth application with your provider**

    Create a new client for the function and set `https://gateway.example.com/function/profile/auth/callback` as the redirect (callback) URI. Note the client ID, and the client secret when your client has one.

3. **Create the secrets**

    Each function needs its own signing key. Generate one and store it as an OpenFaaS secret:

    ```bash
    faas-cli secret generate | faas-cli secret create profile-signing-key \
      --from-file=/dev/stdin
    ```

    When your client has a secret, store it as well, for example from a file containing the client secret:

    ```bash
    faas-cli secret create profile-client-secret \
      --from-file=client-secret
    ```

    Use `faas-cli secret update` instead if a secret already exists.

4. **Read the session in your handler**

    After a successful login the watchdog forwards the `of_session` cookie with every request. A minimal handler reads the cookie, redirects the visitor to `{oauth_base_url}/auth/login` when there is no session, and uses the tokens stored in the session:

    ```go
    package function

    import (
        "encoding/base64"
        "encoding/json"
        "net/http"
        "os"
        "strings"
    )

    // sessionCookie matches the value claim of the session JWT the watchdog
    // forwards with every request. Its signature is verified by the watchdog.
    type sessionCookie struct {
        Value struct {
            IDToken     string `json:"id_token"`
            AccessToken string `json:"access_token"`
        } `json:"value"`
    }

    // Handle reads the verified session and returns the user's ID token claims.
    func Handle(w http.ResponseWriter, r *http.Request) {
        cookie, err := r.Cookie("of_session")
        if err != nil {
            // No session - send the visitor through the login flow.
            base := strings.TrimRight(os.Getenv("oauth_base_url"), "/")
            http.Redirect(w, r, base+"/auth/login", http.StatusFound)
            return
        }

        var session sessionCookie
        if !decodePayload(cookie.Value, &session) {
            http.Error(w, "invalid session", http.StatusUnauthorized)
            return
        }

        // Use the session, e.g. return the user's claims from the ID token.
        raw := session.Value.IDToken
        if raw == "" {
            raw = session.Value.AccessToken
        }
        claims := map[string]any{}
        if !decodePayload(raw, &claims) {
            http.Error(w, "unable to read session", http.StatusUnauthorized)
            return
        }

        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(claims)
    }

    // decodePayload decodes the base64 payload segment of a JWT into target.
    // The watchdog already verified the signature before forwarding the cookie.
    func decodePayload(token string, target any) bool {
        parts := strings.Split(token, ".")
        if len(parts) != 3 {
            return false
        }
        data, err := base64.RawURLEncoding.DecodeString(parts[1])
        if err != nil {
            return false
        }
        return json.Unmarshal(data, target) == nil
    }
    ```

5. **Configure the function**

    Add the OAuth environment variables and bind the secrets in `stack.yaml`:

    ```yaml
    version: 1.0
    provider:
      name: openfaas
      gateway: https://gateway.example.com
    functions:
      profile:
        lang: golang-middleware
        handler: ./profile
        image: ttl.sh/openfaas-examples/profile:latest
        environment:
          oauth_enabled: 'true'
          oauth_base_url: https://gateway.example.com/function/profile
          oauth_client_id: profile
          oauth_client_secret: profile-client-secret
          oauth_issuer_url: https://signet.example.com
          oauth_signing_key: profile-signing-key
        secrets:
          - profile-client-secret
          - profile-signing-key
    ```

    Replace the example values with your own, e.g. your gateway URL, the issuer URL of your provider and your client ID.

6. **Build and deploy**

    ```bash
    faas-cli up --tag=sha
    ```

    Visit the function's public URL in a browser and sign in. Your handler is called with the verified session cookie on every request.

### Full examples

Full, ready-to-run examples are available in the [of-watchdog-oauth-examples](https://github.com/welteki/of-watchdog-oauth-examples) repository. It contains three functions that demonstrate different OAuth login patterns:

| Function | Auth flow | Pattern |
| -------- | --------- | ------- |
| `oauth-demo` | OIDC | Server-side rendered Go page that decodes the session cookie and shows the user's ID token claims |
| `github-oauth-demo` | GitHub OAuth App | Server-side rendered Go page that calls the GitHub API with the session's access token |
| `react-oauth-demo` | OIDC | React single-page app (SPA) that reads the user's profile from a JSON API |

## Related

* [Authentication for functions](/reference/authentication/)
* [Built-in function authentication with IAM](/openfaas-pro/iam/function-authentication/)
