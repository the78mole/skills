---
name: cjk-filename-renaming
description: Guide for renaming files and directories containing Chinese/CJK characters from vendor deliverables. Use this when dealing with non-ASCII filenames from Chinese hardware manufacturers (e.g., GBK-encoded filenames on Linux/UTF-8 systems).
---

# Renaming Files with Non-ASCII (CJK) Characters

## Problem

Chinese manufacturers deliver files with CJK characters in filenames that can cause issues on Linux systems (UTF-8). Often the filenames use **GBK encoding** instead of UTF-8.

## Detection and Decoding

1. First, try to decode the byte string as **UTF-8**.
2. On `UnicodeDecodeError`, try **GBK**.
3. Use `os.fsencode()` / byte paths to avoid encoding issues with `os.walk`.

```python
def decode_filename(raw_bytes):
    try:
        return raw_bytes.decode('utf-8')
    except UnicodeDecodeError:
        return raw_bytes.decode('gbk', errors='replace')
```

## CJK Unicode Ranges

Relevant Unicode blocks for detection:

- `\u4e00-\u9fff` – CJK Unified Ideographs (most common)
- `\u3400-\u4dbf` – CJK Unified Ideographs Extension A
- `\uf900-\ufaff` – CJK Compatibility Ideographs

Regex pattern:
```python
import re
CJK_PATTERN = re.compile(r'[\u4e00-\u9fff\u3400-\u4dbf\uf900-\ufaff]+')
```

## Replacement Rules

When replacing CJK characters with a placeholder (e.g., `xxx`):

- **Insert separators** when adjacent characters are alphanumeric to prevent merging.
- Example: `Linux软件` → `Linux-xxx` (not `Linuxxxx`)
- Example: `修改说明.txt` → `xxx.txt`

```python
def replace_cjk(name, replacement='xxx'):
    result = CJK_PATTERN.sub(replacement, name)
    # Insert separators where needed
    result = re.sub(r'([a-zA-Z0-9])xxx', r'\1-xxx', result)
    result = re.sub(r'xxx([a-zA-Z0-9])', r'xxx-\1', result)
    return result
```

## Renaming Order

- **Rename directories bottom-up** (deepest level first) so that parent paths remain valid during renaming.
- On conflicts (target name already exists): Use numbered suffixes (`xxx`, `xxx-2`, `xxx-3`).

## Complete Algorithm

```python
import os, re

def rename_cjk_tree(root_path):
    root_bytes = os.fsencode(root_path)
    renames = []

    for dirpath, dirnames, filenames in os.walk(root_bytes, topdown=False):
        # First rename files
        for fname in filenames:
            decoded = decode_filename(fname)
            if CJK_PATTERN.search(decoded):
                new_name = replace_cjk(decoded)
                renames.append((
                    os.path.join(dirpath, fname),
                    os.path.join(dirpath, os.fsencode(new_name))
                ))
        # Then rename directories
        for dname in dirnames:
            decoded = decode_filename(dname)
            if CJK_PATTERN.search(decoded):
                new_name = replace_cjk(decoded)
                renames.append((
                    os.path.join(dirpath, dname),
                    os.path.join(dirpath, os.fsencode(new_name))
                ))

    for old, new in renames:
        os.rename(old, new)
```
