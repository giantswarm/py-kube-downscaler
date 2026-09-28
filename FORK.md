# The Giant Swarm line of py-kube-downscaler

This repository is Team Planeteers' fork of [caas-team/py-kube-downscaler](https://github.com/caas-team/py-kube-downscaler).
Giant Swarm runs py-kube-downscaler through the [`giantswarm/kube-downscaler-app`](https://github.com/giantswarm/kube-downscaler-app)
chart, and this fork is where that image is pinned, patched, built, scanned and published. It answers one question
from one place: **which py-kube-downscaler are we running, and why does it differ from upstream?**

## Branches

| Branch | What it is | Who moves it |
|---|---|---|
| `main` | upstream's `main` as of the fork. Never edited, never the target of a pull request. | nobody |
| `giantswarm` (default) | **The line**: the upstream release tag the chart runs ("the pin") + the carried patches + this fork's own files. Every change is a pull request against it. | pull requests; a repository admin for a re-pin |
| `fork/<topic>` | pull-request branches against `giantswarm` | the team |
| `upstream/<topic>` | a carried patch on upstream's tag, shaped as the upstream pull request | the team |

## Pin

| | |
|---|---|
| Upstream release | **v26.4.0** (2026-04-08, `67661849`), upstream's latest release |
| Derived how | `git merge-base giantswarm v26.4.0` with upstream's tags fetched |
| When it moves | when upstream releases a version the chart needs, or when upstream carries every patch below: re-pin (below) |

## Carried patches

| Patch | Why | Commit on `giantswarm` | Upstream |
|---|---|---|---|
| Leave HPA targets to the HPA and skip KEDA-owned HPAs: with `horizontalpodautoscalers` included and downtime replicas above 0, a workload an HPA targets is treated as excluded and only the HPA's `minReplicas` is lowered; HPAs owned by a ScaledObject are never scaled directly; `downscaler/original-replicas: -1` is honoured on ScaledObjects only | the downscaler scaled a Deployment down and lowered its HPA's `minReplicas`, and the HPA scaled the Deployment straight back up every cycle; KEDA copies a ScaledObject's `downscaler/original-replicas: -1` onto its HPA, which the downscaler then wrote as `minReplicas: -1` (a 422 every cycle). giantswarm/giantswarm#37952 | `af6a4df` (the squash of pull request #2) | to file: the upstream-shaped commit with DCO sign-off is branch `upstream/hpa-aware` here, on v26.4.0; no upstream issue or pull request covers it. Queued in giantswarm/giantswarm#37742 (row 79) |
| Security fixes of the image: `apk upgrade` in the runtime stage of the `Dockerfile`, urllib3 ≥ 2.7.0 in `poetry.lock` | the scan in `.circleci/config.yml` fails on fixable HIGH findings of the pinned release: Alpine's openssl, util-linux and sqlite, urllib3 2.6.3 | `62a349e` (the squash of pull request #3) | not for upstream as is; falls away at the re-pin onto a release that carries them |
| Fork infrastructure: this file, `CODEOWNERS`, `renovate.json5` (disabled), `.circleci/config.yml`, `.trivyignore`; upstream's GitHub Actions workflows removed (they publish to ghcr.io and prune it) | the line's CI and publishing | the `giantswarm` branch history | not for upstream; ours to keep |

## Publishing

CircleCI (`.circleci/config.yml`) runs upstream's tests on every branch. A push to `giantswarm` publishes a dev build and
a tag `vX.Y.Z` a release to `gsoci.azurecr.io/giantswarm/py-kube-downscaler/kube-downscaler:X.Y.Z`. The index carries
`io.giantswarm.upstream.version`, the pin.

The line's versions are its own semver, above the pin: the first release is `v26.4.1` (upstream v26.4.0 plus the carried
patches). A carried patch or a pipeline change is a patch bump, a re-pin at least a minor bump onto a version above the
new pin. The orb's version computation (gitsemver) knows only bare `vX.Y.Z` tags, so there are no `-gs.N` suffixes. Nothing
is published from GitHub Actions, by hand, or to ghcr.io. The retagger's mirror of upstream's own images
(`gsoci.azurecr.io/giantswarm/py-kube-downscaler`) is a different repository and is not used by the chart.

A release is tagged on `giantswarm` after its pull request merged: `git tag -a vX.Y.Z -m "…"` and push the tag.

| Release | Pin | Carries |
|---|---|---|
| `v26.4.1` (2026-09-28) | upstream v26.4.0 | HPA-aware downscaling, the image security fixes |

The line's version numbers can match a later upstream release (upstream may tag its own v26.4.1). They are different
artifacts: this table and the image's `io.giantswarm.upstream.version` annotation say which upstream a line release
carries, and the line's images live under their own path. A re-pin fetches upstream's tags into their own namespace
(`git fetch upstream 'refs/tags/*:refs/tags/upstream/*'`), so they never overwrite the line's tags.

## Re-pin

1. Fetch upstream's tags (`refs/tags/upstream/*`, above) and branch `fork/repin-<tag>` from the new tag.
2. Replay the fork infrastructure commits and every carried patch upstream does not contain; drop the rows of those it
   does.
3. Update "Pin" and the `upstream_annotation` in `.circleci/config.yml`.
4. A repository admin moves `giantswarm` to the reviewed branch; tag the next minor above the new pin; bump kube-downscaler-app.
