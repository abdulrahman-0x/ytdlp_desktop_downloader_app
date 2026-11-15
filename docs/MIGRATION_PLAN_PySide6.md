# خطة الهجرة الكاملة: CustomTkinter → PySide6 6.12.0

> **الحالة:** خطة معتمدة (لم يبدأ التنفيذ) · **الفرع المستهدف:** `feature/pyside6-migration`
> **تاريخ الإعداد:** 2026-11-15 · **التحقق:** كل الادعاءات التحقق منها من المصدر في جلسة الإعداد (انظر الملحق أ).

## 1. السياق والأهداف

النقل الكامل لطبقة الواجهة من `customtkinter`/`tkinter` إلى **PySide6 6.12.0** مع تحديث الاختبارات وخط البناء، وإزالة `customtkinter` نهائيًا. طبقة `core/` و`utils/` لا تُمس إطلاقًا.

- **بيئة التنفيذ:** Windows · Python 3.13.8 (`C:\Program Files\Python313`)
- **الهدف:** `PySide6==6.12.0` — أحدث إصدار على PyPI وقت الإعداد، ويدعم Python 3.13 رسميًا (دعم 3.13 أُضيف في 6.8.1، التحسينات للـ 3.14/3.15 في 6.10.0/6.12.0 حسب توثيق Qt الرسمي)

## 2. المبدأ التصميمي الحاسم

`DownloadController` يستدعي `app.after(100, self._poll_queue)` و`app.after_cancel(...)` (آلية tkinter) — انظر `core/download_controller.py:478-479` و`core/download_controller.py:484-488`.

**الحل:** مُهايئ جديد `ui/qt_after.py` يوفر `after(ms, fn)` و`after_cancel(id)` مبنية على `QTimer`، ويُمرَّر إلى `DownloadController.set_app()`.

**سلامة الخيوط (شرط اعتماد):** `QTimer` يجب أن يُنشأ ويُلغى في خيط الواجهة فقط. تصميم المُهايئ يعالج الاستدعاءات القادمة من Worker Threads:

- `QtAfterAdapter(QObject)` يُنشأ ويحي في خيط الواجهة (parented إلى النافذة الرئيسية)
- قناة تجميع واحدة `Signal(object)` تحمل رسائل `("install", id, ms, fn)` و`("cancel", id)` — مع اتصال `AutoConnection`: تنفيذ **مباشر** إذا جاء الاستدعاء من خيط الواجهة، و**طابور Queued** إذا جاء من خيط عامل (التنفيذ دائمًا في خيط الواجهة)
- المعرفات تُسند مسبقًا تحت `threading.Lock` فيمكن إرجاع المعرف فورًا من أي خيط (توافق مع عقد tkinter)
- سجل `{id: QTimer}` يُمَس في خيط الواجهة فقط؛ إلغاء معرف غير مثبت بعد (سباق طابور) يُسجل في مجموعة `_pending_cancel` ليُتخطى عند وصول رسالة التثبيت
- ترتيب FIFO لطابور الإشارات يضمن اتساق install/cancel لنفس الخيط

النتيجة:
- `core/download_controller.py` يبقى كما هو دون أي تعديل
- `tests/test_download_controller.py` (709 أسطر) يبقى كما هو دون أي تعديل

## 3. خريطة الاستبدال (متحقق من وجود كل الرموز بتجربة تشغيل فعلية)

| CustomTkinter / tkinter (الحالي) | PySide6 (الهدف) |
|---|---|
| `ctk.CTk()` + `mainloop()` | `QApplication` + `QMainWindow` + `exec()` |
| `CTkFrame` / `CTkButton` / `CTkLabel` | `QWidget` / `QPushButton` / `QLabel` |
| `CTkEntry` + `StringVar.trace_add` | `QLineEdit` + `textChanged` |
| `CTkOptionMenu` | `QComboBox` |
| `CTkTabview` | `QTabWidget` |
| `CTkToplevel` (SettingsDialog) | `QDialog` (modal عبر `exec()` بدل `grab_set()`) |
| `CTkCheckBox` | `QCheckBox` |
| `CTkImage` + Pillow resize | `QImage`/`QPixmap` (يُبقى Pillow لتفادي تغيير سلوك المصغرات) |
| `iconbitmap(.ico)` | `setWindowIcon(QIcon(...))` |
| `after(0, fn)` داخل الواجهة | `QTimer.singleShot(0, fn)` |
| `fd.askdirectory` | `QFileDialog.getExistingDirectory` |
| `ctk.set_appearance_mode(...)` | بناء `QPalette` داكن/فاتح حسب مفتاح `ui.theme` |
| استطلاع الأحداث عبر `after(100)` | نفس الاستطلاع عبر `QTimer` (المُهايئ §2) |
| الخيط المنفصل + `after(0, ...)` للرجوع | نفس النمط — الخيوط و`queue.Queue` تبقى كما هي |

## 4. نطاق الملفات

### إعادة كتابة (~1,727 سطرًا واجهة)

