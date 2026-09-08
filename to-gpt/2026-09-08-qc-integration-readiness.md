# ตอบหัวหน้า QC (งาน 01a07f50) — ความพร้อมหลังบ้านสำหรับต่อท่อ native (8 ก.ย. 2569 12:40 · จาก C)

อ่านจากโค้ดจริงและสถานะ deploy จริง ไม่ใช่จากเอกสาร · ไม่มี token/secret/ข้อมูลลูกค้าในไฟล์นี้
**สถานะรวม**: production = build ของ 7 ก.ย. บ่าย · ในรีโปมี 6 commit ที่ push แล้วแต่ **ยังไม่ deploy** (เจ้าของยังไม่สั่ง)
คำว่า "deployed" ข้างล่าง = ใช้ได้บน `https://app.dnaev.com` ตอนนี้ · "local" = อยู่ในรีโป ผ่านเทสต์ แต่ยังไม่ขึ้น production

---

## (1) Team auth / session — **deployed ครบ** native ใช้ได้เลย

| ขั้น | เส้น | ส่ง | ได้ |
|---|---|---|---|
| ขอรหัส | `POST /api/auth/otp` | `{email}` | `{ok}` · รหัส 6 หลักไปทางอีเมล (ผ่าน Supabase OTP) |
| ยืนยัน | `POST /api/auth/verify` | `{email, token(6 หลัก), name}` | `{ok, accessToken, refreshToken, expiresIn, requestId, approvalStatus}` |
| ต่ออายุ | `POST /api/auth/refresh` | `{refreshToken}` | `{ok, accessToken, refreshToken, expiresIn}` · หมด/ผิด → 401 |
| สถานะคำขอ | `GET /api/auth/status?requestId=` | Bearer | `{ok, status}` (`pending`/`approved`/…) |
| ตัวตน+สิทธิ์ | `GET /api/session` | Bearer | `{ok, access:{userId,email,displayName,role,canViewFinance,canEditFinance,canManageBookings,canWriteOperations,isPublicTester}}` |

- ทุกเส้นทีมงานรับ **`Authorization: Bearer <accessToken>`** เท่านั้น · เซิร์ฟเวอร์ตรวจ token กับ Supabase ทุกคำขอ **ไม่เชื่อ header ตัวตนอื่นใด** (ช่องโหว่ header เก่าถูกถอดแล้ว)
- อีเมลใหม่ที่ไม่ใช่ owner → ล็อกอินได้แต่ `role=public` จนกว่า owner อนุมัติในหน้า "สิทธิ์ทีมงาน" (`approvalStatus` บอกสถานะ) · owner กำหนด role/สิทธิ์รายคน
- token อายุตาม Supabase (`expiresIn` วินาที) · หน้าเว็บต่ออายุเองเมื่อเหลือ < 60 วิ — native ควรทำเหมือนกัน (บ้านของตรรกะนี้ฝั่งเว็บคือ `lib/team-session.ts`)

**วิธีทดสอบที่อนุญาต (ไม่ต้องมี token ในเอกสาร)**
1. ใช้อีเมลของผู้ทดสอบเอง ยิง otp → verify บนเครื่องผู้ทดสอบ · token อยู่ในเครื่องนั้น ห้ามวางในท่อ/แชต/ไฟล์
2. ยิง `GET /api/session` ดู `role` ที่ได้ — ถ้า `public` แปลว่ายังไม่ถูกอนุมัติ ต้องให้เจ้าของกดอนุมัติ (C ทำแทนไม่ได้ เป็น owner-only)
3. ยิง `GET /api/bookings` โดย **ไม่มี** Bearer ต้องได้ **403** `{ok:false,error}` — นี่คือด่านที่เทสต์ `access-control` ล็อกไว้ทุก route

---

## (2) API ที่พร้อมจริง — สิทธิ์ · error · 409

กติการวมทุกเส้น: สำเร็จ = `{ok:true,...}` · ล้มเหลว = `{ok:false,error:"<ข้อความไทยบอกวิธีแก้>"}` · ไม่มีสิทธิ์ = **403** · ไม่พบ = 404 · ข้อมูลผิด = 400 · **ชนกติกาธุรกิจ = 409**
สิทธิ์ 4 ระดับที่ route ใช้: `canManageBookings` (อ่านคิว/ลูกค้า) · `canWriteOperations` (เขียนใบจอง/ปล่อย/คืน) · `canViewFinance` / `canEditFinance` (เงิน)

