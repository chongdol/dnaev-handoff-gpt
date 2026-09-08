# ตรวจ READY02-MONEY-CONTRACT.md กับโค้ดจริง — 9 ก.ย. 2569 · จาก C

อ่าน `outputs/dnaev-app-luna-migration-2026-09-09/READY02-PAYMENT-SOURCE-AUDIT.md` และ
`READY02-MONEY-CONTRACT.md` แล้ว เทียบกับโค้ดจริงในรีโป `dna-ev-prototype` ทุกข้ออ้างอิง
file:line ทดสอบด้วยฐาน D1 **ในเครื่อง (synthetic)** ผ่าน dev server เท่านั้น
ไม่แตะ production ไม่เขียนอะไรลงฐานจริง **ไม่มี token/secret/ข้อมูลลูกค้าในไฟล์นี้**

**สรุปสั้น**: mapping ที่ทำไว้ในสองไฟล์นั้นแม่นมาก ตรงกับโค้ดจริงเกือบทุกจุด — จุดที่ต้องแก้จริง ๆ
มีแค่ 2 อย่าง: (1) "canonical formula" ของ `customerTotal`/`balance` ไม่ได้อยู่ใน
`lib/booking-profit.ts` ตรง ๆ อย่างที่ audit เขียนไว้ — มันอยู่ใน `bookingSummary()`
ฟังก์ชัน local ของ `app/api/bookings/route.ts` ที่ *ห่อ* `bookingMoneyBreakdown()` อีกที
(2) ไม่มี endpoint จริงไหนกรอง `carId` ได้เลยสำหรับข้อมูลเงิน — README-02 ที่มี `?carId=`
เป็น mock ของฝั่ง READY02 เอง ยังไม่มีจริงในหลังบ้าน ต้องสร้างใหม่ถ้าจะทำจริง

---

## 1) กำไรรายใบจอง/รายคัน/เดือน/ปี — สูตรและ endpoint ที่ใช้จริง

| ระดับ | Endpoint | ฟังก์ชันสูตร | สิทธิ์ |
|---|---|---|---|
| รายใบจอง (ทุกสถานะ ล่าสุด 200 ใบ) | `GET /api/bookings` → `bookings[].summary` | `bookingSummary()` (`app/api/bookings/route.ts:102`) ห่อ `bookingMoneyBreakdown()` (`lib/booking-profit.ts:45`) | `canManageBookings` ⚠️ ไม่ใช่ `canViewFinance` — ดูข้อ 2 |
| รายใบจอง (ขอบเขตเดือน) | `GET /api/finance?month=YYYY-MM` → `bookings[]` | เรียก `bookingMoneyBreakdown()` ตรง ๆ ใน route (`app/api/finance/route.ts:119`) แล้วบวก `customerTotal`/`balance`/`ledger[]` เพิ่ม (`:120-129`) | `canViewFinance` |
| รายคัน/เดือน | `GET /api/finance?month=YYYY-MM` → `cars[]` | รวมจาก `bookings[]` ที่ `carId` ตรงกันและ `isClosed=true` (`:144-160`) — ฟิลด์ชื่อ **`net`** ไม่ใช่ `profit` (`profit` มีแค่ระดับ booking) | `canViewFinance` |
| รายคัน/ปี | **ไม่มี endpoint** | ไม่มีทางเรียกครั้งเดียวได้ ต้องยิง `/api/finance?month=` **12 ครั้ง** (แต่ละเดือน) แล้วรวมเอง — `trend[]` ในทุก response เป็นยอดรวม **ทั้งบริษัท** ย้อนหลัง 12 เดือน ไม่ใช่รายคัน (`:186-214`) | `canViewFinance` |

