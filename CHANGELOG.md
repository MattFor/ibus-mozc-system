# Changelog

## [1.2.2] - 11:51 CEST 10-11-2026

### Fixed

- Japanese no longer turns itself on after the computer was left alone. `mozc_server` drops a session
  idle for an hour and the one `ibus-mozc` recreates starts in hiragana; `kana-switch follow` now
  touches the session every ten minutes while English is on (`KANA_KEEPALIVE_S`), and repairs one
  that already expired.
- `kana-switch`, `mozc-keymap` and `ime-probe` print their usage for `--help` instead of an empty
  line.
- `install` no longer points at a `_documentation` directory the repository does not ship.

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

[1.2.2]: https://github.com/MattFor/ibus-mozc-system/releases/tag/v1.2.2
[1.2.1]: https://github.com/MattFor/ibus-mozc-system/releases/tag/v1.2.1
