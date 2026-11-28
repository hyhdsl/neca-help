# GitHub workflows

The repository has three workflows: quality gating, Release assets and container images. All are defined in repository YAML; releases are not simulated locally.

## 1. `ci.yml`: quality gate

- Trigger: pushes to `main`, `feat/**`, `fix/**`, `chore/**`, and every Pull Request.
- Permissions: repository contents read-only (`contents: read`); no release key.
- Jobs: `build-and-test` and the dependent `docker-smoke`.
- Checks: `gofmt`, `go vet`, `govulncheck`, ASCII platform templates, locale-key consistency, Markdown links/fences/help-site structure, `go test -race ./...`, Linux amd64/arm64 cross-compilation, shadow comparison and a `version` smoke test; then build the lite/web images and verify their entry points.
- Do not merge on failure; `main` branch protection requires `build-and-test`.

## 2. `release.yml`: GitHub Release

- Trigger: pushing any `v*` tag.
- Preflight: tag format, whether the tagged commit is on `main` or `release/x.y`, and whether that commit's `build-and-test` succeeded.
- Release job: uses the protected `release-signing` environment; only `v*` tags may enter.
- Flow: checkout → build Linux amd64/arm64 `neca`, `necad` and `neca-web-helper` → generate `checksums.txt` → import the release private key into a temporary GNUPGHOME → sign and self-verify → export the public key → create the Release.
- Assets: six binaries + `checksums.txt` + `checksums.txt.asc` + `neca-gpg-key.asc`, nine in total.
- Pre-release rule: a tag containing `-` (for example `v1.6.0-preview.1`) is marked Pre-release; stable tags are not.
- The release comes from the tagged commit; do not change code after tagging and expect the Release to update. See [GPG verification](gpg.md).

## 3. `docker.yml`: multi-architecture GHCR images

- Trigger: any `v*` tag, or manual `workflow_dispatch`.
- Permissions: `contents: read`, `packages: write`.
- Flow: build binaries → QEMU/Buildx → log in to GHCR → build and push two images:
  - `ghcr.io/hyhdsl/neca` (lite/CLI);
  - `ghcr.io/hyhdsl/neca-web` (Web/helper).
- Platforms: `linux/amd64`, `linux/arm64`.
- Tags: images use the Git tag; stable tags also update `latest`; preview tags containing `-` do not update `latest`.

## Release order

1. Merge the change through a PR to `main` and wait for `build-and-test`.
2. Create and push an annotated tag, for example `v1.6.0`.
3. Confirm both `release.yml` and `docker.yml` succeed.
4. Check the nine Release assets, checksums, signature and GHCR tags.
5. Deployment and rollback use a separately approved operations process; this page records no server details.

## Troubleshooting

- **Release succeeds but Docker fails**: check GHCR package permissions or registry login; Release assets do not depend on Docker.
- **A preview updated latest**: do not overwrite manually; check the tag for `-` and the metadata enablement conditions.
- **CI shadow comparison fails**: first check whether the old/new text differs; update the normalization only for intended changes, never skip the gate.
