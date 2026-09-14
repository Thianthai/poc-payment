# 02 — Object List

สถานะ ณ วันที่อัปเดตล่าสุด · Claude อัปเดตตารางนี้ตามที่เห็นใน `git log`
หลังผู้ใช้ push object ผ่าน abapGit

| # | Object | Type | ใครสร้าง | Status |
|---|---|---|---|---|
| 1 | `YPOC_PAYMENT` | Package | ผู้ใช้ (ADT) | ✅ [`src/package.devc.xml`](../src/package.devc.xml) |
| 2 | `YCL_PAYMENT` | Class (console, `IF_OO_ADT_CLASSRUN`) | ผู้ใช้ (ADT) | 🟡 rev 17 final ส่งแล้ว 2026-09-14 · รอ activate + push abapGit |

Legend: ⬜ ยังไม่สร้าง · 🟡 สร้างแล้วยังไม่ push · ✅ push ขึ้น repo แล้ว

ชื่อ confirm แล้ว 2026-09-11 · abapGit serialize ด้วย `FOLDER_LOGIC = FULL`, `STARTING_FOLDER = /src/`

## Config ที่ไม่ใช่ repository object

ไม่มี communication scenario / arrangement — `I_JournalEntryTP` เรียกในเครื่อง

| # | สิ่งที่ทำ | ที่ | Status |
|---|---|---|---|
| C1 | **Custom Logic `YY1_FIN_ACDOC_ITEM_SUBSTITUTIO`** (BAdI `FIN_ACDOC_ITEM_SUBSTITUTION`) — ถ้า `AccountingDocumentType = SA` และ `TaxCode = O1` → `AssignmentReference = '94000000052026001'` (fix ค่าสำหรับ POC · functional ต้องการเพื่อรายงานภาษี) | Fiori: Custom Logic (Key User Extensibility) · published 2026-09-11 11:35 | ✅ อยู่นอก abapGit |

> ตอนทำจริง BAdI ต้อง derive จาก invoice ไม่ใช่ fix — ทางที่เป็นไปได้: `YCL_PAYMENT` ส่ง key ของ invoice
> มากับ header ใบ SA (`Reference1InDocumentHeader` / `AccountingDocumentHeaderText`) แล้ว BAdI อ่านจาก
> `accountingdocheader` · และเลข item ท้าย (`…001` vs ตัวอย่าง `…003`) ต้องให้ functional ยืนยัน

## ลำดับการทำ

```
1. Object 1          สร้าง package บน ADT → link abapGit → push ให้ SAP serialize baseline
2. export data       รัน SQL ใน docs/04 กับเอกสารตัวอย่าง → ส่งผลกลับมา
3. Object 2          copy console class จาก chat → รัน F9 แบบ simulate
4. เทียบ payload      กับเอกสารตัวอย่าง จนตรงทุกบรรทัด
5. post จริง          gc_simulate = abap_false → เทียบเอกสารที่ได้กับตัวอย่าง
```

## Log การทดสอบ

