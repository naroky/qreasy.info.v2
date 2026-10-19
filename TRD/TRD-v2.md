# QREASY.INFO Version 2 — Technical Requirements Document (TRD)

| รายการ | รายละเอียด |
| --- | --- |
| เอกสาร | Technical Requirements Document — QREASY.INFO V2 |
| สถานะ | Draft 0.4 (รอทีมพัฒนาและเจ้าของผลิตภัณฑ์ทบทวน) |
| วันที่ | 19 ตุลาคม 2026 |
| อ้างอิงข้อกำหนดธุรกิจ | [BRD/BRD-v2.md](../BRD/BRD-v2.md) |
| ข้อมูลระบบเดิม | [v1/oldfeaturev1.md](../v1/oldfeaturev1.md) |

### ประวัติการแก้ไข

| เวอร์ชัน | วันที่ | รายละเอียด |
| --- | --- | --- |
| 0.1 | 19 ต.ค. 2026 | ร่างแรก สถาปัตยกรรมแบบ static สร้าง QR บนเบราว์เซอร์ |
| 0.2 | 19 ต.ค. 2026 | เพิ่ม backend, ฐานข้อมูล, ระบบยืนยันตัวตน, CMS, การจัดการ Ads/GA ผ่าน CMS, data model และ API |
| 0.3 | 19 ต.ค. 2026 | เพิ่ม Affiliate (click tracking), หลายเครือข่ายโฆษณาและ A/B test, แบนเนอร์ผู้สนับสนุน และข้อกำหนด Phase 2 (Dynamic QR, สถิติการสแกน, subscription และ payment) |
| 0.4 | 19 ต.ค. 2026 | V2 เป็นบริการฟรี รายได้จากพื้นที่โฆษณาเท่านั้น ส่วนที่เป็น Phase 2 (หัวข้อ 4.13, ตาราง/API/event/NFR ที่ระบุ Phase 2) พักไว้และไม่อยู่ในขอบเขตการพัฒนา |

> **ขอบเขตการพัฒนาปัจจุบัน:** ทุกข้อที่ระบุว่า "Phase 2" ในเอกสารนี้**พักไว้** ทีมพัฒนาไม่ต้องทำ เก็บไว้เพื่ออ้างอิงเท่านั้น (ดู BRD หัวข้อ 3.2 และ 6.12)

> ข้อกำหนดแต่ละข้อมีรหัส `TR-xx` และอ้างอิงกลับไปยัง `BR-xx` ใน BRD ส่วนเทคโนโลยีในหัวข้อที่ 3 เป็นข้อเสนอที่ยังรอการยืนยัน

## 1. ภาพรวมสถาปัตยกรรม

ระบบแบ่งเป็น 3 ส่วน

1. **QR Generator (สาธารณะ):** สร้าง QR Code บนเบราว์เซอร์ทั้งหมด Guest ใช้งานได้โดยไม่ต้องส่งข้อมูลไปเซิร์ฟเวอร์และไม่ต้องผ่าน CAPTCHA (BR-07, BR-08, BR-43)
2. **Member Area:** สมัคร/Login, โปรไฟล์ และ "QR Code ของฉัน" ข้อมูลไปถึงเซิร์ฟเวอร์เฉพาะตอนสมาชิกกดบันทึก
3. **CMS (`/admin`):** จัดการผู้ใช้, Ads, GA, Affiliate, แบนเนอร์ผู้สนับสนุน, Dashboard และ Audit Log เข้าได้เฉพาะ Admin ที่ผ่าน 2FA แล้ว
4. **Redirect endpoint:** `/go/{id}` สำหรับนับคลิก Affiliate และแบนเนอร์ ต้องตอบเร็วและทำงานที่ edge (ส่วน `/r/{code}` สำหรับ Dynamic QR พักไว้พร้อม Phase 2)

```text
                    ┌────────────────────── CDN / Edge ──────────────────────┐
ผู้ใช้ (เบราว์เซอร์) ──►│ หน้าเว็บสาธารณะ (SSG/ISR)   Member Area (SSR)   CMS (SSR) │
 │                   └───────────────┬─────────────────────────────────────────┘
 ├─ QR Engine (JS, client-side)      │ Server Actions / Route Handlers
 ├─ AdSense / GA4 (หลัง consent)      ▼
 │                          ┌──────────────────┐     ┌────────────────────────┐
 │                          │ Application      │────►│ PostgreSQL (managed)   │
 │                          │ (Next.js server) │     └────────────────────────┘
 │                          │  - Auth          │────►  Redis (rate limit, cache)
 │                          │  - Member API    │────►  Object storage (โลโก้, รูปโปรไฟล์)
 │                          │  - Admin API     │────►  Email service (ยืนยันอีเมล, รีเซ็ตรหัสผ่าน)
 │                          └──────────────────┘────►  GA4 Data API (Dashboard)
 └─ Google / LINE OAuth ◄──────────── OAuth callback
```

## 2. ข้อจำกัดทางเทคนิค

- ใช้โดเมน `www.qreasy.info` เดิม (BR-50)
- ต้องทำงานกับ Google AdSense และ Google Analytics 4 (A-03)
- ทีมพัฒนายังไม่มีโค้ดของ V1 จึงย้ายได้เฉพาะพฤติกรรมที่สังเกตจากภายนอก
- เพราะมีระบบสมาชิก จึงเปลี่ยนจากเว็บ static เป็นแอปที่มี server runtime และฐานข้อมูล

## 3. เทคโนโลยีที่เสนอ

| ส่วน | ข้อเสนอ | เหตุผล |
| --- | --- | --- |
| ภาษา | TypeScript | ใช้ type ร่วมกันได้ทั้งฝั่งเบราว์เซอร์และเซิร์ฟเวอร์ |
| Framework | Next.js (App Router) | หน้าสาธารณะใช้ SSG/ISR เพื่อ SEO ส่วนหน้าสมาชิกและ CMS ใช้ SSR พร้อม Server Actions |
| UI | Tailwind CSS + shadcn/ui | ทำ responsive และ dark mode ได้ง่าย และมีคอมโพเนนต์ตาราง/ฟอร์มสำหรับ CMS |
| QR Engine | `qr-code-styling` (หรือ `qrcode` + โค้ดวาดเอง) | รองรับโลโก้ รูปทรงจุด และส่งออก PNG/SVG |
| PDF | `jspdf` (lazy-load) | ไม่เพิ่มขนาดไฟล์ที่ต้องโหลดตอนเปิดหน้า |
| i18n | `next-intl` | รองรับ route `/th` และ `/en` |
| Validation | `zod` | ใช้ schema เดียวกันทั้งฝั่ง client และ server |
| Database | PostgreSQL แบบ managed (เช่น Neon, Supabase หรือ AWS RDS) | ข้อมูลเชิงสัมพันธ์ (user, role, QR, audit) ทำ transaction ได้ |
| ORM / Migration | Drizzle ORM (หรือ Prisma) | type-safe และมีระบบ migration |
| Authentication | Better Auth (หรือ Auth.js) | รองรับ email/password, OAuth (Google, LINE), email verification, 2FA (TOTP), session และ plugin สำหรับ admin |
| Password hashing | Argon2id (หรือ scrypt ตามค่าเริ่มต้นของไลบรารี) | มาตรฐาน OWASP |
| Cache / Rate limit | Redis (เช่น Upstash) | rate limit ใช้ร่วมกันได้ข้ามหลาย serverless instance |
| Object storage | S3-compatible (เช่น Cloudflare R2 หรือ AWS S3) | เก็บโลโก้ของ QR ที่บันทึกและรูปโปรไฟล์ |
| Email | Resend หรือ Amazon SES พร้อมตั้งค่า SPF/DKIM/DMARC | ส่งอีเมลยืนยันตัวตนและรีเซ็ตรหัสผ่านได้เชื่อถือได้ |
| Hosting | Vercel (หรือ container บน Cloud Run / Fly.io) | มี CDN, HTTPS และ preview deployment |
| Bot protection | Cloudflare Turnstile หรือ reCAPTCHA v3 | ผู้ใช้ไม่ต้องคลิกยืนยัน |
| Consent | CMP ที่ Google รับรอง + Google Consent Mode v2 | สอดคล้องกับ PDPA และนโยบาย AdSense |
| เครือข่ายโฆษณา | Google AdSense (หลัก), Ezoic หรือ Google Ad Manager ผ่านพาร์ตเนอร์ MCM (ทดสอบ A/B) | เริ่มจาก AdSense แล้ววัดผลก่อนเปลี่ยน |
| Affiliate | Shopee Affiliate, Lazada Affiliate, Involve Asia, ACCESSTRADE Thailand | แพลตฟอร์มที่ผู้ใช้ชาวไทยซื้อของจริง |
| Web Vitals ต่อกลุ่ม A/B | ไลบรารี `web-vitals` ส่งค่าเข้า GA4 พร้อม `ad_variant` | ใช้เทียบประสิทธิภาพระหว่างเครือข่ายโฆษณา |
| _(พักไว้)_ Payment, Link safety, Geo/UA, Queue | Opn Payments/2C2P, Web Risk API, edge geo header + `ua-parser-js`, QStash/Cloudflare Queues | ใช้กับ Phase 2 เท่านั้น |
| Testing | Vitest, Playwright | unit, integration และ end-to-end test |

