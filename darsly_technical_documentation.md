# درسلي — التوثيق التقني الكامل (Technical Documentation)
## Database Schema Design & REST API Contracts

**الإصدار:** 1.0 (مبني على PRD المواصفات التقنية الكاملة، 2026-07-10 + قرارات أحمد 2026-07-14/15)
**تاريخ الإعداد:** 2026-07-21
**المُعِد:** Senior Software Architect
**الجمهور المستهدف:** Backend Developers, Frontend Developers, QA, DevOps

---

## ⚠️ ملاحظة معمارية مهمة — التعديل المطلوب من صاحب المشروع

> **تم إلغاء ميزة "المحفظة المالية" (4.20 في الـ PRD الأصلي) بالكامل بناءً على طلبك الصريح.**
> هذا التوثيق **لا يتضمن** كيان `Wallet`/`WalletBalance`/`WalletTransaction`، ولا مفهوم "شحن رصيد" أو "الدفع من المحفظة".
> بدلاً من ذلك:
> - **الطالب يدفع مباشرة** في كل مرة (كارت / فوري / إنستاباي) لكل عملية شراء (حصة، مادة، مادة تعليمية) — بدون رصيد مسبق الدفع.
> - **أرباح المدرّس تُحسب كمُجمّع (Aggregate) من جدول `payments` مباشرة**، وليس كرصيد مخزَّن في كيان منفصل. طلب السحب (`Withdrawal`) يُنشأ ويُحسب في وقت الطلب من مجموع `payments.net_amount` الناجحة غير المسحوبة سابقًا.
> - هذا التبسيط **قرار معماري مبني على طلبك المباشر**، وليس استنتاجًا. أي تفاصيل أعمق (مثل: هل يوجد نافذة انتظار (Settlement Window) قبل السحب) تبقى **قرار مفتوح** لم يُطرح بعد.

> **تنويه عام حسب قاعدة "عدم اختلاق القيم":** أي حقل أو قاعدة عمل كانت "قرار مفتوح" صراحةً في المستند الأصلي المرفوع، ولا زالت مفتوحة، تم تصميمها هنا بأفضل تخمين هندسي معقول (Best-Effort Assumption) **مع وضع علامة `🟡 Open Decision`** بجانبها في الجدول المناسب. لا تعتبر أي حقل مُعلَّم بهذه العلامة نهائيًا — راجعه مع أحمد قبل التنفيذ.

---

## جدول المحتويات

