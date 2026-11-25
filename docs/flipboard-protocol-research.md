# Flipboard client protocol: research notes

Research date: 2026-11-25. Goal: decide how a Windows desktop Flipboard client
(FBWin), built on egui and storytold's craft-libs, should talk to Flipboard.

## Summary

| Option | Status | Auth | Fit for FBWin |
|---|---|---|---|
| Flipboard app's own API (`fbprod.flipboard.com/v1`) | Private and undocumented. Only evidence is an old third-party gist | Unknown (login call) | **Avoid.** Unsupported, could break any time, likely against ToS |
| **Surf API** (`api.surf.social/v1`) | **Official and public**, run by Flipboard, with MIT-licensed SDKs in Python, TS, Go and Java | API key or OAuth 2.0 + PKCE | **Recommended** primary protocol |
| ActivityPub | Flipboard federates its accounts and magazines | Not needed to read | Optional, read-only |
| RSS (`flipboard.com/@user.rss`) | Public | None | Simple read-only fallback |

Flipboard publishes no public API for the classic Flipboard app. Its supported developer
surface is **Surf**, Flipboard's social browser and platform. Surf aggregates Bluesky,
Mastodon, RSS, podcasts, YouTube and Leaflet. The rest of this document describes the Surf
API, worked out from the official SDK source in
[Flipboard/surf-sdks](https://github.com/Flipboard/surf-sdks), file `go/surf.go`,
client version 1.3.0. I did not read the hosted API reference at
<https://developers.surf.social/devportal/v1/api-docs>, so check the details there
before relying on them.

## Surf API: wire protocol

- **Transport:** HTTPS REST, with JSON request and response bodies.
- **Base URL:** `https://api.surf.social/v1`. Developer-portal endpoints (diagnostics,
  RTB) are on `https://surf.social/devportal/v1`.
- **Request headers:** `Accept: application/json`, plus `Content-Type: application/json`
  when there is a body, and a `User-Agent` that names the client.
- **Size:** the README advertises 94 endpoints in 15 categories.

### Authentication

| Credential | Format | Sent as | Use |
|---|---|---|---|
| App API key | `surf_sk_live_…` / `surf_sk_test_…` | `X-API-Key: <key>` | Acting as the developer account (bots, server to server) |
| OAuth access token | `surf_at_…` (a JWT) | `Authorization: Bearer <token>` | Acting on behalf of a signed-in user. **This is what FBWin needs** |

Send exactly one of the two. If you put a JWT in `X-API-Key`, the server returns 401.

**OAuth 2.0 Authorization Code + PKCE (S256)**

1. Open the user's browser at
   `GET https://api.surf.social/v1/oauth/authorize?client_id=…&redirect_uri=…&scope=read%20write&response_type=code&state=…&code_challenge=…&code_challenge_method=S256`
2. Exchange the code: `POST /v1/oauth/token` with a JSON body
   `{grant_type:"authorization_code", client_id, code, redirect_uri, code_verifier}`.
   The response is `{access_token, token_type, expires_in, refresh_token, scope}`.
3. Refresh: `POST /v1/oauth/token` with `{grant_type:"refresh_token", client_id, refresh_token}`.
   **Refresh tokens rotate.** Each refresh invalidates the old token, so persist the new
   one right away.
4. Revoke: `POST /v1/oauth/revoke` with `{client_id, token}`.

The token requests are JSON bodies, not form-encoded, and the SDK sends no client secret.
That suits a public desktop client.

To use it, you must
[apply for developer access](https://developers.surf.social/devportal/v1/developer/apply)
and register an application to get a `client_id`.

### Errors, rate limits and retries

- **Error body:** `{"error": "<code>", "error_description": "<text>"}`.
  - 401: bad credentials
  - 403: missing scope
  - 404: not found
  - 429: rate limited
- **Rate-limit headers** on each response: `X-RateLimit-Limit`, `X-RateLimit-Remaining`,
  `X-RateLimit-Reset`.
- **What the SDK retries** (3 retries by default):
  - 429: waits for `Retry-After` seconds, capped at 60.
  - 5xx and network errors: exponential backoff of 1, 2, 4… seconds, capped at 60.
  - Multipart uploads are never retried.
- **Status codes worth noting:** `206` on `GET /media/attachments/{id}` means the
  media is still processing, so poll again.

### Pagination

- Pagination uses a cursor.
- The response holds an array under a key that depends on the endpoint (`posts` for
  `/feed/posts`), plus `cursor` or `next_cursor`.
- Pass it back as `?cursor=…`. A missing, empty or `null` cursor means there are no more pages.

### Identifiers

- Feeds are addressed by a **surf id**, for example `surf/topic/technology`.
- Networks are service ids such as `surf/service/bluesky`; bare names like `bluesky` also work.
- Write calls take an optional `?service=bluesky|mastodon`. Without it, the server uses the
  user's default linked account, preferring Bluesky.
- Long-form documents and publications use AT URIs.

### Endpoints most relevant to a reader client

| Area | Endpoints (all under `/v1`) |
|---|---|
| Feeds | `GET /feed?surf_id=`, `GET /feed/posts?surf_id=&limit=&cursor=&sort=&services=&since=`, `GET /feed/following`, `GET /feed/speeddial` |
| Posts | `GET /statuses/{id}`, `GET /statuses/{id}/context`, `POST /statuses`, `DELETE /statuses/{id}` |
| Post actions | `POST /statuses/{id}/favourite`, `/unfavourite`, `/reblog`, `/unreblog`, `/bookmark`, `/unbookmark`, `/pin`, `/unpin` |
| Search | `GET /search/posts`, `/search/maestra/feeds`, `/search/bluesky/searchActors`, `/search/rss/search`, `/search/discover`, `/search/publications` |
| Account | `GET/PATCH /account`, `GET /account/lookup`, `/account/links`, `POST /accounts/{id}/follow`, `/unfollow`, `/account/connected-apps` |
| Custom feeds | `GET/POST /custom`, `GET/PUT/DELETE /custom/{id}`, `/clone`, `/publish`, `/unpublish`, `/custom/{id}/operators` |
| Notifications | `GET /notifications?limit=&cursor=`, `POST /notifications/read` |
| Preferences | `GET/PATCH /preferences/account` |
| Content | `GET /content/resolve`, `/content/extract`, `/content/language`, `/content/topics`, `/content/enrich` |
| Images | `GET /image/info`, `/image/resize` (bytes), `/image/colors`, `/image/moderate` |
| Media | `POST /media/upload` and `POST /media/attachments` (multipart, field `file`), `GET /media/attachments/{id}` |
| Long-form | `GET /documents/{at-uri}`, `/publications/{at-uri}`, `/publications/{at-uri}/documents` |
| Audio / podcasts | `/audio/radio/*`, `/audio/briefing/*`, `/audio/transcript`, `/audio/episodes/search`, `/audio/popular/*`, `POST /audio/tts` |
| Playback | `POST /playback/batch`, `GET /playback/recent`, `GET /playback/positions` |
| AI | `GET /ai/ask`, `/ai/feed-summary`, `/ai/thread-summary`, `POST /ai/feed-builder`, `/ai/fact-check` |
| Sonars (saved watches) | `GET/POST /sonars`, `GET/PATCH/DELETE /sonars/{id}`, `/sonars/{id}/matches`, `POST /sonars/preview` |

The API has no push channel. Notifications and sonar matches are polled.

## Other routes

- **ActivityPub:** Flipboard accounts and magazines federate, so they can be read as
  ordinary ActivityPub actors. Flipboard's
  [activitypub repo](https://github.com/Flipboard/activitypub) contains only a README.
- **RSS:** `https://flipboard.com/@<user>.rss` needs no auth. It suits a no-login mode.

## Notes for FBWin's design

- **No Rust SDK exists.** Write a thin `reqwest` (or `ureq`) client that mirrors
  `go/surf.go`. It is small and regular: one request helper, a retry policy, a
  paginator, and the same error type.
- **OAuth redirect on desktop:** use a loopback listener such as
  `http://127.0.0.1:<port>/callback`, or a custom `fbwin://` URI scheme. I haven't checked
  which redirect URIs Surf accepts; confirm that when you register the app.
- **Token storage:** keep the refresh token in Windows Credential Manager (for example
  with the `keyring` crate), not in `config.toml`. Store the rotated token after every
  refresh, and serialize refreshes behind a mutex, as the Go SDK does.
- **Keep the UI from freezing:** run network I/O off the egui thread (an async runtime
  or worker threads plus channels) and call `ctx.request_repaint()` when results arrive.

## storytold/craft-libs

- **What it is:** a Rust workspace (edition 2024), Apache-2.0, at
  [storytold/craft-libs](https://github.com/storytold/craft-libs).
- **Toolchain:** requires **Rust 1.95+**; installed here: 1.100.0. Its UI integrations
  target **egui 0.36**, so FBWin must use the same egui version.
- **Release:** the only release, `v0.1.0`, covers five crates: `craft-ui`,
  `craft-settings`, `craft-quit`, `craft-atomic`, `craft-print`. The other ~200 crates
  exist only on `main`, so pinning them means pinning a `rev`.
- **Distribution:** not on crates.io; use git dependencies. Pin every craft-libs
  dependency to the same tag or rev.
- **Crates relevant to FBWin:**
  - `craft-ui`: egui icon buttons, tabs, docking, themes
  - `craft-settings`: `config.toml` storage with platform folders, atomic saves and migrations
  - `craft-quit`: egui close handling
  - `craft-dialogs`: non-blocking native dialogs
  - `craft-keymap`: key bindings
  - `craft-i18n`: translations
  - `craft-html` and `craft-markdown`: rendering post and article bodies
  - `craft-atomic`: crash-safe file writes
- **No HTTP or OAuth client:** craft-libs doesn't include one, so FBWin brings its own.

## Sources

- [Flipboard/surf-sdks](https://github.com/Flipboard/surf-sdks): `README.md`, `go/surf.go`, `go/oauth.go`
- [Surf developer portal](https://developers.surf.social)
- [Flipboard begins to federate](https://about.flipboard.com/inside-flipboard/flipboard-begins-to-federate/)
- [RSS feeds on Flipboard](https://about.flipboard.com/inside-flipboard/rss-feed-on-flipboard/)
- [Gist using fbprod.flipboard.com/v1](https://gist.github.com/raphaelschaad/5c377caa55a752109960) (unofficial)
- [storytold/craft-libs](https://github.com/storytold/craft-libs)
