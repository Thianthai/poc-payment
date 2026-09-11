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
- **Input คือ invoice** (company code + เลขเอกสาร + ปี) — class อ่านบรรทัด invoice จาก
  released CDS แล้ว derive บรรทัด payment (customer / deferred tax → output tax / WHT / bank)
  ไม่ใช่ fix 5 บรรทัดลง constant · ตัวอย่าง invoice `9400000005` → payment `3300000017`
  (requirement เพิ่ม 2026-09-11)
- ค่าที่ไม่ได้มาจาก invoice (house bank, tax code mapping, อัตรา WHT ถ้า fix) เป็น constant
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
- ชื่อ field ของ `Post` parameter ให้ยึด abstract entity จริงบน tenant
  (`D_JournalEntryPostParameter` + `D_JournalEntryPost*ItemP`) ที่ผู้ใช้ paste จาก ADT
  ตามที่บันทึกใน `docs/01` — ห้ามเดาชื่อ field / node เอง
  (node จริงคือ `_GLItems` / `_ARItems` / `_APItems` / `_TaxItems` / `_WithHoldingTaxItems`
  ไม่ใช่ `_APARItems` / `_ProductTaxItems` ที่ community เขียน)

## จุดที่พลาดง่าย

### เจอเองบน tenant แล้ว

- **`TaxDeterminationDate` บังคับ** ถ้า company code เปิด time-dependent tax — ไม่ส่งจะได้
  `Time dependent taxes: tax date has to be filled from caller` +
  `For tax code DM, the key date 00/00/0000 does not fall in any validity period` (2026-09-11)
- `EXECUTE post` ที่ fail ได้ `FAILED` + `REPORTED` ปกติ **ไม่ dump** (อย่างน้อยเคส validation)
- **`CONVERT KEY OF i_journalentrytp FROM %pid` ใน console class = dump `BEHAVIOR_STATEMENT_ILLEGAL`**
  (ใช้ได้เฉพาะ save phase ของ RAP BO) — และ dump เกิด**หลัง** `COMMIT ENTITIES` เอกสารจึง post ไปแล้ว
  แต่ console ว่าง → หาเลขเอกสารด้วย `SELECT I_JournalEntry WHERE DocumentReferenceID` แทน (2026-09-11)
- console ว่างเปล่าหลัง F9 = dump · ดูที่ ADT Feed Reader → Runtime Errors
- **บรรทัดภาษีส่งเป็น `_GLItems` + `TaxCode` ไม่ได้** — ระบบมองเป็น base line แล้วฟ้อง
  `Tax statement item missing for tax code DM` → ต้องใช้ `_TaxItems` (2026-09-11)
- **Thai localization ต้องส่ง `BusinessPlace`** (`0000` = head office ตามเอกสารตัวอย่าง)
  ไม่งั้น `Enter a business place.` (2026-09-11)
- **เอกสารที่มี deferred tax code (`DM`) ทุก G/L line ต้องมี tax code** —
  `G/L account item without tax code in document with deferred taxes` (2026-09-11)
  → บรรทัดโอน DM→O1 ในเอกสารตัวอย่าง `3300000017` เป็นของที่ app Post Incoming Payments
  generate ตอน **clear** invoice ไม่ใช่บรรทัดที่ post มือ · API post ไม่ clear จึงทำซ้ำไม่ได้
  · check นี้ดู **ทุก** G/L line รวมบัญชี bank ที่ tax category ว่าง (ใส่ tax code ไม่ได้) —
  พิสูจน์ rev 6 (`O0` บน WHT line ก็ยัง fail) → payment + deferred tax transfer ต้องเป็น**คนละเอกสาร**
- **เอกสารที่มีแต่ `_TaxItems` post ไม่ได้บน TH** — `Enter a business place.` · tax item ไม่มี field
  BusinessPlace (derive จาก G/L/AR line ในเอกสารเดียวกันเท่านั้น) · `ConditionType = 'MWAS'` ต้องใส่
  ไม่งั้น `KSCHL is empty` · `TaxDeterminationDate` บน tax item ห้ามใส่ (doc) (rev 7–8, 2026-09-11)
- **tax account (`0021082005` / `0021082003`, tax category `>`) บังคับ tax code** —
  post เป็น G/L line เปล่า ๆ ได้ `G/L account 21082003 requires a valid tax code` (rev 10)
  · post เป็น G/L + tax code = base line → `Tax statement item missing` (rev 2)
  → บรรทัด DM→O1 ผ่าน API นี้ไม่ได้เลยบน TH ไม่ว่าทางไหน (สรุป 2026-09-11)
- ทางที่ลองแล้วทั้งหมด (rev 2–15): G/L + tax code + tax item amount 0 → `Entry of tax for DM 003 … tax base 0`
  (check อ่าน base ของ G/L line เอง · `TaxBaseAmount` ใน `_CurrencyAmount` ของ G/L item ไม่มีผล) ·
  tax item เลข item ซ้ำกับ G/L → `Line item entered several times` (BO ไม่รวมบรรทัดแบบ BAPI) ·
  `TaxItemAcctgDocItemRef` → FF 817 (TH ไม่ใช่ taxes-by-item) — **ห้ามเสียเวลาลองซ้ำ**
- inline `DATA( )` ใน `IMPORTING` ของ functional method call ที่อยู่ใน `IF` ใช้ไม่ได้
- data element `acpi_zuonr` (assignment) ไม่ released → ใช้ `TYPE c LENGTH 18` เอง

### จากชุมชน (ยังไม่ได้เจอเอง)

- `Post` เรียกจาก **console class / local class ได้** (SAP doc มีตัวอย่าง)
  ที่ห้ามคือเรียกจาก RAP action / determination นอก save sequence ของ BO ตัวเอง
  → `BEHAVIOR_STATEMENT_ILLEGAL`
- post แล้ว fail = **short dump** ไม่ใช่ message สวย ๆ — SAP แนะนำให้เรียก `Validate`
  ก่อน `Post` แต่มีรายงานว่า `Validate` ไม่มีบนบาง public tenant → เช็คใน ADT ก่อน
- `Post` **ไม่ clear open item** — เอกสาร DZ ที่ได้จะค้างเป็น open item บน customer
- `Post` **ไม่มี test-run flag** → ใช้ simulate guard ในโค้ดแทน
- ต้อง `COMMIT ENTITIES` หลัง `MODIFY ENTITIES … EXECUTE post` ไม่งั้นไม่ได้เอกสาร
- `_CurrencyAmount` ต้องระบุ `CurrencyRole = '00'` (transaction currency)
  ยอด **เครดิตใส่ค่าติดลบ** ไม่มี field DebitCreditCode แยก
