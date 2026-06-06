Sprint 1: Registration & Auth - Technical Blueprint
This blueprint outlines the architecture for the first phase of development: the Authentication and Registration system.

1. Security Design: National ID Handling
To ensure privacy and enforce the "One Person = One Account" rule without exposing raw National IDs in the database, we use a dual-column approach:

Unique Constraint (Hashing):

We use SHA-256 hashing (with a system-wide secret salt) to generate a deterministic hash.
Column: national_id_hash (Unique Index).
Purpose: O(1) lookup to prevent duplicate registrations.
Data Recovery (Encryption):

We use AES-256-GCM symmetric encryption via the cryptography Python library.
Column: national_id_encrypted (Byte array).
Purpose: Allows authorized Admins/HR to decrypt and view the original National ID.
2. Database Schema (PostgreSQL)
Table: Users
Mermaid diagram
3. API Endpoints (FastAPI)
POST /api/auth/register
Request Payload: full_name, email, phone, national_id, password
Flow:
Validate inputs (e.g., National ID format).
Compute hash = SHA256(national_id + SECRET_SALT).
Query DB for national_id_hash == hash. If exists -> Return 400 "هذا الرقم مسجل مسبقاً".
Compute encrypted = AES_Encrypt(national_id, ENCRYPTION_KEY).
Hash the password with Bcrypt.
Save to DB.
Generate and return Access Token (JWT) & Refresh Token.
POST /api/auth/login
Request Payload: email (or national_id), password
Flow:
Find user by email.
Verify Bcrypt password.
Return JWT Access & Refresh Tokens.
POST /api/auth/refresh
Flow:
Validate Refresh Token.
Issue a new Access Token.
GET /api/auth/me
Flow:
Requires valid Access Token.
Returns user profile (excluding sensitive fields like encrypted National ID).
4. Frontend Implementation (Next.js)
Registration Form (/register):

Client-side validation using Zod and React Hook Form.
Display Arabic error messages (e.g., "الرقم القومي يجب أن يكون 14 رقماً").
On success, store JWT in HTTP-Only cookies (for Next.js App Router security) or LocalStorage.
Login Form (/login):

Simple email/password form.
Redirects to the Dashboard upon success.
Auth Context / Middleware:

Next.js Middleware (middleware.ts) to protect routes like /exam, /lab, /interview.
Redirect unauthenticated users to /login.
5. Security & Environment Variables
The system will require the following secrets defined in .env:

env

# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/assessments
# JWT
JWT_SECRET_KEY=your_super_secret_jwt_key
JWT_REFRESH_SECRET_KEY=your_super_secret_refresh_key
ACCESS_TOKEN_EXPIRE_MINUTES=30
REFRESH_TOKEN_EXPIRE_DAYS=7
# National ID Encryption (Must be exactly 32 bytes / 256 bits, base64 encoded)
ENCRYPTION_KEY=DUMMY_32_BYTE_BASE64_KEY_HERE===
NATIONAL_ID_SALT=some_random_salt_for_sha256
























مقدمة عامة







الهدف من هذا المخطط هو بناء نظام تسجيل دخول وإنشاء حساب آمن وقابل للتوسع، مع التركيز على متطلب أساسي: لا يمكن أن يكون هناك أكثر من حساب لنفس الشخص (One Person = One Account)، مع الحفاظ على سرية الرقم القومي (National ID) وعدم تخزينه بشكل نصي عادي في قاعدة البيانات. سيتم استخدام تقنيات حديثة: FastAPI للواجهة الخلفية، Next.js للواجهة الأمامية، PostgreSQL كقاعدة بيانات.

1. التصميم الأمني للتعامل مع الرقم القومي
الرقم القومي هو معلومة حساسة للغاية (PII - Personally Identifiable Information). لا يمكن تخزينه كنص عادي لأسباب قانونية وأمنية. لكننا نحتاج إلى:

التأكد من عدم وجود حساب مكرر: يجب أن نتمكن من البحث بسرعة عما إذا كان هذا الرقم القومي قد استُخدم من قبل.

قدرة محدودة على استعادة الرقم الأصلي: جهة معينة (مثل المسؤولين أو قسم الموارد البشرية) قد تحتاج إلى رؤية الرقم القومي للتحقق من الهوية في حالات نادرة.

