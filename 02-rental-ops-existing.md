# ก้อนสอง — แอป Rental Ops ที่มีอยู่จริง (หลังบ้าน DNA EV)

> แผนที่ของระบบที่ใช้งานจริงอยู่วันนี้ ตรวจจากโค้ดจริง 7 กันยายน 2569
> ไม่มีข้อมูลส่วนบุคคล ไม่มีคีย์ลับ ไม่มีตัวเลขเงินจริง — มีแต่โครงสร้าง

---

## 1. ภาพรวมเทคนิค

| เรื่อง | ของจริง |
|---|---|
| เฟรมเวิร์ก | Next.js 16 + React 19 (TypeScript) |
| โฮสต์ | Cloudflare Worker ตัวเดียว · โดเมน `app.dnaev.com` |
| ฐานข้อมูล | Cloudflare **D1** (SQLite) ผ่าน drizzle-orm · **R2** เก็บไฟล์ (สลิป เอกสาร) |
| ล็อกอิน | อีเมล + รหัส OTP ผ่าน Supabase · API ทุกเส้นตรวจ Bearer token เท่านั้น |
| งานตามเวลา | `scheduled()` ของ Worker (cron) — ตัวปลุกสรุปเช้า + งานใกล้ถึง |
| เทสต์ | 339 เทสต์ 33 ไฟล์ (สูตรเงิน สิทธิ์ สำรอง/กู้ฐาน สายพาน LINE) |
| deploy | สคริปต์ wrangler · ทดสอบในเครื่องกับ D1 จริงก่อน deploy ทุกครั้ง |
| สำรอง | GitHub private + bundle + export/import ฐานทั้งก้อน |
| **ไม่มี** | LLM/OCR ใด ๆ · สร้าง PDF/สัญญา · PWA/แอปเนทีฟ · ตารางงานแยก |

## 2. หน้าจอที่มี

**แอปทีม `/`** — หน้าเดียว 10 แท็บ (client component ไฟล์เดียว ~2,600 บรรทัด):

| แท็บ | ทำอะไร |
|---|---|
| `today` | คิวรถวันนี้ งานรับ–ส่ง ปุ่มปล่อยรถ/ปิดคืน |
| `calendar` | ปฏิทินรถ×วัน กด 2 จังหวะเลือกช่วง → ไปหน้าจองพร้อมคันนั้น |
| `booking` | สร้าง/แก้ใบจอง (ใหญ่สุด) · **วางข้อความสรุป Booking จาก LINE แล้วกรอกให้อัตโนมัติ** · ค้นลูกค้าเก่าเติมให้ · เลือกคันจริงจากรุ่น/สี/ทะเบียน · แถบขั้นบันไดราคา |
| `line` | กล่องร่างใบจองที่บอทเลขาส่งมา ทีมรีวิวแล้วแปลงเป็นใบจอง |
| `fleet` | คลังรถ รูป สี สถานะ ราคาเริ่มต้น |
| `customers` | ลูกค้าเดิม ประวัติ ยอดค้าง/เกิน |
| `finance` | กำไร-ขาดทุนรายเดือน รายการเงิน ค่างวด ส่วนแบ่ง |
| `deposits` | คิวคืนเงินประกัน หัก/คืน |
| `history` | ประวัติใบจอง |
| `team` | สิทธิ์ทีม อนุมัติคนล็อกอินใหม่ |

**หน้าสาธารณะ** (ไม่ต้องล็อกอิน มีการ์ดของตัวเองทุกเส้น):
`/book` ลูกค้าเช็ครถว่าง+ราคา แล้วกด “ขอจอง” · `/book/status/[token]` ดูสถานะคำขอ · `/confirm/[token]` ลูกค้ากดยืนยันสรุป Booking · `/app-demo` เดโม่ฝั่งลูกค้า

ลูกค้าเห็นรถเป็นรหัส **“DNA n”** ไม่เห็นทะเบียน ไม่เห็นชื่อเจ้าของรถ (ตัดที่ระดับ API)

## 3. API ทั้งหมด (`app/api/`)

