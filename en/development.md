# Development, license and feedback

## Source and documentation

- Main repository: <https://github.com/hyhdsl/neca>
- Install, upgrade and rollback: <https://neca.nekobiglazyer.work/en/#/install>
- Internal development documents and assessment records are kept locally by the
  maintainer and are not published in the repository.

## Local build

Development runs in Docker:

~~~bash
docker build -f Dockerfile.dev -t neca-dev .
docker compose run -T --rm dev make all
docker compose run -T --rm dev make shadow
~~~

Common commands:

| Command | Purpose |
|---|---|
| `make all` | gofmt + vet + ASCII + tests + cross-compile |
| `make vuln` | govulncheck dependency vulnerability scan |
| `make shadow` | shadow comparison between the old bash implementation and neca |
| `make check-locales` | zh-CN/zh-TW/en-US locale key consistency |
| `make docker` | build the lite and neca-web images |

## Architecture

- `internal/core`: business core, independent of the presentation layer;
- `internal/cli`: argument parsing, subcommands and output;
- `internal/web`: Web server, sessions, templates and API;
- `internal/helper`: root helper protocol and allowlist;
- `cmd/neca`, `cmd/necad`, `cmd/neca-web-helper`: the three entry points;
- `internal/core/ports`: replaceable interfaces (index and configuration sources).

lite and neca-web share the same core and data directory; a full image is not
published.

## Release process

1. Merge into `main` and pass CI.
2. Create a `vX.Y.Z` tag.
3. `release.yml` builds binaries and checksums, then signs them with the dedicated release key.
4. `docker.yml` pushes GHCR images.
5. Upgrade the server with a pinned tag and retain rollback images.

Preview tags (`vX.Y.Z-preview.N`) are not stable releases; old previews retain only
the source archive.

## Multilingual documentation

Simplified Chinese, Traditional Chinese and English help pages cover the same topics and are maintained manually. The Simplified Chinese documentation is authoritative for behavior and release notes. The language switch is in the page header and sidebar.

## Contributing and feedback

- Describe issues or proposed changes at <https://github.com/hyhdsl/neca/issues> first.
- PRs should include tests and license-impact notes for maintainer review.
- See [GitHub workflows](github-workflows.md) and [shadow comparison](shadow-comparison.md) for CI and release internals.
- For changes, fork the repository and follow AGPL and the
  [secondary-development requirements](https://github.com/hyhdsl/neca/blob/main/docs/SECONDARY-DEVELOPMENT.md).
- For security issues, do not paste private keys, passwords or exploitable details in
  public; contact the maintainer first.

## License

**AGPL-3.0-only**. Full text:

<https://github.com/hyhdsl/neca/blob/main/LICENSE>

The project also includes AI disclosure and disclaimers; see `NOTICE` and the README.
