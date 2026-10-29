# Changelog

## [Unreleased]

### Fixed

- A `recent` other than `update` and `creation` is an `InvalidConfigException` (an
  `\InvalidArgumentException` still), which `Registry::usable()` leaves out.
- `sandbox` and `resolve_references` given as strings read `"false"` and `"0"` as false
  (`Config::bool()`): `sandbox: 'false'` called PISTE's sandbox. Requires omnisense/omnilex with
  `Config::bool()`.