| เส้น | หน้าที่ | ใครเรียก |
|---|---|---|
| `auth/otp` `auth/verify` `auth/refresh` `auth/status` `session` | ล็อกอิน OTP + สถานะผู้ใช้ | แอปทีม |
| `bookings` | **GET** รายการ · **POST** สร้างใบจอง (~40 ช่อง) · **PATCH** ทำรายการตาม `action` | แอปทีม · สคริปต์ภายนอก |
| `booking-requests` | คำขอจองจากเว็บลูกค้า — **ประตูเดียวที่คนนอกเขียนฐานได้** มี rate limit | `/book` |
| `booking-confirmation` | ลูกค้ากดยืนยันสรุป (token เก็บเป็น hash) | `/confirm/[token]` |
| `vehicles` | รถ + วันว่าง + ราคา (สาธารณะ ไม่มีทะเบียน/ชื่อลูกค้า เพดานช่วง 400 วัน) | `/book` · แอปทีม |
| `cars` | เปลี่ยนสี/รูปรถ (`set_appearance` เท่านั้น — **ไม่มีทางสร้างรถใหม่จากหน้าจอ**) | แอปทีม |
| `customers` | ค้นลูกค้าเดิมด้วยเบอร์/ชื่อ → ข้อมูลครบ + ใบล่าสุด + ประวัติ 10 ใบ (คืน PII หลังการ์ดสิทธิ์) | แอปทีม |
| `finance` | กำไร-ขาดทุนรายเดือน | แท็บ finance |
| `deposits` | คิวคืนเงินประกัน | แท็บ deposits |
| `evidence` | **POST** อัปโหลดไฟล์ 1 ไฟล์/ครั้ง → R2 · **GET ?file=1** ดึงไฟล์ · หมวด: สลิป, `id_card`, `driver_license`, `address_proof`, `customer_with_car`, `refund_account` | แอปทีม |
| `staff` | รายชื่อคนรับ–ส่งรถ (`operations_staff` แยกจากบัญชีล็อกอิน) | แอปทีม |
| `team` | สิทธิ์ทีม อนุมัติ login | เจ้าของ |
| `line/webhook` `line/drafts` `line/groups` `line/reminders` `line/status` | รับ event LINE (ตรวจ HMAC) · ร่างใบจอง · กลุ่มหลัก · เตือน | LINE · แอปทีม |
| `notifications` | กล่องเตือนในแอป (GET/PATCH) | แอปทีม |
| `integrations/secretary` | **Bridge ให้บอทเลขา LINE** — GET อ่าน (view: รถว่าง/ประวัติ/งาน/สถิติคนทัก/ตัวอย่างสรุปเช้า/ประวัติ cron) · POST รับร่าง Booking (เขียนได้เฉพาะร่าง + เตือน) | บอทเลขา |
| `admin/export` `admin/import` | สำรอง/กู้ฐานทั้งก้อน (token แยก) | เจ้าของ |

### POST /api/bookings รับช่องเหล่านี้
ลูกค้า: `customer phone lineId nickname birthDate idNumber license licenseExpiry docAddress emergencyName emergencyPhone`
รถ/เวลา: `carId startDate startTime endDate endTime`
เงิน: `customRate originalRate discountReason rentalTotal bookingDepositRequired bookingDepositReceived deposit ownerShare companyFeePercent paymentDueAt` + ค่าธรรมเนียม/ต้นทุนย่อย (`delivery_fee pickup_fee paid_amount delivery_labor delivery_travel pickup_labor pickup_travel cleaning_cost charging_cost other_cost`)
งาน: `pickupBy dropoffBy pickupLocation returnLocation contactChannel usageArea notes` · บัญชีคืนเงิน: `refundBankName refundAccountName refundAccountNumber`
สถานะเริ่มต้น: `confirmed` ถ้าทีมสร้างและมัดจำครบ · ไม่งั้น `pending`

### สถานะใบจองและการเปลี่ยนสถานะ
`pending → confirmed → active → completed` (+ `returned` · `cancelled` · `rejected`)
ผ่าน `PATCH /api/bookings` ด้วย `action`: `update_booking` · `update_financial_summary` · `prepare_confirmation` (ส่งสรุปให้ลูกค้ากดยืนยัน) · `release_vehicle` (→ `active`, ตั้ง `pickup_done`) · `complete_return` (→ `completed`, ตั้ง `dropoff_done`) · `cancel_booking`