1. [الاتفاقيات العامة (Conventions)](#1-الاتفاقيات-العامة)
2. [تصميم قاعدة البيانات (Database Schema)](#2-تصميم-قاعدة-البيانات)
3. [توثيق REST API لكل ميزة](#3-توثيق-rest-api)
   - 4.1 المصادقة (Auth)
   - 4.2 لوحة التحكم (Dashboard)
   - 4.3 المواد الدراسية (Subjects)
   - 4.4 الطلاب (Students)
   - 4.5 المجموعات (Groups)
   - 4.6 الجدول (Schedule)
   - 4.7 الحضور (Attendance)
   - 4.8 الواجبات (Homework)
   - 4.9 الامتحانات (Exams)
   - 4.10 الدرجات (Grades)
   - 4.11 المدفوعات (Payments)
   - 4.12 المواد التعليمية (Learning Materials)
   - 4.13 التقارير (Reports)
   - 4.14 إشعارات النظام (Notifications)
   - 4.15 واتساب (WhatsApp)
   - 4.16 الرسائل (Messages)
   - 4.17 البحث (Search)
   - 4.18 الأرشيف (Archive)
   - 4.19 الإعدادات (Settings)
   - 4.21 الحصص الأونلاين (Online Sessions)


---

## 1. الاتفاقيات العامة

### 1.1 الأساسيات
- **Base URL:** `https://api.darsly.app/v1`
- **الصيغة:** JSON فقط لكل الطلبات والردود (`Content-Type: application/json`)، عدا رفع الملفات (`multipart/form-data`).
- **المصادقة:** `Authorization: Bearer <access_token>` (JWT) في كل طلب محمي.
- **الترقيم (Pagination):** أي Endpoint بيرجع قائمة بيستخدم `?page=1&limit=20` مع رد بالشكل:
```json
{
  "data": [ ... ],
  "meta": { "page": 1, "limit": 20, "total_items": 143, "total_pages": 8 }
}
```
- **الفلترة والفرز:** `?filter[field]=value&sort=-created_at` (السالب = تنازلي).
- **التاريخ/الوقت:** ISO 8601 UTC دايمًا في الـ API (`2026-07-21T09:30:00Z`)؛ التحويل لتوقيت القاهرة يتم في الفرونت إند.
- **معرّفات الكيانات:** UUID v4 كنص (string) في كل مكان — مش أرقام تسلسلية — لمنع تخمين/تعداد سجلات مدرّس تاني.
- **العملة:** كل المبالغ بالقروش (Integer, smallest currency unit) لتفادي أخطاء الفاصلة العشرية، والعملة الافتراضية EGP (🟡 Open Decision في المصدر — لم تُذكر صراحة، هذا افتراض هندسي قياسي).

### 1.2 شكل رد الخطأ الموحّد
كل خطأ (4xx/5xx) بيرجع بنفس الشكل:
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "رقم الموبايل مطلوب",
    "details": [
      { "field": "phone", "message": "هذا الحقل مطلوب" }
    ]
  }
}
```

### 1.3 أكواد الحالة القياسية (Status Codes) المستخدمة عبر كل الـ API
| Code | المعنى | يُستخدم متى |
|---|---|---|
| `200 OK` | نجاح قراءة/تحديث | GET, PUT, PATCH ناجحة |
| `201 Created` | نجاح إنشاء | POST أنشأ سجل جديد |
| `204 No Content` | نجاح بدون محتوى راجع | DELETE ناجح |
| `400 Bad Request` | خطأ تحقق (Validation) | حقول ناقصة/غير صحيحة |
| `401 Unauthorized` | التوكن غايب/منتهي/غير صالح | كل الـ Endpoints المحمية |
| `403 Forbidden` | التوكن صالح لكن الدور/النطاق مايسمحش | مثلاً طالب بيحاول يوصل لبيانات مدرّس تاني |
| `404 Not Found` | السجل مش موجود، أو موجود بس برّه نطاق المستخدم (لإخفاء الوجود) | |
| `409 Conflict` | تعارض حالة (مثلاً حساب مكرر، تعارض جدول) | |
| `422 Unprocessable Entity` | الطلب صحيح شكليًا لكن مخالف لقاعدة عمل | مثلاً سحب أكبر من الرصيد المتاح |
| `429 Too Many Requests` | تجاوز حد المحاولات (مثلاً OTP) | |
| `500 Internal Server Error` | خطأ سيرفر غير متوقع | |

### 1.4 قاعدة الصلاحيات العابرة (Cross-Cutting Authorization Rule)
**كل استعلام في كل Endpoint لازم يتفلتر ضمنيًا على مستوى الـ API/Database بـ:**
- لو الطالب هو المستخدم: `student_id = current_user.id` (أو ما يعادله حسب الميزة).
- لو المدرّس هو المستخدم: `teacher_id = current_user.id` — أي سجل تحت مدرّس تاني بيرجع `404 Not Found` (مش `403`) لإخفاء وجوده.
- **ولي الأمر ليس له أي Endpoint مصادقة أو جلسة في هذا الإصدار** — هو مجرد حقل `guardian_phone` على كيان `Student` ومستقبل واتساب فقط.

### 1.5 اصطلاح توثيق البنود المفتوحة
كل حقل/قاعدة عمل غير محسومة في مصدر المتطلبات يظهر في هذا التوثيق بعلامة **`🟡 Open Decision`** مع أفضل افتراض هندسي بجانبها. لا تُبنى منطق عمل نهائي على هذه الافتراضات قبل الرجوع لأحمد.

---

## 2. تصميم قاعدة البيانات

### 2.1 مخطط العلاقات المختصر (Entity Relationship Overview)

```
Account (1) ──┬── (1) Teacher ──(1)───(N) Student
              └── (1) Student

Teacher (1) ──(N) Subject ──(1)───(N) Group ──(N)───(N) Student   [عبر StudentGroupEnrollment]
Group   (1) ──(N) RecurringScheduleRule
Group   (1) ──(N) Session ──(1)───(N) AttendanceRecord ──(1)───(N) AttendanceAuditTrailEntry
Group/Student (1) ──(N) Homework ──(1)───(N) HomeworkSubmission
Subject/Group (1) ──(N) Exam ──(1)───(N) ExamQuestion
Exam    (1) ──(N) ExamSubmission
Student (1) ──(N) GradeEntry ──(0..1)── Exam
Subject/Group/Student (1) ──(N) LearningMaterial
Student (1) ──(N) Payment ──(0..1)── LearningMaterial/Session
Teacher (1) ──(N) Withdrawal
Account (1) ──(N) Notification
Student (1) ──(N) WhatsAppMessage  [عبر guardian_phone، بدون FK لحساب]
Teacher (1) ──(N) Message ──(1) Student
Account (1) ──(1) UserSettings
Account (1) ──(N) OTPCode
```

**نوع العلاقات الأساسية:**
| العلاقة | النوع |
|---|---|
| Teacher → Student | One-to-Many |
| Teacher → Subject | One-to-Many |
| Subject → Group | One-to-Many |
| Student ↔ Group | Many-to-Many (عبر `student_group_enrollments`) |
| Group → Session | One-to-Many |
| Session → AttendanceRecord | One-to-Many |
| AttendanceRecord → AttendanceAuditTrailEntry | One-to-Many |
| Homework (Group أو Student) → HomeworkSubmission | One-to-Many |
| Exam → ExamQuestion | One-to-Many |
| Exam → ExamSubmission | One-to-Many |
| Student → GradeEntry | One-to-Many |
| Exam → GradeEntry | One-to-Many (اختياري) |
| Student → Payment | One-to-Many |
| Teacher → Withdrawal | One-to-Many |
| Account → Notification | One-to-Many |
| Teacher ↔ Student → Message | One-to-Many لكل زوج |

---

### 2.2 الجداول التفصيلية

#### 2.2.1 `accounts`
الهوية الأساسية المشتركة بين المدرّس والطالب (Table Inheritance عبر `role`).

| الحقل | النوع | مطلوب | ملاحظات |
|---|---|---|---|
| `id` | UUID | ✅ (PK) | |
| `role` | ENUM(`teacher`,`student`) | ✅ | يُحدَّد وقت الإنشاء، غير قابل للتغيير |
| `phone` | VARCHAR(20), UNIQUE nullable | ⚠️ (موبايل أو إيميل مطلوب، مش الاتنين) | صيغة E.164 |
| `email` | VARCHAR(255), UNIQUE nullable | ⚠️ | صيغة RFC 5322 |
| `password_hash` | VARCHAR(255) | ✅ | bcrypt/argon2 |
| `preferred_language` | ENUM(`ar`,`en`) | ✅ | افتراضي `ar`؛ مصدر الحقيقة لـ RTL |
| `is_active` | BOOLEAN | ✅ | افتراضي `true` |
| `created_at` | TIMESTAMP | ✅ | |
| `updated_at` | TIMESTAMP | ✅ | |

> **قيد تكامل:** `CHECK (phone IS NOT NULL OR email IS NOT NULL)` + فهرس فريد جزئي على كل من `phone` و`email` عبر جدول `accounts` بالكامل (مش لكل مدرّس على حدة) — مطابقةً لنص المصدر.

#### 2.2.2 `teachers`
| الحقل | النوع | مطلوب | PK/FK | ملاحظات |
|---|---|---|---|---|
| `id` | UUID | ✅ | PK, FK → `accounts.id` | نفس الـ id بتاع الحساب (1:1) |
| `full_name` | VARCHAR(255) | ✅ | | |
| `profile_photo_url` | VARCHAR(500) | ❌ | | |
| `registration_link_slug` | VARCHAR(100), UNIQUE | ✅ | | يُستخدم في رابط التسجيل الذاتي `/register/{slug}` |
| `created_at` | TIMESTAMP | ✅ | | |

#### 2.2.3 `students`
| الحقل | النوع | مطلوب | PK/FK | ملاحظات |
|---|---|---|---|---|
| `id` | UUID | ✅ | PK, FK → `accounts.id` | |
| `teacher_id` | UUID | ✅ | FK → `teachers.id` | إجباري حتى وقت "معلّق"، لأن رابط التسجيل نفسه يحدد المدرّس |
| `full_name` | VARCHAR(255) | ✅ | | |
| `guardian_phone` | VARCHAR(20) | 🟡 Open Decision (افتراض: إجباري) | | القناة الوحيدة لولي الأمر (واتساب) |
| `academic_level` | VARCHAR(100) | 🟡 Open Decision (افتراض: نص حر) | | |
| `school_name` | VARCHAR(255) | ❌ | | |
| `profile_photo_url` | VARCHAR(500) | ❌ | | |
| `status` | ENUM(`pending`,`approved`,`rejected`) | ✅ | | 3 حالات منفصلة (محسوم) |
| `student_code` | VARCHAR(20), UNIQUE | ✅ (يُولَّد تلقائيًا عند الموافقة/الإضافة المباشرة) | | |
| `qr_code_value` | VARCHAR(255), UNIQUE | ✅ (يُولَّد مع `student_code`) | | |
| `is_archived` | BOOLEAN | ✅ | | افتراضي `false`؛ الأرشفة = تعطيل قابل للاسترجاع (استنتاج من 4.18، **يحتاج تأكيد أحمد**) |
| `archived_at` | TIMESTAMP | ❌ | | |
| `created_at` | TIMESTAMP | ✅ | | |
| `updated_at` | TIMESTAMP | ✅ | | |

> **قيد تكامل غير قابل للتفاوض:** تعطيل/أرشفة الطالب **لا يعمل** `CASCADE DELETE` على أي جدول مرتبط (حضور، درجات، مدفوعات، امتحانات، واجبات).

#### 2.2.4 `otp_codes`
| الحقل | النوع | مطلوب | PK/FK | ملاحظات |
|---|---|---|---|---|
| `id` | UUID | ✅ | PK | |
| `account_id` | UUID | ✅ | FK → `accounts.id` | |
| `code_hash` | VARCHAR(255) | ✅ | | لا يُخزَّن الكود بالنص الصريح |
| `channel` | ENUM(`sms`,`whatsapp`,`email`) | 🟡 Open Decision | | قناة الإرسال غير محسومة في المصدر |
| `purpose` | ENUM(`login`) | ✅ | | إجباري لكل تسجيل دخول (محسوم) |
| `expires_at` | TIMESTAMP | ✅ | | مدة الصلاحية 🟡 Open Decision |
| `attempt_count` | INTEGER | ✅ | | افتراضي 0؛ حد المحاولات 🟡 Open Decision |
| `consumed_at` | TIMESTAMP | ❌ | | |
| `created_at` | TIMESTAMP | ✅ | | |

#### 2.2.5 `subjects`
| الحقل | النوع | مطلوب | PK/FK | ملاحظات |
|---|---|---|---|---|
| `id` | UUID | ✅ | PK | |
| `teacher_id` | UUID | ✅ | FK → `teachers.id` | |
| `name` | VARCHAR(255) | ✅ | | تكرار الاسم لنفس المدرّس 🟡 Open Decision (افتراض: مسموح) |
| `default_price` | INTEGER (قروش) | ❌ | | اختياري |
| `created_at` | TIMESTAMP | ✅ | | |
| `updated_at` | TIMESTAMP | ✅ | | |

#### 2.2.6 `groups`
| الحقل | النوع | مطلوب | PK/FK | ملاحظات |
|---|---|---|---|---|
| `id` | UUID | ✅ | PK | |
| `subject_id` | UUID | ✅ | FK → `subjects.id` | |
| `teacher_id` | UUID | ✅ | FK → `teachers.id` | مُكرَّر لتسريع الفلترة/الصلاحيات |
| `name` | VARCHAR(255) | 🟡 Open Decision (افتراض: إجباري) | | |
| `academic_level` | VARCHAR(100) | 🟡 Open Decision | | |
| `price` | INTEGER (قروش) | ❌ | | يُستخدم حسب أولوية التسعير (🟡 Open Decision كبير — راجع 4.3) |
| `is_archived` | BOOLEAN | ✅ | | افتراضي `false` |
| `created_at` | TIMESTAMP | ✅ | | |
| `updated_at` | TIMESTAMP | ✅ | | |

#### 2.2.7 `student_group_enrollments`
كيان الربط Many-to-Many بين الطالب والمجموعة — **يحافظ على التاريخ عند النقل** (سجل قديم يُقفل بـ `ended_at`، سجل جديد يُنشأ).

| الحقل | النوع | مطلوب | PK/FK | ملاحظات |
|---|---|---|---|---|
| `id` | UUID | ✅ | PK | |
| `student_id` | UUID | ✅ | FK → `students.id` | |
| `group_id` | UUID | ✅ | FK → `groups.id` | |
| `started_at` | TIMESTAMP | ✅ | | |
| `ended_at` | TIMESTAMP | ❌ | | `NULL` = تسجيل نشط حاليًا |
| `created_at` | TIMESTAMP | ✅ | | |

#### 2.2.8 `recurring_schedule_rules`
| الحقل | النوع | مطلوب | PK/FK | ملاحظات |
|---|---|---|---|---|
| `id` | UUID | ✅ | PK | |
| `group_id` | UUID | ✅ | FK → `groups.id` | |
| `day_of_week` | INTEGER (0-6) | ✅ | | |
| `start_time` | TIME | ✅ | | |
| `duration_minutes` | INTEGER | ✅ | | حد أدنى/أقصى 🟡 Open Decision |
| `recurrence_end_date` | DATE | 🟡 Open Decision (افتراض: NULL = بلا نهاية) | | |
| `is_active` | BOOLEAN | ✅ | | |
| `created_at` | TIMESTAMP | ✅ | | |

#### 2.2.9 `sessions`
حصة فعلية واحدة — سواء مولّدة من `recurring_schedule_rules` أو فردية. يمتد منها الحصص الأونلاين (4.21).

| الحقل | النوع | مطلوب | PK/FK | ملاحظات |
|---|---|---|---|---|
| `id` | UUID | ✅ | PK | |
| `group_id` | UUID | ❌ | FK → `groups.id` | `NULL` لو حصة فردية 1:1 بدون مجموعة |
| `student_id` | UUID | ❌ | FK → `students.id` | يُستخدم فقط للحصص الفردية 1:1 |
| `recurring_rule_id` | UUID | ❌ | FK → `recurring_schedule_rules.id` | `NULL` لو حصة فردية مستقلة |
| `teacher_id` | UUID | ✅ | FK → `teachers.id` | |
| `scheduled_at` | TIMESTAMP | ✅ | | |
| `duration_minutes` | INTEGER | ✅ | | |
| `status` | ENUM(`scheduled`,`rescheduled`,`cancelled`,`completed`,`conflicted`) | ✅ | | `conflicted` = تعارض تم تجاوزه بواسطة المدرّس |
| `delivery_mode` | ENUM(`in_person`,`online`) | ✅ | | |
| `online_platform` | ENUM(`google_meet`,`zoom`) | ❌ | | إجباري لو `delivery_mode = online`، يُختار لكل حصة |
| `external_meeting_id` | VARCHAR(255) | ❌ | | معرّف الاجتماع عند Meet/Zoom |
| `join_url` | VARCHAR(500) | ❌ | | |
| `recording_url` | VARCHAR(500) | ❌ | | 🟡 Open Decision (مكان التخزين مؤجل للفريق التقني) |
| `created_at` | TIMESTAMP | ✅ | | |
| `updated_at` | TIMESTAMP | ✅ | | |

#### 2.2.10 `attendance_records`
| الحقل | النوع | مطلوب | PK/FK | ملاحظات |
|---|---|---|---|---|
| `id` | UUID | ✅ | PK | |
| `session_id` | UUID | ✅ | FK → `sessions.id` | |
| `student_id` | UUID | ✅ | FK → `students.id` | |
| `status` | ENUM(`present`,`absent`,`late`,`excused`) | ✅ | | |
| `recorded_via` | ENUM(`manual`,`qr_scan`) | ✅ | | |
| `created_at` | TIMESTAMP | ✅ | | |
| `updated_at` | TIMESTAMP | ✅ | | يتغيّر مع كل تعديل، مع تسجيل موازٍ في سجل التدقيق |

> قيد فريد: `UNIQUE(session_id, student_id)`.

#### 2.2.11 `attendance_audit_trail_entries`
كيان جديد بالكامل، ناتج مباشر عن قرار أحمد — نطاق مؤكد.

| الحقل | النوع | مطلوب | PK/FK | ملاحظات |
|---|---|---|---|---|
| `id` | UUID | ✅ | PK | |
| `attendance_record_id` | UUID | ✅ | FK → `attendance_records.id` | |
| `changed_by_teacher_id` | UUID | ✅ | FK → `teachers.id` | |
| `old_status` | ENUM(`present`,`absent`,`late`,`excused`) | ❌ | | `NULL` لأول إنشاء |
| `new_status` | ENUM(`present`,`absent`,`late`,`excused`) | ✅ | | |
| `note` | TEXT | 🟡 Open Decision (افتراض: اختياري) | | |
| `created_at` | TIMESTAMP | ✅ | | |

#### 2.2.12 `homeworks`
| الحقل | النوع | مطلوب | PK/FK | ملاحظات |
|---|---|---|---|---|
| `id` | UUID | ✅ | PK | |
| `teacher_id` | UUID | ✅ | FK → `teachers.id` | |
| `target_type` | ENUM(`group`,`student`) | ✅ | | استهداف حصري — مجموعة أو طالب، مش الاتنين |
| `group_id` | UUID | ❌ | FK → `groups.id` | مطلوب لو `target_type=group` |
| `student_id` | UUID | ❌ | FK → `students.id` | مطلوب لو `target_type=student` |
| `title` | VARCHAR(255) | ✅ | | |
| `instructions` | TEXT | ✅ | | حد أقصى 🟡 Open Decision |
| `attachment_urls` | JSON (array of string) | ❌ | | أنواع/أحجام مسموحة 🟡 Open Decision |
| `due_at` | TIMESTAMP | ✅ | | |
| `created_at` | TIMESTAMP | ✅ | | |

#### 2.2.13 `homework_submissions`
| الحقل | النوع | مطلوب | PK/FK | ملاحظات |
|---|---|---|---|---|
| `id` | UUID | ✅ | PK | |
| `homework_id` | UUID | ✅ | FK → `homeworks.id` | |
| `student_id` | UUID | ✅ | FK → `students.id` | |
| `submission_type` | ENUM(`file`,`text`) | ✅ | | |
| `text_content` | TEXT | ❌ | | مطلوب لو `submission_type=text` |
| `attachment_url` | VARCHAR(500) | ❌ | | مطلوب لو `submission_type=file` |
| `is_late` | BOOLEAN | ✅ | | يُحسب مقارنةً بـ `due_at` |
| `submitted_at` | TIMESTAMP | ✅ | | |
| `updated_at` | TIMESTAMP | ✅ | | تسليمات متعددة: 🟡 Open Decision (افتراض: تستبدل نفس السجل) |

> قيد فريد: `UNIQUE(homework_id, student_id)` (بناءً على افتراض "تستبدل" أعلاه).

#### 2.2.14 `exams`
| الحقل | النوع | مطلوب | PK/FK | ملاحظات |
|---|---|---|---|---|
| `id` | UUID | ✅ | PK | |
| `teacher_id` | UUID | ✅ | FK → `teachers.id` | |
| `subject_id` | UUID | ❌ | FK → `subjects.id` | مادة و/أو مجموعة (🟡 Open Decision لمعنى الجمهور عند الاتنين معًا) |
| `group_id` | UUID | ❌ | FK → `groups.id` | |
| `title` | VARCHAR(255) | 🟡 Open Decision (افتراض: إجباري) | | |
| `source` | ENUM(`built_in`,`imported`) | ✅ | | |
| `auto_grading_enabled` | BOOLEAN | ✅ | | متاح فقط للأنواع المدعومة (اختيار من متعدد) |
| `time_limit_minutes` | INTEGER | 🟡 Open Decision | | نافذة زمنية غير محسومة |
| `created_at` | TIMESTAMP | ✅ | | |

#### 2.2.15 `exam_questions`
| الحقل | النوع | مطلوب | PK/FK | ملاحظات |
|---|---|---|---|---|
| `id` | UUID | ✅ | PK | |
| `exam_id` | UUID | ✅ | FK → `exams.id` | |
| `question_type` | ENUM(`multiple_choice`,`text`) | 🟡 Open Decision | | أنواع الأسئلة غير محسومة كاملة؛ MCQ فقط مؤكد كقابل للتصحيح التلقائي |
| `question_text` | TEXT | ✅ | | يدعم RTL |
| `options` | JSON (array) | ❌ | | لأسئلة الاختيار من متعدد فقط |
| `correct_option_index` | INTEGER | ❌ | | للتصحيح التلقائي |
| `points` | INTEGER | ✅ | | |
| `order_index` | INTEGER | ✅ | | |

#### 2.2.16 `exam_submissions`
| الحقل | النوع | مطلوب | PK/FK | ملاحظات |
|---|---|---|---|---|
| `id` | UUID | ✅ | PK | |
| `exam_id` | UUID | ✅ | FK → `exams.id` | |
| `student_id` | UUID | ✅ | FK → `students.id` | |
| `answers` | JSON | ✅ | | `[{question_id, answer}]` |
| `score` | DECIMAL(6,2) | ❌ | | يُملأ تلقائيًا أو يدويًا |
| `grading_status` | ENUM(`pending`,`auto_graded`,`manually_graded`) | ✅ | | |
| `submitted_at` | TIMESTAMP | ✅ | | |

> قيد فريد: `UNIQUE(exam_id, student_id)`.

#### 2.2.17 `grade_entries`
سجل درجات مستقل — يغطي درجات الامتحانات والواجبات والتقييمات المخصصة.

| الحقل | النوع | مطلوب | PK/FK | ملاحظات |
|---|---|---|---|---|
| `id` | UUID | ✅ | PK | |
| `teacher_id` | UUID | ✅ | FK → `teachers.id` | |
| `student_id` | UUID | ✅ | FK → `students.id` | |
| `exam_submission_id` | UUID | ❌ | FK → `exam_submissions.id` | `NULL` لو درجة مستقلة |
| `category` | ENUM(`exam`,`homework`,`participation`,`custom`) | 🟡 Open Decision (افتراض: enum ثابت) | | |
| `value` | DECIMAL(6,2) | ✅ | | صيغة الدرجة (رقم/حرف/نسبة) 🟡 Open Decision — تم اختيار رقمي |
| `max_value` | DECIMAL(6,2) | ✅ | | |
| `note` | TEXT | ❌ | | |
| `created_at` | TIMESTAMP | ✅ | | |

> **قرار مفتوح مهم يؤثر هنا:** هل تصحيح امتحان/واجب يُنشئ `grade_entry` أوتوماتيك أم بخطوة يدوية — التصميم أعلاه يدعم الحالتين (FK اختياري لـ `exam_submission_id`، وإدخال مباشر لأي حالة).

#### 2.2.18 `learning_materials`
| الحقل | النوع | مطلوب | PK/FK | ملاحظات |
|---|---|---|---|---|
| `id` | UUID | ✅ | PK | |
| `teacher_id` | UUID | ✅ | FK → `teachers.id` | |
| `target_type` | ENUM(`subject`,`group`,`student`) | ✅ | | نمط الاستهداف الثلاثي |
| `subject_id` / `group_id` / `student_id` | UUID | ❌ (واحد منهم فقط حسب `target_type`) | FK متعددة | |
| `material_type` | ENUM(`pdf`,`video`,`image`,`link`,`other`) | ✅ | | |
| `title` | VARCHAR(255) | ✅ | | |
| `file_url` | VARCHAR(500) | ❌ | | مطلوب لو مش `link` |
| `external_link` | VARCHAR(500) | ❌ | | مطلوب لو `link` |
| `access_mode` | ENUM(`view_only`,`downloadable`) | ✅ | | |
| `pricing_mode` | ENUM(`free`,`paid`) | ✅ | | محسوم: لكل عنصر على حدة |
| `price` | INTEGER (قروش) | ❌ | | مطلوب لو `pricing_mode=paid` |
| `file_size_bytes` | BIGINT | ❌ | | مرتبط بحد التخزين (🟡 Open Decision) |
| `created_at` | TIMESTAMP | ✅ | | |

#### 2.2.19 `payments`
**بعد إلغاء المحفظة: كل الدفعات مباشرة، بدون رصيد وسيط.**

| الحقل | النوع | مطلوب | PK/FK | ملاحظات |
|---|---|---|---|---|
| `id` | UUID | ✅ | PK | |
| `teacher_id` | UUID | ✅ | FK → `teachers.id` | مُكرَّر لتسريع تجميع الأرباح |
| `student_id` | UUID | ✅ | FK → `students.id` | |
| `payable_type` | ENUM(`session`,`subject`,`group`,`learning_material`,`manual`) | ✅ | | `manual` = دفعة كاش يسجلها المدرّس بدون ربط بعنصر محدد |
| `payable_id` | UUID | ❌ | FK متعددة الأشكال | `NULL` لو `payable_type=manual` |
| `method` | ENUM(`cash`,`card`,`fawry`,`instapay`) | ✅ | | حسب القرار #8 (بدون محفظة إلكترونية بعد الإلغاء) |
| `amount` | INTEGER (قروش) | ✅ | | |
| `net_amount` | INTEGER (قروش) | ✅ | | بعد خصم رسوم البوابة (إن وجدت)؛ يُستخدم لحساب الأرباح القابلة للسحب |
| `status` | ENUM(`pending`,`completed`,`failed`) | ✅ | | |
| `gateway_reference` | VARCHAR(255) | ❌ | | مرجع بوابة الدفع (المزوّد لسه 🟡 Open Decision) |
| `recorded_by_teacher_id` | UUID | ❌ | FK → `teachers.id` | مُعبَّأ فقط لو `method=cash` |
| `is_withdrawn` | BOOLEAN | ✅ | | يتحول `true` عند تنفيذ طلب سحب يغطي هذه الدفعة |
| `created_at` | TIMESTAMP | ✅ | | |

#### 2.2.20 `withdrawals`
| الحقل | النوع | مطلوب | PK/FK | ملاحظات |
|---|---|---|---|---|
| `id` | UUID | ✅ | PK | |
| `teacher_id` | UUID | ✅ | FK → `teachers.id` | |
| `amount` | INTEGER (قروش) | ✅ | | يُتحقق أنه ≤ مجموع `payments.net_amount` غير المسحوبة |
| `destination_method` | ENUM(`bank_card`,`instapay`,`e_wallet`) | ✅ | | |
| `destination_reference` | VARCHAR(255) | ✅ | | رقم كارت/محفظة الوجهة |
| `status` | ENUM(`requested`,`processing`,`completed`,`rejected`) | ✅ | | مدة المعالجة/الرسوم 🟡 Open Decision |
| `requested_at` | TIMESTAMP | ✅ | | |
| `processed_at` | TIMESTAMP | ❌ | | |

#### 2.2.21 `notifications`
| الحقل | النوع | مطلوب | PK/FK | ملاحظات |
|---|---|---|---|---|
| `id` | UUID | ✅ | PK | |
| `account_id` | UUID | ✅ | FK → `accounts.id` | |
| `event_type` | ENUM(`attendance`,`upcoming_session`,`payment_reminder`,`exam_announcement`,`grade_update`,`homework_assigned`,`new_message`,`registration_approved`,`registration_rejected`) | ✅ | | |
| `related_entity_type` | VARCHAR(50) | ❌ | | |
| `related_entity_id` | UUID | ❌ | | |
| `title` | VARCHAR(255) | ✅ | | |
| `body` | TEXT | ✅ | | |
| `is_read` | BOOLEAN | ✅ | | افتراضي `false` |
| `created_at` | TIMESTAMP | ✅ | | سياسة الاحتفاظ 🟡 Open Decision |

#### 2.2.22 `whatsapp_messages`
| الحقل | النوع | مطلوب | PK/FK | ملاحظات |
|---|---|---|---|---|
| `id` | UUID | ✅ | PK | |
| `teacher_id` | UUID | ✅ | FK → `teachers.id` | |
| `student_id` | UUID | ✅ | FK → `students.id` | لتحديد `guardian_phone` وقت الإرسال |
| `guardian_phone_snapshot` | VARCHAR(20) | ✅ | | نسخة من الرقم وقت الإرسال (تتبّع تاريخي) |
| `trigger_type` | ENUM(`automatic_qr_attendance`,`manual`) | ✅ | | |
| `message_kind` | ENUM(`attendance`,`payment_reminder`,`grade_update`,`homework_assigned`,`exam_announcement`,`other`) | ✅ | | |
| `content_source` | ENUM(`template`,`free_text`) | ✅ | | الاتنين متاحين، اختيار المدرّس |
| `content` | TEXT | ✅ | | |
| `delivery_status` | ENUM(`queued`,`sent`,`delivered`,`failed`) | ✅ | | مزوّد واتساب (Meta الرسمي/BSP) 🟡 Open Decision |
| `created_at` | TIMESTAMP | ✅ | | |

#### 2.2.23 `messages`
رسائل 1:1 بين المدرّس والطالب فقط.

| الحقل | النوع | مطلوب | PK/FK | ملاحظات |
|---|---|---|---|---|
| `id` | UUID | ✅ | PK | |
| `teacher_id` | UUID | ✅ | FK → `teachers.id` | |
| `student_id` | UUID | ✅ | FK → `students.id` | يحدد المحادثة الثنائية |
| `sender_role` | ENUM(`teacher`,`student`) | ✅ | | |
| `content` | TEXT | ✅ | | حد الطول/المرفقات 🟡 Open Decision |
| `is_read` | BOOLEAN | ✅ | | |
| `created_at` | TIMESTAMP | ✅ | | |

#### 2.2.24 `user_settings`
| الحقل | النوع | مطلوب | PK/FK | ملاحظات |
|---|---|---|---|---|
| `account_id` | UUID | ✅ | PK, FK → `accounts.id` | علاقة 1:1 |
| `notification_prefs` | JSON | ✅ | | دقة التحكم لكل نوع حدث 🟡 Open Decision |
| `withdrawal_destination_default` | VARCHAR(255) | ❌ | | للمدرّس فقط |
| `saved_payment_methods` | JSON | ❌ | | للطالب فقط (بدون بيانات كارت حساسة — Tokenized فقط) |
| `updated_at` | TIMESTAMP | ✅ | | |

---

## 3. توثيق REST API

> لكل ميزة: جدول الـ Endpoints، ثم أمثلة Request/Response JSON صريحة وكاملة للعمليات الأساسية.
> **الرمز `{id}` يعني UUID.** أي endpoint غير مذكور صراحةً كـ "عام (Public)" يتطلب `Authorization: Bearer`.

---

### 4.1 المصادقة (Auth)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| POST | `/auth/register` | تسجيل حساب جديد (مدرّس أو طالب ذاتي) | عام |
| POST | `/auth/login` | خطوة أولى: هوية + باسورد | عام |
| POST | `/auth/otp/verify` | خطوة ثانية: تأكيد OTP وإصدار التوكن | عام (يتطلب `login_challenge_token`) |
| POST | `/auth/otp/resend` | إعادة إرسال OTP | عام (يتطلب `login_challenge_token`) |
| POST | `/auth/logout` | إنهاء الجلسة الحالية | مدرّس/طالب |
| GET | `/students/pending` | قائمة طلبات التسجيل المعلّقة | مدرّس |
| PUT | `/students/{id}/approval` | موافقة/رفض طالب معلّق | مدرّس (لطلابه فقط) |
| POST | `/students` | إضافة طالب فردي (موافق تلقائيًا) | مدرّس |
| POST | `/students/bulk-import` | استيراد طلاب جماعي (Excel/Sheets) | مدرّس |

**POST `/auth/register`**
Request:
```json
{
  "role": "student",
  "teacher_slug": "ahmed-math-2026",
  "phone": "+201012345678",
  "email": null,
  "password": "StrongPass123",
  "full_name": "سارة أحمد"
}
```
Response `201 Created`:
```json
{
  "data": {
    "id": "b3f1c2a0-1111-4a2b-9c3d-000000000001",
    "role": "student",
    "status": "pending",
    "message": "طلب التسجيل بتاعك تحت المراجعة وبينتظر موافقة المدرّس."
  }
}
```
Response `409 Conflict` (حساب مكرر):
```json
{
  "error": {
    "code": "ACCOUNT_ALREADY_EXISTS",
    "message": "فيه حساب بنفس الرقم أو الإيميل موجود بالفعل.",
    "details": []
  }
}
```

**POST `/auth/login`**
Request:
```json
{ "identifier": "+201012345678", "password": "StrongPass123" }
```
Response `200 OK`:
```json
{
  "data": {
    "login_challenge_token": "chg_9f8e7d6c5b4a",
    "otp_channel": "sms",
    "expires_in_seconds": 300
  }
}
```

**POST `/auth/otp/verify`**
Request:
```json
{ "login_challenge_token": "chg_9f8e7d6c5b4a", "otp_code": "482913" }
```
Response `200 OK`:
```json
{
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIs...",
    "refresh_token": "rt_3a2b1c0d9e8f",
    "role": "student",
    "account_id": "b3f1c2a0-1111-4a2b-9c3d-000000000001",
    "redirect_to": "/dashboard/student"
  }
}
```
Response `400 Bad Request` (كود خاطئ):
```json
{ "error": { "code": "INVALID_OTP", "message": "الكود غلط، حاول تاني", "details": [] } }
```

**PUT `/students/{id}/approval`**
Request:
```json
{ "decision": "approved" }
```
Response `200 OK`:
```json
{
  "data": {
    "id": "b3f1c2a0-1111-4a2b-9c3d-000000000001",
    "status": "approved",
    "student_code": "STU-2026-0421",
    "qr_code_value": "QR-STU-2026-0421-XZ91"
  }
}
```
> لو `"decision": "rejected"` يرجع نفس الشكل بـ `"status": "rejected"` بدون `student_code`/`qr_code_value`.

**Status Codes:** `201` تسجيل ناجح · `200` نجاح تسجيل دخول/موافقة · `400` بيانات خاطئة أو OTP خاطئ · `401` توكن مؤقت منتهي · `409` حساب مكرر · `422` طالب مش في حالة معلّق أثناء محاولة الموافقة/الرفض.

---

### 4.2 لوحة التحكم (Dashboard)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/dashboard/teacher` | ملخص لوحة المدرّس | مدرّس |
| GET | `/dashboard/student` | ملخص لوحة الطالب | طالب |

