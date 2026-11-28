# Shadow comparison (shadow)

The shadow comparison checks that the refactored Go implementation and the retained legacy bash scripts produce the same output and PKI state for read-only and key write paths. It is a gate in the CI `build-and-test` job.

## Goals

- Compare list, verify, issuance, export, index sync, revoke/unrevoke, platform settings and delete paths.
- Compare stdout/stderr, key `index.txt` columns, the meta directory and the client bundle tree.
- Normalize dynamic values (dates, serials, temporary paths), but never ignore real behavioral differences.

## Fixture

`testdata/gen-fixture.sh` creates a minimal PKI in a temporary directory: TMP01 (valid, windows/android), TMP02 (valid, ios), TMP03 (revoked; the CRL contains its serial), TMP05 (orphan certificate for index sync) and DEADBEEF... (revoked record only in index/CRL). The fixture never reads or modifies a real PKI.

## Run

Inside the development container:

~~~bash
make shadow
~~~

The script builds the current binaries, runs the legacy scripts and new commands in separate directories with matching temporary environment variables, and diffs each case.

## Normalization

The comparison normalizes OpenSSL dates and remaining days, serials and temporary absolute paths, the new `cert verify` chain ✓/✗ marker, and time-related temporary fields in bundle directories. Normalization removes only these explainable differences; command output, index state, CRL entries and file trees are still compared strictly.

## Pass criterion

The last line must be the script's success marker (currently Simplified Chinese):

~~~text
[OK]   影子对比全部通过（diff = 0）
~~~

Any `[ERR]` makes `make shadow` exit non-zero and blocks CI.

## Boundaries and safety

- Use only temporary fixtures; no real CA, client private keys or server configuration.
- Do not call Docker reload; the test configuration disables side effects.
- Never commit fixture passwords, private keys or generated material.
- The comparison verifies behavioral consistency; it does not replace security review, real-environment regression or rollback drills.
