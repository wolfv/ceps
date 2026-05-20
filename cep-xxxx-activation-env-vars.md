# CEP XXXX - Structured Activation Environment Variables

<table>
<tr><td> Title </td><td> Structured Activation Environment Variables </td>
<tr><td> Status </td><td> Draft </td></tr>
<tr><td> Author(s) </td><td> Wolf Vollprecht &lt;wolf@prefix.dev&gt;</td></tr>
<tr><td> Created </td><td> May 20, 2026</td></tr>
<tr><td> Updated </td><td> May 20, 2026</td></tr>
<tr><td> Discussion </td><td> NA </td></tr>
<tr><td> Implementation </td><td> https://github.com/prefix-dev/rattler-build (planned) </td></tr>
<tr><td> Requires </td><td> CEP 32 (environment layout) </td></tr>
</table>

> The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT",
  "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as
  described in [RFC2119][RFC2119] when, and only when, they appear in all capitals, as shown here.

## Abstract

[CEP 32] defines `etc/conda/env_vars.d/*.json` as a directory of JSON
documents that each map an environment variable name to a literal string.
That format is intentionally minimal: it can set variables on activation,
but it cannot reference other variables (notably `${PREFIX}`), it cannot
prepend or append to `PATH`-style variables, and it cannot vary by
platform without the package having to ship per-subdir builds with
different JSON files.

This CEP defines a second, structured activation format under a new
sibling directory `etc/conda/env.d/`. Files in this directory MAY use
variable expansion, list-valued prepend / append / remove operations, and
conditional `if` blocks evaluated at activation time. The legacy
`env_vars.d/` format is unchanged and remains the canonical place for
plain string-to-string mappings.

This CEP also defines a YAML form of the same schema that recipe build
tools can convert into the on-disk JSON.

## Motivation

The plain `env_vars.d/` format has several limitations that recipe
authors today work around with hand-written shell and `.bat` activation
scripts:

1. **No reference to `${PREFIX}`.** The JSON values are static strings.
   To embed the install prefix, package builders rely on the standard
   prefix-replacement mechanism, which only works for the literal
   placeholder used at build time and is awkward to compose with other
   path fragments.
2. **No way to prepend or append.** A package that wants to add a
   directory to `PATH`, `PKG_CONFIG_PATH`, `LD_LIBRARY_PATH`, etc. cannot
   do so via `env_vars.d/`: a plain assignment overwrites the variable.
   Authors fall back to `activate.d/*.sh` and a parallel `*.bat` for
   Windows.
3. **No platform branching.** A single package that needs different
   values on Windows vs Unix must ship multiple files or, again, fall
   back to per-shell activation scripts.
4. **Three-way duplication for cross-shell support.** Activation scripts
   typically come as `.sh`, `.bat`, and `.ps1` triples (see [special
   files documentation][rb-special-files]) just to set the same handful
   of variables on different shells.

The result is that the simplest case (set a variable on activation) uses
the structured `env_vars.d/` mechanism, but anything beyond it falls off
a cliff into hand-written shell scripts. This CEP closes that gap.

## Specification

### Directory

For channels and packages implementing this CEP:

```text
$CONDA_PREFIX/etc/conda/env.d/*.json
```

Each file in this directory is an activation-variables document
following the schema below.

The legacy `etc/conda/env_vars.d/` directory defined by [CEP 32] is
unchanged. Conformant clients MUST continue to process it, and MUST
process `env.d/` in addition.

### File Schema

Each `*.json` file MUST be a JSON object with the following shape:

```json
{
  "version": 1,
  "vars": {
    "<NAME>": "<scalar | operation-object>",
    ...
  },
  "if": [
    {
      "selector": "<selector-expression>",
      "vars": { "<NAME>": "<scalar | operation-object>", ... }
    },
    ...
  ]
}
```

| Field      | Type    | Required | Description                                                                                  |
| ---------- | ------- | -------- | -------------------------------------------------------------------------------------------- |
| `version`  | integer | yes      | Schema version. MUST be `1` for this CEP.                                                    |
| `vars`     | object  | no       | Variables to apply unconditionally. Keys are variable names, values are scalars or operations. |
| `if`       | array   | no       | Ordered list of conditional blocks; each block has its own `vars` map gated by `selector`.   |

A conformant client MUST reject a file whose `version` it does not
recognize, and MUST log a warning if the file contains keys not defined
by the recognized version.

### Variable Values

Each value under a `vars` mapping is either:

1. A **scalar** — string, number, or boolean. Scalars assign the variable
   to that value, overwriting any previous value. Numbers and booleans
   are coerced to their JSON string representation when exporting to the
   process environment.
2. An **operation object** — a mapping with exactly one of the keys
   defined below.

#### Operations

| Key       | Value          | Effect on activation                                                                  | Effect on deactivation                                       |
| --------- | -------------- | ------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| `set`     | scalar         | Assign the variable to this value.                                                    | Restore the variable to its pre-activation value (or unset). |
| `prepend` | array of scalars | Insert the given entries at the front of the variable, in order, using the platform-appropriate separator (`;` on Windows for `PATH`-like variables, `:` elsewhere). Existing entries are preserved. | Remove the entries that were prepended by this file.        |
| `append`  | array of scalars | Insert the given entries at the end of the variable, in order, with the same separator rules. | Remove the entries that were appended by this file.          |
| `unset`   | `true`         | Unset the variable for the duration of activation.                                    | Restore the variable to its pre-activation value.            |