**GET `/dashboard/teacher`**
Response `200 OK`:
```json
{
  "data": {
    "total_students": 84,
    "todays_sessions": [
      { "id": "s1", "group_name": "الرياضيات - الصف الثالث", "scheduled_at": "2026-07-21T15:00:00Z" }
    ],
    "total_payments_this_month": 1250000,
    "total_earnings": 5400000,
    "overdue_payments_count": 6,
    "attendance_summary": { "present_rate": 0.91, "absent_rate": 0.06, "late_rate": 0.03 },
    "overdue_students": [
      { "student_id": "st1", "full_name": "محمد علي", "overdue_amount": 30000 }
    ],
    "total_groups": 5,
    "ongoing_exams_count": 2,
    "exam_questions_sent_count": 18,
    "pending_approvals_count": 3
  }
}
```

**GET `/dashboard/student`**
Response `200 OK`:
```json
{
  "data": {
    "upcoming_sessions": [
      { "id": "s1", "subject_name": "الرياضيات", "scheduled_at": "2026-07-22T14:00:00Z" }
    ],
    "group": { "id": "g1", "name": "الرياضيات - الصف الثالث" },
    "homeworks": [ { "id": "h1", "title": "واجب الوحدة 3", "due_at": "2026-07-25T00:00:00Z", "submitted": false } ],
    "exams": [ { "id": "e1", "title": "امتحان نصف الفصل", "status": "upcoming" } ],
    "grades": [ { "id": "gr1", "category": "exam", "value": 27, "max_value": 30 } ],
    "recent_notifications": [ { "id": "n1", "title": "تم تسجيل حضورك", "created_at": "2026-07-21T10:00:00Z" } ]
  }
}
```
**Status Codes:** `200` نجاح · `401` غير مصرَّح.

