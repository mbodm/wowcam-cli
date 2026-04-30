# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

`wowcam` is a Python async CLI that reads `./wowcam.xml` and, for every CurseForge addon page URL listed under the active profile, concurrently fetches the addon's zip via the **wowcam-cache** API (https://github.com/mbodm/wowcam-cache) and extracts it into the profile's target folder. End users receive a single-file Nuitka-built executable plus a `wowcam.xml` sample configuration; nothing else.

Greenfield; no source yet. The decisions below are load-bearing — do not change them without checking with the user.

## Stack

- Python: track the **latest stable release** — bump `requires-python` in `pyproject.toml` as new versions land. `async`/`await` throughout.
- `httpx` for streamed async HTTP; stdlib `zipfile` via `asyncio.to_thread` for extraction (no extra async-zip dep)
- **uv** manages venv and deps; metadata in `pyproject.toml`; package at `src/wowcam/`; entry point `python -m wowcam`
- Type hints everywhere, but **no static type checker** (no mypy / pyright / ty) — rely on hints + IDE feedback
- Ruff with **defaults** — no `[tool.ruff]` section unless a concrete rule is needed
- No tests, no test framework

## Common commands

```bash
uv sync                          # install / refresh deps
uv run python -m wowcam          # run CLI from source (reads ./wowcam.xml)
uv run ruff format               # format
uv run ruff check --fix          # lint with autofix
```

Nuitka one-file builds (run on the matching host):

```bash
# Windows x64
uv run nuitka --onefile --assume-yes-for-downloads --include-distribution-metadata=wowcam -o wowcam.exe src/wowcam/__main__.py

# macOS arm64
uv run nuitka --onefile --macos-create-app-bundle=no --include-distribution-metadata=wowcam -o wowcam src/wowcam/__main__.py
```

## Configuration (`wowcam.xml`)

The CLI reads `./wowcam.xml` from the current working directory. Sample at the repo root: `wowcam.xml.sample`. Two top-level sections under `<wowcam>`:

- **`<general>`** — `<profile>` (name of the active profile), `<token>` (wowcam-cache API token), and optional `<temp>` (a *parent root* directory; wowcam creates an owned subfolder inside it — see `__main__.py`).
- **`<profiles>`** — one child per profile, named after the profile (e.g. `<retail>`, `<classic>`); each contains `<folder>` (the target unzip folder) and `<addons>` with one or more `<url>` entries pointing at CurseForge addon pages (e.g. `https://www.curseforge.com/wow/addons/raiderio`). Profiles let one config switch between WoW flavors by changing `<general><profile>`.

**Path values support env-var expansion** via `os.path.expandvars` — `%VAR%` syntax expands on Windows, `$VAR` / `${VAR}` on POSIX. The Windows-flavored sample uses `%TEMP%` and `%PROGRAMFILES(X86)%`; macOS users would use `$TMPDIR` / hardcoded paths in their own XML.

## Architecture

Four single-responsibility modules under `src/wowcam/`. **Do not blur these boundaries** — the leaf modules (`config`, `download`, `unzip`) stay pure: no FS lifecycle, no progress prints, no CLI concerns.

### `config.py`

Parses `./wowcam.xml` and returns a config object holding: profile name, target folder (`Path`), optional temp dir (`Path | None`), API token (`str`), and addon names (`list[str]` — the addon slug from each Curseforge `<url>`).

- **Public API**: single function `load()` returning the config object described above. Sync (XML parse is small and fast — no `asyncio.to_thread` needed); `__main__.py` calls it without `await`.
- Expands env vars in path values via `os.path.expandvars`
- **Paths in the returned config object are always absolute.** After env-var expansion, any relative path is resolved against CWD (e.g. `Path(...).resolve()`). Downstream modules and `__main__.py` can rely on this — they never see a relative path.
- **Strict URL validation**: every `<url>` must match `https://www.curseforge.com/wow/addons/<slug>` (trailing slash tolerated); the slug is the path segment after `/wow/addons/`; anything else fails validation
- **Required**: file exists, well-formed XML, `<general><profile>` set, `<general><token>` present and non-empty, the named profile defined under `<profiles>` with a `<folder>` and an `<addons>` element. A missing `<addons>` is a structural (file-format) validation failure. An *empty* `<addons>` is **not** a `config.py` failure — it returns a valid config object with an empty addon-names list, which `__main__.py` checks and reports separately. Slugs unique within that profile.
- **Optional**: `<general><temp>` — when absent or empty, `__main__.py` falls back to `Path.cwd() / "temp-downloads"`
- Any validation failure raises `ConfigError` with a plain-language message (chained via `from e` where an underlying exception exists, e.g. `xml.etree.ElementTree.ParseError`)
- **No FS mutation**

### `download.py`

The **only** module that touches HTTP. Single coroutine: `async def download(addon_name: str, token: str, dest: Path)`. wowcam-cache is itself a caching proxy in front of a separate scraper API — full chain: wowcam → wowcam-cache → wowcam-scraper → CurseForge. Internally `download` does two GETs:

1. **Resolve.** `GET https://wowcam.mbodm.deno.net/get?addon={name}&token={token}` (URL-encode both). On success the response is a flat JSON object — only the top-level `downloadUrl` is used; all other fields are ignored. Example:

   ```json
   {
     "addonSlug": "raiderio",
     "downloadUrl": "https://mediafilez.forgecdn.net/files/8013/820/RaiderIO-v202604300600.zip",
     "scrapedAt": "2026-04-30T18:00Z",
     "fromCache": true,
     "infoMessage": "Used addon from cache (1h max age)",
     "statusInfo": "HTTP 200 OK"
   }
   ```

   Status-code mapping: `401`/`403` → invalid or missing token; `502` → wowcam-cache could not reach the wowcam-scraper API; `507` → wowcam-cache is full (KV entry cap reached); anything else non-2xx or a missing `downloadUrl` → "unexpected response from wowcam-cache".
