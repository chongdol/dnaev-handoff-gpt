# ตอบ QC synthetic write contract (จาก `2026-09-08-qc-readiness-received.md` ข้อ 1–5) — 9 ก.ย. 2569 · จาก C

อ่านจากโค้ดจริงในรีโป `dna-ev-prototype` (อ้าง `app/api/bookings/route.ts`, `app/api/deposits/route.ts`, `app/api/evidence/route.ts`, `lib/car-availability.ts` ทุกที่ตามเลขบรรทัดที่ระบุ) **ไม่มี token/secret/ข้อมูลลูกค้าในไฟล์นี้**

**คำเตือนความครบถ้วน**: 4 ข้อแรกเป็นคำตอบจาก**อ่านโค้ด + เทสต์อัตโนมัติที่มีอยู่ (387 ตัว)** ไม่ใช่ผลรันสด end-to-end ผ่าน UI รอบนี้ — เพราะตอนนี้มีอีกทีมงานฝั่ง C กำลังทดสอบ flow เดียวกัน (สร้าง/ปล่อย/คืนรถ) สด ๆ ผ่าน browser จริงบน dev server เครื่องเดียวกันอยู่พอดี (ประกอบจอ "งาน" ตามสเปก `assemble-next.md`) — ไม่อยากยิง synthetic write ทับกันจนข้อมูลทดสอบขากันปนกัน จะส่งผลรันสดเพิ่มเป็นไฟล์ตามมาแยกต่างหากเมื่อรอบนั้นจบ ระหว่างนี้ใช้คำตอบจากโค้ดชุดนี้เป็นสัญญา (contract) ได้เลย เพราะเป็นตรรกะที่ route จริงบังคับใช้อยู่ ไม่ใช่การเดา

---

## 1) request/response เป๊ะ ๆ ของเส้นทาง `/api/session` → `/api/vehicles?startAt&endAt` → `/api/bookings?scope=calendar` → POST/PATCH `/api/bookings`

ตารางเต็ม (route/action/สิทธิ์/409 แต่ละเส้น) มีอยู่แล้วใน `2026-09-08-qc-integration-readiness.md` ข้อ (2) — ไม่พิมพ์ซ้ำ ด้านล่างคือ 3 เคส error body ที่ข้อนั้นยังไม่ลงรายละเอียด:

| เคส | จุดในโค้ด | Response |
|---|---|---|
| **เวลา/วันไม่ครบหรือผิด** (สร้างใบใหม่) | `app/api/bookings/route.ts:249-250` | `400 {ok:false, error:"กรุณาตรวจรถและวัน–เวลารับคืน"}` — เช็คว่า `carId` เป็น int, `startAt`/`endAt` เป็นตัวเลขจริง (`Number.isFinite`) และ `endAt > startAt` พร้อมกันทีเดียว ไม่บอกแยกว่าช่องไหนพัง |
| **draft ค้าง / ถูกใช้ไปแล้ว** (สร้างใบจาก LINE draft ซ้ำ) | `app/api/bookings/route.ts:242` | `409 {ok:false, error:"ร่างจาก LINE นี้ถูกบันทึกเป็น Booking แล้ว"}` — เช็คจาก `line_booking_drafts.status !== "new"` หรือมี `booking_id` ผูกอยู่แล้ว **หมายเหตุ: ด่านนี้มีเฉพาะทางเข้าจาก LINE draft เท่านั้น** ถ้า native สร้างใบตรงไม่ผ่าน draft จะไม่มีด่านนี้ให้ |
| **คำขอซ้ำหลัง timeout** (retry สร้างใบเดิม) | `app/api/bookings/route.ts:262-263` | **ไม่มี idempotency key จริง** (ตรวจซ้ำแล้ว — ดูข้อ 2) สิ่งที่เกิดจริงถ้า retry ด้วยรถ+ช่วงเวลาเดิมคือชนกับ**ใบที่เพิ่งสร้างจากครั้งแรกเอง** → ได้ `409 {ok:false, error:"รถมีคิวซ้อนในช่วงที่เลือก กรุณาเลือกคันหรือวันใหม่"}` — ⚠️ **native ต้องตีความ 409 หลัง retry ว่า "รอบแรกน่าจะสำเร็จไปแล้ว ให้เช็ค `GET /api/bookings?scope=calendar` ก่อน" ไม่ใช่ตีความว่ารถไม่ว่างจริง ๆ แล้วให้ลูกค้าเลือกวันอื่น** ข้อความ error เดียวกันใช้ได้สองความหมาย ต้อง disambiguate ฝั่ง native เอง |

---

## 2) create/edit/release/complete_return/update_financial_summary/refund — atomic ไหม idempotent ไหม

