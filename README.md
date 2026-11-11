# authentik-privacyidea

privacyIDEA as a second factor in [authentik](https://goauthentik.io):

* **Push**: privacyIDEA sends a notification to the *privacyIDEA Authenticator* app. The user approves it on the phone, then selects Continue.
* **TOTP / HOTP**: the user types a code from any authenticator app that is enrolled in privacyIDEA.
* **Email / SMS**: challenge-response tokens are triggered too, and their codes go in the same field.

Tokens are enrolled and managed in privacyIDEA. authentik only asks privacyIDEA to verify them.

## How it works

authentik has no loader for third-party code, so this plugin is an **authentik blueprint**: stages, prompts and Python expression policies that authentik imports and runs natively. You don't need a fork or a custom image.

```
Identification ─▶ Password ─▶ privacyIDEA stage ─▶ User login
                                   │
       render ── static prompt expression ── POST /validate/triggerchallenge  (push sent)
       submit ── validation policy
                   ├─ code entered  → POST /validate/check  user+pass=code
                   └─ field empty   → GET  /validate/polltransaction (waits up to PUSH_WAIT s)
                                      → POST /validate/check  user+transaction_id+pass=""
```

* The push is sent **once per browser session** when the stage first renders. Re-renders, such as after a wrong code, reuse it. The pending `transaction_id` is kept server-side in authentik's cache, keyed by session and user. It never goes to the browser, so a challenge approved in one session can't complete a login in another.
* **It fails closed.** If privacyIDEA can't be reached, isn't configured, or returns an error, the stage refuses to continue. The page itself still loads and asks for a code.
* Every result writes an authentik event: `custom_privacyidea_mfa` (method `push`/`otp`/`passthru`, plus success) and `custom_privacyidea_challenge_triggered`.

## Install

1. **Configure the authentik server container** with these environment variables. Copy `docker-compose.override.yml`, or add them to your Helm `server.env`. Only the server needs them.

   | Variable | Default | Meaning |
   |---|---|---|
   | `PRIVACYIDEA_URL` | (required) | Base URL, e.g. `https://pi.example.com` |
   | `PRIVACYIDEA_REALM` | (privacyIDEA default realm) | Realm the users live in |
   | `PRIVACYIDEA_VERIFY_TLS` | `true` | `true`, `false`, or a path to a CA bundle inside the container |
   | `PRIVACYIDEA_SERVICE_USER` / `_PASS` / `_REALM` | (empty) | privacyIDEA admin used to trigger challenges (recommended) |
   | `PRIVACYIDEA_TRIGGER_MODE` | `auto` | `service` = triggerchallenge with the service admin; `emptypass` = `/validate/check` with an empty password; `auto` = service if credentials are set, otherwise emptypass; `none` = never send a push (codes only) |
   | `PRIVACYIDEA_USER_ATTRIBUTE` | `username` | Field sent as the privacyIDEA `user`: `username`, `email`, or `attributes.some.path` |
   | `PRIVACYIDEA_PUSH_WAIT` | `15` | Seconds the submit request waits for push approval |
   | `PRIVACYIDEA_POLL_INTERVAL` | `1` | Seconds between polls while waiting |
   | `PRIVACYIDEA_CHALLENGE_TTL` | `110` | Seconds a pending push is reused before a new one is sent. Keep it below privacyIDEA's challenge validity (default 120). |
   | `PRIVACYIDEA_ALLOW_NO_TOKEN` | `false` | `true` = skip the stage for users with no active privacyIDEA token. Needs the service admin. |
   | `PRIVACYIDEA_TIMEOUT` | `10` | HTTP timeout per request, in seconds |

2. **Import the blueprint** `blueprints/privacyidea-mfa.yaml`. Either mount it at `/blueprints/custom/` (as in the override file), or go to *Admin → Customization → Blueprints → Create* and paste it. This creates:
   * the flow `privacyidea-authentication-flow` (identification → password → privacyIDEA → login), reusing the default stages
   * the prompt stage `privacyidea-mfa`, prompts `privacyidea-status`/`-otp`/`-resend`, and policies `privacyidea-validate` and `privacyidea-skip-if-no-token`

3. **Use it.** Set the flow as the brand's authentication flow (*System → Brands → Default flows*), or on individual providers.

   To add it to an **existing** flow instead, bind the stage `privacyidea-mfa` after the password stage with *Evaluate when stage is run* on and *Evaluate when flow is planned* off. Then bind `privacyidea-skip-if-no-token` to that binding with *Failure result = pass*. Remove the built-in MFA validation stage from that flow so users aren't asked twice.

   To change the flow's slug, set the blueprint context: `flow_slug`, `flow_name`.

## privacyIDEA configuration

* **Push token:** configure the push provider: Firebase, or poll-only via the `push_wait`/poll settings on privacyIDEA ≥ 3.6. Then enroll tokens with the privacyIDEA Authenticator app.
* **OTP PIN:** this plugin sends only the code, not PIN+code. Add an *authentication* policy for the realm with `otppin = none`. If users really have token PINs, they have to type PIN+code in the code field. `userstore` won't work, because authentik doesn't pass the password on.
* **Service admin**, used in `service`/`auto` mode. Create an admin (or a restricted admin realm) whose *admin* policy grants only `triggerchallenge` and, if you use `ALLOW_NO_TOKEN`, `tokenlist`.
* **Email/SMS tokens** in `emptypass` mode: add `challenge_response_allow` for those token types. In `service` mode, triggerchallenge covers them.
* **No-token users:** for full fail-closed behaviour, leave `passOnNoToken` / `passthru` off. If you turn them on, users with no token get through in `emptypass` mode, because privacyIDEA itself accepts them.
* Brute force: privacyIDEA's token fail counter (`maxfail`) locks tokens after repeated wrong codes. Leave it on.

## Develop / test

```bash
python3 build.py              # src/*.py  →  blueprints/privacyidea-mfa.yaml
pip install pytest flask requests pyyaml
cd tests && python3 -m pytest -q
```

Edit the Python in `src/`, never the generated YAML. `src/_lib.py` is prepended to every expression, because authentik expressions can't import each other.

The tests run the expressions from the built blueprint through the same `def handler(...)` wrapper authentik uses, against a mock privacyIDEA. They cover push approve, timeout, decline and resend; TOTP right and wrong; email challenge; all trigger modes; JWT refresh; attribute mapping; cross-session replay; privacyIDEA down; and the skip policy.

## Notes and limitations

* Waiting for the push holds one authentik web worker thread for up to `PUSH_WAIT` seconds per login. The default of 15 suits most deployments. Lower it if you have many concurrent logins.
* The page doesn't continue by itself after approval. The user selects Continue, either before or after approving. Fixing that needs a custom stage with its own front-end component, which means maintaining a fork of authentik.
* Codes are only checked against privacyIDEA. They're stored in the flow's `prompt_data` until the flow ends, so don't put a *User Write* stage after this stage in the same flow.
* Written against authentik 2026.x (prompt-stage expression API) and the privacyIDEA 3.x `/validate` API.

MIT licensed.
