# evilo plugin template

A provider-neutral, multi-module Kotlin/Wasm template. Its remote is
`https://github.com/crowforkotlin/evilo-plugin-templatea`; the local directory
may remain `evilo-plugin-template`. No real content-provider implementation
belongs here. The bundled text/storage example is labeled demonstration content.

## Structure and naming

```text
evilo-plugin-core/                 Portable content, requests and parsing
evilo-plugin/                      WasmlineService adapter and WASI entrypoint
  plugin.json                      Release identity, permissions, catalog metadata
  icon.png                         Published icon; replace the placeholder
  icon.svg                         Editable icon source
evilo-plugin-build-logic/
  base/                            JVM/WASI library convention
  plugin/                          Descriptor, signing, source ZIP, catalog export
gradle/                            Version catalog and Gradle wrapper
```

The root is `evilo-plugin-template-root`. Explicit settings includes and prefixed
modules follow Wasmline's layout. Build conventions live in an included build,
not `buildSrc`. The core does not depend on the guest or its generated fields;
the guest passes the release descriptor to the portable implementation.

Fork this repository as `evilo-plugin-<provider>`. Name its guest module
`evilo-plugin-<provider>`, core `evilo-plugin-<provider>-core`, and root
`evilo-plugin-<provider>-root`. Update settings includes, type-safe dependencies,
packages, and the descriptor together. Use `crow.evilo.plugin.<provider>` or your
own reverse-DNS package. Implement the provider in that fork, not in this template
or the app. Storage is scoped to the publisher identity and plugin ID, so one
publisher's plugin cannot access another publisher's private key-value data.

## One maintained SDK

Modules consume `io.github.crowforkotlin.evilo:evilo-library:1.0.0`, which contains
both portable media models and the plugin/host protocol. This SDK is maintained
once in evilo's `evilo-library`. Never copy protocol/model source here.
Before SDK publication, use `-Pevilo.sdkPath=../evilo` to substitute the actual
SDK build. A third protocol repository is not required.

`EviloPluginService` exposes synchronous `describe`, `start`, and `resume` JSON
envelopes. `EviloHostService` exposes capabilities and private plugin storage.
The host performs asynchronous, permission-checked HTTP; the plugin constructs
requests and parses responses using explicit continuation state.

Bind services in the WASI entrypoint, but defer host lookup until an operation.
`describe()` has no network/storage side effects. Service methods cannot suspend,
overload, expose properties, or use generics, default arguments, varargs, or
service parameters. Unknown required capabilities are rejected; optional ones
need a defined fallback. SDK, protocol, plugin, and app versions are distinct.

Edit `evilo-plugin/plugin.json`: `descriptor` owns signed identity, version,
media kinds, languages, protocol, and API/media/storage permissions; `catalog`
owns authors, introduction, and optional region, NSFW, creation date, and update
policy. Authors may be strings or objects with `name` and optional `url`.
`createdAt` may be `YYYY-MM-DD` or a full timestamp and may be omitted.
Generated guest fields and signed metadata still use the unchanged descriptor
object. It must fit Wasmline's 16 KiB metadata-string limit.

## Publisher signing identities

Each publisher owns and preserves its own Ed25519 signing key. An independent
author generates a private seed **once locally**, backs it up privately, and
reuses it for future releases. Do not request or distribute the official
publisher's private key. Never commit keys, put them in a source/package archive,
or print them in logs.

For GitHub Actions, store that preserved seed in the repository Secret
`EVILO_PRIVATE_KEY_HEX`. It is the template's only required Secret. The workflow
derives the public key from the seed; it never generates a fresh release key.
The default `keyId` is `publisher-<fingerprint>`, where the fingerprint is derived
from SHA-256 of the public key bytes. The optional repository variable
`EVILO_KEY_ID` overrides that identifier. A key ID is a label, not proof of trust.