---

### 4.3 المواد الدراسية (Subjects)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/subjects` | قائمة مواد المدرّس (أو مواد الطالب المسجّل فيها) | مدرّس (كامل) / طالب (قراءة فقط) |
| GET | `/subjects/{id}` | تفاصيل مادة | مدرّس / طالب (لو مسجّل فيها) |
| POST | `/subjects` | إنشاء مادة | مدرّس |
| PUT | `/subjects/{id}` | تعديل مادة | مدرّس |
| DELETE | `/subjects/{id}` | حذف مادة | مدرّس |

**POST `/subjects`**
Request:
```json
{ "name": "الرياضيات", "default_price": 15000 }
```
Response `201 Created`:
```json
{
  "data": {
    "id": "sub_001",
    "teacher_id": "t_001",
    "name": "الرياضيات",
    "default_price": 15000,
    "groups_count": 0,
    "created_at": "2026-07-21T10:00:00Z"
  }
}
```

**GET `/subjects`**
Response `200 OK`:
```json
{
  "data": [
    { "id": "sub_001", "name": "الرياضيات", "default_price": 15000, "groups_count": 3 }
  ],
  "meta": { "page": 1, "limit": 20, "total_items": 4, "total_pages": 1 }
}
```

**DELETE `/subjects/{id}`**
Response `204 No Content` (بدون محتوى).
Response `409 Conflict` (لو فيها مجموعات نشطة — سلوك الحذف 🟡 Open Decision، الافتراض هنا "منع الحذف"):
```json
{ "error": { "code": "SUBJECT_HAS_ACTIVE_GROUPS", "message": "لا يمكن حذف مادة مرتبطة بمجموعات نشطة", "details": [] } }
```
**Status Codes:** `200/201/204` نجاح · `400` اسم ناقص/سعر سالب · `403` مادة تحت مدرّس تاني · `404` غير موجودة · `409` حذف مادة مرتبطة بمجموعات.

---

### 4.4 الطلاب (Students)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/students` | قائمة طلاب المدرّس (نشطين، بدعم فلترة `status`) | مدرّس |
| GET | `/students/{id}` | بروفايل طالب | مدرّس / الطالب نفسه |
| PUT | `/students/{id}` | تعديل بروفايل | مدرّس (كامل) / طالب (نطاق محدود 🟡 Open Decision) |
| POST | `/students/{id}/enrollments` | تسجيل الطالب في مادة/مجموعة إضافية | مدرّس |
| POST | `/students/{id}/archive` | تعطيل/أرشفة طالب | مدرّس |
| POST | `/students/{id}/restore` | استرجاع طالب من الأرشيف | مدرّس |

*(انظر أيضًا `/students`, `/students/{id}/approval`, `/students/bulk-import` في قسم 4.1.)*

**GET `/students/{id}`**
Response `200 OK`:
```json
{
  "data": {
    "id": "st_001",
    "teacher_id": "t_001",
    "full_name": "محمد علي",
    "phone": "+201098765432",
    "email": null,
    "guardian_phone": "+201155667788",
    "academic_level": "الصف الثالث الإعدادي",
    "school_name": "مدرسة النصر",
    "status": "approved",
    "student_code": "STU-2026-0421",
    "qr_code_value": "QR-STU-2026-0421-XZ91",
    "is_archived": false,
    "groups": [ { "id": "g1", "name": "الرياضيات - الصف الثالث" } ],
    "created_at": "2026-06-01T09:00:00Z"
  }
}
```

**PUT `/students/{id}`**
Request:
```json
{ "full_name": "محمد علي أحمد", "guardian_phone": "+201155667788", "academic_level": "الصف الثالث الإعدادي" }
```
Response `200 OK`: نفس شكل GET أعلاه بالقيم المحدثة.

**POST `/students/{id}/archive`**
Response `200 OK`:
```json
{ "data": { "id": "st_001", "is_archived": true, "archived_at": "2026-07-21T12:00:00Z" } }
```
**Status Codes:** `200` نجاح · `400` حقل إجباري ناقص · `403` طالب تحت مدرّس تاني، أو طالب بيحاول يعدّل حقل مقصور على المدرّس · `404` غير موجود.

---

### 4.5 المجموعات (Groups)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/groups` | قائمة المجموعات (فلترة بـ `subject_id`) | مدرّس (كامل) / طالب (عضويته فقط) |
| GET | `/groups/{id}` | تفاصيل مجموعة | مدرّس / طالب عضو |
| POST | `/groups` | إنشاء مجموعة | مدرّس |
| PUT | `/groups/{id}` | تعديل مجموعة | مدرّس |
| DELETE | `/groups/{id}` | حذف/أرشفة مجموعة | مدرّس |
| POST | `/groups/{id}/enrollments` | تسجيل طالب في المجموعة | مدرّس |
| POST | `/groups/{id}/enrollments/{student_id}/transfer` | نقل طالب لمجموعة أخرى (يحافظ على التاريخ) | مدرّس |

**POST `/groups`**
Request:
```json
{ "subject_id": "sub_001", "name": "الرياضيات - الصف الثالث - سبت/اثنين", "academic_level": "الصف الثالث الإعدادي", "price": 15000 }
```
Response `201 Created`:
```json
{
  "data": {
    "id": "g_001",
    "subject_id": "sub_001",
    "name": "الرياضيات - الصف الثالث - سبت/اثنين",
    "academic_level": "الصف الثالث الإعدادي",
    "price": 15000,
    "students_count": 0,
    "is_archived": false,
    "created_at": "2026-07-21T10:00:00Z"
  }
}
```

**POST `/groups/{id}/enrollments`**
Request:
```json
{ "student_id": "st_001" }
```
Response `201 Created`:
```json
{
  "data": { "id": "enr_001", "student_id": "st_001", "group_id": "g_001", "started_at": "2026-07-21T10:05:00Z", "ended_at": null }
}
```

**POST `/groups/{id}/enrollments/{student_id}/transfer`**
Request:
```json
{ "to_group_id": "g_002" }
```
Response `200 OK`:
```json
{
  "data": {
    "closed_enrollment": { "id": "enr_001", "group_id": "g_001", "ended_at": "2026-07-21T11:00:00Z" },
    "new_enrollment": { "id": "enr_002", "group_id": "g_002", "started_at": "2026-07-21T11:00:00Z", "ended_at": null }
  }
}
```
**Status Codes:** `200/201` نجاح · `400` بيانات ناقصة · `403` مجموعة تحت مدرّس تاني · `404` مادة/طالب غير موجود · `409` حذف مجموعة فيها طلاب مسجَّلين (🟡 Open Decision — الافتراض هنا: منع).

---

### 4.6 الجدول (Schedule)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/schedule/sessions` | قائمة الحصص (فلترة بتاريخ/مجموعة) | مدرّس (الكل) / طالب (حصصه فقط) |
| GET | `/schedule/sessions/{id}` | تفاصيل حصة | مدرّس / طالب معني |
| POST | `/schedule/recurring-rules` | إنشاء جدول أسبوعي متكرر لمجموعة | مدرّس |
| POST | `/schedule/sessions` | إنشاء حصة فردية | مدرّس |
| PUT | `/schedule/sessions/{id}` | تعديل/تأجيل حصة | مدرّس |
| POST | `/schedule/sessions/{id}/cancel` | إلغاء حصة | مدرّس |
| POST | `/schedule/conflict-check` | فحص تعارض قبل الحفظ | مدرّس |

**POST `/schedule/recurring-rules`**
Request:
```json
{ "group_id": "g_001", "day_of_week": 6, "start_time": "15:00:00", "duration_minutes": 60, "recurrence_end_date": null }
```
Response `201 Created`:
```json
{
  "data": {
    "id": "rr_001", "group_id": "g_001", "day_of_week": 6,
    "start_time": "15:00:00", "duration_minutes": 60,
    "recurrence_end_date": null, "is_active": true,
    "generated_sessions_count": 12
  }
}
```

**POST `/schedule/conflict-check`**
Request:
```json
{ "scheduled_at": "2026-07-25T15:00:00Z", "duration_minutes": 60, "exclude_session_id": null }
```
Response `200 OK`:
```json
{
  "data": {
    "has_conflict": true,
    "conflicting_sessions": [
      { "id": "s_099", "group_name": "اللغة العربية", "scheduled_at": "2026-07-25T15:30:00Z" }
    ]
  }
}
```

**PUT `/schedule/sessions/{id}`**
Request:
```json
{ "scheduled_at": "2026-07-25T16:00:00Z", "apply_to": "this_session_only" }
```
> `apply_to`: `this_session_only` | `entire_series` — 🟡 Open Decision (المصدر لم يحسم هل التعديل يشمل السلسلة كلها).

Response `200 OK`:
```json
{ "data": { "id": "s_050", "status": "rescheduled", "scheduled_at": "2026-07-25T16:00:00Z" } }
```
**Status Codes:** `200/201` نجاح · `400` بيانات وقت خاطئة · `403` مجموعة تحت مدرّس تاني · `404` غير موجود · اكتشاف التعارض **استشاري وليس حظرًا** (لا يُرجع `409`)، الحفظ بيكمل بحالة `conflicted` لو المدرّس أكّد.

---

### 4.7 الحضور (Attendance)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/attendance/sessions/{session_id}` | جلب/بدء تسجيل حضور حصة | مدرّس |
| POST | `/attendance/sessions/{session_id}/submit` | إرسال حضور دفعة (Batch) | مدرّس |
| PUT | `/attendance/records/{id}` | تعديل سجل حضور (بعد الإرسال) | مدرّس |
| POST | `/attendance/qr-scan` | تسجيل حضور بمسح QR | مدرّس |
| GET | `/attendance/records/{id}/audit-trail` | جلب سجل تعديلات الحضور | مدرّس |
| GET | `/attendance/students/{student_id}` | سجل حضور طالب | مدرّس / الطالب نفسه |

**POST `/attendance/sessions/{session_id}/submit`**
Request:
```json
{
  "records": [
    { "student_id": "st_001", "status": "present" },
    { "student_id": "st_002", "status": "absent" },
    { "student_id": "st_003", "status": "late" }
  ]
}
```
Response `201 Created`:
```json
{
  "data": {
    "session_id": "s_050",
    "records_created": 3,
    "whatsapp_notifications_queued": 3
  }
}
```

**PUT `/attendance/records/{id}`**
Request:
```json
{ "status": "excused", "note": "عذر طبي مقدَّم من ولي الأمر" }
```
Response `200 OK`:
```json
{
  "data": {
    "id": "att_010",
    "status": "excused",
    "audit_entry": {
      "id": "audit_055",
      "old_status": "absent",
      "new_status": "excused",
      "changed_by_teacher_id": "t_001",
      "created_at": "2026-07-21T13:00:00Z"
    },
    "whatsapp_notification_sent": true
  }
}
```

**POST `/attendance/qr-scan`**
Request:
```json
{ "session_id": "s_050", "qr_code_value": "QR-STU-2026-0421-XZ91" }
```
Response `201 Created`:
```json
{ "data": { "id": "att_011", "student_id": "st_001", "status": "present", "recorded_via": "qr_scan" } }
```
Response `404 Not Found` (QR مش تابع لمجموعة الحصة):
```json
{ "error": { "code": "STUDENT_NOT_IN_SESSION_GROUP", "message": "الطالب غير مسجَّل في هذه المجموعة", "details": [] } }
```
**Status Codes:** `200/201` نجاح · `400` حالة غير صحيحة (لازم تكون من الأربعة) · `403` حصة تحت مدرّس تاني · `404` طالب غير مسجَّل في المجموعة/QR غير مطابق.

