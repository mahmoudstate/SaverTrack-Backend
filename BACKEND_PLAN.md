# SaverTrack Backend - Bank Linking Plan

مرجع تنفيذي لمشروع الـ backend المسؤول عن ربط البنوك الحقيقية (Open Banking) بتطبيق Saver.
هذا الملف يُقرأ في بداية كل جلسة عمل على المشروع ويُمشى عليه بالترتيب.

> ملاحظة: تطبيق Saver الحالي (React + Capacitor/iOS، local-only) يبقى كما هو تمامًا.
> هذا المشروع منفصل بالكامل: repo مستقل، سيرفر مستقل. الموبايل يكلّم هذا الـ backend عبر HTTPS فقط.

---

## 1. الهدف (Goal)

- المستخدم يربط حسابه البنكي الحقيقي (UK) مرة واحدة.
- الـ backend يسحب الأرصدة + المعاملات تلقائيًا ويصنّفها.
- الموبايل يعرض البيانات دي جنب الحسابات اليدوية الحالية.
- المصروفات تتسجّل أوتوماتيك بدل الإدخال اليدوي.

---

## 2. مزود الخدمة (Provider)

- **البداية: Plaid** (Sandbox أولًا، ثم Production لاحقًا بعد تسجيل الشركة).
- Client ID (Sandbox): محفوظ في `.env` (NOT committed).
- Sandbox secret: محفوظ في `.env` فقط، لا يُكتب في أي ملف داخل git.
- التصنيف يجي جاهز من Plaid عبر `personal_finance_category` (v2).

### قاعدة معمارية إلزامية: Provider Abstraction Layer
- كل الكود يتعامل مع **شكل بيانات موحّد خاص بنا** (`NormalizedTransaction`, `NormalizedAccount`)، مش شكل Plaid الخام.
- ملف واحد فقط (`providers/plaid.js`) هو اللي "يعرف" Plaid ويحوّل ردوده للشكل الموحّد.
- الهدف: لو غيّرنا لـ TrueLayer/Yapily مستقبلًا، نكتب adapter جديد فقط، والباقي (DB، الموبايل، التصنيف) لا يتغيّر.

---

## 3. الـ Tech Stack

- **Runtime:** Node.js (نفس لغة التطبيق، JS).
- **Framework:** Fastify (أخف وأسرع من Express، ومناسب للـ API).
- **Database:** PostgreSQL.
- **ORM/Query:** Prisma أو knex (يُقرّر أول جلسة).
- **Token encryption:** libsodium (`sodium-native`) أو AWS KMS لاحقًا في Production.
- **Hosting (لاحقًا):** Render أو Railway للبداية، ثم AWS لو كبر.
- **Plaid SDK:** `plaid` (official Node client).

---

## 4. الـ Data Model (مبدئي)

```
users
  id, email, created_at

bank_connections           # كل ربط ببنك واحد لمستخدم
  id, user_id, provider ('plaid'), item_id,
  access_token_encrypted, cursor (for transactions/sync),
  institution_name, status, created_at, updated_at

accounts                   # الحسابات داخل كل connection
  id, connection_id, provider_account_id, name, type,
  mask, current_balance, available_balance, currency, updated_at

transactions               # الشكل الموحّد
  id, account_id, provider_txn_id, amount, currency, date,
  merchant_name, raw_description, category (Saver category),
  provider_category, pending (bool), created_at
```

---

## 5. الـ Endpoints (API بتاعتنا احنا)

كل الـ endpoints محميّة بـ auth للمستخدم (JWT أو session، يُقرّر لاحقًا).

```
POST /link/token            -> ينشئ Plaid link_token ويرجّعه للموبايل
POST /link/exchange         -> يستقبل public_token من الموبايل،
                               يبدّله بـ access_token، يخزّنه مشفّر،
                               يسحب الحسابات لأول مرة
GET  /accounts              -> حسابات المستخدم (الشكل الموحّد)
GET  /transactions          -> معاملات المستخدم (مصنّفة)
POST /sync                  -> يشغّل transactions/sync يدويًا (وأيضًا يتم دوريًا)
DELETE /connections/:id     -> يفصل ربط بنك ويحذف التوكن
```

