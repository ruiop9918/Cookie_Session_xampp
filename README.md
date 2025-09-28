 # คำสั่ง / ฟังก์ชัน / การตั้งค่า ที่เกี่ยวข้องกับการจัดการ ```Cookie``` และ ```Session``` ใน **PHP** มาให้แบบครบ (พร้อมตัวอย่างโค้ดและคำอธิบายเป็นภาษาไทย) — เหมาะใช้เป็น cheat-sheet เพื่อเอาไปใช้จริงได้ทันที 👍

__ผมจะจัดเป็น 3 ส่วนหลัก:__
- Cookie (สร้าง / อ่าน / ลบ / options)
- Session (เริ่ม / อ่าน/เขียน / เปลี่ยน id / ปิด / เก็บที่อื่น)
- คำสั่ง/ตั้งค่าอื่น ๆ ที่ควรรู้ + ตัวอย่าง flow (login / remember-me / logout) และ best practices ด้านความปลอดภัย

## 1) Cookie (คำสั่งหลัก + ตัวอย่าง)
#### **setcookie()** — สร้างหรืออัพเดต cookie
```php
php 

// แบบเก่า (common)
setcookie("name", "value", time() + 3600, "/", "example.com", true, true);

// แบบใหม่ (PHP >= 7.3) — ส่งเป็น array options สะดวกกว่า
setcookie("name", "value", [
    'expires' => time() + 3600,
    'path' => '/',
    'domain' => 'example.com',
    'secure' => true,
    'httponly' => true,
    'samesite' => 'Lax'   // 'Lax' | 'Strict' | 'None'
]); 
```

- ```name``` — ชื่อ cookie
- ```value``` — ค่า (จะถูก URL-encode อัตโนมัติ ถ้าใช้แบบเก่า)
- ```expires``` — timestamp (หน่วยวินาที) - ถ้าอยากลบให้ใส่เป็นอดีต เช่น ```time() - 3600```
- ```path``` — ขอบเขต path ที่ cookie ใช้ (```/``` คือทั้งโดเมน)
- ```domain``` — กำหนดโดเมน (เช่น ```.example.com``` ให้ครอบ subdomains)
- ```secure``` — ส่ง cookie เฉพาะบน HTTPS (true/false) — ถ้าใช้ ```SameSite=None``` ต้องตั้ง ```secure=true```
- ```httponly``` — ถ้า true จะไม่สามารถอ่าน cookie ด้วย  JavaScript (```document.cookie```) — ช่วยป้องกัน XSS
- ```samesite``` — ค่าป้องกัน cross-site request: 'Lax', 'Strict', 'None'

###### **ข้อสำคัญ**: ต้องเรียก ```setcookie()``` ก่อน มีการส่ง output ใด ๆ ไปยัง browser (เช่น ```echo``` หรือ HTML) — เพราะมันส่ง header ```Set-Cookie```
---
**__setrawcookie()__** — เหมือน **__setcookie()__** แต่ไม่ทำ URL-encoding ค่า
```php
php 

setrawcookie("name", "value;with;chars", time()+3600);

```
- ใช้เมื่อคุณต้องการส่งค่าไม่ถูกแปลง
---
**__อ่านค่า__** cookie
```php
php

if (isset($_COOKIE['name'])) {
    $val = $_COOKIE['name'];
}
```
- **หมายเหตุ**: หลังเรียก ```setcookie()``` แล้ว ค่าใน ```$_COOKIE``` จะยังไม่เปลี่ยนใน request เดียวกัน — browser จะส่ง cookie ตัวใหม่ใน request ต่อไป ถ้าต้องการใช้ทันที ให้ ```$_COOKIE['name'] = 'value';``` เพิ่มเอง
---
**__ลบ cookie (ให้ browser ลบ)__**
```php
php

setcookie("name", "", time() - 3600, "/"); // ย้อนเวลา => หมดอายุ
// และถ้าตั้ง domain/path เดิมไว้ ก็ต้องลบด้วยค่าเดียวกัน
```
---
**__ข้อจำกัด / เกร็ด__**
- ขนาดต่อ cookie ประมาณ ~4KB (รวมชื่อ+ค่า) — อย่าใส่ข้อมูลมาก
- จำนวน cookie ต่อโดเมนมีขีดจำกัดในเบราว์เซอร์แต่ละตัว
- ห้ามเก็บข้อมูลสำคัญ/raw password ใน cookie — ใช้ token ที่สุ่ม แล้วเก็บ hash ใน DB แทน
---