---

### 4.8 الواجبات (Homework)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/homeworks` | قائمة واجبات (المدرّس: كل واجباته، الطالب: واجباته المستهدف بيها) | مدرّس / طالب |
| GET | `/homeworks/{id}` | تفاصيل واجب | مدرّس / طالب معني |
| POST | `/homeworks` | إنشاء واجب | مدرّس |
| GET | `/homeworks/{id}/submissions` | قائمة تسليمات الواجب | مدرّس |
| POST | `/homeworks/{id}/submissions` | تسليم واجب | طالب |

**POST `/homeworks`**
Request:
```json
{
  "target_type": "group",
  "group_id": "g_001",
  "title": "واجب الوحدة 3",
  "instructions": "حل تمارين الصفحة 45 وارفع الحل كملف PDF",
  "attachment_urls": ["https://cdn.darsly.app/hw/unit3.pdf"],
  "due_at": "2026-07-28T23:59:00Z"
}
```
Response `201 Created`:
```json
{
  "data": {
    "id": "hw_001", "target_type": "group", "group_id": "g_001",
    "title": "واجب الوحدة 3", "due_at": "2026-07-28T23:59:00Z",
    "created_at": "2026-07-21T10:00:00Z"
  }
}
```

**POST `/homeworks/{id}/submissions`**
Request:
```json
{ "submission_type": "file", "attachment_url": "https://cdn.darsly.app/submissions/st001_hw001.pdf" }
```
Response `201 Created`:
```json
{
  "data": {
    "id": "sub_hw_001", "homework_id": "hw_001", "student_id": "st_001",
    "submission_type": "file", "is_late": false,
    "submitted_at": "2026-07-25T18:00:00Z"
  }
}
```
**Status Codes:** `200/201` نجاح · `400` استهداف غير صحيح (لازم مجموعة أو طالب مش الاتنين) أو محتوى تسليم ناقص · `403` واجب مش مسند للطالب · `404` غير موجود.

---

### 4.9 الامتحانات (Exams)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/exams` | قائمة امتحانات | مدرّس / طالب (المستهدف بيها) |
| GET | `/exams/{id}` | تفاصيل امتحان (بدون إجابات صحيحة للطالب) | مدرّس / طالب |
| POST | `/exams` | إنشاء امتحان (بناء داخل النظام) | مدرّس |
| POST | `/exams/import` | استيراد امتحان من ملف | مدرّس |
| POST | `/exams/{id}/questions` | إضافة سؤال | مدرّس |
| POST | `/exams/{id}/submissions` | تسليم إجابات الطالب | طالب |
| PUT | `/exams/{id}/submissions/{submission_id}/grade` | تصحيح يدوي | مدرّس |

**POST `/exams`**
Request:
```json
{ "subject_id": "sub_001", "group_id": "g_001", "title": "امتحان نصف الفصل", "auto_grading_enabled": true }
```
Response `201 Created`:
```json
{
  "data": {
    "id": "ex_001", "subject_id": "sub_001", "group_id": "g_001",
    "title": "امتحان نصف الفصل", "source": "built_in",
    "auto_grading_enabled": true, "questions_count": 0
  }
}
```

**POST `/exams/{id}/questions`**
Request:
```json
{
  "question_type": "multiple_choice",
  "question_text": "ناتج 5 × 6 =",
  "options": ["25", "30", "35", "40"],
  "correct_option_index": 1,
  "points": 5,
  "order_index": 1
}
```
Response `201 Created`:
```json
{ "data": { "id": "q_001", "exam_id": "ex_001", "question_type": "multiple_choice", "points": 5 } }
```

**POST `/exams/{id}/submissions`**
Request:
```json
{ "answers": [ { "question_id": "q_001", "answer": 1 } ] }
```
Response `201 Created`:
```json
{
  "data": {
    "id": "esub_001", "exam_id": "ex_001", "student_id": "st_001",
    "grading_status": "auto_graded", "score": 5,
    "submitted_at": "2026-07-30T10:00:00Z"
  }
}
```
**Status Codes:** `200/201` نجاح · `400` إجابات ناقصة/سؤال غير صحيح · `403` طالب برّه نطاق الامتحان · `404` غير موجود · `409` تسليم مكرر لنفس الامتحان.

---

### 4.10 الدرجات (Grades)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/grades` | سجل درجات (فلترة `student_id`/`group_id`/`subject_id`) | مدرّس (الكل) / طالب (درجاته فقط) |
| POST | `/grades` | إضافة درجة (مرتبطة بامتحان أو مستقلة) | مدرّس |
| PUT | `/grades/{id}` | تعديل درجة | مدرّس |
| DELETE | `/grades/{id}` | حذف درجة | مدرّس |

**POST `/grades`**
Request:
```json
{
  "student_id": "st_001",
  "exam_submission_id": null,
  "category": "participation",
  "value": 9,
  "max_value": 10,
  "note": "مشاركة فعّالة في الحصة"
}
```
Response `201 Created`:
```json
{
  "data": {
    "id": "grd_001", "student_id": "st_001", "category": "participation",
    "value": 9, "max_value": 10, "created_at": "2026-07-21T14:00:00Z"
  }
}
```

**GET `/grades?student_id=st_001`**
Response `200 OK`:
```json
{
  "data": [
    { "id": "grd_001", "category": "participation", "value": 9, "max_value": 10 },
    { "id": "grd_002", "category": "exam", "value": 27, "max_value": 30, "exam_submission_id": "esub_001" }
  ],
  "meta": { "page": 1, "limit": 20, "total_items": 2, "total_pages": 1 }
}
```
**Status Codes:** `200/201` نجاح · `400` قيمة درجة غير صحيحة · `403` طالب برّه نطاق المدرّس · `404` غير موجود.

---

### 4.11 المدفوعات (Payments)

> ⚠️ **بدون محفظة إلكترونية (تم إلغاؤها).** الطالب يدفع مباشرة؛ أرباح المدرّس تُحسب من `payments` مباشرة عند الطلب.

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/payments` | قائمة دفعات (فلترة `student_id`/`status`/تاريخ) | مدرّس |
| GET | `/payments/{id}` | تفاصيل دفعة | مدرّس / الطالب المعني |
| POST | `/payments/cash` | تسجيل دفعة كاش يدويًا | مدرّس |
| POST | `/payments/checkout` | بدء دفعة رقمية (كارت/فوري/إنستاباي) | طالب |
| POST | `/payments/{id}/gateway-callback` | Webhook تأكيد الدفع من بوابة الدفع | عام (موقّع من البوابة) |
| GET | `/payments/overdue` | قائمة الطلاب المتأخرين في الدفع | مدرّس |
| GET | `/payments/earnings-summary` | ملخص الأرباح القابلة للسحب | مدرّس |
| POST | `/withdrawals` | طلب سحب أرباح | مدرّس |
| GET | `/withdrawals` | قائمة طلبات السحب | مدرّس |

**POST `/payments/cash`**
Request:
```json
{ "student_id": "st_001", "payable_type": "group", "payable_id": "g_001", "amount": 15000 }
```
Response `201 Created`:
```json
{
  "data": {
    "id": "pay_001", "student_id": "st_001", "method": "cash",
    "amount": 15000, "net_amount": 15000, "status": "completed",
    "recorded_by_teacher_id": "t_001", "created_at": "2026-07-21T10:00:00Z"
  }
}
```

**POST `/payments/checkout`**
Request:
```json
{ "payable_type": "learning_material", "payable_id": "lm_007", "method": "card" }
```
Response `201 Created`:
```json
{
  "data": {
    "id": "pay_002", "status": "pending", "method": "card",
    "amount": 5000, "checkout_redirect_url": "https://pay.gateway.example/checkout/pay_002"
  }
}
```
> **ملاحظة:** بوابة الدفع التقنية المحددة (Paymob/Fawaterak/Kashier) 🟡 Open Decision — الشكل أعلاه عام (Gateway-Agnostic) ويُحدَّث وقت اختيار المزوّد.

**GET `/payments/earnings-summary`**
Response `200 OK`:
```json
{
  "data": {
    "total_earned": 5400000,
    "withdrawable_amount": 3200000,
    "already_withdrawn": 2200000,
    "pending_withdrawal_requests": 1
  }
}
```

**POST `/withdrawals`**
Request:
```json
{ "amount": 1000000, "destination_method": "bank_card", "destination_reference": "•••• 4521" }
```
Response `201 Created`:
```json
{ "data": { "id": "wd_001", "amount": 1000000, "status": "requested", "requested_at": "2026-07-21T15:00:00Z" } }
```
Response `422 Unprocessable Entity` (تجاوز الرصيد المتاح):
```json
{ "error": { "code": "INSUFFICIENT_WITHDRAWABLE_BALANCE", "message": "المبلغ المطلوب أكبر من الرصيد القابل للسحب", "details": [] } }
```
**Status Codes:** `200/201` نجاح · `400` مبلغ سالب/طريقة غير مدعومة · `403` دفعة/طالب تحت مدرّس تاني · `404` غير موجود · `422` سحب يتجاوز الرصيد.

---

### 4.12 المواد التعليمية (Learning Materials)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/learning-materials` | قائمة مواد تعليمية (فلترة `subject_id`/`group_id`) | مدرّس (الكل) / طالب (المتاح له) |
| GET | `/learning-materials/{id}` | تفاصيل عنصر | مدرّس / طالب (حسب صلاحيته والدفع) |
| POST | `/learning-materials` | رفع/إضافة عنصر | مدرّس |
| PUT | `/learning-materials/{id}` | تعديل عنصر (الوصول/التسعير) | مدرّس |
| DELETE | `/learning-materials/{id}` | حذف عنصر | مدرّس |
| GET | `/learning-materials/{id}/download` | تحميل (لو `downloadable` ومدفوع مسبقًا) | طالب مؤهَّل |

**POST `/learning-materials`**
Request:
```json
{
  "target_type": "group",
  "group_id": "g_001",
  "material_type": "pdf",
  "title": "ملخص الوحدة الثالثة",
  "file_url": "https://cdn.darsly.app/materials/unit3-summary.pdf",
  "access_mode": "downloadable",
  "pricing_mode": "paid",
  "price": 5000
}
```
Response `201 Created`:
```json
{
  "data": {
    "id": "lm_007", "target_type": "group", "group_id": "g_001",
    "material_type": "pdf", "title": "ملخص الوحدة الثالثة",
    "access_mode": "downloadable", "pricing_mode": "paid", "price": 5000,
    "created_at": "2026-07-21T09:00:00Z"
  }
}
```

**GET `/learning-materials/{id}/download`**
Response `200 OK`:
```json
{ "data": { "download_url": "https://cdn.darsly.app/materials/unit3-summary.pdf?token=temp_xyz&exp=1753123456" } }
```
Response `402/403` (لو العنصر مدفوع وميتدفعش):
```json
{ "error": { "code": "PAYMENT_REQUIRED", "message": "يجب دفع ثمن هذا العنصر قبل التحميل", "details": [] } }
```
**Status Codes:** `200/201` نجاح · `400` نوع/حجم ملف غير مدعوم · `403` عنصر برّه نطاق الطالب أو محاولة تحميل عنصر "عرض فقط" برابط مباشر · `404` غير موجود.

---

### 4.13 التقارير (Reports)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/reports/financial` | تقرير مالي/أرباح | مدرّس فقط |
| GET | `/reports/payments` | تقرير مدفوعات مفصَّل | مدرّس فقط |
| GET | `/reports/attendance` | تقرير حضور | مدرّس فقط |
| GET | `/reports/student-performance` | تقرير أداء (فردي/مجموعة/مادة) | مدرّس فقط |
| POST | `/reports/{report_type}/export` | تصدير تقرير PDF/Excel | مدرّس فقط |

**GET `/reports/financial?date_from=2026-07-01&date_to=2026-07-31`**
Response `200 OK`:
```json
{
  "data": {
    "period": { "from": "2026-07-01", "to": "2026-07-31" },
    "total_collected": 5400000,
    "by_method": { "cash": 1200000, "card": 3000000, "fawry": 800000, "instapay": 400000 },
    "total_withdrawn": 2200000,
    "overdue_total": 180000
  }
}
```

**POST `/reports/financial/export`**
Request:
```json
{ "format": "pdf", "date_from": "2026-07-01", "date_to": "2026-07-31" }
```
Response `201 Created`:
```json
{ "data": { "export_id": "exp_001", "status": "processing", "download_url": null } }
```
> التصدير غير متزامن للتقارير الكبيرة؛ الحالة تُتابع عبر `GET /reports/exports/{export_id}`. **دعم RTL كامل في PDF** 🟡 Open Decision (تحدٍ تقني منفصل).