`prepend` and `append` use `os.pathsep` semantics: `;` on Windows, `:`
elsewhere. A future CEP MAY introduce explicit separator control; this
CEP does not.

#### Variable Expansion

String values (including scalar values, entries inside `prepend` /
`append` arrays, and the inside of `selector` expressions where the
grammar permits) MAY contain references of the form:

```text
${NAME}
${NAME:-default}
```

Conformant clients MUST expand these at activation time. The lookup
order is:

1. Variables already set in the activating process environment
   (including `${PREFIX}`, which the activator MUST provide).
2. Variables that have been set by earlier files in the same activation
   sequence (see **Processing Order** below).
3. The fallback value after `:-`, if the reference uses the default
   form. Without a default, an unresolved reference MUST cause activation
   of that file to fail with a diagnostic; the rest of the activation
   sequence SHOULD continue so a single bad file does not prevent the
   environment from activating.

Literal `$` and `{` characters are written as `$$` and `\{`
respectively. (`}` does not require escaping outside a reference.)

Variable references inside `prepend` / `append` arrays are expanded
**per entry**, before separator joining.

#### Reserved Variables

The activator MUST make the following variables available for expansion
even if they are not present in the parent environment:

| Name              | Value                                                                |
| ----------------- | -------------------------------------------------------------------- |
| `PREFIX`          | The absolute path of the activated environment (`$CONDA_PREFIX`).    |
| `CONDA_PREFIX`    | Same as `PREFIX`.                                                    |
| `CONDA_SUBDIR`    | The platform subdir of the environment (e.g. `linux-64`).            |
| `CONDA_DEFAULT_ENV` | The conda-visible name of the environment, when known.            |

A client MAY expose additional variables; recipe authors SHOULD limit
themselves to the table above for portability.

### Conditional Blocks

The `if` array contains zero or more conditional blocks. Each block has
the form:

```json
{
  "selector": "<selector-expression>",
  "vars": { ... }
}
```

The activator MUST evaluate each `selector` and apply the block's
`vars` only when the selector evaluates to true. Blocks are evaluated
**in order**, so a later block's assignments override earlier ones, and
both override the unconditional `vars`.

#### Selector Grammar

A selector expression is a boolean expression over a small set of
identifiers. It reuses the grammar used by conda recipe `if:` selectors:

- **Atoms**: `win`, `unix`, `linux`, `osx`, `freebsd`, `emscripten`,
  `wasi`, and the architecture identifiers `x86_64`, `aarch64`,
  `arm64`, `ppc64le`, `s390x`, `riscv64`, `wasm32`. Atoms evaluate to
  true when the activated environment's subdir matches.
- **Operators**: `and`, `or`, `not`, parentheses.
- **Equality on the subdir string**: `subdir == "linux-aarch64"` and
  `subdir != "win-64"`.
- **Equality on environment variables**: `env.NAME == "value"`. The
  activator MUST read `NAME` from the activating process environment.

A client encountering a selector it cannot parse MUST treat that block
as not applying and SHOULD emit a diagnostic.

The grammar is intentionally a subset of the recipe selector grammar so
that authors moving between `recipe.yaml` and `env.d/*.json` do not need
to learn a second language.

### Processing Order

The activator MUST process files in this order:

1. Files under `etc/conda/env_vars.d/` (as defined by [CEP 32]), in
   ascending lexicographical order.
2. Files under `etc/conda/env.d/`, in ascending lexicographical order.
3. The `conda-meta/state` document (as defined by [CEP 32]).

Within a single file, the activator MUST apply unconditional `vars`
first, then each `if` block in document order.

Later assignments overwrite earlier ones; `prepend` and `append`
accumulate on top of whatever value is current after earlier
processing.

### Deactivation

On deactivation, the activator MUST undo every change made on
activation, in reverse order:

- A variable that was assigned via `set` (or a scalar) is restored to
  its pre-activation value, or unset if it had none.
- Entries added via `prepend` / `append` are removed; the variable is
  unset if no other entries remain and the variable had no
  pre-activation value.
- A variable suppressed via `unset` is restored to its pre-activation
  value.

The pre-activation value of any variable touched by an `env.d/` file
SHOULD be recorded by the activator (for example in a sidecar file
under `conda-meta/`) so that nested activations and shell-spawning tools
can deactivate correctly. The exact storage mechanism is left to the
client.

### Errors

If a file under `env.d/` is malformed (invalid JSON, unknown `version`,
schema violation), the activator MUST skip that file and emit a
diagnostic. Activation of the environment as a whole MUST continue.

If variable expansion fails (unknown `${NAME}` with no default), the
activator MUST skip the offending value, leaving any earlier value
intact, and emit a diagnostic.

## Recipe Form