2. **Download.** `GET` the `downloadUrl` (may redirect — CurseForge CDN) and write the response body to `dest`. Buffering the full zip in RAM is fine for this single-user CLI.

Other rules:

- Module-level lazy-init `httpx.AsyncClient` reused across calls (no per-call construction, no explicit shutdown — process exit cleans up); rely on httpx defaults for timeouts and max-redirect cap; set `follow_redirects=True`
- `CACHE_BASE_URL = "https://wowcam.mbodm.deno.net/get"` is a module constant, not user-configurable
- **No retries**, no prints, no addon-name parsing (config does that), no FS lifecycle beyond writing `dest`

### `unzip.py`

Single coroutine: `async def unzip(zip_path: Path, target: Path)` — stdlib `zipfile` via `asyncio.to_thread`.

- **Hard contract**: extracted content lands *only* inside `target`, no exceptions (zip-slip defense)
- For each entry, validate the resolved path stays inside `target` *before* writing. On any escape, reject the whole archive (raise) and extract nothing.
- **Nothing else**

### `__main__.py`

Orchestrator. Calls `config.load()`, manages the temp dir lifecycle, defines `process_addon`, fans it out, prints `###`, handles errors. **Never imports `httpx`** — main never sees the HTTP layer; it just calls `download(...)` and `unzip(...)`.

- **Entry shape**: sync `main() -> int` wraps an async `_run()` via `asyncio.run`. `main()` catches `KeyboardInterrupt` (prints `interrupted` to stderr, returns `130`) so Ctrl-C never produces a traceback.
- **CLI flags**: `--help` prints a brief usage that makes clear the tool takes no arguments and always reads `./wowcam.xml` from CWD, then exits `0`. `--version` prints the version and exits `0`. Any other argument fails with usage on stderr and exits `2`. With no arguments, the tool runs normally.
  - **Version source**: single source of truth is `[project].version` in `pyproject.toml`. Code reads it via `importlib.metadata.version("wowcam")` — no duplicated `__version__` constant. For Nuitka one-file builds this requires `--include-distribution-metadata=wowcam` in the build command (already shown above) so the metadata is bundled into the executable.
- **Empty addon list**: after `config.load()`, if the returned addon-names list is empty (well-formed XML with `<addons></addons>` and no `<url>` children), print a plain-language error to stderr (e.g. "no URLs in XML config file") and exit `1`. No temp dir is created, no fan-out happens. This is a `__main__.py` concern, not a `config.py` one.
- **Effective temp dir** is a wowcam-owned subfolder (named `wowcam-downloads` under the configured `<temp>`, or `./temp-downloads` if `<temp>` is unset). The parent root is **never** touched.
  - **Why a subfolder**: `<temp>` may legally point at a system-wide path like `%TEMP%` (`C:\Users\<name>\AppData\Local\Temp`); wiping that root would destroy other apps' state. The owned subfolder isolates wowcam's lifecycle from anything else.
- **Temp-dir lifecycle** (operates only on the subfolder, never its parent): wipe contents at start (or create if missing); remove the subfolder entirely at exit, on success *and* on error.
- **Target unzip folder**: at start, **always wipe** its contents if non-empty; create if missing. Every run begins with a clean target — the tool owns this folder fully and any pre-existing files are removed.
- **Per-addon flow**: build `dest = temp_dir / f"{name}.zip"`, then `await download(name, token, dest)`, then `await unzip(dest, target_folder)`.
- **Combined task** `process_addon(name)`: download → print `###` → unzip → print `###`. Two `###` per successful addon. Returns `None`; on failure it does not catch — exceptions propagate and are collected by `gather(..., return_exceptions=True)` for the failure summary below.
- **Fan-out is best-effort**: one failed addon must not abort the rest. After fan-out completes, print a short failure summary to **stderr** (one line per failure: `<addon-name>: <plain-language reason>`); exit `1` if any failed, else `0`. **No concurrency cap.**
- Top-level `try`/`except` wraps the whole thing for the user-friendly error messages described below — this catches **setup** errors (e.g. "wowcam.xml not found", "profile X not defined", "token missing"); per-addon failures are handled by the gather summary above, not here.

## Progress UI

The only stdout output during normal operation is `###` (one per successful download, one per successful unzip).

## Error handling & exit codes

- Each leaf module defines its own exception type — `ConfigError` in `config.py`, `DownloadError` in `download.py`, `UnzipError` in `unzip.py`. On failure they raise that type with a plain-language message and chain the real exception via `raise XError("...") from e`. The original is preserved so logging can be added to `main()` later without losing it.
- `main()` differentiates by type to print a single user-friendly line to **stderr**: `ConfigError` (and other setup failures) is caught by the top-level `try`/`except` around orchestration; `DownloadError` / `UnzipError` arrive as exception values inside the per-addon `gather(..., return_exceptions=True)` results and are inspected by type in the failure summary.
- Never print tracebacks. Never write a log file. Never show users `httpx.ReadTimeout` / `zipfile.BadZipFile` / etc. directly — leaf modules translate those into plain language inside the `XError` they raise.
- All user-facing strings (errors, future status text) are in **English**.
- Exit codes follow Unix convention: `0` on success, `2` for CLI/usage errors, `130` for SIGINT (Ctrl-C), `1` otherwise.
