# Whiskers Formal Grammar

## Notation

This grammar uses PEG (Parsing Expression Grammar). PEG is deterministic — there is no ambiguity. Rules are matched in order; the first match wins.

| Notation | Meaning |
|----------|---------|
| `←` | rule definition |
| `/` | ordered choice (try left first) |
| `*` | zero or more |
| `+` | one or more |
| `?` | zero or one (optional) |
| `!` | negative lookahead — match only if the following does NOT match |
| `" "` | literal string |
| `[ ]` | character class |
| `( )` | grouping |
| `(* *)` | comment |

---

## Grammar

```peg
template       ← node* EOF

node           ← tag / text

text           ← (!'{{' ANY)+

(* Comments and triple-mustache must be checked before regular tags *)
tag            ← '{{!' (!'}}' ANY)* '}}'
               / '{{{' _ keypath _ '}}}'          (* unescaped, triple-mustache *)
               / '{{' _ tag_inner _ '}}'          (* regular tag *)

tag_inner      ← '/' _ keypath                    (* section close *)
               / '&' _ keypath                    (* unescaped, ampersand form *)
               / '>' _ '*' _ keypath             (* dynamic partial *)
               / '>' _ keypath                    (* partial *)
               / '=' _ DELIM _ DELIM _ '='       (* set delimiter *)
               / opener _ keypath (_ ':' _ IDENT)? (_ argument)*   (* section open *)
               / keypath                          (* variable *)

opener         ← '#'              (* truthy *)
               / '^'             (* falsy *)
               / '*'             (* exists *)
               / '%'             (* absent *)
               / '~'             (* not null *)
               / '?'             (* null *)
               / '<'             (* parent template *)
               / '$'             (* block *)

keypath        ← '.'                              (* current value *)
               / '@' IDENT ('.' IDENT)*           (* metadata vars and @root; @strict reserved for strict blocks *)
               / '.' IDENT ('.' IDENT)*           (* local lookup; .alias.key form *)
               / IDENT ('.' IDENT)*               (* normal lookup; alias.key form *)

argument       ← '"' (!'"' ANY)* '"'             (* quoted string literal *)
               / '*' keypath                      (* dynamic context value *)
               / [0-9]+ ('.' [0-9]+)?            (* numeric literal *)
               / IDENT                            (* unquoted string literal *)

IDENT          ← [a-zA-Z_] [a-zA-Z0-9_-]*
DELIM          ← (![= ] ANY)+                    (* non-space, non-= sequence *)
_              ← ' '*
ANY            ← .
EOF            ← !.
```

---

## Semantic rules

These constraints cannot be expressed in the grammar and must be enforced at runtime.

### Key resolution

| Syntax | Resolves |
|--------|----------|
| `name` | Search current context, then parent chain |
| `.name` | Current context only — no parent traversal |
| `alias.key` | Named scope `alias`, then `key` within it |
| `.alias.key` | Named scope `alias`, no traversal within it |
| `@root` | Root context |
| `@root.key` | `key` in root context |
| `@index`, `@first`, etc. | Iteration metadata — only valid inside an array section |

### Section sigils

`#` and `^` are the truthy/falsy openers inherited from Mustache. Each condition has a dedicated single-character opener:

| Opener | Enters block when |
|--------|-------------------|
| `#` | value is truthy |
| `^` | value is falsy |
| `*` | key exists in data |
| `%` | key is absent |
| `~` | value is not null |
| `?` | value is null |

The three pairs test different axes — presence, nullness, and truthiness — and nest: `key exists (*) ⊃ value not null (~) ⊃ value truthy (#)`. Every truthy value is non-null; every non-null value exists — but not vice versa. A missing key fails all three positive checks.

All section forms set the context to the resolved value, consistent with `#`.

### Partials

Partials are inherited from Mustache. `{{> name}}` includes another template by name, rendered in the current context. The partial name follows keypath syntax, allowing namespaced names (`{{> shared.header}}`).

How partials are resolved (filesystem, registry, callback, etc.) is left to the implementation.

### Dynamic partials

`{{>*name}}` resolves `name` to a string at runtime and uses that string as the partial name. If the key is absent or the resolved name does not match a known partial, nothing is rendered. Set Delimiter tags do not affect dynamic name resolution.

### Set delimiter

`{{= new_open new_close =}}` changes the tag delimiters for all subsequent tags in the template. The new delimiters take effect immediately and persist until changed again.

The grammar above describes the set-delimiter tag format but cannot express the state change itself — implementations must handle delimiter tracking in a stateful lexer phase. The grammar rule `DELIM` matches any non-whitespace, non-`=` sequence.

The `{{{...}}}` triple-mustache form is tied to the default `{{`/`}}` delimiters. After a delimiter change, use the `&` form inside the new delimiters for unescaped output: `<% & foo %>`.

Restoring defaults uses the current delimiters: `<%={{ }}=%>`.

### Template inheritance

`{{< parent}}...{{/parent}}` injects a parent template, optionally overriding named blocks. `{{$ block}}default{{/block}}` defines an overridable region with default content.

Inside a parent tag, `{{$ block}}override{{/block}}` replaces the named block in the parent. A parent tag with no block overrides is equivalent to a partial. Block names live in a separate namespace from both partials and context keys. If the named parent is not found, nothing is rendered.

### Named scopes

A `:alias` suffix on a section opening tag binds the scope name for the duration of that block. Alias names must not shadow `@` metadata names.

### Lambda arguments

When a key resolves to a function it is called as a lambda:

- **Variable** `{{lambda}}` — called with no arguments; return value rendered (HTML-escaped) and interpolated.
- **Section** `{{#lambda}}text{{/lambda}}` — called with the raw, unrendered section text; return value treated as a new template and re-rendered in the current context.

Space-separated arguments after the key name are passed alongside the raw text. Providing arguments to a non-function key is an error.

Quoted and unquoted strings are literals: `"foo bar"` passes `foo bar`, and `foo` passes `foo`. Numeric literals are passed as numbers. A `*` prefix makes an argument dynamic, so `*foo` resolves `foo` from the current context and `*@root.foo` resolves from the root context.

### Strict mode

In strict mode, any variable or section tag that references a missing key throws an error. The `*` and `%` openers are exempt — `*` and `%` never throw regardless of mode.

`~` and `?` are not exempt — `~key` and `?key` throw if `key` is absent.

Null values do not throw. Strict mode checks key presence only; a key that exists with a null value is valid. `{{foo}}` where `foo` is null renders as `""` in both modes. To branch on nullness, use `~`/`?`.

Section tags follow the same rule as variables: `{{#foo}}` throws in strict mode if `foo` is absent. Guard with `*` when existence is uncertain: `{{*foo}}{{#foo}}...{{/foo}}{{/foo}}`.

Strict mode is activated by a CLI/render flag (global) or a `{{#@strict}}...{{/@strict}}` block in the template. The block may appear at any nesting depth. `@strict` is a reserved name — it is never looked up in data.

### Iteration metadata

`@index`, `@number`, `@first`, `@last`, and `@length` are injected automatically when iterating an array. They are not available outside of array sections.

`@root` is always available at any depth.

### Standalone tags

A tag is standalone if it occupies a line by itself with only optional surrounding whitespace. Standalone tags consume the entire line including the trailing newline, leaving no blank line in the output.

The following tag types are standalone-eligible: all section opens and closes (all openers), comments, set delimiter, partials, dynamic partials, parent tags (`<`), block tags (`$`), and `{{#@strict}}`.

Variable tags (`{{name}}`, `{{{name}}}`, `{{&name}}`) are never standalone.
```
