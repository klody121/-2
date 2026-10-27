# الأسبوع الثاني — نظام إدارة تأجير المعدات والأجهزة

**إعداد: مشعل حسن المحمدي**

المهمة: صلاحيات الأدوار (مالك، مستأجر، مدير)، وعرض طلبات المالك وقبولها أو رفضها.

يفترض هذا التنفيذ أن المقصود بطلبات التنسيق هو طلبات التأجير، وأن ملفات PHP في مجلد واحد بجانب `db.php` من الأسبوع الأول، وأن تسجيل الدخول يحفظ `user_id` في الجلسة.

| الدور | الصلاحية |
|---|---|
| مالك owner | عرض طلبات معداته والموافقة عليها أو رفضها |
| مستأجر renter | الوصول إلى صفحات المستأجر |
| مدير admin | الوصول إلى صفحات إدارة النظام |

## 1. قاعدة البيانات

نفّذ التعديل مرة واحدة داخل قاعدة `equipment_rental`:

```sql
USE equipment_rental;

ALTER TABLE users
ADD role ENUM('owner', 'renter', 'admin')
NOT NULL DEFAULT 'renter';

CREATE TABLE rental_requests (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    owner_id INT UNSIGNED NOT NULL,
    renter_id INT UNSIGNED NOT NULL,
    equipment_id INT UNSIGNED NOT NULL,
    status ENUM('pending', 'approved', 'rejected')
        NOT NULL DEFAULT 'pending',
    FOREIGN KEY (owner_id) REFERENCES users(id),
    FOREIGN KEY (renter_id) REFERENCES users(id)
);
```

إذا كان جدول الطلبات موجودًا لدى الفريق، طابق أسماءه وأعمدته بدل إنشاء جدول مكرر. `equipment_id` يطابق معرّف المعدة عند الفريق. عند إنشاء الطلب يجب استخراج المالك من سجل المعدة في قاعدة البيانات، والمستأجر من الجلسة، وعدم الوثوق بمعرّف مالك يرسله النموذج.

الحساب الجديد مستأجر افتراضيًا. تعيين المالك والمدير يكون من جهة موثوقة، وليس من نموذج التسجيل العام.

## 2. التحقق من الصلاحيات — auth.php

```php
<?php
session_start();
require_once __DIR__ . '/db.php';

if (!isset($_SESSION['user_id'])) {
    header('Location: login.php');
    exit;
}

$stmt = $conn->prepare('SELECT role FROM users WHERE id = ?');
$stmt->bind_param('i', $_SESSION['user_id']);
$stmt->execute();
$stmt->bind_result($role);

if (!$stmt->fetch()) {
    $_SESSION = [];
    session_destroy();
    exit('الحساب غير موجود');
}
$stmt->close();

function requireRole(string $required): void {
    global $role;
    if ($role !== $required) {
        http_response_code(403);
        exit('ليس لديك صلاحية');
    }
}

$_SESSION['csrf'] ??= bin2hex(random_bytes(32));
```

في بداية صفحة المستأجر قبل أي HTML:

```php
<?php
require_once __DIR__ . '/auth.php';
requireRole('renter');
```

لصفحة المدير استخدم `requireRole('admin')`، ولصفحة المالك استخدم `requireRole('owner')`. لا تضف `session_start()` مرة أخرى في الصفحات التي تستدعي `auth.php`.

## 3. عرض طلبات المالك — owner_requests.php

```php
<?php
require_once __DIR__ . '/auth.php';
requireRole('owner');

$stmt = $conn->prepare(
    "SELECT id, equipment_id FROM rental_requests
     WHERE owner_id = ? AND status = 'pending'"
);
$stmt->bind_param('i', $_SESSION['user_id']);
$stmt->execute();
$stmt->bind_result($id, $equipment);

while ($stmt->fetch()):
?>
<form action="process_request.php" method="post">
    <p>طلب <?= (int) $id ?> — المعدة <?= (int) $equipment ?></p>
    <input type="hidden" name="id" value="<?= (int) $id ?>">
    <input type="hidden" name="csrf"
           value="<?= htmlspecialchars($_SESSION['csrf'], ENT_QUOTES) ?>">
    <button name="decision" value="approved">موافقة</button>
    <button name="decision" value="rejected">رفض</button>
</form>
<?php endwhile; ?>
```

## 4. معالجة الموافقة والرفض — process_request.php

```php
<?php
require_once __DIR__ . '/auth.php';
requireRole('owner');

if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    http_response_code(405);
    exit;
}

$token = $_POST['csrf'] ?? '';
if (!is_string($token) ||
    !hash_equals($_SESSION['csrf'], $token)) {
    http_response_code(403);
    exit('طلب غير صالح');
}

$id = filter_input(INPUT_POST, 'id', FILTER_VALIDATE_INT);
$decision = $_POST['decision'] ?? '';

if (!$id || $id < 1 ||
    !in_array($decision, ['approved', 'rejected'], true)) {
    http_response_code(400);
    exit('بيانات غير صحيحة');
}

$stmt = $conn->prepare(
    "UPDATE rental_requests SET status = ?
     WHERE id = ? AND owner_id = ? AND status = 'pending'"
);
$stmt->bind_param('sii', $decision, $id, $_SESSION['user_id']);
$stmt->execute();

echo $stmt->affected_rows === 1
    ? 'تم تحديث الطلب'
    : 'الطلب غير متاح أو تمت معالجته';
```

## التحقق عند الدمج

- المستأجر والمدير لا يستطيعان فتح معالجة موافقات المالك.
- المالك لا يستطيع تعديل طلب يخص مالكًا آخر.
- الطلب المعالج لا يقبل معالجة ثانية.
- الطلب بدون رمز الحماية الصحيح يرفض.

هذه الأكواد تغيّر حالة الطلب فقط. التحقق من توفر المعدة ومواعيد الحجز وربط الطلب بجدول المعدات يحتاج مخطط جداول الفريق. لم تُختبر على قاعدة بيانات المشروع.

## رفع الملف

في مستودع GitHub اختر **Add file → Upload files**، وارفع هذا الملف ثم اضغط **Commit changes**. الملف توثيق يحتوي الأكواد؛ لتشغيلها انسخ كل مقطع PHP إلى الملف المحدد باسمه أعلاه.
