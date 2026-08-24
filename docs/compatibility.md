# Compatibility and Versioning

## Scope

`silo-plugin-sdk` is the public build-time contract for Go plugin authors.

This repository is released as a semver-governed Go module. Third-party plugins and first-party consumers should depend on tagged releases, not on sibling repo checkouts or workspace-only overrides.

The compatibility boundary includes:

- protobuf messages and gRPC services under `pkg/pluginproto/silo/plugin/v1`
- runtime bootstrap behavior in `pkg/pluginsdk/runtime`
- manifest helpers in `pkg/pluginsdk/manifest`
- config validation helpers in `pkg/pluginsdk/config`
- generic capability metadata conversion helpers in `pkg/pluginsdk/convert`
- canonical image-variant strings in `pkg/pluginsdk/imagevariant`

## Versioning Rules

- Treat the SDK as a semver boundary.
- Publish semver tags from this repository and consume those tags from downstream repos.
- Prefer additive protobuf evolution.
- Avoid renaming or removing protobuf fields, services, or enum values in `v1`.
- Keep plugin capability expansion additive: new functionality should arrive as new capability families or additive fields, not breaking rewrites of existing ones.
- First-party consumers should not merge code that depends on new SDK packages or symbols until the required SDK tag exists.

## Consumer Rules

- `Silo`, `silo-plugin-tvdb`, and `silo-plugin-tmdb` should pin released SDK tags in `go.mod`.
- CI and release pipelines should build with `GOWORK=off` and without checking out this repo as a sibling source dependency.
- Local `go.work` files and temporary `replace` directives are acceptable for development, but they must not be committed as the release path.

## Open Vocabularies

Some contract fields carry an open string vocabulary rather than an enum, so
Silo can add values without a breaking protobuf change. Plugins must tolerate
values they do not recognize.

Image variants (`ResolveImageURLRequest.variant`,
`ResolveImageURLsRequest.variant`, `ResolveCatalogImageURLsRequest.variant`) are
the current example. The canonical values are exported from
`pkg/pluginsdk/imagevariant`: `card`, `featured`, `large`, `full`, `original`,
listed smallest to largest. `large` (~780px posters and stills, ~1280px logos
and backdrops) was added between `featured` and `full` once Silo gained
client-selectable image sizes; adding it is an additive change and does not
require a plugin update.

A plugin receiving an unknown variant MUST degrade gracefully to its nearest
supported size — or its default size — and MUST NOT return an error. Returning
an error turns a slightly-wrong image size into a missing image. Use a `switch`
with a `default` arm rather than an exhaustive match, and do not assume the
constants shipped in any given SDK tag are the complete set. The same rule
applies to any future open vocabulary added to `v1`.

## Runtime Compatibility

- `silo_api_version` is the coarse runtime compatibility gate between Silo and a plugin binary.
- Host installs should reject incompatible API versions before runtime startup.
- A plugin binary should return the same manifest shape that Silo installs, except that binaries may compute their checksum dynamically at runtime.

## Go Support

The supported public authoring path today is Go-only.

The protobuf and gRPC contracts are the long-term compatibility source of truth, but non-Go authoring is not an official support target in this release.

## Self-Describing Binary Guidance

If a plugin should be installable by direct binary upload:

- embed a manifest template in the binary
- compute the executable checksum at runtime
- return that populated manifest from `Runtime.GetManifest`

This keeps the plugin installable without requiring external repo state at upload time.