## 4. ข้อกำหนดเชิงฟังก์ชันทางเทคนิค

### 4.1 QR Engine และประเภทข้อมูล

| รหัส | ข้อกำหนด | อ้างอิง |
| --- | --- | --- |
| TR-01 | สร้าง QR Code ตามมาตรฐาน ISO/IEC 18004 บนเบราว์เซอร์ทั้งหมด ไม่ส่งข้อมูลที่ Guest กรอกออกจากเครื่อง | BR-01, BR-43 |
| TR-02 | ตรวจ URL ด้วย `URL` API อนุญาตเฉพาะ scheme `http` และ `https` ถ้าผู้ใช้ไม่ใส่ scheme ให้เติม `https://` ให้อัตโนมัติ | BR-02, BR-44 |
| TR-03 | ปฏิเสธ scheme อันตราย เช่น `javascript:`, `data:`, `vbscript:` และ `file:` ทั้งฝั่ง client และตอนบันทึกฝั่ง server | BR-44 |
| TR-04 | รองรับข้อความ UTF-8 รวมภาษาไทย และแจ้งเตือนเมื่อข้อมูลยาวเกินกว่าที่ QR Code รองรับที่ระดับ Error Correction ที่เลือก | BR-03 |
| TR-05 | สร้าง payload Wi-Fi รูปแบบ `WIFI:T:<WPA\|WEP\|nopass>;S:<ssid>;P:<password>;H:<true\|false>;;` พร้อม escape อักขระ `\ ; , : "` | BR-04 |
| TR-06 | สร้าง payload `mailto:` (to, subject, body), `tel:` และ `SMSTO:` โดย encode ตาม RFC ที่เกี่ยวข้อง | BR-05 |
| TR-07 | สร้าง vCard เวอร์ชัน 3.0 (N, FN, ORG, TITLE, TEL, EMAIL, URL, ADR) | BR-06 |
| TR-08 | อัปเดตพรีวิวแบบ debounce 150–300 ms หลังผู้ใช้หยุดพิมพ์ และไม่ทำให้หน้าเว็บค้างระหว่างพิมพ์ | BR-07 |
| TR-09 | ขั้นตอนสร้าง QR Code ต้องไม่มี CAPTCHA ไม่บังคับ Login และไม่เรียก API ฝั่งเซิร์ฟเวอร์ | BR-08 |

### 4.2 การปรับแต่ง

| รหัส | ข้อกำหนด | อ้างอิง |
| --- | --- | --- |
| TR-10 | ใช้ color picker และช่องกรอกค่า HEX สำหรับสีจุดและสีพื้นหลัง รองรับพื้นหลังโปร่งใสสำหรับ PNG/SVG | BR-10 |
| TR-11 | ขนาดส่งออก PNG ตั้งแต่ 256 ถึง 4096 px ค่าเริ่มต้น 1024 px และมี quiet zone อย่างน้อย 4 module | BR-11 |
| TR-12 | ให้เลือก Error Correction L/M/Q/H ค่าเริ่มต้น M และบังคับเป็น H เมื่อใส่โลโก้ | BR-12, BR-13 |
| TR-13 | อัปโหลดโลโก้ชนิด PNG/JPEG/SVG ขนาดไม่เกิน 2 MB จำกัดพื้นที่โลโก้ไม่เกิน 20% ของพื้นที่ QR และ sanitize ไฟล์ SVG ก่อนใช้ สำหรับ Guest ให้ประมวลผลบนเครื่องเท่านั้น | BR-13 |
| TR-14 | มีรูปทรงจุดอย่างน้อย square, rounded, dots และรูปทรงมุมอย่างน้อย square, rounded | BR-14 |
| TR-15 | คำนวณ contrast ratio ระหว่างสีจุดกับพื้นหลัง เตือนเมื่อต่ำกว่า 4:1 และเตือนเมื่อจุดสีอ่อนกว่าพื้นหลัง (inverted) | BR-15 |

### 4.3 การส่งออกและแชร์

| รหัส | ข้อกำหนด | อ้างอิง |
| --- | --- | --- |
| TR-20 | ส่งออก PNG ผ่าน Canvas โดยตั้งชื่อไฟล์ `qreasy-<type>-<yyyyMMdd-HHmmss>.png` | BR-20 |
| TR-21 | ส่งออก SVG แบบ vector จริง (ไม่ใช่ภาพ raster ห่อใน SVG) | BR-21 |
| TR-22 | ส่งออก PDF ขนาด A4 หรือขนาดที่ผู้ใช้กำหนด โดย lazy-load ไลบรารี PDF | BR-22 |
| TR-23 | คัดลอกภาพด้วย `navigator.clipboard.write` (ClipboardItem PNG) และซ่อนปุ่มเมื่อเบราว์เซอร์ไม่รองรับ | BR-23 |
| TR-24 | แชร์ไฟล์ด้วย Web Share API Level 2 (`navigator.canShare({ files })`) และซ่อนปุ่มเมื่อไม่รองรับ | BR-24 |

### 4.4 หน้าเว็บ เนื้อหา และ i18n

| รหัส | ข้อกำหนด | อ้างอิง |
| --- | --- | --- |
| TR-30 | ออกแบบแบบ mobile-first รองรับความกว้างหน้าจอตั้งแต่ 320 px และไม่มี horizontal scroll (CMS รองรับตั้งแต่ 768 px ขึ้นไป) | BR-30 |
| TR-31 | ใช้ route แยกภาษา `/th/...` และ `/en/...` มี `hreflang` และ `x-default` ส่วน `/` ให้เลือกภาษาจากค่าที่สมาชิกตั้งไว้ จาก cookie หรือจาก `Accept-Language` ตามลำดับ | BR-31, BR-66 |
| TR-32 | เก็บข้อความ UI ทั้งหมด รวมถึงเทมเพลตอีเมล ในไฟล์แปลภาษา ห้าม hard-code ข้อความในคอมโพเนนต์ | BR-31 |
| TR-33 | หน้า static ได้แก่ About, Contact, Privacy Policy, Terms, FAQ เขียนเนื้อหาเป็น MDX แยกตามภาษา | BR-32, BR-33 |
| TR-34 | ฟอร์ม Contact ส่งไปที่ server action ตรวจ input ด้วย zod ป้องกันด้วย Turnstile/reCAPTCHA v3 และจำกัด 5 ครั้งต่อ IP ต่อชั่วโมง | BR-32 |
| TR-35 | ผ่านเกณฑ์ WCAG 2.2 ระดับ AA ได้แก่ label ครบ ใช้งานด้วยคีย์บอร์ดได้ มี focus ที่มองเห็นได้ และมี `alt` สำหรับภาพพรีวิว ครอบคลุมทั้งหน้าสาธารณะ หน้าสมาชิก และ CMS | BR-34 |
| TR-36 | รองรับ `prefers-color-scheme` และมีปุ่มสลับธีม ส่วนพรีวิว QR ยังคงใช้สีที่ผู้ใช้เลือก | BR-35 |

### 4.5 SEO, Analytics, โฆษณา และ Consent

| รหัส | ข้อกำหนด | อ้างอิง |
| --- | --- | --- |
| TR-40 | ทุกหน้าสาธารณะมี `<title>`, meta description, Open Graph, canonical URL และ JSON-LD (`WebApplication`, `FAQPage`) ส่วนหน้าสมาชิก หน้า auth และ CMS ต้องตั้ง `noindex, nofollow` | OBJ-03 |
| TR-41 | สร้าง `robots.txt` และ `sitemap.xml` อัตโนมัติ โดยไม่รวม `/admin`, `/account` และหน้า auth | OBJ-03 |
| TR-42 | ตำแหน่งโฆษณาต้องจองพื้นที่ล่วงหน้าเพื่อไม่ให้เกิด layout shift และต้องไม่อยู่ระหว่างฟอร์มกับปุ่มดาวน์โหลด | BR-40 |
| TR-43 | ห้ามส่งเนื้อหาที่ผู้ใช้กรอก อีเมล หรือชื่อไปกับ event ใด ๆ ของ GA (รายการ event อยู่ในหัวข้อ 7) | BR-41, BR-43 |
| TR-44 | ก่อนผู้ใช้ให้ความยินยอม ห้ามโหลดสคริปต์ที่ตั้งคุกกี้ด้านโฆษณาหรือการวิเคราะห์ ให้ใช้ Consent Mode v2 (`ad_storage`, `analytics_storage`, `ad_user_data`, `ad_personalization` เริ่มต้นเป็น `denied`) | BR-42 |
| TR-45 | ผู้ใช้เปลี่ยนการตั้งค่าความยินยอมได้ภายหลังจากลิงก์ใน footer และสมาชิกเปลี่ยนได้จากหน้าตั้งค่าบัญชีด้วย | BR-42 |

### 4.6 การย้ายจาก V1

| รหัส | ข้อกำหนด | อ้างอิง |
| --- | --- | --- |
| TR-50 | redirect 301 จาก `/createqr` (GET) ไปยังหน้าแรก ส่วน POST ไปที่ `/createqr` ให้ตอบกลับด้วยหน้าแรกหรือ 303 See Other | BR-51 |
| TR-51 | เก็บ favicon และชื่อแบรนด์ `QREASY.INFO` ไว้ตามเดิมจนกว่าจะมีแนวทางแบรนด์ใหม่ (Q-04) | BR-50 |
| TR-52 | ย้าย GA4 property และ AdSense publisher ID เดิมมาใช้ เพื่อให้ข้อมูลต่อเนื่อง และใส่ค่าเริ่มต้นนี้ใน `site_settings` ตอน seed ฐานข้อมูล | BR-41, BR-90 |