The official CopyManga publisher uses its own repository's already configured
`EVILO_PRIVATE_KEY_HEX` Secret and defaults to the existing official key ID
`evilo-root-v1`. The app receives only the matching official public key through
`EVILO_PUBLIC_KEY_HEX`; it does not receive a publisher's private seed.

A third-party catalog can declare its publisher's public key, but that declaration
does not automatically trust the publisher. Installation requires approval of the
specific version, public-key fingerprint, and permissions. Wasmline signature
verification remains mandatory, and the approved manifest digest is pinned so
changed bytes cannot silently reuse that approval. A different publisher cannot
replace an installed plugin by reusing its ID. Users never enter raw key fields.
A valid signature does not prove that code is harmless.

Manual export remains available with the preserved publisher key in an external
signing directory. Pass `-Pevilo.signingKeyDirectory=/external/directory` to use
that directory; keep its private key out of the checkout and exported archives.

## Storage and networking

Storage needs explicit permission and is isolated by publisher identity and
signed plugin ID. A different publisher with the same ID cannot access another
publisher's key-value storage. Default
limits are 1 MiB total, 256 entries, 64 KiB/value, and 128-character keys. Plugins
own schema migrations. Storage is not an encrypted credential vault or file API.

Declare exact origins, paths, methods, and headers. Wildcards, cookies, ambient
credentials, and redirects are rejected. Signing does not authorize undeclared
network access. Browsers still need CORS and obey mixed-content rules; no proxy
is supplied. The current runtime exposes no guest fuel or linear-memory budget.

## GitHub Actions release workflow

`.github/workflows/plugin-release.yml` is configured for `v*` source tags and
manual dispatch naming an existing tag. It runs on GitHub-hosted `ubuntu-24.04`;
no self-hosted runner or fresh signing-key generation is required. The tag
`vX.Y.Z` supplies the release version; do not reuse a tag for changed source.

Configuration:

- Required repository Secret: `EVILO_PRIVATE_KEY_HEX`, the publisher's preserved
  Ed25519 private seed in hexadecimal.
- Optional repository variable: `EVILO_KEY_ID`, overriding the default
  `publisher-<fingerprint>` signing identifier.
- `PLUGIN_MODULE` is autodiscovered from the guest `plugin.json`; set it only
  when the checkout needs an explicit module selection.
- Optional repository variables: `EVILO_SDK_REPOSITORY` and
  `EVILO_SDK_REVISION`. Set both to check out an unpublished SDK at a full
  immutable commit. That checkout stays outside the plugin source archive;
  otherwise the build uses the published SDK dependency.

The signing job has read-only repository permissions. It restores the Secret to
an external temporary signing directory, derives the matching public key and
key ID, exports the release, and cleans up restored private material. A separate
publish job has repository write permissions and receives public release files,
not the Secret or private signing directory.

The publish job creates immutable tag `package-vX.Y.Z` containing the raw release
files and appends the release to `main`'s root `plugins.json`. Catalog merging
preserves each historical release's effective metadata with explicit overrides
when shared defaults change. Existing package files, URLs, and source hashes must
never be overwritten. A GitHub Release provides a package ZIP and the exact
source ZIP; its public files include the manifest, icon, and versioned catalog.
The stable subscription URL points to `main/plugins.json`, not a GitHub Release
page or a ZIP.

Workflow configuration is not evidence that a run, build, signature check, or
publication succeeded. See [.github/README.md](.github/README.md) for the workflow
and helper-script contract. Manual export below remains supported.

If a GitHub Release is deleted or an asset upload fails after the package-tag
push, dispatch the current workflow from `main` with the existing `source_tag`
and `republish=true`. This mode restores public assets from the immutable package
tag (or the source tag for legacy releases) without building, signing, reading
the private-key Secret, rewriting tags, or changing the stable catalog. It
creates a draft, uploads and checks asset digests, then publishes it. Retries
upload only missing assets and reject different existing bytes.

## Manual release export