| الملف | الأسطر الحالية |
|---|---|
| `app.py` | 50 |
| `ui/main_window.py` | 710 |
| `ui/playlist_panel.py` | 556 |
| `ui/settings_dialog.py` | 144 |
| `ui/startup_check.py` | 75 |
| `ui/logs_panel.py` | 69 |
| `ui/quality_selector.py` | 67 |
| `ui/progress_widget.py` | 56 |
| `ui/qt_after.py` (جديد) | ~30 |

### اختبارات تُحدَّث (~2,028 سطرًا)

**الاستراتيجية (معدّلة بعد التحقق):** الاختبارات الحالية تشغّل ويدجت customtkinter **حقيقية** (جذر `ctk.CTk()` مسحوب في fixture الجلسة — `tests/conftest.py:19-29`)، وليست stubs. لذا فإن النقل = تشغيل PySide6 حقيقية على منصة `QT_QPA_PLATFORM=offscreen`: تحديث `conftest.py` (fixture الجلسة يصبح `QApplication` بمنصة offscreen)، وتكييف استدعاءات الواجهة في الثمانية ملفات: `test_main_window.py` (840)، `test_settings_flow.py` (378)، `test_playlist_panel.py` (372)، `test_startup_check.py` (159)، `test_progress_widget.py` (80)، `test_app.py` (99)، `test_quality_selector.py` (69)، `test_logs_panel.py` (31) — مع الحفاظ على التأكيدات المنطقية نفسها (حالات الأزرار، تسلسل الـ callbacks، الإعدادات).

**اختبار selftest بدون GUI (ملاحظة معتمدة):** يوجد أصلًا `test_main_selftest_returns_before_gui` (`tests/test_app.py:71-77`) يُفشل الاختبار إذا أُنشئت نافذة أو غيّر الثيم. يُضاف إليه اختبار خاص بـ Qt: مسار `YTDLP_DESKTOP_SELFTEST` يجب أن لا يُنشئ `QApplication` إطلاقًا (بإحلال `pytest.fail` مكان `app_module.QApplication`)، وتأكيد أن `QApplication.instance() is None` بعده.

### لا تُمَس

`core/` بالكامل (config_manager, dep_checker, download_controller, format_builder, info_extractor, validators)، `utils/`، `tests/test_download_controller.py` (709)، `tests/test_config_manager*.py`، `tests/test_dep_checker.py`، `tests/test_format_builder.py`، `tests/test_info_extractor.py`، `tests/test_validators.py`، `tests/test_paths.py`، `tests/test_file_utils.py`.

## 5. المراحل التنفيذية

1. **التهيئة** — إنشاء فرع `feature/pyside6-migration`، ثم `pip install PySide6==6.12.0`، وتحديث `requirements.txt`: استبدال `customtkinter>=5.2.0` بـ `PySide6==6.12.0` (يبقى `yt-dlp[default]==2026.08.19`, `yt-dlp-ejs>=0.8.0`, `Pillow>=10.0.0`, `requests>=2.31.0`)
2. **المُهايئ** — كتابة `ui/qt_after.py` وربطه في نقطة استدعاء `set_app()`
3. **إعادة الكتابة** — `app.py` و`startup_check.py` أولًا (تشغيل تجريبي)، ثم `main_window.py` والويدجت الفرعية، ثم `playlist_panel.py` و`settings_dialog.py`
4. **الاختبارات** — تحديث `conftest.py` لضبط `QT_QPA_PLATFORM=offscreen` قبل استيراد PySide6، وتحديث اختبارات الواجهة الثمانية
5. **خط البناء**:
   - `build_windows.ps1` (سطر 111): فحص المتطلبات `import PyInstaller, PIL, customtkinter` ← استبدال `customtkinter` بـ `PySide6`
   - `ytdownloader.spec`: إضافة `"tkinter"` إلى قائمة `excludes` (حاليًا تستثني `tcl8` فقط، الأسطر 40-48) — hooks PySide6 الرسمية في PyInstaller 6.22.3 تلتقط QtWidgets/QtGui/QtCore تلقائيًا حسب المستورد فعلًا
   - **شرط سلامة استبعاد tkinter (متحقق):** imports الـ tkinter/`customtkinter` محصورة في `app.py` والوحدات الخمس المعاد كتابتها و`conftest.py`/اختبارين (grep `^\s*(import tkinter|from tkinter|import customtkinter|from customtkinter)` على كامل المستودع — لا استيراد في `core/` أو `utils/` أو `build_assets/`). قبل بناء Release يُعاد تشغيل نفس الـ grep ويجب أن يكون ناتجه صفرًا خارج `tests/`
   - سلوك `YTDLP_DESKTOP_SELFTEST` (استيراد التطبيق وخروجه برمز 0 دون فتح نافذة) يبقى كما هو — `app.py:34-38`
