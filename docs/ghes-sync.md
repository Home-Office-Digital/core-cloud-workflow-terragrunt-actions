# GHES synchronisation

This repository is mirrored to two GitHub Enterprise Server (GHES) instances by
`.github/workflows/sync_with_ghes.yaml`, which calls the reusable workflow
`Home-Office-Digital/core-cloud-workflows-actions-sync/.github/workflows/sync_hub.yaml@1.1.0`.

## Triggers

- Push to `main`
- Push of any tag (`*`)

Both sync jobs run on every trigger, in parallel and independently.

## Destinations

| Setting            | Live                                                        | Test                                                           |
|--------------------|-------------------------------------------------------------|----------------------------------------------------------------|
| GHES URL           | https://github.cc-live-ops-tooling.core.homeoffice.gov.uk   | https://github.cc-test-ops-tooling.np.core.homeoffice.gov.uk   |
| GHES owner (org)   | `home-office-sandbox`                                       | `Home-Office-Test`                                             |
| Destination repo   | `core-cloud-workflow-terragrunt-actions`                    | `core-cloud-workflow-terragrunt-actions`                       |
| GitHub environment | `live-ops-tooling`                                          | `test-ops-tooling`                                             |
| Runner (`runs-on`) | `cc-ghec-actions-runner-live`                               | `cc-ghec-actions-runner-test`                                  |

The destination repository name is taken from this repository's name
(`github.event.repository.name`) by the reusable workflow; it is not configured here.

## What the reusable workflow does

1. Checks out this repository with full history (`fetch-depth: 0`).
2. Generates a short-lived GitHub App installation token for the destination GHES
   using the environment's `APP_ID` and `PRIVATE_KEY`.
3. Creates the destination repository if it does not exist. The creation request
   sets `visibility: public` (public within that GHES instance), issues off, projects off.
4. Force-pushes branches (`git push --all --force`) and tags (`git push --tags --force`)
   to the destination.

Because of step 4, any change made directly on the GHES copy is overwritten on the
next push to `main` here. The GHES copies are read-only mirrors.

## Runners

The runners are organisation-level ARC (Actions Runner Controller) scale sets, visible at
organisation level as `cc-ghec-actions-runner-live1-*` and `cc-ghec-actions-runner-test1-*`.
They carry no labels; `runs-on` must match the scale-set name. They sit inside the Home
Office network so they can reach the GHES API (`/api/v3`) and Git endpoints.

A job that waits 24 hours without a runner is cancelled automatically by GitHub. A run
showing a duration of about `1d 0h 0m` with result `cancelled` means no runner picked it up:
check the scale-set name and that the runner group is shared with this repository.

## Environments and secrets

Both environments restrict deployments to branch `main` and tags `*`. No required
reviewers or wait timer are configured. "Allow administrators to bypass configured
protection rules" is left at the GitHub default (enabled).

Each environment holds two secrets, consumed by the reusable workflow via `secrets: inherit`:

| Secret        | Purpose                                                      |
|---------------|--------------------------------------------------------------|
| `APP_ID`      | GitHub App ID registered on the destination GHES             |
| `PRIVATE_KEY` | PEM private key for that GitHub App                          |

Because the job declares `environment: <name>`, `secrets.APP_ID` resolves to that
environment's value, not a repository-level one.

## Source of truth and provisioning

AWS Secrets Manager is the source of truth. The GitHub environment secrets are working
copies; the workflow does not call AWS at run time.

- AWS accounts: CCLiveOpsTooling (live pair) and CCTestOpsTooling (test pair), one secret each
- Region: eu-west-2 (London) in both accounts
- Secret name: `reusable-workflows-sync-github-app-credentials`
- Key names: `app_id` and `private_key` (PEM, multi-line)

Provisioning (approved process, per the team):

1. Sign in to the AWS console for the matching account, CCLiveOpsTooling or CCTestOpsTooling (requires the corporate network and VPN).
2. Secrets Manager > the secret above > Retrieve secret value.
3. In this repository: Settings > Environments > the environment > Add environment secret.
   Create `APP_ID` and `PRIVATE_KEY` with the values for that destination.
4. Repeat for the other environment. Do not assume test and live share credentials.
5. Never paste the values anywhere else (issue, PR, Slack, logs, screenshots).

## Rotation

When the GitHub App key is rotated in AWS Secrets Manager, the GitHub environment
secrets must be updated by hand using the provisioning steps above. Nothing alerts on
drift: the symptom is the sync failing at the "Generate GitHub App Token" step with an
authentication error, on the first push after rotation.

## Troubleshooting

| Symptom                                                        | Likely cause                                              |
|----------------------------------------------------------------|-----------------------------------------------------------|
| Run queued for ~24h then cancelled                             | No runner matches `runs-on`; check scale-set name/group   |
| Fails at "Generate GitHub App Token"                           | Missing/stale `APP_ID` or `PRIVATE_KEY`; rotation drift   |
| Fails at "Create GHES repo" with 403                           | GitHub App lacks repo-creation permission in that org     |
| Fails at push with 403                                         | GitHub App lacks contents write / not installed on repo   |
| Workflow does not start on push to `main`                      | Workflow disabled (Actions tab shows a yellow banner)     |

## Known limitation

Mirroring alone does not make this repository usable on GHES. Its workflows and actions
reference `Home-Office-Digital/...` paths (13 occurrences in `.github/` and `actions/`)
that do not exist on the destination instances, and `uses:` cannot take expressions, so
they cannot be parameterised in place. Making the mirrors functional needs a rewrite step
in the sync and the dependency repositories mirrored too; this is tracked separately.
The mirrored `sync_with_ghes.yaml` will also attempt to run on GHES and fail harmlessly.
