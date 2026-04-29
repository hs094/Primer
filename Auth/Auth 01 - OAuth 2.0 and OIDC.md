# 01 — OAuth 2.0 and OIDC

🔑 OAuth 2.0 = delegated **authorization** (give app X permission to act on resource Y). OIDC = **authentication** layer on top — adds an `id_token` so the app actually learns who the user is.

Sources: https://datatracker.ietf.org/doc/html/rfc6749 · https://openid.net/specs/openid-connect-core-1_0.html

## Two Different Jobs
| Spec | Answers |
|---|---|
| **OAuth 2.0** | "Can this app call this API on behalf of the user?" |
| **OIDC** | "Who is the user signing in?" |

OAuth alone has no standard "user info" — apps abused `access_token` introspection to fake login. OIDC fixes that.

## Tokens
| Token | Audience | Purpose |
|---|---|---|
| `access_token` | **Resource server** (API) | Permits API calls; opaque or JWT (RFC 9068, `typ: at+jwt`) |
| `id_token` | **Client** (your app) | Proves user identity; always JWT, validate `iss`/`aud`/`nonce`/`exp` |
| `refresh_token` | **Auth server** | Mint new access tokens; long-lived, rotate on use |

⚠️ Never send the `id_token` to your API as a credential — wrong audience.

## Flows (Pick By Client Type)
| Flow | Use For |
|---|---|
| **Authorization Code + PKCE** | SPAs, mobile, native, server web — the **default** for any user-facing app |
| **Client Credentials** | M2M / service-to-service — no user, just `client_id` + `client_secret` |
| **Device Code** | TVs, CLIs, IoT — input-constrained devices |
| ~~Implicit~~ | Deprecated (token in URL fragment, leaks) |
| ~~ROPC~~ | Deprecated (app sees the password) |

## Auth Code + PKCE Sketch
1. Client → `/authorize?response_type=code&code_challenge=...&...`
2. User logs in at the auth server, consents
3. Auth server redirects with `?code=...`
4. Client → `/token` with `code` + `code_verifier` → gets tokens

💡 In OIDC, add `scope=openid` (plus `profile`/`email`) to also get an `id_token`.

## Tags
[[Auth]] [[OAuth]] [[OIDC]] [[PKCE]] [[JWT]] [[Access Token]] [[ID Token]]
