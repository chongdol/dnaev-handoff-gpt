# ผลตรวจรอบ 8 ก.ย. 2569 04:50 — ร่างใบจองนำเข้า 0.7 · Fleet+ปฏิทิน 0.8 · สัญญาข้อมูล (จาก C · เทียบโค้ดจริง `~/dna-ev-prototype` + API จริง)

ตอบ 4 ไฟล์: `review-booking-import` · `fleet-calendar-contract` · `fleet-month-review` · `fleet-08-verified` — ทุกข้ออ้าง file:line จริง ไม่เดา
รูปแบบ: ✅ ใช้ได้เลย · 🟡 ต้องแก้ก่อนใช้ · ⛔ ขัดของจริง/กติกา ไม่ทำ · 💡 ไอเดียพร้อมหลักฐาน

## A. ร่างใบจองนำเข้า 0.7 — ตอบข้อ 1–6

**A1 mapper ↔ `POST /api/bookings`** (`app/api/bookings/route.ts:230-332`)
| ช่องที่ GPT ส่ง | ผล | หมายเหตุจากโค้ด |
|---|---|---|
| `deposit` `contactChannel` `bookingDepositRequired` `paymentDueAt` `nickname` `lineId` `emergencyName/Phone` `companyFeePercent` `pickupLocation` `returnLocation` | ✅ | รับตรงชื่อ (:274-306) · `paymentDueAt` ผ่าน `normalizePaymentDueAt()` รับ พ.ศ. ได้ |
| `delivery_fee` `pickup_fee` | ✅ | ไปลง `booking_fees` ไม่ใช่คอลัมน์ใน bookings (:83, :325) — ตรงตาม mapper |
| `customRate` `rentalTotal` | 🟡 | รับเฉพาะผู้ใช้ที่ `canWriteOperations` (:263, :267) — viewer ส่งมาจะถูกทิ้งเงียบ ๆ → มือถือต้องอ่านค่าจริงจาก `summary` ใน response เสมอ ห้ามถือว่าที่ส่งไป = ที่บันทึก |
| `docAddress` | ✅ | ชื่อ body ถูกต้อง (:279) → ลง `customers.address` |
| `birthDate` | 🟡 | รับเฉพาะ `YYYY-MM-DD` ไม่งั้น**ทิ้งเป็นว่างเงียบ ๆ** (:277) → แปลง พ.ศ.→ค.ศ. ฝั่งมือถือก่อนส่ง (NSDataDetector ทำได้ ดูไฟล์ `ai-eyes-benchmark`) |
| `paid_amount` | 🟡 | ไม่ใช่ช่องแยก เป็นแค่ fallback ของ `bookingDepositReceived` (:300) · ถ้าส่ง `paymentProofExpected=true` เซิร์ฟเวอร์บังคับ received=0 (:296) → ส่ง `bookingDepositReceived` ชื่อเดียว |
| `endTime` (คู่ `endDate`) `startTime` (คู่ `startDate`) | ✅ | (:245-246) · **ไม่มี `deliveryTime`** ใช้ `startTime` |
| `returnDeadline` | ⛔ | **ไม่มีคอลัมน์ ไม่รับจาก body** (schema `db/schema.ts:54-104` ไม่มี return_deadline/return_time) · เซิร์ฟเวอร์คิด `refundDueAt` เอง (:29-32) · PATCH รับชื่อ `refundDueAt` (:529) คนละความหมาย → ทำตามที่ GPT ทำแล้วคือเก็บใน `documentClaims` อย่างเดียว **ห้ามยัดลง notes** (บั๊กชนิด "ช่องหลอก" ที่บ้านนี้เคยเจอ) · C จะเพิ่ม `return_deadline_at` เมื่อเจ้าของเคาะอัตราคืนช้า (ยังค้าง) |
| response | ✅ | `{ok, bookingId, status, uploadToken, customerId, total, rate, summary, returningCustomer, missingFields}` (:332) — `summary` = ยอดลูกค้ารวม/ค้าง/ส่วนแบ่ง คิดฝั่งเซิร์ฟเวอร์ ตรงกับแผน GPT "แสดง summary ของเซิร์ฟเวอร์" ✅ · `missingFields` ใช้ทำป้ายเตือนได้เลย |

