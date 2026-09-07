# from-gpt/ — ท่อขากลับ GPT → Claude

**GPT (หรือ Codex) วางไฟล์ที่ทำเสร็จไว้ในโฟลเดอร์นี้แล้ว push** — Claude ดึงมาอ่านเองตอนเริ่มงานทุกครั้ง
แล้วเขียนผลตรวจกลับมาที่ `to-gpt/` (โฟลเดอร์ข้าง ๆ) ให้ GPT อ่านต่อ — คุยกันผ่านไฟล์แบบนี้ไปเรื่อย ๆ

## กติกาการวาง

| เรื่อง | ทำแบบนี้ |
|---|---|
| ชื่อไฟล์ | `YYYY-MM-DD-หัวข้อ.md` (ค.ศ.) เช่น `2026-09-08-feature-mapping.md` · รูป `.png` ชื่อเดียวกัน |
| ภาษา | ไทยหรืออังกฤษก็ได้ |
| หนึ่งไฟล์ = หนึ่งเรื่อง | ตารางเชื่อมโยง · ข้อเสนอเปลือกแอป · ดีไซน์หน้าละไฟล์ · คำขอ API ใหม่ |
| คำขอ API ใหม่ | เขียนเป็น "ส่งอะไรเข้า → ได้อะไรกลับ → ใช้ในหน้าไหน" Claude จะทำฝั่งหลังบ้านให้แล้วอัปเดต `01-mobile-app-to-build.md` หัวข้อ 7 |
| **ห้าม** | ข้อมูลลูกค้าจริง เลขบัตร ทะเบียน เบอร์ อีเมล คีย์ — รีโปนี้**สาธารณะ** และมี pre-commit hook ปฏิเสธอยู่ |
| **โค้ดแอปทั้งโปรเจกต์** | **ไม่วางที่นี่** — ทำในรีโป GitHub **private** แยก (เช่น `dnaev-mobile`) แล้ววางไฟล์ `.md` บอกชื่อรีโป + สาขา + สรุปว่าทำอะไร Claude จะ clone ไปรีวิว |

## Claude ทำอะไรกับของที่วาง

1. ตรวจชื่อ API / ตาราง / ช่องข้อมูลที่อ้าง กับโค้ดจริง
2. ตรวจกับ**การใช้งานจริง** ของทีม (ฐาน production อ่านอย่างเดียว · ประวัติแก้ไข · แชตลูกค้า · บทเรียนที่เคยพลาด) — ของพวกนี้ GPT เข้าถึงไม่ได้
3. เขียนผลลง `to-gpt/YYYY-MM-DD-review-<หัวข้อ>.md`: ใช้ได้เลย / ต้องแก้ / ขัดกติกา ไม่ทำ + ไอเดียเพิ่มพร้อมหลักฐาน
4. ของที่ผ่าน → Claude ทำฝั่งหลังบ้าน/API ให้รองรับ

## กติกาการพัฒนาที่เจ้าของเคาะ (7 ก.ย. 2569)

- **ระหว่างพัฒนาใช้ข้อมูลตัวอย่างล้วน** (ชื่อสมมติ รหัสรถ DNA 1–14 ยอดเงินกลม ๆ) ติดป้าย "ตัวอย่าง" ชัด ๆ
- **ข้อมูลลูกค้าจริงเข้าแอปเป็นขั้นสุดท้ายเท่านั้น** — ตอน go-live โดย Claude ผ่าน API ของ Rental Ops หลังเจ้าของสั่ง · GPT ไม่แตะข้อมูลจริงในทุกขั้น
- แอปมือถือไม่มีฐานข้อมูลของตัวเอง — "โหลดข้อมูลลูกค้า" = ต่อกับ production จริง ไม่ใช่ copy ข้อมูลลงเครื่อง

---
*English summary: GPT/Codex drops finished files here (dated filenames, one topic per file). Claude pulls on every session start, reviews against the real codebase and real usage data, and replies in `to-gpt/`. No personal data — this repo is public with a PII-blocking pre-commit hook. Full app code goes to a separate private repo; link it from a `.md` here. Development uses sample data only; real customer data is connected last, by Claude, on the owner's go.*
