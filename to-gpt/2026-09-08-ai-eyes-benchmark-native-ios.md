# ผลทดสอบ "ตา/หู/สมอง" ของแอป — ข้อสรุปที่กระทบดีไซน์ (8 ก.ย. 2569 · จาก C)

**เจ้าของสั่ง:** ประหยัดสุด · ห้ามเข้าข้างค่ายใด · ต้องฝังใน iPhone · **ของฟรีที่ติดมากับ iPhone/Mac คิดก่อน**
C ทดสอบจริง 13 ตัว (Apple ในเครื่อง · Qwen/GLM/ERNIE จีน · Gemini/GPT/Mistral/Claude) บนรูปสมมติ 4 ชนิด 2 ระดับความยาก + ประโยคสั่งงานไทย 6 ข้อ ด้วยสคริปต์เดียวกัน

## ผลที่กระทบดีไซน์โดยตรง
1. **Apple Vision OCR ในเครื่อง = 15/15 · 14/15 ชุดยาก · 0.4 วิ · ฿0 · รูปบัตรไม่ออกจาก iPhone** — ชนะทุกคลาวด์ที่ทดสอบ (Gemini 2.5 Flash-Lite เท่ากันแต่ 4 วิ + ส่งรูปออก · Claude Haiku 7/15 ชุดยาก)
2. ของฟรี Apple อื่นที่ทดสอบแล้วใช้ได้: **Speech th-TH ในเครื่อง** (พูดแทนพิมพ์ ไม่ต้องเน็ต) · **NSDataDetector** อ่านวันที่ พ.ศ. "12/09/2569 เวลา 14:00" ถูก · **QR สลิป** อ่านได้ · **ตัวเข้าใจคำสั่งแบบกติกา** 11/11 ประโยคประจำ (ไม่ใช้ AI) · ยืนยันจากหน้าทางการ: **APNs push ฟรีไม่จำกัด** (แทนโควตา LINE 300) · Face ID แทน OTP · EventKit นัดลงปฏิทิน · Live Activity "งานวันนี้" บนล็อกสกรีน · VisionKit กล้องตัดขอบบัตรเอง (iOS 17+)
3. ❌ Apple Foundation Models (LLM ในเครื่อง) **ไม่รองรับไทย** → ประโยคนอกกติกาส่งคลาวด์ถูก (Mistral Small / Qwen3-VL-32B ได้ 6/6 · ฿3–5/เดือน)

## สิ่งที่ GPT ต้องเปลี่ยน
- **แอปต้องเป็น native Swift (iOS 17+)** ไม่ใช่เว็บห่อ — ของฟรีข้อ 2 ทั้งหมดใช้ได้เฉพาะ native → เฟส E "ห่อเป็นแอป" เลื่อนขึ้นมาเป็นเฟสแรกคู่กับ A (ปรับ `01-mobile-app-to-build.md` หัวข้อ 11 แล้ว)
- ออกแบบ "ส่งรูป" เป็น **กล้องสแกนเอกสาร** (VisionKit) ไม่ใช่ปุ่มแนบไฟล์ · ผลอ่านขึ้นเป็นการ์ดใน < 1 วิ ไม่มีสถานะ "กำลังส่งขึ้นเซิร์ฟเวอร์"
- ช่องพิมพ์ล่างจอต้องมี **ปุ่มไมค์** (Speech ในเครื่อง) เป็นท่าหลักเท่าพิมพ์
- การ์ด/เตือนทีม = **APNs + Live Activity** ไม่ต้องประหยัดจำนวนแจ้งเตือนอีกต่อไป (ต่างจากยุค LINE)
- ล็อกอิน = Face ID · ป้าย "review" บนการ์ดเอกสารเกิดเมื่อ checksum/regex ไม่ผ่าน → ค่อยถามว่าจะส่งรูปให้ตาสำรองคลาวด์ไหม (ผู้ใช้กดเอง = PDPA ชัด)

## ยังไม่พิสูจน์
รูปจริงของทีม · Siri สั่งงานไทยบนเครื่องจริง · ประโยคจริงของทีมสำหรับตัวกติกา — C จะขอตัวอย่างจากเจ้าของ

รายละเอียดเต็ม (ตารางคะแนน 13 ตัว · ราคา · PDPA รายค่าย) อยู่ฝั่งเจ้าของ ไม่ลงรีโปสาธารณะ
*English: on-device Apple Vision OCR beat every cloud model tested (15/15, 14/15 hard, 0.4 s, free, images never leave the phone). Free Apple frameworks cover speech-to-text (Thai, on-device), date/phone detection, slip QR, push, Face ID, calendar. Consequence: the app must be native Swift (iOS 17+); "wrap as app" moves to phase one. Cloud LLM is only a fallback for sentences the rule-based parser can't handle.*
