|         |                       |
| ------- | --------------------- |
| Authors | Ilia Kebets, Simon Goumaz |
| Reviewers |                     |
| Date    | 2021-12-21            |
| Version | 2                     |

# ==WIP== OAuth support + aggregated queries


## Motivation

1. **Offer app developers a service for aggregated queries**. This is the main driver, and the reason why we design both features together: on one hand customers won't care about OAuth on its own, on the other we don't want to implement new features on top of our aging auth flow.
2. **Adopt a/the standard app auth flow**, which simplifies understanding for customers, and lets them easily use existing libs etc.
3. **Improve security** by controlling which app can perform an access request for which permissions. Currently, what we have is on one side the [`TRUSTED_APPS` setting](https://github.com/pryv/config-template-pryv.io/blob/master/pryv.io/cluster/config-leader/conf/template-platform.yml#L65) in the platform config (which can only control apps that are allowed to login as the user), and on the other full reliance on the user's decision in the app auth flow (where any app can ask for any set of permissions).


## Implementation scope

- App registration mechanism (at least basic)
- OAuth 2.0 ("authentication code" flow) support as alternative to current auth flow, which will be eventually deprecated
- Aggregation service for apps to list their authorized users and query those users' data as a whole (distributing individual queries to each account internally)
  - Out of scope: maintaining aggregated data


## Proposition

**OAuth support, back-end**
Implement new service running on the "reg-master" machine (`service-oauth`==?==)
- Stores app registration data centrally alongside other register data
- Serves API for app registration and OAuth flow
- Load expected to be minimal

**OAuth support, front-end**
Implement ==new web app/rewrite `app-web-auth3`== on "web" machine as default front-end
- ==TBD: SPA for registering, managing & deleting apps (or do it manually, or via a basic CLI)==
- SPA for OAuth flow
- Serves existing "access" pages for [legacy app auth](https://api.pryv.com/reference/#auth-request) support

Legacy auth flow will be deprecated but supported by default, with a config switch to disable it.

**Aggregated queries**
Implement new service running on each "core" machine ==? formally speaking this should almost be a new role, no?==
- Load potentially heavy, must be distributed
- App must authenticate to perform aggregated queries ==how to do auth if app secret lives on reg? replicate secret across cores is messy… have each app speak to the one core hosting the app's associated user account?==


### 1. Register app

- Done manually via e.g. admin panel? ==Q: how about supporting customers who want to offer a registration interface to devs?==
- "Read-only permission access" ==? found in GDoc==
- Storage: event of type `oauth/declaration` (or rather `oauth/app-registration`?) containing data ==? how can that work across the platform?==:
  - `client_id` (generated)
  - `client secret` (generated, maybe only for relevant app types – see below)
  - Name
  - Icon
  - Description
  - Homepage URL
  - Privacy policy URL
  - Redirect URL(s) (must be HTTPS)
  - App type (e.g. to limit usage of `client_secret` to "web server" or "service" apps)
  - Default/maximum scope?
  - "horizontal token" ==? found in original design doc==

See [some examples](https://www.oauth.com/oauth2-servers/client-registration/registering-new-application/).


### 2. Authorize app

Support "authorization code" flow ==(earlier docs mentioned "implicit" but it's deprecated now)==

==Q: what base path for endpoints? `/oauth/`? (Pryv legacy auth: `/access/`)==

For ref.: [Pryv legacy implementation](https://api.pryv.com/reference/#authenticate-your-app)


#### `GET /authorize`

Initiates an app's request to access a user's data.

Params:
- `client_id`: app id registered beforehand in platform config
- `response_type`: must be `code`
- `state`: encrypted string for possible client-side data to be persisted during the auth flow; if it contains a random component, serves as protection against CSRF
- `redirect_uri` (optional): where to redirect to after auth; URL must be preregistered with the app as well
- `scope` (optional): requested permissions; may be required to be within a scope preregistered with the app (TODO: check whether any particular format, e.g. space-separated)
- PKCE, relying on client-generated code verifier (cryptographically random string using the characters `A-Z`, `a-z`, `0-9`, and the punctuation characters `-._~`, 43 to 128 characters long):
  - `code_challenge_method`: e.g. `S256` for SHA256
  - `code_challenge`: Base64-URL-encoded string of the SHA256 hash of the code verifier

==TBD: authentication & authorization GUI here, see [recommandations there](https://www.oauth.com/oauth2-servers/authorization/the-authorization-interface/)==
- app-web-auth ==? GDoc==

Returns HTTP `302` redirect to the given or preregistered URL with query params: 
- `code`: authorization code, used to obtain access token; the info that needs to be associated with it is:
  - `client_id`, `redirect_uri`
  - User identification
  - Expiration date: within 10 minutes (OAuth spec), more likely 30–60 secs
  - Unique id
  - PKCE `code_challenge_method` & `code_challenge`
- `state`: as described in params above

If an error occurs with authorization, the redirection will include param(s):
- `error`: code identifying the error
  - `access_denied`: The user did not authorize the app
  - `invalid_request`: The request is missing a required parameter, includes an invalid parameter value, or is otherwise malformed. 
  - `unauthorized_client`: The client is not authorized to request an authorization code using this method.
  - `invalid_scope`: The requested scope is invalid, unknown, or malformed.
  - `server_error`: The authorization server encountered an unexpected condition which prevented it from fulfilling the request.
  - `temporarily_unavailable`: The authorization server is currently unable to handle the request due to a temporary overloading or maintenance of the server.
- `error_description` (optional): human-readable description (for the developer); valid chars = ASCII minus `"` and `\`
- `error_uri` (optional): points to a human-readable web page with info about the error (for the developer)

If the client ID or redirect URL is invalid, no redirection is made and a HTTP `40X` ==TBD== response is given along with appropriate error text.


#### `POST /token`

Lets an app obtain an access token in exchange for the code obtained via `/authorize`.

Params:
- `grant_type`: must be `authorization_code` or `refresh_token` (if used)
- `code`: the code obtained via `/authorize`
- `redirect_uri` (optional): same as for `/authorize`
- `client_id`
- `client_secret` ==TBD: should not be used for "public" (browser/mobile) apps==
- PKCE: `code_verifier`: to verify that the client has the secret that was used to generate the `code_challenge` in `/authorize`

Returns HTTP `200` with body JSON params:
- `access_token`: access token to make API requests on core
- `refresh_token` (optional): token to obtain new access token if current one expires (best practice: renew it every time to make it possible to detect a stolen one and block all associated access tokens)
- `token_type`: `Bearer`
- `expires`: TTL in seconds, e.g. 3600

Errors ==TBD==


### 3. Make authenticated requests to API

For ref.: [Pryv legacy implementation](https://api.pryv.com/reference/#authorization)

Requests must carry valid access token in the `Authorization` header, prefixed by `Bearer`; e.g. `Authorization: Bearer RsT5OjbzRn430zqMLgV3Ia"`

If the token is invalid, returns HTTP `401` with `WWW-Authenticate: Bearer error="invalid_token"` header and body JSON params as for others API errors, e.g.:
- `error`: `invalid_token`
- …

==Q: must the error code be `invalid_token` in OAuth? Pryv uses `invalid-access-token`==

==TODO: complete from [GDoc](https://docs.google.com/document/d/1FjTBG-GanCyNIypEDZjA7cc_Ysl0WEV0yKJJTA1gohk/edit#)==


## References

- [OAuth.com - OAuth 2.0 Simplified](https://www.oauth.com/)
- [oauth2-server — package documentation](https://oauth2-server.readthedocs.io/en/latest/index.html)
