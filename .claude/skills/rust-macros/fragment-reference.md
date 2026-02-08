# Fragment Follow-Set Restrictions and Repetition Reference

## Follow-Set Restrictions

These are the most common source of confusing `macro_rules!` compile errors. Each fragment specifier restricts which tokens can follow it:

| Specifier | Allowed Followers |
|-----------|-------------------|
| `expr`, `stmt` | `=>`, `,`, `;` |
| `pat_param` | `=>`, `,`, `=`, `\|`, `if`, `in` |
| `pat` | `=>`, `,`, `=`, `if`, `in` |
| `path`, `ty` | `=>`, `,`, `=`, `\|`, `;`, `:`, `>`, `>>`, `[`, `{`, `as`, `where`, or a `block` metavariable |
| `vis` | `,`, an identifier (except raw `priv`), any type-starting token, or an `ident`/`ty`/`path` metavariable |
| `ident`, `block`, `item`, `lifetime`, `literal`, `meta`, `tt` | **No restrictions** |

**Key insight:** `tt` is the only specifier that losslessly captures arbitrary input. `$($tail:tt)*` is the standard idiom for "the rest."

## Repetition Operators

```rust
$( ... )*      // Zero or more
$( ... )+      // One or more
$( ... )?      // Zero or one (no separator allowed)
```

**Separator rules:**
- Common separators: `,` and `;`
- Cannot be a delimiter or repetition operator
- `?` operator does not support separators

**Transcription rules:**
- Metavariable must appear at same nesting depth in transcriber as in matcher
- Each repetition must contain at least one metavariable
- Multiple metavariables in same repetition must bind to same number of fragments
