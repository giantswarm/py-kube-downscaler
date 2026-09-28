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
| Leave HPA targets to the HPA and skip KEDA-owned HPAs: with `horizontalpodautoscalers` included and downtime replicas above 0, a workload an HPA targets is treated as excluded and only the HPA's `minReplicas` is lowered; HPAs owned by a ScaledObject are never scaled directly; `downscaler/original-replicas: -1` is honoured on ScaledObjects only | the downscaler scaled a Deployment down and lowered its HPA's `minReplicas`, and the HPA scaled the Deployment straight back up every cycle; KEDA copies a ScaledObject's `downscaler/original-replicas: -1` onto its HPA, which the downscaler then wrote as `minReplicas: -1` (a 422 every cycle). giantswarm/giantswarm#37952 | the squash of the patch's pull request | to file: the upstream-shaped commit with DCO sign-off is branch `upstream/hpa-aware` here, on v26.4.0; no upstream issue or pull request covers it. Queued in giantswarm/giantswarm#37742 |
| Fork infrastructure: this file, `CODEOWNERS`, `renovate.json5` (disabled), `.circleci/config.yml`, `.trivyignore`; upstream's GitHub Actions workflows removed (they publish to ghcr.io and prune it) | the line's CI and publishing | the `giantswarm` branch history | not for upstream; ours to keep |

## Publishing

CircleCI (`.circleci/config.yml`) runs upstream's tests on every branch. A push to `giantswarm` publishes a dev build and
a tag `vX.Y.Z-gs.N` a release to `gsoci.azurecr.io/giantswarm/py-kube-downscaler/kube-downscaler:X.Y.Z-gs.N`, where
`X.Y.Z` is the pin and `N` counts the line's releases on it. The index carries `io.giantswarm.upstream.version`. Nothing
is published from GitHub Actions, by hand, or to ghcr.io. The retagger's mirror of upstream's own images
(`gsoci.azurecr.io/giantswarm/py-kube-downscaler`) is a different repository and is not used by the chart.

A release is tagged on `giantswarm` after its pull request merged: `git tag -a v26.4.0-gs.N -m "…"` and push the tag.

## Re-pin

1. Fetch upstream's tags and branch `fork/repin-<tag>` from the new tag.
2. Replay the fork infrastructure commits and every carried patch upstream does not contain; drop the rows of those it
   does.
3. Update "Pin", the `upstream_annotation` in `.circleci/config.yml`, and the release tag's `X.Y.Z`.
4. A repository admin moves `giantswarm` to the reviewed branch; tag `vX.Y.Z-gs.1`; bump kube-downscaler-app.