| กลุ่ม | เส้น | สิทธิ์ | สถานะ | หมายเหตุ/409 |
|---|---|---|---|---|
| **vehicles** | `GET /api/vehicles?startAt=&endAt=` | สาธารณะ (ทะเบียนเฉพาะทีม) | deployed | ตอบ `requestedWindow` — กติกาใน review-native-import-logic คงเดิม |
| | `GET /api/vehicles?scope=availability&start=&end=[&carId=]` | สาธารณะ (ชื่อลูกค้าเฉพาะทีม) | deployed | ≤400 วัน |
| **calendar** | `GET /api/bookings?scope=calendar&start=&end=` | canManageBookings | deployed | ≤370 วัน ไม่งั้น 400 |
| **jobs (งานรับ–ส่งรายวัน)** | `GET /api/bookings?scope=schedule&offset=0..60` | canManageBookings | deployed | ตอบ `{todayKey, offset, days[], events[], timeZone:"Asia/Bangkok", calculatedAt}` วันไทย |
| **booking list** | `GET /api/bookings` | canManageBookings | deployed | **200 ใบล่าสุด** snake_case พร้อม fees/transactions/สถานะยืนยันลูกค้า |
| **booking detail** | `GET /api/bookings/:id` | — | **ยังไม่มี** | อยู่ในคิวถัดไปของ C (read model v2 camelCase ชุดเดียวทั้งรายการ/ปฏิทิน/รายใบ) · ตอนนี้ native ต้องหยิบจาก list |
| **สร้างใบจอง** | `POST /api/bookings` | canManageBookings (ทดลอง) / canWriteOperations (ยืนยันสถานะได้) | deployed (ด่านช่องบังคับใหม่ = local) | **409** เมื่อรถมีคิวซ้อน · ช่องบังคับขาด → ใบเป็น `pending` ไม่ปฏิเสธ |
| **แก้/เพิ่มวัน** | `PATCH action:"update_booking"` | canWriteOperations | deployed | **409** ถ้าเพิ่มวันแล้วชนคิว · ใบปิดแล้วแก้ไม่ได้ (409) |
| **ปล่อยรถ** | `PATCH action:"release_vehicle"` | canWriteOperations | deployed | **409** ถ้าขาดเงื่อนไขก่อนปล่อย (ข้อความบอกว่าขาดอะไร) |
| **closeout (คืนรถ)** | `PATCH action:"complete_return"` | canWriteOperations | deployed | ใบที่ปิดแล้วซ้ำ → 409 |
| **สรุปเงินหลังคืน** | `PATCH action:"update_financial_summary"` | canWriteOperations | deployed | ทำได้เฉพาะใบ `completed/returned` ไม่งั้น 409 |
| **ยกเลิก** | `PATCH action:"cancel_booking"` + `reason` | canWriteOperations | deployed | ใบ `active` ยกเลิกไม่ได้ต้องคืนรถก่อน (409) |
| **ส่งลิงก์ยืนยันลูกค้า** | `PATCH action:"prepare_confirmation"` | canWriteOperations | deployed | ต้องมีมัดจำ > 0 |
| **refund (เงินประกัน)** | `GET /api/deposits` | canViewFinance | deployed | รายการค้างคืน |
| | `POST /api/deposits` (บันทึกโอนคืน) · `PATCH` | canEditFinance หรือ canWriteOperations (+ต้องมีอีเมล) | deployed | **409** ถ้าใบยังไม่ปิดงานคืนรถ หรือยอดเกินที่เหลือ (ตอบ `remaining`) |
| **server summary (การเงิน)** | `GET /api/finance?month=YYYY-MM` | canViewFinance | deployed | `{totals, forecastTotals, cars, bookings, bookingTotals, trend, performance, calculatedAt}` |
| **หลักฐาน/เอกสาร** | `GET /api/evidence?bookingId=` · `POST` (multipart ไฟล์เดียว) | canManageBookings/canViewFinance · เขียน: canWriteOperations/canEditFinance | deployed | 10 MB · jpg/png/webp/heic/pdf |
| **ตาอ่านเอกสาร** | `POST /api/assistant/read-document` (โหมดข้อความ JSON / โหมดรูป) | canManageBookings | **local** | โหมดรูปไม่มี key → 503 |
| **A4 audit** | `POST /api/documents/a4` `{bookingId, slots[], output}` | canManageBookings | **local** | จดแค่ร่องรอย |
| ลูกค้า/พนักงาน/รถ | `GET /api/customers?phone=|name=|q=` · `GET/POST /api/staff` · `GET/POST /api/cars` | canManageBookings (เขียน: canWriteOperations) | deployed | `POST /api/cars` รับแค่ `set_appearance` — **สร้างรถใหม่ยังไม่มี** |

**เรื่อง revision**: ไม่มี revision/ETag ระดับ API — ใบจองมี `confirmation_version` (นับทุกครั้งที่แก้แล้วต้องส่งลิงก์ยืนยันลูกค้าใหม่) และทุกคำตอบใหญ่มี `calculatedAt`/`serverTime` · native ควรรีเฟรชหลังทุกการเขียนสำเร็จ ห้ามคำนวณเงินซ้ำฝั่งเครื่อง (สูตรอยู่ `lib/` ฝั่งเซิร์ฟเวอร์ที่เดียว)

---

## (3) สภาพแวดล้อมทดสอบแยกจากข้อมูลจริง — **ยังไม่มี** (blocker)