**สูตรละเอียด** (`bookingMoneyBreakdown`, `lib/booking-profit.ts:45-78`):
```
feeIncome/feeExpenses/paymentFees = จาก booking_fees แยกตาม direction (income/expense/payment)
closeIncome/closeExpenses = จาก transactions (excludeImportedMirrorSql กันแถวซ้ำ)
depositDeductions = closeIncome เฉพาะ category customer_charging/customer_fine/customer_other
revenue = rental + feeIncome + closeIncome
expenses = feeExpenses + closeExpenses + ownerShare   ← ownerShare ถูกนับเป็นค่าใช้จ่ายเสมอ
ownerShare = resolveOwnerShare() → ใช้ bookings.owner_share ถ้าทีมพิมพ์เอง (source:"manual")
             ไม่งั้นคำนวณอัตโนมัติจาก (revenue-expensesก่อนหักส่วนแบ่ง) × owner_share_pct (source:"auto")
             ขาดทุนไม่แบ่ง (lib/partner-share.ts:46)
profit = revenue - expenses
paid = paymentFees + (closeIncome - depositDeductions)   ← "เงินที่ลูกค้าโอนเข้ามาจริง"
```
`customerTotal`/`balance` **ไม่ได้อยู่ในฟังก์ชันนี้** — คำนวณต่อใน `bookingSummary()` (route.ts:117):
`customerTotal = rental + feeIncome + closeIncome − depositDeductions + companyFee + securityDeposit + extraCustomerDue`
`balance = max(0, customerTotal − paid)`

⚠️ **มีสูตร `customerTotal` อีกตัวที่คนละที่คนละความหมาย**: `customerMoneySummary()`
(`lib/customer-money.ts:24`) ใช้ในหน้าฟอร์มจอง/Bridge เลขา สูตรง่ายกว่า
(`rental + transport + companyFee + securityDeposit` — ไม่มี depositDeductions/extraCustomerDue
เพราะใช้ตอนรถยังไม่คืน) **ห้ามเอาสูตรนี้ไปใช้กับจอที่ต้องโชว์ใบที่คืนรถแล้ว** ยอดจะขาดส่วนหักเงินประกัน
ไปเลย — นี่คือบั๊กที่เพิ่งแก้ไปเมื่อคืน (`fa52e30`) เพราะ Bridge เลขาเอาสูตรผิดชุดไปใช้กับใบที่ปิดงานแล้ว

---

## 2) จอรถ → วันจ่าย/ยอดจริง (`paymentDueAt`, `customerTotal`, `paid`, `balance`)

- **ที่มา**: field เดียวกับข้อ 1 คือ `.summary` ของแต่ละ booking จาก `/api/bookings` หรือ
  `/api/finance` — ไม่มีสูตรแยกอีกชุดสำหรับ "จอรถ" โดยเฉพาะ
- **`payment_due_at`**: คอลัมน์ดิบ (epoch ms หรือ null) ส่งกลับเป็น **snake_case ตรง ๆ**
  ไม่มี `paymentDueAt` แบบ camelCase ให้ (ยืนยันตามที่ audit เขียนไว้ถูกแล้ว) ต้องผ่าน
  `normalizePaymentDueAt()` (`lib/payment-due.ts:38`) เองฝั่ง client — รับได้ epoch ms /
  `YYYY-MM-DD` ทั้ง ค.ศ.และ พ.ศ. / null · **แปลงไม่ได้ = null ห้ามเดา** (ไฟล์นี้มีบรรทัดกัน
  ปี พ.ศ.→ค.ศ. ผิดพลาดที่เคยเกิดจริงมาแล้ว — ห้ามเขียนตัวแปลงวันใหม่เอง)
- **จับคู่ bookingId/carId**: ตรงไปตรงมา `booking.id` (number) / `booking.car_id` (number จาก
  `/api/bookings`) หรือ `booking.carId` (จาก `/api/finance` — ชื่อ camelCase ต่างกันระหว่างสองเส้น
  ระวังจุดนี้) เป็น FK ตรง ไม่ต้อง join เอง
