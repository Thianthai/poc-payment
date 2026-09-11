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

### ยังพิสูจน์ไม่ได้ (รออะไรอยู่)

| หัวข้อ | รอ |
|---|---|
| ชื่อ field จริงของ `_GLItems` / `_ARItems` | ✅ header ยืนยันแล้ว · รอ item entity จาก ADT |
| `Validate` มีบน tenant หรือไม่ | เช็คใน ADT |
| BO รับ `BusinessTransactionType = RFPI` หรือไม่ | ลอง post จริง |
| บรรทัดภาษี (003/004) ส่งเป็น `_GLItems` + `TaxCode` ได้ไหม หรือต้องใช้ `_TaxItems` | ลอง post จริง |
| post จริงได้เลขเอกสาร | รอขั้น 5 |

## งานที่ยังค้าง

| # | เรื่อง | สถานะ |
|---|---|---|
| 1 | ~~export payment ตัวอย่าง~~ | ✅ |
| 2 | ~~export **invoice** `9400000005` (Q1–Q5)~~ | ✅ mapping อยู่ใน docs/04 |
| 3 | ~~export payment `3300000017` แบบ `SELECT *` (Q6)~~ | ✅ ทุกบรรทัดใส่เอง ไม่มี auto line |
| 4 | ยืนยันชื่อ field / node ของ `Post` parameter จาก ADT (abstract entity) | ⬜ |
| 5 | เขียน `YCL_PAYMENT` | ⬜ รอข้อ 4 + คำตอบ requirement + confirm โครง class |
