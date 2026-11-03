# Changelog

## [1.2.1] - 01:51 CEST 03-11-2026

### Added

- CI: ruff, basedpyright, shellcheck, shfmt and Prettier on every push; tagged releases are cut from
  the changelog.
- Repo tooling: `.editorconfig`, `.gitattributes`, `ruff.toml`, `pyrightconfig.json`,
  `.prettierrc.json`, Dependabot.
- README badges for CI, release and license.

### Changed

- README states Python 3.10+ is required.

### Fixed

- Lint and type-check findings across the Python and shell scripts; no behaviour change.

**Full Changelog**: https://github.com/MattFor/ibus-mozc-system/compare/v1.2.0...v1.2.1

## [1.2.0] - 23:10 CEST 05-10-2026

**Full Changelog**: https://github.com/MattFor/ibus-mozc-system/compare/v1.1.0...v1.2.0

## [1.1.0] - 02:14 CEST 08-09-2026

### Added

- Proper support for electron, flatpak apps. As well as a special case for Steam.

## [1.0.0] - 20:32 CEST 17-08-2026

Initial release.

[1.2.1]: https://github.com/MattFor/ibus-mozc-system/releases/tag/v1.2.1
