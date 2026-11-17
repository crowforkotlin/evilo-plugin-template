# Publishing a signed release

Operator runbook for this repository. `.github/workflows/plugin-release.yml`
signs and publishes on GitHub-hosted `ubuntu-24.04`; this file is the supported
path through it. See [.github/README.md](../.github/README.md) for the job
contract and helper scripts.

One publisher, one plugin ID, `main` as the default branch.

## 1. Keys

The workflow needs one Secret and, for repositories that verify against the app,
one Variable. Both are per-repository: a key configured on another repository is
not visible here.

| Setting | Kind | Required | Purpose |
| --- | --- | --- | --- |
| `EVILO_PRIVATE_KEY_HEX` | Secret | yes | Preserved Ed25519 seed, 64 lowercase hex characters. Signs the manifest. |
| `EVILO_KEY_ID` | Variable | no | Label override; defaults to `publisher-<fingerprint>`. |
| `PLUGIN_MODULE` | Variable | no | Only when more than one `*/plugin.json` exists. |

This template's workflow reads only `secrets.EVILO_PRIVATE_KEY_HEX` and
`vars.EVILO_KEY_ID`; it never reads a public key, so `EVILO_PUBLIC_KEY_HEX` is
not required here. Your app build needs the derived public key instead — see
evilo's `docs/plugin-key-setup.md`.

If your fork also copies `verify_app_key.py` and `identity-check.yml` from a
repository that ships them, that pair additionally needs `EVILO_PUBLIC_KEY_HEX`
and fails the export when the Secret derives a different key. Pick one kind and
reference it consistently: `vars.EVILO_PUBLIC_KEY_HEX` and
`secrets.EVILO_PUBLIC_KEY_HEX` are different settings.

A public verification key is not secret material in the end: the exported
`plugins.json` carries `availableVersions[].signing.publicKeyHex` in clear text,
because the app must read it to verify a signature. A Secret hides it from the
settings page and from anonymous API callers, not from anyone who fetches your
catalog.

### Creating the key pair

Generate once, back up the seed privately, never commit it.

```sh
mkdir -p ~/.evilo/signing && chmod 700 ~/.evilo/signing
openssl genpkey -algorithm ED25519 -out ~/.evilo/signing/private.pem
openssl pkey -in ~/.evilo/signing/private.pem -outform DER | tail -c 32 \
  | od -An -tx1 -v | tr -d ' \n' > ~/.evilo/signing/private.key
openssl pkey -in ~/.evilo/signing/private.pem -pubout -outform DER | tail -c 32 \
  | od -An -tx1 -v | tr -d ' \n' > ~/.evilo/signing/public.key
printf 'keyId=publisher-%s\npublicKeyHex=%s\n' \
  "$(openssl pkey -in ~/.evilo/signing/private.pem -pubout -outform DER | tail -c 32 | sha256sum | cut -d' ' -f1)" \
  "$(cat ~/.evilo/signing/public.key)" > ~/.evilo/signing/authority.properties
rm ~/.evilo/signing/private.pem
```

`checkPluginAuthority` requires `private.key`, `public.key`, and
`authority.properties` in one directory outside the checkout, with the metadata
matching the key, so keep all three.

Upload them:

```sh
gh secret set EVILO_PRIVATE_KEY_HEX --repo OWNER/REPO < ~/.evilo/signing/private.key
```

