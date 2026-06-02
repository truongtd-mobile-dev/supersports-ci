# Supersports CI Sandbox

GitHub Actions workflow for building Supersports v567app from Gitea source.

## Trigger

Actions tab → "Mobile Build Internal Testing (Android)" → Run workflow → choose branch, environment, kinds.

## Source code

Cloned at runtime from a private Gitea instance — host and repo path live
in the `PRIVATE_HOST` and `PRIVATE_REPO_PATH` repository secrets.

## Secrets / vars required

See `.github/workflows/build-android.yml` for the full list.
