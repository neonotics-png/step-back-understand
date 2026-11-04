# uv comprehension quiz

1. A colleague says "uv is fast because it's written in Rust." Why is that explanation incomplete? Name the main non-language reasons uv can resolve and install faster than pip, and explain how uv can often learn a package's dependencies without downloading the whole wheel.

2. uv keeps one global cache, yet a warm install of a large environment finishes in milliseconds and takes almost no extra disk space. How does it do this? What happens when the cache and the virtual environment are on different filesystems, and why is the `symlink` link mode discouraged?

3. uv ships as a single static binary with no dependency on Python. Why does that matter in practice compared with pip, which runs inside an interpreter? Include how uv gets a Python interpreter when the machine has none, or the wrong version.

4. `uv pip compile requirements.in` and `uv lock` both "pin" dependencies, but they produce different kinds of output. What is the key difference? Why might `uv.lock` list the same package twice at different versions, and why can a project with `requires-python = ">=3.8"` fail to lock when one dependency needs Python 3.10 or later, even though you only ever run 3.12?

5. Explain the difference between `uv sync --locked` and `uv sync --frozen`. When would you use each one, for example in CI versus in an early Docker layer? Also, does a new release of a dependency on PyPI make an existing `uv.lock` "outdated"?

6. In a uv project, a developer runs `uv pip install rich` to try something out. The next `uv sync` removes `rich` from the `.venv`. Explain why, and describe how uv's two interfaces (the `uv pip` interface and the project interface) differ in philosophy. What should the developer have done instead?

7. A company hosts an internal package `acme-utils` on a private index and also uses PyPI. An attacker publishes `acme-utils` version 99.0 on PyPI. What does pip do with `--extra-index-url`, and what does uv do by default? Explain the reasoning behind uv's default and the setting that changes it.

8. After switching a Docker image from `pip install` to `uv pip install`, install time drops sharply, but the container's first request after startup is noticeably slower than before. What is the most likely cause, why did uv choose this default, and how would you fix it?
