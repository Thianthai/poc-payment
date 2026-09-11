# CLAUDE.md — YPOC_PAYMENT

Project instructions สำหรับ repo นี้ ใช้ร่วมกับ global rules ที่ `~/.claude/CLAUDE.md`
ถ้าขัดกัน ให้ยึด global rules เป็นหลัก

## บริบทของ project

POC เรียก BO interface `I_JournalEntryTP` action `Post` ด้วย EML จาก ABAP Cloud
console class บน S/4HANA Cloud Public Edition เพื่อ post **incoming payment**

- **ไม่ใช่** production RICEFW — ไม่ต้องทำ RAP ของตัวเอง, ไม่ต้องทำ UI, ไม่ต้องทำ error framework
- ขอบเขตคือ "พิสูจน์ว่าเรียก `Post` แล้วได้เอกสารหน้าตาเดียวกับเอกสารตัวอย่าง" เท่านั้น
- **ไม่ clear open item** — ตกลงแล้ว 2026-09-11 ว่า POC นี้ post อย่างเดียว
  (`I_JournalEntryTP~Post` ไม่ clear ให้อยู่แล้ว · clearing เป็นเรื่องของ
  [`poc-clearing`](https://github.com/Thianthai/poc-clearing))
- ข้อมูลทดสอบ fix ใน code ทั้งหมด (constant + internal table ใน private section)
- ไม่มี communication scenario / arrangement — เรียก BO ในเครื่อง ไม่มี HTTP

## Naming ที่ใช้ใน project นี้

| Object | ชื่อ |
|---|---|
| Package | `YPOC_PAYMENT` |
| Console class | `YCL_PAYMENT` |

confirm แล้ว 2026-09-11 · prefix ตัวแปรตาม global rules
(`gc_` / `lv_` / `lo_` / `ls_` / `lt_` / `iv_` / `rv_` …)

## กฎเฉพาะ repo นี้

- **ห้ามเขียนไฟล์ ABAP ลง repo** (`*.clas.abap`, `*.xml` ของ ADT object ฯลฯ)
  ส่ง code เป็น code block ใน chat ให้ผู้ใช้ copy ไปสร้างใน ADT
  `docs/05-console-class.md` เก็บได้เพราะเป็น markdown snapshot ไม่ใช่ไฟล์ serialize
- `.abapgit.xml` และ `src/**/package.devc.xml` ให้ SAP serialize มาเองตอนผู้ใช้
  link abapGit จาก ADT — Claude ห้ามเขียนล่วงหน้า
- ค่าที่ยังไม่รู้ (company code, เลขบัญชี, customer) ให้ใส่เป็น placeholder ที่เห็นชัด
  เช่น `CHANGE_ME` แล้วบอกผู้ใช้ว่าต้องเติมอะไรบ้าง อย่าเดาค่าจริง
- **simulate guard**: `gc_simulate = abap_true` เป็นค่าเริ่มต้นเสมอ
  พิมพ์ payload ออก console โดยไม่ยิง `EXECUTE post` · post จริงต่อเมื่อผู้ใช้สั่ง
- ชื่อ field ใน `%param` / `_GLItems` / `_APARItems` ที่เขียนในเอกสารมาจาก community +
  ความจำ **ยังไม่ได้ยืนยันกับ tenant** — ให้ผู้ใช้เช็ค code completion ใน ADT
  (`TYPE TABLE FOR ACTION IMPORT i_journalentrytp~post`) แล้ว Claude แก้เอกสารตามของจริง

## จุดที่พลาดง่าย (รวบรวมจาก community · ยังไม่ได้เจอเองบน tenant)

- `Post` เรียกจาก **console class / local class ได้** (SAP doc มีตัวอย่าง)
  ที่ห้ามคือเรียกจาก RAP action / determination นอก save sequence ของ BO ตัวเอง
  → `BEHAVIOR_STATEMENT_ILLEGAL`
- post แล้ว fail = **short dump** ไม่ใช่ message สวย ๆ — SAP แนะนำให้เรียก `Validate`
  ก่อน `Post` แต่มีรายงานว่า `Validate` ไม่มีบนบาง public tenant → เช็คใน ADT ก่อน
- `Post` **ไม่ clear open item** — เอกสาร DZ ที่ได้จะค้างเป็น open item บน customer
- `Post` **ไม่มี test-run flag** → ใช้ simulate guard ในโค้ดแทน
- ต้อง `COMMIT ENTITIES` หลัง `MODIFY ENTITIES … EXECUTE post` ไม่งั้นไม่ได้เอกสาร
- เลขเอกสารจริงได้จาก `MAPPED` → `%pid` + `CONVERT KEY OF i_journalentrytp`
  หรือ `COMMIT ENTITIES RESPONSE OF i_journalentrytp`
- `_CurrencyAmount` ต้องระบุ `CurrencyRole = '00'` (transaction currency)
  ยอด **เครดิตใส่ค่าติดลบ** ไม่มี field DebitCreditCode แยก
