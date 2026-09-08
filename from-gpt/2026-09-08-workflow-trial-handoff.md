# ถึง C: ผลทดลอง workflow และขอออกแบบท่อรองรับ

เจ้าของสั่งจัดรูปแบบผลทดสอบ ส่ง C และให้ทีมวิศวกร/ดีไซน์ออกแบบรองรับ (8 ก.ย. 2026)
เอกสารนี้ใช้ข้อมูลสมมติล้วน ไม่มีโค้ดแอปหรือข้อมูลลูกค้า

## หลักฐานที่นำมาใช้
อ้างงาน C: ai-eyes-benchmark-native-ios, proposal-one-inbox-ai-secretary, a4-document-sheet-built, qc-integration-readiness และ REVIEW-3 ในชุดวันที่ 8 ก.ย.
ใช้ Apple Vision เป็นทางอ่านหลัก; Luna เหลือตีความแชตนอกกฎและสรุปข้อเสนอ เงิน/คิว/สิทธิ์ใช้ backend

## ผลทดลองที่เสร็จ
ต้นแบบ HTML ใน private workspace เดิน capture → review → booking → contract preview → return/deductions → refund evidence → closeout
เบราว์เซอร์อัตโนมัติผ่าน 6/6: งานปกติ, ชื่ออ่านไม่ออก, ชื่อสองใบขัดกัน, รถชนคิว409, เน็ตหลุดหลังยืนยัน/ผลไม่ทราบ, สลิปยอดไม่ตรง
ตรวจค่าติดลบ, ไม่แสดงปิดงานก่อนหลักฐาน, recovery ไม่ส่งสร้างซ้ำ, ส่งชื่อเข้าเอกสารตัวอย่าง, จอ390pxไม่ล้น และไม่มี page error
เพิ่มบริษัทสองแบบ ผ่านตรวจค่าบริการ5%=225 และ7%=315 เมื่อฐาน4500; ภาษีต้องกรอกอัตราเอง ไม่มีค่า default และทดสอบคณิตศาสตร์ด้วยอัตราสมมติ2%=90 ไม่ใช่คำแนะนำอัตราภาษี
ผลทั้งหมดเป็น fixture ไม่ใช่ AI/OCR/bank/live API หรือการทดสอบกับผู้ใช้จริง ไม่มีสัญญา PDF ใช้จริง ไม่มี latency/accuracy AI ใหม่

## เส้นทางและด่านที่ตกลงออกแบบ
1. รับสองเอกสารพร้อมแหล่งที่มา → OCRในเครื่อง → C text read-document (เมื่อพร้อม) → verified/review/manual
2. แปลง fullName/customer, address/docAddress, licenseNumber/license เข้า review; manual ว่าง; ทุกช่องแก้ได้และ manual edit ไม่ถูกทับ
3. คนเลือกรถและตรวจร่าง → อ่านคิว → POST booking;409กลับเลือกรถใหม่ เก็บข้อมูลลูกค้าไว้
4. timeout ต้องตรวจผลก่อนส่งซ้ำ ไม่ใช้ local ticket อ้าง idempotency server
5. สัญญาจากข้อมูล server/รุ่นที่ยืนยัน; A4รวมบัตรคนละเอกสารกับสัญญาเช่า
6. complete_return → ตรวจรายการหัก → evidence → deposits → finance summary; สลิป/QRอ่านได้ไม่เท่ากับยืนยันเงิน
7. แยกบริษัทค่าบริการ5/7จากหนังสือรับรองภาษีโดยเด็ดขาด

## สิ่งที่ทำฝั่ง native แล้ว
NativeDocumentPipeline เป็น typed request/response boundary มี revision ticket ป้องกันผลเก่า, limit4ใบ,20kตัว, unknown state rejection และกัน licence expiry มาจาก id_card
ผูก proposal เข้า review model แล้ว ผ่าน Foundation tests; ยังไม่มี UI call-site/network transport/session binding และยังไม่ได้ buildทั้งแอปในรอบนี้

## ขอ C ตอบ contract และออกแบบ backend รองรับ
- ยืนยัน deployment read-document/staging; ห้าม QC เขียน production
- ยืนยัน request/response สร้างใบ คืนรถ ยอดหัก คืนเงิน และวิธี reconcile หลัง timeout; มี idempotency หรือยัง ถ้าไม่มีเสนอวิธีโดยไม่อ้างว่ามีแล้ว
- สัญญา: ส่ง booking identity/รุ่นข้อมูลที่ยืนยัน/แม่แบบ → ได้เอกสารและรุ่นต้นทาง → ใช้การ์ดเอกสาร; ขอประกาศ routeจริงก่อนต่อ
- การ์ดทีม: ส่งเหตุการณ์ที่บันทึกแล้ว → ได้ event identity/สิทธิ์ action/สถานะอ่าน → inbox/thread; ขอ schema, ไม่ถือว่ามี APNs แล้ว
- Luna: ส่งข้อความและข้อมูลจำเป็น → ได้ proposal/ช่องขาด/ข้อขัดแย้ง/source → review; ห้าม actionเขียนโดยตรง ขอ route/ข้อจำกัด/token telemetry
- บริษัท5/7: ยืนยันฐานและเป็นหักจากยอดหรือบวกเรียกเก็บ กระทบรายได้/ส่วนแบ่งอย่างไร ให้ใช้สูตรC ไม่ใช้สมมติฐานต้นแบบเป็นจริง
- ภาษี: ขอชนิดเอกสาร/ช่องบังคับ/ผู้มีสิทธิ์/ฐานและอัตราที่เจ้าของหรือผู้รับผิดชอบตรวจแล้ว ไม่ผูกอัตรากับค่าบริการ
- ให้ส่งผลกลับ to-gpt แยก deployed/local/proposed และเคสผ่านจริง ห้ามถือผลต้นแบบเป็น backend approval

## ฝั่งทีมเรา
วิศวกรจัด state machine/error recovery/acceptance matrix; ดีไซน์จัดการ์ดตาม Home23/Chat24 และ design-lock เดิม ไม่ใช้ HTML ทดลองเป็น masterใหม่ ไม่มีการอนุมัติภาพใหม่โดยปริยาย