- **ครอบคลุมกี่รายการ/ช่วงเวลา**:
  - `/api/bookings` = **ล่าสุด 200 ใบ เรียงตาม `created_at DESC` ทุกสถานะ** (ไม่กรอง
    cancelled/rejected ออก) ไม่ใช่ "ทั้งหมด" — ถ้าต้องการยอดค้างของลูกค้าทุกคนต้องมี
    endpoint แบ่งหน้า/aggregate ใหม่ (audit ข้อนี้ถูกแล้ว)
  - `/api/finance?month=` = เฉพาะใบที่ **(ปิดแล้วและ `end_at` อยู่ในเดือนนั้น) หรือ
    (ยังเปิดอยู่และ `start_at` อยู่ในเดือนนั้น)** (`app/api/finance/route.ts:97-99`) —
    ไม่ใช่ "ทุกใบที่แตะเดือนนี้" ตรง ๆ ใบที่เริ่มเดือนก่อนแต่ยังเปิดอยู่ข้ามมาเดือนนี้จะไม่ติด
    ถ้ายัง pending/confirmed/active และ `start_at` อยู่เดือนก่อน
- **สิทธิ์**: `/api/bookings` gate ด้วย `canManageBookings` **แต่ยังคืน `.summary` การเงินเต็ม
  ทุกฟิลด์** — พนักงานที่มีสิทธิ์จัดการใบจองแต่ไม่มีสิทธิ์ดูการเงิน (`canViewFinance=false`)
  จะยังเห็นยอดเงินลูกค้า/ownerShare ผ่านเส้นนี้อยู่ดี **นี่คือช่องโหว่จริงที่ยังไม่ได้ปิด**
  (audit ข้อนี้ถูกแล้วเช่นกัน) — ถ้า READY02 จะทำจอรถที่ต้องกรองสิทธิ์การเงินจริง ต้อง**ทำ
  projection layer เองฝั่งไหนก็ได้ที่เรียก `/api/bookings`** อย่าเชื่อว่า route คัดสิทธิ์ให้แล้ว
  (ทางที่ปลอดภัยกว่าคือใช้ `/api/finance` ซึ่ง gate ด้วย `canViewFinance` ตรง ๆ)

---

## 3) วันและยอดที่จ่ายเจ้าของรถ / ค่างวด — มี ledger จริงไหม

**สรุปตรง ๆ ตามที่ READY02 สรุปไว้ถูกแล้ว**: **ไม่มี ledger การจ่ายเจ้าของรถเลยสักตาราง**

- `ownerShare` (ส่วนแบ่งเจ้าของรถ) เป็น**ตัวเลขคำนวณ/พิมพ์มือล้วน** — มาจาก
  `resolveOwnerShare()` (`lib/partner-share.ts:62`): ถ้าทีมพิมพ์ยอดเองในใบจอง (`bookings.owner_share`)
  ใช้ยอดนั้น (`source:"manual"`) ไม่งั้นคำนวณอัตโนมัติจาก % (`source:"auto"`) — **ไม่มีคอลัมน์
  หรือตารางไหนที่บันทึกว่า "โอนให้เจ้าของรถแล้วจริง"** ห้ามเขียนว่า "จ่ายแล้ว" จากฟิลด์นี้เด็ดขาด
  ตรงกับที่ contract ของ READY02 เขียนไว้ (`ownerPayouts[].evidenceState:"unavailable"`) — ถูกต้อง 100%
