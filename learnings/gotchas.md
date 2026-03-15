# Gotchas & Edge Cases

Entries are appended by agents as they encounter footguns and edge cases.

---

## Skills must live in global skill dirs
**Date:** 2026-02-07
**Context:** Factory skills configuration
**Tags:** skills, factory, configuration, droids, agents

### Problem / Observation
Skills were mistakenly created inside a project repo under `.factory/droids` or `.claude/agents`, which are for droids/agents rather than skills.

### Resolution / Insight
Create skills only in the global root skill directories (e.g., `~/.factory/skills` and `~/.claude/skills`) so they are discoverable and loaded correctly.

---

## macOS tempfile paths can flip between /var and /private/var
**Date:** 2026-03-15
**Context:** Python pathlib + macOS temp dirs
**Tags:** macos, pathlib, tempfile, symlink, tests

### Problem / Observation
On macOS, `tempfile.TemporaryDirectory()` often returns a `/var/...` path, but calling `Path.resolve()` converts it to `/private/var/...` because `/var` is a symlink. String-equality assertions then fail even when both paths reference the same location.

### Resolution / Insight
Avoid forcing `resolve()` on caller-provided temp paths when exact string preservation matters (for env vars/log outputs), or normalize both sides with `os.path.realpath()`/`Path.resolve()` in tests before asserting.

### Commands / Code
```python
def apply_live_defaults(secret_dir: Path = _project_root() / ".secrets") -> dict[str, Any]:
    # Keep caller-provided path form (don't call secret_dir.resolve())
    private_key_path = secret_dir / "kalshi_private_key.key"
```

```python
self.assertEqual(os.path.realpath(actual), os.path.realpath(expected))
```

