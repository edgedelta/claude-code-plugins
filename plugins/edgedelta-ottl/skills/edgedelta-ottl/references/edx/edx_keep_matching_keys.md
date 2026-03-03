# edx_keep_matching_keys - Keep Keys by Regex Pattern

**Function Type**: EDX Editor
**Purpose**: Keeps only keys matching regex patterns (deletes all others)
**Signature**: `edx_keep_matching_keys(target, patterns) -> void`
**Return Type**: void (modifies map in-place)
**Minimum Agent Version**: v1.23.0
**Source**: /pkg/ottlext/funcs/func_edx_keep_matching_keys.go

---

## Quick Copy

```yaml
# Keep only metric fields
edx_keep_matching_keys(attributes, ["^metric_", "^gauge_"])
```

---

## Overview

edx_keep_matching_keys is a pattern-based whitelist filter.

---

## Parameters

### target
- **Type**: map
- **Required**: Yes

### patterns
- **Type**: []string (regex patterns)
- **Required**: Yes

---

## Examples

### Example 1: Keep Metrics Only
```yaml
edx_keep_matching_keys(attributes, ["^metric\\.", "^counter\\."])
```

### Example 2: Keep User Fields
```yaml
edx_keep_matching_keys(attributes, ["^user_", "^account_"])
```

---

## Notes

- **Version**: v1.23.0
- **Inverse**: Opposite of edx_delete_matching_keys