**Status Codes:** `200` نجاح · `201` بدء تصدير · `400` نطاق تاريخ غير صحيح · `403` وصول غير مسموح للطالب/ولي الأمر · `408/504` انتهاء مهلة تقرير كبير (🟡 Open Decision).

---

### 4.14 إشعارات النظام (Notifications)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/notifications` | قائمة إشعارات المستخدم الحالي | مدرّس / طالب |
| PUT | `/notifications/{id}/read` | تعليم إشعار كمقروء | صاحب الإشعار فقط |
| PUT | `/notifications/read-all` | تعليم كل الإشعارات كمقروءة | مدرّس / طالب |

**GET `/notifications?is_read=false`**
Response `200 OK`:
```json
{
  "data": [
    {
      "id": "n_001", "event_type": "grade_update", "title": "تم إضافة درجة جديدة",
      "body": "حصلت على 27/30 في امتحان نصف الفصل", "is_read": false,
      "related_entity_type": "grade_entry", "related_entity_id": "grd_002",
      "created_at": "2026-07-21T14:05:00Z"
    }
  ],
  "meta": { "page": 1, "limit": 20, "total_items": 12, "total_pages": 1 }
}
```
**Status Codes:** `200` نجاح · `403` إشعار مش تابع للمستخدم · `404` غير موجود.

---

### 4.15 واتساب (WhatsApp)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| POST | `/whatsapp/broadcast` | إرسال يدوي بضغطة واحدة لعدد مستقبلين | مدرّس |
| GET | `/whatsapp/messages` | سجل الرسائل المرسَلة | مدرّس |
| GET | `/whatsapp/templates` | قائمة القوالب الجاهزة | مدرّس |

**POST `/whatsapp/broadcast`**
Request:
```json
{
  "student_ids": ["st_001", "st_002", "st_005"],
  "message_kind": "payment_reminder",
  "content_source": "template",
  "template_id": "tpl_payment_reminder_v1",
  "custom_text": null
}
```
Response `201 Created`:
```json
{
  "data": {
    "queued_count": 3,
    "skipped": [
      { "student_id": "st_009", "reason": "MISSING_GUARDIAN_PHONE" }
    ],
    "messages": [
      { "id": "wa_001", "student_id": "st_001", "delivery_status": "queued" }
    ]
  }
}
```
> سلوك التخطي عند غياب رقم ولي الأمر (تخطي بصمت/بتحذير/منع الإرسال كله) 🟡 Open Decision — الافتراض أعلاه: **تخطي مع تقرير**.

**Status Codes:** `201` نجاح إرسال (كامل أو جزئي) · `400` قائمة مستقبلين فاضية · `403` طالب تحت مدرّس تاني · `422` كل المستقبلين بدون رقم ولي أمر صالح.

---

### 4.16 الرسائل (Messages)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/messages/conversations/{student_id}` | جلب محادثة 1:1 | مدرّس (لطلابه) / الطالب (لمدرّسه) |
| POST | `/messages/conversations/{student_id}` | إرسال رسالة | مدرّس / طالب |
| PUT | `/messages/{id}/read` | تعليم رسالة كمقروءة | المستقبل فقط |

**POST `/messages/conversations/{student_id}`**
Request:
```json
{ "content": "من فضلك راجع حل السؤال الثالث قبل الحصة الجاية" }
```
Response `201 Created`:
```json
{
  "data": {
    "id": "msg_001", "teacher_id": "t_001", "student_id": "st_001",
    "sender_role": "teacher", "content": "من فضلك راجع حل السؤال الثالث قبل الحصة الجاية",
    "is_read": false, "created_at": "2026-07-21T16:00:00Z"
  }
}
```
**Status Codes:** `200/201` نجاح · `400` محتوى فاضي · `403` طالب معطَّل/معلَّق يحاول المراسلة، أو محادثة مع طالب تحت مدرّس تاني · `404` غير موجود.

---

### 4.17 البحث (Search)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/search?q={query}&types=students,groups,exams` | بحث شامل محدود بصلاحيات الدور | مدرّس / طالب |

**GET `/search?q=محمد`**
Response `200 OK`:
```json
{
  "data": {
    "students": [ { "id": "st_001", "full_name": "محمد علي", "student_code": "STU-2026-0421" } ],
    "groups": [],
    "exams": [],
    "homeworks": [],
    "learning_materials": [],
    "payments": []
  }
}
```
**Status Codes:** `200` نجاح (حتى لو فاضي) · `400` `q` أقل من الحد الأدنى للأحرف.

---

### 4.18 الأرشيف (Archive)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/archive?category=students` | عرض عناصر مؤرشفة (فلترة `category`) | مدرّس فقط |
| POST | `/archive/{category}/{id}/restore` | استرجاع عنصر مؤرشف | مدرّس فقط |

**GET `/archive?category=groups`**
Response `200 OK`:
```json
{
  "data": [
    { "id": "g_099", "type": "group", "name": "الرياضيات - دفعة 2025", "archived_at": "2026-06-01T00:00:00Z" }
  ],
  "meta": { "page": 1, "limit": 20, "total_items": 3, "total_pages": 1 }
}
```

**POST `/archive/students/{id}/restore`**
Response `200 OK` (المجموعة القديمة لسه نشطة — استرجاع تلقائي):
```json
{ "data": { "id": "st_050", "is_archived": false, "restored_group_id": "g_001" } }
```
Response `200 OK` (المجموعة القديمة اتأرشفت — يحتاج تعيين يدوي):
```json
{ "data": { "id": "st_050", "is_archived": false, "restored_group_id": null, "requires_group_assignment": true } }
```
**Status Codes:** `200` نجاح · `403` عنصر تحت مدرّس تاني · `404` غير موجود.

---

### 4.19 الإعدادات (Settings)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/settings` | جلب إعدادات المستخدم الحالي | مدرّس / طالب |
| PUT | `/settings` | تحديث إعدادات | مدرّس / طالب |
| PUT | `/settings/language` | تبديل اللغة (يفعّل RTL فورًا) | مدرّس / طالب |
| PUT | `/settings/password` | تغيير الباسورد | مدرّس / طالب |

**PUT `/settings/language`**
Request:
```json
{ "preferred_language": "ar" }
```
Response `200 OK`:
```json
{ "data": { "preferred_language": "ar", "text_direction": "rtl" } }
```

**PUT `/settings`**
Request:
```json
{
  "notification_prefs": { "grade_update": true, "payment_reminder": true, "new_message": false },
  "withdrawal_destination_default": "bank_card:•••• 4521"
}
```
Response `200 OK`:
```json
{
  "data": {
    "notification_prefs": { "grade_update": true, "payment_reminder": true, "new_message": false },
    "withdrawal_destination_default": "bank_card:•••• 4521",
    "updated_at": "2026-07-21T17:00:00Z"
  }
}
```
**Status Codes:** `200` نجاح · `400` قيمة غير صحيحة · `401` باسورد قديم خاطئ عند التغيير.

---

### 4.21 الحصص الأونلاين (Online Sessions)

> امتداد لكيان `sessions` (4.6) بـ `delivery_mode=online`. Google Meet/Zoom هما مزوّدا الفيديو الفعليين؛ درسلي طبقة جدولة/وصول فقط.

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| POST | `/schedule/sessions/online` | إنشاء حصة أونلاين واختيار المنصة | مدرّس |
| POST | `/schedule/sessions/{id}/join` | جلب رابط/بيانات الانضمام | مدرّس / طالب مؤهَّل |
| GET | `/schedule/sessions/{id}/recording` | جلب رابط التسجيل (بعد انتهاء الحصة) | مدرّس / طالب مؤهَّل (🟡 Open Decision لمين بالظبط) |

**POST `/schedule/sessions/online`**
Request:
```json
{ "group_id": "g_001", "scheduled_at": "2026-07-22T15:00:00Z", "duration_minutes": 60, "online_platform": "zoom" }
```
Response `201 Created`:
```json
{
  "data": {
    "id": "s_101", "delivery_mode": "online", "online_platform": "zoom",
    "external_meeting_id": "823 471 0192",
    "join_url": "https://zoom.us/j/8234710192",
    "status": "scheduled", "scheduled_at": "2026-07-22T15:00:00Z"
  }
}
```
> توقيت توليد `join_url` (وقت الإنشاء أم وقت البدء) وسلوك الانضمام (iframe داخل درسلي أم تاب خارجي) 🟡 Open Decision — مؤجَّل صراحةً لجلسة تصميم مخصصة مع الفريق التقني.

**GET `/schedule/sessions/{id}/join`**
Response `200 OK`:
```json
{ "data": { "join_url": "https://zoom.us/j/8234710192", "role_in_meeting": "host" } }
```
**Status Codes:** `200/201` نجاح · `400` منصة غير مدعومة · `403` حصة تحت مدرّس تاني، أو طالب غير مسجَّل بالمجموعة · `404` غير موجود · `503` انقطاع تكامل Meet/Zoom (🟡 Open Decision لسلوك المعالجة).

---

### 4.3 المواد الدراسية (Subjects)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/subjects` | قائمة مواد المدرّس (أو مواد الطالب المسجّل فيها) | مدرّس (كامل) / طالب (قراءة فقط لمواده) |
| POST | `/subjects` | إنشاء مادة | مدرّس |
| GET | `/subjects/{id}` | تفاصيل مادة | مدرّس / طالب (لو مسجّل فيها) |
| PUT | `/subjects/{id}` | تعديل مادة | مدرّس |
| DELETE | `/subjects/{id}` | حذف مادة | مدرّس |

**POST `/subjects`**
Request:
```json
{ "name": "الرياضيات", "default_price": 15000 }
```
Response `201 Created`:
```json
{
  "data": {
    "id": "subj_001",
    "teacher_id": "tch_001",
    "name": "الرياضيات",
    "default_price": 15000,
    "created_at": "2026-07-21T09:00:00Z"
  }
}
```

**GET `/subjects`**
Response `200 OK`:
```json
{
  "data": [
    { "id": "subj_001", "name": "الرياضيات", "default_price": 15000, "groups_count": 3 }
  ],
  "meta": { "page": 1, "limit": 20, "total_items": 4, "total_pages": 1 }
}
```

**Status Codes:** `201` إنشاء ناجح · `200` نجاح · `400` اسم ناقص/سعر سالب · `403` مادة تحت مدرّس تاني (يظهر كـ `404`) · `404` غير موجودة · `422` حذف مادة ليها مجموعات نشطة (🟡 سلوك الحذف قرار مفتوح — الافتراض الحالي: منع الحذف وإرجاع `422`).

---

### 4.4 الطلاب (Students)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/students` | قائمة طلاب المدرّس (نشطين، مع فلاتر) | مدرّس |
| POST | `/students` | إضافة طالب فردي | مدرّس |
| POST | `/students/bulk-import` | استيراد جماعي (Excel/Sheets) | مدرّس |
| GET | `/students/{id}` | بروفايل طالب | مدرّس / طالب (نفسه فقط) |
| PUT | `/students/{id}` | تعديل بروفايل | مدرّس (كامل) / طالب (نطاق محدود 🟡 Open Decision) |
| POST | `/students/{id}/enrollments` | تسجيل الطالب في مادة/مجموعة إضافية | مدرّس |
| POST | `/students/{id}/archive` | تعطيل/أرشفة الطالب | مدرّس |
| POST | `/students/{id}/restore` | استرجاع طالب من الأرشيف | مدرّس |

**POST `/students`**
Request:
```json
{
  "full_name": "محمد علي",
  "phone": "+201098765432",
  "email": null,
  "guardian_phone": "+201055556666",
  "academic_level": "الصف الثالث الإعدادي",
  "school_name": "مدرسة النصر",
  "password": "InitialPass456",
  "group_ids": ["grp_001"]
}
```
Response `201 Created`:
```json
{
  "data": {
    "id": "st_045",
    "full_name": "محمد علي",
    "status": "approved",
    "student_code": "STU-2026-0422",
    "qr_code_value": "QR-STU-2026-0422-AB12",
    "created_at": "2026-07-21T09:10:00Z"
  }
}
```

**POST `/students/bulk-import`** (`multipart/form-data`)
Request: `file: students_batch.xlsx`
Response `201 Created`:
```json
{
  "data": {
    "created": [
      { "id": "st_046", "full_name": "علياء حسن", "student_code": "STU-2026-0423" }
    ],
    "errors": [
      { "row": 7, "reason": "رقم الموبايل مكرر", "raw_row": { "full_name": "أحمد سامي", "phone": "+201000000000" } }
    ]
  }
}
```
> شكل خطأ كل صف 🟡 Open Decision — الافتراض أعلاه هو أفضل ممارسة قياسية.

**GET `/students/{id}`**
Response `200 OK`:
```json
{
  "data": {
    "id": "st_045",
    "full_name": "محمد علي",
    "phone": "+201098765432",
    "email": null,
    "guardian_phone": "+201055556666",
    "academic_level": "الصف الثالث الإعدادي",
    "school_name": "مدرسة النصر",
    "profile_photo_url": null,
    "status": "approved",
    "student_code": "STU-2026-0422",
    "qr_code_value": "QR-STU-2026-0422-AB12",
    "is_archived": false,
    "groups": [ { "id": "grp_001", "name": "الرياضيات - الصف الثالث" } ],
    "created_at": "2026-07-21T09:10:00Z"
  }
}
```

