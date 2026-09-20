# แผนการพัฒนา: จองห้องทำงานในห้องสมุด (Booking)

อ้างอิง: `spec .md` | Spec ID: SPEC-BKG-001 | Status: Draft v2

## 1. สรุปแนวทาง

1. ผู้ใช้บริการที่ยืนยันตัวตนแล้วเลือกวัน ห้อง และ slot ที่เปิดให้จองภายใน 7 วัน (IF-IDP-01, FR-BKG-01)
2. ระบบค้นหาสมาชิกด้วยรหัสนักศึกษาหรือรหัสบุคลากร และใช้ Member ID ภายในระบบ (IF-LIB-01, ASM-03)
3. ระบบตรวจสอบ booking หรือการใช้งานที่ซ้อนทับ ก่อนยืนยันการจอง (FR-BKG-02, ASM-04, ASM-06)
4. ระบบบันทึกการจองแบบป้องกันการจองซ้อน ออกหมายเลข และเปลี่ยนสถานะห้องใน slot ที่เลือก (FR-BKG-04, AC-BKG-01)
5. ระบบส่งการแจ้งเตือนแบบ asynchronous และ retry ภายใน 3 นาที สูงสุด 5 ครั้ง โดยไม่ขวางการบันทึกการจอง (IF-NOT-01, FR-BKG-05, NFR-REL-02)

## 2. เทคโนโลยีที่ใช้

| สิ่งที่เลือก | มาจาก | หมายเหตุ |
|---|---|---|
| React (Vite) | ทีมเลือกเอง ไม่ได้มาจาก spec | หน้าจอเลือกวัน ห้อง slot และผลการจอง รองรับ FR-BKG-01, FR-BKG-03, FR-BKG-06 |
| Python FastAPI | ทีมเลือกเอง ไม่ได้มาจาก spec | API สำหรับค้นหาความว่าง ตรวจสอบ และสร้างการจอง รองรับ FR-BKG-01 ถึง FR-BKG-06 |
| MySQL | CON-TECH-01 | ฐานข้อมูลการจอง สถานะห้อง และ audit log |
| TLS 1.2 ขึ้นไป | NFR-SEC-01 | ใช้กับการรับส่งข้อมูลระหว่าง client กับ API |
| กลไกคิวงาน asynchronous | IF-NOT-01 | ใช้ส่งข้อความและ retry โดยไม่รอผลส่งในคำขอจอง |

## 3. โมเดลข้อมูล

| Entity | ฟิลด์หลัก | รองรับ |
|---|---|---|
| Booking | `booking_id`, `member_id`, `room_id`, `booking_date`, `slot_id`, `status`, `created_at` | FR-BKG-02, FR-BKG-04, FR-BKG-05, AC-BKG-01, AC-BKG-02 |
| Room | `room_id`, `room_type`, `capacity`, `status` | FR-BKG-01, FR-BKG-03, FR-BKG-06 |
| BookingSlot | `slot_id`, `booking_date`, `start_time`, `end_time`, `availability_status` | FR-BKG-01, FR-BKG-03, FR-BKG-04, FR-BKG-06 |
| Member reference | `member_id` และรหัสอ้างอิงที่ใช้เรียกระบบสมาชิก | IF-LIB-01, ASM-03 |
| NotificationJob | `job_id`, `booking_id`, `channel`, `status`, `attempt_count`, `next_attempt_at` | IF-NOT-01, FR-BKG-05, NFR-REL-02, AC-BKG-04 |
| AuditLog | `audit_id`, `accessor_id`, `accessed_at`, `member_id`, `resource_type`, `resource_id` | DOM-PDPA-01, AC-BKG-06 |

ตาราง `Booking` จะเก็บเฉพาะ `member_id` เป็นข้อมูลอ้างอิงภายใน ไม่เก็บข้อมูลส่วนตัวที่ไม่จำเป็น (IF-LIB-01) การกำหนดชนิดข้อมูล ดัชนี และข้อจำกัดความเป็นเอกลักษณ์ต้องรองรับการตรวจสอบการจองซ้อนและ MySQL (CON-TECH-01, FR-BKG-02, FR-BKG-04)

## 4. API / หน้าจอ

### หน้าจอ

