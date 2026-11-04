# uv comprehension quiz: answer key

## 1. Why uv is fast (it's not just Rust)

**Model answer:** Much of uv's speed comes from design decisions rather than the language. Python packaging standards made it possible to read static metadata without running code (PEP 517/518/621 and PEP 658 metadata on the index). uv also downloads and builds in parallel, uses the PubGrub resolver to prune dead ends, and installs from a global cache. To learn a wheel's dependencies, uv first tries PEP 658 metadata served by the index. If that isn't available, it uses HTTP range requests to read only the zip central directory and the METADATA file, and it downloads the whole wheel or builds from source only as a last resort. Rust adds further gains: no interpreter startup, true thread parallelism, zero-copy deserialization of cached data, and compact version encoding. uv also skips legacy work, such as eggs, pip.conf and default bytecode compilation.

**Key points for full credit:**
- Speed comes mostly from architecture and standards: static and PEP 658 metadata, parallelism, the PubGrub solver and the global cache, not "Rust" alone.
- It describes the metadata fallback chain: PEP 658 metadata, then HTTP range requests for the wheel's METADATA, then a full download or build.
- It mentions at least one legitimately Rust/binary-specific gain (no interpreter startup, threads without the GIL, zero-copy cache reads) or a legacy feature uv skips.

**Sources:** https://nesbitt.io/2025/12/26/how-uv-got-so-fast ; https://astral.sh/blog/uv

## 2. Global cache and link modes

**Model answer:** uv stores unpacked wheels once in a global cache and then links them into each environment instead of copying them. It uses copy-on-write clones on macOS and Linux and hardlinks on Windows, so a "warm" install mostly creates links and metadata, which costs almost no time or disk. When the cache and the environment are on different filesystems, linking is impossible, so uv falls back to slower full copies. That's why the docs recommend putting both on the same filesystem, and why Docker setups that use cache mounts set `UV_LINK_MODE=copy`. Symlink mode is discouraged because it couples the environment to the cache: `uv cache clean` would delete the files the symlinks point to and break every installed package.

**Key points for full credit:**
- A warm install links from the cache (reflink/clone or hardlink) instead of re-downloading, re-building or copying.
- Across filesystems, uv can't link and falls back to copying (slower, more disk), so keep both on the same filesystem or set link-mode copy explicitly.
- Symlinks make the environment depend on the cache's continued existence, so cleaning or pruning the cache breaks installs.

**Sources:** https://docs.astral.sh/uv/concepts/cache/ ; https://docs.astral.sh/uv/reference/settings/#link-mode

## 3. A single static binary that doesn't need Python

**Model answer:** pip is a Python package that runs inside a particular interpreter and installs into it, so you need a working Python, and the right one, before you can do anything. uv is a standalone Rust binary that can be installed independently of any Python. It can target any interpreter, create virtual environments for it, and replace pip, pip-tools, virtualenv, pipx and pyenv-style workflows with one tool. When no suitable interpreter exists, uv discovers one or automatically downloads a "managed" CPython build from Astral's python-build-standalone project, guided by `requires-python`, `.python-version` or `--python`. That's how `uvx tool` or `uv run` can work on a fresh machine with no Python at all.

**Key points for full credit:**
- uv is decoupled from any interpreter: it bootstraps without Python and manages environments for many interpreters, while pip is bound to the interpreter it runs in.
- uv can install or download Python itself (managed builds from python-build-standalone), automatically when needed.
- This decoupling is what lets one binary unify the toolchain: pip, pip-tools, virtualenv, pipx (uvx) and pyenv.

**Sources:** https://astral.sh/blog/uv ; https://docs.astral.sh/uv/concepts/python-versions/ ; https://astral.sh/blog/uv-unified-python-packaging

## 4. Universal lockfile vs platform-specific compile

**Model answer:** By default, `uv pip compile` resolves for the current platform and Python version only, like pip-tools. `uv.lock` is a universal resolution: one file that is valid across operating systems, architectures and Python versions, so locking on macOS also covers Linux and Windows. To do this, uv's resolver "forks" on environment markers. When requirements differ by platform or Python version, it can select different versions of the same package for different marker branches, so the lockfile can list a package more than once. Because the lock must hold for every Python the project claims to support, the project's `requires-python` must be a subset of each dependency's `requires-python`. If a dependency needs 3.10 or later but the project says 3.8 or later, there's no valid solution for 3.8 and 3.9, so locking fails. The fix is to raise the project's `requires-python` floor.

**Key points for full credit:**
- `uv pip compile` is platform-specific by default (unless run with `--universal`), while `uv.lock` is universal and cross-platform.
- The forking resolver can choose different versions per marker branch, which is why the same package can appear more than once.
- Resolution must satisfy the whole declared `requires-python` range (uv uses only its lower bound), not just the interpreter you run, so a dependency with a higher floor breaks the lock.

**Sources:** https://docs.astral.sh/uv/concepts/resolution/ ; https://docs.astral.sh/uv/concepts/projects/layout/

