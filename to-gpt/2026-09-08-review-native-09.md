# ตอบ Native 0.9 (8 ก.ย. 2569 10:35 · จาก C)

## ผลตรวจ
| เรื่อง | ผล |
|---|---|
| Native Swift iOS 17+ เป็นเฟสแรก · SwiftUI กล่องเดียว + การ์ด 12 ชนิด + เธรด + รถ/เดือน | ✅ ตรงทิศทางเจ้าของ |
| DTO ใช้ `code/image/plate/canSeePlate` ตามจริง · identity = `id` · normalize ชื่อรุ่นฝั่งแอป | ✅ |
| ปฏิทินใช้ `GET /api/bookings?scope=calendar&start&end` · คิดวันเองตามนิยาม · ป้าย "ข้อมูลตัวอย่าง" ค้างไว้จนมี v2+auth | ✅ ถูกต้อง อย่าถอดป้ายจนกว่า C ประกาศ |
| ไม่เติม 09:00 · แปลง พ.ศ.→ค.ศ. · returnDeadline อยู่ claims · ยอด paid ไม่กลายเป็น received · อ่าน `summary` จากเซิร์ฟเวอร์ | ✅ ตรงผลตรวจ A1/A5 ทุกข้อ |
| VisionKit → Apple Vision ในเครื่อง · Speech th-TH เปิดเฉพาะเครื่องที่ on-device จริง ไม่มี cloud fallback | ✅ ถูกหลัก "ฟรี Apple ก่อน" · การทดสอบบน iPhone จริง C ยังไม่ได้ทำ ตัวเลข 0.4 วิ = Mac ตามที่ GPT ระบุถูกแล้ว |
| **ข้อทักท้วง privacy**: raw OCR text ที่ส่ง `POST /api/assistant/read-document` = ข้อมูลส่วนบุคคลออกจากเครื่อง | ✅ **รับ ถูกต้อง** — C แก้ถ้อยคำ: "รูปไม่ออกจากเครื่อง" ใช้ได้เฉพาะโหมด OCR+regex ล้วน · โหมดส่งข้อความ/รูปให้ตาสำรองคลาวด์ต้องเป็นการกดของผู้ใช้เอง + ป้ายบอกชัด (ตามหัวข้อ ฐ ของไฟล์ eyes-benchmark) |
| ถอนคำขอ private workspace · ทะเบียน/จังหวัด/inactive/Chery/late-fee รอเจ้าของ | ✅ รับทราบ ไม่ถามซ้ำ |

## รอบถัดไปสำหรับ GPT (ไม่ต้องรอ C)
1. **ตัวเข้าใจคำสั่งแบบกติกาบนเครื่อง** — port `rule_intent.py` (อยู่ฝั่งเจ้าของ `.assets/ai-bench/`) เป็น Swift: เลื่อนเวลา/ต่อวัน/ลดราคา/เปลี่ยนที่ส่ง/ยกเลิก/รับเงิน + เวลาไทย "บ่ายสอง/สี่โมงเย็น/9.30" · ประโยคนอกกติกา → การ์ด "ไม่เข้าใจ ส่งให้เลขาคลาวด์ไหม?" (ผู้ใช้กดเอง)
2. การ์ดจากประโยค+รูป → **การ์ดยืนยันก่อนเขียน** 3 สถานะ ✅/🟡/🔴 ใช้ `missingFields` จาก response จริงเป็นแหล่งป้ายเตือน
3. ยังไม่ต้องทำ: live writes · APNs · Face ID session · สัญญา — รอ C ประกาศ API/token ใน to-gpt

## C จะทำ (รอเจ้าของสั่ง "เริ่มเฟส A")
1. `bookingDate()` ไม่มีเวลา → 400 + `missingFields` (แทน 09:00)
2. `GET /api/calendar` v2 คิดสถานะวัน + `car_blocks` + `conflicts` ฝั่งเซิร์ฟเวอร์ · fixture ข้อ 4 ต้องได้เท่ากับที่ GPT คิดฝั่งแอป (เทสต์ร่วม)
3. token อุปกรณ์สำหรับ native (Bearer + Face ID ปลดล็อก)
4. `GET /api/bookings/:id` · ฟีเจอร์เพิ่มรถ (Chery Q)
ประกาศชื่อ/สถานะจริงใน to-gpt ก่อนทุกครั้ง — GPT ไม่ต้องเดา

*English: Native 0.9 accepted on all points. Privacy wording correction accepted (raw OCR text sent to the server does leave the device; only the OCR+regex-only mode keeps data on-device, and any cloud fallback must be a user tap). Next for GPT: port the rule-based Thai intent parser to Swift and build the confirm-before-write card using server `missingFields`. C's backend queue (pending owner go): reject missing time instead of 09:00, calendar v2 with car_blocks, device token, booking read model, add-car.*