- หน้าเลือกวัน ห้อง และ slot: แสดงข้อมูลภายใน 7 วัน ห้องที่ว่างอยู่ทั้งวัน และ slot ตาม UC-09 (FR-BKG-01, ASM-01, ASM-08, ASM-09)
- หน้าแสดงผลการจอง: แสดงหมายเลขการจองเมื่อบันทึกสำเร็จหรือเมื่อส่งข้อความไม่สำเร็จ (FR-BKG-04, FR-BKG-05)
- ข้อเสนอทางเลือก: แสดงห้องหรือ slot ใกล้เคียง 3 ตัวเลือกในวันเดียวกัน เมื่อรายการเดิมถูกจองเต็มระหว่างยืนยัน (FR-BKG-03, ASM-05)
- หน้าปรับเงื่อนไขการค้นหา: รับประเภทห้องหรือจำนวนผู้ใช้ แล้วแสดงผลความว่างที่คำนวณใหม่ (FR-BKG-06)

### API

- `GET /booking/availability?from_date=&to_date=&room_type=&user_count=` รับช่วงวันที่ไม่เกิน 7 วันและเงื่อนไขห้อง/จำนวนผู้ใช้; ส่งห้องที่ว่างทั้งวันและ slot ที่ว่างตาม UC-09 (FR-BKG-01, FR-BKG-06)
- `POST /booking/validate` รับ `member_id`, `room_id`, `booking_date`, `slot_id`; ส่งผลตรวจสอบสิทธิ์สมาชิกและ booking/การใช้งานที่ซ้อนทับ พร้อมหมายเลขเดิมเมื่อปฏิเสธ (IF-LIB-01, ASM-03, FR-BKG-02)
- `POST /booking` รับ `member_id`, `room_id`, `booking_date`, `slot_id`; ส่งหมายเลขการจองและผลสำเร็จ หรือแจ้งว่าถูกจองเต็มพร้อมทางเลือก 3 รายการ (FR-BKG-03, FR-BKG-04, AC-BKG-01, AC-BKG-03)
- `GET /booking/{booking_id}` รับหมายเลขการจอง; ส่งรายละเอียดการจองที่ผู้ใช้มีสิทธิ์เข้าถึง และสร้าง audit log ทุกครั้งที่เข้าถึง (DOM-PDPA-01, AC-BKG-06)
- งาน `notification.send` รับ `booking_id` และช่องทาง Email/LINE; ส่งแบบ asynchronous และสร้าง/อัปเดตงาน retry ภายใน 3 นาที สูงสุด 5 ครั้ง (IF-NOT-01, FR-BKG-05, NFR-REL-02, AC-BKG-04)

การเรียก API ที่เกี่ยวกับข้อมูลการจองต้องได้รับผลยืนยันตัวตนจากระบบบัญชีมหาวิทยาลัยก่อน (IF-IDP-01) และการรับส่งข้อมูลต้องใช้ TLS 1.2 ขึ้นไป (NFR-SEC-01)

## 5. ตารางตรวจ Constraints

| Constraint ID | ถูกนำไปใช้ที่ไหนใน plan | สถานะ |
|---|---|---|
| CON-TECH-01 | เทคโนโลยี MySQL และโมเดลข้อมูล Booking, Room, BookingSlot, NotificationJob, AuditLog | ใช้แล้ว |
| DOM-PDPA-01 | Entity AuditLog และ endpoint `GET /booking/{booking_id}` บันทึก accessor, เวลา, Member ID; retention ไม่น้อยกว่า 1 ปี | ใช้แล้ว |
| IF-IDP-01 | ตรวจ identity ก่อนเรียก API ที่เกี่ยวกับการจองและข้อมูลการจอง | ใช้แล้ว |
| IF-LIB-01 | Member reference และ `POST /booking/validate` ใช้รหัสนักศึกษา/บุคลากรเพื่อค้น Member ID โดยไม่เก็บข้อมูลส่วนตัวเกินจำเป็น | ใช้แล้ว |
| IF-NOT-01 | งาน `notification.send` และ NotificationJob แบบ asynchronous ไม่รอผลส่งใน `POST /booking` | ใช้แล้ว |

## 6. แผนทดสอบจาก Acceptance Criteria

