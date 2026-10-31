# Repository Guidelines

## Project Structure & Module Organization

`swarmchat/` contains the Python server and client: `server.py` handles HTTP, `api.py` implements requests, `store.py` manages persistence, `policy.py` defines permissions, and `cli.py` exposes commands. `files.py`, `config.py`, and `view.py` handle attachments, settings, and the viewer. Browser HTML, CSS, and JavaScript live in `swarmchat/page/`. Keep modules focused and runtime dependencies limited to Python’s standard library.

`tests/` holds integration and regression tests. `bench/` contains benchmarks and `RESULTS.md`; `docs/` contains images. Consult `DESIGN.md` for architecture.

## Build, Test, and Development Commands

Run commands from the repository root with Python 3.9 or newer:

- `python3 -m pip install -e .` — install an editable checkout and the `swarmchat` command.
- `python3 -m swarmchat --help` — inspect CLI commands without installation.
- `python3 -m swarmchat serve --data /tmp/swarmchat-dev` — start a local server at `http://127.0.0.1:8484` with isolated data.
- `python3 -m unittest discover -s tests -v` — run the full test suite.
- `python3 -m unittest discover -s tests -p 'test_transport.py' -v` — run transport tests.
- `python3 -m pip wheel . --no-deps -w dist` — build a distribution wheel.

The viewer needs no frontend build step; preserve its assets in package data.

## Coding Style & Naming Conventions

Use four-space indentation in Python, `snake_case` functions and variables, `PascalCase` classes, and uppercase constants. Match the existing two-space JavaScript indentation and semicolons. Keep docstrings concise and explain concurrency, persistence, and permission invariants. No formatter or linter is configured; follow nearby code and maintain Python 3.9 compatibility.

## Testing Guidelines

Use standard-library `unittest`, naming files `test_*.py` and methods `test_<behavior>`. Exercise real servers with temporary data directories and dynamically assigned ports; clean up clients, threads, and subprocesses. Add regression tests for changed behavior, especially authorization, storage recovery, transport, and viewer safety. CI tests Python 3.9 and 3.13 on Linux, macOS, and Windows, and verifies packaged viewer assets. No numerical coverage threshold is configured.

## Commit & Pull Request Guidelines

History currently contains only `Initial commit`; no detailed message convention is established. Use concise imperative subjects. Describe the change, rationale, test results, and linked issues in PRs; include screenshots for viewer changes. Run Git pushes through `with-proxy git push`. Record progress timestamps in PT.

## Security & Configuration

Keep tokens and runtime data out of commits. Use `SWARMCHAT_SERVER` and `SWARMCHAT_TOKEN` for agent configuration. Preserve authentication before request-body reads and escaping of agent content in the viewer.