**__2) Session (ฟังก์ชันหลัก + ตัวอย่าง)__**
```session_start()``` — เปิด session (ต้องเรียกก่อนส่ง output)
```php
php

session_start();
```
- ถ้าไม่มี session นี้ จะสร้าง session id ใหม่และส่ง cookie PHPSESSID ให้ browser
---
```$_SESSION``` — เก็บข้อมูล session (superglobal)
```php
php

$_SESSION['user_id'] = 5;
echo $_SESSION['username'];
```
---
```session_status()``` — เช็คสถานะ session
```php
php

if (session_status() !== PHP_SESSION_ACTIVE) {
    session_start();
}
```
---
```session_id()``` — อ่าน/ตั้ง session id ปัจจุบัน
```php
php

$cur = session_id(); // อ่าน
session_id("newid123"); // ตั้ง (ต้องก่อน session_start())
```
---
```session_name()``` — อ่าน/ตั้งชื่อ cookie session (default: PHPSESSID)
```php
php

session_name("MYSESSID"); // ต้องเรียกก่อน session_start()
```
---
```session_regenerate_id([true|false])``` — สร้าง session id ใหม่ (ป้องกัน session fixation)
```php
php

session_regenerate_id(true); // true = ลบ session เก่า (delete old)
```
- ควรเรียกหลัง login สำเร็จ เพื่อป้องกัน fixation
---
```session_unset()``` — ลบตัวแปรใน ```$_SESSION```
```php
php

session_unset(); // เคลียร์ตัวแปร session แต่ยังคง session id อยู่
```
---
```session_destroy()``` — ทำลาย session data และ session id ฝั่ง server
```php
php

session_destroy(); // ลบข้อมูล session ฝั่ง server แต่ไม่ลบ cookie ฝั่ง client อัตโนมัติ
```
- หลัง ```session_destroy()``` แนะนำลบ cookie session ด้วย ```setcookie(session_name(), '', time()-3600, '/');```
---
```session_write_close()``` — เขียน session แล้วปล่อย lock
```php
php

session_write_close();
```
- ใช้เมื่ออยากปล่อย session lock ก่อนส่ง response ยาว ๆ (เช่น หลังบันทึกข้อมูลแล้ว ให้ปล่อยเพื่อให้ request อื่นเข้าถึง session ได้)
---
```session_set_cookie_params()``` — ตั้งค่าพารามิเตอร์ cookie ของ session ก่อน ```session_start()```
```php
php

// แบบเก่า
session_set_cookie_params(0, "/", "example.com", true, true);

// แบบ PHP >= 7.3 (array)
session_set_cookie_params([
  'lifetime' => 0,
  'path' => '/',
  'domain' => 'example.com',
  'secure' => true,
  'httponly' => true,
  'samesite' => 'Lax'
]);

session_start();
```
- ```lifetime``` = เวลา (วินาที) ของ session cookie (0 = จนปิด browser)
---
```session_get_cookie_params()``` — คืนค่า configuration ของ cookie ปัจจุบัน
```php
php

$params = session_get_cookie_params();
print_r($params);
```
---
```session_save_path()``` / ```session_save_handler()``` / ```session_set_save_handler()``` 
— ตั้งที่เก็บ session (เช่น ไฟล์, DB, redis)
- ```session_save_path('/path/to/dir');``` — เปลี่ยน path เก็บไฟล์ session (ต้อง writable)
- ใช้ ```session_set_save_handler()``` เพื่อเก็บ session ใน DB/Redis ออกแบบเอง (advance)
---
```session.gc_maxlifetime``` — ตั้งเวลาที่ garbage collector จะลบ session files (ini setting)
```php
php

ini_set('session.gc_maxlifetime', 1440); // หน่วยวินาที (default 1440 = 24 นาที)
```
- **สำคัญ**: session.cookie_lifetime (cookie ใน browser) กับ gc_maxlifetime (server-side expiration) ต้องสอดคล้องกัน ถ้า cookie ยังหวังแต่ server ลบ data แล้ว cookie จะชี้ไปยัง session ที่ไม่มีจริง
---
```session.use_strict_mode``` — ป้องกัน session fixation
```php
php

ini_set('session.use_strict_mode', 1);
```
- เมื่อเปิด strict mode, หาก session id ที่ browser ขอไม่มีใน server จะถูกปฏิเสธและสร้าง id ใหม่
---
**3) คำสั่ง/ตั้งค่าอื่น ๆ ที่เกี่ยวข้อง (ini options สำคัญ)**
- ```session.cookie_secure``` — หากตั้งเป็น 1 จะส่ง cookie เฉพาะ HTTPS
- ```session.cookie_httponly``` — ถ้า 1 จะไม่ให้ JS อ่าน cookie session
- ```session.cookie_samesite``` — ตั้ง samesite default (PHP 7.3+)
- ```session.cookie_lifetime``` — ค่า default ของ session cookie lifetime
- ```session.save_path``` — พาธไฟล์ session (ถ้าเก็บเป็นไฟล์)
- ```session.gc_probability``` และ session.gc_divisor — ควบคุมความน่าจะเป็นการ trigger GC