Commands here are instructions, not recorded results or authorization to run
builds, tests, packaging, servers, uploads, installation, or screenshots. Use the
pinned JDK/Kotlin/Wasmline versions and compatible WASI Node.js. An optional
`wasmline.localPath` composite may execute upstream build tasks.

Commit the actual implementation first. Supply its full immutable source commit,
source repository, and directly served HTTPS release URLs without redirects:

```sh
./gradlew :evilo-plugin:exportPluginRelease \
  -Pevilo.signingKeyDirectory="$PUBLISHER_SIGNING_DIRECTORY" \
  -Pevilo.sdkPath=../evilo \
  -Pplugin.sourceRevision="$SOURCE_COMMIT" \
  -Pplugin.releaseBaseUrl="$RELEASE_BASE_URL" \
  -Pplugin.sourceUrl="$SOURCE_REPOSITORY_URL" \
  -Pplugin.publishedAt="$PUBLISHED_AT" \
  -Pwasmline.autoDownload=true
```

`SOURCE_COMMIT` must contain this implementation, not just a license or binaries.
`PUBLISHED_AT` accepts `YYYY-MM-DD` (midnight UTC) or a full ISO-8601 timestamp,
and must not precede an explicitly supplied creation date. Publication dates are
always supplied, never taken from the current clock. `plugin.sourceSnapshotUrl`
is optional: pass an immutable snapshot URL to override the release's source
link, or omit it to inherit `plugin.sourceUrl`. The actual source commit and
archive/hash remain required. Downloads are opt-in. Export writes a version-specific directory under
`evilo-plugin/build/release/` containing `manifest.wlm`, content-addressed
artifacts, actual source ZIP/hash, PNG, and `plugins.json`.

The reproducible ZIP includes both modules, build logic, specs, wrapper, docs,
icons, and license; it excludes build caches, Git, keys, and local properties.
The operator reviews working source versus the linked commit. Signed source
hashes do not prove build reproducibility or code safety.

Replace placeholder PNG/SVG icons before publication. The app loads bounded
PNG/JPEG/WebP, not remote SVG; icons must fit 1 MiB and 512-pixel dimensions.
Upload all artifacts and the exact source ZIP. Never reuse a version code for
changed code or permissions, and never replace an existing release's source hash.

## Host an independent plugin source

The editable `evilo-plugin/plugin.json` is a build specification, not a
subscription catalog. Publish the exported `plugins.json` at a direct HTTP(S)
URL and share that address. No PR to evilo's bundled source is required, and the
catalog may be renamed or served from an endpoint without a `.json` suffix.

The exported root only contains `name` and `plugins`. Kinds, languages, protocol,
and permissions appear once per plugin; releases inherit them and carry only
version/code, manifest URL and SHA-256, Ed25519 signing identity, source commit/archive/hash,
publication date, and any explicit source snapshot link. Current version and update dates are derived by
the app. There is no review field, `$schema` property, or external schema file.

Compact catalogs require an evilo app with `CatalogParser` normalization. That
parser also accepts the previous full format; older apps without it need a full
catalog. For future updates, keep the subscription URL stable and retain older
release entries. If permissions, protocol, kinds, or languages differ between
versions, use explicit per-release overrides rather than changing old releases
through common defaults. The catalog can declare a publisher public key, but
hosting or subscribing to it does not automatically trust that key.

In evilo, add the JSON URL in Browse and choose a compatible release. For a
third-party publisher, review and approve the version, public-key fingerprint,
and permissions before installation. Wasmline verification and the approved
manifest digest still apply. End users do not enter WLM URLs, key IDs, or keys.
See evilo's `docs/plugin-catalog.md` for catalog and mandatory-update semantics.

## Manual catalog export tests

With the configured JDK, the build-logic tests exercise common-field export,
source metadata, dates, and legacy-field removal without loading signing keys
or contacting a live provider:

```sh
./gradlew -p evilo-plugin-build-logic :plugin:test
```

These are instructions, not execution records. Tests and builds require a
separate explicit request.