لحل هذين الشرطين المتناقضين جزئيًا (التعقيد السريع مقابل إمكانية الفك)، تم اعتماد نهج العمودين المزدوج (Dual-Column Approach).

1.1 عمود التجزئة (Hashing) للتفرد والبحث السريع
الخوارزمية: SHA-256 (وهي دالة تجزئة أحادية الاتجاه، أي لا يمكن عكسها).

التمليح (Salting): يتم إضافة قيمة سرية عامة للنظام (SECRET_SALT) إلى الرقم القومي قبل التجزئة. هذا يمنع هجمات جداول قوس قزح (Rainbow Tables) حتى لو تسربت قاعدة البيانات.

العمود في قاعدة البيانات: national_id_hash من نوع VARCHAR(64) أو TEXT، مع فهرس فريد (Unique Index).

كيفية الاستخدام عند التسجيل:

يستقبل الخادم الرقم القومي national_id من المستخدم.
يُنفذ: hash = SHA256(national_id + SECRET_SALT).
يتم عمل استعلام على قاعدة البيانات: SELECT id FROM users WHERE national_id_hash = hash.
إذا وُجد صف، نرفض التسجيل برسالة "هذا الرقم مسجل مسبقاً".
إذا لم يُوجد، نكمل بقية خطوات التسجيل.
لماذا هذا آمن؟ حتى لو تسربت قاعدة البيانات، لا يمكن لأي مهاجم معرفة الرقم القومي الأصلي من هذا العمود، لأنه لا يمكن عكس SHA-256، ولأن السالت (salt) سري ولا يعرفه المهاجم.

1.2 عمود التشفير (Encryption) لإمكانية استعادة البيانات
الخوارزمية: AES-256-GCM. هذا تشفير متماثل (نفس المفتاح للتشفير وفك التشفير). وضع GCM يضمن أيضًا سلامة البيانات (Integrity) من خلال علامة المصادقة (Authentication Tag).

العمود في قاعدة البيانات: national_id_encrypted من نوع BYTEA (أو BLOB). يحتوي هذا العمود على النص المشفر، بالإضافة إلى الـ Nonce (رقم يستخدم لمرة واحدة) وعلامة المصادقة (التنسيق النهائي يعتمد على المكتبة المستخدمة).

كيفية الاستخدام:

عند التسجيل: encrypted = AES_Encrypt(national_id, ENCRYPTION_KEY).
يُخزن في قاعدة البيانات.
عندما يريد مسؤول (Admin/HR) مُصرَّح له الاطلاع على الرقم القومي لمستخدم معين (عبر واجهة إدارية خاصة)، يُنفذ decrypted = AES_Decrypt(national_id_encrypted, ENCRYPTION_KEY) ويعرض له الرقم الأصلي.
لماذا هذا ضروري؟ هناك حالات إدارية وقانونية تحتاج فيها المؤسسة إلى رؤية الرقم القومي (مثلاً: مطابقة البيانات مع جهة حكومية). التشفير يسمح بذلك بشكل آمن، على عكس التجزئة التي لا تسمح أبدًا بمعرفة النص الأصلي.

إدارة المفاتيح: مفتاح التشفير ENCRYPTION_KEY يجب أن يكون سريًا للغاية. يتم تخزينه في متغيرات البيئة (.env) ويُدار بشكل آمن (مثل استخدام Azure Key Vault أو AWS KMS في بيئة الإنتاج).

خلاصة النهج المزدوج: عمود للتجزئة (بحث سريع + منع التكرار + أمان ضد التسريب) وعمود للتشفير (استعادة محدودة بصلاحيات خاصة).

2. مخطط قاعدة البيانات (PostgreSQL)
سنستخدم قاعدة بيانات PostgreSQL لأنها تدعم BYTEA، والفهارس الفريدة، وهي قوية ومستقرة.

