# รับคำตอบ 3 ไฟล์ครบ → Native 0.9 (8 ก.ย. 2569)

อ่านตามลำดับที่เจ้าของสั่งจาก handoff d4082b9:
1. review-fleet-calendar-import
2. ai-eyes-benchmark-native-ios
3. proposal-one-inbox-ai-secretary

## สิ่งที่แก้/เริ่มแล้ว
- Native Swift iOS17+ เป็นเฟสแรกจริง: default app เป็น SwiftUI กล่องเดียว พร้อมตัวอย่างการ์ด12ชนิด, เธรดใบจอง และรถ/เดือน. Legacy Web ยังเก็บไว้ตรวจงานเดิม; ไม่อ้างว่าย้ายทุกฟีเจอร์เสร็จ.
- รถ12คัน,4รหัสDNAยืนยัน; normalize model display, identityเป็นid. DTOใช้ code/image/plate/canSeePlate ตามจริง; ไม่มีmodelKey/photoUrlสมมติ. Fixture nativeไม่มีทะเบียน.
- ใช้ path GET /api/bookings?scope=calendar&start=<ms>&end=<ms>; ไม่มี /api/calendar ที่ deploy. Domainอ่าน raw intervals; Bangkok/exclusiveend/union/pending/conflict. เพราะไม่มีcar_blocksและยังไม่เชื่อมauth จึงคงป้าย **ข้อมูลตัวอย่าง** และไม่แสดงfreeเป็นคิวจริง.
- Mapperแปลง ISO พ.ศ.→ค.ศ. พร้อมเตือนวันที่ก่อน/หลัง; ขาดเวลามีwarning startTime/endTime ไม่เติม09:00. returnDeadlineอยู่claimsไม่ยัดnotes/refundDueAt; เอกสารเคลมยอดpaidไม่กลายเป็นreceived. ยอดบันทึกจริงต้องอ่านserver summary.
- เพิ่ม VisionKit document camera → Apple Visionเดิมในเครื่อง; ไมค์Speech th-THเปิดเฉพาะเครื่องรองรับon-deviceจริง ไม่มีcloudfallback. กล้อง/เสียงต้องทดสอบiPhoneจริงต่อ.
- ค่าใช้จ่ายภาพ/benchmark AI รอบนี้0calls: reuseรูปเดิม. ผลOCR0.4วิเป็นผล C รายงาน ไม่ใช้เป็นการรับประกันแอปบนทุกเครื่อง.

## ผลตรวจ
Private source checkpoint 528842e. SwiftUI BUILD SUCCEEDED. Native UI3testsผ่าน:ไม่มีWKWebViewในจอใหม่,scrollถึงการ์ดท้ายสุด,keyboardไม่บังปุ่ม,เดือนDNA1 Sep9days/Oct4days,day6มีสองเวลา. รอบวันที่ไทย/พื้นการ์ดมืดทดสอบเฉพาะ2เคสผ่านเพิ่ม. Foundation calendar7testsผ่าน + JSmappertestsผ่าน.

## ขอบเขตจริง
การ์ด/ร่างยังเป็นreviewfixture,ไม่ได้แปลงข้อความอิสระลงทุกช่อง. ไม่ต่อlivewrites,LINE,สัญญา,เงินหรือAPNs. FaceIDเป็นการปลดล็อกcredentialหลังมีsession ไม่ใช่เครื่องมือสร้างteam sessionเอง. Ownerกำลังรับภาพnativeเพื่อตรวจรายละเอียด; ไม่อ้างว่าownerอนุมัติภาพแล้ว.

ถอนคำขอให้Cเข้าหรือเขียนprivateworkspaceGPT. ทะเบียน/จังหวัด/inactive/Chery/latefeeรอเจ้าของ ไม่ถามCซ้ำ. ท่อAPIที่ยังไม่deployรอประกาศชื่อและสถานะจริงก่อนต่อ.

หมายเหตุการสื่อสาร privacy: OCRในเครื่องไม่ส่งรูปออก แต่ถ้านำraw OCR textส่งPOST read-document ข้อความส่วนบุคคลย่อมออกจากเครื่องไปbackend จึงห้ามใช้คำว่าเลขบัตรไม่ออกจากเครื่องกับโหมดนั้น. รอบนี้แอปยังไม่ส่งทั้งรูปและข้อความ.
