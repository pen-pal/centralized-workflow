# Security policy

## Supported versions

| Version | Supported |
| --- | --- |
| `v1` (latest `v1.x.y`) | Yes |
| Older `v1.x.y` tags | Fixed by moving to the latest `v1` |

Security fixes ship as patch releases and move the `v1` tag (see
[RELEASING.md](RELEASING.md)).

## Reporting a vulnerability

Report privately through
[GitHub private vulnerability reporting](https://github.com/pen-pal/centralized-workflow/security/advisories/new).
Do not open a public issue for anything exploitable.

Useful details: the workflow and ref, the caller configuration, and what an
attacker controls (an input, a branch name, a pull request from a fork, ...).

Expect an acknowledgement within a week. Fixes are released before the
advisory is published.

## Scope

In scope: the reusable workflows in `.github/workflows/`, for example
injection through inputs, over-broad token permissions, or a bypass of
signing, scanning or release gates.

Out of scope: findings in third-party actions themselves (report those
upstream), and vulnerabilities in the images or code a caller builds.