### 4.7 ระบบสมาชิกและการยืนยันตัวตน

| รหัส | ข้อกำหนด | อ้างอิง |
| --- | --- | --- |
| TR-60 | สมัครด้วยอีเมลและรหัสผ่าน รหัสผ่านยาว 8–128 ตัวอักษร ตรวจกับรายการรหัสผ่านที่เคยรั่วไหล (เช่น HIBP k-anonymity API) และ hash ด้วย Argon2id หรือ scrypt | BR-60 |
| TR-61 | ส่งลิงก์ยืนยันอีเมลแบบใช้ครั้งเดียว อายุ 24 ชั่วโมง ก่อนยืนยันจะ Login ได้ แต่ใช้ฟีเจอร์สมาชิกไม่ได้ และขอส่งซ้ำได้ไม่เกิน 3 ครั้งต่อชั่วโมง | BR-60 |
| TR-62 | Login ด้วย Google ผ่าน OpenID Connect (scope: `openid email profile`) ถ้าอีเมลซ้ำกับบัญชีที่มีอยู่และอีเมลยืนยันแล้ว ให้เชื่อมบัญชี (account linking) | BR-61 |
| TR-63 | Login ด้วย LINE Login v2.1 (OIDC) ถ้า LINE ไม่ส่งอีเมลกลับมา ให้ขออีเมลจากผู้ใช้และยืนยันก่อนเปิดใช้งาน | BR-62 |
| TR-64 | Session เก็บใน cookie แบบ `HttpOnly`, `Secure`, `SameSite=Lax` มีอายุ 7 วัน ถ้าเลือก "จดจำการเข้าสู่ระบบ" จะเป็น 30 วัน และต่ออายุแบบ sliding | BR-63 |
| TR-65 | ลิงก์รีเซ็ตรหัสผ่านใช้ได้ครั้งเดียว อายุ 1 ชั่วโมง หลังรีเซ็ตให้ยกเลิก session อื่นทั้งหมด และระบบต้องตอบกลับเหมือนกันไม่ว่าอีเมลนั้นมีบัญชีหรือไม่ (กันการเดาอีเมล) | BR-63 |
| TR-66 | จำกัดการ Login ผิด 5 ครั้งต่อบัญชีต่อ 15 นาที และ 20 ครั้งต่อ IP ต่อ 15 นาที เมื่อเกินให้หน่วงเวลาและบังคับ Turnstile | BR-70 |
| TR-67 | หน้าสมัครสมาชิก Login และลืมรหัสผ่านใช้ Turnstile/reCAPTCHA v3 ตรวจฝั่ง server | BR-70 |
| TR-68 | บันทึกการยอมรับ Terms และ Privacy Policy (เวอร์ชันเอกสาร และเวลา) และค่า opt-in ของอีเมลข่าวสารไว้ในตาราง `consents` | BR-69 |
| TR-69 | ส่งอีเมลแจ้งเตือนเมื่อเปลี่ยนรหัสผ่าน เปลี่ยนอีเมล (ส่งไปทั้งอีเมลเก่าและใหม่) หรือเพิ่ม/ลบวิธี Login | BR-71 |

### 4.8 ฟีเจอร์ของสมาชิก

| รหัส | ข้อกำหนด | อ้างอิง |
| --- | --- | --- |
| TR-70 | ปุ่ม "บันทึก" ในหน้า Generator ส่งข้อมูลไปที่ server เฉพาะเมื่อสมาชิกกดเอง ได้แก่ ประเภท QR, payload, การตั้งค่าการแสดงผล (JSON) และชื่อ ถ้ายังไม่ Login ให้พาไปหน้า Login แล้วกลับมาพร้อมข้อมูลเดิม | BR-64 |
| TR-71 | หน้า "QR Code ของฉัน" (`/account/qr`) แสดงรายการแบบแบ่งหน้า (20 รายการต่อหน้า) ค้นหาตามชื่อ กรองตามประเภท และมีภาพตัวอย่างที่ render ฝั่ง client จากข้อมูลที่บันทึก | BR-64 |
| TR-72 | จำกัดจำนวน QR ที่บันทึกต่อบัญชีตามค่าใน `site_settings.max_saved_qr` (ค่าเริ่มต้น 100 รอผล Q-07) | BR-64 |
| TR-73 | ถ้าเป็น QR ประเภท Wi-Fi ให้เตือนว่ารหัสผ่านจะถูกบันทึก และมีตัวเลือก "ไม่บันทึกรหัสผ่าน" | BR-64, R-07 |
| TR-74 | Preset เก็บเฉพาะการตั้งค่าการแสดงผล (ไม่มี payload) จำกัด 20 รายการต่อบัญชี | BR-65 |
| TR-75 | โลโก้ที่สมาชิกบันทึกและรูปโปรไฟล์ อัปโหลดไป object storage ผ่าน pre-signed URL ตรวจ MIME type และขนาดฝั่ง server, strip metadata (EXIF) และ sanitize SVG | BR-64, BR-66 |
| TR-76 | การลบบัญชีต้องยืนยันด้วยรหัสผ่านหรือ re-auth ผ่าน OAuth แล้ว soft delete ทันที (Login ไม่ได้) และ hard delete ข้อมูลส่วนบุคคล ไฟล์ และ QR หลังพ้นช่วงเวลาที่กำหนดใน Q-09 (ค่าเริ่มต้น 30 วัน) | BR-67 |
| TR-77 | ส่งออกข้อมูลส่วนตัวเป็นไฟล์ JSON (โปรไฟล์, QR ที่บันทึก, preset, consents) ผ่านลิงก์ดาวน์โหลดที่หมดอายุภายใน 24 ชั่วโมง | BR-68 |

### 4.9 CMS — การจัดการผู้ใช้

| รหัส | ข้อกำหนด | อ้างอิง |
| --- | --- | --- |
| TR-80 | CMS อยู่ที่ `/admin` ตรวจ role ฝั่ง server ทุก request (middleware + ทุก server action) ห้ามพึ่งการซ่อนเมนูฝั่ง client อย่างเดียว | BR-75 |
| TR-81 | Admin และ Super Admin ต้องเปิด 2FA แบบ TOTP พร้อม recovery code ถ้ายังไม่เปิด 2FA จะเข้า `/admin` ไม่ได้ session ของ CMS มีอายุสูงสุด 8 ชั่วโมง และหมดอายุเมื่อไม่ใช้งาน 30 นาที | BR-85 |
| TR-82 | รายการผู้ใช้ใช้ server-side pagination (25/50/100 ต่อหน้า) ค้นหาด้วยอีเมลหรือชื่อ (ILIKE + index `pg_trgm`) และกรองตาม status, role, provider, ช่วงวันที่สมัคร และ Login ล่าสุด | BR-76 |
| TR-83 | หน้ารายละเอียดผู้ใช้แสดงข้อมูลโปรไฟล์, วิธี Login ที่ผูกไว้, สถานะยืนยันอีเมล, วันที่สมัคร, Login ล่าสุด, จำนวน QR, session ที่ใช้งานอยู่ และ Audit Log ที่เกี่ยวกับผู้ใช้นั้น โดยไม่แสดงเนื้อหา payload ของ QR | BR-77 |
| TR-84 | การระงับผู้ใช้จะเปลี่ยน `status = suspended` บันทึกเหตุผล และยกเลิกทุก session ทันที ผู้ใช้ที่ถูกระงับ Login ไม่ได้และเห็นข้อความแจ้งพร้อมช่องทางติดต่อ | BR-78 |
| TR-85 | การลบผู้ใช้โดย Admin ต้องพิมพ์อีเมลของผู้ใช้เพื่อยืนยัน และใช้กระบวนการเดียวกับ TR-76 Admin ลบหรือระงับบัญชีที่มี role เท่ากันหรือสูงกว่าตัวเองไม่ได้ | BR-79 |
| TR-86 | Admin ส่งอีเมลยืนยันหรือรีเซ็ตรหัสผ่านซ้ำได้ (ใช้ rate limit เดียวกับ TR-61, TR-65) ห้ามมี API ที่อ่านหรือกำหนดรหัสผ่านของผู้ใช้ | BR-80 |
| TR-87 | ปุ่ม "Logout ทุกอุปกรณ์" ลบ session ทั้งหมดของผู้ใช้ในฐานข้อมูล | BR-81 |
| TR-88 | Super Admin เปลี่ยน role ได้ ระบบต้องมี Super Admin อย่างน้อย 1 คนเสมอ และ Super Admin คนแรกสร้างผ่าน CLI/seed script เท่านั้น ห้ามสร้างผ่านหน้าเว็บ | BR-82 |
| TR-89 | ทุก mutation ใน CMS บันทึกลงตาราง `audit_logs` (actor, action, target, ค่าก่อน/หลังที่ตัดข้อมูลอ่อนไหวออกแล้ว, IP, user agent, เวลา) แบบ append-only ห้ามแก้หรือลบผ่านแอป และเก็บไว้อย่างน้อย 1 ปี | BR-83 |
| TR-90 | ส่งออก CSV ได้เฉพาะ id, ชื่อที่แสดง, อีเมล, role, status, วันที่สมัคร และ Login ล่าสุด ตามตัวกรองที่เลือก และบันทึกการส่งออกใน Audit Log | BR-84 |