---

## 6. تدفق Plaid (Integration Flow)

1. الموبايل يطلب `POST /link/token` -> الـ backend ينشئ `link_token` (scopes: transactions).
2. الموبايل يفتح **Plaid Link** (SDK) بالـ link_token، المستخدم يختار بنكه ويسجّل دخول.
3. Plaid Link يرجّع `public_token` للموبايل.
4. الموبايل يبعت الـ public_token لـ `POST /link/exchange`.
5. الـ backend يبدّله بـ `access_token` (دائم) + `item_id`، يخزّنه **مشفّر**.
6. سحب البيانات عبر `transactions/sync` (بيستخدم cursor للتحديثات incremental).

### Sandbox testing
- بيانات دخول البنك الوهمي: `user_good` / `pass_good`.
- في Sandbox ممكن نستخدم `sandbox/public_token/create` لتخطّي واجهة Link في الاختبار السريع.

---

## 7. التصنيف (Categorization)

- **المرحلة 1:** ناخد `personal_finance_category` من Plaid ونعمل **mapping** لكاتوجريز Saver الموجودة (جدول ثابت: Plaid category -> Saver expCat).
- **المرحلة 2:** نتعلّم من تعديلات المستخدم (لو غيّر تصنيف تاجر، نفتكره لنفس التاجر مستقبلًا).
- الـ merchant name بيجي منظّف من Plaid (fill rate ~97%).

---

## 8. الأمان (Security) - إلزامي قبل أي بيانات حقيقية

- `access_token` و أي بيانات حساسة **تُخزّن مشفّرة** في الـ DB (encryption at rest على مستوى العمود).
- الأسرار (Plaid secret, DB creds, encryption key) في `.env` / secrets manager، **never in git**.
- `.gitignore` يشمل `.env` من أول commit.
- HTTPS فقط، لا توكنز في URLs أبدًا.
- التوكنز لا تصل الموبايل إطلاقًا، تعيش في الـ backend فقط.
- audit log للأحداث المهمة (ربط/فصل بنك، سحب بيانات).
- rotation/refresh: Plaid access_token دائم لكن لازم معالجة `ITEM_LOGIN_REQUIRED` (إعادة ربط).

---

## 9. المراحل (Milestones)

- [ ] **M0 - Setup:** repo، npm init، Fastify، Postgres، `.env`، `.gitignore`، README.
- [ ] **M1 - Link flow:** `/link/token` + `/link/exchange` شغّالين على Sandbox.
- [ ] **M2 - Data pull:** سحب accounts + transactions وتطبيع الشكل + تخزينهم.
- [ ] **M3 - Categorization mapping:** جدول Plaid->Saver + عرض مصنّف.
- [ ] **M4 - Encryption:** تشفير التوكنز + مراجعة أمان.
- [ ] **M5 - Mobile integration:** ربط تطبيق Saver بالـ backend (Plaid Link SDK + API calls).
- [ ] **M6 - Production readiness:** تسجيل الشركة، استبيان risk + security، سياسة الأمان، الانتقال لـ Production.

---

## 10. Production Readiness (لاحقًا)

- يتطلب **شركة مسجّلة في UK** (Ltd أو Sole trader) لاستبيان risk diligence.
- استبيان security diligence: نجاوبه بناءً على النظام الحقيقي بعد ما نبنيه (M4).
- مسودة security policy: نكتبها كمرجع قبل الاستبيان (سريعة، مؤجّلة حاليًا بطلب المستخدم).
- Free trial في Plaid: Production access لـ 10 اتصالات حقيقية (للتجربة الأولى الحقيقية).

---

## 11. قرارات مؤجّلة (Open decisions)

- Auth للمستخدمين: JWT مؤقت للتطوير، القرار النهائي في M1.
- ORM: Prisma vs knex.
- Hosting النهائي.
- هل الحسابات المربوطة تظهر كنوع منفصل في الموبايل أم تندمج مع نظام البنوك الحالي.