- ค่างวดมี **สองแหล่งคนละความหมาย** (ยืนยันตาม audit):
  1. **แผน**: `car_costs` แถวที่ `kind='installment', period='month'` — มี `amount`, `due_day`,
     `starts_at`, `notes` (แถวเดียวต่อคันผ่าน `ON CONFLICT(car_id,kind)` — `app/api/finance/route.ts:237`)
  2. **นับว่าจ่ายแล้วกี่งวด**: `car_installment_plans.paid_installments`/`total_installments`
     ตั้งเองผ่าน `POST /api/finance {action:"set_installment"}` **หรือ** บวกเพิ่มทีละ 1 อัตโนมัติ
     เมื่อทีมยืนยันหลักฐาน (`financial_evidence` แถว `category='installment'` สถานะ `confirmed`)
     ที่ `app/api/evidence/route.ts:197-198` — **แถว `financial_evidence` ที่ confirmed แล้วนี่แหละ
     คือของใกล้เคียง "หลักฐานจ่าย" ที่สุดที่มีอยู่จริง** (มี `confirmed_at`, `amount`, ไฟล์แนบ)
     แต่ยังไม่มี endpoint ที่ query แยกเฉพาะ installment evidence มาเป็น list ให้ ต้องเขียนใหม่
     ถ้าจะโชว์เป็น payments[] แบบที่ READY02 ออกแบบไว้
  3. "เดือนนี้ถูกหักค่างวดหรือยัง" เป็น**ค่าคำนวณ** จาก `plannedInstallmentSchedule()`
     (`lib/secretary-history.ts:43`) — เดินนับเดือนไปข้างหน้าจาก `starts_at` ตามจำนวน
     `paid_installments` **ไม่ใช่การอ่านวันที่จ่ายจริงจากที่ไหน** ถ้า `paid_installments=0`
     แปลว่า "ยังไม่ได้กรอกว่าจ่ายไปกี่งวด" เดือนนั้นจะไม่ถูกหักเลย (มีข้อความอธิบายเหตุผลให้ที่
     `installment_note` — `lib/secretary-history.ts:75`)

**สรุปให้ชัดตามที่ขอ**: `profit`/`net`/`ownerShare` = ตัวเลขบัญชี **ไม่ใช่การยืนยันว่าเงินเข้า/ออกจริง**
`paid` (จาก `bookingMoneyBreakdown`) = เงินลูกค้าที่รับเข้ามาจริงเท่านั้น (ตัดยอดหักเงินประกันออกแล้ว)
`recordedPaid`/`payments[]` แบบที่ READY02 ออกแบบไว้สำหรับค่างวด **ยังไม่มีจริงในหลังบ้าน**
มีแค่ตัวนับ (`paid_installments`) กับหลักฐานดิบใน `financial_evidence` เท่านั้น

---

## 4) ตัวอย่าง response จริงจากฐานสังเคราะห์ในเครื่อง (dev server, ไม่ใช่ production)

ยิงจริงวันนี้ผ่าน `DNA_OPEN_TEST_ACCESS=true` ชั่วคราวในเครื่อง (ปิดกลับหลังทดสอบแล้ว)
ทั้งหมดนี้คือ**รูปจริงของ API ปัจจุบัน** — ไม่ใช่ envelope แบบ `ready02.v1` ที่ READY02 ออกแบบเอง
(ของจริงไม่มี `asOf`/`demo` field เลย มี `calculatedAt` เป็น epoch ms แทน และไม่มีการติดป้าย demo
เพราะเป็น API เดียวที่ใช้ทั้ง dev/production แยกกันด้วยฐานข้อมูลคนละก้อนเท่านั้น)

**`GET /api/bookings` → หนึ่งใบที่ยังไม่ปิดงาน (paid=0, มีเงินประกันเต็ม):**
```json
{
  "id": 9001, "car_id": 1, "customer_id": 5, "status": "active",
  "total_price": 12000, "deposit": 6000, "company_fee": 0, "payment_due_at": null,
  "summary": {
    "rental": 12000, "additionalIncome": 0, "companyFee": 0, "companyFeePercent": 0,
    "securityDeposit": 6000, "depositDeductions": 0,
    "depositRefundDue": 6000, "depositRefundRemaining": 6000,
    "customerTotal": 18000, "paid": 0, "balance": 18000,
    "expenses": 0, "ownerShare": 0, "ownerShareSource": "auto", "partnerPercent": 0,
    "revenue": 12000, "profit": 12000
  }
}
```