ตัวอย่างการตั้งค่า (run-time):
```php
php

ini_set('session.cookie_secure', 1);
ini_set('session.cookie_httponly', 1);
ini_set('session.cookie_samesite', 'Lax');
ini_set('session.gc_maxlifetime', 86400); // 1 วัน
```
---
### 4) Flow ตัวอย่าง: Login + Remember-me (แบบปลอดภัย) — โค้ดตัวอย่างสั้น ๆ
__(A) Login (เซฟ token แบบปลอดภัยใน DB และ cookie)__
```php
php

// หลังตรวจ password สำเร็จ
session_start();
$_SESSION['user_id'] = $user_id;
session_regenerate_id(true); // ป้องกัน fixation

if ($remember) {
    // สร้าง token (raw) แล้วเก็บ hash ใน DB
    $token = bin2hex(random_bytes(32));
    $tokenHash = password_hash($token, PASSWORD_DEFAULT);

    // เก็บ hash ลง DB: UPDATE users SET remember_token=? WHERE id=?
    // ส่ง cookie ให้ client (HttpOnly, Secure, samesite)
    setcookie('remember_token', $token, [
        'expires' => time() + 60*60*24*30,
        'path' => '/',
        'secure' => true,
        'httponly' => true,
        'samesite' => 'Lax'
    ]);
    setcookie('user_id', $user_id, time() + 60*60*24*30, '/', 'example.com', true, true);
}
```
__(B) Auto-login จาก cookie (ที่ ```index.php``` ก่อน redirect)__
```php
php

session_start();
if (!isset($_SESSION['user_id']) && isset($_COOKIE['remember_token'], $_COOKIE['user_id'])) {
    $user_id = (int) $_COOKIE['user_id'];
    $token = $_COOKIE['remember_token'];

    // ดึง remember_token hash จาก DB
    // ถ้า password_verify($token, $db_hash) === true -> valid
    // ให้ set $_SESSION และ session_regenerate_id()
}
```
__(C) Logout (ลบ session, cookie และ token ใน DB)__
```php
php

session_start();
// เก็บ user_id ก่อนทำลาย session
$user_id = $_SESSION['user_id'] ?? null;

// ลบ session
session_unset();
session_destroy();

// ลบ cookie ใน browser
setcookie('remember_token', '', time() - 3600, '/');
setcookie('user_id', '', time() - 3600, '/');

// ลบ token ใน DB โดยใช้ $user_id (หากไม่ null)
if ($user_id) {
    $stmt = $conn->prepare("UPDATE users SET remember_token = NULL WHERE id = ?");
    $stmt->bind_param("i", $user_id);
    $stmt->execute();
}
```
---
### 5) Best practices & Pitfalls (ที่ควรรู้จริง ๆ)

