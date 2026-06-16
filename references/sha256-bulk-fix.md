# Sha256 Bulk Fix for Raw Sources

When lint reveals many sha256 mismatches (missing, placeholder, or drifted), use this
`execute_code` pattern to fix all of them in a single pass. It walks every `.md` file
in `raw/`, computes the body sha256 (everything after the closing `---`), and either
adds or updates the `sha256:` frontmatter field.

## Script

```python
import os, re, hashlib

WIKI = os.environ["WIKI_PATH"]  # your wiki root (native Windows path, e.g. D:\wiki)
raw_dir = os.path.join(WIKI, 'raw')

stats = {'added': 0, 'updated': 0, 'skipped': 0}

for root, dirs, files in os.walk(raw_dir):
    for f in files:
        if not f.endswith('.md'):
            continue
        path = os.path.join(root, f)
        with open(path, 'r', encoding='utf-8') as fh:
            original = fh.read()

        # Must have frontmatter
        if not original.startswith('---'):
            stats['skipped'] += 1
            continue

        end = original.find('---', 3)
        if end == -1:
            stats['skipped'] += 1
            continue

        fm = original[3:end]
        body = original[end+3:]  # everything after closing ---

        # Compute current body sha256
        current_sha = hashlib.sha256(body.encode('utf-8')).hexdigest()

        # Check existing sha256
        sha_match = re.search(r'sha256:\s*(\S+)', fm)

        if sha_match:
            stored = sha_match.group(1)
            if stored == current_sha:
                stats['skipped'] += 1
                continue

            # Update sha256 — replace the stored value with current
            new_fm = fm.replace(f'sha256: {stored}', f'sha256: {current_sha}')
            new_content = '---\n' + new_fm + '\n---' + body
            with open(path, 'w', encoding='utf-8') as fh:
                fh.write(new_content)
            stats['updated'] += 1
        else:
            # No sha256 field — add one after ingested line
            ingested_match = re.search(r'ingested:.*', fm)
            if ingested_match:
                insert_pos = ingested_match.end()
                new_fm = fm[:insert_pos] + '\nsha256: ' + current_sha + fm[insert_pos:]
            else:
                new_fm = fm.rstrip() + '\nsha256: ' + current_sha

            new_content = '---\n' + new_fm + '\n---' + body
            with open(path, 'w', encoding='utf-8') as fh:
                fh.write(new_content)
            stats['added'] += 1

print(f"Done: {stats['added']} added, {stats['updated']} updated, {stats['skipped']} skipped")
```

## Verification

After running, verify with a drift re-check:

```python
drifts = 0
for root, dirs, files in os.walk(raw_dir):
    for f in files:
        if not f.endswith('.md'):
            continue
        path = os.path.join(root, f)
        with open(path, 'r', encoding='utf-8') as fh:
            text = fh.read()
        end = text.find('---', 3)
        if end == -1:
            continue
        fm = text[3:end]
        sha = re.search(r'sha256:\s*(\S+)', fm)
        if not sha:
            continue
        body = text[end+3:]
        current = hashlib.sha256(body.encode('utf-8')).hexdigest()
        if current != sha.group(1):
            drifts += 1

print(f"Drifts remaining: {drifts}")  # should be 0
```

## When to Use

- After initial wiki creation where placeholder hashes were used
- After discovering systemic sha256 drift across many files
- As part of the lint workflow when sha256 mismatches exceed ~10 files (bulk-fix is faster than per-file patches)
