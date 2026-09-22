# Deployment

How a change actually reaches production, in prose, not the literal scripts (those reference real local paths and real store identifiers that aren't part of this snapshot).

## The app (Android, Windows, Web)

1. CI (`.github/workflows/ci.yml`) runs static analysis and the full test suite on every push and PR against `main`, scoped to only the app's own directory so an unrelated backend or marketing change never triggers it.
2. A merge to `main` doesn't auto-publish. Publishing is a deliberate, separate step: bump the build number, build the release artifact (an app bundle for Android, a signed package for Windows), and upload it through that platform's publishing API.
3. Android publishes to a closed testing track first, then gets promoted to production once it's been validated there, using a real publisher service account rather than manual console clicks for the routine part of that flow.
4. Windows has no package-upload API at all on the platform side, a real constraint, not a tooling gap, so the build and signing steps automate, and the final submission step happens by hand in the platform's own developer dashboard.
5. Debug symbols get archived alongside the release (attached to a tagged GitHub Release), so a crash from an obfuscated production build can still be symbolicated after the local build artifacts are gone.

## The backend Worker

1. Its own CI workflow (`.github/workflows/integrity-worker-ci.yml`) runs the Worker's test suite, scoped to only that directory.
2. Deploying is a separate, explicit step (`wrangler deploy`) using a scoped API token, never bundled into the app's own release process, the two have no shared release cadence.
3. A separately-deployed sandbox environment exists for end-to-end testing against the payment processor's sandbox, so a test transaction never touches production secrets or the live custom domain.

## The marketing site

1. Its CI workflow (`.github/workflows/marketing-site-ci.yml`) checks one specific thing: that the privacy policy and terms of service pages' "Effective &lt;date&gt;" line gets updated whenever their actual content does. That check exists because it didn't happen once, real content shipped to the privacy policy without the date line moving, caught after the fact, fixed by adding a CI check with an explicit escape hatch for genuinely cosmetic edits rather than forcing a fake date bump every time.
2. Deploying rebuilds the public web trial from the app's own source (so it never ships stale) and pushes both that and the static marketing pages to the Worker in one step.
3. After deploying, the live site gets checked over HTTP, not just trusted to have worked, each real page is requested and its status code verified, so a broken deploy is caught immediately rather than discovered by a visitor.