| วันที่ | ทำอะไร | ผล |
|---|---|---|
| 2026-09-11 | เปิด project · confirm scope (post only, no clearing) + ชื่อ object | ✅ |
| 2026-09-11 | สร้าง `YPOC_PAYMENT` + link abapGit + push baseline | ✅ `.abapgit.xml` + `package.devc.xml` จาก SAP |
| 2026-09-11 | export เอกสารตัวอย่าง `3300000017` ด้วย Q1–Q3 | ✅ ทุก field compile ผ่าน · 5 บรรทัด (4 G/L + 1 customer) · `BusinessTransactionType = RFPI` |
| 2026-09-11 | requirement เพิ่ม: payment ต้อง derive จาก invoice · export invoice `9400000005` Q1–Q5 | ✅ invoice open · customer line มี `WithholdingTaxCode XX` · deferred tax line เป็น `ItemType T` base 5,999 |
| 2026-09-11 | export payment `3300000017` แบบ `SELECT *` (Q6) | ✅ ทั้ง 5 บรรทัดใส่เอง (`IsAutomaticallyCreated` ว่าง) · WHT line เป็น G/L ธรรมดา · tax line เป็น G/L + tax code |
| 2026-09-11 | ยืนยัน field ของ `Post` parameter จาก ADT (4 abstract entity) · ส่ง code `YCL_PAYMENT` รอบแรก | ✅ docs/05 · `gc_simulate = abap_true` |
| 2026-09-11 | activate `YCL_PAYMENT` | ✅ หลังแก้ 2 จุด: inline `DATA( )` ใน IMPORTING ของ functional call ใน `IF` ใช้ไม่ได้ · `acpi_zuonr` ไม่ released → ใช้ `TYPE c LENGTH 18` เอง |
| 2026-09-11 | รัน F9 simulate (invoice `9400000005`) | ✅ payload 5 บรรทัดตรงกับ `3300000017` ทุก field · balance 0 · **ฝั่ง derive ปิดจ๊อบ** |
| 2026-09-11 | reverse payment ตัวอย่าง `3300000017` ผ่าน Manage Journal Entries (reason 01) | ✅ reversal document **`3300000019`** · invoice `9400000005` ยัง open |
| 2026-09-11 | **post จริงครั้งที่ 1** (`gc_simulate = abap_false`) | ❌ `FAILED` ไม่ dump — `Time dependent taxes: tax date has to be filled from caller` / `For tax code DM/O1, the key date 00/00/0000 does not fall in any validity period` → เพิ่ม `TaxDeterminationDate` ใน header · ไม่มี MSG เรื่อง `RFPI` |
| 2026-09-11 | **post จริงครั้งที่ 2** (+ `TaxDeterminationDate`) | ❌ `FAILED` — `Tax statement item missing for tax code DM` (G/L line + tax code ถูกมองเป็น base line → ต้องย้าย [3]/[4] ไป `_TaxItems`) · `Enter a business place.` (Thai → `BusinessPlace = 0000` ตาม Q6) · tax date ผ่านแล้ว |
| 2026-09-11 | ส่ง rev 3: tax line → `_TaxItems` (`MWS`, `IsDirectTaxPosting`) · `BusinessPlace 0000` ทุก G/L / AR line | ✅ ส่งแล้ว |
| 2026-09-11 | **post จริงครั้งที่ 3** (rev 3) | ❌ `FAILED` — `G/L account item without tax code in document $ 1 with deferred taxes` · tax statement / business place ผ่านแล้ว · = กฎ deferred tax: ทุก G/L line ต้องมี tax code → บรรทัด DM/O1 ของตัวอย่างเป็นของที่ Post Incoming Payments generate ตอน clear ไม่ใช่ post มือ · รอผู้ใช้เลือก A (ตัด tax line เหลือ 3 บรรทัด) / B (ใส่ tax code 0% บน bank/WHT) |
| 2026-09-11 | เลือก **A** → ส่ง rev 4 (comment `_taxitems` ออก · 3 บรรทัด) | ✅ |
| 2026-09-11 | **post จริงครั้งที่ 4–6** (rev 4) | ✅ **POST สำเร็จ** — ได้ `3300000020`, `3300000021` (กดซ้ำ · reverse ทิ้งแล้ว) และ **`3300000024`** · console ว่างเพราะ dump หลัง commit (คาดว่า `CONVERT KEY` ใช้นอก save sequence ไม่ได้) → รอ dump + export เอกสารเทียบ |
| 2026-09-11 | Feed Reader: `BEHAVIOR_STATEMENT_ILLEGAL` — `Statement "CONVERT KEY" is not allowed with this status` ใน `post_entry` (เหมือนกันทั้ง 3 ครั้ง) → ส่ง rev 5 ตัด `CONVERT KEY` + `gc_simulate = abap_true` | ✅ |
| 2026-09-11 | export `3300000024` Q2+Q3+Q6 เทียบกับ `3300000017` | ⚠️ bank + WHT + header ตรงทุก field · **customer line PK 11 (credit memo) แทน 15** + `WithholdingTaxCode` ว่าง · ไม่มี DM/O1 ตาม option A · รอผู้ใช้เลือกปิด POC / ลอง `_WithHoldingTaxItems` |
| 2026-09-11 | ผู้ใช้สั่ง: หาทาง post DM/O1 ให้ได้ก่อน | 🟡 ทาง 1 = tax code 0% บน bank/WHT (รอ Q7/Q8) · ทาง 2 = เอกสารแยก `_TaxItems` อย่างเดียว |
| 2026-09-11 | Q7/Q8: bank `0011092001` tax category ว่าง (ใส่ tax code ไม่ได้) · WHT `0011047003` = `*` · มี `O0` 0% | 🟡 แผน rev 6: `O0` บน WHT line + คืน `_TaxItems` |
| 2026-09-11 | ผู้ใช้ระบุข้อกำหนดเต็ม: 5 บรรทัด **customer ต้องเป็น PK 15** | ❌ **PK 15 ทำไม่ได้ด้วย `I_JournalEntryTP`** — API สร้าง AR ได้แค่ 01/11 (SAP doc) · ไม่มี field posting key · พิสูจน์แล้ว `3300000024` = 11 → รอผู้ใช้/functional ตัดสิน: ยอมรับ 11 (ทำ rev 6 ต่อ) หรือเปลี่ยน API เป็น Bank Statement |
| 2026-09-11 | สำรวจ API ทางเลือก (docs/01) — ไม่มี API post incoming payment บน Public Edition · ทางมาตรฐาน = Bank Statement (`SAP_COM_0316`) | 🟡 เสนอหยุดไล่ DM/O1 บน JE Post · รอผู้ใช้ตอบว่า payment จริงมาจากไหน |
| 2026-09-11 | ผู้ใช้สั่งตัด PK 15 ออกก่อน ลองเอา DM/O1 ให้ได้ → ส่ง rev 6 (`O0` บน WHT line + คืน `_TaxItems`) | ✅ ส่งแล้ว |
| 2026-09-11 | **post จริงครั้งที่ 7** (rev 6) | ❌ `G/L account item without tax code in document with deferred taxes` **เหมือนเดิม** → check ดูทุก G/L line รวม bank ที่ใส่ tax code ไม่ได้ → **เอกสารเดียว 5 บรรทัดปิดสนิท** · เหลือทาง 2 เอกสาร (DZ 3 บรรทัด + SA `_TaxItems` ล้วน) |
| 2026-09-11 | rev 6b: ถอด constants ให้อ่านง่าย (ผู้ใช้ขอ) · ส่ง **rev 7**: 2 เอกสารใน commit เดียว (DZ + SA `_TaxItems` ล้วน) | ✅ ส่งแล้ว (แก้ `%cid` → `%pid` ใน late REPORTED) |
| 2026-09-11 | **post จริงครั้งที่ 8** (rev 7) | ⚠️ เอกสาร 1 DZ: **`Document check - no errors`** · เอกสาร 2 SA: `Enter a business place.` + `KSCHL is empty` → rollback ทั้งคู่ |
| 2026-09-11 | อ่าน doc ProductTaxItem | `ConditionType` required เมื่อ classification→condition เป็น 1:n · `TaxDeterminationDate` บน tax item = "Do not use" · **ไม่มี field BusinessPlace** ใน tax item / header → เอกสาร tax item ล้วนน่าจะ post ไม่ได้บน TH · เสนอ rev 8 (`MWAS`, ลบ tax date) เพื่อ isolate blocker |
| 2026-09-11 | ส่ง rev 8 | ✅ |
| 2026-09-11 | **post จริงครั้งที่ 9** (rev 8) | ❌ เอกสาร 1 `no errors` · เอกสาร 2 เหลือ **`Enter a business place.` ตัวเดียว** (`MWAS` ถูก) → tax item ล้วนไม่มีที่ใส่ business place → **ทางเอกสารแยกตัน** · รอผู้ใช้เลือก ปิด POC / ลองลูกเล่นสุดท้าย (G/L line + tax code + tax item amount 0) |
| 2026-09-11 | ผู้ใช้ขอลอง rev 9: เอกสารเดียว 5 บรรทัด เรียง customer/WHT/DM/O1/bank | ❌ **post จริงครั้งที่ 10** — `G/L account item without tax code in document with deferred taxes` เหมือนเดิม → ลำดับไม่มีผล · สรุป: บน TH `I_JournalEntryTP` post DM→O1 ไม่ได้ทั้งรวมใบและแยกใบ |
| 2026-09-11 | ผู้ใช้ขอลอง rev 10: tax line เป็น `_GLItems` ระบุ tax account ตรง ๆ ไม่มี tax code | ❌ **post จริงครั้งที่ 11** — `G/L account 21082003 requires a valid tax code` → tax account บังคับ tax code · **ครบ 4 ทาง 4 กำแพง** (G/L ไม่มี code / G/L + code / tax item รวมใบ / tax item แยกใบ) |
| 2026-09-11 | ผู้ใช้ขอลอง rev 11: `_GLItems` [3]/[4] + tax code DM/O1 (re-test rev 2 พร้อม business place) | ❌ **post จริงครั้งที่ 12** — `Tax statement item missing for tax code DM` เหมือน rev 2 → G/L + tax code = base line เสมอ · เติม tax item ก็จะไปชน deferred check ที่ bank (rev 3/6/9) → **ไม่มีทางเหลือ** เสนอปิด POC |
| 2026-09-11 | ผู้ใช้ขอลอง rev 12: G/L + tax code + `_TaxItems` amount 0 อ้าง base line | ❌ **post จริงครั้งที่ 13** — FF 817 `Taxes by item is not activated and therefore not permitted in transfer` → `TaxItemAcctgDocItemRef` ใช้ได้เฉพาะ line-by-line tax (US/CA/BR) · ส่ง rev 13 ลบออก |
| 2026-09-11 | ส่ง rev 13 | ❌ **post จริงครั้งที่ 14** — `Entry of tax for DM 003 1000 0021082005 > is not possible because of tax base 0` → G/L line บน tax account = direct tax line หา base จาก tax item **เลข item เดียวกัน** (ของเราอยู่ item 6/7) · ผ่าน `Tax statement item missing` แล้ว |
| 2026-09-11 | ส่ง rev 14: tax item เลข 3/4 ตรงกับ G/L line · direct · ยอด/base จริง | ❌ **post จริงครั้งที่ 15** — `FI/CO interface: Line item entered several times` → item number ต้อง unique ข้าม node · BO ไม่รวม G/L + tax item แบบ BAPI |
| 2026-09-11 | ส่ง rev 15: tax item 6/7 direct amount 0 base ±5,999 (ตัวเลือกสุดท้าย) | ❌ **post จริงครั้งที่ 16** — `tax base 0` เหมือน rev 13 → check อ่าน base ของ G/L line เอง · `TaxBaseAmount` บน G/L item ไม่ถูกส่งต่อ · **หมดทุกทาง (16 รอบ)** → เสนอปิด POC |
| 2026-09-11 | ผู้ใช้ขอกลับไปเวอร์ชัน 2 ใบ (rev 8) แล้วเปลี่ยนใบ 2 เป็น `_GLItems` + tax code DM/O1 + business place (rev 16) | ❌ **post จริงครั้งที่ 17** — ใบ 1 `no errors` · ใบ 2 `Tax statement item missing for tax code DM` (G/L + tax code = base line ไม่ว่าใบไหน) · เสนอ rev 17 เติม tax item amount 0 ในใบ 2 (คาดว่าชน `tax base 0`) |
| 2026-09-14 | **functional ยืนยัน: split เป็น 2 ใบได้** (DZ payment + ใบโอน deferred tax แยก) | ✅ |
| 2026-09-14 | ✅ **POST ผ่านทั้ง 2 ใบ** — payment **`3300000026`** (DZ 3 บรรทัด · ref `POC0911113752`) + deferred tax **`7200000001`** (SA · ref `09060002`) · reverse แล้วเป็น `3300000030` / `7900000000` | ใบ 2 = คู่ dummy `_GLItems` **`0011054001`** ±419.93 (ไม่มี tax code · มี business place) + `_TaxItems` DM/O1 direct `MWAS` base ±5,999 → คู่ G/L ให้ tax item derive business place · ใบไม่มี bank/WHT → deferred check ไม่ฟ้อง |
| 2026-09-14 | เทียบ `7200000001` บรรทัด 003/004 กับ `3300000017` | ✅ G/L · DM/O1 · ±419.93 · base ±5,999 · TaxType A · group 001/002 · bplace · TaxDate เหมือน · ⚠️ `TransactionTypeDetermination` ว่าง (ตัวอย่าง `MWS`) · assignment O1 = `…001` (derive เอง) · มีคู่ `0011054001` เพิ่ม |
| 2026-09-14 | Q7 `0011054001`: tax category ว่าง (เหมือน bank) · ไม่ใช่ OIM | → check deferred tax ไม่ได้ขึ้นกับบัญชี แต่ขึ้นกับโครงเอกสาร (ใบไม่มี AR line / RFBU) · assignment O1 มาจาก BAdI C1 |
| 2026-09-14 | ส่ง **rev 17 final** (2 ใบ · `lv_simulate = abap_true`) · sync snapshot / README / docs/01 | ✅ รอผู้ใช้ activate + push abapGit (ยังไม่ push repo ตามที่สั่ง) |
| 2026-09-14 | **feedback จาก POC clearing**: clear `3300000026` ไม่ได้ — AIF `/FINAC`: "The open items display different withholding tax information from the relevant business partner master record" · customer `0001000082` มี WHT type ใน master แต่บรรทัดลูกหนี้ที่ API สร้างไม่มี WHT info (`WithholdingTaxCode` ว่าง) · Fiori เติมจาก master ให้เอง API ไม่เติม | 🟡 rev 18: เพิ่ม `_WithHoldingTaxItems` (amount 0 · ทุก type ที่ master มี) · ได้ field list + master (1 type: `MA`/`09`, WHT agent Yes) → ส่ง rev 18 |
| 2026-09-14 | ส่ง **rev 18**: ใบ 1 + `_WithHoldingTaxItems` MA/09 amount 0 manual · `lv_simulate = abap_false` | ✅ **post ผ่านทั้ง 2 ใบ** — DZ **`3300000031`** (ref `POC0914055500`) + SA **`7200000002`** · commit MSG: `Document posted successfully: BKPFF 330000003110002026` / `720000000210002026` · ส่งให้ POC clearing ทดสอบ |
| 2026-09-14 | Q6 `3300000031` + header 3 ใบ | ✅ บรรทัดลูกหนี้ `WithholdingTaxCode = XX` แล้ว (เหมือน `3300000017/005`) · ⚠️ ref ใบ SA ถูกทับเป็น `09060003` (ใบก่อน `09060002`) — น่าจะมี header substitution generate ref แบบ `MMDD+running` ให้ SA · ใบ DZ คง `POC…` · `POSTED:` query จึงเจอแค่ DZ → rev 19 จะ parse เลขจาก MSG commit แทน |

