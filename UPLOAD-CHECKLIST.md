# قائمة الرفع — fire-smoke-detector

هذا المجلد يحوي الملفات التي يمكن توليدها بأمان. **الكود المصدري ليس هنا** —
ارفعه من جهازك.

---

## ١. ما في هذا المجلد (جاهز للرفع كما هو)

```
README.md              نصّك الأصلي + قسم "حالة المشروع"
.gitignore             يستثني الأسرار والنماذج والسجلات
.env.example           قالب أسرار Telegram بقيم وهمية
config.example.yaml    قالب الإعداد بعناوين نطاق التوثيق (192.0.2.x)
LICENSE                MIT باسمك
models/README.md       شرح الملفات المتوقعة وسبب عدم رفعها
models/labels.txt      0 fire / 1 smoke
results/README.md      ما سيُقاس بعد وصول العتاد
```

**تنبيه:** عندك أصلًا `.env.example` و`config.example.yaml` في مشروعك.
قارنهما بما هنا و**قدّم نسختك** إن اختلفت أسماء المفاتيح — ملفاتي مشتقّة من
جدول الإعداد في README، وقد لا تطابق الكود حرفًا بحرف.

---

## ٢. ما ترفعه من جهازك

```
Dockerfile
docker-compose.yml
docker-compose.cpu.yml
requirements.txt
src/main.py
src/config.py
src/camera.py
src/detector.py
src/alerts.py
src/healthcheck.py
src/utils/logging.py
scripts/run_local_mock.sh
scripts/test_config.yaml
scripts/export_colab.md
scripts/train_merged_colab.ipynb
scripts/deploy_pi.md
```

---

## ٣. ما لا يُرفع أبدًا

| الملف | السبب |
|---|---|
| `.env` | توكن Telegram و chat_id الحقيقية |
| `config.yaml` | روابط RTSP بكلمات مرور الكاميرات وعناوينها |
| `models/*.tflite` | حجم كبير، ويُعاد توليده من دليل التصدير |
| `logs/` | قد تحوي لقطات من كاميرات حقيقية |

---

## ٤. افحص قبل الرفع

### أ. الأسرار المكتوبة داخل الكود

افتح كل ملف `.py` وابحث عن:

```
token      chat_id      rtsp://      password      192.168.      api_key
```

أي قيمة حقيقية تجدها انقلها إلى `.env` أو `config.yaml` واقرأها من هناك.

### ب. الدفتر `train_merged_colab.ipynb`

دفاتر Jupyter **تحفظ مخرجات التنفيذ داخل الملف**. قد تحوي مسارات جهازك،
ومفاتيح Kaggle أو Roboflow، وصورًا من بياناتك. افتحه ونفّذ
`Kernel → Restart & Clear Output` ثم احفظه قبل الرفع.

### ج. `scripts/test_config.yaml`

اسمه يوحي بأنه للاختبار، لكنه قد يحوي روابط RTSP حقيقية جرّبتها.
افتحه وتحقّق.

### د. لقطات الشاشة

إن أضفت صورًا توضيحية، احجب عناوين IP وأسماء الكاميرات ونصوص Telegram.

---

## ٥. الرفع

الواجهة: `Add file → Upload files`، واسحب المجلدات **بعد** استبعاد ما في القسم ٣.

> **مهم:** `.gitignore` **لا يعمل** مع الرفع عبر المتصفح. يحمي فقط عند
> استخدام `git` من الطرفية. عبر المتصفح، الاستبعاد يدوي بالكامل.

عبر الطرفية (أأمن):

```bash
git clone https://github.com/Mohsen-Iseidyah/fire-smoke-detector.git
cd fire-smoke-detector
# انسخ ملفاتك هنا
git add .
git status          # اقرأ القائمة سطرًا سطرًا قبل المتابعة
git commit -m "Add detection pipeline, Docker deployment and configuration templates"
git push
```

`git status` هو خط دفاعك الأخير. إن ظهر فيه `.env` أو `config.yaml` فتوقّف
وراجع `.gitignore`.

---

## ٦. إن تسرّب سرّ

حذف الملف لا يكفي — يبقى في تاريخ Git.

1. عطّل التوكن فورًا من @BotFather (`/revoke`) وأنشئ غيره.
2. غيّر كلمات مرور الكاميرات المكشوفة.
3. ثم عالج التاريخ أو احذف المستودع وأعد إنشاءه نظيفًا.

الخطوة الأولى هي العاجلة. الباقي يحتمل التأجيل.