- อย่าเก็บ password เป็น plaintext — ต้องใช้ ```password_hash()``` และตรวจด้วย ```password_verify()```
- อย่าเก็บ password hash ที่ client สร้างมา — server ต้องเป็นผู้ hash เสมอ
- ใช้ HTTPS เสมอ (ตั้ง ```secure => true```) — ถ้าไม่ใช้ HTTPS cookie ที่มี sensitive info เสี่ยงมาก
- ตั้ง ```httponly => true``` สำหรับ cookie token/session ที่ไม่ต้องให้ JS เข้าถึง
- ใช้ SameSite='Lax' (หรือ 'Strict' ถ้าต้องการ) เพื่อป้องกัน CSRF ในระดับหนึ่ง — ถ้ต้องการ cross-site POST, ต้องวางแผนดี (และ ```SameSite=None``` ต้อง ```secure=true```)
- regenerate session id หลัง ```login (session_regenerate_id(true)```) เพื่อป้องกัน session fixation
- ลบ token ทั้งฝั่ง client และ DB ใน logout — มิฉะนั้น token ยังคง valid
- ระวัง headers already sent — เรียก ```session_start()``` และ ```setcookie()```ก่อนส่ง output
- Cookie จะมีผลใน request ถัดไป — ถ้ตั้งแล้วต้องใช้ทันที ให้อัปเดต ```$_COOKIE ```เอง
- อย่าพึ่งพา cookie เพียงอย่างเดียวสำหรับ authorization — ใช้ session/DB validation เสมอ
- หากต้องการ session ยาว ๆ ให้ใช้ remember-me token แทน การขยาย ```gc_maxlifetime``` บางครั้งยังไม่เพียงพอ (server GC อาจช้าหรือถูกตั้งค่าต่างกันในหลาย server)
---
### 6) ตัวอย่าง cheat-sheet ย่อ (คำสั่งที่ใช้บ่อยสุด)
```php
php

// Cookie
setcookie("name","value", time()+3600, "/", "example.com", true, true);
setrawcookie("name","value"); // ไม่ encode
$_COOKIE['name']; // อ่าน
setcookie("name","", time()-3600, "/"); // ลบ

// Session
session_start();
$_SESSION['user_id'] = 1;
session_regenerate_id(true);
session_unset();
session_destroy();
session_set_cookie_params(['lifetime'=>0,'path'=>'/','secure'=>true,'httponly'=>true,'samesite'=>'Lax'])
```
---
# ตัวอย่าง config สำหรับ php.ini ที่ใช้งานจริงได้เลย (เหมาะกับเว็บที่มีระบบ login / session / cookie)
### ตำแหน่งไฟล์ ```php.ini```
```makefile
makefile

ถ้าใช้ XAMPP: ไฟล์อยู่ที่

C:\xampp\php\php.ini
```
__หรือ__
```bash
bash

/etc/php.ini
```

