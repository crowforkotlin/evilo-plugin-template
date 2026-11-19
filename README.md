# evilo plugin template

A Kotlin/Wasm plugin repository for the evilo app, built on the Wasmline plugin
protocol. It ships a self-contained demonstration novel that reads and writes
host storage; it makes no network requests and is not a real content provider.

Use it as the starting point for a provider repository.

## Layout

| Path | Contents |
| --- | --- |
| `evilo-plugin/` | `plugin-spec.json`, WASI entrypoint, and the `EviloPluginService` adapter |
| `evilo-plugin-core/` | Portable request construction and parsing; no guest dependency |
| `evilo-plugin-build-logic/base/` | JVM/WASI library convention |
| `evilo-plugin-build-logic/plugin/` | Descriptor, signing, source ZIP, and catalog export |
| `.github/` | Release and identity workflows plus their scripts |
| `gradle/` | Version catalog and wrapper |

Build conventions live in an included build rather than `buildSrc`.

## Rename this repository

Change these together when you fork:

| Location | Value |
| --- | --- |
| `settings.gradle.kts` | `rootProject.name` and the `include(...)` module names |
| `evilo-plugin/build.gradle.kts` | The `projects.eviloPluginCore` accessor |
| Kotlin packages | `crow.evilo.plugin.<provider>` or your own reverse-DNS name |
| `evilo-plugin/plugin-spec.json` | `descriptor.id`, `descriptor.name`, and the `catalog` fields |
| `evilo-plugin/icon.png`, `icon.svg` | Replace both placeholders |

Provider code belongs in your fork only.

## SDK

Modules consume `io.github.crowforkotlin.evilo:evilo-library`, which holds the
portable media models and the plugin/host protocol. Its version is pinned in
`gradle/libs.versions.toml` as `evilo-plugin-sdk`. `-Pevilo.useLocalSdk`
substitutes an unpublished local build for development.

`EviloPluginService` exposes synchronous `describe`, `start`, and `resume` JSON
envelopes, with continuation state held explicitly by the plugin.
`EviloHostService` exposes host capabilities and private plugin storage. The host
performs HTTP on the plugin's behalf; the plugin builds requests and parses
responses.

Bind services in the WASI entrypoint and defer host lookup until an operation.
`describe()` performs no host calls. Service methods cannot suspend, overload,
expose properties, or use generics, default arguments, varargs, or service
parameters. Unknown required capabilities are rejected at load; optional ones
need a defined fallback.

## plugin-spec.json

`descriptor` holds the signed identity: version, version code, media kinds, and
languages. `catalog` holds authors, the introduction, and optional region, NSFW,
and creation date. Authors may be strings or objects with `name` and an optional
`url`. `createdAt` accepts `YYYY-MM-DD` or a full timestamp and may be omitted.
The descriptor must fit Wasmline's 16 KiB metadata-string limit.

There is no permission block. The host forwards whatever method, path, headers,
and body the plugin asks for, subject to the request limits below; the server
decides whether to accept it. Installing a plugin is the authorization decision.

## Signing

Each publisher owns one Ed25519 seed and reuses it for every release. Generate it
once locally, back it up privately, and never commit it, put it in a source or
package archive, or print it in logs. The workflow derives the public key from
the seed and never creates a new key.

| Repository setting | Purpose |
| --- | --- |
| Secret `EVILO_PRIVATE_KEY_HEX` | Preserved Ed25519 seed, 64 hex characters. Required. |
| Variable `EVILO_KEY_ID` | Overrides the default `publisher-<fingerprint>` label. |
| Variable `PLUGIN_MODULE` | Selects the guest module when autodiscovery is ambiguous. |

A key ID is a label, not proof of trust. The app verifies the manifest signature
against the key embedded in the manifest, then pins the approved manifest digest
for that release. A valid signature does not prove that code is harmless.

## Storage and networking

Storage is isolated by publisher identity and signed plugin ID, so a different
publisher reusing an ID cannot read another plugin's values. Limits: 1 MiB total,
256 entries, 64 KiB per value, 128-character keys. Plugins own their schema
migrations. This is not an encrypted credential vault or a file API.

The network is reachable through the host, which enforces:
- absolute HTTP(S) URLs only, with cleartext allowed only when the subscribing
  catalog was added with that consent;