### 4.10 CMS — การจัดการ Ads และ GA

| รหัส | ข้อกำหนด | อ้างอิง |
| --- | --- | --- |
| TR-91 | ค่าการตั้งค่าทั้งหมดเก็บในตาราง `site_settings` และ `ad_slots` ฝั่งเว็บอ่านผ่าน cache (revalidate ทันทีเมื่อ Admin บันทึก หรือ TTL ≤ 60 วินาที) | BR-86, BR-87, BR-90 |
| TR-92 | `ads_enabled` (global) และ `ad_slots` ประกอบด้วย key, ชื่อ, `ad_client` (`ca-pub-\d{16}`), `ad_slot_id` (ตัวเลข), รูปแบบ (responsive/fixed), ขนาดที่จองไว้, หน้าที่แสดง (route pattern) และ enabled โดยตรวจรูปแบบ ID ก่อนบันทึก | BR-86, BR-87 |
| TR-93 | `/ads.txt` เสิร์ฟจากค่า `site_settings.ads_txt` เป็น `text/plain` และตรวจรูปแบบบรรทัดก่อนบันทึก | BR-88 |
| TR-94 | Layout ของ `/login`, `/register`, `/forgot-password`, `/account/**` และ `/admin/**` ต้องไม่ render ad component และไม่โหลดสคริปต์ AdSense | BR-89 |
| TR-95 | `ga_enabled` และ `ga_measurement_id` (`G-[A-Z0-9]+`) ตั้งค่าได้จาก CMS ห้ามโหลดสคริปต์ GA ใน `/admin/**` | BR-90 |
| TR-96 | Dashboard ดึงตัวเลขของสมาชิกจากฐานข้อมูล (สมาชิกทั้งหมด, สมาชิกใหม่ 7/30 วัน, active 30 วัน, QR ที่บันทึก) และดึงตัวเลข GA (users, sessions, `qr_download` แยกตามประเภทและรูปแบบไฟล์) ผ่าน GA4 Data API ด้วย service account แบบ read-only แล้ว cache ไว้ 1 ชั่วโมง | BR-92 |
| TR-97 | `ads_hide_for_members` เมื่อเปิด จะไม่ render โฆษณาให้ผู้ใช้ที่ Login อยู่ | BR-93 |
| TR-98 | การเปลี่ยนค่าใน 4.10 ทุกครั้งต้องบันทึก Audit Log ตาม TR-89 | BR-83 |

### 4.11 Affiliate

| รหัส | ข้อกำหนด | อ้างอิง |
| --- | --- | --- |
| TR-100 | คอมโพเนนต์ `AffiliateCard` เลือกสินค้าจาก `affiliate_products` ที่ enabled และผูกกับประเภท QR ปัจจุบัน (ถ้าไม่มีให้ใช้สินค้าทั่วไปที่ผูกกับ `any`) แสดงสูงสุด 3 รายการ เรียงตาม `sort_order` และ render ฝั่ง server หรือ cache เพื่อไม่เพิ่ม JS ที่โหลดตอนเปิดหน้าแรก | BR-100 |
| TR-101 | CMS มี CRUD ของ `affiliate_products` อัปโหลดรูปไป object storage (ตาม TR-75) และตรวจว่าลิงก์ปลายทางเป็น `https` และอยู่ใน allowlist ของโดเมนแพลตฟอร์ม (เช่น `shopee.co.th`, `s.shopee.co.th`, `lazada.co.th`, `s.lazada.co.th`, โดเมนของ Involve Asia และ ACCESSTRADE) | BR-101 |
| TR-102 | ทุกการ์ดแสดงป้าย "ลิงก์ Affiliate" / "Affiliate link" จากไฟล์แปลภาษา และทุกลิงก์มี `rel="sponsored nofollow noopener"` และ `target="_blank"` | BR-102 |
| TR-103 | ลิงก์ในการ์ดชี้ไปที่ `/go/a/{productId}?p={placement}&t={qrType}` ซึ่งเพิ่มตัวนับใน Redis (`INCR` ต่อ product/placement/type/วัน) แล้วตอบ 302 ไปยังลิงก์ Affiliate ทันที ไม่เก็บ IP, user id หรือคุกกี้ มี job รวมค่าลงตาราง `affiliate_click_daily` ทุกชั่วโมง | BR-103 |
| TR-104 | `/go/*` กรอง bot ด้วย user agent พื้นฐานก่อนนับ ตั้ง `noindex` และใส่ `Disallow: /go/` ใน robots.txt | BR-103 |
| TR-105 | Layout ของหน้า auth และ `/admin/**` ไม่ render `AffiliateCard` | BR-104 |
| TR-106 | Dashboard แสดงคลิกต่อสินค้า แพลตฟอร์ม และตำแหน่ง ส่วน CTR คำนวณจาก impression ซึ่งนับเมื่อการ์ดเข้าสู่หน้าจอ (IntersectionObserver ≥ 50% อย่างน้อย 1 วินาที) โดยส่ง beacon ไปที่ `/go/i` แบบรวมเป็นชุด | BR-105 |
| TR-107 | A/B test การ์ด Affiliate ใช้กลไกเดียวกับ TR-112 โดยใช้ experiment key แยก | BR-106 |

### 4.12 เครือข่ายโฆษณา, A/B Test และแบนเนอร์ผู้สนับสนุน

| รหัส | ข้อกำหนด | อ้างอิง |
| --- | --- | --- |
| TR-110 | ตำแหน่งโฆษณามาตรฐาน: `generator_side` (เดสก์ท็อป ≥ 1024 px, 300×600 หรือ 300×250), `generator_below` (มือถือ, responsive ใต้พรีวิว), `mobile_anchor` (sticky ด้านล่าง ปิดได้), `article_inline` (ในบทความ/FAQ) ทุกตำแหน่งจองพื้นที่ล่วงหน้า (TR-42) และต้องห่างจากปุ่มดาวน์โหลด/คัดลอก/แชร์อย่างน้อย 150 px ทั้งแนวตั้งและแนวนอน ห้ามซ้อนกันบนหน้าจอมือถือ | BR-110, R-10 |
| TR-111 | สร้าง interface `AdProvider` (`loadScript()`, `renderSlot(slot)`, `destroySlot(slot)`, `consentUpdate(state)`) และมี implementation `adsense`, `ezoic`, `gam` โดยให้ `ad_slots` เก็บค่าแยกตาม provider ผ่าน field `provider_config` (jsonb) ส่วนหน้าเว็บโหลดสคริปต์ของ provider เดียวต่อ page view เท่านั้น | BR-111 |
| TR-112 | ตาราง `experiments` (key, variants พร้อมน้ำหนัก %, status, start/end) ใช้ middleware สุ่ม variant ครั้งแรกแบบ deterministic จาก random id ที่เก็บใน cookie `qe_id` (first-party, 90 วัน) แล้วส่งค่า variant ให้ layout โดย cookie นี้ไม่มีข้อมูลส่วนบุคคล ถ้ายังไม่ได้ consent ด้านโฆษณา ก็ยังใช้ variant ได้แต่ provider ต้องทำงานในโหมด non-personalized ตาม Consent Mode | BR-112 |
| TR-113 | ส่ง user property `ad_variant` ไป GA4 และส่งค่า LCP, INP, CLS ผ่าน `web-vitals` เป็น event `web_vitals` พร้อม variant | BR-113 |
| TR-114 | หน้า "Experiments" ใน CMS แสดงตามแต่ละ variant: page view (GA4 Data API), อัตรา `qr_download`, Web Vitals p75 และให้ Admin กรอกรายได้จากรายงานของ provider (หรือนำเข้า CSV) เพื่อคำนวณ RPM จากนั้นกด "เลือกผู้ชนะ" เพื่อปรับน้ำหนักเป็น 100% ได้ | BR-113 |
| TR-115 | แบนเนอร์ผู้สนับสนุนเก็บในตาราง `sponsor_banners` (รูป ไทย/อังกฤษ, alt, ลิงก์, slot, ช่วงวันที่, priority) ถ้าวันนี้มีแบนเนอร์ที่ active ใน slot นั้น ให้แสดงแทนโฆษณาเครือข่าย ลิงก์ผ่าน `/go/s/{bannerId}` และมีป้าย "ผู้สนับสนุน" | BR-114 |
| TR-116 | นับ impression ของแบนเนอร์ด้วยกลไกเดียวกับ TR-106 และ CMS ส่งออกรายงาน impression/คลิกรายวันเป็น CSV ต่อแบนเนอร์ได้ | BR-115 |

### 4.13 Phase 2 — Dynamic QR, Subscription และ Payment (พักไว้)

> **พักไว้** ไม่อยู่ในขอบเขตการพัฒนาปัจจุบัน

