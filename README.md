# supersports-ci

GitHub Actions CI pipeline for the Supersports React Native app.

App source lives in a private Gitea repo and is cloned into `real-code/` at
build time. Build logic (prebuild, gym, gradle) is driven by Fastlane — the
`Fastfile` lives in the app repo, not here.

---

## Entry Points

Three `workflow_dispatch` workflows appear in the **Actions** tab:

| Workflow | File | Inputs | Produces |
|---|---|---|---|
| **Internal Testing** | `internal-testing.yml` | `branch`, `environment` (staging\|live) | iOS ad-hoc IPA + Android APK in parallel |
| **TestFlight** | `testflight.yml` | `branch`, `environment` (staging\|live) | iOS TestFlight upload |
| **AAB** | `aab.yml` | `branch` | Android App Bundle (always live) |

Default branch for all workflows: `develop`.

**Concurrency:** different branches always build in parallel. Same branch + same
environment queues (Internal Testing, AAB) or cancels the older run (TestFlight).

---

## Architecture

```
internal-testing.yml  ──┬──▶  .github/workflows/_ios.yml      (workflow_call)
testflight.yml       ───┘         └── composite actions (see below)
                        ┌──▶  .github/workflows/_android.yml   (workflow_call)
aab.yml              ───┘         └── composite actions (see below)

Composite actions (.github/actions/):
  checkout-source      — SSH trust + shallow clone from Gitea
  write-app-secrets    — .env file (Firebase config is committed in the app repo)
  setup-js             — Node 24, node_modules cache, yarn install
  setup-ruby-fastlane  — Ruby 3.2.6, gem cache, bundle install
  notify-failure       — POST failure notification to webhook
```

Each platform's build logic is defined **once** in its reusable workflow and
called by every entry point that needs it. Adding a new entry point requires
only a thin orchestrator — no duplication of steps.

The reusable workflows invoke:
```
bundle exec fastlane setup platform:<p> env:<e> kind:<k>
```
which dispatches to the correct Fastlane lane in the app repo's `Fastfile`.

---

## Cache Strategy

Bump the `v*` prefix in the relevant workflow to force a cold rebuild.

| Cache | Key inputs | Scope | ~Size |
|---|---|---|---|
| `yarn-v1` — node_modules | `yarn.lock` + OS + Node major | cross-platform | ~400 MB |
| `gem-v1` — fastlane gems | `Gemfile.lock` + OS | per-platform | ~33 MB |
| `cocoapods-v1` — pod downloads | `yarn.lock` + OS | iOS only | ~200 MB |
| `gradle-v1` — deps + build cache | `yarn.lock` + OS | Android only | ~500 MB |
| `ios-build-v2` — `ios/` + DerivedData | config hash + environment | per-env | ~1.4 GB |
| `android-build-v1` — `android/` | config hash + environment | per-env | ~300 MB |

**Config hash** covers: `app.config.js`, `app.json`, `plugins/**`, `yarn.lock`,
`assets/logo-app/**`, `assets/braze/**`, `assets/checkout/**`, `firebase/**`
(+ `assets/launchscreen-ios/**` for iOS only).

`ios/DerivedData` is stored inside `ios/` so it is captured by `ios-build-v2`.
Without this, `gym(clean: false)` on a warm cache still cold-compiles.

`PREBUILD_CACHE_HIT` is passed to Fastlane so it can skip `expo prebuild --clean`
and set `gym(clean:)` correctly on cache hits.

---

## Secrets & Variables

### Repository-level (available to all environments)

| Name | Description |
|---|---|
| `SSH_PRIVATE_KEY` | ED25519 private key with read access to the Gitea app repo |
| `PRIVATE_HOST` | Gitea hostname without port (e.g. `git.example.com`) |
| `PRIVATE_REPO_PATH` | Repo path on Gitea (e.g. `org/repo.git`) |
| `NOTIFICATION_WEBHOOK_URL` | Webhook URL for build failure notifications |
| `DRIVER_KEY_JSON` | Google service-account JSON for Drive artifact uploads |

### Environment-scoped — `staging` and `live`

| Name | Used by | Description |
|---|---|---|
| `ENV_FILE` | both | Contents of `.env.stg` / `.env.live` |
| `MATCH_GIT_URL` | iOS | SSH URL of the fastlane Match certificates repo |
| `MATCH_PASSWORD` | iOS | Passphrase for the Match repo |
| `TEAM_ID` | iOS | Apple Developer Team ID |
| `DIAWI_API_TOKEN` | iOS adhoc | Diawi upload token for ad-hoc IPA distribution |
| `ASC_API_KEY_JSON` | iOS TestFlight | App Store Connect API key JSON (`key_id`, `issuer_id`, `key`) |
| `ANDROID_KEYSTORE_BASE64` | Android live | Base64-encoded `.jks` release keystore |
| `ANDROID_KEYSTORE_PASSWORD` | Android live | Keystore password |
| `ANDROID_KEY_ALIAS` | Android live | Key alias |
| `ANDROID_KEY_PASSWORD` | Android live | Key password |

### Repository variables (non-secret)

| Name | Description |
|---|---|
| `GG_DRIVE_FOLDER_ID` | Root Google Drive folder ID for artifact uploads |
| `PROJECT_NAME` | Project name used in Drive folder structure and notifications |

---

## Adding a New Entry Point

1. Create a thin orchestrator in `.github/workflows/`.
2. Call `_ios.yml` and/or `_android.yml` via `uses: ./.github/workflows/_<platform>.yml`.
3. Pass `branch`, `environment`, and `kind` inputs. Add `secrets: inherit`.

No changes to reusable workflows or composite actions are needed unless the
build logic itself changes.