Give the public half to the app build (evilo's `docs/plugin-key-setup.md`). If
your fork ships `identity-check.yml`, run it once to compare the Secret with the
app key; it builds and signs nothing:

```sh
gh workflow run identity-check.yml --repo OWNER/REPO --ref main
gh run list --repo OWNER/REPO -w identity-check.yml -L 1
```

## 2. Choosing the next version

Edit the guest `*/plugin.json` only. `descriptor.version` must be a new semver,
`descriptor.versionCode` a strictly larger integer than every published entry,
and both must match the source you are shipping. Changed code, permissions,
hosts, or headers always need a new version: never re-tag, never re-export,
never reuse a version to change its bytes.

```sh
python3 - <<'PY'
import json, pathlib
spec = json.load(open('evilo-plugin/plugin.json'))['descriptor']
print('building :', spec['version'], spec['versionCode'])
path = pathlib.Path('plugins.json')
if path.exists():
    plugin, = json.loads(path.read_text())['plugins']
    published = [(v['version'], v['versionCode']) for v in plugin['availableVersions']]
    print('published:', published)
    assert all(int(code) < spec['versionCode'] for _, code in published)
    assert all(version != spec['version'] for version, _ in published)
else:
    print('published: none (first release)')
PY
```

## 3. Verify locally

```sh
./gradlew :evilo-plugin-core:jvmTest
./gradlew -p evilo-plugin-build-logic :plugin:test

VERSION=1.0.1            # must equal descriptor.version
REPOSITORY=OWNER/REPO
./gradlew :evilo-plugin:exportPluginRelease \
  -Pevilo.signingKeyDirectory="$HOME/.evilo/signing" \
  -Pplugin.sourceRevision="$(git rev-parse HEAD)" \
  -Pplugin.releaseBaseUrl="https://raw.githubusercontent.com/$REPOSITORY/package-v$VERSION/releases/$VERSION" \
  -Pplugin.sourceUrl="https://github.com/$REPOSITORY" \
  -Pplugin.publishedAt="$(date -u +%Y-%m-%d)" \
  -Pwasmline.autoDownload=true
```

Add `-Pevilo.sdkPath=../evilo` when the SDK is unpublished. A successful export
proves the descriptor, signing identity, source ZIP, and catalog export. Delete
`*/build/release/` afterwards so a local export cannot be mistaken for published
bytes.

## 4. Commit, tag, push

The workflow checks out the tag with `fetch-depth: 0`, requires a clean tree, and
stops unless the tag resolves to exactly that commit.

```sh
git switch main
git add -A
git commit -m "feat(provider): describe the change"
git push origin main
git tag v1.0.1          # must equal descriptor.version
git push origin v1.0.1  # triggers plugin-release.yml
```

## 5. What the workflow does

1. `package` (read-only): resolves module, version, plugin ID, and revision from
   the tagged `plugin.json`, restores the Secret into `$RUNNER_TEMP`, runs
   `:exportPluginRelease`, copies only public files, deletes the key material.
2. `publish` (write): refuses if `package-vX.Y.Z` exists, switches to the default
   branch, copies the exported files to `releases/X.Y.Z/`, appends the release to
   `plugins.json` while older entries keep their effective
   permissions/protocol/kinds/languages, commits
   `release(plugins): publish signed version X.Y.Z`, tags `package-vX.Y.Z`, and
   pushes branch and tag atomically.
3. Release assets: `restore_release.py` creates a draft Release, uploads the
   package ZIP, exact source ZIP, `manifest.wlm`, `plugins.json`, and `icon.png`,
   verifies sizes and SHA-256 digests, then publishes it.

The catalog push happens before asset upload, so an upload failure leaves the
catalog valid and is repaired with `republish=true`, never with a new version.

## 6. Verify the publication

```sh
gh run list --repo "$REPOSITORY" -w plugin-release.yml -L 3
gh release view v1.0.1 --repo "$REPOSITORY"

curl -fsSL "https://raw.githubusercontent.com/$REPOSITORY/main/plugins.json" \
  | python3 -c "import json,sys; p=json.load(sys.stdin)['plugins'][0]; \
print([(v['version'], v['versionCode'], v.get('manifestSha256')) for v in p['availableVersions']])"

curl -fsSL "https://raw.githubusercontent.com/$REPOSITORY/package-v1.0.1/releases/1.0.1/manifest.wlm" \
  | sha256sum   # must equal that release's manifestSha256
```

Confirm older versions are still listed and that `git rev-parse v1.0.1` is the
commit you intended. Never delete or move a tag.

## 7. Recover assets without rebuilding

If the Release, an asset, or an upload step is missing but the tags exist:

```sh
gh workflow run plugin-release.yml --repo "$REPOSITORY" --ref main \
  -f source_tag=v1.0.1 -f republish=true
```

`republish` skips signing and packaging, reads no Secret, restores assets from
immutable `package-vX.Y.Z` files (falling back to `vX.Y.Z` for legacy packages),
uploads only missing assets, rejects different existing bytes, and leaves the
catalog untouched.

## Rotating the key

Rotation changes the identity the app trusts. Every installed user must install
a new app build carrying the new public key; already-installed plugin releases
signed by the old key fail verification in the new app, and new releases fail in
the old app. Treat it as a breaking app release, not a plugin release.

1. Generate a new pair as above into a fresh directory; keep the old one.
2. Put the new public key in the app build (evilo's `docs/plugin-key-setup.md`)
   and rebuild the APK. The app embeds only the public key.
3. Replace `EVILO_PRIVATE_KEY_HEX` on this repository with the new seed.
4. If your fork ships `identity-check.yml`, run it; it fails until the Secret and
   the app key are the same pair.
5. Publish the plugin under a **new version and a larger `versionCode`** — the app
   pins the approved manifest digest per version, so reusing a version for a
   differently signed manifest is rejected.
6. Retain the old seed until no supported app build trusts it. Releases signed by
   the old key stop verifying once the app ships the new one; releases signed by
   the new key fail in the old app.

While the app is in development, rotating is safe and cheap: rebuild the APK with
the new public key and reinstall, then publish the plugin release. No migration
is needed because nothing is in production.

## Failure modes

| Symptom | Cause |
| --- | --- |
| `Set PLUGIN_MODULE to the guest module name.` | More than one `*/plugin.json`; set the variable. |
| `Source tag must match descriptor.version` | Tag and `descriptor.version` disagree. |
| `Commit all released source before packaging.` | Dirty checkout at the tagged commit. |
| `Never reuse published versions; versionCode must increase.` | Version or `versionCode` not advanced. |
| `Package tag already exists; never overwrite it.` | That version is already published; bump instead. |
| `Package does not match its source tag.` | Tag points at a different commit than the export. |
| `Configure repository variable EVILO_PUBLIC_KEY_HEX …` | Only when the fork ships `verify_app_key.py`; variable absent or not 64 lowercase hex characters. |
| `The Secret derives a different public key than the application trusts.` | Secret and app key disagree; do not tag. Only reachable when the fork ships `verify_app_key.py`. |
| `Official packages must use key ID evilo-root-v1` | Only when the fork ships `verify_app_key.py`; `EVILO_KEY_ID` overridden away from that label. |
| `Missing publisher key` / `Invalid authority key format` | Secret absent, malformed, or restored inside the checkout. |
| `Could not resolve io.github.crowforkotlin.evilo:evilo-library` | The pinned SDK version is not on Maven Central yet; see evilo's `docs/library-publishing.md`. |
| `Existing asset X differs; never overwrite it.` | Committed bytes changed after publication. |

A workflow file is configuration, not evidence that a run signed or published
anything.