This CEP also defines an optional YAML form intended for use by recipe
build tools. The YAML form is **not** consumed by the activator: build
tools MUST translate it into the on-disk JSON format above and write it
to `etc/conda/env.d/<package_name>.json` inside the package payload.

The YAML form lives under an `activation:` (or comparable) section in a
recipe and uses the same schema:

```yaml
activation:
  vars:
    MY_PKG_HOME: ${{ "${PREFIX}/share/mypkg" }}
    PATH:
      prepend:
        - ${{ "${PREFIX}/opt/mypkg/bin" }}
  if:
    - selector: win
      vars:
        MY_PKG_CFG: ${{ "${PREFIX}\\etc\\mypkg.ini" }}
    - selector: unix
      vars:
        MY_PKG_CFG: ${{ "${PREFIX}/etc/mypkg.conf" }}
```

Note the two distinct templating layers:

| Layer        | Syntax  | Evaluated by             | When                  |
| ------------ | ------- | ------------------------ | --------------------- |
| Recipe Jinja | `{{ … }}` | The build tool (rattler-build, conda-build, …) | At build time |
| Activation expansion | `${VAR}` | The activator | At environment activation |

Recipe authors typically want the activator to expand `${PREFIX}`, so
the recipe Jinja layer SHOULD be used only when something must be
resolved at build time (build-time toggles, recipe context, etc.). The
`${PREFIX}` form should be passed through verbatim into the generated
JSON.

A recipe build tool that supports this CEP:

1. SHOULD validate the YAML against the schema before emitting JSON.
2. MUST emit a single JSON file at
   `$PREFIX/etc/conda/env.d/<package_name>.json`, with `version: 1`
   and the same structure (modulo Jinja substitution).
3. SHOULD warn if a recipe uses an unconditional `vars` entry whose
   value contains a platform-specific path separator (e.g. backslashes
   on a non-Windows-only output).

## Backwards Compatibility

Clients that do not implement this CEP will ignore `etc/conda/env.d/`,
since they look only at `env_vars.d/`. Packages that need to support
older clients SHOULD continue to emit the simple variables they care
about into `env_vars.d/` *in addition to* `env.d/`. The build tool MAY
provide a "lower to legacy" mode that emits a best-effort
`env_vars.d/*.json` from the unconditional, expansion-free subset of an
`activation:` block.

Packages MUST NOT rely on the activator merging `env.d/` and
`env_vars.d/` for the same variable in a particular way other than the
processing order above; in particular, a `prepend` in `env.d/` will be
applied *after* a flat assignment in `env_vars.d/` and will therefore
include the `env_vars.d/` value as one of its trailing entries.

## Examples

### Simple expansion of `${PREFIX}`

```json
{
  "version": 1,
  "vars": {
    "MY_PKG_HOME": "${PREFIX}/share/mypkg",
    "MY_PKG_CFG":  "${PREFIX}/etc/mypkg.conf"
  }
}
```

### Prepending to `PATH`

```json
{
  "version": 1,
  "vars": {
    "PATH": { "prepend": ["${PREFIX}/opt/mypkg/bin"] }
  }
}
```

On activation, `${PREFIX}/opt/mypkg/bin` is added to the front of
`PATH`. On deactivation, that entry is removed; the rest of `PATH` is
left untouched.

### Platform branching

```json
{
  "version": 1,
  "if": [
    { "selector": "win",
      "vars": { "MY_PKG_CFG": "${PREFIX}\\etc\\mypkg.ini" } },
    { "selector": "unix",
      "vars": { "MY_PKG_CFG": "${PREFIX}/etc/mypkg.conf" } }
  ]
}
```

### Multiple operations on the same variable

```json
{
  "version": 1,
  "vars": {
    "PKG_CONFIG_PATH": {
      "prepend": ["${PREFIX}/lib/pkgconfig", "${PREFIX}/share/pkgconfig"]
    }
  },
  "if": [
    { "selector": "linux and aarch64",
      "vars": {
        "PKG_CONFIG_PATH": { "append": ["${PREFIX}/lib/aarch64-linux-gnu/pkgconfig"] }
      } }
  ]
}
```

The unconditional `prepend` runs first; the conditional `append` runs
afterwards and only on `linux-aarch64`.

## Future Work

- **Explicit list separators.** Currently `prepend` / `append` use
  `os.pathsep`. A future revision MAY introduce a `separator` operator
  field for variables that are not path-like (e.g. `,`-separated lists).
- **Cross-package ordering.** Filenames already provide lexicographic
  ordering, but a richer dependency / priority mechanism may be needed
  for ecosystems where many packages contend for the same variable.
- **Activation-time hooks beyond environment variables.** Out of scope
  here; activation scripts in `activate.d/` continue to serve that
  purpose.

## References

- [CEP 32 - Conda environment layout][CEP 32]
- [rattler-build special files documentation][rb-special-files]

## Copyright

All CEPs are explicitly [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/).

[RFC2119]: https://www.ietf.org/rfc/rfc2119.txt
[CEP 32]: ./cep-0032.md
[rb-special-files]: https://rattler.build/latest/special_files/