| รหัส | ข้อกำหนด | อ้างอิง |
| --- | --- | --- |
| TR-120 | Dynamic QR เข้ารหัส URL `https://qreasy.info/r/{code}` โดย `code` เป็น base62 ยาว 7 ตัวอักษร สุ่มจาก CSPRNG และตรวจไม่ให้ซ้ำ (unique index) ให้ภาพ QR สั้นและสแกนง่าย สมาชิกแก้ `target_url` ได้ตลอดโดยใช้การตรวจตาม TR-02/TR-03 | BR-120 |
| TR-121 | `/r/{code}` ทำงานที่ edge อ่าน mapping จาก Redis (cache-aside จาก PostgreSQL, invalidate ทันทีเมื่อแก้ไข) แล้วตอบ `302` พร้อม `Cache-Control: no-store` เพื่อให้แก้ปลายทางได้ทันที ถ้า QR ถูกปิดหรือหมดอายุ ให้แสดงหน้าแจ้งพร้อมสถานะ `410` หรือ `451` ตามสาเหตุ | BR-120, BR-129, BR-130 |
| TR-122 | หลัง redirect ส่ง scan event เข้า queue แบบ async (`waitUntil`) ประกอบด้วย qr id, เวลา, ประเทศและภูมิภาคจาก header ของ edge, ประเภทอุปกรณ์/OS/เบราว์เซอร์จาก UA และ visitor hash = `HMAC(salt_ของวัน, ip + ua)` ซึ่งใช้นับสแกนไม่ซ้ำต่อวัน และเปลี่ยน salt ทุกวัน ห้ามเก็บ IP จริง | BR-121 |
| TR-123 | Worker รวม event ลง `qr_scan_daily` (qr_id, date, country, region, device, os, scans, unique_scans) และเก็บ event ดิบไม่เกิน 90 วัน หน้าสถิติแสดงกราฟรายวันย้อนหลัง 7/30/90 วัน และส่งออก CSV ได้ | BR-121 |
| TR-124 | ตาราง `plans` (code, ชื่อ, ราคา THB, interval month/year, features jsonb, active) และ `entitlements` คำนวณจาก subscription ที่ active ใช้ฟังก์ชัน `can(user, feature)` ฝั่ง server ในทุกจุดที่ตรวจสิทธิ์ Pro ห้ามตรวจจาก role | BR-122, BR-133 |
| TR-125 | ชำระผ่าน hosted payment page หรือ tokenization ของ gateway เท่านั้น ระบบไม่รับหรือเก็บเลขบัตร (PCI DSS SAQ A) บัตรใช้ recurring charge หรือ schedule ของ gateway ส่วน PromptPay สร้าง QR ชำระต่อรอบ และส่งอีเมลแจ้งเตือนก่อนหมดอายุ 7 วันและ 1 วัน | BR-123 |
| TR-126 | สถานะการชำระเงินอัปเดตจาก webhook ของ gateway เท่านั้น ต้องตรวจ signature, ทำงานแบบ idempotent (เก็บ event id) และ reconcile กับ API ของ gateway ทุกวัน ห้ามเชื่อ redirect ฝั่ง client | BR-123 |
| TR-127 | สร้างเลขที่ใบเสร็จต่อเนื่องไม่ซ้ำ และสร้าง PDF ใบเสร็จ/ใบกำกับภาษีพร้อมข้อมูลผู้ซื้อ (ชื่อ, ที่อยู่, เลขผู้เสียภาษีถ้ามี) โดยรูปแบบสุดท้ายรอ Q-12 | BR-124 |
| TR-128 | เมื่อ `can(user, 'ad_free')` เป็นจริง layout จะไม่โหลดสคริปต์ของเครือข่ายโฆษณาเลย (ไม่ใช่แค่ซ่อน) | BR-125 |
| TR-129 | Bulk: อัปโหลด CSV สูงสุด 500 แถวต่อครั้ง ตรวจทุกแถวก่อนสร้าง แสดงแถวที่ผิด แล้วสร้างไฟล์ ZIP (PNG/SVG) บนเบราว์เซอร์ด้วย Web Worker เพื่อลดภาระเซิร์ฟเวอร์ ถ้าเลือก Dynamic ให้สร้าง short code ฝั่ง server เป็นชุดก่อน | BR-126 |
| TR-130 | PDF พร้อมพิมพ์ใช้ vector, ตั้ง bleed 3 mm และ crop mark และเลือกขนาดกระดาษ/สติกเกอร์ได้ | BR-127 |
| TR-131 | หน้า Billing: ดูแพ็กเกจปัจจุบัน วันหมดอายุ ประวัติการชำระและดาวน์โหลดใบเสร็จ ยกเลิกการต่ออายุ (มีผลเมื่อสิ้นรอบ) อัปเกรด (คิดเงินตามสัดส่วน) และเปลี่ยนบัตร | BR-128 |
| TR-132 | เมื่อ subscription หมดอายุ Dynamic QR ยังทำงานต่อตาม `site_settings.dynamic_grace_days` (ค่าเริ่มต้น 30 วัน รอ Q-15) แล้วจึงเปลี่ยนเป็นหน้าแจ้งหมดอายุที่มีปุ่มต่ออายุสำหรับเจ้าของ ข้อมูลไม่ถูกลบจนกว่าเจ้าของจะลบบัญชี | BR-129 |
| TR-133 | ตรวจ `target_url` ด้วย Safe Browsing/Web Risk ตอนสร้างและแก้ไข (ถ้าไม่ผ่านจะบันทึกไม่ได้) และตรวจซ้ำทุก QR ทุก 24 ชั่วโมง ถ้าพบลิงก์อันตรายให้ปิดอัตโนมัติและแจ้งเจ้าของ มีหน้า `/report?code=` ให้แจ้ง abuse และมีคิว "Reported QR" ใน CMS ให้ Admin ตรวจและปิด/เปิด QR ได้ (บันทึก Audit Log) | BR-130 |
| TR-134 | CMS จัดการ `plans` และ `coupons` (เปอร์เซ็นต์หรือจำนวนเงิน, จำนวนครั้ง, วันหมดอายุ) ดูรายการ subscription และ payment และสั่งคืนเงินผ่าน API ของ gateway (จำกัดเฉพาะ Super Admin และบันทึก Audit Log) | BR-131 |
| TR-135 | Dashboard คำนวณ MRR (รายปีคิดเป็น ÷ 12), จำนวน Pro ที่ active, อัตราการเปลี่ยนจากฟรีเป็น Pro ราย 30 วัน และ churn รายเดือน จากตาราง `subscriptions` | BR-132 |
| TR-136 | สมาชิกฟรีสร้าง Dynamic QR ได้ตาม `site_settings.free_dynamic_limit` (ค่าเริ่มต้น 1 รอ Q-16) และ QR ของสมาชิกฟรีแสดงหน้าคั่น (interstitial) ที่มีโฆษณาก่อน redirect ได้ถ้า Admin เปิดใช้ | BR-133 |

## 5. Data Model (เบื้องต้น)

ชื่อตารางและฟิลด์อาจปรับตามไลบรารี Auth ที่เลือก ตารางกลุ่ม auth (`users`, `accounts`, `sessions`, `verifications`) ให้ใช้ schema ที่ไลบรารีกำหนดแล้วเพิ่มฟิลด์ที่จำเป็น

| ตาราง | ฟิลด์หลัก | หมายเหตุ |
| --- | --- | --- |
| `users` | id (uuid), email (unique, citext), email_verified_at, name, image_url, locale, role (`member`/`admin`/`super_admin`), status (`active`/`suspended`/`pending_deletion`), suspended_reason, last_login_at, created_at, updated_at, deleted_at | |
| `accounts` | id, user_id, provider (`credential`/`google`/`line`), provider_account_id, password_hash (เฉพาะ credential) | unique (provider, provider_account_id) |
| `sessions` | id, user_id, token_hash, expires_at, ip, user_agent, created_at | เก็บ hash ของ token ไม่เก็บ token จริง |
| `verifications` | id, identifier, token_hash, purpose (`verify_email`/`reset_password`), expires_at, used_at | |
| `two_factor` | user_id, totp_secret (encrypted), recovery_codes_hash, enabled_at | |
| `qr_codes` | id, user_id, name, type, payload (encrypted), style (jsonb), logo_object_key, created_at, updated_at | index (user_id, created_at) |
| `qr_presets` | id, user_id, name, style (jsonb), logo_object_key, created_at | |
| `consents` | id, user_id, kind (`terms`/`privacy`/`marketing`), version, granted (bool), created_at | append-only |
| `site_settings` | key (pk), value (jsonb), updated_by, updated_at | เช่น `ads_enabled`, `ads_txt`, `ga_enabled`, `ga_measurement_id`, `max_saved_qr`, `ads_hide_for_members` |
| `ad_slots` | id, key, name, ad_client, ad_slot_id, format, reserved_size, routes (text[]), enabled, updated_by, updated_at | |
| `audit_logs` | id, actor_user_id, action, target_type, target_id, before (jsonb), after (jsonb), ip, user_agent, created_at | append-only |
| `affiliate_products` | id, name_th, name_en, image_object_key, price_hint, url, platform, qr_types (text[]), sort_order, enabled, updated_by, updated_at | |
| `affiliate_click_daily` | product_id, placement, qr_type, date, impressions, clicks | pk (product_id, placement, qr_type, date) |
| `sponsor_banners` | id, sponsor_name, image_th_key, image_en_key, alt_th, alt_en, url, slot_key, starts_at, ends_at, priority, enabled | |
| `sponsor_banner_daily` | banner_id, date, impressions, clicks | |
| `experiments` | id, key, variants (jsonb: [{name, weight, config}]), status, starts_at, ends_at, winner | |
| `experiment_revenue` | experiment_id, variant, date, revenue_thb, source | Admin กรอกหรือนำเข้าจากรายงานของ provider |

