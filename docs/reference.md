# Workflow reference

Inputs, outputs, secrets and the token permissions each workflow needs.
A called workflow can only narrow the caller's token, so grant the listed
permissions on the calling job. Durations are whole seconds.

- **Code quality:** [`test.yml`](#testyml), [`precommit.yml`](#precommityml), [`coverage.yml`](#coverageyml), [`pr-title.yml`](#pr-titleyml)
- **Security and supply chain:** [`audit.yml`](#audityml), [`gitleaks.yml`](#gitleaksyml), [`codeql.yml`](#codeqlyml), [`dep-review.yml`](#dep-reviewyml), [`iac-scan.yml`](#iac-scanyml), [`ossf-scorecard.yml`](#ossf-scorecardyml), [`sarif-upload.yml`](#sarif-uploadyml)
- **Build and ship:** [`build.yml`](#buildyml), [`container-scan.yml`](#container-scanyml), [`dast.yml`](#dastyml), [`deploy.yml`](#deployyml)
- **Infrastructure:** [`terraform.yml`](#terraformyml), [`k8s-validate.yml`](#k8s-validateyml)
- **Release and housekeeping:** [`app-release.yml`](#app-releaseyml), [`cleanup.yml`](#cleanupyml), [`notify.yml`](#notifyyml)

## Code quality

### test.yml

Needs `contents: read`.

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `runner` | no | `ubuntu-latest` | Runner label for the Rust jobs |
| `clippy_all_targets` | no | `false` | Also lint tests, benches and examples with all features and `--locked` |

Caches with `Swatinem/rust-cache`, keyed per job on the toolchain and lockfile.

### precommit.yml

Needs `contents: read`. Installs stable (with rustfmt and clippy), nightly
(with rustfmt and clippy), `cargo-udeps`, Python and `pre-commit` (release zipapp pinned by SHA-256).

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `runner` | no | `ubuntu-latest` | Runner label for the precommit job |

| Output | Description |
| --- | --- |
| `precommit_duration` | Seconds taken by the precommit job |

### coverage.yml

Needs `contents: read`. Runs `cargo llvm-cov --all-features --locked
--workspace`, writes a summary table to the job summary and uploads
`lcov.info` as artifact `coverage-lcov` (7 days).

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `runner` | no | `ubuntu-latest` | Runner label |
| `min_coverage` | no | `0` | Fail when line coverage (%) is below this. `0` only reports |

| Output | Description |
| --- | --- |
| `line_coverage` | Line coverage in percent, two decimals |

### pr-title.yml

Needs `pull-requests: read`. Pull requests only: the title must be a
[Conventional Commit](https://www.conventionalcommits.org/). With squash
merges the title becomes the commit, which `app-release.yml` reads.

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `types` | no | Conventional Commits defaults | Allowed types, one per line |
| `require_scope` | no | `false` | Require a scope, e.g. `feat(api): ...` |

## Security and supply chain

### audit.yml

Needs `contents: read`, `checks: write`, `issues: write`.

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `runner` | no | `ubuntu-latest` | Runner label for the cargo-deny job |

### gitleaks.yml

Needs `contents: read`, `security-events: write`. Secret scan with
[gitleaks](https://github.com/gitleaks/gitleaks) (release binary pinned by
SHA-256). Uses the caller's `.gitleaks.toml` and `.gitleaksignore`; findings
are redacted in the log.

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `full_history` | no | `true` | Scan every commit; `false` scans only the checked-out files |
| `sarif` | no | `true` | Upload findings to code scanning |

### codeql.yml

Needs `contents: read`, `actions: read`, `security-events: write`. One job
per language, `build-mode: none`. Results go to code scanning; alerts do not
fail the job (gate them with a branch ruleset's code scanning rule).

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `languages` | no | `["rust", "actions"]` | CodeQL languages, as a JSON array. `actions` analyses the workflow files |
| `queries` | no | `security-extended` | `default`, `security-extended` or `security-and-quality` |

### dep-review.yml

Needs `contents: read`, `pull-requests: write`. Pull requests only: fails
when the PR adds a dependency with a known advisory or a denied license.
Needs the dependency graph.

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `fail_on_severity` | no | `high` | `low`, `moderate`, `high` or `critical` |
| `deny_licenses` | no | | Comma-separated SPDX licenses that fail the PR |
| `comment` | no | `on-failure` | PR summary comment: `always`, `on-failure` or `never` |

### iac-scan.yml

Needs `contents: read`, `security-events: write`. Trivy misconfiguration
scan of Dockerfiles, Terraform, Kubernetes manifests and Helm charts. With
`sarif`, every severity goes to code scanning; the gate uses `fail_severity`.

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `path` | no | `.` | Directory to scan |
| `fail_severity` | no | `CRITICAL,HIGH` | Fail on these severities. Empty disables the gate |
| `skip_dirs` | no | | Comma-separated directories to skip |
| `sarif` | no | `true` | Upload findings to code scanning |

### ossf-scorecard.yml

Needs `contents: read`, `actions: read`, `security-events: write`. Runs
[OpenSSF Scorecard](https://scorecard.dev/) on the calling repo and sends
the results to code scanning. No inputs. Run it on the default branch.

### sarif-upload.yml

Needs `security-events: write`. Uploads SARIF artifacts made earlier in the
same caller run (for example `container-scan.yml` with `sarif: true`) to
code scanning. Each artifact becomes its own category, so per-arch results
do not collide. Waits for each upload to finish processing.

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `pattern` | no | `sarif-*` | Artifact name pattern to upload |

## Build and ship

### build.yml

Needs `contents: read`, `packages: write`, `id-token: write`.

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `sha` | yes | | Image tag to publish, normally the commit SHA |
| `repo_name` | yes | | Image name under `ghcr.io/<owner>/` |
| `docker_targets` | no | `["final"]` | Dockerfile stages to build, as a JSON array. The first keeps plain tags, others get a `-<target>` suffix |
| `build_args` | no | | Extra docker build args, one `KEY=VALUE` per line. `ARCH` and `GITHUB_ORGANIZATION` are always set |
| `runner_amd64` | no | `ubuntu-latest` | Runner label for amd64 builds |
| `runner_arm64` | no | `ubuntu-24.04-arm` | Runner label for arm64 builds |
| `branch_slug`, `dependency_clone`, `dependency_repo`, `dependency_ref` | no | | Unused. Kept so existing callers that pass them do not break |

| Secret | Required | Description |
| --- | --- | --- |
| `GH_ACTION_PAT` | no | PAT for GHCR and private deps. Falls back to `GITHUB_TOKEN` |
| `GH_ACTION_ACTOR` | no | User for `GH_ACTION_PAT`. Falls back to `github.actor` |
| `GH_ACTION_USER` | no | Passed to the build as secret `github_username`. Falls back to `github.actor` |

| Output | Description |
| --- | --- |
| `build_duration` | Seconds taken by the last image build job to finish |
| `digest` | Manifest list digest (`sha256:...`) of the first docker target, as signed |

Tags: `<sha>`, plus `<version>` and `<major>.<minor>` on semver tags. Each
image carries BuildKit SBOM and provenance (`mode=min`) attestations.

### container-scan.yml

Needs `packages: read`, plus `pull-requests: write` for the Scout PR comment.

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `sha` | yes | | Image tag to scan. Ignored when `digest` is set |
| `digest` | no | | Manifest list digest to scan instead of the tag, normally `build.yml`'s `digest` output |
| `repo_name` | yes | | Image name under `ghcr.io/<owner>/` |
| `archs` | no | `["amd64", "arm64"]` | Architectures to scan, as a JSON array. `sbom` and `scout` run on amd64 only |
| `tools` | no | `["anchore", "sbom", "trivy"]` | Scanners to run, as a JSON array. Add `"scout"` only with `DOCKER_USER` / `DOCKER_PAT` set |
| `fail_severity` | no | `CRITICAL,HIGH` | Trivy fails the scan on fixable findings of these severities. Empty disables the gate |
| `sarif` | no | `false` | Also save Trivy findings as `sarif-trivy-image-<arch>` artifacts; upload them with `sarif-upload.yml` |
| `branch_slug` | no | | Unused. Kept so existing callers that pass it do not break |

| Secret | Required | Description |
| --- | --- | --- |
| `GH_ACTION_PAT` | no | PAT for GHCR. Falls back to `GITHUB_TOKEN` |
| `GH_ACTION_ACTOR` | no | User for `GH_ACTION_PAT`. Falls back to `github.actor` |
| `DOCKER_USER` | no | Docker Hub user, needed for the scout tool |
| `DOCKER_PAT` | no | Docker Hub token, needed for the scout tool |

| Output | Description |
| --- | --- |
| `anchore_scan_duration` | Seconds taken by the anchore scan |
| `anchore_sbom_scan_duration` | Seconds taken by the sbom scan |
| `trivy_scan_duration` | Seconds taken by the trivy scan |
| `docker_scout_scan_duration` | Seconds taken by the scout scan |

### dast.yml

Needs `contents: read`, `packages: read`. Starts `image` as a service
container and runs the [ZAP baseline scan](https://www.zaproxy.org/docs/docker/baseline-scan/)
(passive: spider and inspect responses, no attacks) against it. The HTML and
JSON report is uploaded as artifact `zap-report`.

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `image` | yes | | Image to run, ideally `ghcr.io/<owner>/<repo>@sha256:...` |
| `port` | no | `8080` | Port the container listens on |
| `path` | no | `/` | Path to start crawling from |
| `fail_on_warnings` | no | `false` | Fail on WARN alerts too. By default only rules set to FAIL in `rules_file` fail |
| `rules_file` | no | | ZAP rules file in the caller repo (tab-separated: id, `IGNORE`/`WARN`/`FAIL`, name) |

| Secret | Required | Description |
| --- | --- | --- |
| `GH_ACTION_PAT` | no | PAT to pull the image. Falls back to `GITHUB_TOKEN` |
| `GH_ACTION_ACTOR` | no | User for `GH_ACTION_PAT`. Falls back to `github.actor` |

### deploy.yml

Needs no token permissions; it uses `DISPATCH_PAT`. Sends event type
`update-image-tag` with payload `target_dir`, `repo_name`, `app_name`, `sha`,
`digest` (empty string when not passed).

| Input | Required | Description |
| --- | --- | --- |
| `sha` | yes | Image tag to deploy |
| `digest` | no | Image digest, normally `build.yml`'s `digest` output, forwarded so the deploy repo can pin it |
| `branch_slug` | yes | Branch name, used as the target folder for non-release refs |
| `repo_name` | yes | Source repo name, passed through to the deploy repo |
| `app_name` | yes | App name, passed through to the deploy repo |

| Secret | Required | Description |
| --- | --- | --- |
| `DISPATCH_PAT` | yes | PAT allowed to send `repository_dispatch` to the deploy repo |
| `DEPLOY_REPO` | yes | Name of the deploy (GitOps) repo under the same owner |

| Output | Description |
| --- | --- |
| `deploy_duration` | Seconds taken by the deploy job |

## Infrastructure

### terraform.yml

Needs `contents: read`. `terraform fmt -check`, `init -backend=false` and
`validate` (no backend, no cloud credentials), then `tflint --recursive`.
Pair with `iac-scan.yml` for misconfigurations.

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `working_directory` | no | `.` | Directory holding the root module |
| `terraform_version` | no | `latest` | Version constraint, e.g. `~1.9` |
| `tflint_version` | no | `latest` | tflint version, e.g. `v0.55.0` |

### k8s-validate.yml

Needs `contents: read`. `helm lint --strict` and `helm template` for a chart,
then [kubeconform](https://github.com/yannh/kubeconform) (release binary
pinned by SHA-256) on the manifests and the rendered chart. CRDs are checked
against the [CRDs catalog](https://github.com/datreeio/CRDs-catalog).

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `manifests_path` | no | | Directory of plain manifests. Set this, `chart_path`, or both |
| `chart_path` | no | | Helm chart directory |
| `values_file` | no | | Values file for the chart |
| `kubernetes_version` | no | `1.33.0` | Kubernetes version to validate against |
| `ignore_missing_schemas` | no | `false` | Skip resources with no known schema instead of failing |
| `helm_version` | no | `latest` | Helm version, e.g. `v3.19.0` |

## Release and housekeeping

### app-release.yml

Needs `contents: write`, `pull-requests: write`, `issues: write`.
[release-please](https://github.com/googleapis/release-please) for the
calling app: keeps a release PR with the next version and changelog from
Conventional Commits; merging it tags `vX.Y.Z` and publishes the release.
Call on push to the default branch.

`GITHUB_TOKEN` can only open the release PR when the repo allows it
(Settings, Actions, General, "Allow GitHub Actions to create and approve pull
requests"). Tags and PRs made with `GITHUB_TOKEN` also start no other
workflow, so either pass `RELEASE_TOKEN` or gate build and deploy in the same
caller workflow on `release_created`.

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `release_type` | no | `rust` | Strategy: `rust`, `simple`, `node`, `go`, `helm`, ... |
| `config_file` | no | | `release-please-config.json` for monorepos; replaces `release_type` |
| `manifest_file` | no | | `.release-please-manifest.json`, with `config_file` |

| Secret | Required | Description |
| --- | --- | --- |
| `RELEASE_TOKEN` | no | PAT so the release PR and tag start CI. Falls back to `GITHUB_TOKEN` |

| Output | Description |
| --- | --- |
| `release_created` | `true` when this run tagged a release |
| `tag_name` | Tag of the created release, e.g. `v1.2.3` |
| `version` | Version of the created release, e.g. `1.2.3` |

### cleanup.yml

Needs `actions: write`. Call on `pull_request` with `types: [closed]`:
deletes the Actions caches the PR's runs created, so they stop counting
against the 10 GB repo limit. No inputs.

### notify.yml

Needs no token permissions.

| Input | Required | Description |
| --- | --- | --- |
| `sha`, `branch_slug`, `repo_name`, `commit_message`, `commit_url`, `commit_actor`, `event_name`, `workflow_name`, `run_url` | yes | Commit and run details shown in the alert |
| `job_status` | yes | One of `success`, `failure`, `started`, `cancelled` |
| `precommit_duration`, `build_duration`, `anchore_scan_duration`, `anchore_sbom_scan_duration`, `trivy_scan_duration`, `docker_scout_scan_duration`, `deploy_duration` | no | Shown as fields when set |

| Secret | Required | Description |
| --- | --- | --- |
| `SLACK_WEBHOOK_URL` | yes | Slack incoming webhook URL |
