# evilo plugin template

A provider-neutral Kotlin/Wasm plugin for [evilo](https://github.com/crowforkotlin/evilo),
built on the Wasmline plugin protocol. It ships a self-contained demonstration
novel that reads and writes host-owned key/value storage; it makes no network
requests and is not a real content provider.

Remote: `https://github.com/crowforkotlin/evilo-plugin-templatea`.
The local directory name does not matter.

## Layout

| Path | Contents |
| --- | --- |
| `evilo-plugin/` | WasmlineService adapter, WASI entrypoint, `plugin.json`, icons |
| `evilo-plugin-core/` | Portable content, request construction, parsing |
| `evilo-plugin-build-logic/base/` | JVM/WASI library convention |
| `evilo-plugin-build-logic/plugin/` | Descriptor, signing, source ZIP, catalog export |
| `gradle/` | Version catalog and wrapper |

Build conventions live in an included build, not `buildSrc`. The core does not
depend on the guest or its generated fields.

## Rename for your provider

Fork as `evilo-plugin-<provider>`, then change these together:

| Location | Value |
| --- | --- |
| `settings.gradle.kts` | `rootProject.name`, `include(...)` module names |
| `evilo-plugin/build.gradle.kts` | `projects.eviloPluginCore` accessor |
| Kotlin packages | `crow.evilo.plugin.<provider>` or your reverse-DNS name |
| `evilo-plugin/plugin.json` | `descriptor.id`, `name`, `catalog` fields |
| `evilo-plugin/icon.png`, `icon.svg` | Replace both placeholders |

Implement the provider in your fork. Do not add provider code to this template or
to the app.

## SDK

Modules consume `io.github.crowforkotlin.evilo:evilo-library:1.0.0`, which holds both
the portable media models and the plugin/host protocol. The SDK has its own repository,
`evilo-library`; never copy protocol or model source here. Use
`-Pevilo.sdkPath=../evilo-library` to substitute an unpublished SDK build.

`EviloPluginService` exposes synchronous `describe`, `start`, and `resume` JSON
envelopes. `EviloHostService` exposes capabilities and private plugin storage.
The host performs asynchronous, permission-checked HTTP; the plugin builds
requests and parses responses with explicit continuation state.

Bind services in the WASI entrypoint and defer host lookup until an operation.
`describe()` performs no host calls. Service methods cannot suspend, overload,
expose properties, or use generics, default arguments, varargs, or service
parameters. Unknown required capabilities are rejected; optional ones need a
defined fallback.

## plugin.json

`descriptor` owns the signed identity: version, version code, media kinds,
languages, protocol, and API/media/storage permissions. `catalog` owns authors,
introduction, and optional region, NSFW, and creation date. Authors may be
strings or objects with `name` and optional `url`. `createdAt` accepts
`YYYY-MM-DD` or a full timestamp, and may be omitted. The descriptor must fit
Wasmline's 16 KiB metadata-string limit.

## Signing

Each publisher owns one Ed25519 seed and reuses it for every release. Generate it
once locally, back it up privately, and never commit it, put it in a source or
package archive, or print it in logs. The workflow derives the public key from
the seed; it never creates a new key.

| Repository setting | Purpose |
| --- | --- |
| Secret `EVILO_PRIVATE_KEY_HEX` | Preserved Ed25519 seed, 64 hex characters. Required. |
| Variable `EVILO_KEY_ID` | Overrides the default `publisher-<fingerprint>` label. |
| Variable `PLUGIN_MODULE` | Selects the guest module when autodiscovery is ambiguous. |

A key ID is a label, not proof of trust. A catalog may declare its publisher's
public key, but declaring it does not trust it: the app asks for approval of the
exact version, public-key fingerprint, and permissions, and pins the approved
manifest digest. Wasmline signature verification is always required. Users never
enter key IDs, public keys, or seeds. A valid signature does not prove that code
is harmless.

## Storage and networking

Storage requires the explicit `storage` permission and is isolated by publisher
identity and signed plugin ID, so a different publisher reusing the ID cannot
read another plugin's values. Limits: 1 MiB total, 256 entries, 64 KiB per value,
128-character keys. Plugins own their schema migrations. This is not an encrypted
credential vault or a file API.

Declare exact origins, paths, methods, and headers. Wildcards, cookies, ambient
credentials, and redirects are rejected. Signing does not authorize undeclared
network access. Browsers still apply CORS and mixed-content rules; no proxy is
supplied. The runtime exposes no guest fuel or linear-memory budget.

## Releases

`.github/workflows/plugin-release.yml` runs on GitHub-hosted `ubuntu-24.04` for
`v*` source tags and for manual dispatch naming an existing tag. `vX.Y.Z` must
equal `descriptor.version`.

Step-by-step operator procedure — key setup, version selection, verification,
tagging, publication checks, asset recovery, rotation, and failure modes — is in
[docs/publishing.md](docs/publishing.md). See
[.github/README.md](.github/README.md) for the job contract and helper scripts.

Never overwrite existing package files, reuse a version for changed source, or
replace a published source hash. Keep `main/plugins.json` as the subscription
address; Release pages and ZIP URLs are not catalogs.

## Manual export

Commit the implementation first. Supply its full commit, the source repository,
and directly served HTTPS release URLs:

```sh
./gradlew :evilo-plugin:exportPluginRelease \
  -Pevilo.signingKeyDirectory="$PUBLISHER_SIGNING_DIRECTORY" \
  -Pevilo.sdkPath=../evilo-library \
  -Pplugin.sourceRevision="$SOURCE_COMMIT" \
  -Pplugin.releaseBaseUrl="$RELEASE_BASE_URL" \
  -Pplugin.sourceUrl="$SOURCE_REPOSITORY_URL" \
  -Pplugin.publishedAt="$PUBLISHED_AT" \
  -Pwasmline.autoDownload=true
```

`PUBLISHED_AT` accepts `YYYY-MM-DD` (midnight UTC) or a full ISO-8601 timestamp
and is never read from the clock. `plugin.sourceSnapshotUrl` is optional; omit it
to inherit `plugin.sourceUrl`. Export writes `manifest.wlm`, content-addressed
artifacts, the source ZIP and its hash, `icon.png`, and `plugins.json` under
`evilo-plugin/build/release/`.

The source ZIP contains both modules, build logic, specs, wrapper, docs, icons,
and license, and excludes build caches, Git, keys, and `local.properties`. Signing
a source hash does not prove reproducible builds or code safety.

## Catalog format

The exported root contains only `name` and `plugins`. Kinds, languages, protocol,
permissions, and the signing identity appear once per plugin; each release
carries version, version code, manifest URL and SHA-256, source commit/archive/hash,
publication date, and an optional source snapshot. Current version and update
dates are derived by the app. There is no review field or external schema.

Compact catalogs require an app with `CatalogParser` normalization; that parser
also accepts the previous full format. Host the exported `plugins.json` at any
direct HTTP(S) URL and share that address. No pull request against evilo's bundled
source is required. Keep the subscription URL stable and retain older release
entries; give a release explicit overrides when its permissions, protocol, kinds,
or languages differ from the plugin defaults.

## Tests

Build-logic tests cover catalog export, common-field precedence, optional source
snapshot and publication inputs, and invalid dates. They use local fixtures and
read no signing keys.

```sh
./gradlew -p evilo-plugin-build-logic :plugin:test
```