6. **التحقق** — `pytest` كامل (491 اختبارًا) + `ruff check` (بلا إضافات) + بناء `build_windows.bat` + `YTDLP_DESKTOP_SELFTEST=1` + تشغيل يدوي للنافذة على ويندوز حقيقي مع **قائمة فحص RTL الرباعية** (§6)
7. **التنظيف (شرط: نجاح كل ما سبق أولًا)** — إزالة `customtkinter` من بيئة التطوير فقط بعد نجاح جميع اختبارات Release، وتحديث الإشارات إليه في `README.md` و`FAQ.md` و`GEMINI.md` و`AGENTS.md` (grep للكلمة قبل الإغلاق — قاعدة "كل تغيير كود يستوجب تغيير توثيق")

## 6. المخاطر المعلنة

- **حجم الحزمة:** DLLs الخاصة بـ Qt ستزيد مجلد `Release\` بوضوح مقارنة بـ tkinter — لا مفر منها
- **المظهر:** لن يطابق ستايل CustomTkinter حرفيًا؛ لوحة `QPalette` الداكنة تعادل الأجواء العام
- **بيئة pip:** يوجد PySide6 6.9.3 مثبتًا في site-packages المستخدم؛ `pip install PySide6==6.12.0` سيستبدله — يجب التأكد أن بيئة البناء نفسها هي المستخدمة في الاختبار
- **RTL (تحفظ معتمد):** الواجهة عربية؛ القرار الافتراضي: الحفاظ على ترتيب العناصر البصري الحالي LTR مع نص عربي، كما هو الآن (`QApplication.setLayoutDirection(Qt.RightToLeft)` متاح لو حُسم الاتجاه لاحقًا — سطر واحد). **قائمة فحص يدوي إلزامية** بعد إعادة الكتابة، على منصة ويندوز الحقيقية (وليس offscreen):
  1. محاذاة النصوص العربية في الـ labels/الأزرار (معاينة عرض)
  2. اتجاه أزرار التحكم وترتيبها البصري (استعلام/تحميل/إلغاء/متابعة)
  3. ترتيب التبويبات وعناوينها
  4. عرض أسماء الملفات والمسارات المختلطة عربي/إنجليزي في حقول الإدخال (مجلد الحفظ) وفي لوحة السجلات وقائمة قائمة التشغيل — تسجيل أي انعكاس أو قص نص
- **إزالة customtkinter من بيئة التطوير:** مؤجلة حتى نجاح **جميع** اختبارات Release (بناء `build_windows.bat` + selftest + تشغيل يدوي) — المرحلة 7

## الملحق أ — سجل التحقق (docs-guard)

| # | الادعاء | الحكم | الدليل (التحقق الفعلي في جلسة 2026-11-15) |
|---|---|---|---|
| 1 | Python 3.13.8 مثبت | ✅ | `python --version` → `Python 3.13.8` (`C:\Program Files\Python313\python.exe`) |
| 2 | أحدث PySide6 = 6.12.0 | ✅ | `pip index versions PySide6` → `LATEST: 6.12.0` (القائمة: 6.12.0 … 6.8.0.2) |
| 3 | دعم Python 3.13 منذ 6.8.1 | ✅ | توثيق Qt الرسمي عبر Context7 `/websites/doc_qt_io_qtforpython-6` — بند PYSIDE-2751 "Support for Python 3.13 has been added" |
| 4 | المشروع يستخدم customtkinter | ✅ | `requirements.txt:3` و`app.py:21` و`ui/*.py` جميعها |
| 5 | عدد أسطر الواجهة (§4) | ✅ | قياس فعلي لكل ملف عبر PowerShell `Measure-Object -Line` |
| 6 | `after(100)` في الـ controller | ✅ | `core/download_controller.py:478-479` (مصدر مقروء حرفيًا) |
| 7 | 491 اختبارًا | ✅ | `pytest --collect-only -q` → `491 tests collected in 2.45s` |
| 8 | فحص المتطلبات في سطر 111 من `build_windows.ps1` | ✅ | grep: `& python -c "import PyInstaller, PIL, customtkinter"` |
| 9 | قائمة excludes في spec | ✅ | `ytdownloader.spec:40-48` (تستثني test/pytest/unittest/tcl8/devscripts/ytdlp_plugins/bundle — لا tkinter) |
| 10 | hooks PySide6 رسمية في PyInstaller | ✅ | PyInstaller 6.22.3: `hook-PySide6.py` + 57 ملف وحدة (QtWidgets/QtGui/QtCore…) في مجلد hooks |
| 11 | offscreen + رموز Qt الـ 18 | ✅ | تشغيل فعلي: `QT_QPA_PLATFORM=offscreen` مع استيراد كل الرموز (QMainWindow…QPalette) + RTL — نجح على PySide6 6.9.3 |
| 12 | ~~"15 finding كلها E402"~~ | ⚠️ منحرف | **الواقع:** 14 × E402 + 1 × F401 (unused-import قابل للإصلاح) — `ruff check . --statistics`. الـ `FAQ.md:140` نفسه يقول "all E402" وهو غير دقيق |