# Release automation

`.github/workflows/plugin-release.yml` signs and publishes on GitHub-hosted
`ubuntu-24.04`. It runs for `v*` source tags and for manual dispatch naming an
existing tag. `vX.Y.Z` supplies the version and must equal `descriptor.version`;
`package-vX.Y.Z` holds the immutable exported files.

A workflow file is configuration, not evidence that a run signed, published, or
passed anything.

## Configuration

| Setting | Purpose |
| --- | --- |
| Secret `EVILO_PRIVATE_KEY_HEX` | Preserved Ed25519 seed, 64 hex characters. The only required Secret. |
| Variable `EVILO_KEY_ID` | Overrides the default `publisher-<fingerprint>` label, derived from SHA-256 of the public key bytes. |
| Variable `PLUGIN_MODULE` | Selects the guest module; otherwise autodiscovered from the sole `*/plugin-spec.json`. |

The seed is generated once locally and reused. The workflow derives the public
key from it and never creates a key. A key ID is a label, not proof of trust.

## Jobs

`package` has read-only repository permissions. It restores the Secret into an
external temporary directory, runs `:exportPluginRelease`, copies only public
files, and deletes the restored key material.

`publish` has write permissions and receives public files only:

1. Creates immutable `package-vX.Y.Z` with the signed `manifest.wlm`, every
   referenced artifact, the exact source ZIP and hash, the icon, and the
   versioned catalog.
2. Appends the release to `main`'s `catalog.json`, preserving older releases'
   effective metadata. Changed kinds, languages, or source URL become explicit
   per-release overrides on older entries.
3. Creates a GitHub Release with a package ZIP and the exact source ZIP.

Never overwrite package files, reuse a version for changed source, or replace a
published source hash. Keep `main/catalog.json` as the subscription address.

## Recover a Release

Dispatch from `main` with the existing `source_tag` and `republish=true`:

```sh
gh workflow run plugin-release.yml --ref main \
  -f source_tag=v1.0.0 -f republish=true
```

If the source tag is missing remotely, push the preserved local tag; never
recreate it at a new commit. This job skips signing and packaging, does not read
the Secret, reads public files from `package-vX.Y.Z` (falling back to `vX.Y.Z` for
legacy packages), uploads only missing assets, rejects different existing bytes,
and then publishes the draft.

## Helper scripts

| Script | Contract |
| --- | --- |
| `release_identity.py` | Reads the sole guest `plugin-spec.json`, requires a clean checkout at `v<version>`, writes module, version, plugin ID, revision, and derived URLs to `GITHUB_OUTPUT`. |
| `publisher_key.py DIR` | Restores the Secret and derives `private.key`, `public.key`, and `authority.properties` outside the checkout. Never generates keys. |
| `export_bundle.py SRC OUT` | Copies public files and verifies manifest, source-archive, and artifact hashes. |
| `publish_release.py BUNDLE` | Publishes immutable package files, appends the stable catalog, then delegates assets to `restore_release.py`. Retry only with `republish=true`. |
| `restore_release.py TAG` | Restores Release assets from immutable Git blobs. Builds nothing, signs nothing, executes no archived script, and leaves the catalog unchanged. |
| `merge_catalog.py OLD NEW OUT` | Appends an exported release, refusing version reuse and non-increasing version codes. |

## Installation trust

A catalog-declared publisher key is not trusted automatically. Approval covers
the exact version and public-key fingerprint; Wasmline verification stays
mandatory and the approved manifest digest is pinned. Different publishers
cannot replace an installed plugin by reusing its ID, and storage is scoped by
publisher identity and plugin ID. Users never enter seeds, key IDs, or raw keys.
A valid signature does not prove code safety.

New-version builds and public-asset recovery are separate operations.
