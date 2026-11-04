# Understanding uv, Astral's Python package manager

*Last checked against the uv docs and GitHub releases on 2026-11-03. The latest release then was uv 0.13.7.*

## 1. Overview

uv replaces a whole stack of Python tools with one fast Rust program:

| Job | What you used before | uv equivalent |
|---|---|---|
| Install packages into an environment | `pip` | `uv pip install` |
| Pin requirements | `pip-tools` (`pip-compile`, `pip-sync`) | `uv pip compile`, `uv pip sync` |
| Create virtual environments | `venv`, `virtualenv` | `uv venv` |
| Install and manage Python itself | `pyenv`, deadsnakes, python.org installers | `uv python install / pin / upgrade` |
| Run or install CLI tools in isolation | `pipx` | `uvx` / `uv tool run`, `uv tool install` |
| Manage projects (add deps, lock, sync, build, publish) | Poetry, PDM, Hatch, `build`, `twine` | `uv init / add / lock / sync / run / build / publish` |
| Single-file scripts with dependencies | ad-hoc venvs | `uv run script.py` with PEP 723 inline metadata |

Underneath all of that, uv rests on four pieces:

1. **One static binary.** There is no Python bootstrap problem, and it starts in milliseconds.
2. **One global, content-addressed cache.** Every package is downloaded and unpacked once per machine, then linked into any number of environments.
3. **One universal lockfile (`uv.lock`).** It is resolved once for every platform and Python version you support, and each machine installs the subset that applies to it.
4. **Two interfaces over the same engine.** The low-level `uv pip …` interface works like pip. The high-level project interface (`uv add`, `uv run`, …) is built around `pyproject.toml` and `uv.lock`.

The two interfaces share the same resolver, installer and cache. They differ in who owns the environment's state. With `uv pip`, you do, imperatively, the way you would with pip. With the project interface, `pyproject.toml` and `uv.lock` own it, and uv keeps `.venv` matching them.

---

## 2. The single Rust binary

### What you actually get
- Installing uv gives you `uv` plus `uvx`. `uvx` is a thin alias for `uv tool run`.
- uv needs no Python to run. It is self-contained native code, and the Linux builds are statically linked (musl). That fixes the classic problem where pip lives *inside* a Python install, so a broken Python means a broken installer and each interpreter carries its own pip.
- Install it any of these ways:
  - `curl -LsSf https://astral.sh/uv/install.sh | sh` (standalone installer, which enables `uv self update`)
  - `brew install uv`
  - `pipx install uv` or `pip install uv`. The PyPI wheel just ships the binary.
  - In Docker, copy the binary from the official image: `COPY --from=ghcr.io/astral-sh/uv:<version> /uv /uvx /bin/`.

### Why it is fast
uv is often quoted at 10–100x faster than pip. Rust is part of the reason, but the bigger wins are architectural:

- **Parallelism everywhere.** Downloads, unpacking, metadata fetching and installs run concurrently. pip does most of this serially.
- **Smart metadata fetching.** To resolve, uv needs each candidate's dependency list, not the whole wheel. It uses PEP 658 / PEP 714 metadata files when the index provides them. Otherwise it uses HTTP range requests to read only the zip central directory and the `METADATA` file from a remote wheel.
- **A modern resolver.** uv uses a PubGrub-based solver (Astral maintains a fork of `pubgrub-rs`). It backtracks efficiently and gives human-readable explanations when no solution exists ("because X depends on Y<2 and you require Y>=2…").
- **Installation is mostly linking, not copying.** Section 3 covers this; it is the biggest reason warm installs feel instant.
- **Less default work.** uv does not compile `.pyc` bytecode at install time, and `uv venv` does not seed pip or setuptools into new environments. Both are opt-in: `--compile-bytecode` and `--seed`.