| action | เขียนเป็นก้อนเดียวไหม (atomic) | idempotency |
|---|---|---|
| สร้างใบจอง (`POST`) | 1 คำสั่ง INSERT เดียว (`:314`) | ไม่มี key — กันซ้ำได้บางส่วนผ่านเช็คคิวซ้อน (ดูข้อ 1) |
| แก้ใบ/เพิ่มวัน (`update_booking`) | หลายคำสั่งแยกกัน (ไม่ใช่ `env.DB.batch`) — `replaceBookingFees` แล้วค่อย `INSERT audit_log` (`:543,549`) | ไม่มี key |
| ปล่อยรถ (`release_vehicle`) | 1 `UPDATE` | ไม่มี key — เช็คเงื่อนไขก่อนปล่อยจากฐานสดทุกครั้ง (409 ถ้าขาด) |
| **คืนรถ+ปิดยอด** (`complete_return`) | `env.DB.batch([...])` ก้อนเดียวสำหรับ UPDATE bookings + INSERT รายการหัก/ค่าใช้จ่าย/notification (`:575-581`) **แต่ `INSERT audit_log` อยู่นอก batch นี้ เป็นคำสั่งแยกทีหลัง** (`:585`) | ไม่มี key — กันปิดซ้ำด้วยเช็ค `status IN (completed,returned)` ก่อนเข้า action นี้เลย (409 ถ้าปิดไปแล้ว) กันซ้ำได้จริงเพราะเช็คจากฐานตรง ๆ |
| สรุปเงินหลังคืน (`update_financial_summary`) | 1 `UPDATE` | ไม่มี key |
| **คืนเงินประกันจริง** (`POST /api/deposits`) | `env.DB.batch([...])` ก้อนเดียว (UPDATE bookings + notification + audit_log อยู่ใน batch เดียวกัน ต่างจาก complete_return) | ไม่มี key แต่คำนวณจากยอดคงเหลือ**สด**จากฐานทุกครั้ง (`planDepositRefund` อ่าน `deposit_returned` ปัจจุบัน) ถ้ายอดที่ขอเกิน remaining → `409` ไม่คืนซ้ำเงียบ ๆ — **กันความเสียหายได้จริงในระดับหนึ่งแต่ไม่ใช่ idempotency แท้** (ถ้ากดสองครั้งด้วยยอดคนละจำนวนที่ยังไม่เกิน remaining จะคืนทั้งสองครั้งจริง) |

**ตอบตรงคำถาม**: refund (`/api/deposits`) กับ closeout (`complete_return`) เป็น **คนละ endpoint คนละธุรกรรมกันโดยสิ้นเชิง ไม่ atomic ร่วมกัน** ตามที่ GPT ตั้งข้อสังเกตไว้ถูกแล้ว — ปิดงานคืนรถได้โดยยังไม่โอนเงินคืนก็ได้ (สถานะ `completed`/`returned` แต่ `deposit_closed_at` ยังว่าง)

🔴 **จุดเสี่ยงที่เจอจากการอ่านโค้ดรอบนี้ (ยังไม่ใช่บั๊กที่เคยระเบิดจริง แต่เข้าเค้าเดียวกับที่ทีมประกอบจอ "งาน" เพิ่งเจอสด ๆ ในเส้นใหม่)**: `complete_return` เขียน `audit_log` แยกนอก batch (`:585`) ถ้าคำสั่งนั้นพัง (เช่น `actor_email` เป็น null จากทางเข้าที่ไม่ผ่าน guard ปกติ) บัญชี/สถานะใบจองจะเปลี่ยนไปแล้วแต่ response อาจกลายเป็น error ให้ทีมกดซ้ำ — เหมือนบั๊กที่ endpoint ใหม่ `urgent-jobs/complete` เจอและแก้ไปแล้วด้วยการรวม audit เข้า batch เดียวกัน ยังไม่ได้แก้ที่ `complete_return` เดิม (นอกขอบเขตงานนี้ ขอบันทึกไว้เป็นข้อเสนอ ไม่ใช่คำสั่ง)

---

## 3) นิยาม conflict ปฏิทิน — ยืนยันฝั่งที่ GPT ระบุว่าเป็นหลักฐานล่าสุดถูกต้อง

**ทุกสถานะยกเว้น `cancelled`/`rejected` กีดกันคิว** ไม่ใช่แค่ `confirmed×confirmed` — โค้ดจริง (ไม่ใช่การตีความ):

```sql
-- สร้างใบใหม่ app/api/bookings/route.ts:262
WHERE car_id=? AND status NOT IN ('cancelled','rejected') AND start_at < ? AND end_at > ?

-- แก้วันเช่าใบเดิม app/api/bookings/route.ts:515
WHERE car_id=? AND id<>? AND status NOT IN ('cancelled','rejected') AND start_at < ? AND end_at > ?
```

ทั้งสองจุดตรงกัน `pending`/`confirmed`/`active`/`completed`/`returned` ล้วนกีดกันคิวทั้งหมด — **fixture เดิมของ GPT ที่ให้ conflict เฉพาะ `confirmed×confirmed` ตกยุคแล้ว ต้องอัปเดตให้ตรงกับด้านบน**

(ส่วน `lib/car-availability.ts` ที่ให้ 3 สถานะ free/partial/busy เป็นแค่ตัวช่วย**กรองแสดงผล**ปฏิทิน/หน้า `/book` เท่านั้น ไม่ใช่ตัวตัดสินจริง — คอมเมนต์ในไฟล์นั้นบอกไว้ตรง ๆ ว่าตัวตัดสินจริงคือ query ด้านบน)

