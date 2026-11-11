# Medot official plugin registry

This repository is the public index for official Medot plugin packages. It is a
data repository: the catalog and schema are served from `main`, and package
bytes are attached to GitHub Releases. Plugin source repositories may stay
private.

> **Current state:** the catalog is intentionally empty. No plugin packages are
> listed yet. An empty `plugins` array is a valid
> catalog meaning "no official packages", not an error.

## Files

| File | Purpose |
| --- | --- |
| `catalog.json` | Machine-readable index. Served from `https://raw.githubusercontent.com/Little-Princess-Studio/medot-official-plugin-registry/main/catalog.json`. |
| `catalog.schema.json` | JSON Schema (draft 2020-12) for catalog schema 1. |
| `README.md` | This document. |

These files are copied from the Medot editor repository's `registry/` staging
directory. To seed the public repository, copy `catalog.json`,
`catalog.schema.json`, and this README to its root; no editor source, build
configuration, or `package.json` is needed. Do not edit generated catalog hashes
by hand.

## Catalog shape

```json
{
  "schemaVersion": 1,
  "plugins": []
}
```

Each plugin entry contains `pluginId`, `name`, `description`, a strict numeric
`version` (`x.y.z`), `sdkVersion`, mandatory `dependencies`, and one artifact per
target:

```json
{
  "target": "windows-x86_64",
  "url": "https://github.com/Little-Princess-Studio/medot-official-plugin-registry/releases/download/v1.2.3/example-plugin-windows-x86_64.medot-template",
  "size": 123456,
  "sha256": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
}
```

`size` and `sha256` describe the `.medot-template` file bytes. `target` is
`any` for a frontend-only package, or one of `windows-x86_64`,
`windows-aarch64`, `macos-x86_64`, `macos-aarch64`, `linux-x86_64`,
`linux-aarch64`.

The schema fixes the official download repository and target set. Cross-field
rules (unique plugin IDs and targets, dependency existence, acyclic dependency
graph) are enforced by the generator.

## Editor consumption

The Medot editor consumes this catalog from the fixed URL above. **Manage
Plugins** has a Local plugins view and an Official plugins view. The official
list is fetched only when the user clicks **Fetch official list**; there is no
background polling. Install and update use the native
`get_official_plugin_catalog` and `install_official_plugin` commands. The native
side re-fetches and validates the catalog, downloads the artifact for the
current target (or `any`), verifies size and SHA-256, validates the package
descriptor, and keeps the previous package for rollback. Downgrades and
same-version reinstalls are rejected.

## Publishing flow

The authoritative authoring instructions live in the Medot editor repository at
`docs/official-plugin-registry.md`. The short version:

1. Build one `.medot-template` artifact per target in the editor checkout.
2. Create a release in this repository with the final tag.
3. Upload the artifacts first. Uploaded asset names must match the local
   basenames used by the generator.
4. Verify the release-asset URLs resolve.
5. Generate/merge the catalog with the editor's dependency-free Node CLI:

   ```bash
   node scripts/plugins/catalog.mjs \
     --catalog registry/catalog.json \
     --tag v1.2.3 \
     --name "Example Plugin" \
     --description "Plugin description." \
     dist/example-plugin-windows-x86_64.medot-template
   ```

6. Review the diff, copy `catalog.json` to this repository's root, and commit.

Upload before updating the catalog: a catalog entry must never point at a
missing asset.

## Install order

Dependencies are plugin IDs. Publish and install dependencies first, then their
consumers. The runtime refuses installation when a dependency is missing and
refuses removal while installed consumers remain; the Official plugins view also
disables install/update until dependencies are available. The generator rejects
a catalog entry whose dependency is not already present.

## Trust and limits

- Hashes prove artifact integrity, not publisher identity. Schema 1 has no
  signatures or attestations. Trust this repository, its maintainers, HTTPS, and
  release access controls.
- Installed plugins execute with the editor's privileges; there is no JavaScript
  sandbox. Install only trusted packages.
- The current editor accepts template SDK 1 only. Native packages must match the
  user's OS/architecture.
- Catalog schema 1 has no mirror or alternate-download field. Changing the
  repository or target set requires a schema version change.
- Release assets can be replaced by maintainers without changing the URL. The
  catalog hash is the pin; changing bytes requires a new version and a
  regenerated catalog. Keep old release assets available for installed users.
- The editor fetches the catalog only on an explicit **Fetch official list**
  action. It does not auto-update packages. Installation always re-fetches and
  re-validates the catalog natively.
- Package limits are 4096 files, 512 MiB of decoded content, and a 768 MiB
  package envelope. The native backend enforces the 768 MiB download cap. The
  catalog is capped at 1 MiB by both the generator and the native backend.
- Version components must each fit an unsigned 64-bit integer, matching the
  native parser.
- Catalog schema 1 has no mirror field and the catalog URL is fixed in the
  native backend. Changing the repository, raw catalog URL, or target set
  requires a schema version change.
