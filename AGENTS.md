# Repository Guidelines

## Project Overview

pdf2htmlEX renders PDF files into HTML using modern web technologies: native HTML text with precise font and positioning, flexible output (single-file HTML or on-demand page loading), links, outlines, printing, SVG backgrounds, and Type 3 font support.

This checkout (`cutlassfish`) is a community fork of upstream pdf2htmlEX kept alive for open collaboration. Fork-specific behavior, per `README.md`:

- "Out of source building"
- "Integration of latest Cairo code"
- "Rewritten handling of obscured/partially obscured text - now much more accurate" — `--correct-text-visibility` samples 4 inset points per character bbox; mode `1` = fully occluded text dropped from the HTML layer (default), mode `2` = partially occluded text moved to the rasterized background layer at `--covered-text-dpi` (default `300`).
- "Some support for transparent text"; DPI clamped so output graphics do not blow up.
- README recommends `--font-size-multiplier 1 --zoom 25` plus an external `scale` transform for maximum positional accuracy (avoids browser rounding errors).

License: GPLv3+ for the package as a whole; `LICENSE` grants relaxed terms for some resource files. Full docs live on the [wiki](https://github.com/pdf2htmlEX/pdf2htmlEX/wiki); `INSTALL` is a stub pointing there.

## Architecture & Data Flow

Single-threaded C++ application (no threads, no mutexes, no async anywhere in `pdf2htmlEX/src/`). It is a **Poppler `OutputDev`** that reuses fontforge internals for font dumping, so the build statically links pinned Poppler + FontForge versions (see `buildScripts/Readme.md`).

Pipeline in `pdf2htmlEX/src/pdf2htmlEX.cc` (`main()`):

1. `ArgParser` (hand-rolled `getopt_long` wrapper, not argparse/boost) fills the single global `param` struct (`Param.h`).
2. `check_param()` → `prepare_directories()` (`mkdtemp` under `param.tmp_dir`) → `create_directories(param.dest_dir)` → `setupSignalHandler()`.
3. `globalParams = std::make_unique<GlobalParams>(...)`; `PDFDocFactory().createPDFDoc(fileName, ownerPW, userPW)`.
4. `std::unique_ptr<HTMLRenderer>(new HTMLRenderer(argv[0], param))->process(doc.get())` — no `delete`, output is written during `process()`.

`HTMLRenderer::process()` is a two-pass walk:

- **Pass 1 — `Preprocessor`** (`Preprocessor.h/.cc`, a second `OutputDev` with `useDrawChar=true`): collects used glyph codes per font (`get_code_map()`) and max page width/height.
- **Background rendering** — `BackgroundRenderer::getBackgroundRenderer(param.bg_format, this, param)` plus `getFallbackBackgroundRenderer()`. Concrete strategies: `SplashBackgroundRenderer` (PNG/JPG) and `CairoBackgroundRenderer` (SVG/bitmap). Selected by `param.bg_format`.
- **Pass 2 — per-page loop** (`for(int i = param.first_page; i <= param.last_page; ++i)` + `doc->displayPage(...)`), dispatching to the `HTMLRenderer` hooks.

Data flow and where it lands:

| Source | Path through code | Output |
|---|---|---|
| Text (`drawString`) | `HTMLRenderer/text.cc` → `HTMLTextLine` → `HTMLTextPage` → state diffing via `StateManager.h` | `param.f_pages` (`pages.fs`), CSS classes deduped by `AllStateManager` |
| Fonts (`GfxFont`) | `install_font` → `install_embedded_font`/`install_external_font` → `dump_embedded_font`/`dump_type3_font` → `ffw.*` (FontForge via C ABI) → `export_remote_font`/`export_local_font` | `<dest-dir>/*.{woff,ttf,otf}`, `@font-face` CSS in `f_css.fs` |
| Images/drawings | `image.cc` / `draw.cc` → `BackgroundRenderer` → `render_page`/`embed_image` | background PNG/JPG/SVG files |
| Links | `link.cc` (`processLink`, `get_linkaction_str`) | anchors in pages output |
| Outline | `outline.cc` | `f_outline.fs` |
| Forms | `form.cc` (`process_form`) | form HTML/CSS |
| Covered text | `DrawingTracer` callbacks (`on_char_drawn`, `on_char_clipped`, `on_non_char_drawn`) → `CoveredTextDetector` → `HTMLRenderer::is_char_covered` | text excluded from / moved into layers |

Outputs are held as `struct { std::ofstream fs; std::string path; } f_outline, f_pages, f_css;` plus `std::ofstream * f_curpage`. Glyph→font metadata lives in `std::unordered_map<long long, FontInfo> font_info_map`; `FontInfo` carries `id, use_tounicode, em_size, space_width, ascent, descent, is_type3, font_size_scale`.

**Key extension points**: adding an output backend means subclassing `BackgroundRenderer`; adding PDF feature handling means adding/overriding an `HTMLRenderer` hook in the matching `.cc` under `src/HTMLRenderer/` (files are split by concern, not by class).

## Key Directories

| Path | Purpose |
|---|---|
| `pdf2htmlEX/src/` | Core C++ sources: entry point, CLI, text/state model, preprocessors. |
| `pdf2htmlEX/src/HTMLRenderer/` | `HTMLRenderer` (Poppler `OutputDev`) implementation split by area: `general.cc`, `state.cc`, `text.cc`, `font.cc`, `image.cc`, `draw.cc`, `link.cc`, `outline.cc`, `form.cc`. |
| `pdf2htmlEX/src/BackgroundRenderer/` | Background rasterization strategies (`Splash*`, `Cairo*`) behind the `BackgroundRenderer` interface. |
| `pdf2htmlEX/src/util/` | Shared helpers: `namespace.h`, `const.*`, `math.*`, `unicode.*`, `encoding.*`, `path.*`, `misc.*`, `mingw.*`, `ffw.[ch]` (FontForge C ABI), `SignalHandler.*`, `css_const.h.in`. |
| `pdf2htmlEX/share/` | Runtime data shipped with the binary: `*.in` templates for CSS/JS, `manifest`, `LICENSE`, `pdf2htmlEX-64x64.png`, `build_css.sh`, `build_js.sh`. Generated `base.css`, `fancy.css`, `pdf2htmlEX.js` and `.min.*` are gitignored. |
| `pdf2htmlEX/3rdparty/` | Vendored minifiers and shims: `yuicompressor/yuicompressor-2.4.8.jar`, `closure-compiler/compiler.jar`, `PDF.js/compatibility.js`. |
| `pdf2htmlEX/test/` | Python 3 unittest output tests + Selenium/Firefox browser tests + fixtures. See Testing & QA. |
| `buildScripts/` | Authoritative build/package/release shell scripts (`buildInstallLocallyApt`, `getPoppler`, `createImagesApt`, `uploadImages`, …). Docs: `buildScripts/Readme.md`. |
| `docker/` | `Dockerfile.trixie` — the only container definition. Builds this checkout (Debian trixie + bookworm-incompatible Poppler deps); see `.dockerignore`. |
| `patches/` | FontForge patches (currently not applied; see Runtime/Tooling Preferences). |
| `archive/` | Legacy Debian/PPA packaging (`debian/`, `createDebianPackage`, `build_for_ppa.py`); dead relative to `buildScripts/`. |

## Development Commands

Primary supported entry point (Linux; `buildScripts/Readme.md` "strongly encourages" using these scripts because the source depends on Poppler/FontForge internals):

```sh
./buildScripts/buildInstallLocallyApt     # fetch + build static deps + build + install
./buildScripts/buildInstallLocallyAlpine  # experimental
./buildScripts/createImagesApt            # AppImage + .deb (+ container image)
./buildScripts/runTests                   # test deps + output tests + browser tests
./buildScripts/uploadImages               # reportEnvs -> uploadGitHubRelease -> uploadContainerImage
./buildScripts/travisLinuxDoItAll         # full CI pipeline
```

Underlying CMake invocation (`buildScripts/buildPdf2htmlEX`, out-of-source only — `pdf2htmlEX/build/` is gitignored):

```sh
cd pdf2htmlEX
rm -rf build; mkdir build; cd build
cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=$PDF2HTMLEX_PREFIX ..
make $MAKE_PARALLEL          # MAKE_PARALLEL="-j $(nproc)"
```

Install:

```sh
cd pdf2htmlEX/build && sudo make install
# poppler-data: make install prefix=$PDF2HTMLEX_PREFIX datadir=$PDF2HTMLEX_PREFIX/share/pdf2htmlEX
```

Docker (builds this checkout, not a clone of upstream):

```sh
docker build -f docker/Dockerfile.trixie -t pdf2htmlex:trixie .
docker run --rm pdf2htmlex:trixie --version
```

Typical manual run (as the tests invoke it):

```sh
pdf2htmlEX --data-dir <share/pdf2htmlEX> --dest-dir <out> input.pdf
```

Root has **no** `CMakeLists.txt`; the only one is `pdf2htmlEX/CMakeLists.txt`.

## Code Conventions & Common Patterns

**C++ (C++20)**
- Everything lives in `namespace pdf2htmlEX`; `src/util/namespace.h` hoists std names (`string`, `endl`, …) into scope.
- Header guards are macro-based `#ifndef X_H__`; `#pragma once` is never used.
- Classes `CamelCase` (`HTMLRenderer`, `CairoBackgroundRenderer`); own methods/functions `snake_case` (`dump_css`, `install_font`, `check_state_change`). Poppler overrides keep Poppler casing (`startPage`, `drawString`).
- Members have **no trailing underscore** (`param`, `cur_tx`, `font_info_map`, `inTransparencyGroup`); `AllStateManager` state fields are `lower_snake` (`transform_matrix`, `font_size`).
- Config and managers are stored as `const Param & param;` / `AllStateManager & all_manager;` references, not copies; accessors are marked `const` (`text_zoom_factor() const`, `print_scale() const`).
- Ownership: `std::unique_ptr` (`bg_renderer`, `fallback_bg_renderer`, `doc`, `arg_entries`), `std::shared_ptr` for `FormPageWidgets`; raw pointers only in C-ish glue.
- Errors: `throw "Cannot read the file"` / `throw string("Cannot open ") + fn` (yes: `const char*` throws); `ArgParser` throws `std::string`. `main()` catches, prints `Error: ...`, and fatal paths call `exit(EXIT_FAILURE)` — there is no exception hierarchy, so match local style rather than inventing one.
- Logging: ad-hoc `cerr`/`printf` gated by `param.quiet` / `param.debug`; `text.cc` has a local `#define HR_DEBUG(x)` compiled out by default. Do not add a logging framework.
- Include order in `.cc`: own header, then std, then Poppler (`<OutputDev.h>`, `<GfxState.h>`), then project (`"pdf2htmlEX-config.h"`, `"util/..."`).
- Build flags: `-Wall -Woverloaded-virtual`; Release `-O2 -DNDEBUG`, Debug `-ggdb -pg`.

**Shell (`buildScripts/`, `pdf2htmlEX/test/`)**
- POSIX `sh`, `#!/bin/sh`, `set -ev`, each script sources `./buildScripts/reSourceVersionEnvs` (generated) and prints a `----` banner.
- Naming is `verb + Target` (`getPoppler`, `buildPoppler`, `createAppImage`, `uploadGitHubRelease`), with per-distro suffixes `Apt` / `Dnf` / `Alpine`.
- Env knobs: `UNATTENDED` toggles `--assume-yes`, `MAKE_PARALLEL`, `PDF2HTMLEX_PREFIX` (default `/usr/local`), `PDF2HTMLEX_PATH`. Static deps build into repo-root `poppler/`, `poppler-data/`, `fontforge/`; artifacts land in `imageBuild/`. Distro detected from `/etc/os-release` → `BUILD_OS`, `BUILD_DIST`.

**Templates / generated files**
- `.in` files are the source of truth (`src/pdf2htmlEX-config.h.in`, `src/util/css_const.h.in`, `share/*.css.in`, `share/pdf2htmlEX.js.in`, `pdf2htmlEX.1.in`, `test/test.py.in`); their generated counterparts are gitignored — edit the `.in`, never the generated file.
- CSS class shorthands are declared once in `src/css_class_names.cmakelists.txt` (`CSS_LINE_CN "t"`, `CSS_FILL_COLOR_CN "fc"`, `CSS_PAGE_FRAME_CN "pf"`, …) and flow into `namespace pdf2htmlEX::CSS`.
- `PDF2HTMLEX_VERSION` comes from the environment (`$ENV{PDF2HTMLEX_VERSION}`), never hardcoded in CMake.

## Important Files

| File | Why it matters |
|---|---|
| `pdf2htmlEX/src/pdf2htmlEX.cc` | `main()`, full CLI option table, validation, the whole top-level pipeline. |
| `pdf2htmlEX/src/Param.h` | The single global options struct; every option touches this. |
| `pdf2htmlEX/src/ArgParser.{h,cc}` | getopt wrapper; add new CLI flags here + in `Param.h` + the man page. |
| `pdf2htmlEX/src/HTMLRenderer/HTMLRenderer.h` | Core `OutputDev` declaration and renderer state; `.cc` files implement per-area hooks. |
| `pdf2htmlEX/src/HTMLRenderer/general.cc` | Ctor/dtor, `process()` page loop, `startPage`/`endPage`, `dump_css`, `embed_file`. |
| `pdf2htmlEX/src/StateManager.h` | CSS class dedup (`AllStateManager`); where output size/perf decisions are made. |
| `pdf2htmlEX/src/HTMLTextPage.{h,cc}`, `HTMLTextLine.{h,cc}` | Per-page/per-line text model, state diffing, optimization. |
| `pdf2htmlEX/src/CoveredTextDetector.{h,cc}`, `DrawingTracer.{h,cc}` | `--correct-text-visibility` implementation. |
| `pdf2htmlEX/src/util/ffw.c` + `ffw.h` | FontForge C ABI boundary (`ffw_init`, `ffw_load_font`, `ffw_reencode_*`, `ffw_auto_hint`, …). `ffw.h` notes `fontforge.h` cannot be included from C++. |
| `pdf2htmlEX/CMakeLists.txt` | The only build file: hard-coded dep paths, C++20, `ENABLE_SVG`, generated files, install rules, CTest entries. |
| `pdf2htmlEX/src/pdf2htmlEX-config.h.in`, `src/util/css_const.h.in` | Build-time injected constants (`PDF2HTMLEX_PREFIX`, `PDF2HTMLEX_DATA_PATH`, `ENABLE_SVG`, CSS class names). |
| `pdf2htmlEX/pdf2htmlEX.1.in` | Man page; user-visible docs for every option (369 lines, `.SH` sections). |
| `buildScripts/versionEnvs` | Single source of truth for versions/artifact naming. |
| `docker/Dockerfile.trixie` | The only container image recipe; builds this checkout against the pinned Poppler/FontForge. |
| `buildScripts/Readme.md` | Authoritative build documentation (top-level scripts and individual steps). |
| `pdf2htmlEX/test/test.py.in` | Template for the generated `test.py` shared path/`Common` base. |
| `CONTRIBUTING.md` | Contribution channels, bug-report and PR rules, license grant. |

## Runtime/Tooling Preferences

- **Platform: Linux only.** `.travis.yml` states: "Since we do not have direct access to either MacOS or Windows machines on which to develop WE DO NOT support building on MacOS or Windows." AppImage "will not currently work on MacOS or Alpine based machines"; `buildInstallLocallyAlpine` is marked experimental. This checkout is developed on Windows but the toolchain is Linux (WSL/container).
- **Compiler/tooling**: GCC via `-Wall -Woverloaded-virtual`, `-std=c++23`, `pkg-config`, `make -j$(nproc)`, CMake >= 3.28 (Poppler's minimum).
- **Dependencies are pinned and statically linked, not found by CMake**: poppler `poppler-26.09.0`, fontforge `20251009` (both in `buildScripts/versionEnvs`), plus `poppler-data-0.4.12`. `CMakeLists.txt` hard-codes `../poppler/build/...` and `../fontforge/build/lib/libfontforge.a`, so the tree layout `repo/{pdf2htmlEX,poppler,poppler-data,fontforge}` is mandatory. `pkg_check_modules(CAIRO REQUIRED cairo>=1.10.0)`, `pkg_check_modules(GLIB2 REQUIRED glib-2.0 gio-2.0 gobject-2.0)` (fontforge's `inc/ffglib.h` reaches into glib/gio via `fontforge/baseviews.h`), `find_package(Freetype REQUIRED)` and `find_package(Threads REQUIRED)` are the package lookups. `-lintl` is added only when `/usr/lib/libintl.so` exists (Alpine).
- **Host requirements for the pinned Poppler**: `poppler-26.09.0` needs CMake >= 3.28, glib >= 2.80, cairo >= 1.18, freetype >= 2.13, fontconfig >= 2.15 — i.e. Debian >= 13 (trixie); Bookworm (cmake 3.25, glib 2.74, cairo 1.16, freetype 2.12) is too old. Also set `-DENABLE_HARFBUZZ=OFF` for Poppler unless libharfbuzz-dev is installed (it is `ON` by default and hard-fails configuration).
- **Only build option**: `-DENABLE_SVG=ON|OFF` (default `ON`); turning it on hard-requires `cairo-svg.h` (FATAL_ERROR otherwise) and gates SVG background/Type 3 support.
- **JS/CSS minification** runs as an `ALL` target via `pdf2htmlEX/share/build_css.sh` (YUI Compressor 2.4.8) and `pdf2htmlEX/share/build_js.sh` (Closure Compiler `SIMPLE_OPTIMIZATIONS`); both require a JDK/java. `3rdparty/PDF.js/build.sh` uses `ADVANCED_OPTIMIZATIONS`.
- **Version flow**: `PDF2HTMLEX_VERSION` env var → CMake → generated headers + man page. Artifact names: `$PDF2HTMLEX_VERSION-$PDF2HTMLEX_BRANCH-$BUILD_DATE-$BUILD_OS-$BUILD_DIST-$MACHINE_ARCH`.
- **Release automation** env: `GITHUB_USERNAME`, `GITHUB_TOKEN`, `TRAVIS_REPO_SLUG` for `uploadGitHubRelease` (writes `~/.netrc`, needs `jq`); `DOCKER_HUB_USERNAME` (+ optional `DOCKER_HUB_PASSWORD`) for `uploadContainerImage`.
- **CI**: `.github/workflows/build.yml` — `on: [push]`, `ubuntu-24.04`, runs `./buildScripts/buildInstallLocallyApt`, uploads `pdf2htmlEX/build` as artifact. It does **not** run tests. `.travis.yml` (legacy) runs `./buildScripts/travisLinuxDoItAll`.
- **Docker image** (`docker/Dockerfile.trixie`): builds *this* checkout (no `git clone`), in **two stages**. The named `builder` stage (`docker build --target builder …` is addressable) carries the toolchain plus the pinned Poppler/FontForge, split into cache-friendly layers; the final `debian:trixie-slim` stage copies only `/usr/local/bin/pdf2htmlEX`, `/usr/local/share/pdf2htmlEX` (incl. poppler-data) and the man page, and installs only the shared libraries `ldd` reports (`libcairo2 libfreetype6 libfontconfig1 libpng16-16 libjpeg62-turbo libopenjp2-7 libxml2 libglib2.0-0t64 libwoff1` — the last one is needed because Debian ships the WOFF2 encoder only as a shared library, and FontForge is built with `ENABLE_WOFF2`). The local-build scripts install the matching development package (`libwoff-dev` on apt, `woff2-devel` on dnf, `woff2-dev` on Alpine) so `buildInstallLocallyApt` gets WOFF2 support too. That is the difference between ~182 MB and ~1.7 GB: **deleting files in a later `RUN` never shrinks an image**, it only hides them, so the single-stage variant kept its 894 MB build layer (the minification JRE alone is 198 MB) plus the 47 MB source tree. Windows checkouts store the scripts with CRLF, so each stage strips `\r` before running them; `sed -i 's/sudo //g'` drops the root-unnecessary `sudo`. `.dockerignore` keeps `.git`, `.codegraph` and any unpacked `poppler/`, `fontforge/`, `poppler-data/` out of the build context. Env baked into the builder: `PDF2HTMLEX_BRANCH=cutlassfish`, `PDF2HTMLEX_PREFIX=/usr/local`, `MAKE_PARALLEL` (build-arg, default `-j6`).
- **Upgrading the pinned versions** (the recurring maintenance job here): bump `POPPLER_VERSION` / `FONTFORGE_VERSION` in `buildScripts/versionEnvs`, re-check Poppler's minimum tool/library versions and its CMake option names in `buildScripts/buildPoppler` (options are added/removed/renamed every few releases — dead ones only warn), then fix the Poppler API drift in `pdf2htmlEX/src`. Compile errors are the *easy* half — some poppler changes are silent, so re-render the fixtures and check them (`pdf2htmlEX` still exits 0). Drift seen going 24.06.1 → 26.09.0: **`GfxState::getCurX()/getCurY()` silently changed meaning** — in 24.06 `textMoveTo()` wrote the text cursor into `curX/curY`, from ~25.x the text cursor lives in `getCurTextX()/getCurTextY()` and `curX/curY` are maintained only for path construction (`setCurX/setCurY` no longer exist anywhere). Reading `getCurX()` for text positions compiles fine but collapses every `<div class="t">` onto one CSS `x`/`y` class (whole page of text stacked in one spot); see `HTMLRenderer/state.cc` and `DrawingTracer.cc`, which now mirror poppler's own `Gfx::doShowText` idiom (`textTransformDelta(0, getRise(), …)` then `transform(getCurTextX() + rise_x, getCurTextY() + rise_y, …)`). Also: `std::array` return values (`GfxState::getCTM`/`getTextMat`, `GfxFont::getFontBBox`/`getFontMatrix`, `Matrix::m`, `AnnotColor::getValues`, `OutputDev::beginTransparencyGroup` bbox), `std::string` instead of `GooString*` (`OutputDev::drawString`, `beginString`), `std::vector<int>` instead of `int*` (`Gfx8BitFont`/`GfxCIDFont::getCodeToGIDMap`, `getCIDToGID`), `Object::getStream()->getDict()` instead of `streamGetDict()`, `Stream::toUnsignedChars()` instead of `streamGetChars()` (`Stream::getChars` is private), `GfxFont::getWMode()` as `enum class WritingMode`, `SplashError` as `enum class` (`SplashError::NoError`, no `splashOk`), `SplashOutputDev` ctor losing `reverseVideo`/`ignorePaperColor`/font-engine args, and `CharCodeToUnicode` losing its reference counting (`getToUnicode()` returns a bare `const` pointer owned by the font). **Poppler aborts (SIGABRT) on a type-mismatched `Object` accessor** — `OBJECT_TYPE_CHECK` calls `abort()` — so always use the accessor matching the runtime object type; guessing `getDict()` on a stream object kills the process with no message.
- **Caveats to respect**: the `getDevLibraries*` package lists are incomplete (none of them lists freetype, fontconfig or glib, all of which the pinned Poppler needs), so treat `docker/Dockerfile.trixie` as the reliable list. `patches/fontforge-*.patch` target fontforge `20170731` and the apply loop in `buildFontforge` is commented out. `archive/` is Python 2 / Ubuntu `disco` era and dead.

## Testing & QA

Three layers under `pdf2htmlEX/test/`; `test/README.md` is the entry doc. Test framework is **Python 3 `unittest`** ("a collection of python3 unittests of the output of pdf2htmlEX").

1. **Output tests** — `test_output.py` (Python) and `testOutput` (shell mirror).
   - `./runLocalTestsPython` → `python3 test_output.py`; `./runLocalTestsShell` → `./testOutput`.
   - `run_test_case(input_file, args=[], expected_output_files=None)` asserts **filenames only** (`assertCountEqual(result['output_files'], expected_output_files)`) — never file contents.
   - Fixtures: `test/test_output/` (`1-page.pdf`, `2-pages.pdf`, `3-pages.pdf`, `issue501`).
   - Inside the built image the suite runs as `cd /src/pdf2htmlEX/test && python3 test_output.py` after repointing `Common.PDF2HTMLEX_PATH` in the generated `test.py` at `/usr/local/bin/pdf2htmlEX` (the image drops the `build/` directory the template points at); 23 tests, all passing on the current toolchain.
2. **Local browser tests** — `test_local_browser.py` on `browser_tests.py`, driving Firefox via Selenium; run with `./runLocalBrowserTests` (starts `/usr/bin/Xvfb -- :99 -ac -screen 0 1280x1920x16`, `export DISPLAY=:99.0`).
   - Args: `DEFAULT_PDF2HTMLEX_ARGS = ['--fit-width', 800, '--last-page', 1]`, window `800x1200`.
   - Assertion is **exact**: `ImageChops.difference(ref_img, out_img)` must yield `diff_bbox is None` — no pixel tolerance knob.
   - Fixtures: `browser_tests/<name>.pdf` (+ optional `<name>.tex`) and reference `browser_tests/<name>/<name>.html`; `basefilename = splitext(filename)[0]` means names must match exactly. PNG diffs land in `/tmp/pdf2htmlEX/png` (`*.ref.png`, `*.out.png`, `*.diff.png`) and are never committed.
   - `test_fail` is intentionally inverted (diff ⇒ success); `test_text_visibility` is `@unittest.skip`ped due to a known clipping bug.
   - Containerised recipe (no host Xvfb/Firefox needed): in a container from the built image run `apt-get install firefox-esr xvfb python3-selenium python3-pil`, drop a `geckodriver` binary in `/usr/local/bin`, then run `test_local_browser.py` under `Xvfb :99` with `DISPLAY=:99`. Debian's `python3-selenium` ships **no** Selenium Manager binary, so `webdriver.Firefox()` must be given `service=Service(executable_path="/usr/local/bin/geckodriver")` — easiest via a throwaway shim that patches `selenium.webdriver.Firefox` before importing the test module. Screenshots/diffs land in `/tmp/pdf2htmlEX/png` and `/tmp/pdf2htmlEX/html` (`*.out.png`, `*.ref.png`, `*.diff.png`, `*.{out,ref,diff}.html`) and are never committed.
3. **Remote browser tests** — `test_remote_browser.py` (Sauce Labs; one class per `BROWSER_MATRIX` entry). Needs `SAUCE_USERNAME`, `SAUCE_ACCESS_KEY`, `TRAVIS_BRANCH`, `TRAVIS_PULL_REQUEST`, `TRAVIS_JOB_NUMBER`, `TRAVIS_BUILD_NUMBER`, `BASEURL='http://localhost:8000/'`. README states these are "not fully implemented or (re)tested"; the file is stale (`sys.exc_clear()` is Python 2, `generate_image` references an undefined `page_must_load`).

**Glue**
- `test.py` is **generated** from `test.py.in` by `configure_file` in `pdf2htmlEX/CMakeLists.txt`; CMake injects `@PDF2HTMLEX_PATH@`, `@PDF2HTMLEX_TMPDIR@`, `@PDF2HTMLEX_DATDIR@`, `@PDF2HTMLEX_PNGDIR@`, `@PDF2HTMLEX_OUTDIR@`, `@PDF2HTMLEX_PREDIR@`, `@PDF2HTMLEX_HTMDIR@` (all `/tmp/pdf2htmlEX/...`). Never hand-edit `test/test.py`. Also configure `build/` before running tests.
- `Common.setUp()` copies `share/manifest` (minus the `#TEST_IGNORE_BEGIN`/`#TEST_IGNORE_END` block) plus `share/base.min.css` and `test/fancy.min.css` into `DATDIR` — so the resource minification target must have run before tests.
- CTest: `include(CTest)` adds `test_basic` (`test/test_output.py`) and `test_browser` (`test/test_local_browser.py`) — both invoked with a bare `python`.
- `./produceHtmlForBrowserTests` pre-renders each fixture (`--fit-width=800 --last-page=1`) into `PREDIR`.
- Deps: `./installAutomaticTestSoftwareApt` (`sudo apt -y install wget diffutils zip python3 python3-pip xvfb firefox`, geckodriver `v0.26.0`, `pip3 install selenium Pillow`), `./installAutomaticTestSoftwareDnf` (dnf variant), `./installManualTestSoftware` (`graphicsmagick-imagemagick-compat okular`).
- Whole suite: `./buildScripts/runTests` → `cd pdf2htmlEX/test`, install deps, `runLocalTestsShell`, `runLocalBrowserTests`, then zips `/tmp/pdf2htmlEX/html` and `/tmp/pdf2htmlEX/png`. Nothing calls it in CI (the Travis call is commented out).

**Adding a test** (`test/README.md` "Add new test cases"): drop a single-page PDF with a meaningful name (`issueNNN.pdf` for regressions, grayscale unless the test needs color, include the `.tex` source for browser fixtures) into `test_output/` or `browser_tests/`; add a `test_<name>` method plus a reference HTML at `browser_tests/<name>/<name>.html`; generate the reference with `P2H_TEST_GEN=1 test/test.py test_issueXXX` (`GENERATING_MODE = bool(os.environ.get('P2H_TEST_GEN'))` regenerates instead of asserting). `./regenerateTest <testName>` regenerates a reference HTML; `./compareTestImages <testName>` opens a diff viewer. Note `test/README.md` still names scripts that no longer exist (`runLocalTests`, `runRemoteBrowserTests`, `installAutomaticTestSoftware`, `regenerateTestHtml`) — use `runLocalTestsShell`/`runLocalTestsPython`, `installAutomaticTestSoftwareApt`/`Dnf`, `regenerateTest`.
