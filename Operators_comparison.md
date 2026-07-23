# PostgreSQL Array, JSON, and JSONB Operator Comparison

| Operator | Array | JSON | JSONB | Purpose |
|----------|:-----:|:----:|:-----:|---------|
| `=` | ✅ | ✅ | ✅ | Equality comparison |
| `<>` | ✅ | ✅ | ✅ | Not equal |
| `<` | ✅ | ❌ | ❌ | Less than (lexicographical comparison) |
| `<=` | ✅ | ❌ | ❌ | Less than or equal |
| `>` | ✅ | ❌ | ❌ | Greater than |
| `>=` | ✅ | ❌ | ❌ | Greater than or equal |
| `@>` | ✅ | ❌ | ✅ | Contains |
| `<@` | ✅ | ❌ | ✅ | Is contained by |
| `&&` | ✅ | ❌ | ❌ | Overlap (common elements) |
| `\|\|` | ✅ | ❌ | ✅ | Concatenate / Merge |
| `->` | ❌ | ✅ | ✅ | Get JSON object by key or array index |
| `->>` | ❌ | ✅ | ✅ | Get JSON value as text |
| `#>` | ❌ | ✅ | ✅ | Get nested JSON object |
| `#>>` | ❌ | ✅ | ✅ | Get nested JSON value as text |
| `?` | ❌ | ❌ | ✅ | Key or array element exists |
| `?\|` | ❌ | ❌ | ✅ | Any key exists |
| `?&` | ❌ | ❌ | ✅ | All keys exist |
| `-` | ❌ | ❌ | ✅ | Remove key or array element |
| `#-` | ❌ | ❌ | ✅ | Remove nested key or path |

---