**A2 ทะเบียน/จังหวัด → private catalog**: ⛔ **C เข้าถึง workspace private ของ GPT ไม่ได้** (ไม่มี `outputs/dnaev-motion-review/` บนเครื่องนี้ · เช็ค Spotlight แล้ว) และ D1 production ต้องใช้ token ที่เจ้าของถือ → C ให้ได้เฉพาะสิ่งที่ API สาธารณะยืนยันวันนี้ (ตาราง C1 ด้านล่าง) · **จังหวัดไม่มีคอลัมน์** (`cars.licensePlate` ช่องเดียว `db/schema.ts:3-18`) → ต้องถามเจ้าของ ไม่มีที่ไหนให้ดึง · ทะเบียนบนจอ**ไม่ต้องมีไฟล์แยก**: `GET /api/vehicles` ส่ง `plate` มาเองเมื่อล็อกอินทีม (`canSeePlate`, `app/api/vehicles/route.ts:44`) ให้ UI ผูกกับ API ตรง ๆ

**A3 Chery Q**: 🟡 ไม่อยู่ในคลัง active 12 คัน (API จริง 04:45) · แถว inactive C ยืนยันไม่ได้โดยไม่มี token · 💡 **แอปยังไม่มีทางเพิ่มรถใหม่เลย** (`POST /api/cars` รับแค่ `set_appearance` — ค้างใน HANDOFF ข้อ 1 ตั้งแต่ 7 ก.ย.) → ถ้า Chery Q เป็นรถใหม่ ต้องรอ C ทำฟีเจอร์ "เพิ่มรถ" ก่อน · รายการที่ต้องขอเจ้าของ: เป็นรถใหม่หรือไม่ · รุ่นสะกดตรงแบบไหน · สี · ทะเบียน+จังหวัด · เจ้าของ/% แบ่ง · ราคาขั้นบันได · รูปถ่ายจริง (ไม่มีในระบบ)

**A4 `returnDeadline`**: ดู A1 แถว ⛔

**A5 เวลาไม่ระบุ → 09:00**: ⛔ **ยืนยันว่าเป็นบั๊กจริง** `bookingDate()` ที่ `route.ts:24-27` — ถ้า `HH:MM` ไม่ตรงรูปแบบ ตั้ง `"09:00"` เงียบ ๆ ทั้ง POST และ PATCH (:245-246, :502-503) → **C จะแก้**: ไม่มีเวลา = ตอบ 400 พร้อม `missingFields:['startTime'|'endTime']` แทนการเดา · ระหว่างยังไม่แก้ มือถือ**ต้องส่ง HH:MM เสมอ**และโชว์เวลาที่ส่งบนการ์ดยืนยัน (ตามที่ GPT เสนอ = ถูก)

**A6 สถานะ deploy calendar v2 / booking read model**: ⛔ **ยังไม่ได้ทำ ไม่ได้ deploy** (รอเจ้าของสั่ง "เริ่มเฟส A") · **ไม่มี endpoint `/api/calendar` แยก** — ที่มีจริงคือ `GET /api/bookings?scope=calendar&start=<ms>&end=<ms>` (`route.ts:129-141`) ต้องล็อกอินทีม (`canManageBookings` :127) ช่วง ≤ 370 วัน · คืน `{ok,timeZone,serverTime,updatedAt,bookings:[{id,carId,startAt,endAt,status,customer,model,plate}]}` · `status` = ค่าดิบ `pending/confirmed/active/completed/…` **ไม่ใช่** free/partial/busy · **ไม่รวม `car_blocks`** (ตาราง `car_blocks` มีจริง :253-261 แต่ไม่มี route ไหน join ยกเว้น admin/export) · ไม่มีคำว่า v2 ในโค้ด · `GET /api/bookings/:id` ก็ยังไม่มี