- no URL credentials and no path traversal;
- at most 32 request headers of at most 4096 characters, with cookies, `Host`,
  `Connection`, `Content-Length`, `Origin`, and forwarding headers rejected;
- request bodies of at most 64 KiB and responses bounded per call.

Redirects are not followed. Browsers still apply CORS and mixed-content rules;
no proxy is supplied. The runtime exposes no guest fuel or linear-memory budget.

## Releases

`.github/workflows/plugin-release.yml` runs on GitHub-hosted `ubuntu-24.04` for
`v*` source tags and for manual dispatch naming an existing tag. `vX.Y.Z` must
equal `descriptor.version`.

The step-by-step operator procedure is in [docs/publishing.md](docs/publishing.md);
the job contract and helper scripts are in [.github/README.md](.github/README.md).

Never overwrite existing package files, reuse a version for changed source, or
replace a published source hash.

The catalog is one document every subscriber downloads, so publishing keeps it
under 10 MiB. When appending a release would exceed that, `merge_catalog.py` drops
the oldest releases and reports how many. A release at or above the plugin's
`minimumUsableVersionCode` is never dropped: it is a promise to subscribers. If
the promises alone exceed the limit, the publish fails rather than break one.
Override the limit only when the app's `MAX_CATALOG_BYTES` changes too.

## Manual export

Commit the implementation first, then supply its full commit, the source
repository, and directly served HTTPS release URLs:

```sh
./gradlew :evilo-plugin:exportPluginRelease \
  -Pevilo.signingKeyDirectory="$PUBLISHER_SIGNING_DIRECTORY" \
  -Pevilo.useLocalSdk \
  -Pplugin.sourceRevision="$SOURCE_COMMIT" \
  -Pplugin.releaseBaseUrl="$RELEASE_BASE_URL" \
  -Pplugin.sourceUrl="$SOURCE_REPOSITORY_URL" \
  -Pplugin.publishedAt="$PUBLISHED_AT" \
  -Pwasmline.autoDownload=true
```

`PUBLISHED_AT` accepts `YYYY-MM-DD` (midnight UTC) or a full ISO-8601 timestamp
and is never read from the clock. `plugin.sourceSnapshotUrl` is optional; omit it
to inherit `plugin.sourceUrl`. Export writes `manifest.wlm`, content-addressed
artifacts, the source ZIP and its hash, `icon.png`, and `catalog.json` under
`evilo-plugin/build/release/`.

The source ZIP contains both modules, build logic, specs, wrapper, docs, icons,
and license, and excludes build caches, Git, keys, and `local.properties`. Signing
a source hash does not prove reproducible builds or code safety.

## Catalog files

| Path | Written by | Role |
| --- | --- | --- |
| `evilo-plugin/plugin-spec.json` | you | The only hand-edited file: `descriptor` and `catalog`. No URLs, hashes, or dates beyond `createdAt`. |
| `releases/<version>/catalog.json` | the release workflow | Immutable snapshot of one published version, frozen into the `package-v<version>` tag. |
| `catalog.json` at the repository root | the release workflow | The subscription catalog. Every release appends to it. |

`exportPluginRelease` computes the manifest and source-archive SHA-256 values,
derives artifact URLs from `-Pplugin.releaseBaseUrl`, and writes the catalog. Do
not hand-edit anything under `releases/` or the root `catalog.json`: the recovery
path checks the recorded hashes and refuses different bytes.

The exported root contains only `name` and `plugins`. Kinds, languages, and the
signing identity appear once per plugin; each release carries version, version
code, manifest URL and SHA-256, source commit/archive/hash, and publication date.
Current version and update dates are derived by the app. There is no review field
and no external schema.

Host the root `catalog.json` at any direct HTTP(S) URL and share that address.
Keep the subscription URL stable. Older release entries are dropped automatically
once the document reaches its byte budget; set `minimumUsableVersionCode` if a
version must stay available. Give a release explicit overrides when its kinds or
languages differ from the plugin defaults.

## Tests

Build-logic tests cover catalog export, common-field precedence, optional source
snapshot and publication inputs, and invalid dates. They use local fixtures and
read no signing keys.

```sh
./gradlew -p evilo-plugin-build-logic :plugin:test
```