---
## php.ini ตัวอย่าง (เหมาะกับทุกโปรเจค)
```ini
ini

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;  PHP.INI - CONFIG สำหรับ PROJECT ทั่วไป
;  ปรับค่าแล้ว restart Apache/XAMPP/NGINX
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

; =========[ ERROR HANDLING ]=========
display_errors = Off       ; ❌ ไม่ให้ error โผล่บนหน้าเว็บ
log_errors = On            ; ✅ ให้เขียน error log
error_log = "C:\xampp\php\logs\php_error.log" ; ตำแหน่งไฟล์ log (เปลี่ยนได้)

; =========[ TIMEZONE ]=========
date.timezone = Asia/Bangkok   ; ตั้ง timezone

; =========[ UPLOAD/POST ]=========
upload_max_filesize = 10M      ; ✅ ขนาดไฟล์อัปโหลดสูงสุด (เปลี่ยนได้)
post_max_size = 12M            ; ควรมากกว่า upload_max_filesize เล็กน้อย

; =========[ MEMORY & EXECUTION ]=========
memory_limit = 256M            ; ✅ หน่วยความจำสูงสุดที่ PHP ใช้ได้
max_execution_time = 30        ; ✅ เวลาสูงสุดของการรันสคริปต์ (วินาที)

; =========[ SESSION ]=========
session.gc_maxlifetime = 1440  ; อายุ session (วินาที) → 1440 = 24 นาที
session.cookie_httponly = 1    ; ป้องกัน JavaScript อ่าน session cookie
session.cookie_secure = 0      ; ตั้งเป็น 1 ถ้าใช้ HTTPS
session.use_strict_mode = 1    ; ป้องกัน session hijacking

; =========[ OPCACHE (ทำให้เว็บเร็วขึ้น) ]=========
opcache.enable = 1
opcache.enable_cli = 1
opcache.memory_consumption = 128
opcache.interned_strings_buffer = 8
opcache.max_accelerated_files = 10000
opcache.validate_timestamps = 1
opcache.revalidate_freq = 2
```
---
## 📍 จุดที่ควรเปลี่ยนบ่อย ๆ

1. ```upload_max_filesize``` และ ```post_max_size```
→ ถ้าโปรเจคมีอัปโหลดไฟล์ใหญ่ ต้องเพิ่ม (เช่น 100M)

2. ```memory_limit```
→ ถ้าใช้ Laravel, WordPress, หรือเว็บที่มี process หนัก ให้เพิ่มเป็น 512M

3. ```max_execution_time```
→ งานทั่วไป 30 วินาทีพอ, งานประมวลผลหนัก เช่น import Excel ตั้ง 300

4. ```session.gc_maxlifetime```
→ ถ้าอยากให้ login ค้างนานขึ้น ตั้งเป็น 86400 (1 วัน) หรือ 604800 (7 วัน)

5. ```error_log```
→ กำหนด path ไฟล์ log ให้ง่ายต่อการตรวจสอบ (เช่น storage/logs/php_error.log)
---
### วิธีสร้างไฟล์ ```php_error.log``` ใน XAMPP

1. ไปที่โฟลเดอร์ ```C:\xampp\php\```
2. ถ้ายังไม่มีโฟลเดอร์ ```logs``` → ให้สร้างโฟลเดอร์ใหม่ชื่อ ```logs```
    - คลิกขวา > New > Folder > ตั้งชื่อ logs
3. เข้าไปในโฟลเดอร์ ```logs```
4. สร้างไฟล์เปล่า ๆ ชื่อ ```php_error.log```
    - คลิกขวา > New > Text Document
    - ตั้งชื่อ ```php_error.log``` (เปลี่ยน ```.txt``` เป็น ```.log```)
    - ถ้า Windows ไม่ยอมให้เปลี่ยนนามสกุล → ไปที่ View > File name extensions แล้วติ๊ก ✔ ก่อน
5. เปิด ```php.ini``` (ปกติอยู่ที่ ```C:\xampp\php\php.ini```)
6. ค้นหาคำว่า ```error_log``` แล้วแก้เป็น:
```ini
ini

log_errors = On
error_log = "C:\xampp\php\logs\php_error.log"
```
7. บันทึกไฟล์ ```php.ini```
8. Restart Apache ผ่าน XAMPP Control Panel
---
### 📌 วิธีเช็คว่า log ทำงานจริง

สร้างไฟล์ ```test_error.php``` ใน ```htdocs:```
```php
php

<?php
// จงใจเขียนผิดให้เกิด error
echo $undefinedVar;

