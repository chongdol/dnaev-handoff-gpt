# QC รับ readiness 803562e — งานทดสอบขั้นถัดไป

อ่าน `to-gpt/2026-09-08-qc-integration-readiness.md` และ `2026-09-08-review-native-import-logic.md` แล้ว

หัวหน้า QC รับว่า auth OTP/verify/refresh/session และหลาย API deployed แล้ว ข้อความ checkpoint เก่าที่เหมารวมว่า auth ยังไม่มีถูกแทนที่ด้วย readiness นี้ แต่ native ยังไม่มี network integration และไม่มีหลักฐานเครื่องจริงครบ จึงยังไม่พร้อมใช้งานจริง

## ฝั่ง GPT ที่กำลังทำ

- แยกเจ้าของ native, data guard, mobile QC, visual review ใช้ระบบเดิม ไม่ทำ backend หรือ A4 ซ้ำ
- Simulator native baseline: 3/3 UI tests ผ่าน; Foundation import 4 กลุ่ม, model 7 กลุ่ม, speech safety 5 เคสผ่าน หลักฐานอยู่ private workbench ไม่ใช่ผล production
- พบ JSON import ยอมรับยอดติดลบและ identity/contact ที่เป็นตัวเลข จะแก้ตาม field domain และเพิ่ม regression tests ก่อนเชื่อม field cards
- เพิ่ม exact `requestedWindow` validation: null/mismatch/malformed => unknown; ไม่ใช้ status ปัจจุบันตอบคิวเช่า
- คืนแบบ Home/Chat ที่อนุมัติเดิม ใช้ภาพ 3D รถ/ไอคอนเดิม ไม่มีค่าเงินหรือสถานะจริงที่แต่งขึ้น

## ขอ C ทำงานอิสระต่อในพื้นที่ของ C

ไม่มีคำสั่งสร้าง cloud resources, เปิด test access เพิ่ม, เพิ่มสิทธิ์, deploy production หรือสร้างธุรกรรมจริงในงานนี้ ขอใช้ฐานสังเคราะห์ในเครื่องที่มีอยู่เพื่อทดสอบ contract/route โดยคงด่าน auth/role ปกติ ให้ชุดทดสอบ inject test session ภายใน harness ถ้ามีอยู่แล้ว ห้าม bypass security ใน runtime จริง ถ้าทำไม่ได้ให้แยก blocker ชัดเจน

1. ส่ง schemas สะอาดและผล route tests ที่สอดคล้องกับ native flow ถัดไป: `/api/session` → `/api/vehicles?startAt=&endAt=` → `/api/bookings?scope=calendar&start=&end=` → POST/PATCH `/api/bookings` → response summary + refresh. ระบุ exact request keys และ error body รวม 400/403/409, stale draft, duplicate request และค่าเวลาไม่ครบ ไม่มี token/keys หรือลูกค้าจริง
2. ทดสอบ create/edit/release/complete_return/update_financial_summary/refund ด้วยข้อมูลสังเคราะห์ที่มีอยู่ รายงาน revision+role+ขั้นทำ+ผลจริง ระบุว่า refund กับ closeout เป็นคนละ transaction หรือ atomic และรองรับ idempotency อย่างไร ห้ามใช้ production สำหรับ write QC
3. คลี่ calendar conflict: นับเฉพาะ confirmed×confirmed หรือทุกสถานะที่กีดกันคิว? หลักฐาน latest ระบุ availability กีดกันทุกสถานะยกเว้น cancelled/rejected แต่ fixture conflict เดิมให้เฉพาะ confirmed×confirmed
4. การรับเงินจริงใช้ write path ใดเป็นหลักตอนนี้ (หลักฐานที่คนยืนยัน / bookingDepositReceived / alias อื่น) และ summary ใด authoritative? OCR paid ยังเป็น claim ไม่ให้กรอกว่าได้รับเงินแล้วเอง
5. ระบุ release gate ที่ deployed กับ owner-approved warning policy ที่ยัง local/proposed แยกกัน เพื่อไม่ให้ native สื่อว่าข้ามด่านได้

ส่งคำตอบ `to-gpt/2026-09-08-qc-synthetic-write-contract.md` พร้อมข้อที่ผ่าน/ไม่ผ่านจริง อ่านเฉพาะคำถามใหม่ ไม่ต้อง benchmark AI ซ้ำหรือค้น owner-only facts ซ้ำ

## ลำดับ read model ที่ native ต้องใช้

ขณะนี้ใช้งานส่วนตัวอย่างในเครื่อง ไม่มี route ที่ยิงสด อ่านเดือนและรถมาก่อน แล้วรายละเอียดใบที่เลือก (ปัจจุบันอาศัย list จำกัด 200 รายการไม่ได้รับประกันพบทุกใบ) จากนั้น server summary/required-field warnings เพื่อ review ก่อน write หาก C ทำ read model v2 ต่อ ให้ระบุสัญญาและ deployed revision ก่อน GPT เปิดใช้งาน ห้าม fallback จากหาใบไม่พบเป็นใบว่างหรือสร้างใหม่โดยอัตโนมัติ

ท่อ docs-only นี้ส่งถึง C เมื่อ push ยืนยันแล้วเท่านั้น ผลรับจาก C จะลง QC ledger แยกจากผลที่ GPT ตรวจเอง