**ตารางของ Phase 2 (พักไว้ ยังไม่ต้องสร้าง)**

| ตาราง | ฟิลด์หลัก | หมายเหตุ |
| --- | --- | --- |
| `dynamic_qr` | id, user_id, qr_code_id, code (unique), target_url, status (`active`/`disabled_by_owner`/`disabled_by_admin`/`unsafe`/`expired`), last_safety_check_at, created_at, updated_at | |
| `qr_scan_events` | id, dynamic_qr_id, scanned_at, country, region, device, os, browser, visitor_hash | เก็บ 90 วัน, partition ตามเดือน |
| `qr_scan_daily` | dynamic_qr_id, date, country, region, device, os, scans, unique_scans | |
| `qr_reports` | id, dynamic_qr_id, reason, detail, reporter_email (optional), status, handled_by, created_at | |
| `plans` | id, code, name_th, name_en, price_thb, interval, features (jsonb), active | |
| `subscriptions` | id, user_id, plan_id, status (`active`/`past_due`/`canceled`/`expired`), current_period_start, current_period_end, cancel_at_period_end, gateway, gateway_customer_id, gateway_schedule_id | |
| `payments` | id, subscription_id, amount_thb, method (`card`/`promptpay`), status, gateway_charge_id, receipt_no, paid_at, refunded_at | |
| `coupons` | id, code, kind (`percent`/`amount`), value, max_redemptions, redeemed_count, expires_at, active | |
| `webhook_events` | id (event id จาก gateway), type, payload, processed_at | ใช้ทำ idempotency |

- `qr_codes.payload` และ `two_factor.totp_secret` เข้ารหัสระดับแอปด้วย AES-256-GCM โดยใช้ key จาก environment หรือ KMS
- ทุกตารางที่เป็นข้อมูลของสมาชิกต้อง query ด้วย `user_id` ของ session เสมอ เพื่อกัน IDOR

## 6. API / Routes (เบื้องต้น)

ใช้ Server Actions เป็นหลัก ส่วน Route Handler ใช้เฉพาะกรณีที่ต้องการ HTTP endpoint จริง (OAuth callback, ไฟล์ดาวน์โหลด, `ads.txt`)

| Route / Action | สิทธิ์ | หน้าที่ |
| --- | --- | --- |
| `/api/auth/*` | สาธารณะ | สมัคร, Login, Logout, OAuth callback, ยืนยันอีเมล, รีเซ็ตรหัสผ่าน, 2FA (จัดการโดยไลบรารี Auth) |
| `saveQr`, `updateQr`, `deleteQr`, `listQr` | Member (ยืนยันอีเมลแล้ว) | จัดการ QR ที่บันทึก |
| `savePreset`, `deletePreset`, `listPresets` | Member | จัดการ preset |
| `updateProfile`, `changePassword`, `requestDataExport`, `deleteAccount` | Member | จัดการบัญชีตัวเอง |
| `admin.listUsers`, `admin.getUser` | Admin | ดูผู้ใช้ |
| `admin.suspendUser`, `admin.unsuspendUser`, `admin.deleteUser`, `admin.revokeSessions`, `admin.resendEmail` | Admin | จัดการผู้ใช้ |
| `admin.setRole` | Super Admin | แต่งตั้งหรือถอดถอน Admin |
| `admin.exportUsersCsv` | Admin | ส่งออก CSV |
| `admin.updateSettings`, `admin.upsertAdSlot`, `admin.deleteAdSlot` | Admin | ตั้งค่า Ads/GA |
| `admin.getDashboard` | Admin | ตัวเลข Dashboard |
| `admin.listAuditLogs` | Admin (เห็นเฉพาะของตัวเอง) / Super Admin (เห็นทั้งหมด) | ดู Audit Log |
| `GET /ads.txt` | สาธารณะ | เสิร์ฟค่า ads.txt |
| `GET /go/a/{productId}`, `GET /go/s/{bannerId}` | สาธารณะ | นับคลิกแล้ว 302 ไปปลายทาง |
| `POST /go/i` | สาธารณะ (rate limited) | รับ impression beacon แบบเป็นชุด |
| `admin.upsertAffiliateProduct`, `admin.deleteAffiliateProduct` | Admin | จัดการสินค้า Affiliate |
| `admin.upsertSponsorBanner`, `admin.exportSponsorReport` | Admin | จัดการแบนเนอร์ผู้สนับสนุน |
| `admin.upsertExperiment`, `admin.importExperimentRevenue`, `admin.pickWinner` | Admin | จัดการ A/B test |

**Phase 2 (พักไว้)**

| Route / Action | สิทธิ์ | หน้าที่ |
| --- | --- | --- |
| `GET /r/{code}` | สาธารณะ (edge) | redirect ของ Dynamic QR |
| `GET/POST /report` | สาธารณะ (Turnstile) | แจ้ง QR ที่น่าสงสัย |
| `createDynamicQr`, `updateDynamicTarget`, `toggleDynamicQr`, `getScanStats`, `exportScanCsv` | Member ที่มีสิทธิ์ตาม `can()` | จัดการ Dynamic QR |
| `startCheckout`, `cancelSubscription`, `changePlan`, `listPayments`, `downloadReceipt` | Member | จัดการแพ็กเกจและการชำระเงิน |
| `POST /api/webhooks/payment` | gateway (ตรวจ signature) | รับสถานะการชำระเงิน |
| `admin.listReportedQr`, `admin.setDynamicQrStatus` | Admin | จัดการ abuse |
| `admin.upsertPlan`, `admin.upsertCoupon`, `admin.listSubscriptions` | Admin | จัดการแพ็กเกจ |
| `admin.refundPayment` | Super Admin | คืนเงิน |

## 7. Analytics Event Specification (GA4)

| Event | Parameters | เมื่อไร |
| --- | --- | --- |
| `qr_type_select` | `type` | เลือกประเภท QR |
| `qr_generate` | `type` | พรีวิวแรกที่สร้างสำเร็จในแต่ละ session ของหน้า |
| `qr_customize` | `option` (color/logo/shape/ec/size) | เปลี่ยนการตั้งค่า |
| `qr_download` | `type`, `format`, `is_member` | ดาวน์โหลด |
| `qr_copy`, `qr_share` | `type` | คัดลอกหรือแชร์ |
| `qr_save` | `type` | สมาชิกบันทึก QR |
| `sign_up` | `method` (email/google/line) | สมัครสำเร็จ (event มาตรฐาน GA4) |
| `login` | `method` | Login สำเร็จ (event มาตรฐาน GA4) |
| `language_switch` | `to` | สลับภาษา |
| `affiliate_click` | `product_id`, `platform`, `placement`, `qr_type` | คลิกการ์ด Affiliate (นับซ้ำซ้อนกับ TR-103 ได้ แต่ GA ได้เฉพาะผู้ที่ consent) |
| `sponsor_click` | `banner_id`, `slot` | คลิกแบนเนอร์ผู้สนับสนุน |
| `web_vitals` | `metric`, `value`, `ad_variant` | วัด Core Web Vitals ต่อกลุ่ม A/B (TR-113) |
| `view_pricing` | `source` | เปิดหน้า Pricing (Phase 2) |
| `begin_checkout` | `plan`, `interval` | เริ่มชำระเงิน (Phase 2, event มาตรฐาน GA4) |
| `purchase` | `transaction_id`, `value`, `currency=THB`, `plan` | ชำระสำเร็จ ส่งจากฝั่ง server ผ่าน Measurement Protocol หลังได้ webhook (Phase 2) |
| `dynamic_qr_create` | `is_trial` | สร้าง Dynamic QR (Phase 2) |

- event ที่ระบุว่า Phase 2 (`view_pricing`, `begin_checkout`, `purchase`, `dynamic_qr_create`) พักไว้
- ตั้ง user property `user_type` = `guest` หรือ `member` และ `ad_variant` ตาม TR-113
- ถ้าจะใช้ `user_id` ของ GA ให้ใช้ค่า hash (HMAC) ของ user id ภายใน ห้ามใช้อีเมล และส่งเฉพาะเมื่อได้ `analytics_storage = granted`

## 8. ข้อกำหนดที่ไม่ใช่ฟังก์ชัน (Non-functional Requirements)

### 8.1 ประสิทธิภาพ

| รหัส | ข้อกำหนด |
| --- | --- |
| NFR-01 | Core Web Vitals ที่ p75 บนมือถือ: LCP ≤ 2.5 s, INP ≤ 200 ms, CLS ≤ 0.1 |
| NFR-02 | JavaScript ที่โหลดตอนเปิดหน้าแรก (ไม่รวมโฆษณา) ≤ 150 KB หลัง gzip โดยโค้ดของระบบสมาชิกและ CMS ต้องไม่ถูกรวมเข้ามาในหน้าแรก |
| NFR-03 | สร้างพรีวิว QR ขนาด 1024 px ภายใน 100 ms บนมือถือระดับกลาง |
| NFR-04 | คะแนน Lighthouse ด้าน Performance, Accessibility, Best Practices และ SEO ≥ 90 สำหรับหน้าสาธารณะ |
| NFR-05 | API ของสมาชิกและ CMS มี latency p95 ≤ 500 ms (ไม่รวม GA4 Data API) |
| NFR-06 | `/go/*` และ `/r/{code}` ตอบ redirect ภายใน p95 ≤ 100 ms ที่ edge (ไม่รวมเวลาเครือข่ายของผู้ใช้) |
| NFR-07 | เครือข่ายโฆษณาใดที่ทำให้ Web Vitals p75 เกินเกณฑ์ NFR-01 ห้ามได้น้ำหนัก traffic เกิน 20% จนกว่าจะแก้ไขได้ |

