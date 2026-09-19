# نظام كشف الحرائق والدخان الذكي (Raspberry Pi 4 + Coral Edge TPU)

نظام مراقبة إنتاجي يعمل **24/7** بمعالجة **محلية بالكامل**: يسحب بث كاميرات
المراقبة عبر **RTSP**، ويحلّل الإطارات على مسرّع **Coral Edge TPU** بحثاً عن
لهب/دخان، وعند رصد خطر يلتقط صورة اللحظة ويرسلها فوراً عبر **Telegram**.

> لا يُرفع أي فيديو إلى السحابة. الإنترنت مطلوب فقط لحظة إرسال التنبيه.

---

## حالة المشروع

| المرحلة | الحالة |
|---|---|
| المسار الكامل في وضع المحاكاة (`mock`) | ✅ يعمل |
| اختبار دقّة النموذج على المعالج (`cpu` / x86) | 🔄 قيد التنفيذ |
| النشر على Raspberry Pi 4 + Coral (`edgetpu`) | ⏳ لم يُختبر بعد |
| قياس الأداء وعدد الكاميرات المتزامنة | ⏳ بانتظار العتاد |

الكود مكتمل والمعمارية مستقرّة. أرقام الأداء تُضاف إلى `results/` بعد التشغيل
على العتاد الفعلي. المخاطرة المعروفة الوحيدة عند النقل موثّقة في قسم النموذج
أدناه (إصدار `tflite-runtime`).

---

## المحتويات