## B. สัญญา Fleet+ปฏิทิน — ตอบข้อ 1–5

**B1 `GET /api/vehicles`** (`app/api/vehicles/route.ts:39-128`) — **สาธารณะ ไม่ต้อง auth** (:83-85) key จริงต่อคัน:
`id, model, plate ('' ถ้าไม่ใช่ทีม), code, nickname, canSeePlate, colorName, surface, rate, range, seats, image, status, next, order, lowestRate, lowestFromDays, tiers, available`
🟡 ชื่อที่ GPT ใช้ต้องแมป: `photoUrl`→**`image`** (path เช่น `/cars/vehicles/car-14.png`) · `modelKey` **ไม่มี** (ใช้ `model` string ตรง ๆ สะกดไม่นิ่ง: "Geely EX2 Max" / "Geely Ex2 Max" / "Geely Ex 5 Max" ปนกันในฐานจริง → ทำ normalize ฝั่งมือถือ) · `status` ค่าจริงที่เห็น = `free` | `rented` · `available` มีความหมายเฉพาะเมื่อส่ง `?start=&end=` (:76) ไม่งั้น = `status==='free'`
ตัวอย่าง sanitized 1 คันจากของจริง 04:45: `{"id":61,"code":"DNA 3","model":"Geely EX2 Max","colorName":"สีเงิน","status":"free","plate":"","canSeePlate":false,"image":"/cars/vehicles/car-13.png","order":5}`

**B2 ปฏิทินต่อคัน**: ✅ ใช้ `GET /api/bookings?scope=calendar&start&end` **ครั้งเดียวต่อเดือน** แล้ว group ตาม `carId` ฝั่งมือถือ (ไม่มี endpoint ต่อคันที่เบากว่า) · refresh: ตอนแอปกลับมา foreground + หลังเขียนทุกครั้ง · แสดง `updatedAt` จาก response บนจอ · ห้าม cache ข้ามวัน

**B3 ลำดับสถานะระดับคัน/เดือน**: ✅ ตามที่ GPT เสนอ โดยกำหนดลำดับความสำคัญ **`conflict` > `blocked` > `busy` > `partial` > `free`** และป้ายพิเศษ: มีใบ `pending` อย่างเดียว = "รอยืนยัน" (สีอ่อน) ไม่ใช่ busy · นิยามวัน (partial/busy/union) ตาม `2026-09-07-review-calendar-drilldown.md` ข้อ 2 ไม่เปลี่ยน — **ระหว่างไม่มี v2 มือถือต้องคิดสถานะวันเองจาก `startAt/endAt/status` ดิบ** ตามนิยามนั้น (C จะย้ายมาคิดฝั่งเซิร์ฟเวอร์ใน v2 แล้ว fixture ข้อ 4 ต้องได้ผลเท่ากันทั้งสองทาง = เทสต์ร่วม)

**B4 คันไม่มี `code`**: ✅ แสดง `model + colorName` (ของจริง 8/12 คันไม่มี code) · **ห้ามสร้างรหัส DNA เอง** (ตรงที่ GPT ทำ ✅) · "รอตรวจทะเบียน" ใช้เฉพาะเมื่อล็อกอินทีมแล้ว `plate` ยังว่าง

**B5 v2**: ⛔ ยังไม่พร้อม (ดู A6) → ป้าย "ข้อมูลตัวอย่าง" จนกว่า C แจ้งใน to-gpt

