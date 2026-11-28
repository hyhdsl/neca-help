# GPG verification

> Applies to neca v1.5.1 and later stable releases. There are two independent signature chains: Release asset `checksums.txt` and Git tags. They use different keys and cannot replace each other.

## 1. Know the two chains

| Object | Signing key | Fingerprint | Purpose |
|---|---|---|---|
| Release asset `checksums.txt` | neca release signing (2026) | `E514EB7C153AA24A3E4241F846CC03EE21F66795` | Verify all release binaries |
| Git tag | Existing personal neca key | `4DE8259A9C3E832137506D3C9398D6D10FD5A903` | Verify the tagged commit |

From v1.5.1 onward, release assets are signed by the dedicated Ed25519 release key. v1.5.0 and earlier may use the existing neca key. Git tags continue to use the existing personal key.

Do not rely on `Good signature` alone; always check the actual signer fingerprint.

## 2. Download verification material

For the current stable v1.6.0 (replace the tag for another release):

~~~bash
gh release download v1.6.0 --repo hyhdsl/neca \
  --pattern checksums.txt \
  --pattern checksums.txt.asc \
  --pattern neca-gpg-key.asc
~~~

The Release asset `neca-gpg-key.asc` and the repository file `keys/neca-release-signing.asc` are public-key entry points. Check the fingerprint through both channels when possible.

## 3. Import the public key and check the fingerprint

~~~bash
gpg --import neca-gpg-key.asc
gpg --fingerprint E514EB7C153AA24A3E4241F846CC03EE21F66795
~~~

Expected:

~~~text
E514 EB7C 153A A24A 3E42 41F8 46CC 03EE 21F6 6795
neca release signing (2026) <release@neca.invalid>
~~~

If the fingerprint differs, stop and do not run the downloaded binaries.

## 4. Verify release assets

~~~bash
gpg --verify checksums.txt.asc checksums.txt
sha256sum -c checksums.txt
~~~

All three conditions must hold:

1. `checksums.txt.asc` is a valid signature.
2. The signer fingerprint is `E514EB7C153AA24A3E4241F846CC03EE21F66795`.
3. `sha256sum -c` passes for every file.

## 5. Verify the Git tag

Git tags use the existing personal neca key, independent of release-asset signing:

~~~bash
git fetch --tags
git tag -v v1.6.0
~~~

Expected signer fingerprint:

~~~text
4DE8259A9C3E832137506D3C9398D6D10FD5A903
~~~

Verifying only release assets does not prove that the tag points to the same source commit.

## 6. Troubleshooting

### Good signature, different fingerprint

Stop. `Good signature` only means the signature matches some imported public key, not that it is the neca release key.

### Public key will not import

Confirm that you downloaded `neca-gpg-key.asc`, not the signature file. On Windows, Kleopatra or the GnuPG command line can import it; the `gpg` path may differ by installation.

### sha256sum fails

Delete the downloaded binary and download it again. Do not ignore or repair a failed checksum.

### Verifying older releases

v1.5.0 and earlier release assets may use the existing neca key. Follow that Release's key and notes. Historical releases are not re-signed.

## 7. Public-key sources

- Repository: `keys/neca-release-signing.asc`
- Release: `neca-gpg-key.asc`
- Fingerprint: `E514EB7C153AA24A3E4241F846CC03EE21F66795`

The public key has no CA chain; trust comes from cross-checking the repository, the Release and the fingerprint.

## Related

- [Install, upgrade and rollback](install.md)
- [Security, permissions and backups](security.md)
- [Troubleshooting](troubleshooting.md)