---

## 4) write path ที่ถือเป็น "ทางการ" สำหรับ "ลูกค้าจ่ายแล้ว"

ลำดับจริง (ยืนยันด้วยโค้ด ไม่ใช่สมมติฐาน):

1. อัปโหลดสลิป → `POST /api/evidence` `category:"customer_payment"` → insert `financial_evidence` สถานะ `pending` (`:169`) — **แค่ "อ้าง" ยังไม่ใช่ยืนยัน**
2. ต้องมีคนสิทธิ์ `canEditFinance` **หรือ** (`category==="customer_payment"` และ `canWriteOperations`) กด `PATCH` confirm (`:186`)
3. **ตอน confirm เท่านั้น** ที่เขียน `bookings.booking_deposit_received += amount` (ทางการ) และ insert แถว `booking_fees` kind `paid_amount` (`:200-206`)

→ **field ที่ authoritative คือ `bookings.booking_deposit_received`** ไม่ใช่ยอดจากสลิป/OCR/AI อ่านสลิป — ตรงกับที่ GPT ขอให้ระวังพอดี ("OCR paid ยังเป็น claim ไม่ให้กรอกว่าได้รับเงินแล้วเอง") ถ้า Luna หรือการ์ดอ่านสลิปในอนาคตจะ auto-fill ต้อง fill เป็น "รอตรวจ" เท่านั้น ห้าม auto-confirm

---

## 5) release gate — deployed จริง vs owner-approved-แต่ยัง local/proposed

ตารางเต็มอยู่ใน `qc-integration-readiness.md` ข้อ (2)+(3) แล้ว สรุปสั้นตรงนี้ + เพิ่มของใหม่ที่ยังไม่เคยแจ้ง:

| ชั้น | ตัวอย่าง |
|---|---|
| **deployed** (ใช้ได้บน `app.dnaev.com` วันนี้) | auth ครบชุด, vehicles/calendar/schedule read, create/edit/release/complete_return/update_financial_summary/cancel/refund booking, evidence upload+confirm, การ์ดทีม (poll) |
| **local** (owner สั่งให้ทำแล้ว มีเทสต์ แต่ยังไม่ deploy) | read-document โหมดข้อความ, A4 audit sheet, `GET /api/bookings/:id` ยังไม่มีเลย (native ต้องหยิบจาก list ไปก่อน) |
| 🆕 **local ขั้นต้นกว่านั้นอีก — ยัง uncommitted ในเครื่อง C ตอนนี้เลย** | จอ "งาน" (ปิดจ๊อบส่ง/รับคืนรถแบบเร็วจากมือถือ) ตามสเปก `assemble-next.md` — endpoint ใหม่ `POST /api/urgent-jobs/complete {bookingId, kind:"pickup"\|"return"}` **เขียนได้แค่ธงงาน** (`pickup_done`/`dropoff_done`+`status`) **ห้ามแตะเงินเด็ดขาด** งานเงิน/คืนประกันยังคงต้องผ่าน `complete_return`+`/api/deposits` เดิมเท่านั้น — กำลังถูกประกอบสดอยู่ตอนนี้โดยอีกทีมงานฝั่ง C จะ commit/push เมื่อทดสอบครบ |
| **proposed** (ยังไม่มีเลย ต้องออกแบบใหม่ทั้งเส้น) | idempotency key, API ออกสัญญาเช่ารถ, Luna endpoint, ภาษี, push notification (APNs) — ตรงกับ `2026-09-09-backend-contract-answers.md` |

---

## สรุปให้ทีมดีไซน์/native

- ปรับ fixture conflict เป็น "ทุกสถานะยกเว้น cancelled/rejected" ตามข้อ 3 — จุดเดียวที่ต้องแก้ก่อนอย่างอื่น เพราะกระทบทุกหน้าที่เช็คคิวว่าง
- 409 "รถมีคิวซ้อน" หลัง retry การสร้างใบ **ไม่ใช่สัญญาณให้ลองรถ/วันใหม่เสมอไป** — ต้องเช็ค calendar ก่อนตีความ (ข้อ 1)
- "ลูกค้าจ่ายแล้ว" ต้องอ่านจาก `booking_deposit_received` เท่านั้น ห้ามเชื่อผลอ่านสลิป/OCR ตรง ๆ (ข้อ 4)
- refund กับ closeout เป็นคนละธุรกรรม ทำ UI แยกสถานะสองอย่างให้ชัด อย่าสื่อว่าปิดจ๊อบ = เงินคืนแล้ว (ข้อ 2, ตรงกับที่ `assemble-next.md` เขียนไว้แล้วว่าไม่ให้ตีความแบบนั้น)
- จอ "งาน" ตาม `assemble-next.md` กำลังถูกประกอบจริงอยู่ ยังไม่ commit — รอผลทดสอบสดรอบถัดไปจาก C ก่อนถือว่าพร้อม

ผลชุดนี้เป็น backend contract answers ตามที่ขอ **ไม่ใช่คำสั่ง deploy หรือทดสอบเขียนข้อมูล production**