| AC ID | ชื่อ test | ทดสอบอย่างไร |
|---|---|---|
| AC-BKG-01 | `test_AC_BKG_01_create_booking` | เตรียมผู้ใช้ยืนยันตัวตนแล้วและ Study Room A slot 09.00-10.00 ว่าง; ยืนยันการจอง แล้วตรวจว่ามี Booking, มีหมายเลข และ slot เปลี่ยนเป็นไม่ว่าง |
| AC-BKG-02 | `test_AC_BKG_02_reject_overlapping_booking` | เตรียม booking ที่ยังไม่สิ้นสุดหรือสถานะกำลังใช้งานและเวลาซ้อนทับ; พยายามจองซ้ำ แล้วตรวจว่าถูกปฏิเสธและได้หมายเลข booking เดิม |
| AC-BKG-03 | `test_AC_BKG_03_offer_alternatives_without_duplicate` | ให้ผู้ใช้อีกคนยืนยันรายการเดียวกันก่อนคำขอที่สอง; ตรวจข้อความแจ้งเตือน ทางเลือก 3 รายการในวันเดียวกัน และไม่เกิด booking ซ้อน |
| AC-BKG-04 | `test_AC_BKG_04_retry_notification` | ทำให้ระบบแจ้งเตือนไม่ตอบสนอง; ยืนยันการจอง แล้วตรวจว่า Booking และหมายเลขยังอยู่ และ NotificationJob มีกำหนด retry ภายใน 3 นาที ไม่เกิน 5 ครั้ง |
| AC-BKG-05 | `test_AC_BKG_05_availability_p95` | สร้างการทดสอบโหลดผู้ใช้พร้อมกัน 200 คนที่ค้นหาความว่าง; วัด p95 ของ response time และตรวจว่าไม่เกิน 2 วินาที |
| AC-BKG-06 | `test_AC_BKG_06_booking_access_audit_log` | เปิดดูข้อมูลการจองของผู้ใช้; ตรวจ AuditLog ที่มีผู้เข้าถึง เวลา และ Member ID |

การทดสอบเพิ่มเติมควรผูกกับ NFR-SEC-01 และ NFR-USE-01 โดยตรวจ TLS 1.2 ขึ้นไป และทดสอบผู้ใช้ใหม่ 10 คนให้ 8 คนทำรายการสำเร็จภายใน 3 นาที (NFR-SEC-01, NFR-USE-01)

## 7. ลำดับงาน

1. กำหนด schema และดัชนี MySQL สำหรับ Booking, Room, BookingSlot, NotificationJob และ AuditLog โดยไม่เก็บข้อมูลส่วนตัวที่ไม่จำเป็น (CON-TECH-01, IF-LIB-01, DOM-PDPA-01)
2. เชื่อมผลยืนยันตัวตนและการค้นหา Member ID ก่อนอนุญาตการใช้งาน booking (IF-IDP-01, IF-LIB-01, ASM-03)
3. สร้างบริการค้นหาห้อง/slot ภายใน 7 วัน โดยใช้ข้อมูล slot จาก UC-09 และแสดงห้องที่ว่างทั้งวัน (FR-BKG-01, FR-BKG-06, ASM-01, ASM-08, ASM-09)
4. สร้างการตรวจสอบ booking หรือการใช้งานที่ซ้อนทับ และส่งหมายเลข booking เดิมเมื่อปฏิเสธ (FR-BKG-02, ASM-04, ASM-06, AC-BKG-02)
5. สร้างการยืนยันการจองแบบป้องกัน race condition บันทึก Booking ออกหมายเลข และเปลี่ยนสถานะ slot (FR-BKG-04, AC-BKG-01)
6. เพิ่มการตรวจกรณีรายการเต็มระหว่างยืนยัน พร้อมค้นหาและแสดงทางเลือก 3 รายการในวันเดียวกัน (FR-BKG-03, ASM-05, AC-BKG-03)
7. เพิ่มคิวแจ้งเตือนแบบ asynchronous และ retry ภายใน 3 นาที สูงสุด 5 ครั้ง โดยไม่ทำให้การจองล้มเหลว (IF-NOT-01, FR-BKG-05, NFR-REL-02, AC-BKG-04)
8. เพิ่ม audit log, TLS และการตรวจสิทธิ์การเข้าถึงข้อมูลการจอง (DOM-PDPA-01, NFR-SEC-01, AC-BKG-06)
9. ทดสอบตาม AC ทั้งหมด รวม load test 200 คนและ usability test ผู้ใช้ใหม่ 10 คน (AC-BKG-01 ถึง AC-BKG-06, NFR-PERF-01, NFR-USE-01)

## 8. สิ่งที่ยังไม่ทำ

Spec v2 ไม่มีรายการ Open Questions คงค้าง จึงไม่มีส่วนของฟีเจอร์ที่ต้องหยุดรอคำตอบตามหัวข้อนี้

ขอบเขต UC-02, UC-03, UC-09 และ UC-13 ที่ระบุเป็น Out of scope จะไม่สร้างในแผนนี้; แผนนี้เพียงอ่านข้อมูล slot จาก UC-09 และรับผลยืนยันตัวตนจาก UC-13 ตาม precondition (Out of scope, ASM-01, IF-IDP-01)