**ตัวกันคิวชน** = SQL ตรงในไฟล์ใบจอง 2 จุด (ตอนสร้าง + ตอนแก้วัน): `car_id เดียวกัน AND status ไม่ใช่ cancelled/rejected AND ช่วงเวลาซ้อน` — ไม่ได้อยู่ใน lib/ · ไฟล์ปฏิทิน (`car-availability.ts`) เป็นแค่ตัวช่วยแสดงผล ไม่ใช่ตัวตัดสิน

## 4. ฐานข้อมูล — 23 ตาราง

| กลุ่ม | ตาราง | หมายเหตุ |
|---|---|---|
| หลัก | `cars` `customers` `bookings` `booking_confirmations` `booking_requests` `booking_fees` `booking_upload_tokens` `car_blocks` | `car_blocks` = บล็อกรถ (ซ่อม/ห้ามจอง) |
| เงิน | `transactions` `car_costs` `car_installment_plans` `pricing_tiers` `financial_evidence` | `pricing_tiers` = ขั้นบันไดราคา 5 ช่วงต่อคัน · `car_costs` kind=installment คือค่างวดรถ |
| ทีม | `team_members` `team_login_requests` `operations_staff` | คนรับ–ส่งรถแยกจากบัญชีล็อกอิน |
| LINE | `line_booking_drafts` `line_groups` `line_daily_contacts` `line_reminder_log` | `line_daily_contacts` นับคนทัก ไม่เก็บชื่อ/ข้อความ (hash) |
| ระบบ | `notifications` `audit_log` `scheduled_run_log` | `audit_log.actor_email` NOT NULL |

**`cars`**: `model licensePlate color dailyRate ownerSharePct ownerName isActive rangeKm seats bodyType photoUrl` — `licensePlate` มี unique index
**`customers`**: `name phone lineId nickname birthDate idNumber licenseNumber licenseExpiry address emergencyName emergencyPhone usualArea notes`
**`bookings`** (คอลัมน์สำคัญ): `customerId carId startAt endAt pricePerDay originalRate totalPrice bookingDepositRequired bookingDepositReceived deposit companyFeePercent ownerShare status pickupBy dropoffBy pickupDone dropoffDone pickupLocation returnLocation paymentDueAt refundDueAt refundBank* usageArea notes createdBy`
— `pickupBy/dropoffBy` เก็บ**ชื่อคนเป็นข้อความ** ไม่ใช่ id · งานรับ–ส่งมีแค่ 2 ช่องติ๊กนี้ ไม่มีตารางงาน

## 5. สูตรเงิน — บ้านเดียวใน `lib/` (กติกาเหล็ก: หน้าจอ/API ห้ามคิดซ้ำ)