جدول Users
sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    full_name VARCHAR(255) NOT NULL,
    national_id_hash VARCHAR(64) UNIQUE NOT NULL,  -- طول 64 حرف عشري (hex)
    national_id_encrypted BYTEA NOT NULL,          -- البيانات المشفرة
    phone VARCHAR(20) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,           -- Bcrypt ينتج 60 حرفًا عادة
    is_admin BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- إنشاء فهارس إضافية لتسريع البحث عند تسجيل الدخول
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_phone ON users(phone);
شرح كل عمود:
id: معرّف داخلي من نوع UUID (أفضل من integer متسلسل للأمان والتوزع). يُستخدم كمفتاح أساسي وفي علاقات الجداول المستقبلية (مثل جدول النتائج).

full_name: اسم المستثل الكامل، نص عادي. ليس حساسًا.

national_id_hash: النص المُجزأ (SHA-256 مع ملح). فريد لضمان عدم تكرار الرقم القومي. لا يمكن عرضه أو استخدامه خارج الخادم.

national_id_encrypted: البيانات المشفرة بصيغة ثنائية. لا يمكن قراءتها مباشرة.

phone: رقم الهاتف، فريد، يستخدم كواحدة من طرق تسجيل الدخول (اختياري).

email: البريد الإلكتروني، فريد، سيُستخدم أساسًا لتسجيل الدخول.

password_hash: كلمة المرور بعد تطبيق خوارزمية Bcrypt (وهي خوارزمية قوية ومقاومة لهجمات القوة الغاشمة لأنها بطيئة عمدًا).

is_admin: علم (boolean) لتحديد ما إذا كان المستخدم لديه صلاحيات إدارية. في البداية يكون false.

created_at: طابع زمني لتسجيل وقت إنشاء الحساب.

ملاحظات إضافية على قواعد البيانات:
استخدام TIMESTAMP WITH TIME ZONE لتجنب مشكلات المناطق الزمنية.

الفهارس الفريدة على national_id_hash، email، phone تمنع التكرار على مستوى قاعدة البيانات، مما يوفر طبقة أمان إضافية بجانب التحقق في التطبيق.

3. نقاط نهاية (Endpoints) API باستخدام FastAPI
سننشئ واجهة برمجية (API) من نوع RESTful. سنستخدم FastAPI لأنه سريع، ويوفر توثيق تلقائي (Swagger)، ويدعم التحقق من الأنواع (Pydantic).

3.1 نقطة التسجيل: POST /api/auth/register
دالة الطلب (Request Payload) باستخدام Pydantic:

python
from pydantic import BaseModel, EmailStr, Field, validator

class UserRegister(BaseModel):
    full_name: str = Field(..., min_length=3, max_length=100)
    email: EmailStr
    phone: str = Field(..., regex=r'^01[0-9]{9}$')  # مثال: أرقام مصر
    national_id: str = Field(..., min_length=14, max_length=14)
    password: str = Field(..., min_length=8)

    @validator('national_id')
    def validate_national_id(cls, v):
        if not v.isdigit():
            raise ValueError('الرقم القومي يجب أن يحتوي على أرقام فقط')
        # هنا يمكن إضافة خوارزمية التحقق من الرقم القومي (checksum) إن وجدت
        return v
التدفق التفصيلي:

استقبال الطلب: يقوم FastAPI بقراءة JSON والتحقق من صحة الحقول بناءً على UserRegister.

التحقق من التنسيق: التأكد من أن الرقم القومي 14 رقمًا، وأن البريد الإلكتروني صحيح، إلخ.

حساب تجزئة الرقم القومي مع الملح:

python
import hashlib
import os
SALT = os.getenv("NATIONAL_ID_SALT").encode()
national_id_bytes = request.national_id.encode()
hash_obj = hashlib.sha256(national_id_bytes + SALT)
national_id_hash = hash_obj.hexdigest()
التحقق من وجود الرقم من قبل:

python
existing_user = await db.fetch_one(
    "SELECT id FROM users WHERE national_id_hash = :hash",
    {"hash": national_id_hash}
)
if existing_user:
    raise HTTPException(status_code=400, detail="هذا الرقم مسجل مسبقاً")
تشفير الرقم القومي (AES-256-GCM): باستخدام مكتبة cryptography:

python
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import base64

key = base64.b64decode(os.getenv("ENCRYPTION_KEY"))
nonce = os.urandom(12)  # 12 bytes for GCM
cipher = AESGCM(key)
encrypted = cipher.encrypt(nonce, request.national_id.encode(), None)
# تخزين nonce + encrypted بشكل مستمر أو منفصل.
# الطريقة الشائعة: national_id_encrypted = nonce + encrypted
national_id_encrypted = nonce + encrypted
تجزئة كلمة المرور باستخدام Bcrypt:

