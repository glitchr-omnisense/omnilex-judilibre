# Changelog

## [Unreleased]

## [1.0.0] - 2026-11-13

### Added

- `policy.json`: the service's usage policy as data (limits, licence, credit, what a request needs), read at its pages on 2026-11-10 and named in `composer.json` (`extra.omnisense.policy`); the core honours it on every request.

### Fixed

- A `recent` other than `update` and `creation` is an `InvalidConfigException` (an
  `\InvalidArgumentException` still), which `Registry::usable()` leaves out.
- `sandbox` and `resolve_references` given as strings read `"false"` and `"0"` as false
  (`Config::bool()`): `sandbox: 'false'` called PISTE's sandbox. Requires omnisense/omnilex with
  `Config::bool()`.

### Changed

- Requires `omnisense/omnilex: ^1.0`: stable versions, so `composer require omnilex/judilibre` installs in a project at the default minimum-stability. `extra.branch-alias` names 1.x (`1.x-dev`); the CI (`.github/workflows/ci.yml`, glitchr-studio/ci's shared workflow) and the release on a tag (`release.yml`).