### ยังพิสูจน์ไม่ได้ (รออะไรอยู่)

| หัวข้อ | รอ |
|---|---|
| ~~ชื่อ field จริงของ `_GLItems` / `_ARItems` / `_CurrencyAmount`~~ | ✅ paste จาก ADT ครบแล้ว 2026-09-11 |
| `Validate` มีบน tenant หรือไม่ | เช็คใน ADT |
| ~~BO รับ `BusinessTransactionType = RFPI` หรือไม่~~ | ✅ รับ — post ผ่านด้วย `RFPI` |
| ~~บรรทัดภาษี (003/004) ส่งเป็น `_GLItems` + `TaxCode` ได้ไหม~~ | ❌ ไม่ได้ (`Tax statement item missing`) → rev 3 ใช้ `_TaxItems` + `MWS` + direct · รอผล post |
| ~~post จริงได้เลขเอกสาร~~ | ✅ `3300000024` (2026-09-11) |
| ~~console ว่างหลัง post — dump ตรงไหน~~ | ✅ `BEHAVIOR_STATEMENT_ILLEGAL` ที่ `CONVERT KEY` (post_entry บรรทัด 39) → rev 5 ตัดออก |
| ~~เอกสารที่ได้เหมือน `3300000017` ไหม~~ | ✅ เทียบแล้ว (docs/04) — bank/WHT/header เหมือน · ต่าง: ไม่มี DM/O1 (option A) · customer PK **11** แทน 15 · `WithholdingTaxCode` ว่าง |

## งานที่ยังค้าง

| # | เรื่อง | สถานะ |
|---|---|---|
| 1 | activate rev 17 + F9 simulate เช็ค payload 2 ใบ | ⬜ |
| 2 | push `YCL_PAYMENT` ขึ้น abapGit → Claude อัปเดต status เป็น ✅ | ⬜ |
| 3 | (production) BAdI derive assignment จาก invoice แทนค่า fix · เลข item ท้าย `…001` vs `…003` ให้ functional ยืนยัน | ⏸️ นอก scope POC |
| 4 | (production) PK 15 / clearing / WHT code — ต้องเปลี่ยน API (Bank Statement) หรือยอมรับ PK 11 + Clearing API | ⏸️ นอก scope POC |