python
import bcrypt
password_hash = bcrypt.hashpw(request.password.encode(), bcrypt.gensalt(rounds=12))
إدراج البيانات في قاعدة البيانات:

python
await db.execute("""
    INSERT INTO users (full_name, national_id_hash, national_id_encrypted,
                       phone, email, password_hash)
    VALUES (:full_name, :hash, :encrypted, :phone, :email, :pwd)
""", values={
    "full_name": request.full_name,
    "hash": national_id_hash,
    "encrypted": national_id_encrypted,  # bytes
    "phone": request.phone,
    "email": request.email,
    "pwd": password_hash.decode()   # bcrypt returns bytes
})
إنشاء وتوقيع JSON Web Tokens (JWT):

Access Token: قصير العمر (مثلاً 30 دقيقة) يحتوي على user_id و is_admin.

Refresh Token: طويل العمر (مثلاً 7 أيام) يُستخدم للحصول على Access Token جديد بدون إعادة تسجيل الدخول.

استخدام مكتبة python-jose (أو PyJWT).

توقيع التوكنات بمفاتيح سرية منفصلة (JWT_SECRET_KEY و JWT_REFRESH_SECRET_KEY).

إرجاع التوكنات للمستخدم:

json
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "token_type": "bearer"
}
3.2 نقطة تسجيل الدخول: POST /api/auth/login
الطريقة الموصى بها: السماح بتسجيل الدخول باستخدام email أو national_id (مع إعلام المستخدم بأنه يمكن استخدام أي منهما). ولكن وفق المخطط، يمكن البدء بالبريد الإلكتروني فقط.

التدفق:

استقبال {"email": "user@example.com", "password": "..."}.

البحث عن المستخدم في قاعدة البيانات باستخدام البريد الإلكتروني:

sql
SELECT id, full_name, password_hash, is_admin FROM users WHERE email = :email
إذا لم يُعثر، إرجاع 401 (Unauthorized) برسالة "بريد إلكتروني أو كلمة مرور غير صحيحة" (رسالة عامة لأمان أعلى).

التحقق من كلمة المرور باستخدام bcrypt:

python
if not bcrypt.checkpw(password.encode(), stored_password_hash.encode()):
    raise HTTPException(status_code=401, detail="بيانات دخول غير صحيحة")
إنشاء Access Token و Refresh Token (كما في التسجيل).

إرجاعهما.

3.3 تحديث التوكن: POST /api/auth/refresh
التدفق:

يتوقع الطلب وجود refresh_token في الجسم (أو في هيدر التفويض).

التحقق من توقيع Refresh Token باستخدام JWT_REFRESH_SECRET_KEY.

استخراج user_id منه.

(اختياري) التحقق من أن هذا الـ Refresh Token لا يزال صالحًا في قاعدة البيانات (إذا أردت تنفيذ قائمة سوداء أو إصدارات).

إنشاء Access Token جديد (بنفس البيانات) وإرجاعه.

لا يتم إصدار Refresh Token جديد في كل مرة (يمكن إضافة دورة حياة منفصلة).

3.4 الحصول على بيانات المستخدم الحالي: GET /api/auth/me
التدفق:

يستخدم هذا المسار اعتماد على Access Token المرسل في هيدر Authorization: Bearer <token>.

يتم فك التوكن، استخراج user_id.

استعلام قاعدة البيانات لجلب البيانات العامة (مع استبعاد national_id_encrypted و national_id_hash):

sql
SELECT id, full_name, email, phone, is_admin, created_at FROM users WHERE id = :user_id
إرجاعها كـ JSON.

4. التنفيذ في الواجهة الأمامية (Next.js)
سنستخدم Next.js مع تطبيق الراوتر الجديد (App Router) لأنه يوفر تحكمًا أفضل في التخزين المؤقت والـ Middleware.

4.1 نموذج التسجيل (/register)
المكتبات المستخدمة: react-hook-form لإدارة النموذج، zod للتحقق من الصحة، zod-resolver لربطها مع react-hook-form.

التحقق من صحة العميل:

typescript
const schema = z.object({
  full_name: z.string().min(3, "الاسم الكامل مطلوب (3 أحرف على الأقل)"),
  email: z.string().email("بريد إلكتروني غير صالح"),
  phone: z.string().regex(/^01[0-9]{9}$/, "رقم الهاتف يجب أن يكون 11 رقماً يبدأ بـ 01"),
  national_id: z.string().length(14, "الرقم القومي يجب أن يكون 14 رقمًا").regex(/^\d+$/, "أرقام فقط"),
  password: z.string().min(8, "كلمة المرور يجب أن تكون 8 أحرف على الأقل"),
  confirm_password: z.string()
}).refine(data => data.password === data.confirm_password, {
  message: "كلمتا المرور غير متطابقتين",
  path: ["confirm_password"]
});
عرض رسائل الخطأ بالعربية: باستخدام خاصية errors من react-hook-form.

إرسال الطلب: استخدام fetch إلى http://localhost:8000/api/auth/register.

تخزين التوكنات: إذا أردت تخزينها في httpOnly cookie، فلا يمكن للجافاسكربت التعامل معها مباشرة. الأفضل أن يقوم الخادم بإرجاع التوكنات في جسم الرد، ويقوم الـ Frontend بتخزينها في localStorage (أقل أمانًا من httpOnly ولكن مقبول للتطوير) أو استخدام مكتبة مثل next-auth (Auth.js) لإدارة الجلسات. وفق المخطط، يمكن تخزينها في localStorage مع مراعاة المخاطر (XSS). للتحسين، يمكن تخزين Access Token في الذاكرة و Refresh Token في httpOnly cookie (ولكن يحتاج إلى تكوين الخادم لإرسالها كـ cookies).

بعد النجاح: إعادة توجيه المستخدم إلى /dashboard أو /exam.

4.2 نموذج تسجيل الدخول (/login)
نموذج أبسط: email و password فقط.

عند الإرسال إلى POST /api/auth/login، استلام التوكنات وتخزينها.

إعادة التوجيه إلى /dashboard.

4.3 سياق المصادقة (Auth Context) و Middleware
Auth Context: يوفر حالة تسجيل الدخول للمكونات (مثل معرفة ما إذا كان المستخدم مسجلاً للدخول وصلاحياته). يمكن استخدام useContext أو zustand.

يتحقق من وجود Access Token في localStorage.

يحاول تحديثه إذا كان منتهيًا باستخدام Refresh Token.

يوفر دوال login, logout, refresh.

Middleware في Next.js (middleware.ts): لحماية المسارات مثل /exam, /lab, /interview.

typescript
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const token = request.cookies.get('access_token')?.value; // إذا استخدمنا cookies
  const isAuthPage = request.nextUrl.pathname.startsWith('/login') || 
                     request.nextUrl.pathname.startsWith('/register');
  
  if (!token && !isAuthPage) {
    return NextResponse.redirect(new URL('/login', request.url));
  }
  
  if (token && isAuthPage) {
    return NextResponse.redirect(new URL('/dashboard', request.url));
  }
  
  return NextResponse.next();
}

export const config = {
  matcher: ['/exam/:path*', '/lab/:path*', '/interview/:path*', '/login', '/register'],
};
ملاحظة مهمة: في هذا المخطط، يتم تخزين JWT في localStorage، لكن middleware لا يمكنه الوصول إلى localStorage (لأنه يعمل على الخادم). لذلك، من الأفضل فعلاً تخزين التوكنات في httpOnly cookies التي يرسلها الخادم بعد التسجيل/تسجيل الدخول (عبر Set-Cookie). هذا أكثر أمانًا من localStorage ويجعل middleware يعمل بسهولة. سيكون تعديل بسيط في API: بدلاً من إرجاع التوكنات في جسم الرد، يتم تعيينها كـ cookies مع علامة HttpOnly و Secure في الإنتاج.

5. الأمان ومتغيرات البيئة
5.1 متغيرات البيئة المطلوبة في ملف .env:
env
# قاعدة البيانات
DATABASE_URL=postgresql://user:pass@localhost:5432/assessments