| เรื่อง | ไฟล์ | ฟังก์ชันหลัก |
|---|---|---|
| กำไร/รายรับ/จ่ายแล้ว ต่อใบ + รายเดือน | `booking-profit.ts` | `bookingMoneyBreakdown()` `monthlyProfitScore()` |
| เงินฝั่งลูกค้า (ค้าง/เกิน) | `customer-money.ts` | `customerMoneySummary()` `outstandingBreakdown()` |
| ราคาเช่าตามขั้นบันได + นับวัน + ราคาบนการ์ด | `rental-pricing.ts` | `tierRateForDays()` `rentalDayCount()` `carCardPrice()` |
| เหมาเดือน / แปลงข้อความราคา | `monthly-rate.ts` | `flatMonthlyQuote()` `parseMoneyText()` |
| ค่าบริการนามบริษัท | `company-fee.ts` | `companyFeeAmount()` |
| ส่วนแบ่งเจ้าของรถ | `partner-share.ts` | `partnerProfitSplit()` `resolveOwnerShare()` |
| เงินประกัน หัก/คืน | `security-deposit.ts` `deposit-refund.ts` | `securityDepositSettlement()` `planDepositRefund()` |
| การ์ดกดโอนทีละก้อน | `payment-milestones.ts` | `paymentMilestones()` |
| วันนัดจ่าย + ลำดับตามเงิน | `payment-due.ts` | `normalizePaymentDueAt()` `paymentChaseRank()` |
| ค่างวดตกเดือนไหน | `secretary-history.ts` | `plannedInstallmentSchedule()` |
| วันว่าง/ปฏิทิน (แสดงผล) | `car-availability.ts` | `dayAvailabilityState()` `rangeIsBookable()` `monthGrid()` |
| จับคู่รถกับคันจริง | `booking-car-match.ts` | ทะเบียน → รหัสรถ → รุ่น+สี → รุ่น |
| **เอกสารไทย** | `thai-documents.ts` | `isValidThaiIdNumber()` `toGregorianDate()` (กัน พ.ศ.) **`extractDocumentFields()` `reviewScannedDocument()` — เขียนแล้ว ยังไม่มีใครเรียก** |
| แกะข้อความ LINE เป็นร่างใบจอง | `line-booking.ts` | `parseLineBookingSummary()` `bookingMissing()` |
| สมองตัวปลุก (สูตรล้วน) | `daily-briefing.ts` | `buildMorningBriefing()` `buildDueSoonAlert()` `jobsDueWithin()` `unpaidNeedingChase()` |
| ตัวปลุกต่อฐาน+LINE | `scheduled-briefing.ts` | `runScheduledBriefing(mode, now, dryRun)` |

ไฟล์อื่นใน lib/: `booking-confirmation` `booking-request` `imported-transactions` `integration-auth` `line-*` `owner-notifications` `secretary-*` `team-removal` `vehicle-identity`

## 6. สิทธิ์

- `getAccess(request)` ใน `db/access.ts` คือด่านของทุก API · ตัวตนจาก **Bearer token ตรวจกับ Supabase เท่านั้น** ห้ามเชื่อ header อื่น
- บทบาท `owner` / `staff` / `viewer` / `public` + ธง `canViewFinance` `canEditFinance` `canManageBookings` (→ `canWriteOperations`)
- staff ใหม่ต้องมี `team_login_requests` ที่เจ้าของกด `approved`
- **route สาธารณะทุกเส้นต้องขึ้นทะเบียน `PUBLIC_ROUTES` พร้อมการ์ดของตัวเอง** — เทสต์สแกนทุก route บังคับ
- PDPA ตัดที่**ระดับ API** ไม่ใช่ซ่อนบนจอ

## 7. สายพาน LINE + บอทเลขา + ตัวปลุก

```
LINE OA ──webhook──▶ บอทเลขา (Vercel + Neon + OpenRouter/Gemini Flash)  ← สมองคนละตัว
                          │ Bridge: GET/POST /api/integrations/secretary (Bearer + X-Line-* headers)
                          ▼
                     Rental Ops (source of truth)
                          │ line_booking_drafts → ทีมรีวิวในแท็บ line → แปลงเป็นใบจอง
                          │
     cron ใน Worker ──▶ ตัวปลุก: 07:00 สรุปเช้า · 08–20 ทุกชั่วโมง งานใกล้ถึง ──▶ LINE กลุ่มทีม (ทางเดียว)
                          จดผลทุกรอบลง scheduled_run_log
```

- LINE ให้ webhook URL เดียวต่อบอท และมันชี้ไปบอทเลขา — แอปไม่เห็น event เอง
- โควตา LINE แผนฟรี **300 ข้อความ/เดือน** เลขากับตัวปลุกใช้ก้อนเดียวกัน → ตัวปลุกรวมทุกอย่างเป็นข้อความเดียวต่อรอบ
- Bridge สิทธิ์ 3 ระดับ: เจ้าของ (เต็ม) → กลุ่มทีม (หลังบ้าน) → ลูกค้า (รถว่าง ราคา เท่านั้น)
- `notifications` ในแอป: มีตาราง แต่ส่วนใหญ่**คำนวณสดตอนเปิด** ไม่ได้บันทึกจริง (บันทึกจริงเฉพาะ `owner_change` `line_booking` `booking` `release`)

## 8. เครื่องมือที่อยู่นอกแอป (ต้องรู้ตอนวางแผนรวม)