### 8.2 ความปลอดภัย

| รหัส | ข้อกำหนด |
| --- | --- |
| NFR-10 | บังคับใช้ HTTPS และตั้ง HSTS (`max-age` ≥ 1 ปี, `includeSubDomains`) |
| NFR-11 | ตั้ง Content Security Policy แบบ nonce อนุญาตเฉพาะโดเมนของ Google Ads/Analytics, Turnstile และ OAuth provider ที่จำเป็น พร้อม `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy` และ `frame-ancestors 'none'` สำหรับ `/admin` |
| NFR-12 | แสดงผลข้อมูลที่ผู้ใช้กรอกเป็นข้อความเท่านั้น ห้ามใช้ `dangerouslySetInnerHTML` กับ input ของผู้ใช้ |
| NFR-13 | สแกน dependency ด้วย `npm audit` หรือ Dependabot ใน CI และห้าม deploy เมื่อพบช่องโหว่ระดับ high ขึ้นไป |
| NFR-14 | เก็บ secret (OAuth client secret, DB URL, encryption key, API key) ใน secret manager หรือ environment variable ของ hosting เท่านั้น ห้าม commit ลง repository และต้องหมุนเวียน key ได้ |
| NFR-15 | ป้องกัน CSRF สำหรับทุก mutation (Server Actions ตรวจ Origin และ auth endpoint ใช้ CSRF token) |
| NFR-16 | ทดสอบตาม OWASP ASVS ระดับ 2 สำหรับส่วน authentication, session management และ access control ก่อนเปิดใช้งานจริง |
| NFR-17 | Log ของแอปห้ามมีรหัสผ่าน token payload ของ QR หรืออีเมลแบบเต็ม (ให้ mask) |
| NFR-18 | `/go/*` redirect ได้เฉพาะ URL ที่เก็บในฐานข้อมูลตาม id เท่านั้น ห้ามรับ URL ปลายทางจาก query string (กัน open redirect) |
| NFR-19 | Phase 2: ระบบไม่สัมผัสข้อมูลบัตร (PCI DSS SAQ A), เก็บ webhook secret และ API key ของ gateway ใน secret manager และมีการแจ้งเตือนเมื่อ webhook ล้มเหลวต่อเนื่อง |

### 8.3 ความเป็นส่วนตัว (PDPA)

| รหัส | ข้อกำหนด |
| --- | --- |
| NFR-20 | ไม่บันทึกหรือส่งเนื้อหา QR และไฟล์โลโก้ของ Guest ออกจากเบราว์เซอร์ |
| NFR-21 | ข้อมูลจากฟอร์ม Contact ใช้เพื่อส่งอีเมลถึงผู้ดูแลเท่านั้น ไม่เก็บลงฐานข้อมูล |
| NFR-22 | Privacy Policy ระบุข้อมูลที่เก็บจากสมาชิก วัตถุประสงค์ ระยะเวลาเก็บ ผู้ประมวลผลข้อมูลภายนอก (Google, LINE, hosting, database, email) และวิธีติดต่อเพื่อใช้สิทธิ์ของเจ้าของข้อมูล |
| NFR-23 | เลือก region ของ database และ object storage ให้สอดคล้องกับเงื่อนไขการโอนข้อมูลไปต่างประเทศตาม PDPA (แนะนำ Singapore หรือ region ที่ที่ปรึกษากฎหมายรับรอง) |
| NFR-24 | backup ของฐานข้อมูลเข้ารหัส เก็บไม่เกิน 30 วัน และข้อมูลที่ hard delete แล้วต้องหายจาก backup ภายในรอบการเก็บนั้น |
| NFR-25 | การนับคลิก Affiliate/แบนเนอร์และสถิติการสแกน Dynamic QR ห้ามเก็บ IP จริงหรือ identifier ที่ใช้ติดตามข้ามวันได้ (ใช้ salt ที่เปลี่ยนรายวันตาม TR-122) |
| NFR-26 | Phase 2: เก็บข้อมูลการชำระเงินและใบเสร็จตามระยะเวลาที่กฎหมายบัญชีและภาษีกำหนด แม้ผู้ใช้จะลบบัญชีแล้ว (ระบุไว้ใน Privacy Policy) |

### 8.4 ความพร้อมใช้งานและการรองรับ

| รหัส | ข้อกำหนด |
| --- | --- |
| NFR-30 | Uptime ≥ 99.5% ต่อเดือน (BNFR-03) ถ้าฐานข้อมูลล่ม หน้า QR Generator สำหรับ Guest ต้องยังใช้งานได้ |
| NFR-31 | รองรับ Chrome, Edge, Firefox และ Safari สองเวอร์ชันล่าสุด รวมถึง Safari บน iOS 16 ขึ้นไป และ Chrome บน Android 10 ขึ้นไป |
| NFR-32 | หน้าเว็บยังสร้างและดาวน์โหลด QR Code ได้แม้สคริปต์โฆษณาโหลดไม่สำเร็จหรือถูก ad blocker บล็อก |
| NFR-33 | ฐานข้อมูลมี automated backup รายวันและ point-in-time recovery โดยมี RPO ≤ 24 ชั่วโมงและ RTO ≤ 4 ชั่วโมง |
| NFR-34 | Phase 2: `/r/{code}` มี uptime ≥ 99.9% ต่อเดือน และยัง redirect ได้จาก Redis แม้ PostgreSQL ล่มชั่วคราว (R-15) |
| NFR-35 | ถ้า `/go/*` นับคลิกไม่สำเร็จ (เช่น Redis ล่ม) ต้องยัง redirect ไปปลายทางได้ |

## 9. โครงสร้างโปรเจกต์ (เสนอ)

```text
qreasy.info.v2/
├─ BRD/                     Business requirements
├─ TRD/                     Technical requirements
├─ v1/                      ข้อมูลระบบเดิม
└─ app/                     (จะสร้างเมื่อเริ่มพัฒนา)
   ├─ src/
   │  ├─ app/
   │  │  ├─ [locale]/       หน้าสาธารณะ, auth, account (แยกภาษา)
   │  │  ├─ admin/          CMS
   │  │  ├─ api/auth/       Auth route handlers
   │  │  └─ ads.txt/        route เสิร์ฟ ads.txt
   │  ├─ components/        UI components (public / account / admin)
   │  ├─ lib/qr/            QR engine, payload builders, exporters (ไม่ขึ้นกับ UI)
   │  ├─ lib/auth/          การตั้งค่า auth, guard ตรวจ role
   │  ├─ lib/db/            schema, migrations, queries
   │  ├─ lib/settings/      อ่าน/แคช site_settings และ ad_slots
   │  ├─ lib/analytics/     GA4 event helpers + consent
   │  ├─ lib/audit/         บันทึก audit log
   │  ├─ server/actions/    Server Actions (member / admin)
   │  └─ messages/          th.json, en.json (รวมเทมเพลตอีเมล)
   ├─ content/              MDX ของหน้า static (th/en)
   ├─ scripts/              seed, create-super-admin
   ├─ public/               favicon, รูปภาพ
   └─ tests/                unit, integration และ e2e
```

## 10. การทดสอบ

| ระดับ | สิ่งที่ทดสอบ |
| --- | --- |
| Unit | payload builder ทุกประเภท (รวมการ escape อักขระ), การตรวจ URL, การคำนวณ contrast, การตรวจรูปแบบ AdSense/GA ID |
| Decode test | สร้าง QR ด้วยการตั้งค่าต่าง ๆ (สี โลโก้ รูปทรงจุด EC) แล้ว decode กลับด้วย `jsQR` หรือ `zxing` และต้องได้ payload ตรงกับต้นฉบับ |
| Integration | Server Actions กับฐานข้อมูลทดสอบ (PostgreSQL ใน container) ครอบคลุมการตรวจสิทธิ์ทุก action: Guest, Member ที่ยังไม่ยืนยันอีเมล, Member, Admin, Super Admin, ผู้ใช้ที่ถูกระงับ |
| Security | ทดสอบ IDOR (เข้าถึง QR ของผู้อื่น), privilege escalation (Admin แก้ role ตัวเอง), rate limit และ session หลังระงับหรือรีเซ็ตรหัสผ่าน |
| E2E | เส้นทางหลัก: (1) Guest สร้างและดาวน์โหลด QR (2) สมัคร → ยืนยันอีเมล → บันทึก QR → เปิดจากหน้า "QR Code ของฉัน" (3) Admin Login + 2FA → ค้นหาผู้ใช้ → ระงับ → ผู้ใช้ Login ไม่ได้ (4) Admin ปิด ad slot แล้วหน้าเว็บไม่แสดงโฆษณาตำแหน่งนั้น (5) คลิกการ์ด Affiliate → นับคลิกแล้ว redirect ไปปลายทางที่ถูกต้อง (6) Phase 2: สมัคร Pro ด้วยบัตรทดสอบ → สร้าง Dynamic QR → แก้ปลายทาง → สแกนแล้วไปปลายทางใหม่ → โฆษณาหายทุกหน้า |
| Ad layout | ตรวจทุกตำแหน่งโฆษณาที่ viewport 360, 768, 1280 px ว่าห่างจากปุ่มดาวน์โหลด/คัดลอก/แชร์ตาม TR-110 และไม่เกิด layout shift |
| Experiment | การสุ่ม variant กระจายตามน้ำหนัก (±2% ที่ 10,000 ตัวอย่าง) และผู้ใช้คนเดิมได้ variant เดิมตลอด |
| Payment (Phase 2) | webhook ซ้ำ, ล่าช้า, signature ผิด, ชำระไม่สำเร็จ, คืนเงิน และการต่ออายุ ใช้ sandbox ของ gateway |
| Load (Phase 2) | `/r/{code}` รับ 500 request/วินาที โดย p95 ไม่เกิน NFR-06 |
| Accessibility | `axe-core` ใน Playwright ต้องไม่พบ violation ระดับ serious ขึ้นไป |
| Manual | สแกน QR ตัวอย่างด้วยแอปกล้องของ iOS และ Android จริง และทดสอบ Login ด้วย Google และ LINE บนมือถือจริงก่อนออกแต่ละ release |
| Performance | Lighthouse CI ใน pipeline ตามเกณฑ์ NFR-04 |