**POST `/students/{id}/archive`**
Response `200 OK`:
```json
{ "data": { "id": "st_045", "is_archived": true, "archived_at": "2026-07-21T10:00:00Z" } }
```

**Status Codes:** `201` إنشاء/استيراد · `200` نجاح · `400` حقل إجباري ناقص (اسم/موبايل) · `403`/`404` طالب تحت مدرّس تاني · `409` رقم/إيميل مكرر.

---

### 4.5 المجموعات (Groups)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/groups` | قائمة مجموعات المدرّس | مدرّس |
| POST | `/groups` | إنشاء مجموعة | مدرّس |
| GET | `/groups/{id}` | تفاصيل مجموعة | مدرّس / طالب (عضويته فقط) |
| PUT | `/groups/{id}` | تعديل مجموعة | مدرّس |
| DELETE | `/groups/{id}` | حذف/أرشفة مجموعة | مدرّس |
| POST | `/groups/{id}/enrollments` | تسجيل طالب في المجموعة | مدرّس |
| POST | `/groups/{id}/transfer-student` | نقل طالب لمجموعة تانية (يحافظ على التاريخ) | مدرّس |

**POST `/groups`**
Request:
```json
{ "subject_id": "subj_001", "name": "الرياضيات - الصف الثالث - مجموعة أ", "academic_level": "الصف الثالث الإعدادي", "price": 18000 }
```
Response `201 Created`:
```json
{
  "data": {
    "id": "grp_001",
    "subject_id": "subj_001",
    "name": "الرياضيات - الصف الثالث - مجموعة أ",
    "academic_level": "الصف الثالث الإعدادي",
    "price": 18000,
    "students_count": 0,
    "created_at": "2026-07-21T09:20:00Z"
  }
}
```

**POST `/groups/{id}/transfer-student`**
Request:
```json
{ "student_id": "st_045", "to_group_id": "grp_002" }
```
Response `200 OK`:
```json
{
  "data": {
    "old_enrollment": { "id": "enr_001", "group_id": "grp_001", "ended_at": "2026-07-21T10:00:00Z" },
    "new_enrollment": { "id": "enr_002", "group_id": "grp_002", "started_at": "2026-07-21T10:00:00Z" }
  }
}
```

**Status Codes:** `201`/`200` نجاح · `400` مجموعة بدون مادة · `403`/`404` وصول خارج نطاق المدرّس · `422` حذف مجموعة فيها طلاب مسجلين (🟡 Open Decision — افتراض: يُمنع الحذف).

---

### 4.6 الجدول (Schedule)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/schedule` | عرض الجدول (فلترة `?from=&to=`) | مدرّس / طالب |
| POST | `/schedule/rules` | إنشاء قاعدة تكرار أسبوعية لمجموعة | مدرّس |
| PUT | `/schedule/rules/{id}` | تعديل قاعدة التكرار | مدرّس |
| DELETE | `/schedule/rules/{id}` | إلغاء قاعدة التكرار | مدرّس |
| POST | `/sessions` | إنشاء حصة فردية (خارج التكرار) | مدرّس |
| PUT | `/sessions/{id}/reschedule` | تأجيل/تعديل موعد حصة | مدرّس |
| POST | `/sessions/{id}/cancel` | إلغاء حصة | مدرّس |

**POST `/schedule/rules`**
Request:
```json
{ "group_id": "grp_001", "day_of_week": 6, "start_time": "16:00:00", "duration_minutes": 90 }
```
Response `201 Created`:
```json
{ "data": { "id": "rule_001", "group_id": "grp_001", "day_of_week": 6, "start_time": "16:00:00", "duration_minutes": 90, "is_active": true } }
```

**PUT `/sessions/{id}/reschedule`**
Request:
```json
{ "scheduled_at": "2026-07-23T17:00:00Z" }
```
Response `200 OK` (بدون تعارض):
```json
{ "data": { "id": "sess_010", "status": "rescheduled", "scheduled_at": "2026-07-23T17:00:00Z" } }
```
Response `200 OK` (بتعارض — يُسمح للمدرّس بالتجاوز بعد تحذير في الفرونت):
```json
{
  "data": { "id": "sess_010", "status": "conflicted", "scheduled_at": "2026-07-23T17:00:00Z" },
  "warnings": [ { "code": "SCHEDULE_CONFLICT", "conflicting_session_id": "sess_011" } ]
}
```
**Status Codes:** `201`/`200` نجاح · `400` وقت غير صالح · `403`/`404` خارج نطاق المدرّس.

---

### 4.7 الحضور (Attendance)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/sessions/{id}/attendance` | قائمة حضور حصة | مدرّس |
| PUT | `/sessions/{id}/attendance` | تسجيل/تعديل حضور يدوي (Bulk) | مدرّس |
| POST | `/attendance/qr-scan` | تسجيل حضور بمسح QR | مدرّس |
| GET | `/attendance/{id}/audit-trail` | سجل تعديلات سجل حضور معيّن | مدرّس |
| GET | `/students/{id}/attendance` | تاريخ حضور طالب | مدرّس / طالب (نفسه) |

**PUT `/sessions/{id}/attendance`**
Request:
```json
{ "records": [ { "student_id": "st_045", "status": "present" }, { "student_id": "st_046", "status": "absent" } ] }
```
Response `200 OK`:
```json
{
  "data": [
    { "id": "att_001", "student_id": "st_045", "status": "present", "recorded_via": "manual" },
    { "id": "att_002", "student_id": "st_046", "status": "absent", "recorded_via": "manual" }
  ]
}
```

**POST `/attendance/qr-scan`**
Request:
```json
{ "session_id": "sess_010", "qr_code_value": "QR-STU-2026-0422-AB12" }
```
Response `201 Created`:
```json
{
  "data": { "id": "att_003", "student_id": "st_045", "status": "present", "recorded_via": "qr_scan", "created_at": "2026-07-21T16:01:00Z" },
  "side_effects": { "whatsapp_notification_sent": true }
}
```
Response `404 Not Found` (كود مش تابع لأي طالب في نطاق المدرّس):
```json
{ "error": { "code": "INVALID_QR_CODE", "message": "كود QR غير صالح", "details": [] } }
```

**GET `/attendance/{id}/audit-trail`**
Response `200 OK`:
```json
{
  "data": [
    { "id": "aud_001", "old_status": null, "new_status": "present", "changed_by_teacher_id": "tch_001", "created_at": "2026-07-21T16:01:00Z" },
    { "id": "aud_002", "old_status": "present", "new_status": "late", "changed_by_teacher_id": "tch_001", "created_at": "2026-07-21T16:15:00Z" }
  ]
}
```
**Status Codes:** `201`/`200` نجاح · `400` قائمة فارغة · `404` كود QR غير صالح / حصة غير موجودة.

---

### 4.8 الواجبات (Homework)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/homeworks` | قائمة واجبات (فلترة `?group_id=&student_id=`) | مدرّس / طالب |
| POST | `/homeworks` | إنشاء واجب (لمجموعة أو طالب) | مدرّس |
| GET | `/homeworks/{id}` | تفاصيل واجب + التسليمات | مدرّس / طالب |
| PUT | `/homeworks/{id}` | تعديل واجب | مدرّس |
| DELETE | `/homeworks/{id}` | حذف واجب | مدرّس |
| POST | `/homeworks/{id}/submissions` | تسليم واجب | طالب |
| PUT | `/homeworks/{id}/submissions/{sid}/grade` | تصحيح/تقييم تسليم | مدرّس |

**POST `/homeworks`**
Request:
```json
{ "target_type": "group", "group_id": "grp_001", "title": "واجب الوحدة 3", "instructions": "حل تمارين ص 45-47", "due_at": "2026-07-25T00:00:00Z" }
```
Response `201 Created`:
```json
{ "data": { "id": "hw_001", "target_type": "group", "group_id": "grp_001", "title": "واجب الوحدة 3", "due_at": "2026-07-25T00:00:00Z", "created_at": "2026-07-21T09:30:00Z" } }
```

**POST `/homeworks/{id}/submissions`**
Request:
```json
{ "submission_type": "file", "attachment_url": "https://cdn.darsly.app/uploads/hw_sub_991.pdf" }
```
Response `201 Created`:
```json
{ "data": { "id": "sub_001", "homework_id": "hw_001", "student_id": "st_045", "is_late": false, "submitted_at": "2026-07-24T20:00:00Z" } }
```
**Status Codes:** `201` نجاح إنشاء/تسليم · `400` استهداف مزدوج (مجموعة وطالب معًا) · `403`/`404` خارج النطاق · `409` تسليم مكرر (لو القرار = عدم السماح بالاستبدال).

---

### 4.9 الامتحانات (Exams)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/exams` | قائمة امتحانات | مدرّس / طالب |
| POST | `/exams` | إنشاء امتحان (Built-in) | مدرّس |
| POST | `/exams/import` | استيراد امتحان من ملف خارجي | مدرّس |
| GET | `/exams/{id}` | تفاصيل امتحان + الأسئلة | مدرّس / طالب |
| PUT | `/exams/{id}` | تعديل امتحان | مدرّس |
| DELETE | `/exams/{id}` | حذف امتحان | مدرّس |
| POST | `/exams/{id}/questions` | إضافة سؤال | مدرّس |
| POST | `/exams/{id}/submissions` | تسليم إجابات الطالب | طالب |
| GET | `/exams/{id}/submissions/{sid}` | نتيجة تسليم معيّن | مدرّس / طالب (نفسه) |
| PUT | `/exams/{id}/submissions/{sid}/grade` | تصحيح يدوي | مدرّس |

**POST `/exams`**
Request:
```json
{ "subject_id": "subj_001", "group_id": "grp_001", "title": "امتحان نصف الفصل", "source": "built_in", "auto_grading_enabled": true }
```
Response `201 Created`:
```json
{ "data": { "id": "exam_001", "title": "امتحان نصف الفصل", "auto_grading_enabled": true, "created_at": "2026-07-21T09:40:00Z" } }
```

**POST `/exams/{id}/questions`**
Request:
```json
{ "question_type": "multiple_choice", "question_text": "ناتج 5 + 3 يساوي؟", "options": ["6", "7", "8", "9"], "correct_option_index": 2, "points": 5, "order_index": 1 }
```
Response `201 Created`:
```json
{ "data": { "id": "q_001", "exam_id": "exam_001", "question_type": "multiple_choice", "points": 5, "order_index": 1 } }
```

**POST `/exams/{id}/submissions`**
Request:
```json
{ "answers": [ { "question_id": "q_001", "answer": 2 } ] }
```
Response `201 Created` (تصحيح تلقائي فوري):
```json
{
  "data": {
    "id": "esub_001",
    "exam_id": "exam_001",
    "student_id": "st_045",
    "score": 5,
    "grading_status": "auto_graded",
    "submitted_at": "2026-07-21T18:00:00Z"
  }
}
```
**Status Codes:** `201` نجاح · `400` إجابات ناقصة · `403`/`404` خارج النطاق · `409` تسليم مكرر للامتحان نفسه.

---

### 4.10 الدرجات (Grades)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/grades` | قائمة درجات (فلترة `?student_id=&group_id=&category=`) | مدرّس / طالب |
| POST | `/grades` | إدخال درجة يدوية | مدرّس |
| PUT | `/grades/{id}` | تعديل درجة | مدرّس |
| DELETE | `/grades/{id}` | حذف درجة | مدرّس |

**POST `/grades`**
Request:
```json
{ "student_id": "st_045", "category": "participation", "value": 9, "max_value": 10, "note": "مشاركة ممتازة" }
```
Response `201 Created`:
```json
{ "data": { "id": "gr_010", "student_id": "st_045", "category": "participation", "value": 9, "max_value": 10, "created_at": "2026-07-21T19:00:00Z" } }
```
**Status Codes:** `201`/`200` نجاح · `400` `value > max_value` · `403`/`404` خارج النطاق.

---