**สัญญาเช่า** — สคริปต์ Python 4 ตัวบนเครื่อง Mac (แยกตามรุ่น: EX2 · EX2-นิติบุคคล · ORA5 · DEEPAL)
- กรอกลง PDF เทมเพลต 4 หน้าตามพิกัดที่กำหนดแล้ว (pdfplumber + pypdf + Chrome headless) → PDF ปิดแก้ไข
- ตรวจเอง: `check_money()` สมการยอดเงิน (หักมัดจำ + ประกัน + ค่าส่ง/รับ + เพิ่มเติม) · `check_plate()` ทะเบียนต้องมีจังหวัด
- มี `ค้นลูกค้า.py` อ่านตาราง customers ของแอป และ `ส่งเข้าแอป.py` ยิง `POST /api/bookings` สร้างใบ `pending`
- **ข้อจำกัด: ต้องเปิด Mac ถึงออกสัญญาได้**

**บอทเลขา LINE** — โปรเจกต์แยก (Vercel + Neon) ใช้ Gemini 2.5 Flash ผ่าน OpenRouter · มีตัวอ่านรูปอยู่แล้ว (เป็นตัวที่เคยอ่านทะเบียนผิดแต่ตอบมั่นใจ) · ถูก prompt สั่ง**ห้ามอ่านเลขบัตร**โดยตั้งใจ · ห้ามย้ายโฮสต์

## 9. บทเรียนที่ระบบใหม่ต้องเคารพ (เคยพลาดจริงทุกข้อ)

1. **“ช่องหลอก”** — ช่องบนหน้าจอที่ไม่ได้ผูกกับคอลัมน์จริง ทีมกรอกแล้วหายเงียบ (เคยเป็นกับ nickname, license_expiry, วันนัดจ่าย) → ทุกช่องต้องพิสูจน์ว่าลงฐานและดึงกลับได้
2. **เลขเดียวกันสองหน้าจอต้องมาจากฟังก์ชันเดียว** → แก้กติกาเงินแล้วต้องยิงทุกเส้นที่โชว์เลขนั้นมาเทียบ
3. **ห้ามข้อมูลปลอม/hardcode บนหน้าจอ** — ยังไม่มีของจริงให้ขึ้น “—”
4. **PDPA ตัดที่ API** ไม่ใช่ซ่อนบนจอ · ลูกค้าเห็นแค่ “DNA n”
5. **บิลด์เขียว ≠ ใช้ได้** — ต้องรันจริงกับฐานจริงในเครื่องก่อน deploy · **เทสต์ครอบสูตรจับบั๊ก SQL ไม่ได้**
6. **ค่าหลอกในคอลัมน์ที่มี FOREIGN KEY ใช้ไม่ได้** (ตัวกันยิงซ้ำเคยพังเงียบด้วยเหตุนี้)
7. **“ไม่มีบันทึก” ≠ “ไม่ได้เกิดขึ้น”** — ระบบที่กลืน error ทุกด่าน debug ไม่ได้
8. **ทีมกรอกวันที่เป็น พ.ศ.** — ทุกทางเข้าต้องแปลง
9. **ราคามี 3 แหล่ง** (ฐาน `pricing_tiers` · ตารางที่แอดมินใช้ตอบลูกค้า · เอกสารแผน) — แก้ที่หนึ่งต้องเตือนอีกสอง
10. **โมเดลเล็กอ่านรูปผิดแต่มั่นใจ** — ต้องมีด่านตรวจนอกโมเดลเสมอ

## 10. ช่องว่างที่รู้แล้ววันนี้

- ไม่มีทางสร้าง**รถคันใหม่**จากหน้าจอ (ใส่ฐานตรงมาตลอด)
- งานรับ–ส่งไม่มี “รับงาน” รายคน · ไม่มีบทบาทคนขับ
- `audit_log` เขียนด้วย SQL ซ้ำกันใน 11 route ไม่มีฟังก์ชันกลาง
- ตัวกันคิวชนอยู่ในไฟล์ route ไม่ใช่ lib/
- กล่องเตือนในแอปส่วนใหญ่ไม่บันทึกจริง
- ไม่มี OCR/LLM/PDF ในแอปเลย