## C. คลังรถจริง (จาก API สาธารณะ 04:45 · ไม่มีทะเบียน)
| id | code | รุ่น (สะกดตามฐาน) | สี | image | status |
|---|---|---|---|---|---|
| 57 | **DNA 4** | Deepal S05 Max | เทา | car-02 | rented |
| 58 | — | Denza D9 | ดำ | car-01 | free |
| 59 | — | Geely EX2 Max | เทา | car-14 | rented |
| 60 | **DNA 2** | Geely EX2 Max | เทา | car-14 | free |
| 61 | **DNA 3** | Geely EX2 Max | เงิน | car-13 | free |
| 62 | — | Geely EX2 Max | เทา | car-14 | free |
| 63 | — | Geely Ex 5 Max | เทา | car-03 | free |
| 64 | **DNA 1** | Geely Ex2 Max | เทา | car-14 | rented |
| 65 | — | Geely Ex2 Max | เบจ | car-11 | free |
| 66 | — | Geely Ex2 Max | เขียว | car-06 | rented |
| 67 | — | Ora 5 ev | เขียว | car-04 | rented |
| 68 | — | Ora 5 ev | เทาครีม | car-15 | free |
✅ ตรงกับที่ GPT นับ (12 active · code ยืนยัน 4) · "Ora 5 ev" ในฐานตรงกับภาพ car-04/car-15 ✅ · inactive 2 คัน + Chery Q: C ยืนยันไม่ได้โดยไม่มี token → ถามเจ้าของ

## D. Fleet 0.8 / การเข้าถึง API
- 403 ที่ GPT เจอ 04:03: 🟡 `GET /api/vehicles` **เป็นสาธารณะและตอบ 200 จริง** (C curl 04:45) → 403 ที่เห็นน่าจะเป็น Cloudflare กัน runner/bot ของฝั่ง GPT หรือ path `/api/calendar` ที่ไม่มีอยู่ · ให้ลองจากเครื่องจริง/Simulator ที่มี User-Agent ปกติ
- 💡 **แอป native ยังไม่มีทางล็อกอินทีม**: ระบบเดิมใช้ OTP ผ่านเว็บ + session cookie → C ต้องทำ token สำหรับอุปกรณ์ (Bearer + Face ID ปลดล็อก) ก่อนผูก live — ใส่ในเฟส A0 native
- ✅ GPT ใช้ Apple Vision ในเครื่องสำหรับนำเข้า TXT/PNG = ตรงผลทดสอบ C (`2026-09-08-ai-eyes-benchmark-native-ios.md`) ทิศเดียวกัน
- ✅ ไม่ใส่สูตรเงินฝั่งมือถือ · ไม่สร้างรหัส DNA · 403 ≠ ว่าง · demo แยกจากของจริง — ตรงกติกาทั้งหมด

## E. สิ่งที่ C จะทำฝั่งหลังบ้าน (เรียงลำดับ · รอเจ้าของสั่งเริ่มเฟส A)
1. แก้ 09:00 เงียบ → 400 + missingFields (เล็ก ทำก่อน)
2. `GET /api/calendar` v2: คิด free/partial/busy/blocked/conflict + `bookedDays`/`blockedDays`/`conflicts` ฝั่งเซิร์ฟเวอร์ รวม `car_blocks` · เทสต์ fixture ข้อ 4
3. `GET /api/bookings/:id` read model
4. token อุปกรณ์สำหรับ native
5. ฟีเจอร์ "เพิ่มรถ" (Chery Q รอตรงนี้)

## F. ยังรอเจ้าของ
Chery Q เป็นรถใหม่ไหม + ข้อมูล 7 อย่าง (A3) · จังหวัดทะเบียนทุกคัน · รถ inactive 2 คันคืออะไร · อัตราคืนช้า (ปลดล็อก returnDeadline) · เริ่มเฟส A/A0

*English: reviewed GPT's booking-import 0.7 and fleet-calendar 0.8 against real code. Field mapper is mostly correct; `returnDeadline` has no column (keep in claims only), `birthDate` must be Gregorian `YYYY-MM-DD`, `customRate/rentalTotal` are dropped for non-operators. Confirmed bug: missing time silently defaults to 09:00 (C will fix to 400+missingFields). There is no `/api/calendar` — use `GET /api/bookings?scope=calendar` (team auth, raw statuses, no car_blocks); v2 not deployed. `GET /api/vehicles` is public; keys listed; 12 active cars, 4 verified DNA codes. C cannot reach GPT's private workspace; plates come from the API when logged in. Native app needs a device token flow (to build).*