### 4.11 المدفوعات (Payments) — دفع مباشر بدون محفظة

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/payments` | سجل المدفوعات (فلترة `?student_id=&status=`) | مدرّس / طالب (خاصة به) |
| POST | `/payments/checkout` | بدء عملية دفع أونلاين (كارت/فوري/إنستاباي) | طالب |
| POST | `/payments/webhook` | استقبال تأكيد بوابة الدفع | عام (موقّع بتوقيع البوابة) |
| POST | `/payments/manual` | تسجيل دفعة كاش يدويًا | مدرّس |
| GET | `/payments/earnings-summary` | ملخص الأرباح القابلة للسحب | مدرّس |
| POST | `/withdrawals` | طلب سحب أرباح | مدرّس |
| GET | `/withdrawals` | سجل طلبات السحب | مدرّس |

**POST `/payments/checkout`**
Request:
```json
{ "payable_type": "group", "payable_id": "grp_001", "method": "card" }
```
Response `200 OK`:
```json
{
  "data": {
    "payment_id": "pay_001",
    "status": "pending",
    "redirect_url": "https://checkout.paymentgateway.com/session/abc123"
  }
}
```

**POST `/payments/manual`**
Request:
```json
{ "student_id": "st_045", "payable_type": "session", "payable_id": "sess_010", "amount": 15000, "method": "cash" }
```
Response `201 Created`:
```json
{
  "data": {
    "id": "pay_002", "student_id": "st_045", "amount": 15000, "net_amount": 15000,
    "method": "cash", "status": "completed", "created_at": "2026-07-21T19:10:00Z"
  }
}
```

**GET `/payments/earnings-summary`**
Response `200 OK`:
```json
{
  "data": {
    "total_earned": 5400000,
    "total_withdrawn": 4000000,
    "available_for_withdrawal": 1400000,
    "pending_payments_amount": 150000
  }
}
```

**POST `/withdrawals`**
Request:
```json
{ "amount": 1000000, "destination_method": "instapay", "destination_reference": "ahmed@instapay" }
```
Response `201 Created`:
```json
{ "data": { "id": "wd_001", "amount": 1000000, "status": "requested", "requested_at": "2026-07-21T19:20:00Z" } }
```
Response `422 Unprocessable Entity` (رصيد غير كافٍ):
```json
{ "error": { "code": "INSUFFICIENT_AVAILABLE_BALANCE", "message": "المبلغ المطلوب أكبر من الأرباح المتاحة للسحب", "details": [] } }
```
**Status Codes:** `201`/`200` نجاح · `400` مبلغ/طريقة غير صالحة · `402`/`400` فشل الدفع من البوابة (يُرصد عبر `payments.status=failed`) · `404` عنصر الدفع غير موجود · `422` سحب أكبر من المتاح.

---

### 4.12 المواد التعليمية (Learning Materials)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/materials` | قائمة مواد (فلترة `?subject_id=&group_id=&student_id=`) | مدرّس / طالب |
| POST | `/materials` | رفع/إضافة مادة تعليمية | مدرّس |
| GET | `/materials/{id}` | تفاصيل مادة | مدرّس / طالب (لو له حق وصول) |
| PUT | `/materials/{id}` | تعديل مادة | مدرّس |
| DELETE | `/materials/{id}` | حذف مادة | مدرّس |
| POST | `/materials/{id}/purchase` | شراء مادة مدفوعة | طالب |

**POST `/materials`**
Request:
```json
{
  "target_type": "group", "group_id": "grp_001", "material_type": "pdf",
  "title": "ملخص الوحدة 3", "file_url": "https://cdn.darsly.app/materials/unit3.pdf",
  "access_mode": "downloadable", "pricing_mode": "paid", "price": 5000
}
```
Response `201 Created`:
```json
{ "data": { "id": "mat_001", "title": "ملخص الوحدة 3", "pricing_mode": "paid", "price": 5000, "created_at": "2026-07-21T19:30:00Z" } }
```

**POST `/materials/{id}/purchase`**
Response `200 OK`:
```json
{ "data": { "payment_id": "pay_003", "status": "pending", "redirect_url": "https://checkout.paymentgateway.com/session/xyz789" } }
```
**Status Codes:** `201`/`200` نجاح · `400` استهداف غير مكتمل / سعر ناقص لمادة مدفوعة · `403` وصول لمادة برّه نطاق الطالب · `409` شراء مكرر لنفس المادة.

---

### 4.13 التقارير (Reports)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/reports/attendance` | تقرير حضور (فلترة `?group_id=&from=&to=`) | مدرّس |
| GET | `/reports/financial` | تقرير مالي (دخل/مصروفات حسب الفترة) | مدرّس |
| GET | `/reports/academic` | تقرير أداء أكاديمي (درجات/امتحانات) | مدرّس |
| GET | `/reports/{type}/export` | تصدير تقرير PDF/Excel | مدرّس |

**GET `/reports/financial?from=2026-07-01&to=2026-07-31`**
Response `200 OK`:
```json
{
  "data": {
    "total_income": 1250000,
    "by_method": { "cash": 400000, "card": 600000, "fawry": 150000, "instapay": 100000 },
    "by_subject": [ { "subject_id": "subj_001", "subject_name": "الرياضيات", "amount": 800000 } ],
    "pending_amount": 150000
  }
}
```

**GET `/reports/financial/export?from=2026-07-01&to=2026-07-31&format=pdf`**
Response `200 OK` (Binary):
```
Content-Type: application/pdf
Content-Disposition: attachment; filename="financial-report-2026-07.pdf"
```
**Status Codes:** `200` نجاح · `400` مدى تاريخ غير صالح · `422` نوع تصدير غير مدعوم.

---

### 4.14 إشعارات النظام (Notifications)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/notifications` | قائمة إشعارات المستخدم | مدرّس / طالب |
| PUT | `/notifications/{id}/read` | تعليم إشعار كمقروء | مدرّس / طالب |
| PUT | `/notifications/read-all` | تعليم الكل كمقروء | مدرّس / طالب |

**GET `/notifications`**
Response `200 OK`:
```json
{
  "data": [
    { "id": "notif_001", "event_type": "attendance", "title": "تم تسجيل حضورك", "body": "تم تسجيل حضورك في حصة الرياضيات", "is_read": false, "created_at": "2026-07-21T16:01:00Z" }
  ],
  "meta": { "page": 1, "limit": 20, "total_items": 12, "total_pages": 1 }
}
```
**Status Codes:** `200` نجاح · `404` إشعار غير موجود لهذا الحساب.

---

### 4.15 واتساب (WhatsApp)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/whatsapp/templates` | قوالب الرسائل الجاهزة | مدرّس |
| POST | `/whatsapp/send` | إرسال رسالة يدوية لولي أمر طالب | مدرّس |
| GET | `/whatsapp/messages` | سجل رسائل واتساب مُرسَلة (فلترة `?student_id=`) | مدرّس |

**POST `/whatsapp/send`**
Request:
```json
{ "student_id": "st_045", "message_kind": "payment_reminder", "content_source": "template", "template_id": "tpl_payment_reminder", "variables": { "amount": "150 جنيه" } }
```
Response `201 Created`:
```json
{ "data": { "id": "wa_001", "student_id": "st_045", "message_kind": "payment_reminder", "delivery_status": "queued", "created_at": "2026-07-21T19:40:00Z" } }
```
**Status Codes:** `201` نجاح · `400` قالب غير موجود / متغيرات ناقصة · `404` طالب بدون `guardian_phone` · `502` فشل مزوّد واتساب.

---

### 4.16 الرسائل (Messages)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/conversations` | قائمة محادثات المدرّس (كل الطلاب اللي فيهم رسائل) | مدرّس |
| GET | `/conversations/{studentId}/messages` | رسائل محادثة معيّنة | مدرّس / طالب (نفسه) |
| POST | `/conversations/{studentId}/messages` | إرسال رسالة | مدرّس / طالب |
| PUT | `/conversations/{studentId}/read` | تعليم المحادثة كمقروءة | مدرّس / طالب |

**POST `/conversations/{studentId}/messages`**
Request:
```json
{ "content": "من فضلك ابعتلي الواجب تاني، الملف مش واضح" }
```
Response `201 Created`:
```json
{ "data": { "id": "msg_001", "sender_role": "teacher", "content": "من فضلك ابعتلي الواجب تاني، الملف مش واضح", "is_read": false, "created_at": "2026-07-21T19:50:00Z" } }
```
**Status Codes:** `201` نجاح · `400` محتوى فارغ · `403`/`404` محادثة برّه نطاق المستخدم.

---

### 4.17 البحث (Search)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/search?q=&type=` | بحث موحّد (`type`: `student`,`group`,`subject`,`material`,`exam`) | مدرّس |

**GET `/search?q=محمد&type=student`**
Response `200 OK`:
```json
{
  "data": [
    { "type": "student", "id": "st_045", "title": "محمد علي", "subtitle": "STU-2026-0422", "url": "/students/st_045" }
  ]
}
```
**Status Codes:** `200` نجاح (حتى لو صفر نتائج) · `400` `q` أقصر من الحد الأدنى (🟡 Open Decision — افتراض: حرفين).

---

### 4.18 الأرشيف (Archive)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/archive/students` | قائمة الطلاب المؤرشفين | مدرّس |
| GET | `/archive/groups` | قائمة المجموعات المؤرشفة | مدرّس |
| POST | `/groups/{id}/archive` | أرشفة مجموعة | مدرّس |
| POST | `/groups/{id}/restore` | استرجاع مجموعة | مدرّس |

**GET `/archive/students`**
Response `200 OK`:
```json
{
  "data": [ { "id": "st_030", "full_name": "خالد سعيد", "archived_at": "2026-06-01T00:00:00Z" } ],
  "meta": { "page": 1, "limit": 20, "total_items": 5, "total_pages": 1 }
}
```
**Status Codes:** `200` نجاح · `404` سجل غير موجود عند الاسترجاع.

---

### 4.19 الإعدادات (Settings)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| GET | `/settings` | إعدادات الحساب الحالي | مدرّس / طالب |
| PUT | `/settings` | تعديل الإعدادات | مدرّس / طالب |
| PUT | `/settings/password` | تغيير كلمة المرور | مدرّس / طالب |

**PUT `/settings`**
Request:
```json
{ "preferred_language": "ar", "notification_prefs": { "attendance": true, "payment_reminder": true, "grade_update": false } }
```
Response `200 OK`:
```json
{ "data": { "preferred_language": "ar", "notification_prefs": { "attendance": true, "payment_reminder": true, "grade_update": false }, "updated_at": "2026-07-21T20:00:00Z" } }
```
**Status Codes:** `200` نجاح · `400` قيمة إعداد غير صالحة · `401` باسورد حالي غلط (عند تغيير الباسورد).

---

### 4.21 الحصص الأونلاين (Online Sessions)

| Method | Endpoint | الوصف | الصلاحية |
|---|---|---|---|
| POST | `/sessions/{id}/online-link` | توليد رابط الاجتماع (Meet/Zoom) لحصة | مدرّس |
| GET | `/sessions/{id}/online-link` | استرجاع رابط الانضمام | مدرّس / طالب |
| PUT | `/sessions/{id}/recording` | إرفاق رابط التسجيل بعد الحصة | مدرّس |

**POST `/sessions/{id}/online-link`**
Request:
```json
{ "platform": "google_meet" }
```
Response `201 Created`:
```json
{ "data": { "session_id": "sess_010", "online_platform": "google_meet", "join_url": "https://meet.google.com/abc-defg-hij", "external_meeting_id": "abc-defg-hij" } }
```
**Status Codes:** `201`/`200` نجاح · `400` منصة غير مدعومة · `502` فشل تكامل مزوّد الاجتماعات · `404` الحصة `delivery_mode != online`.

---

## 5. ملحق: قائمة موحّدة بكل البنود المفتوحة (Open Decisions) المطلوب حسمها مع أحمد

| # | البند | الموقع |
|---|---|---|
| 1 | قناة إرسال OTP الافتراضية ومدة صلاحيته وحد المحاولات | 2.2.4 |
| 2 | هل حقل `guardian_phone` إجباري 100%، وهل يُتحقق من صيغته؟ | 2.2.3 |
| 3 | أولوية التسعير عند تعارض سعر المادة/المجموعة/الحصة الفردية | 2.2.6, 4.3 |
| 4 | الحد الأقصى لمدة/عدد الحصص المتكررة والفرق الزمني بينها | 2.2.8 |
| 5 | مكان تخزين تسجيلات الحصص الأونلاين (سياسة استضافة) | 2.2.9, 4.21 |
| 6 | سلوك تسليم واجب مكرر: استبدال أم رفض | 2.2.13, 4.8 |
| 7 | الأنواع الكاملة المدعومة لأسئلة الامتحان غير الاختيار من متعدد | 2.2.15 |
| 8 | مزوّد بوابة الدفع النهائي (Paymob/Fawry Pay/غيره) والرسوم | 2.2.19, 4.11 |
| 9 | نافذة تسوية السحب (Settlement Window) والرسوم ومدة المعالجة | 2.2.20, 4.11 |
| 10 | مزوّد واتساب الرسمي (Meta Cloud API مباشر أم BSP وسيط) | 2.2.22, 4.15 |
| 11 | حدود حجم/نوع مرفقات الواجبات والرسائل | 2.2.12, 2.2.23 |
| 12 | سياسة الاحتفاظ بالإشعارات القديمة (Retention) | 2.2.21 |
| 13 | الحد الأدنى لطول نص البحث `q` | 4.17 |
| 14 | نطاق تعديل الطالب لبروفايله الشخصي (PUT `/students/{id}`) | 4.4 |

---

*نهاية التوثيق — جاهز لتسليمه لفريق الـ Backend والـ Frontend للبدء الفوري بالتوازي (Backend حسب سكيمات القسم 2، Frontend حسب الـ Mock JSON في القسم 3).*