### Python management sits in the same binary
- uv downloads managed CPython builds from Astral's **python-build-standalone** project, plus PyPy and GraalPy.
- By default it downloads interpreters automatically when needed (`python-downloads = "automatic"`) and prefers managed Pythons over system ones (`python-preference = "managed"`).
- `uv python pin 3.12` writes `.python-version`, which projects and `uv run` respect.
- `uv python upgrade` moves managed installs to the latest patch release. Virtual environments built on them follow along.
- uv supports free-threaded builds (`3.14t`) and debug builds (`3.13d`).
- Interpreter discovery order: managed installs, then interpreters on `PATH`, then (on Windows) the registry and Microsoft Store installs.

---

## 3. The global cache

### Where it lives
- It defaults to `$XDG_CACHE_HOME/uv` or `~/.cache/uv` on macOS and Linux, and `%LOCALAPPDATA%\uv\cache` on Windows.
- You can override it with `--cache-dir`, `UV_CACHE_DIR` or `tool.uv.cache-dir`. `--no-cache` uses a throwaway temporary cache.
- `uv cache dir` prints the location in use. `du -sh "$(uv cache dir)"` shows how big it has grown.

### What is in it
The cache is split into separately versioned **buckets**, one each for registry metadata (the Simple API responses), downloaded wheels, unpacked wheel archives, built source distributions, Git checkouts, interpreter info, and so on.

The directory names carry the bucket version, for example `wheels-vN`, `sdists-vN`, `archive-vN`, `git-vN`, `simple-vN`. The exact numbers change between releases, so look with `ls "$(uv cache dir)"` rather than relying on them.

Each bucket has its own version. When a release changes a bucket's format, uv bumps that bucket and simply stops reading the old one. Older and newer uv versions can share one cache directory without corrupting each other; they just won't share entries in buckets that changed. The docs' example is the metadata bucket moving from v12 to v13 in uv 0.4.13. Don't hand-edit anything in the cache.

### How an install uses it: link, don't copy
Every wheel is unpacked **once** into the cache. Installing into a venv then means creating file entries in `site-packages` that point at those cached files. The `link-mode` setting (or `UV_LINK_MODE`) controls how:

| Mode | What happens | Default on |
|---|---|---|
| `clone` | Copy-on-write reflink (APFS on macOS; btrfs/XFS and similar on Linux). Instant, uses no extra space until a file is modified, and fully independent of the cache afterwards. | macOS, Linux |
| `hardlink` | Hard link to the same inode. Instant with no extra space. Editing a file in the venv also edits the cache copy. | Windows |
| `copy` | Real copy. Slowest, but fully isolated. | – |
| `symlink` | Symlink into the cache. The docs discourage it: `uv cache clean` breaks every environment. | – |

As a result, ten projects that use the same `torch` wheel cost roughly one copy on disk, and creating a fresh venv with dozens of packages takes well under a second once the cache is warm.

Linking only works when the cache and the target venv are on the **same filesystem**. Typical cases where they aren't: a Docker bind mount, a separate `/workspace` volume, an external disk. uv then warns ("Failed to hardlink/clone files; falling back to full copy") and copies. The copy works, but it is slower. The fix is to put the cache on the same volume (`UV_CACHE_DIR=/workspace/.uv-cache`), or set `UV_LINK_MODE=copy` to silence the warning when copying is intended.

### How uv decides a cache entry is still valid
- **Registry packages:** uv follows HTTP caching headers.
- **Direct URLs:** HTTP caching headers plus the URL itself.
- **Git dependencies:** cached by resolved **commit hash**, so a branch reference is re-resolved but already-fetched commits are reused.
- **Local archives:** file modification time.
- **Local source directories:** the mtime of `pyproject.toml`, `setup.py` or `setup.cfg`. If your build has dynamic inputs (version from git, C sources, environment variables), add them to `tool.uv.cache-keys`. Otherwise uv will reuse a stale build of your own package.
- **`--find-links` flat indexes:** assumed immutable and cached by filename.
- **Directories passed on the command line** (`uv pip install .`) are always rebuilt.

When you need to force things:
- `--refresh` / `--refresh-package X` revalidates cached data.
- `--reinstall` / `--reinstall-package X` reinstalls packages.
- `uv cache clean [pkg]` deletes everything, or one package's entries.

