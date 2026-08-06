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