## 5. `--locked` vs `--frozen`, and what "outdated" means

**Model answer:** `--locked` checks that `uv.lock` still matches the project metadata and errors instead of silently re-locking. That makes it right for CI and for the final Docker sync, where a stale lockfile should fail the build. `--frozen` uses the lockfile as-is without checking it against `pyproject.toml`. That's useful in an early Docker layer where only `uv.lock` (and not the full project or workspace) is present, so validation isn't possible. A lockfile counts as outdated only when it no longer matches the project's declared metadata (for example, after adding a dependency or tightening a constraint that excludes a locked version). A new upstream release does not make it outdated. uv keeps preferring locked versions until you run `uv lock --upgrade` or `--upgrade-package`.

**Key points for full credit:**
- `--locked` validates the lockfile against the project and fails if it's out of date. `--frozen` skips validation and trusts the lockfile.
- A correct use case for each: `--locked` for CI and release builds, `--frozen` when the metadata needed for validation isn't available (such as a dependency-only Docker layer).
- New releases on the index don't make the lockfile stale. Upgrades are explicit (`uv lock --upgrade` or `--upgrade-package`).

**Sources:** https://docs.astral.sh/uv/concepts/projects/sync/ ; https://docs.astral.sh/uv/guides/integration/docker/

## 6. The two interfaces and exact sync

**Model answer:** `uv pip` is a low-level, imperative, pip-compatible interface. It changes an environment directly and doesn't touch `pyproject.toml` or `uv.lock`. The project interface (`uv add`, `uv lock`, `uv sync`, `uv run`) is declarative: `pyproject.toml` states intent, `uv.lock` records the resolution, and the `.venv` is derived state that uv makes match the lockfile. `uv sync` is exact by default, so it removes packages the lockfile didn't select, which wiped out the ad-hoc `rich`. The developer should have run `uv add rich` (or `uv add --dev rich`), or for a one-off, `uv run --with rich ...`. The docs explicitly discourage changing the project environment by hand with `uv pip install`.

**Key points for full credit:**
- `uv pip` changes the environment imperatively and bypasses `pyproject.toml` and `uv.lock`. The project commands treat the `.venv` as derived from the lockfile.
- `uv sync` does an exact sync by default and removes extraneous packages (`--inexact` keeps them, and `uv run` syncs inexactly by default).
- The correct fix is `uv add` (or `--dev`, or `uv run --with` for temporary use).

**Sources:** https://docs.astral.sh/uv/concepts/projects/sync/ ; https://docs.astral.sh/uv/concepts/projects/layout/

## 7. Multiple indexes and dependency confusion

**Model answer:** pip merges candidates from all configured indexes and picks the highest version, so the attacker's 99.0 on PyPI wins over the internal package. That's a classic dependency-confusion attack. By default, uv uses the `first-index` strategy: once it finds a package on an index (in priority order), it considers only the versions on that index. So `acme-utils` comes from the internal index and the public 99.0 is never considered. The trade-off is that legitimate newer versions on a later index are also ignored. You can opt into pip-like behavior with `--index-strategy unsafe-best-match` (or `unsafe-first-match`), where the "unsafe" names flag the risk. You can also pin packages to specific indexes via `tool.uv.sources` and `explicit = true` indexes.

**Key points for full credit:**
- pip combines all indexes and takes the best version, so a higher-versioned public package can hijack an internal name.
- uv's default `first-index` limits candidates to the first index that has the package, specifically to prevent dependency confusion.
- `--index-strategy unsafe-best-match` restores pip-like merging (with the risk), and per-package index pinning gives finer control.

**Sources:** https://docs.astral.sh/uv/concepts/indexes/ ; https://docs.astral.sh/uv/pip/compatibility/

## 8. Bytecode compilation default

**Model answer:** pip compiles installed `.py` files to `.pyc` at install time, while uv does not by default. Instead, bytecode is compiled lazily the first time each module is imported. That makes installs faster (and makes some install benchmarks less apples-to-apples), but it moves the cost to first import. In a container, that cost can be paid on every fresh start, especially if the filesystem is read-only or the `.pyc` files aren't persisted. The fix is `--compile-bytecode` or `UV_COMPILE_BYTECODE=1` during the image build, which trades a longer build and a larger image for faster startup. The docs recommend it for Docker images and CLIs.

**Key points for full credit:**
- It identifies the cause: uv skips bytecode compilation by default (pip does it), so modules compile lazily on first import.
- It explains the trade-off behind the default: faster install versus slower first start, which is one of uv's deliberate differences from pip.
- It gives the fix: `--compile-bytecode` or `UV_COMPILE_BYTECODE=1` at build time (for example in the Dockerfile).

**Sources:** https://docs.astral.sh/uv/pip/compatibility/ ; https://docs.astral.sh/uv/guides/integration/docker/ ; https://docs.astral.sh/uv/reference/settings/#compile-bytecode