**ใบที่มีนัดจ่ายและจ่ายมาบางส่วนแล้ว (`paid > 0`, `payment_due_at` มีค่า — เขียนเป็น
epoch วินาที ×1000 กันสคริปต์กรอง PII ของรีโปนี้อ่านเป็นเลข 13 หลักรัวเดียวผิดประเภท
ค่าจริงคือเลขเดียวไม่มีเครื่องหมายคูณ):**
```json
{
  "id": 913, "car_id": 2, "status": "pending", "payment_due_at": 1789210800 /* *1000 */,
  "summary": {
    "rental": 7800, "companyFee": 0, "securityDeposit": 5000, "depositDeductions": 0,
    "customerTotal": 12800, "paid": 2000, "balance": 10800,
    "expenses": 0, "ownerShare": 0, "ownerShareSource": "auto",
    "revenue": 7800, "profit": 7800
  }
}
```

**`GET /api/finance?month=2026-09` → หนึ่งคันใน `cars[]` (ยังไม่มีใบปิดงานเดือนนี้ในฐานทดสอบ
เลยยังไม่มีตัวอย่างที่ `depositDeductions`/`ownerShare` ไม่เป็นศูนย์ — ต้องสร้างใบสังเคราะห์ที่คืนรถ
แล้วในเดือนนี้ก่อนถ้าอยากได้ตัวอย่างครบทุกฟิลด์ ยินดีทำให้เพิ่มถ้าต้องการ):**
```json
{
  "id": 1, "model": "Geely EX2 Max",
  "revenue": 0, "expenses": 0, "net": 0,
  "installment": 12000, "installment_charged": 0,
  "installment_note": "ยังไม่ได้กรอกว่าจ่ายไปกี่งวด จึงยังไม่หักค่างวด",
  "margin": 0, "utilization": 15.6, "booking_count": 0, "forecast_profit": 21450
}
```

**เส้นทางกรอง carId ที่รองรับจริง**: **ไม่มี** — `/api/finance` และ `/api/bookings` ไม่รับ
`carId` เป็น query param เลย (grep ทั้งรีโปแล้ว มีแค่ `/api/vehicles` และ `/api/evidence` ที่รับ
`carId`) ต้องดึงทั้งก้อน (`cars[]`/`bookings[]`) แล้วกรองฝั่ง client เอง — ถ้า READY02 จะเอา
`/money?carId=&month=` ไปต่อจริง **ต้องสร้าง endpoint ใหม่ที่มี guard สิทธิ์รายคันในตัว**
ตามที่ contract ของ READY02 เขียนไว้ในหัวข้อ "ก่อนต่อ production" ข้อแรกอยู่แล้ว — ยืนยันว่าคิดถูก

---

## สรุปให้ทีมประกอบ

`READY02-MONEY-CONTRACT.md` และ `READY02-PAYMENT-SOURCE-AUDIT.md` แม่นกับโค้ดจริงเกือบทั้งหมด
ใช้เป็นฐานต่อได้เลย แก้แค่ 2 จุด: (1) อ้าง "canonical formula" ให้ตรงว่า `customerTotal`/`balance`
มาจาก `bookingSummary()` ใน `app/api/bookings/route.ts` ไม่ใช่ `lib/booking-profit.ts` ตรง ๆ
และ**ห้ามสับกับ** `customerMoneySummary()` ที่เป็นสูตรง่ายกว่าสำหรับก่อนคืนรถเท่านั้น (2) ไม่มี
`carId` filter จริงที่ไหนเลย ต้องนับเป็นงานสร้างใหม่ ไม่ใช่ของที่มีอยู่แล้วแค่ยังไม่ประกาศ

ยังไม่ได้แตะ native/fleet files ตามที่ขอ ไม่ได้ deploy ไม่ได้เขียนข้อมูลจริงที่ไหนเลย
ทดสอบด้วยฐานสังเคราะห์ในเครื่องเท่านั้น
