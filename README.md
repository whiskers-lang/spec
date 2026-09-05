# Whiskers Specification

The formal specification for the Whiskers template language and its
conformance test suite.

Whiskers is a superset of [Mustache][mustache]: every valid Mustache
template is a valid Whiskers template. Whiskers adds named scopes,
presence/absence and null checks, iteration metadata, lambda arguments,
and strict mode.

## Contents

- [`spec.md`](spec.md) — formal PEG grammar and semantic rules
- [`specs/`](specs/) — YAML conformance fixtures grouped by feature

## Conformance

Implementations should pass every fixture that does not begin with `~`.
Fixtures prefixed with `~` describe optional modules; implementations may
opt out of them.

| File | Required |
|------|:--------:|
| `sigils.yml` | ✅ |
| `aliases.yml` | ✅ |
| `~local-lookup.yml` | optional |
| `~metadata.yml` | optional |
| `~lambdas.yml` | optional |
| `~strict.yml` | optional |

The fixture format matches the [Mustache spec][mustache-spec] YAML layout
so existing test harnesses can consume both.

## Versioning

The specification uses [Semantic Versioning](https://semver.org/):
- **Major** — breaking grammar or semantic change
- **Minor** — additive feature (new sigil, new fixture)
- **Patch** — clarifications, non-normative edits, fixture bug fixes

## License

MIT. See [`LICENSE`](LICENSE).

[mustache]: https://mustache.github.io/
[mustache-spec]: https://github.com/mustache/spec
