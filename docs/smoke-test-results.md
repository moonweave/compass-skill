# Smoke Test Results

## 2026-05-31 — Codex hardening pass

### Scope

Verify the `compass` source skill after adding Claude Code manual-only metadata, replacing macOS-incompatible collector snippets, and restricting transcript fallback to the current cwd-derived Claude project directory.

### Static Verification

Command:

```bash
python3 - <<'PY'
from pathlib import Path
import yaml
path = Path('/Users/choemun-yeong/ai-skills-dev/my-skills/compass-skill/SKILL.md')
text = path.read_text()
fm = yaml.safe_load(text.split('---', 2)[1])
assert fm['name'] == 'compass'
assert fm.get('disable-model-invocation') is True
assert fm.get('argument-hint') == '[baseline intent]'
assert '-printf' not in text
assert '~/.claude/projects/*/*.jsonl' not in text
assert 'timeout 30' not in text
assert 'Portable macOS/Linux cache mtime collector' in text
assert 'do not imply tests are healthy' in text
print('compass metadata and portability rules ok')
PY
```

Result: passed.

### macOS Reproduction And Replacement Check

Commands:

```bash
find . -maxdepth 1 -type f -printf '%T@ %p\n'
command -v timeout
```

Result: the GNU-specific `find -printf` command failed on macOS and `timeout` was unavailable, confirming the original portability issue.

Replacement collector command:

```bash
cd /Users/choemun-yeong/ai-skills-dev/my-skills/anneal-skill
uv run python3 - <<'PY'
from pathlib import Path
paths = [Path('.ruff_cache'), Path('coverage.xml')]
files = []
for root in paths:
    if root.is_file():
        files.append(root)
    elif root.is_dir():
        files.extend(p for p in root.rglob('*') if p.is_file())
print('portable collector files', len(files))
if files:
    newest = max(files, key=lambda p: p.stat().st_mtime)
    print(f"{newest.stat().st_mtime:.0f} {newest}")
PY
```

Result:

```text
portable collector files 4
1780052933 .ruff_cache/0.15.5/740700146490957402
```

### Fresh Claude Code Invocation

Command:

```bash
claude --print --no-session-persistence --max-budget-usd 0.20 --allowedTools Read,Bash --permission-mode bypassPermissions '/compass harden local Claude/Codex skills. Do not edit files. Run a compact audit only.'
```

Result: blocked by Claude CLI organization policy, before skill execution:

```text
Your organization has disabled Claude subscription access for Claude Code · Use an Anthropic API key instead, or ask your admin to enable access
```

### Verdict

Local metadata and portability verification passed. True fresh Claude Code invocation remains blocked by account/organization policy. This is not a skill-file failure.