### Concurrency
The cache is safe for **concurrent readers and writers**, so several `uv` processes (parallel CI jobs, multiple terminals) can share it. During installs uv takes a file lock on the *target environment*. Cache-mutating commands such as `uv cache clean` wait up to 5 minutes for other uv processes; `UV_LOCK_TIMEOUT` changes the wait and `--force` skips it.

### Keeping it lean
- `uv cache prune` removes unused entries and is safe to run periodically.
- `uv cache prune --ci` is meant for the end of CI jobs. It drops pre-built wheels, which are cheap to re-download, and keeps wheels you **built from source**, which are expensive to rebuild. Use it before saving a CI cache.

---

## 4. The lockfile (`uv.lock`)

### What it is
- It is a TOML file that **uv manages**. Commit it, but don't hand-edit it.
- It is **universal**: one lockfile covers every OS, architecture and Python version allowed by your `requires-python`. pip-tools' `requirements.txt` is different; it is resolved for exactly the machine that ran `pip-compile`.
- The format is specific to uv. For other tools, export it (see the export section below).

### How a universal resolve works
uv does not resolve for "this machine". It resolves for the whole space of environments allowed by your `requires-python`. When dependencies diverge by environment marker (`sys_platform == 'win32'`, `python_version < '3.11'`, `platform_machine == 'aarch64'`), the resolver **forks**. It solves each region separately and records the result per region. So one package can appear more than once in the lockfile at different versions, each guarded by markers. The top of the file lists the fork regions as `resolution-markers`.

Several settings shape the resolve:
- **`--fork-strategy`:** the default `requires-python` picks the newest version per Python version; `fewest` minimises the number of distinct versions.
- **`[tool.uv] environments = [...]`:** restricts which platforms are solved, for example to Linux and macOS only. This produces a smaller lock with fewer surprises.
- **`required-environments`:** the opposite. Resolution fails unless wheels exist for, say, `linux aarch64`. This is useful for GPU and ARM targets.
- **`[tool.uv] conflicts`:** declares extras or groups that can never be installed together, such as `cpu` and `cu128` torch variants, so uv resolves them separately instead of failing.

### What is inside
A trimmed, illustrative example (hashes shortened):

```toml
version = 1
revision = 3
requires-python = ">=3.11"
resolution-markers = [
    "sys_platform == 'linux'",
    "sys_platform != 'linux'",
]

[[package]]
name = "myapp"
version = "0.1.0"
source = { editable = "." }
dependencies = [
    { name = "httpx" },
    { name = "uvloop", marker = "sys_platform != 'win32'" },
]

[package.metadata]
requires-dist = [
    { name = "httpx", specifier = ">=0.27" },
    { name = "uvloop", marker = "sys_platform != 'win32'" },
]

[[package]]
name = "httpx"
version = "0.28.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [{ name = "anyio" }, { name = "certifi" }, { name = "httpcore" }, { name = "idna" }]
sdist = { url = "https://files.pythonhosted.org/.../httpx-0.28.1.tar.gz", hash = "sha256:75e9…", size = 141406, upload-time = "2024-12-06T15:37:23Z" }
wheels = [
    { url = "https://files.pythonhosted.org/.../httpx-0.28.1-py3-none-any.whl", hash = "sha256:d909…", size = 73517, upload-time = "2024-12-06T15:37:21Z" },
]
```

