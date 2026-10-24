# Windows compatibility patches (local, not upstream)

This checkout is `google-research/harness2` at `main` with two small patches so it runs
natively on Windows instead of WSL. Upstream targets POSIX (see the
`Operating System :: POSIX` classifier and the README's "Use Linux").

Both patches are no-ops on Linux/macOS.

## 1. `harness2/core/schedule.py` — no `fcntl` on Windows

`_write_summary()` did a bare `import fcntl` and held an `flock` while merging the
per-task summary, so **every** task died with
`ModuleNotFoundError: No module named 'fcntl'` before doing any work.

Added a `_exclusive_lock()` context manager: `fcntl.flock(LOCK_EX)` on POSIX,
`msvcrt.locking(LK_NBLCK)` on Windows. `msvcrt` only locks bytes that exist and
`LK_NBLCK` fails instead of blocking, so the helper sizes the lock file to one byte and
retries on `OSError` with a 50 ms sleep, giving real blocking semantics.

## 2. `harness2/core/tree.py` — path separators in tree keys

`read_tree()` and `write_tree()` built relative paths with `str(p.relative_to(d))`,
which yields `scripts\summarize.py` on Windows. Every other module in the package uses
`.as_posix()` and expects POSIX-style keys, so on Windows:

- `adapter.validate_tree()`'s `"scripts/summarize.py" not in harness` always failed,
  so every proposed edit was rejected, twice, and silently fell back to the base tree;
- README stubs were never excluded on read.

Symptom: the demo finished "0 tasks completed; 2 failed" with
`StopIteration` inside `DemoProposer.call`, because no candidate matched the expected
total — every candidate was an unedited copy of the base harness.

Fixed by using `.as_posix()` at both sites.

## Verification

```
uv venv --python 3.12 .venv
uv pip install --python .venv\Scripts\python.exe -e ".[render]"
.\.venv\Scripts\harness2-demo.exe --output runs\demo --k 2
.\.venv\Scripts\python.exe -m pytest tests -q
```

- `harness2-demo`: `Local demo: 2 tasks completed; 0 failed.` (exit 0)
- `pytest tests`: `5 passed, 2 subtests passed`
- `harness2`, `harness2-check`, `harness2-report`, `harness2-setup`,
  `harness2-judge` all reach their argparse help.

## Not verified on Windows

Model-backed benchmarks and the shipped benchmarks (`jb`, `wb`, `lab`) were not run:
they need Vertex AI credentials plus `gcloud`, and `lab` also wants Pandoc/Poppler and
Docker. Those may surface further POSIX assumptions — if so, WSL Ubuntu (already
installed, WSL2) is the supported environment.