?>
```
__เปิดใน browser → หน้าเว็บอาจว่าง แต่ไปเปิด ```C:\xampp\php\logs\php_error.log```
คุณจะเห็นบรรทัด error ประมาณนี้:__

```php
[28-Sep-2025 10:25:34 Asia/Bangkok] PHP Notice:  Undefined variable: undefinedVar in C:\xampp\htdocs\test_error.php on line 3
```
---
## 📌 ตำแหน่งไฟล์ php.ini

- Windows + XAMPP → ```C:\xampp\php\php.ini```
- Linux/Ubuntu (Apache) → ```/etc/php/8.x/apache2/php.ini```
- Linux/Ubuntu (CLI) → ```/etc/php/8.x/cli/php.ini```
- หลังแก้ไฟล์แล้วต้อง ```restart Apache``` หรือ ```restart php-fpm (ถ้าใช้ Nginx)```
---
### 2 เวอร์ชัน ของไฟล์ php.ini ให้คุณเก็บไว้ใช้ได้เลย

 __🟢 1. Dev Mode (สำหรับเขียนโค้ด / Debug)__
- ไฟล์: php_dev.ini
```ini
ini

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; PHP.INI - DEV MODE (สำหรับนักพัฒนา)
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

; ==== ERROR HANDLING ====
display_errors = On          ; ✅ แสดง error บนหน้าเว็บ
display_startup_errors = On
log_errors = On              ; ✅ เขียน error ลง log
error_reporting = E_ALL   ;แสดง error ทุกชนิด ที่ PHP รู้จัก
error_log = "C:\xampp\php\logs\php_error.log"

; ==== TIMEZONE ====
date.timezone = Asia/Bangkok

; ==== FILE UPLOAD ====
file_uploads = On
upload_max_filesize = 50M
post_max_size = 60M
max_input_time = 300
max_execution_time = 300
memory_limit = 512M

; ==== SESSION ====
session.gc_maxlifetime = 86400   ; 1 วัน
session.cookie_httponly = 1
session.cookie_secure = 0   ; ถ้าใช้ HTTPS ให้เปลี่ยนเป็น 1
session.use_strict_mode = 0
session.cookie_lifetime = 0
session.save_path = "C:\xampp\tmp"

; ==== OPCACHE ====
opcache.enable = 0               ; ปิด opcache (เพราะ debug ต้อง refresh code ตลอด)
```
---
 __🔵 2. Production Mode (สำหรับเว็บจริง)__
- ไฟล์: php_prod.ini
```ini
ini

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; PHP.INI - PRODUCTION MODE (สำหรับเซิร์ฟเวอร์จริง)
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

; ==== ERROR HANDLING ====
display_errors = Off         ; ❌ ไม่โชว์ error บนเว็บ (ปลอดภัย)
display_startup_errors = Off
log_errors = On              ; ✅ เก็บ error log
error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT
error_log = "C:\xampp\php\logs\php_error.log"

; ==== TIMEZONE ====
date.timezone = Asia/Bangkok

; ==== FILE UPLOAD ====
file_uploads = On
upload_max_filesize = 100M
post_max_size = 120M
max_input_time = 300
max_execution_time = 300
memory_limit = 1024M

; ==== SESSION ====
session.gc_maxlifetime = 86400   ; 1 วัน
session.cookie_httponly = 1
session.cookie_secure = 1        ; ✅ ต้องใช้ HTTPS
session.use_strict_mode = 1
session.cookie_lifetime = 0
session.cookie_samesite = Lax   ; ป้องกัน CSRF (ปรับเป็น Strict ได้ถ้าเว็บไม่ embed)
session.save_path = "C:\xampp\tmp"
session.name = PHPSESSID        ; ชื่อ session cookie
; ค่าอื่น ๆ ใช้จาก setcookie() ในโค้ด