Key points:
- **Every artifact is pinned with a hash.** That covers the sdist and *all* candidate wheels for all platforms, not just yours. That is why lockfiles for packages like numpy or torch are long.
- **`source` records where each package came from:** a registry URL, `git = "…?rev=…#<commit>"`, a `path`, `editable`, or `virtual` (a workspace member that isn't itself installed).
- **`[package.metadata]` stores the *input* requirements.** This is how uv detects that `pyproject.toml` changed and the lock is stale.
- **Your own project and workspace members** (a monorepo of several packages) all go into **one** shared lockfile.

### The lifecycle: lock, then sync, then run
- `uv lock` resolves and writes `uv.lock`. It **prefers versions already in the lock**, so re-locking after an edit doesn't silently upgrade everything.
- `uv sync` makes `.venv` match the lock **exactly**. Packages not in the lock are removed; `--inexact` keeps them.
- `uv run <cmd>` does an automatic lock-if-needed plus sync, then runs the command. That is why `uv run pytest` "just works" after a `git pull`.
- `uv add httpx` / `uv remove httpx` edit `pyproject.toml`, re-lock and sync in one step. Use `--dev` or `--group docs` for dependency groups (PEP 735) and `--optional extra` for extras.

### Upgrades are always explicit
New releases on PyPI never invalidate your lock.
- `uv lock --upgrade` upgrades everything within your constraints.
- `uv lock --upgrade-package requests` upgrades one package, and `--upgrade-package requests==2.32.3` targets a specific version. The rest stays put.

### CI and production flags

| Flag | Meaning | Use it when |
|---|---|---|
| `--locked` | Error if `uv.lock` is out of date with `pyproject.toml` | **CI**: catches "forgot to commit the lock" |
| `--frozen` | Use the lock as-is, don't check freshness | Docker builds and deploys where `pyproject.toml` might not even be present |
| `--no-sync` | Don't touch the environment | Repeated `uv run` calls in a tight loop |
| `uv lock --check` | Exit nonzero if the lock is stale | Pre-commit hook or CI gate |

### Reproducibility controls
- **`exclude-newer`** (setting or `--exclude-newer`) only considers files uploaded before a cutoff. It accepts a timestamp (`2026-10-01T00:00:00Z`) or a relative "cooldown" (`7 days`, `P7D`). A cooldown is a cheap supply-chain defence: you never install a release in its first week, which is when most malicious uploads get caught and yanked.
- **`--resolution lowest-direct`** installs the *lowest* allowed versions of your direct dependencies. Library authors should run it in CI to prove their lower bounds are real.
- **Overrides and constraints:** `override-dependencies` forcibly replaces a transitive requirement, including loosening someone else's bad upper bound. `constraint-dependencies` narrows versions without adding packages.

### Interop: exporting
- `uv export --format requirements.txt` gives pinned and hashed output for legacy tooling and some Dockerfiles.
- `uv export --format pylock.toml` writes the standard **PEP 751** lockfile. uv can also *install* from one: `uv pip sync pylock.toml` or `uv pip install -r pylock.toml`.
- `uv export --format cyclonedx1.5` produces an SBOM.

`uv.lock` remains the format for uv's own project interface. `pylock.toml` is the interchange format.

---

## 5. The pip-compatible interface (`uv pip …`)

### Purpose
The pip interface is a migration path and an escape hatch. Existing workflows (`requirements.txt`, `pip-tools`, scripts in Dockerfiles) mostly keep working if you put `uv` in front of them:

```bash
uv venv                                  # creates .venv (no pip inside it by default)
uv pip install -r requirements.txt       # like pip install
uv pip install -e ".[dev]"               # editable install with extras
uv pip compile requirements.in -o requirements.txt            # like pip-compile
uv pip compile requirements.in --universal -o requirements.txt # cross-platform output
uv pip sync requirements.txt             # like pip-sync: exact match, removes extras
uv pip list / freeze / show / tree / check / uninstall
```

There is no lockfile or `pyproject.toml` management here, and nothing is "owned". You are imperatively mutating an environment, exactly as with pip. But you still get the fast resolver, the global cache and linking.

### The differences that bite
"pip-compatible" means *familiar*, not *identical*. These are the important deviations, roughly in order of how often they surprise people:

1. **It requires a virtual environment.** uv installs into the active venv or a `.venv` it finds in the current or a parent directory. It won't touch the system Python unless you pass `--system`, or `--python /path/to/python` to target a specific interpreter. `--user` is not supported.
2. **It ignores pip's configuration.** `pip.conf` and `PIP_INDEX_URL`-style variables are ignored. Use `UV_INDEX_URL` / `UV_DEFAULT_INDEX`, `uv.toml`, or `[tool.uv.pip]` in `pyproject.toml`.
3. **It uses one index per package (the `first-index` strategy).** With several indexes configured, uv takes a package's versions only from the **first** index that has it. pip merges all indexes and picks the best version. uv's behaviour is deliberate: it blocks "dependency confusion" attacks, where someone uploads `your-internal-pkg` v99 to PyPI. `--index-strategy unsafe-best-match` restores pip's behaviour.
4. **It skips bytecode compilation.** First imports are slightly slower. Use `--compile-bytecode` / `UV_COMPILE_BYTECODE=1` in Docker images and for CLI startup time.
5. **Build isolation is on by default.** If a package needs `torch` present while it builds (flash-attn and many CUDA extensions), you need `--no-build-isolation`, or better, `[tool.uv] no-build-isolation-package = ["flash-attn"]` plus `extra-build-dependencies`.
6. **`-c` constraints don't apply to build dependencies.** Use `--build-constraint` for those.
7. **Pre-releases resolve as `if-necessary`.** Stable versions are preferred, and a pre-release is chosen only when nothing else satisfies the constraints or you explicitly asked for one.
8. **It is stricter.** uv rejects wheels whose filenames don't match their metadata, `.egg` distributions, some malformed indexes, and (since 0.12) non-PEP 625 sdists other than `.zip`. It ignores `requires-python` *upper* bounds.
9. **`uv pip compile` behaves a bit differently from pip-compile.** It needs `-o` to write a file, and it doesn't emit index URLs unless you pass `--emit-index-url`.
10. **Some options are missing,** such as `--trusted-host`. Keyring authentication works only via `--keyring-provider subprocess`.
11. **Resolution order can matter more than in pip.** uv and pip can legitimately pick different (both valid) solutions. Tighter specifiers make the result deterministic.

### Mixing the two interfaces
Never `uv pip install foo` into a *project's* `.venv` and expect it to stick. `pyproject.toml` and `uv.lock` don't know about `foo`, and the next `uv sync` (or `uv run`, which syncs) will remove it, because sync is exact. In a project, use `uv add foo`, or `uv run --with foo …` for a one-off.

---

## 6. The rest of the toolbox

### Tools (the pipx replacement)
- `uvx ruff check .` runs a tool in a cached, ephemeral environment. The second run is instant.
- `uvx --from 'httpie==3.2.4' http …` pins the version, and `uvx --python 3.12 …` picks the interpreter.
- `uv tool install ruff` installs a tool permanently, with executables in `~/.local/bin`. `uv tool upgrade --all` upgrades every installed tool.

### Scripts (PEP 723)
```python
#!/usr/bin/env -S uv run --script
# /// script
# requires-python = ">=3.12"
# dependencies = ["httpx", "rich"]
# ///
import httpx, rich
...
```
- `uv add --script foo.py httpx` edits that metadata block for you.
- `uv lock --script foo.py` writes a `foo.py.lock` next to the script, which makes a one-file tool reproducible.
- `chmod +x` the file and it runs anywhere uv is installed, with no venv to manage.

### Projects and workspaces
- `uv init` creates an app. Add `--lib` for an installable library or `--package` for a packaged app.
- `uv build` produces an sdist and wheel. With `uv_build` as the build backend this is very fast for pure-Python packages.
- `uv publish` replaces `twine` and supports trusted publishing from GitHub Actions.
- `uv version --bump minor` bumps the project version.
- **Workspaces:** `[tool.uv.workspace] members = ["packages/*"]` gives a Cargo-style monorepo with one lockfile and one `.venv`. Members can depend on each other via `{ workspace = true }` sources.

### Sources and indexes (essential for PyTorch and CUDA)
`[tool.uv.sources]` overrides *where* a dependency comes from, separately from *which versions* are allowed:

```toml
[project]
dependencies = ["torch>=2.8", "mylib"]

[tool.uv.sources]
torch = { index = "pytorch-cu128" }
mylib = { git = "https://github.com/me/mylib", tag = "v1.2.0" }

[[tool.uv.index]]
name = "pytorch-cu128"
url = "https://download.pytorch.org/whl/cu128"
explicit = true   # only used for packages that name it, so numpy etc. still come from PyPI
```

For quick work in the pip interface, `uv pip install torch --torch-backend=auto` (a preview feature) detects the local CUDA driver and picks the matching PyTorch index.

---

## 7. A Docker pattern that uses all of this

This pattern gives slim layers, a cache that survives rebuilds, and an exact lock:

```dockerfile
FROM python:3.12-slim
COPY --from=ghcr.io/astral-sh/uv:0.13.7 /uv /uvx /bin/
ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy UV_PYTHON_DOWNLOADS=never
WORKDIR /app

# 1) deps only: this layer is cached until the lockfile changes
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --locked --no-install-project --no-dev

# 2) your code
COPY . .
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --locked --no-dev

CMD ["uv", "run", "--no-sync", "python", "-m", "myapp"]
```

What each piece does:
- **`UV_LINK_MODE=copy`:** the cache mount is a different filesystem from the image, so copying is expected. Setting it explicitly silences the warning.
- **The cache mount:** the uv cache never ends up inside the image, which keeps the image small, but it speeds up every rebuild.
- **`--locked`:** the build fails if someone changed `pyproject.toml` without re-locking.
- **The two-step sync:** source edits don't invalidate the dependency layer.

---

## 8. A mental model to keep

```
             pyproject.toml  (what you want: ranges, sources, indexes)
                    │  uv lock   (PubGrub, universal, forks on markers)
                    ▼
               uv.lock       (what you get, for every platform, hashed)
                    │  uv sync   (select this platform's subset, exact)
                    ▼
   global cache ──link──▶ .venv  (clone / hardlink / copy from the unpacked wheels)
        ▲
        └── downloads & builds happen once per machine, shared by all projects, tools, scripts

   uv pip …  = same resolver + cache + linker, minus pyproject/lock ownership
```

Rules of thumb:
- **New work:** use the project interface. Commit `uv.lock`. Run `uv sync --locked` in CI and `--frozen` in deploys.
- **Legacy `requirements.txt` repos:** prefix with `uv` (`uv pip …`). Migrate later with `uv init` plus `uv add -r requirements.txt`.
- **Disk:** the cache *saves* space through deduplication and links, but it grows forever. Run `uv cache prune` occasionally and `uv cache prune --ci` before saving CI caches.
- **"Why did my package disappear?"** Something ran `uv sync` or `uv run` against a lock that doesn't include it.
- **"Why is install slow on this box?"** The cache and venv are on different filesystems, so uv fell back to copying.
- **"Why did it pick the wrong torch?"** Check indexes, `explicit = true`, and the `first-index` strategy.

## 9. Context: who maintains uv
Astral (founded by Charlie Marsh) also makes Ruff (a linter and formatter) and ty (a type checker). uv is open source under MIT/Apache-2.0. On 2026-03-19, OpenAI announced it would acquire Astral and fold the team into Codex, and pledged to keep uv, Ruff and ty open source. At the time the deal still needed regulatory approval. I haven't checked whether it has since closed. Development has continued at a rapid pace: 0.13.0 shipped in October 2026 and made Python 3.15 the default stable version.

---

### Sources
- [uv docs: Caching](https://docs.astral.sh/uv/concepts/cache/)
- [uv docs: Locking and syncing](https://docs.astral.sh/uv/concepts/projects/sync/)
- [uv docs: Project structure and files (uv.lock, pylock.toml)](https://docs.astral.sh/uv/concepts/projects/layout/)
- [uv docs: Resolution](https://docs.astral.sh/uv/concepts/resolution/)
- [uv docs: Compatibility with pip](https://docs.astral.sh/uv/pip/compatibility/)
- [uv docs: Python versions](https://docs.astral.sh/uv/concepts/python-versions/)
- [uv docs: Settings reference (link-mode, exclude-newer, python-preference)](https://docs.astral.sh/uv/reference/settings/)
- [uv GitHub releases](https://github.com/astral-sh/uv/releases)
- [Simon Willison: OpenAI acquiring Astral (2026-03-19)](https://simonwillison.net/2026/Mar/19/openai-acquiring-astral)
- [pydevtools: OpenAI acquires Astral](https://pydevtools.com/blog/openai-acquires-astral/)
- [OpenAI: OpenAI to acquire Astral](https://openai.com/te-IN/index/openai-to-acquire-astral/)