1. [البنية المعمارية](#البنية-المعمارية)
2. [المتطلبات (عتاد + برمجيات)](#المتطلبات)
3. [تجربة سريعة دون عتاد (وضع المحاكاة)](#تجربة-سريعة-دون-عتاد)
4. [الإعداد الكامل من الصفر](#الإعداد-الكامل-من-الصفر)
5. [نموذج الكشف (YOLO) والأوضاع](#نموذج-الكشف-yolo-والأوضاع-الثلاثة)
6. [إعداد بوت Telegram](#إعداد-بوت-telegram)
7. [روابط RTSP للكاميرات](#روابط-rtsp-للكاميرات)
8. [مرجع ملف الإعداد](#مرجع-ملف-الإعداد)
9. [التشغيل والمراقبة](#التشغيل-والمراقبة)
10. [حل المشكلات](#حل-المشكلات)
11. [الأمان](#الأمان)
12. [مطابقة معايير القبول](#مطابقة-معايير-القبول)

---

## البنية المعمارية

```
                 ┌──────────────┐   ┌──────────────┐        ┌──────────────┐
  RTSP cam 1 ──► │ CameraWorker │──►│              │        │              │
  RTSP cam 2 ──► │ CameraWorker │──►│  طابور مشترك  │──►  ┌─►│ DetectorWorker│
     ...         │     ...      │   │ (shared queue)│     │  │  (Coral TPU)  │
  RTSP cam N ──► │ CameraWorker │──►│              │     │  └──────┬───────┘
                 └──────────────┘   └──────────────┘     │         │ خطر؟
   كل كاميرا في خيط مستقل + إعادة اتصال        TPU واحد ──┘         ▼
   وأخذ عيّنة إطار كل فترة قابلة للضبط          (استدلال متسلسل)  ┌──────────────┐
                                                               │ AlertManager │──► Telegram
                                                               │ cooldown+retry│   (sendPhoto)
                                                               └──────────────┘
```

- **كاميرا = خيط مستقل:** سحب RTSP وإعادة اتصال بتباعد متزايد. تعطّل كاميرا
  لا يُسقط البقية ولا العملية (عزل أعطال).
- **مسرّع Coral واحد:** كل الاستدلال يمرّ عبر **طابور مشترك** ويُنفَّذ بالتسلسل
  في خيط واحد (لا يمكن لـ TPU واحد خدمة 8 كاميرات بالتوازي).
- **التنبيه في خيط منفصل:** الإرسال البطيء أو الفاشل لا يُعطّل الاستدلال،
  مع إعادة محاولة وفترة تهدئة لكل كاميرا.
- **نبضة حياة (heartbeat):** سجلّ دوري + ملف صحة تقرؤه فحوصات Docker.

---

## المتطلبات

### العتاد (ثوابت المشروع)
- Raspberry Pi 4 Model B — **8GB RAM**.
- Raspberry Pi OS — **64-bit (Bookworm)**.
- Google Coral **USB** Accelerator، موصول بمنفذ **USB 3.0** مباشرةً **دون موزّع (Hub)**.
- اتصال **سلكي (Ethernet)** بالشبكة المحلية.
- مصدر بث: **DVR/NVR** يوفّر **RTSP**.
- بطاقة microSD سعة 64GB (احتياطات الموثوقية مفعّلة لأنها فئة استهلاكية).

### البرمجيات
- Docker + Docker Compose plugin (الخطوات أدناه).
- **لا تثبّت أي مكوّن Coral على النظام المضيف** — كل شيء داخل الحاوية (Python 3.9).

---

## تجربة سريعة دون عتاد

للتأكد أن المسار كامل (سحب ← كشف ← تنبيه) قبل توصيل أي عتاد، شغّل كاشف المحاكاة
مع مصدر اصطناعي وحفظ التنبيهات على القرص (دون Coral ولا كاميرا ولا Telegram):

```bash
cd fire-detector
pip install -r requirements.txt   # أو استخدم Docker مباشرة
bash scripts/run_local_mock.sh
```

سترى سجلّات الكشف، وتُحفظ صور/نصوص التنبيهات في `logs/alerts/`.
أوقف بـ `Ctrl+C`.

> داخل Docker يمكنك تحقيق المثل بضبط `DETECTOR_MODE=mock` و`ALERTS_DRY_RUN=1`.

---

## الإعداد الكامل من الصفر

### 1) تجهيز Raspberry Pi
ثبّت Raspberry Pi OS **64-bit (Bookworm)**، فعّل SSH، ووصّل الكيبل الشبكي.
حدّث النظام:
```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

### 2) تثبيت Docker
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# سجّل الخروج والدخول (أو أعد الإقلاع) لتفعيل صلاحية docker بدون sudo.
newgrp docker
docker --version && docker compose version
```

### 3) توصيل Coral
- وصّل المسرّع بمنفذ **USB 3.0** (الأزرق) مباشرةً، **دون موزّع**.
- تحقق من ظهوره:
```bash
lsusb
# قبل تحميل البرنامج الثابت: 1a6e:089a (Global Unichip)
# بعد التهيئة من مرحلتين: 18d1:9302 (Google Inc) — قد يستغرق 5–10 ثوانٍ
```
> **لا حاجة لتثبيت libedgetpu على المضيف.** المكتبة داخل الحاوية، ونحن نمرّر
> منفذ USB بالكامل عبر `docker-compose.yml` (انظر القيد المتعلق بإعادة التعريف).

### 4) إحضار المشروع وإعداده
```bash
# انسخ مجلد المشروع إلى الجهاز ثم:
cd fire-detector

# 1. الأسرار:
cp .env.example .env
nano .env            # ضع توكن Telegram و chat_id الحقيقيين

# 2. الإعداد:
cp config.example.yaml config.yaml
nano config.yaml     # عرّف الكاميرات وروابط RTSP وفعّل ما يلزم
```

### 5) التشغيل
```bash
docker compose up -d        # يبني الصورة أول مرة ثم يشغّل الحاوية
docker compose logs -f      # متابعة السجلّات
```
- مع `detection.mode: mock` سيعمل النظام فوراً بكاشف محاكاة.
- بعد تجهيز النموذج الحقيقي (القسم التالي) اجعل `mode: cpu` (اختبار) أو `mode: edgetpu` (إنتاج) وأعد التشغيل:
```bash
docker compose up -d --build
```

### 6) التحقق من العمل
```bash
docker compose ps           # الحالة + healthy/unhealthy
docker compose logs -f | grep HEARTBEAT
```
ابحث في السجلّ عن `tpu_active=True` و`Detected N Coral Edge TPU(s)`.

---

## نموذج الكشف (YOLO) والأوضاع الثلاثة

الكشف يتم بنموذج **YOLOv8n** للكشف عن **fire/smoke** (صناديق + فئة + ثقة)،
يُحمَّل كملف **TFLite** ويُفكّ في `src/detector.py`. ثلاثة أوضاع صريحة عبر
`detection.mode` (أو متغير البيئة `DETECTOR_MODE` بأولوية أعلى):

| الوضع | الجهاز | الغرض | ملف النموذج |
|---|---|---|---|
| `mock` | — | تشغيل المسار كاملاً بلا نموذج (اختبار سريع) | لا يلزم |
| `cpu` | المعالج (x86) | **اختبار دقّة** النموذج على الكمبيوتر الآن | `*.tflite` (int8) |
| `edgetpu` | Coral USB | **الإنتاج** على الراسبيري باي | `*_edgetpu.tflite` |

> **تمييز جوهري:** دقّة الكشف لا تتغيّر بتغيّر العتاد — نتيجة CPU (int8) = نتيجة
> Coral. لذلك نختبر **الدقّة** الآن على CPU، ونؤجّل قياس **الأداء** (السرعة/عدد
> الكاميرات المتزامنة) للراسبيري باي + Coral.

### أ) تصدير النموذج (على x86/Colab)

> ⚠️ مُجمِّع Edge TPU لا يعمل على ARM. التصدير/التجميع يتم على **Colab/x86**.

اتبع الدليل خطوة بخطوة: **[`scripts/export_colab.md`](scripts/export_colab.md)**.

> 🎯 **لتحسين الدقّة (إنذارات كاذبة أقل + كشف نار أصغر):** درّب نموذجك على بيانات
> مدمجة تحوي **صوراً سالبة** عبر الدفتر الجاهز
> [`scripts/train_merged_colab.ipynb`](scripts/train_merged_colab.ipynb).
> السبب الجذري للإنذارات الكاذبة هو تدريب النموذج على صور نار فقط بلا مشاهد عادية.
يُنتج من أوزان YOLOv8n ملفّين:
- `fire_smoke.tflite` (int8) → لوضع `cpu`.
- `fire_smoke_edgetpu.tflite` → لوضع `edgetpu`.

ضعهما في `models/` مع `labels.txt` (موجود مسبقاً: `0 fire` / `1 smoke`).

### ب) اختبار الدقّة على الكمبيوتر (وضع cpu)

صورة Docker منفصلة للمعالج (x86، بلا Coral) — لا تلمس صورة الإنتاج:
```bash
# 1) ضع fire_smoke.tflite في models/ واضبط في config.yaml:
#       detection.model_path: /models/fire_smoke.tflite
# 2) شغّل (الوضع cpu مفروض عبر البيئة DETECTOR_MODE=cpu):
docker compose -f docker-compose.cpu.yml up --build
docker compose -f docker-compose.cpu.yml logs -f
```
اعرض ناراً آمنة أمام الكاميرا (شمعة/ولاعة/فيديو نار على شاشة) وراقب `backend=cpu`
في heartbeat ووصول التنبيه بصورة عليها صندوق الكشف. اضبط
`detection.confidence_threshold` لموازنة الحساسية مقابل الإنذارات الكاذبة.

### ج) النقل للإنتاج (راسبيري باي + Coral)

> 📘 **دليل تفصيلي خطوة بخطوة:** [`scripts/deploy_pi.md`](scripts/deploy_pi.md)
> (تجهيز الـ Pi، تمرير USB، البناء ARM64، قائمة تحقق، **قياس الأداء وعدد الكاميرات**).
> الملخّص أدناه:

1. ضع `fire_smoke_edgetpu.tflite` في `models/` واضبط في `config.yaml`:
   `detection.model_path: /models/fire_smoke_edgetpu.tflite` و`detection.mode: edgetpu`.
2. في `docker-compose.yml` أزِل التعليق عن تمرير USB:
   ```yaml
   devices:
     - /dev/bus/usb:/dev/bus/usb
   ```
3. أعد البناء لمعمارية **ARM64** وشغّل:
   ```bash
   docker compose up -d --build
   docker compose logs -f | grep -E "HEARTBEAT|Edge TPU"
   ```
   ابحث عن `tpu_active=True` و`Detected N Coral Edge TPU(s)`. **لا ارتداد صامت
   للمعالج:** إن فشل Coral يُسجَّل الخطأ وتُعاد المحاولة.

> ⚠️ **مخاطرة معروفة لم تُختبر بعد (إصدار tflite-runtime):** صورة الإنتاج تثبّت
> `tflite-runtime==2.5.0` (قديمة، 2021)، وقد لا تدعم بعض عمليات رأس YOLOv8 التي
> تعمل على المعالج ضمن نموذج edgetpu المُجمَّع. **إن فشل تحميل نموذج
> `*_edgetpu.tflite` برسالة «عملية غير مدعومة / unsupported op»،** فارفع
> `tflite-runtime` (والإصدار المطابق له من `pycoral`) في `requirements.txt` ثم
> أعد البناء. هذا هو المكان الوحيد المتوقَّع لمفاجأة عند النقل للراسبيري باي.

---

## إعداد بوت Telegram

1. افتح **@BotFather** على Telegram، أرسل `/newbot`، واحصل على **التوكن**.
2. للحصول على `chat_id`:
   - أرسل أي رسالة للبوت (أو أضِفه إلى مجموعة وأرسل رسالة فيها).
   - افتح: `https://api.telegram.org/bot<TOKEN>/getUpdates`
   - انسخ `chat.id` (للمجموعات يكون رقماً سالباً يبدأ بـ `-100`).
3. ضع القيم في `.env`:
```env
TELEGRAM_BOT_TOKEN=123456789:AA....
TELEGRAM_CHAT_IDS=123456789,-1001234567890   # عدة مستلمين بفاصلة
```
> لإرسال التنبيه لعدة مسؤولين دفعة واحدة، استخدم **معرّف مجموعة** أو عدة معرّفات.

---

## روابط RTSP للكاميرات

**Hikvision (الافتراضي):**
```
rtsp://USER:PASS@IP:554/Streaming/Channels/101
   101 = الكاميرا 1 / البث الرئيسي   |   102 = البث الفرعي
   201 = الكاميرا 2 / البث الرئيسي   |   ...
```
**Dahua:**
```
rtsp://USER:PASS@IP:554/cam/realmonitor?channel=1&subtype=0
```
> تحقق من الرابط بـ VLC أو `ffprobe "rtsp://..."` قبل وضعه في الإعداد.
> استخدام **البث الفرعي (sub stream)** يقلّل الحمل ويكفي للكشف غالباً.

---

## مرجع ملف الإعداد

| المفتاح | الوصف |
|---|---|
| `cameras[].name` | اسم فريد للكاميرا (يظهر في التنبيه والسجلّ). |
| `cameras[].rtsp_url` | رابط RTSP (يحوي بيانات الدخول — لا يُطبع في السجلّ). |
| `cameras[].enabled` | تشغيل/إيقاف مراقبة هذه الكاميرا. |
| `detection.mode` | وضع الكاشف: `mock` \| `cpu` \| `edgetpu` (يتجاوزه `DETECTOR_MODE`). |
| `detection.model_path` | مسار نموذج TFLite (`*.tflite` لـ cpu، `*_edgetpu.tflite` لـ edgetpu). |
| `detection.labels_path` | مسار `labels.txt`. |
| `detection.confidence_threshold` | حدّ الثقة (0–1) لاعتبار الحدث خطراً. |
| `detection.inference_interval_seconds` | فترة أخذ عيّنة إطار لكل كاميرا. |
| `detection.draw_boxes` | رسم صناديق الكشف على صورة التنبيه (`true`/`false`). |
| `detection.consecutive_hits` | عدد الكشوف المطلوبة قبل التنبيه (1 = فوري). |
| `detection.confirm_window_seconds` | النافذة الزمنية لاحتساب الكشوف (0 = اشتراط التتالي). |
| `detection.tiles` | تقسيم الإطار لكشف النار البعيدة (1 = بلا تقسيم، 2 = شبكة 2x2). |
| `detection.save_detections_dir` | حفظ لقطات الكشف لمراجعتها وإعادة تدريب النموذج ("" = مُعطَّل). |
| `detection.alert_labels` | الفئات التي تُعتبر خطراً (مثل `fire`,`smoke`). |
| `alerts.cooldown_seconds` | تهدئة لكل كاميرا بين التنبيهات. |
| `alerts.send_retries` | عدد محاولات إعادة الإرسال عند فشل Telegram. |
| `alerts.dry_run` | `true` = حفظ التنبيه محلياً بدل إرساله (اختبار). |
| `alerts.caption_language` | لغة نص التنبيه: `ar`/`en`/`both`. |
| `system.max_enabled_cameras` | الحد الأقصى للمُفعّلة (8 لمسرّع واحد). |
| `system.reconnect_backoff_seconds` | تباعد إعادة الاتصال المتزايد. |
| `system.heartbeat_interval_seconds` | دورية نبضة الحياة. |
| `system.queue_max_size` | حجم طابور الاستدلال المشترك. |

**الأسرار** (`TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_IDS`) تُقرأ من البيئة (`.env`) فقط.

---

## التشغيل والمراقبة

```bash
docker compose up -d           # تشغيل
docker compose logs -f         # السجلّات الحية
docker compose ps              # الحالة الصحية
docker compose restart         # إعادة تشغيل
docker compose down            # إيقاف
```
- **إعادة تشغيل تلقائية:** `restart: unless-stopped` تعيد الحاوية بعد أي انهيار
  وبعد إقلاع الجهاز عقب انقطاع الكهرباء.
- **نبضة الحياة:** سطر `HEARTBEAT` كل فترة يبيّن عدد الكاميرات المتصلة، حجم
  الطابور، عدد عمليات الاستدلال، وحالة `tpu_active`. ويُكتب ملف `logs/heartbeat`
  الذي يقرؤه `healthcheck` لتحديد صحة الحاوية.
- **التنبيهات (dry-run):** تُحفظ في `logs/alerts/`.

---

## حل المشكلات

**Coral غير مكتشَف (`No Coral Edge TPU detected`):**
- استخدم منفذ **USB 3.0** (الأزرق) مباشرةً **دون موزّع**.
- افحص `lsusb` (راجع المعرّفات أعلاه). افصل وأعد التوصيل لإعادة التهيئة.
- تأكد من بقاء `devices: - /dev/bus/usb:/dev/bus/usb` في `docker-compose.yml`.
- إن لزم، فعّل `privileged: true` (مُعلّق في ملف compose).

**انهيار صامت / إعادة تشغيل متكررة:**
- غالباً نقص الذاكرة المشتركة — زِد `shm_size` في `docker-compose.yml`
  (مثلاً `512mb` أو أكثر مع زيادة عدد/دقة الكاميرات).

**فشل بث RTSP:**
- تحقق من الرابط بـ `ffprobe`/VLC، ومن صحة المستخدم/كلمة المرور والمنفذ.
- جرّب البث الفرعي. النقل TCP مضبوط مسبقاً عبر متغير البيئة في compose.
- إعادة الاتصال تلقائية؛ راقب السجلّ لرسائل `reconnecting`.

**الأداء/الحرارة:**
- لرفع الإنتاجية يمكن استبدال `libedgetpu1-std` بـ `libedgetpu1-max` في
  `Dockerfile` (يعمل أحرّ — وفّر تهوية). راجع التعليق هناك.
- قلّل الحمل بزيادة `inference_interval_seconds` أو تقليل الكاميرات المُفعّلة.

**`tpu_active=False` في السجلّ (وضع حقيقي):**
- يعني الكاشف لم يستخدم Coral — راجع التوصيل/التمرير. النظام لا يرتد للمعالج
  بصمت بل يسجّل الخطأ ويحاول إعادة التهيئة.

---

## الأمان

- **لا أسرار داخل الكود:** توكن Telegram و`chat_id` وبيانات الكاميرات تُقرأ من
  `.env` / `config.yaml` خارج مستودع الكود.
- `.gitignore` يستثني `.env` و`config.yaml` وملفات النماذج والسجلّات.
- **لا تُطبع الأسرار في السجلّات إطلاقاً:** فلتر إخفاء يُقنّع التوكن وكلمات مرور
  RTSP في كل رسائل السجلّ تلقائياً.
- التحليل **محلي بالكامل**؛ لا يُرفع فيديو لأي خدمة سحابية.

---

## مطابقة معايير القبول

| المعيار | أين يتحقق |
|---|---|
| يعمل بالكامل داخل Docker بأمر `docker compose up -d` | `Dockerfile` + `docker-compose.yml` |
| اكتشاف Coral واستخدامه فعلياً (لا ارتداد للمعالج) وتسجيله | `detector.py` (`YoloDetector`, heartbeat) |
| سحب عدة كاميرات RTSP وتحليلها آنياً | `camera.py` + الطابور المشترك |
| احترام `enabled` ورفض > 8 مُفعّلة برسالة واضحة | `config.py` (التحقق الصارم) |
| تنبيه Telegram **مع صورة** + اسم/وقت/ثقة | `alerts.py` (`sendPhoto`) |
| فترة تهدئة تمنع تكرار التنبيهات لنفس الكاميرا | `alerts.py` (cooldown) |
| إعادة اتصال تلقائية دون إسقاط النظام/البقية | `camera.py` (backoff + عزل) |
| معالجة انقطاع الإنترنت بإعادة محاولة ثم تسجيل | `alerts.py` (retry) |
| لا أسرار في الكود ولا في السجلّات | `.env`/`config.py` + فلتر الإخفاء في `utils/logging.py` |
| إعادة تشغيل الجهاز تُعيد تشغيل النظام | `restart: unless-stopped` |
| README يشرح الإعداد من الصفر + النموذج + تمرير USB | هذا الملف |

---

## بنية المشروع

```
fire-detector/
├── docker-compose.yml      # restart, shm_size, تمرير USB، healthcheck
├── Dockerfile              # Python 3.9 + libedgetpu + tflite-runtime + opencv
├── requirements.txt
├── .env.example            # الأسرار (نموذج، دون قيم حقيقية)
├── .gitignore
├── config.example.yaml     # قالب الإعداد
├── README.md
├── models/                 # النموذج + labels.txt (راجع models/README.md)
├── scripts/
│   ├── test_config.yaml    # إعداد اختبار محلي
│   └── run_local_mock.sh   # تجربة سريعة دون عتاد
└── src/
    ├── main.py             # نقطة الدخول + تشغيل/إيقاف منظّم + heartbeat
    ├── config.py           # تحميل الإعداد + تحقق صارم
    ├── camera.py           # سحب RTSP + إعادة اتصال backoff + عزل أعطال
    ├── detector.py         # كاشف YOLO (cpu/edgetpu) + طابور استدلال مشترك + محاكاة
    ├── alerts.py           # Telegram sendPhoto + retry + cooldown
    ├── healthcheck.py      # فحص صحة الحاوية (نبضة الحياة)
    └── utils/logging.py    # سجلّات + إخفاء الأسرار
```

---

### ملاحظات وافتراضات
- صيغة RTSP الافتراضية **Hikvision** (قابلة للتغيير لكل كاميرا في `config.yaml`).
- النظام يبدأ بوضع **المحاكاة** (`mode: mock`) حتى تُجهّز نموذج YOLO (راجع قسم النموذج).
- الكشف بنموذج **YOLOv8n** (كشف صناديق) لا تصنيف؛ ثلاثة أوضاع: `mock`/`cpu`/`edgetpu`.
- إصدارات الحزم في `requirements.txt` خط أساس معروف؛ قد تحتاج لضبطها حسب صورة
  نظامك. عُلّقت بدائل (`libedgetpu1-max`, `privileged`) في مواضعها.
