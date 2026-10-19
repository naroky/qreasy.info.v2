# QREASY V2 Wireframe

Wireframe ของ QREASY.INFO V2 ทุกหน้าจอ ทั้งฝั่งผู้ใช้และ CMS

- ดูและแก้ไขบน canvas: <https://claude.ai/artifact/MJTJYGcyKsXZ6eSWC44UQK>
- ไฟล์ในโฟลเดอร์นี้เป็นสำเนาของ canvas ณ วันที่ 19 ต.ค. 2026 ถ้าแก้บน canvas ภายหลัง ต้อง copy มาเก็บใหม่

ไฟล์ `.dc.html` เป็นรูปแบบของ canvas ใช้สคริปต์ `support.js` ของ canvas ในการแสดงผล ถ้าเปิดตรงจากเครื่องจะเห็นแค่โครง HTML ไม่ครบ ให้ดูผ่านลิงก์ด้านบน ส่วน `canvas.json` เก็บตำแหน่งและชื่อของแต่ละหน้าจอ

## ฝั่งผู้ใช้

| ไฟล์ | หน้าจอ |
| --- | --- |
| `Main.dc.html` | หน้าแรก Fast QR (เดสก์ท็อป) |
| `Mobile.dc.html` | หน้าแรก (มือถือ) |
| `MobileCustomize.dc.html` | เปิดส่วนปรับแต่ง (มือถือ) |
| `MobileAfterDownload.dc.html` | หลังดาวน์โหลด พร้อมการ์ด Affiliate (มือถือ) |
| `MoreTypes.dc.html` | ประเภทอื่น ตัวอย่างฟอร์ม Wi-Fi (มือถือ) |
| `CookieConsent.dc.html` | ขอความยินยอมคุกกี้ (มือถือ) |
| `Login.dc.html` | เข้าสู่ระบบ |
| `Register.dc.html` | สมัครสมาชิก |
| `VerifyEmail.dc.html` | ยืนยันอีเมล |
| `Account.dc.html` | บัญชีของฉัน |
| `MyQr.dc.html` | QR ของฉัน (เดสก์ท็อป) |
| `Faq.dc.html` | หน้าเนื้อหา FAQ |

## CMS

| ไฟล์ | หน้าจอ |
| --- | --- |
| `AdminTwoFA.dc.html` | ยืนยันตัวตนสองชั้น |
| `AdminOverview.dc.html` | ภาพรวม |
| `AdminUsers.dc.html` | จัดการผู้ใช้ |
| `AdminAds.dc.html` | โฆษณาและ GA |
| `AdminAffiliate.dc.html` | Affiliate |
| `AdminAudit.dc.html` | Audit Log |
