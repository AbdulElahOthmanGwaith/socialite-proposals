# دليل تقني للمساهمة في تطوير Laravel Socialite

هذا الدليل يقدم إطاراً تقنياً للمساهمة في تطوير حزمة Laravel Socialite، مع التركيز على تنفيذ المقترحات الرئيسية التي تم تحديدها.

## 1. تنفيذ دعم OpenID Connect (OIDC)

الهدف هو دمج دعم OIDC كفئة مزود أساسية (Core Provider) ضمن Socialite، مما يقلل من الاعتماد على الحلول المجتمعية.

### الخطوات المقترحة:

1.  **إنشاء `OidcProvider`:**
    *   إنشاء فئة `\Laravel\Socialite\Two\OidcProvider` ترث من `AbstractProvider`.
    *   تعديل طريقة `getAuthUrl()` لضمان تضمين النطاق (`scope`) `openid` دائماً.
    *   تعديل طريقة `getTokenUrl()` لطلب `id_token` بالإضافة إلى `access_token`.
2.  **معالجة رمز الهوية (ID Token):**
    *   تطوير طريقة خاصة (مثل `validateAndDecodeIdToken()`) للتحقق من صحة رمز الهوية باستخدام مكتبة JWT (مثل `firebase/php-jwt` المستخدمة بالفعل في Laravel).
    *   يجب أن تتضمن عملية التحقق:
        *   التحقق من التوقيع (Signature) باستخدام مفاتيح المزود العامة (JWKS).
        *   التحقق من المُصدر (`iss`) والجمهور (`aud`).
        *   التحقق من انتهاء الصلاحية (`exp`) والوقت قبل الاستخدام (`nbf`).
3.  **تحديث `mapUserToObject()`:**
    *   تعديل طريقة `mapUserToObject()` لاستخدام المطالبات (Claims) المستخرجة من رمز الهوية (ID Token) لملء بيانات المستخدم (مثل `id` و `email` و `name`) بدلاً من الاعتماد الكلي على استجابة نقطة نهاية المستخدم (`user endpoint`).

## 2. تحسين أدوات الاختبار (Testing Fakes)

الهدف هو جعل `Socialite::fake()` أكثر مرونة وقوة لدعم سيناريوهات الاختبار المعقدة.

### الخطوات المقترحة:

1.  **تعديل `SocialiteManager::fake()`:**
    *   تعديل توقيع الطريقة ليقبل مصفوفة أو دالة (Callback) لتحديد بيانات المستخدم الوهمية لكل مزود.
    *   استبدال Singleton الحالي لـ `\Laravel\Socialite\Contracts\Factory` بـ `FakeSocialiteManager` جديد.
2.  **إنشاء `FakeSocialiteManager`:**
    *   هذه الفئة ستكون مسؤولة عن اعتراض طلبات `driver()` وإرجاع نسخة من `FakeProvider` مهيأة ببيانات المستخدم المخصصة.
3.  **تحديث `FakeProvider`:**
    *   تعديل `FakeProvider` لقبول كائن `User` مهيأ مسبقاً في البناء (Constructor).
    *   تعديل طريقة `user()` في `FakeProvider` لإرجاع كائن `User` الذي تم تمريره بدلاً من إنشاء كائن افتراضي.

## 3. تحديث العقود (Contracts) والتحليل الساكن (Static Analysis)

الهدف هو تحسين التوافق مع أدوات التحليل الساكن مثل PHPStan و Psalm.

### الخطوات المقترحة:

1.  **تحديث `\Laravel\Socialite\Contracts\Provider`:**
    *   إضافة تعريفات للطرق الشائعة التي يتم استخدامها بشكل متكرر في المزودين، مثل:
        *   `public function stateless(): self;`
        *   `public function scopes(array $scopes): self;`
        *   `public function with(array $parameters): self;`
    *   هذا يضمن أن أي تطبيق يوسع المزودين لن يواجه أخطاء "Method not found" من أدوات التحليل الساكن.
2.  **تحسين التوثيق البرمجي (DocBlocks):**
    *   مراجعة شاملة للـ DocBlocks في جميع الفئات الأساسية لضمان دقة أنواع البيانات (Type Hints) والإرجاع (Return Types).

## 4. تبسيط إعدادات إعادة التوجيه

لحل مشكلة إجبار المطور على تحديد `redirect` في ملف الإعدادات حتى عند استخدام `redirectUrl()` برمجياً.

### الخطوات المقترحة:

1.  **تعديل `AbstractProvider::getRedirectUrl()`:**
    *   تعديل المنطق داخل هذه الطريقة لإعطاء الأولوية لـ `redirectUrl` الذي تم تعيينه برمجياً عبر طريقة `setRedirectUrl()`، وتجاهل الإعدادات في ملف `config/services.php` إذا تم تعيينه برمجياً.
    *   يمكن إزالة التحقق من وجود المفتاح في الإعدادات إذا تم تعيينه برمجياً، أو جعل التحقق أكثر تساهلاً.

## الخلاصة

هذه الخطوات توفر خارطة طريق واضحة للمساهمين الراغبين في تطوير Socialite. إن التركيز على هذه النقاط سيضمن أن Socialite يظل حلاً حديثاً، آمناً، ومرناً للمصادقة الاجتماعية في Laravel.
