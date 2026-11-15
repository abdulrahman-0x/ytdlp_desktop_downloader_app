# Migration Plan: CustomTkinter → PySide6 6.12.0

**Target:** PySide6 **6.12.0** (Qt for Python) on **Python 3.13.8** / **Windows**
**Current GUI stack:** Tkinter + CustomTkinter ≥ 5.2
**Status:** Plan — not yet started

---

## 1. Executive summary

This application is currently a **Tkinter/CustomTkinter** desktop app (`app.py` + `ui/*.py`).
Migrating to PySide6 is therefore a **GUI-framework rewrite of the `ui/` layer and the
app bootstrap**, while the non-GUI core (`core/`, `utils/`) stays intact except for one
structural change: the **thread→GUI event bridge** (`queue.Queue` + `after()` polling)
is replaced by Qt **signals/slots**.

PySide6 6.12.0 is **already installed** in this environment and is fully compatible
with Python 3.13.8 on Windows — verified below. The migration is code-side work plus
packaging adjustments, not environment setup.

| Dimension | Today | After migration |
|---|---|---|
| GUI toolkit | `customtkinter` (Tk) | PySide6 6.12.0 (Qt Widgets) |
| Theming | `ctk.set_appearance_mode` | Fusion style + `QPalette` + `QStyleHints.colorScheme` |
| Threads → UI | `queue.Queue` + `after(0/100)` polling | `Signal`/`Slot` queued connections |
| Timers | `widget.after(ms, fn)` | `QTimer` / `QTimer.singleShot(ms, ctx, fn)` |
| Thumbnails | `requests` + Pillow + `ctk.CTkImage` | `requests` + `QImage`/`QPixmap` (Pillow removed from runtime) |
| Tests | `tk_root` fixture, stub-based | `pytest-qt` (`qapp`, `qtbot`) |
| Bundle | Measured **449 MB** (`Release\`; `bin\` binaries dominate) | Adds Qt Widgets DLLs/plugins — measured Core+Gui+Widgets DLLs **25.4 MB** installed; full delta measured in Phase 6 |

---

## 2. Environment & compatibility — verified facts

All facts below were verified in this session against the local environment, PyPI, and
the official Qt for Python documentation (doc.qt.io).

### 2.1 Local environment (verified via `python`/`pip`)

| Item | Value |
|---|---|
| Python | **3.13.8** (`python --version`) |
| pip | 26.3 |
| PySide6 installed | **6.12.0** (also `shiboken6` 6.12.0, `PySide6_Essentials`, `PySide6_Addons`, `PySide6_Pdf` 6.12.0.140, `PySide6_WebEngine` 6.12.0.140) |
| PySide6 on PyPI | **6.12.0 is the latest release** (available versions: 6.12.0, 6.11.3, … 6.8.0.2) |
| PyInstaller | 6.22.3 |
| pytest / pytest-qt | 8.4.2 / installed ✔ (pytest-qt already present) |

### 2.2 PyPI metadata for PySide6 6.12.0 (verified via `pypi.org/pypi/PySide6/6.12.0/json`)

- `requires_python`: **`>=3.10,<3.16`** → Python **3.13.8 is in range** ✔
- Windows wheels exist: `pyside6-6.12.0-cp310-abi3-win_amd64.whl` and `…-win_arm64.whl`
  — the **`cp310-abi3`** stable-ABI tag means one wheel serves **CPython 3.10 → 3.15**,
  including 3.13.8. ✔
- Release date: 2026-10-08.

### 2.3 Official documentation (doc.qt.io) — key statements

- **Requirements:** "Official Python 3.10+"; wheels include Qt binaries — **no system Qt
  installation is required**; venv recommended. ([Getting Started](https://doc.qt.io/qtforpython-6/gettingstarted.html))
- **Install pinned version:** `pip install PySide6==6.12.0`
  ([Getting Started / Installation](https://doc.qt.io/qtforpython-6/gettingstarted.html))
- **6.12.0 release notes** ([PySide6 release notes](https://doc.qt.io/qtforpython-6/release_notes/pyside6_release_notes.html)):
  - `PYSIDE-3221` — initial support for **Python 3.15** added (forward-compatibility).
  - `PYSIDE-3367` — **copy constructors of `QTree/Table/ListWidgetItem` removed**
    (relevant if the playlist list uses `QListWidget`/`QTreeWidget`: never do
    `QListWidgetItem(other_item)`; construct fresh items).
  - `PYSIDE-3335` — `QBitArray.fromBits()/bits()` added.
  - `PYSIDE-3314` — iOS support added.
- **Threading:** signals/slots are **safe across threads** via queued connections;
  `QThread` supports both the *worker-object* (`moveToThread`) and *subclass-`run()`*
  patterns. A `QThread` object *lives in the creating thread* — its slots run there,
  not in the new thread. ([QThread docs](https://doc.qt.io/qtforpython-6/PySide6/QtCore/QThread.html),
  [thread_signals example](https://doc.qt.io/qtforpython-6/examples/example_widgets_thread_signals.html))
- **Timers:** `QTimer.singleShot(msec, context, callable)` replaces `after()`;
  timers need a running event loop and must be (re)started from their own thread.
  ([QTimer](https://doc.qt.io/qtforpython-6/PySide6/QtCore/QTimer.html), [Timers overview](https://doc.qt.io/qtforpython-6/overviews/qtcore-timers.html))
- **Theming:** Qt's **Fusion** style draws from `QPalette`, so one style yields both a
  light and dark variant; `QStyleHints.colorScheme` (added **Qt 6.5**) reports the
  system scheme and can be overridden with `setColorScheme(Qt.ColorScheme.Dark/Light)`;
  reset with `unsetColorScheme()` for "follow system". Do **not** mix `setPalette()`
  with style sheets. ([QStyleHints](https://doc.qt.io/qtforpython-6/PySide6/QtGui/QStyleHints.html),
  [Styling approaches](https://doc.qt.io/qtforpython-6/overviews/qtwidgets-styling-approaches.html),
  [QApplication.setPalette](https://doc.qt.io/qtforpython-6/PySide6/QtWidgets/QApplication.html))
- **Deployment:** PyInstaller is the documented path: `pip install pyinstaller` then
  `pyinstaller --name="MyApplication" --windowed …`. Warning: PyInstaller may use a
  system-installed PySide6 instead of the venv's. Qt DLLs/plugins are collected
  automatically by PyInstaller's bundled `hook-PySide6*` hooks — verified in this
  install: 59 `hook-PySide6*.py` files under `PyInstaller\hooks\`, including
  `hook-PySide6.QtWidgets.py` and `hook-PySide6.QtWebEngineCore.py`.
  ([Deployment with PyInstaller](https://doc.qt.io/qtforpython-6/deployment/deployment-pyinstaller.html))

> **Conclusion:** no environment work is required — 6.12.0 is installed, is the latest,
> and supports CPython 3.13.8 on Windows via its abi3 wheel. All remaining work is code,
> tests, and packaging.

---

## 3. Scope — what changes and what doesn't

### 3.1 In scope (GUI layer + bridge)

Codegraph blast-radius analysis of the current GUI surface:

| File | Role | Migration weight |
|---|---|---|
| `app.py` | Bootstrap: `ctk.CTk()` root, theme, geometry, icon, `mainloop()` | Medium |
| `ui/main_window.py` | `MainWindow` — tabs, URL entry, info, quality, dirs, downloads, controller wiring (19 call sites in `app.py`/tests) | **High** |
| `ui/playlist_panel.py` | Canvas-based scrollable list, rows (checkbox, thumb, title, duration, status badge, hover), selection toolbar (30 call sites) | **High** |
| `ui/settings_dialog.py` | `CTkToplevel` with tabview, option menus, entries, checkboxes, `fd.askdirectory`, `messagebox` | Medium |
| `ui/startup_check.py` | Dependency-check screen (`DependencyChecker` in thread, `after(0)` post-back) | Medium |
| `ui/quality_selector.py` | Two `CTkOptionMenu`s + `StringVar`s | Low |
| `ui/progress_widget.py` | `CTkProgressBar` + labels | Low |
| `ui/logs_panel.py` | Collapsible read-only log textbox | Low |
| `core/download_controller.py` | `threading.Thread` + `queue.Queue` + `_poll_queue()` via `after(100)`; `after_id` bookkeeping | **Medium (structural)** |
| `tests/*` (8 UI test files + `conftest.py`) | `tk_root` fixture; Tk-specific stubs | Medium |
| `requirements.txt` | `customtkinter` → `PySide6==6.12.0`; Pillow moves to build-only | Low |
| `ytdownloader.spec` | Excludes for Tk; defensive excludes for unused Qt modules | Low |
| `build_windows.ps1` | Prerequisite check imports `customtkinter` → `PySide6` | Low |

### 3.2 Out of scope (unchanged)

- `core/info_extractor.py`, `core/format_builder.py`, `core/dep_checker.py`,
  `core/config_manager.py`, `utils/file_utils.py`, `utils/paths.py` — pure logic,
  no Tk imports.
- `bin/` (yt-dlp.exe/ffmpeg/ffprobe/node), `assets/`, the `YTDLP_DESKTOP_SELFTEST`
  packaging self-test contract, and the portable `Release\` layout
  (`utils.paths.app_root()` resolution).
- `build_assets/make_icon.py` keeps using Pillow (build-time only).

---

## 4. Architecture decisions (verified against Qt docs)

### 4.1 Threading model — replace polling with signals (the core structural change)

**Today** (`core/download_controller.py:444`): workers push `(event, data)` tuples into a
`queue.Queue`; `_poll_queue()` drains it on the GUI thread and is re-armed with
`self._app.after(100, self._poll_queue)`; `after_id` must be cancelled on shutdown.
Workers also use `self.after(0, callback)` in `ui/` to hop back to the GUI thread.

**Target:** keep the existing worker logic (it runs `subprocess.Popen` and parses
yt-dlp output — this code is solid and tested) but move the event delivery to Qt:

```python
# core/download_controller.py (target shape)
from PySide6.QtCore import QObject, Signal

class ControllerSignals(QObject):
    # Signal emissions from a worker thread are delivered as QUEUED connections
    # on the GUI thread — documented as thread-safe by Qt.
    progress       = Signal(dict)
    log            = Signal(str)
    done           = Signal()
    error          = Signal(str)
    playlist_item  = Signal(dict)
    playlist_done  = Signal()

class DownloadController:
    def __init__(self, config):
        self.signals = ControllerSignals()
        # worker thread stays a plain threading.Thread (subprocess-driven),
        # but instead of queue.put(...): self.signals.progress.emit(data)
```

Rules taken from the [QThread documentation](https://doc.qt.io/qtforpython-6/PySide6/QtCore/QThread.html):

- Emitting a `Signal` from a non-GUI thread is safe: the connection to a GUI-thread
  slot is automatically a **queued connection**.
- We deliberately do **not** need `QThread`'s event loop (no `moveToThread`) because
  the workers are subprocess/IO driven, not slot-driven. Plain `threading.Thread` +
  `Signal.emit` is the minimal-risk port. (A later refactor can adopt the
  worker-object `QThread` pattern if desired.)
- This deletes `_poll_queue()`, `_after_id`, the 100 ms latency, and the
  `after_cancel` cleanup path entirely.

**Every `after(0, …)` in `ui/`** (thumbnail post-back, info post-back, startup-check
post-back) becomes either `QTimer.singleShot(0, self, callback)` — the documented
`after()` equivalent — or, preferably, a direct `Signal` emission from the worker.

### 4.2 Bootstrap (`app.py`)

| Current | Target |
|---|---|
| `ctk.set_appearance_mode(theme)` | `app.setStyle("Fusion")` + palette/color-scheme (§4.4) |
| `ctk.CTk()` root | `QApplication(sys.argv)` + `QMainWindow` |
| `root.geometry("WxH")`, `root.minsize(...)` | `win.resize(w, h)`, `win.setMinimumSize(700, 500)` |
| `root.iconbitmap(str(logo))` | `win.setWindowIcon(QIcon(str(logo)))` (keep the same `OSError`-guard + test semantics) |
| `StartupCheckFrame(root, on_startup_done)` then `MainWindow(root, config)` in the same root grid | `QStackedWidget` in `QMainWindow`: page 0 = `StartupCheckPage`, page 1 = `MainWindow` (preserves current UX and keeps `MainWindow`'s constructor signature `MainWindow(parent, config)`) |
| `root.mainloop()` | `sys.exit(app.exec())` |
| `YTDLP_DESKTOP_SELFTEST` → return before GUI | **Unchanged**: return *before creating `QApplication`* so the packaged self-test still works headless (this is what `build_windows.ps1` → `Invoke-PackagedSelfTest` relies on) |

HiDPI: Qt 6 scales automatically (the old `AA_EnableHighDpiScaling` attribute is
deprecated/always-on in Qt 6) — no action needed.

### 4.3 Widget mapping

| CustomTkinter / Tk | PySide6 6.12.0 |
|---|---|
| `ctk.CTkFrame` | `QWidget` + `QFrame` (with `QLayout`) |
| `ctk.CTk()` / `CTkToplevel` | `QMainWindow` / `QDialog` (modal via `exec()`; settings uses `grab_set()` today → modal `QDialog` is stricter and safer) |
| `ctk.CTkTabview` | `QTabWidget` (`addTab`, `setCurrentIndex`) |
| `ctk.CTkEntry` + `StringVar` | `QLineEdit` (`.text()` / `.setText()`; `StringVar.trace_add("write")` → `textChanged` signal; `<Return>` bind → `returnPressed`) |
| `ctk.CTkButton(command=…)` | `QPushButton.clicked.connect(…)`; `configure(state="disabled")` → `setEnabled(False)` |
| `ctk.CTkLabel` | `QLabel` (`setText`, `setPixmap`, alignment flags) |
| `ctk.CTkOptionMenu(values=…)` | `QComboBox` (`clear()`/`addItems()`; block signals with `QSignalBlocker` when repopulating) |
| `ctk.CTkCheckBox` + `BooleanVar` | `QCheckBox` (`isChecked()`/`setChecked()`; `toggled` signal) |
| `ctk.CTkProgressBar.set(0..1)` | `QProgressBar` — note the range change: current code produces 0..1 floats (`_safe_percent`) → `setRange(0, 100)` (or `0–1000` for sub-percent precision) and scale |
| `ctk.CTkTextbox` (read-only log) | `QPlainTextEdit` (`setReadOnly(True)`, `appendPlainText()`, `maximumBlockCount` gives free log trimming, `ensureCursorVisible` replaces `.see("end")`) |
| `grid()/pack()` + `grid_columnconfigure(weight)` | `QGridLayout`/`QVBoxLayout`/`QHBoxLayout` + stretch factors (`addStretch`, `setColumnStretch`) |
| `widget.after(ms, fn)` | `QTimer.singleShot(ms, widget, fn)` (context-object overload so the callback dies with the widget) |
| `tkinter.filedialog.askdirectory` | `QFileDialog.getExistingDirectory(self, caption, dir)` |
| `tkinter.messagebox.showerror` | `QMessageBox.critical(self, title, text)` |
| `winfo_children()` sweeps / `destroy()` | layout take/clear loop with `deleteLater()` (deferred C++ deletion) |
| `bind("<Enter>/<Leave>")` hover | `enterEvent`/`leaveEvent` on row widgets, or `QListWidget` item roles |
| `ctk.CTkImage(PIL img)` | `QImage.loadFromData(bytes)` → `QPixmap.fromImage(img).scaled(w, h, Qt.IgnoreAspectRatio, Qt.SmoothTransformation)` — **Pillow no longer needed at runtime** |
| `tk.TclError` guards when widget died | `shiboken6.isValid(widget)` check (or keep `try/except RuntimeError`) before touching a widget whose row may have been rebuilt |

### 4.4 Theming (`ui.theme`: "dark" | "light" | "system")

Verified approach (doc.qt.io: Fusion draws from `QPalette`; `QStyleHints.colorScheme`
added in Qt 6.5):

1. `app.setStyle("Fusion")` once at startup — deterministic look on all Windows builds
   (some styles do not use the palette for all drawing — the Qt docs name the Windows
   Vista and macOS styles — while Fusion draws its light/dark variants from the
   palette).
2. Resolve `ui.theme`:
   - `dark` → install a dark Fusion `QPalette` (custom palette; CTk-like colors:
     `#14141A` window, `#ff2d55` accent to match the brand colors in `make_icon.py`
     — verified against its `fill=(20, 20, 26, 255)` and `fill=(255, 45, 85, 255)`
     values).
   - `light` → light Fusion `QPalette`.
   - `system` → do nothing; Fusion + default palette follows
     `QStyleHints.colorScheme()`; optionally connect
     `app.styleHints().colorSchemeChanged` to re-tint app-specific colors.
3. **Do not** mix `setPalette()` with Qt Style Sheets (documented warning on
   `QApplication.setPalette`).
4. Keep `ui.theme` config keys and `SettingsDialog` behavior identical.

Arabic/RTL: Qt performs bidi text shaping natively (an upgrade over Tk's rendering).
Keep the current layout direction (LTR chrome, right-aligned labels) initially;
a follow-up can set `app.setLayoutDirection(Qt.RightToLeft)` for full mirroring —
out of scope for parity migration.

Fonts: the repo currently ships no `assets/fonts` directory (the build script copies
it only if present — `build_windows.ps1:156-159`), and the code uses only default Tk
fonts (verified: only `tkfont.Font` measuring in `playlist_panel.py:605`). If a font
bundle is added later, register it via `QtGui.QFontDatabase.addApplicationFont()` —
optional follow-up.

### 4.5 Playlist panel strategy

The canvas-scroll + row-frames design maps cleanly onto Qt:

- Container: `QScrollArea` + `setWidgetResizable(True)` + inner `QWidget` with a
  `QVBoxLayout` of row widgets (the documented `QScrollArea` pattern; set a size
  constraint on the layout so the scroll area tracks content size —
  [QScrollArea docs](https://doc.qt.io/qtforpython-6/PySide6/QtWidgets/QScrollArea.html)).
- Row: `QWidget` with `QCheckBox | QLabel(index) | QLabel(pixmap thumb) | title/duration
  labels | status badge` in a `QHBoxLayout`.
- `_clear()` → iterate layout items, `takeAt(0)`, `widget().deleteLater()`.
- `_load_thumb()` → same `requests` fetch in a thread, then
  `QImage.loadFromData()` → `QPixmap` → `label.setPixmap(...)` on the GUI thread via
  `QTimer.singleShot(0, label, lambda: ...)` or a signal; guard with
  `shiboken6.isValid(label)`.
- Alternative (follow-up, not parity): `QListWidget` + `setItemWidget` — note the
  6.12.0 release-note item **PYSIDE-3367: copy constructors of list/table/tree items
  removed** if/when that route is taken.

### 4.6 Startup check page

`StartupCheckFrame._run_checks()` (subprocess-heavy `DependencyChecker` in a thread,
`after(0)` post-back) becomes a `QWidget` page whose checks run in the same
`threading.Thread`, posting results back via a `Signal(list[DepResult])`.
`self.update()` (Tk repaint) is dropped — Qt repaints on the event loop.

---

## 5. Phased execution plan

Each phase is independently testable and leaves the tree buildable. Estimated effort
assumes one developer familiar with the codebase (the `ui/` layer plus `app.py`
measures 1,727 lines).

### Phase 0 — Prep & guardrails (0.5 d)

- [ ] Branch `feat/pyside6-migration`.
- [ ] Record baseline: `python -m pytest -q` green; `build_windows.bat -SkipTests`
      produces a working Release (screenshot-level smoke).
- [ ] Pin current behavior in a short manual QA checklist (theme switch, playlist
      hover/selection, cancel mid-download, settings save).
- **Accept:** baseline captured; nothing changed.

### Phase 1 — Dependencies & bootstrap skeleton (1 d)

- [ ] `requirements.txt`: `customtkinter>=5.2.0` → `PySide6==6.12.0`; move
      `Pillow>=10.0.0` to a new `requirements-build.txt` (used only by
      `build_assets/make_icon.py` once runtime thumbnails are Qt-native).
- [ ] Rewrite `app.py` bootstrap (§4.2): `QApplication`, `QMainWindow` +
      `QStackedWidget` shell, theme resolver, icon, selftest early-return.
- [ ] Stub `StartupCheckPage` and `MainWindow` so the shell runs.
- **Accept:** `python app.py` opens a themed empty window with icon; selftest env var
  exits 0 with no window.

### Phase 2 — Core widgets (2–3 d)

- [ ] `quality_selector.py` → `QComboBox` pair (§4.3); keep the public surface
      (`apply_defaults`, `set_qualities`, `.mode`, `.quality`) so `main_window`
      wiring and tests stay conceptually identical.
- [ ] `progress_widget.py` → `QProgressBar` (0–100 scaling of `_safe_percent`),
      `reset/set_message/set_done/set_error/update_progress` preserved.
- [ ] `logs_panel.py` → `QPlainTextEdit` + collapse toggle; add
      `setMaximumBlockCount` (bounded log memory — small improvement, note in PR).
- [ ] `startup_check.py` → page + thread + `Signal(list)` (§4.6).
- [ ] Port each widget's tests to `pytest-qt` **in the same commit** (see Phase 5
      conventions).
- **Accept:** widgets usable in isolation; their test files green under `pytest-qt`.

### Phase 3 — MainWindow + SettingsDialog (2–3 d)

- [ ] `main_window.py`: full port — `QTabWidget`, `QLineEdit` + `textChanged`
      gating for the URL, `QGridLayout`, info/thumbnail labels, `QFileDialog`
      browse buttons, settings launch, controller wiring.
- [ ] Replace every `self.after(0, …)` with `QTimer.singleShot(0, self, …)` (or
      signals from Phase 4 workers).
- [ ] `settings_dialog.py` → modal `QDialog` + `QTabWidget` + `QComboBox`/`QLineEdit`
      /`QCheckBox` + `QFileDialog.getExistingDirectory` + `QMessageBox.critical`;
      keep `_parse_int` clamping and the exact config keys written by `_save`.
- [ ] Close behavior: `QMainWindow.closeEvent` → `controller.shutdown()` (replaces
      the Tk-destroy path; verify no hang with an active download).
- **Accept:** full single-video + settings flows pass manual QA and ported tests
      (`test_main_window.py`, `test_settings_flow.py`).

### Phase 4 — Playlist panel (2 d)

- [ ] Port `playlist_panel.py` per §4.5 (scroll area, rows, hover, selection count,
      `begin_download/finish_download/update_item/set_playlist/show` API kept).
- [ ] Port `test_playlist_panel.py` incl. the thread-safety tests
      (`test_load_thumb_applies_image_on_main_thread`,
      `test_measure_fonts_handle_tk_unavailable` → replace with Qt equivalents).
- **Accept:** playlist fetch → select → download → per-item status → done flow green.

### Phase 5 — Controller event bridge (1–2 d)

- [ ] `core/download_controller.py`: introduce `ControllerSignals(QObject)`; replace
      `queue.put` with `signals.*.emit`; delete `_poll_queue`, `_after_id`,
      `set_app`; `MainWindow._setup_callbacks` connects slots to signals instead of
      registering dict callbacks.
- [ ] Keep `cancel()`/`shutdown()` semantics (`stop_event`, `_terminate_tree`,
      `thread.join(timeout=3)`); cancellation still tested by
      `tests/test_download_controller.py` (threading tests mostly survive; event
      assertions switch to `qtbot.waitSignal`).
- **Accept:** `test_download_controller.py` green; no 100 ms polling latency
      (progress updates are event-driven).

### Phase 6 — Packaging & build pipeline (1 d)

- [ ] `ytdownloader.spec`:
      - PyInstaller's bundled `hook-PySide6*` hooks collect only imported modules;
        add **defensive excludes**:
        `PySide6.QtWebEngineCore`, `PySide6.QtWebEngineWidgets`,
        `PySide6.QtWebEngineQuick`, `PySide6.QtPdf`, `PySide6.QtQml`,
        `PySide6.QtQuick`, `PySide6.QtQuick3D`, `PySide6.QtCharts`,
        `PySide6.QtDataVisualization`, `PySide6.Qt3D*`, `PySide6.QtTest`
        (measured in the installed 6.12.0 wheel: `Qt6WebEngineCore.dll` alone is
        194.1 MB and all `*WebEngine*` files total 285.9 MB — this is what the
        excludes keep out of the bundle).
      - Keep `excludes=["test", "pytest", "unittest", "tcl8", …]` — `tcl8` becomes
        harmless once Tk is gone; add `"tkinter"`.
      - `COLLECT`/`_internal` layout, `bin/`, `assets/` placement: **unchanged**
        (`utils.paths.app_root()` untouched).
- [ ] `build_windows.ps1`: prerequisite check
      `& python -c "import PyInstaller, PIL, customtkinter"` →
      `& python -c "import PyInstaller, PySide6"`.
- [ ] Build and run the full pipeline: tests → icon → PyInstaller → Release layout
      validation → packaged self-test from arbitrary CWD → bundled binary smoke tests.
- [ ] Compare `Release\` size before/after and record the actual delta against the
      measured 449 MB baseline. Pre-measurement anchor: Qt6 Core+Gui+Widgets DLLs
      alone measure 25.4 MB installed, with plugins and language packs on top —
      replace the estimate with the measured figure when the build completes.
- **Accept:** `build_windows.bat` full pipeline succeeds; packaged self-test exits 0;
      manual smoke of the frozen app (one download) passes.

### Phase 7 — Cleanup & hardening (1 d)

- [ ] Delete Tk/CTk remnants (`import tkinter`, `tkfont`, `customtkinter`): a
      full-text search must come back empty.
- [ ] Remove `tk_root` fixture from `tests/conftest.py`; add a session-scoped
      `qapp` (or rely on `pytest-qt`'s built-in `qapp`).
- [ ] Update every doc surface that mentions the GUI stack (found by grep):
      `README.md:25` ("UI is built with `customtkinter`"), `FAQ.md:10` (same claim),
      and `FAQ.md:140` (test count + ruff status — both drift after the port);
      historical `docs/PRD_DIFF.md` and `docs/PRD_YTDownloader_v2 - نسخة قديمة.md`
      stay as archived records. Note that Qt and Visual C++ runtime DLLs are bundled
      by PyInstaller.
- [ ] Full `python -m pytest -q` + `build_windows.bat` + manual QA checklist sign-off.

**Total: ~10–13 focused working days** (≈ 2–3 calendar weeks with review/QA buffer).

---

## 6. Test migration strategy (`pytest-qt` — already installed)

Verified present in the environment: `pytest-qt` (with pytest 8.4.2).

| Today | Target |
|---|---|
| `conftest.py` session fixture `tk_root` (`ctk.CTk().withdraw()`) | `pytest-qt`'s built-in `qapp` fixture (auto-uses PySide6); `qtbot.addWidget(...)` for parenting/cleanup |
| `widget.update()` / `after(0)`-driven async tests | `qtbot.waitSignal(signal)`, `qtbot.waitUntil(callable)`, `QTest.qWaitForWindowExposed` |
| Tk stub labels/vars (`_StubImage`, `_ThumbLabelStub`, `TclError` swallows) | Qt widget doubles or minimal `QWidget` subclasses; death-guards become `shiboken6.isValid` |
| `test_app.py` icon/theme assertions (`iconbitmap`, `set_appearance_mode`) | `window.windowIcon().isNull()` is False; style/palette assertions |
| Threading tests in `test_download_controller.py` | keep `threading` harness; event delivery asserted with `qtbot.waitSignal(controller.signals.done)` etc. |

Rules: one QApplication per session (never per-test `QApplication()` — pytest-qt
manages it); no `exec()` in tests; widget tests must not touch the filesystem beyond
`tmp_path` (existing `config` fixture already does).

---

## 7. Risks & mitigations

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| 1 | Qt DLLs/plugins grow the portable Release beyond its measured 449 MB baseline (bin\ binaries dominate it) | Certain | Medium | Baseline measured at 449 MB; defensive spec excludes (§Phase 6); measure the delta in Phase 6; PyInstaller collects by import only |
| 2 | PyInstaller picks a *system* PySide6 instead of the venv's (documented pitfall) | Low | Medium | Build inside a venv where PySide6==6.12.0 is the only Qt binding; verify `_internal\PySide6` version after build |
| 3 | Playlist rewrite diverges from tested behavior (hover, selection, status badges) | Medium | High | Keep the public API of `PlaylistPanel` identical; port tests per widget in the same commit |
| 4 | Signal emission from worker threads during interpreter shutdown (app close while downloading) | Medium | Medium | Keep `shutdown()`'s `stop_event → terminate_tree → join(3s)`; `closeEvent` runs it; tests: cancel-during-download regression |
| 5 | 0..1 → 0..100 progress-range conversion bugs | Low | Low | Single conversion point in `ProgressWidget.update_progress`; unit-test boundaries (0, 1, missing total bytes) |
| 6 | Settings dialog modality/parenting changes (`grab_set` → `exec()`) altering flows | Low | Low | Modal `QDialog.exec()`; port `test_settings_flow.py` first in Phase 3 |
| 7 | Deprecation traps in Qt 6.12 (e.g., `QListWidgetItem(other)` copy-ctor removed in 6.12.0 per PYSIDE-3367) | Low | Low | Code review rule: construct fresh items; CI runs full suite |
| 8 | Arabic text metrics differ (Tk vs Qt font shaping) changing row-height math | Medium | Low | Row widgets get their height from layouts (no manual `_measure_font_cache` port needed); visual QA on AR strings |
| 9 | `YTDLP_DESKTOP_SELFTEST` self-test breaks in frozen app (QApplication needs a display) | Low | High | Selftest path returns **before** `QApplication` creation (Phase 1 acceptance criterion); validated by `Invoke-PackagedSelfTest` in the pipeline |
| 10 | Parallel maintenance on Tk UI during migration | Medium | Medium | Timebox: freeze `ui/` changes on `main` during Phases 2–5; single migration branch |

## 8. Rollback

- Single feature branch; `main` remains the Tk version until Phase 7 sign-off.
- `requirements.txt` change lands only in the final merge → reverting the merge
  restores the Tk app fully (no data-format changes: config JSON, `Release\` layout,
  and `bin/`/`assets/` contracts are identical in both versions).
- The `core/` signal-bridge change is the only shared-code risk; it is behaviorally
  equivalent (same event names/payloads), so a revert is safe.

## 9. Definition of done

1. `python -m pytest -q` fully green (all ported tests, `pytest-qt`).
2. `build_windows.bat` end-to-end: tests → icon → PyInstaller → layout validation →
   packaged self-test (exit 0 from arbitrary CWD) → bundled binary smoke tests.
3. Manual QA checklist (Phase 0) passes on the frozen app: theme dark/light/system,
   single video download + cancel, playlist download with failures, settings
   persistence, restart with same geometry.
4. No `tkinter`/`customtkinter` imports remain (pwsh:
   `Get-ChildItem -Recurse -Include *.py | Select-String "customtkinter|tkinter"`;
   or `rg -n "customtkinter|tkinter" .` if ripgrep is installed).
5. `requirements.txt` pins `PySide6==6.12.0`; build requirements isolated in
   `requirements-build.txt`.

---

## 10. Reference links (official documentation)

- Getting Started / requirements / install: https://doc.qt.io/qtforpython-6/gettingstarted.html
- PySide6 release notes (6.12.0 items): https://doc.qt.io/qtforpython-6/release_notes/pyside6_release_notes.html
- QThread + cross-thread signals: https://doc.qt.io/qtforpython-6/PySide6/QtCore/QThread.html
- Thread/signals example: https://doc.qt.io/qtforpython-6/examples/example_widgets_thread_signals.html
- QTimer: https://doc.qt.io/qtforpython-6/PySide6/QtCore/QTimer.html
- QStyleHints.colorScheme (theming): https://doc.qt.io/qtforpython-6/PySide6/QtGui/QStyleHints.html
- Styling approaches (Fusion + QPalette): https://doc.qt.io/qtforpython-6/overviews/qtwidgets-styling-approaches.html
- QScrollArea: https://doc.qt.io/qtforpython-6/PySide6/QtWidgets/QScrollArea.html
- Deployment with PyInstaller: https://doc.qt.io/qtforpython-6/deployment/deployment-pyinstaller.html
- PyPI package metadata: https://pypi.org/pypi/PySide6/6.12.0/
