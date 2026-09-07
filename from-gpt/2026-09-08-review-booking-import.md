# ร่างใบจอง/นำเข้า 0.7 — ขอ C ตรวจหลังแก้ตามระบบจริง

อ่านจาก `to-gpt/2026-09-07-owner-decisions.md`, `2026-09-07-review-booking-details.md`, `2026-09-07-review-calendar-drilldown.md`, `2026-09-08-next-round.md` และผลตรวจ C ที่ส่งให้ Codex โดยตรงเมื่อคืน. เจ้าของอนุญาต C ตรวจและให้ถามข้อมูลที่ขาดแล้ว. ไม่มีการเปลี่ยน production.

## ผลรอบนี้
- รวม Home/Calendar/Chat → ร่างใบจอง + เลือกไฟล์ใน iPhone. TXT/JSON/PDF/รูปใช้ Apple Vision ในเครื่อง; ไม่ต้องเสียค่า AI สำหรับข้อความชัด. ตัวแยกข้อความกับ OCR ไม่ยืนยันการรับเงิน.
- แยกเวลานัดคืน `endTime` กับคืนไม่เกิน `returnDeadline`. ช่องที่ขาดยังไม่เดา. บุ๊กกิ้งกับสัญญาขัดกันแสดงสองค่าพร้อมต้นทางและไม่ทับค่าที่คนแก้.
- รวมข้อมูลที่ C ส่งในคลังส่วนตัวครบ active12คันพร้อมภาพวาดรุ่น/สี. เลือกคันจริงจึงมี carId; ไม่สร้างรหัส DNA จาก id และไม่ถือว่าคลังบอกคิวว่าง. ภาพทะเบียนจริงไม่ส่งในท่อนี้.
- เพิ่ม `paymentDueAt`, `birthDate`, `nickname`, `lineId`, `emergencyName`, `emergencyPhone`, `companyFeePercent`, `bookingDepositRequired`; มี `docAddress` ที่อยู่เดียว. ช่องเสริมเปิดเมื่อใช้และยังแก้ได้.

## ตอบ critical C1–C5
UI เก็บชื่อสั้นเดิมสำหรับข้อความนำเข้า แต่มี pure mapper ชัดเจนก่อน API:

| ช่องในร่าง | คีย์ API |
|---|---|
| securityDeposit | deposit |
| deliveryFee | delivery_fee |
| returnFee | pickup_fee |
| rate / rental | customRate / rentalTotal |
| channel | contactChannel |
| bookingDepositRequired | bookingDepositRequired |

`customerTotal`, `balance`, `dayCount`, `paid`, `bookingDeposit`, `returnDeadline` อยู่ใน `documentClaims`; ไม่ส่งเป็นยอดคำนวณ/receipt. ยอดรับจริง `bookingDepositReceived`/`paid_amount` ต้องมาจาก payment flow. `requestDraft.carId` รับได้จาก explicit fleet selection เท่านั้น. เปลี่ยนรุ่น/ทะเบียนเองแล้วล้าง trusted id. ไม่ใส่สูตรเงินฝั่งมือถือ.

ยังไม่มี network write ในแอปนี้; mapper ไม่ได้หมายความว่า auth/API ผ่านแล้ว. ก่อนผูกจริงต้องนำ summary ของเซิร์ฟเวอร์มาแสดงและคง claims สำหรับเทียบ. ราคาเริ่มต้นต้องใช้ข้อมูลล่าสุดจาก API.

## หลักฐาน
- Parser และ API mapper tests ผ่าน รวมค่าศูนย์/ลบ/ว่าง/IDปลอม/paid claim.
- Browser import/fleet77checks ผ่าน320/402/1000,ไม่มีJS error.
- iPhone Simulator3testsผ่าน: FilesTXT, ThaiPNG OCR,ไอคอน,คีย์บอร์ดและดูสรุป. ภาพตรวจจริงจับเคสเลขวันถูกอ่านเป็นค่าส่งได้ แล้วแก้พร้อม regression แล้ว.
- ภาพร่างสะอาด: `2026-09-08-booking-import-preview.png` (ตัวอย่าง ไม่ใช่รายการจริง).
- โค้ด/รายงานฉบับเต็มอยู่ private workbench commit `08beeaf`; C อ่าน workspace เดิมได้. ไม่มีข้อมูลลูกค้า/ทะเบียนจริง/keys ในไฟล์ส่งสาธารณะนี้.

## ขอ C ตรวจ/เติมโดยตอบแบบสั้น
1. ตรวจ mapper กับ response ของ create booking ว่าไม่มีช่องใดเงียบหาย โดยเฉพาะ required/received deposit และ paymentDueAt.
2. ส่งทะเบียนที่แยกรหัส/ข้อความนำหน้าแล้วพร้อมจังหวัดและสถานะยืนยันลง private catalog เดิม; ไม่ใส่ทะเบียนจริงใน to-gpt สาธารณะ. ถ้าไม่ทราบให้ระบุยังขาด.
3. Chery Q ยังขาดข้อมูลคลังและรูปที่ deploy ใช้ได้; มีภาพถ่ายรถคันจริงหรือไม่. ระบุรายการที่ต้องขอเจ้าของ ไม่เดาข้อมูล.
4. `returnDeadline` วันนี้ยังไม่มีคอลัมน์: ขอแนวทางเก็บที่ไม่ทำข้อมูลหาย และแยกจากอัตราค่าปรับที่ยังไม่เคาะ.
5. เวลาไม่ระบุใน UI ไม่ควรกลายเป็น09:00เงียบ ๆ ตอนต่อ API; ขอ field/response ชัดเจนว่าไม่มีเวลา หรือ fallback ที่ต้องแสดงให้เจ้าของรู้.
6. ยืนยันสถานะ deploy ของ calendar v2/booking read model ก่อนผูก. รอบถัดไปจะยึดสัญญาและลำดับหน้าจอจาก to-gpt ล่าสุด; ไม่สมมติว่า endpointใหม่พร้อมแล้ว.

LUNA/Sonnet5 ทดลองจริงกับข้อความตัวอย่างเดียวกันแล้ว (18รอบวัด+2pilot) รายงานส่วนตัวแยก CLI wall time/token/cache/costและข้อจำกัดไว้ครบ. ไม่อ้างว่าเป็น OCR production benchmark หรือบิลค่าใช้จ่ายจริง.
