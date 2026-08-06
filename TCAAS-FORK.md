# T-CaaS temporary fork

This branch is a temporary bridge, not a permanent fork. It exists only so the
T-CaaS external-dns function can stop depending on a stranded external-dns fork
while our IPv6 patches wait for upstream review.

## Why it exists

Two pull requests against `AbsaOSS/external-dns-infoblox-webhook` are open and
unmerged. Upstream CI has not been approved to run on them because the author is
a first-time contributor.

- [PR #69](https://github.com/AbsaOSS/external-dns-infoblox-webhook/pull/69)
  `feat(infoblox): add AAAA (IPv6) record support`
- [PR #70](https://github.com/AbsaOSS/external-dns-infoblox-webhook/pull/70)
  `feat(infoblox): create PTR records for AAAA records`
  (stacked on PR #69, adds `ip6.arpa` nibble reverse zones)

Upstream release `v1.7.2` has no AAAA support at all. `internal/infoblox/infoblox.go`
is hard-wired to IPv4: the record-type guards reject anything other than `A`, only
`Ipv4Addr` is ever written, the WAPI query parameters hardcode `ipv4addr`, and PTR
companion records are only created for `A` records.

## What is carried

Upstream base tag: `v1.7.2` (commit `37d23e201870b46acb0694b3500b89bbad5f0a8d`,
which is byte-identical to upstream `main` at the time of forking).

Exactly two commits on top of that base, in this order:

1. `1c01b1d` `feat(infoblox): add AAAA (IPv6) record support`
2. `290da42` `feat(infoblox): create PTR records for AAAA records`

Both are the same commits pushed to the PR branches `feat/aaaa-record-support`
and `feat/ipv6-ptr-support`. This branch adds no provider behaviour of its own,
only this document.

## Image

`.github/workflows/tcaas-image.yaml` publishes
`ghcr.io/maxrink/external-dns-infoblox-webhook:<tag>` (amd64 + arm64) on a
`v*-telekom.*` tag push, using only the built-in `GITHUB_TOKEN`.

Current bridge image:

```
ghcr.io/maxrink/external-dns-infoblox-webhook:v1.7.2-telekom.3
```

| Manifest | Digest |
| --- | --- |
| manifest list (OCI index) | `sha256:336bb21bfc4bf4af1c52b0af2d9e66190d9b48e05ee3b1f50468f4fd3b1b44fc` |
| `linux/amd64` | `sha256:f1e69f4c922b415d881a4330f98fd3a198a72546eae887549a66778dfdeac541` |
| `linux/arm64` | `sha256:1eae059d1645713dab2dc2b7d93a435565b2b4e8513082936d45e76b4d147631` |

The GHCR package is public: an anonymous pull token fetches the index and both
per-architecture manifests, so T-CaaS clusters need no image pull secret. The
image is a bridge built from unmerged code; treat the pin as temporary.

Note that `v1.7.2-telekom.3` reports `Gitsha` `e5307471` rather than its own tag
commit `097830d7`. It was published by a `workflow_dispatch` before the SHA
stamping was corrected; the image content is built from the tag, only the
recorded SHA is off. Tags from `v1.7.2-telekom.4` onwards stamp correctly.

### Actions is enabled

GitHub Actions **is** enabled on this fork; an earlier version of this document
claimed the opposite and was wrong. The reason the first two `-telekom.N` tag
pushes published nothing is different, and worth recording:

> A workflow only becomes registered, and its triggers only start being
> evaluated, once GitHub has seen the file on the repository **default branch**.

`tcaas-image.yaml` initially existed only on `tcaas-main`, so it was never
registered: `actions/workflows` listed only the four upstream workflows, and
tag pushes of `v1.7.2-telekom.1`, `.2` and `.3` each created zero runs even
though the workflow file was present on the tagged commits and parsed cleanly.
Adding the file to `main` registered it immediately.

So this one fork-only workflow must stay on `main`. It carries a
`workflow_dispatch` with a `ref` input for on-demand rebuilds, and refuses to
publish unless the resolved ref matches `v*-telekom.*`, so a dispatch from
`main` cannot publish a misleadingly-tagged image.

## Retirement

Retire this branch and this fork as soon as an upstream release contains both
patches. Repoint the T-CaaS external-dns function at the upstream image
(`ghcr.io/absaoss/external-dns-infoblox-webhook`) and delete the
`ghcr.io/maxrink/...` pin.

## How to verify whether upstream has shipped it

```sh
# 1. Are both PRs merged?
gh pr view 69 --repo AbsaOSS/external-dns-infoblox-webhook --json state,mergedAt,mergeCommit
gh pr view 70 --repo AbsaOSS/external-dns-infoblox-webhook --json state,mergedAt,mergeCommit

# 2. Is the code actually in the latest upstream *release*, not just main?
TAG=$(gh release view --repo AbsaOSS/external-dns-infoblox-webhook --json tagName -q .tagName)
gh api "repos/AbsaOSS/external-dns-infoblox-webhook/contents/internal/infoblox/infoblox.go?ref=$TAG" \
  -H 'Accept: application/vnd.github.raw' | grep -c 'RecordTypeAAAA'
```

A merged PR is not enough. Both checks must pass: `mergedAt` non-null for #69 and
#70, and the grep against the released tag returning a non-zero count. Only then
is the upstream image a valid replacement.
