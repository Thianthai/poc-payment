# 02 — Object List

สถานะ ณ วันที่อัปเดตล่าสุด · Claude อัปเดตตารางนี้ตามที่เห็นใน `git log`
หลังผู้ใช้ push object ผ่าน abapGit

| # | Object | Type | ใครสร้าง | Status |
|---|---|---|---|---|
| 1 | `YPOC_PAYMENT` | Package | ผู้ใช้ (ADT) | ✅ [`src/package.devc.xml`](../src/package.devc.xml) |
| 2 | `YCL_PAYMENT` | Class (console, `IF_OO_ADT_CLASSRUN`) | ผู้ใช้ (ADT) | ⬜ |

Legend: ⬜ ยังไม่สร้าง · 🟡 สร้างแล้วยังไม่ push · ✅ push ขึ้น repo แล้ว

ชื่อ confirm แล้ว 2026-09-11 · abapGit serialize ด้วย `FOLDER_LOGIC = FULL`, `STARTING_FOLDER = /src/`

## Config ที่ไม่ใช่ repository object

ไม่มี — `I_JournalEntryTP` เรียกในเครื่อง ไม่ต้องมี communication scenario / arrangement

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
| 1 | ~~export payment ตัวอย่าง~~ | ✅ |
| 2 | ~~export **invoice** `9400000005` (Q1–Q5)~~ | ✅ mapping อยู่ใน docs/04 |
| 3 | ~~export payment `3300000017` แบบ `SELECT *` (Q6)~~ | ✅ ทุกบรรทัดใส่เอง ไม่มี auto line |
| 4 | ~~ยืนยันชื่อ field / node ของ `Post` parameter จาก ADT~~ | ✅ |
| 5 | ~~เขียน `YCL_PAYMENT`~~ | 🟡 activate + simulate ผ่านแล้ว 2026-09-11 · ยังไม่ push abapGit |
| 6 | ~~post จริง~~ | ✅ `3300000024` |
| 7 | เทียบ `3300000024` กับ `3300000017` · แก้ dump หลัง commit · คืน `gc_simulate = abap_true` · push class ผ่าน abapGit | ⬜ |