# JWT
JWT_SECRET_KEY=your_super_secret_jwt_key        # استخدم openssl rand -hex 32
JWT_REFRESH_SECRET_KEY=your_super_secret_refresh_key
ACCESS_TOKEN_EXPIRE_MINUTES=30
REFRESH_TOKEN_EXPIRE_DAYS=7

# التشفير والتجزئة للرقم القومي
ENCRYPTION_KEY=DUMMY_32_BYTE_BASE64_KEY_HERE===   # يجب أن يكون 32 بايت بعد فك base64
NATIONAL_ID_SALT=some_random_salt_for_sha256      # نص عشوائي طويل مثل "fj49f3jf93jf39fj39fj"
5.2 كيفية إنشاء مفتاح تشفير صحيح (32 بايت):
bash
# إنشاء 32 بايت عشوائية وتشفيرها base64
python -c "import secrets, base64; print(base64.b64encode(secrets.token_bytes(32)).decode())"
سيخرج نص مثل "xKj3pQ8rT9zLmN2vW5bC7dF0gH1jK4lP6sA8dF9gH2jK5lQ==" ضعه في ENCRYPTION_KEY.

5.3 الممارسات الأمنية الإضافية:
استخدام HTTPS في الإنتاج لمنع اعتراض البيانات.

حدود معدل الطلبات (Rate Limiting): لمنع هجمات القوة الغاشمة على /login و /register. يمكن استخدام slowapi في FastAPI.

صلاحية Refresh Token: يمكن تخزين نسخة مجزأة من Refresh Token في قاعدة البيانات لإبطالها عند تسجيل الخروج.

تسجيل الخروج: يجب حذف التوكنات من التخزين (ويفضل إضافة endpoint /logout لإبطال Refresh Token).

التحقق من البريد الإلكتروني (Email Verification): لم يُذكر في المخطط، لكن يُنصح به قبل تمكين حساب المستخدم.

عدم تخزين كلمات المرور كـ نص عادي (يتم ذلك باستخدام bcrypt).

5.4 مثال على هيكل المشروع الخلفي (FastAPI):
text
backend/
├── app/
│   ├── main.py
│   ├── config.py          # قراءة متغيرات البيئة
│   ├── database.py        # اتصال PostgreSQL (async باستخدام asyncpg أو SQLAlchemy)
│   ├── models.py          # نماذج Pydantic
│   ├── auth/
│   │   ├── utils.py       # دوال التجزئة والتشفير وإنشاء التوكنات
│   │   ├── dependencies.py # اعتماديات FastAPI (مثلاً get_current_user)
│   │   └── router.py      # نقاط النهاية
│   └── ...
6. اختبار التكامل (اختياري لكن مهم)
اختبار التسجيل: باستخدام curl أو httpx:

bash
curl -X POST http://localhost:8000/api/auth/register \
-H "Content-Type: application/json" \
-d '{"full_name":"محمد أحمد","email":"mohamed@example.com","phone":"01234567890","national_id":"12345678901234","password":"SecurePass123"}'
التحقق من منع التكرار: نفس الطلب مرة أخرى يجب أن يرجع خطأ 400.

تسجيل الدخول: ثم استخدام التوكن للوصول إلى /api/auth/me.

اختبار فك التشفير: (عبر endpoint إداري) التأكد من أن national_id_encrypted يفك بشكل صحيح.

الخلاصة النهائية
هذا المخطط يوفر طبقات متعددة من الأمان:

الرقم القومي محمي بتشفير عكسي (للاسترداد) وتجزئة أحادية الاتجاه (لمنع التكرار).

كلمات المرور محمية بـ bcrypt.

الاتصال عبر JWT مع توكنات محددة العمر.

قاعدة البيانات تطبق قيود التفرد على البريد الإلكتروني، رقم الهاتف، وتجزئة الرقم القومي.

الواجهة الأمامية تستخدم التحقق الصحيح على العميل والخادم، مع حماية المسارات باستخدام middleware.

الشرح أعلاه يغطي كل التفاصيل الدقيقة بدءًا من سبب اختيار كل تقنية وصولاً إلى كيفية كتابة الكود الفعلي وإدارة المفاتيح. إذا أردت التعمق في جزء معين (مثل تنفيذ التشفير أو إعداد middleware مع httpOnly cookies)، يمكنني تقديم شرح إضافي.