opcache.enable = 1
opcache.enable_cli = 1
opcache.memory_consumption = 256
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 20000
opcache.validate_timestamps = 0  ; ✅ เพื่อความเร็ว (ต้อง restart server เวลาอัปเดตโค้ด)
```
---
## มาดูความหมายของ 2 คำสั่ง
**__```error_reporting = E_ALL``` and ```error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT``` แบบละเอียดกันเลย 👇__**

__1. ```error_reporting = E_ALL```__
- หมายถึงให้ __แสดง__ error __ทุกชนิด__ ที่ PHP รู้จัก
- รวมถึง warning, notice, deprecated, strict, fatal error
- ใช้บ่อยในช่วง __พัฒนา (development)__ เพราะจะเห็นปัญหาทุกอย่าง

__2. ```error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT```__
- ```E_ALL``` = แสดง error ทั้งหมด
- ```~E_DEPRECATED``` = ไม่แสดง error ที่เกี่ยวกับฟังก์ชัน deprecated (ฟังก์ชันที่ PHP เตือนว่าต่อไปจะเลิกใช้)
- ```~E_STRICT``` = ไม่แสดง error ที่เกี่ยวกับ strict standards (มาตรฐานโค้ดที่เข้มงวดมาก ๆ เช่น syntax เก่าๆ)

📌 ผลลัพธ์ = จะแสดง error ทุกชนิด __ยกเว้น__ Deprecated และ Strict

---
## 🔎 ใช้ตอนไหน
- ```error_reporting = E_ALL``` → ใช้ตอนพัฒนา เพราะอยากเห็นทุก warning/notice
- ```error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT``` → ใช้ตอน production จริง เพราะบาง warning ไม่ควรให้ user เห็น (เช่น deprecated)
---
## ✅ ตัวอย่างผลต่าง

__สมมติคุณใช้ฟังก์ชันเก่าเช่น ```mysql_connect()``` (ที่ deprecated ใน PHP 7+)__
- ถ้าใช้ ```E_ALL``` → จะมี warning ว่า mysql_connect() is deprecated
- ถ้าใช้ ```E_ALL & ~E_DEPRECATED & ~E_STRICT``` → warning นี้จะไม่แสดง

__👉 สรุปง่าย ๆ__
- __Dev__: ใช้ ```E_ALL``` (เห็นหมด เพื่อแก้ไขโค้ดได้ครบ)
- __Prod__: ใช้ ```E_ALL & ~E_DEPRECATED & ~E_STRICT``` (แสดงเฉพาะ error สำคัญ ไม่รบกวนผู้ใช้)

---
### 📌 วิธีใช้ ร่วมกับ xampp ตำแหน่งไฟล์  ```C:\xampp\php\```

__สรุปให้สั้น ๆ แบบเข้าใจง่าย:__
- PHP ใช้ไฟล์ ```php.ini``` ตัวเดียวเท่านั้น
- ไฟล์ ```php.ini-development``` และ ```php.ini-production``` เป็น template ที่ PHP แถมมาให้
- คุณจะ ลบไฟล์ ```php.ini``` เดิมทิ้งไปได้เลย (ไม่กระทบ)
- เวลาใช้งานจริง แค่เลือก template ที่ต้องการ แล้ว เปลี่ยนชื่อเป็น ```php.ini```
  - เช่น อยากใช้สำหรับพัฒนา → rename ```php.ini-development``` → ```php.ini```
  - อยากใช้สำหรับ production → rename ```php.ini-production``` → ```php.ini```
---
### ✅ เทคนิคที่แนะนำ
__แทนที่จะลบ php.ini ออกไปเลย ผมแนะนำว่า:__
1. เก็บ ```php.ini``` เดิมไว้ → เปลี่ยนชื่อเป็น ```php.ini.backup``` (กันพลาด)
2. เวลาอยากเปลี่ยนค่า →
    - Copy ```php.ini-development``` → ตั้งชื่อ php.ini
    - หรือ Copy ```php.ini-production``` → ตั้งชื่อ php.ini

__แบบนี้คุณจะมีไฟล์สำรองเสมอ เผื่ออยากย้อนกลับ__

---

### วิธีใช้งานจริง (PHP code)
- เมื่อ config เสร็จแล้ว เวลาเขียน PHP ก็ใช้ง่าย ๆ แค่นี้:
```php
php

<?php
session_start(); // ต้องมีทุกหน้า ที่ใช้ session

// สร้าง session หลัง login
$_SESSION['user_id'] = 123;
$_SESSION['username'] = "tangsin";

// อ่าน session
echo "Hello, " . $_SESSION['username'];

// logout (ลบทั้งหมด)
session_unset();
session_destroy();
setcookie(session_name(), '', time() - 3600, '/');

?>
```
---
# สรุป
- **ใช้จริง → แค่ ```session_start()``` บนหน้า PHP ทุกหน้าที่ต้องการใช้ session**