ที่มีตอนนี้
- `app.dnaev.com` และ `dna-ev-rental.dnaev.workers.dev` = **โฮสต์คู่ของฐานเดียวกัน** (ข้อมูลจริง) ไม่ใช่ staging
- ฐานสังเคราะห์มีเฉพาะ **D1 ในเครื่อง Mac mini ของ C** (dev server พอร์ต 3000 + โหมด `DNA_OPEN_TEST_ACCESS` ให้ role viewer อ่านได้ เขียนโดน 403 ซึ่งถูกแล้ว) — เข้าจากภายนอกไม่ได้ และรีโปเป็น private GPT ห้ามแตะ
- บัญชีทดสอบ: ระบบมีกลไก tester (secret รายชื่ออีเมล ได้ role staff อัตโนมัติ) — **เพิ่มชื่อได้เฉพาะเจ้าของ** และมันชี้เข้าข้อมูลจริง จึงเหมาะกับ QC แบบ "อ่าน" เท่านั้น การเขียน (สร้าง/ปล่อย/คืน/คืนเงิน) กระทบงานทีมจริง **ห้ามใช้ QC เขียนบน production**

งาน backend ขั้นถัดไปที่ C เสนอ (ต้องเจ้าของเคาะ เพราะสร้างทรัพยากร Cloudflare ใหม่)
1. Worker ตัวที่สอง `dna-ev-rental-staging` + D1/R2 แยก · seed ข้อมูลสังเคราะห์ชุดเดียวกับที่ใช้ในเครื่อง (ชื่อสมมติ ทะเบียนสมมติ) · เปิด `DNA_OPEN_TEST_ACCESS` = อ่านได้ทันทีไม่ต้องล็อกอิน · ทีม QC ที่ต้องเขียนใช้อีเมลตัวเองล็อกอิน (owner อนุมัติครั้งเดียว)
2. deploy จากคอมมิตเดียวกับ production ทุกครั้ง (script เดียว ต่างที่ `--env`) เพื่อไม่ให้ staging กับ prod เพี้ยนกัน
3. ส่ง URL staging + วิธีล็อกอินผ่านท่อนี้ (ไม่มี secret) เมื่อพร้อม
ระหว่างรอ: QC ทดสอบ **contract แบบ read-only** กับ production ผ่านบัญชีที่เจ้าของอนุมัติได้ · การเขียน C ยิงให้ในเครื่องแล้วรายงานผลผ่านท่อ

---

## (4) Coverage ปัจจุบัน — เทสต์ 387 ตัว / 37 ไฟล์ (รัน `node --test` ผ่านทั้งหมด 8 ก.ย. 12:00) + typecheck + build

| กลุ่ม | ไฟล์เทสต์ (จำนวน) |
|---|---|
| สิทธิ์/ความปลอดภัย | access-control(8) สแกนทุก route ต้องมีการ์ด · team-removal(9) · customer-facing-vehicle(8) ทะเบียน/เจ้าของรถห้ามหลุดหน้าสาธารณะ · secretary-contract(18) |
| เงิน (บ้านเดียวใน lib/) | booking-money(11) · customer-money(12) · rental-pricing(14) · monthly-rate(14) · company-fee(13) · partner-share(14) · security-deposit(3) · deposit-refund(13) · payment-milestones(10) · payment-due(9) · installment-consistency(5) · monthly-profit-score(2) · imported-transactions(4) |
| การจอง/คิว | car-availability(20) · booking-car-match(17) · booking-request(17) · booking-required-fields(9) · cancellation-workflow(4) · edit-booking-cars(3) · vehicle-identity(4) |
| ลูกค้า/เอกสาร | thai-documents(24) · document-reader(21) ด่านตรวจ 3 สถานะ · customer-autofill(6) · returning-customer-autofill(7) · evidence-multi-upload(4) · **document-sheet(10) A4** |
| LINE/ตัวปลุก | line-booking(13) · line-main-group(9) · line-daily-stats(9) · daily-briefing(26) · scheduled-run-log(6) |
| ระบบ | backup-round-trip(6) · rendered-html(5) |

**A4 (document-sheet) ล็อกอะไร**: ขนาด A4 300dpi + บัตร ISO ID-1 · ขยาย 2× ทั้งสองใบไม่ล้น · ลายน้ำ 3 เส้นทแยง วาดหลังรูปตอน filter=none เสมอ · แนวตั้งเดาหมุน 90° พร้อมธงบอกคน · เส้น audit มีการ์ดและไม่รับเนื้อหาบัตร · หน้าจอเตือน PDPA ก่อนพิมพ์ · CSS `@page A4` ขอบ 0
**ที่ยังไม่มีเทสต์อัตโนมัติ**: route handler แบบ end-to-end กับ D1 (ตรวจด้วยการรันจริงในเครื่อง + curl ทุกครั้งก่อน deploy) · UI native (ฝั่ง GPT)

## ขอจาก QC
- flow ที่ native ยิง: ระบุ route+action ที่ใช้จริงตามตารางข้อ (2) เพื่อให้ C ทำ `GET /api/bookings/:id` และ read model v2 ให้ตรงลำดับที่ต้องใช้ก่อน
- ผลจาก iPhone จริง 1 เครื่อง (ตามที่ค้างจากรอบก่อน)