## 11. CI/CD และสภาพแวดล้อม

- ใช้ Git และ GitHub (repository ยังไม่ได้สร้าง)
- Pull Request ทุกครั้งต้องผ่าน lint, type check, unit test, integration test, e2e test และ Lighthouse CI
- Database migration รันอัตโนมัติตอน deploy และต้องเป็นแบบ backward-compatible (expand → migrate → contract)
- มี Preview deployment ต่อ Pull Request โดยใช้ database branch หรือ database แยกสำหรับ preview ส่วน `main` deploy ไปยัง production
- Environment: `development`, `preview`, `production` แยก GA4 property, OAuth client, database และ email sender ทั้งหมด และปิดโฆษณาในทุก environment ที่ไม่ใช่ production

## 12. การติดตามระบบ (Monitoring)

- Uptime monitor ตรวจหน้าแรกและ health check endpoint ทุก 5 นาที
- เก็บ error ทั้งฝั่งเบราว์เซอร์และเซิร์ฟเวอร์ด้วย Sentry หรือบริการเทียบเท่า โดยตัดข้อมูลที่ผู้ใช้กรอกและข้อมูลส่วนบุคคลออกก่อนส่ง
- แจ้งเตือนเมื่อ Login ล้มเหลวผิดปกติ มีการเปลี่ยน role หรือ Admin ส่งออก CSV
- ติดตาม Core Web Vitals จาก Search Console และ GA4
- ติดตามอัตราการส่งอีเมลสำเร็จ (bounce/complaint) จากบริการส่งอีเมล

## 13. การออกแบบเผื่ออนาคต (ไม่เพิ่มงาน)

แพ็กเกจเสียเงิน (Phase 2) พักไว้ แต่แนวทางต่อไปนี้แทบไม่เพิ่มงานใน V2 และช่วยไม่ให้ต้องรื้อระบบถ้ากลับมาทำภายหลัง

- แยก `lib/qr` ให้เป็นอิสระจาก UI
- การตัดสินใจว่าจะแสดงโฆษณาหรือไม่ ให้รวมไว้ในฟังก์ชัน `shouldShowAds(user, route)` ที่เดียว (ใช้กับ TR-94, TR-97 และ TR-105 อยู่แล้ว)
- ตรวจสิทธิ์ผ่าน guard ที่รวมไว้ใน `lib/auth` ไม่กระจายเงื่อนไข role ไปทั่วโค้ด
- จองเส้นทาง `/r/*` และ `/pricing` ไว้ ห้ามใช้กับหน้าอื่น

### 13.1 ส่วนที่ยังไม่อยู่ใน V2

- แพ็กเกจ Pro, Dynamic QR, Bulk และระบบชำระเงิน (หัวข้อ 4.13 พักไว้)
- Public API สำหรับนักพัฒนา
- การจัดการเนื้อหาหน้าเว็บ (About, FAQ, บทความ) ผ่าน CMS
- แพ็กเกจสำหรับทีมหรือองค์กร

## 14. ประเด็นทางเทคนิคที่ต้องตัดสินใจ

| รหัส | คำถาม |
| --- | --- |
| TQ-01 | ใช้ hosting เจ้าไหน (Vercel หรือ container บน Cloud Run / Fly.io) ขึ้นกับงบประมาณใน Q-05 |
| TQ-02 | DNS ของ `qreasy.info` ปัจจุบันจัดการที่ไหน และใครมีสิทธิ์แก้ไข |
| TQ-03 | มีสิทธิ์เข้าถึง GA4 property และบัญชี AdSense เดิมหรือไม่ และสร้าง service account สำหรับ GA4 Data API ได้หรือไม่ |
| TQ-04 | ใช้ Cloudflare Turnstile หรือ reCAPTCHA v3 |
| TQ-05 | บริการส่งอีเมล (Resend, Amazon SES หรืออื่น ๆ) และใช้โดเมนผู้ส่งอะไร (เช่น `no-reply@qreasy.info`) |
| TQ-06 | เลือก CMP สำหรับ Cookie Consent ตัวไหน ต้องเป็นตัวที่ Google รับรองถ้าจะแสดงโฆษณาให้ผู้ใช้ใน EEA/UK |
| TQ-07 | ผู้ให้บริการ PostgreSQL และ region (ดู NFR-23) |
| TQ-08 | ใช้ Better Auth หรือ Auth.js และใช้ Drizzle หรือ Prisma |
| TQ-09 | ใครเป็นผู้ลงทะเบียน OAuth app กับ Google Cloud และ LINE Developers ในนามของเว็บไซต์ |
| TQ-10 | Ezoic มีวิธีเชื่อมต่อหลายแบบ (เช่น ผ่าน nameserver หรือ Cloudflare) แบบใดเข้ากับ hosting และ CSP ที่เลือก หรือจะเลือกพาร์ตเนอร์ Google Ad Manager แทน |
| TQ-11, TQ-12 | _(พักไว้พร้อม Phase 2)_ เลือก payment gateway และ Safe Browsing/Web Risk API |

## 15. ตารางอ้างอิง BR → TR

| BR | TR |
| --- | --- |
| BR-01, BR-02 | TR-01, TR-02 |
| BR-03 | TR-04 |
| BR-04 | TR-05 |
| BR-05 | TR-06 |
| BR-06 | TR-07 |
| BR-07 | TR-08 |
| BR-08 | TR-09 |
| BR-10 – BR-15 | TR-10 – TR-15 |
| BR-20 – BR-24 | TR-20 – TR-24 |
| BR-30 – BR-35 | TR-30 – TR-36 |
| BR-40 – BR-44 | TR-03, TR-42 – TR-45, NFR-20 |
| BR-50, BR-51 | TR-50 – TR-52 |
| BR-60 | TR-60, TR-61 |
| BR-61 | TR-62 |
| BR-62 | TR-63 |
| BR-63 | TR-64, TR-65 |
| BR-64 | TR-70 – TR-73, TR-75 |
| BR-65 | TR-74 |
| BR-66 | TR-31, TR-75 |
| BR-67 | TR-76 |
| BR-68 | TR-77 |
| BR-69 | TR-68 |
| BR-70 | TR-66, TR-67 |
| BR-71 | TR-69 |
| BR-75 | TR-80 |
| BR-76 | TR-82 |
| BR-77 | TR-83 |
| BR-78 | TR-84 |
| BR-79 | TR-85 |
| BR-80 | TR-86 |
| BR-81 | TR-87 |
| BR-82 | TR-88 |
| BR-83 | TR-89, TR-98 |
| BR-84 | TR-90 |
| BR-85 | TR-81 |
| BR-86, BR-87 | TR-91, TR-92 |
| BR-88 | TR-93 |
| BR-89 | TR-94 |
| BR-90 | TR-52, TR-95 |
| BR-91 | TR-43, หัวข้อ 7 |
| BR-92 | TR-96 |
| BR-93 | TR-97 |
| BR-100 | TR-100 |
| BR-101 | TR-101 |
| BR-102 | TR-102 |
| BR-103 | TR-103, TR-104 |
| BR-104 | TR-105 |
| BR-105 | TR-106 |
| BR-106 | TR-107 |
| BR-110 | TR-110 |
| BR-111 | TR-111 |
| BR-112 | TR-112 |
| BR-113 | TR-113, TR-114 |
| BR-114 | TR-115 |
| BR-115 | TR-116 |
| BR-120 | TR-120, TR-121 |
| BR-121 | TR-122, TR-123 |
| BR-122 | TR-124 |
| BR-123 | TR-125, TR-126 |
| BR-124 | TR-127 |
| BR-125 | TR-128 |
| BR-126 | TR-129 |
| BR-127 | TR-130 |
| BR-128 | TR-131 |
| BR-129 | TR-121, TR-132 |
| BR-130 | TR-121, TR-133 |
| BR-131 | TR-134 |
| BR-132 | TR-135 |
| BR-133 | TR-124, TR-136 |
