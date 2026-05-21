# npm-staged-publish-tests

Throwaway repo for experiments on **npm staged publishing × trusted publishing × provenance**, in support of [semantic-release/npm#1160](https://github.com/semantic-release/npm/issues/1160).

The experiment confirms server-side behaviours that cannot be observed from the npm CLI source alone:

- **Test B** — after `npm stage approve`, does the published packument still carry provenance (`dist.attestations`)? Tests [a report by @willow.sh](https://bsky.app/profile/willow.sh/post/3mmeumyotds2n) that provenance is *lost* after a staged approval.
- **Tests C/D** — after `npm stage reject`, can the **same version** be re-staged *with* provenance, or is the version number burned by a pre-existing attestation record? Tests [a speculation by @dominikg.dev](https://bsky.app/profile/dominikg.dev/post/3mmezubp7ps2s). Test D repeats it *without* provenance to isolate the cause.
- **Test E** — control: stage → reject → re-stage with no provenance anywhere.
- **Test F** — manual: look the rejected versions up on <https://search.sigstore.dev/>; the Rekor entry is immutable and will persist regardless of reject.

## Why it matters

`@semantic-release/npm` publishes with `provenance: true`. If a rejected staged version is permanently burned for provenance re-staging, the "re-run semantic-release after a reject" recovery path is invalid for provenance users — semantic-release would re-compute the same version from the same commits and fail forever.

## One-time setup (on npmjs.com)

The experiment publishes the package `@47ng/staged-publishing-test`.

1. The package must already **exist** and be **public** (provenance and staging both require an existing package; provenance requires public access). Publish `0.0.0` once normally if it does not exist yet.
2. Configure a **trusted-publishing relationship** for `@47ng/staged-publishing-test` pointing at this repo (`franky47/npm-staged-publish-tests`) and the workflow `staged-publishing-provenance-experiment.yml`, with **`npm stage publish` allowed** (`--allow-stage-publish`). No `NPM_TOKEN` secret is then needed.
3. Enable **2FA** on the account that will approve/reject.

If you reuse a different package name, update the `PKG` / `PKG_ENC` env in the workflow.

## How to run

Staged publishing has a human 2FA gate, so the experiment is several `workflow_dispatch` runs with a manual approve/reject in between (via `npm stage approve|reject <id>` or npmjs.com).

### Test B — does provenance survive approval?

1. Run the workflow with `step=stage-prov`. Note the printed **version** and **stage id**.
2. Manually `npm stage approve <stage-id>` (2FA).
3. Run with `step=inspect`, `version=<that version>`. Read the job summary.

### Tests C/D — is the version burned by reject?

4. Run with `step=stage-prov`. Note the **version** and **stage id**.
5. Manually `npm stage reject <stage-id>` (2FA).
6. Run with `step=restage`, `version=<that version>`, `provenance=true` — does it fail?
7. Run with `step=restage`, `version=<same version>`, `provenance=false` — does it fail?

### Test E — control, no provenance

8. Run with `step=stage-noprov`. Note the **version** and **stage id**.
9. Manually `npm stage reject <stage-id>` (2FA).
10. Run with `step=restage`, `version=<that version>`, `provenance=false` — expected: success.

Each run's job summary records the outcome. Findings feed back into the
semantic-release/npm staged-publishing plan.
