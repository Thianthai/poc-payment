# 01 — API Reference: BO Interface `I_JournalEntryTP` · action `Post`

## ข้อมูลพื้นฐาน

| หัวข้อ | ค่า |
|---|---|
| API Hub | <https://api.sap.com/bointerface/I_JOURNALENTRYTP> (ต้อง sign in ถึงจะเห็น field list) |
| ประเภท | Released **BO interface** (RAP) — เรียกด้วย EML ในเครื่อง ไม่ใช่ HTTP |
| Entity | `JournalEntry` |
| Operation | `Post` (action) · `Validate` (function — บาง tenant ไม่มี) · `Reverse` · `Change` |
| Underlying BO | `R_JournalEntryTP` |
| Protocol | synchronous — `MODIFY ENTITIES` แล้ว `COMMIT ENTITIES` ได้เลขเอกสารทันที |
| Auth / comm setup | ไม่ต้องมี — ใช้สิทธิ์ของ user ที่รัน console class |

`Post` รองรับ posting ไปที่ G/L, customer / supplier (reconciliation account),
tax account, one extension ledger — **ไม่รองรับ clearing open item**

## รูปแบบการเรียก

```abap
DATA lt_entry TYPE TABLE FOR ACTION IMPORT i_journalentrytp~post.

" เติม %cid + %param (ดู header / item ด้านล่าง)

MODIFY ENTITIES OF i_journalentrytp
  ENTITY journalentry
  EXECUTE post FROM lt_entry
  MAPPED   DATA(ls_mapped)
  FAILED   DATA(ls_failed)
  REPORTED DATA(ls_reported).

COMMIT ENTITIES
  RESPONSE OF i_journalentrytp
  FAILED   DATA(ls_commit_failed)
  REPORTED DATA(ls_commit_reported).
```

ข้อจำกัดเรื่องที่เรียก (จาก SAP):

- เรียกจาก **local program / console class ได้** ← ทางที่ POC นี้ใช้
- ถ้าจะเรียกจาก RAP BO ของตัวเอง ทำได้เฉพาะใน save sequence
  (`finalize` / determination on save / `adjust_numbers`) ไม่งั้น `BEHAVIOR_STATEMENT_ILLEGAL`
- fail ตอน post = **short dump** → SAP แนะนำให้ `Validate` ก่อน

## โครงสร้าง `%param` (จาก community — ยังไม่ยืนยันกับ tenant)

> ⚠️ ชื่อ field ด้านล่างประกอบจากตัวอย่างใน SAP Community + SOAP API ตัวพี่น้อง
> ต้องเช็ค code completion ใน ADT ก่อนใช้จริง แล้วแก้ตารางนี้ให้ตรง

### Header

| Field | Necessity | หมายเหตุ |
|---|---|---|
| `CompanyCode` | mandatory | |
| `BusinessTransactionType` | mandatory | `RFBU` = FI posting |
| `AccountingDocumentType` | mandatory | incoming payment คาดว่า `DZ` |
| `DocumentDate` | mandatory | |
| `PostingDate` | mandatory | ต้องอยู่ใน period ที่เปิด |
| `CreatedByUser` | mandatory | |
| `DocumentReferenceID` | optional | reference (XBLNR) |
| `AccountingDocumentHeaderText` | optional | header text (BKTXT) |
| `AccountingDocument` | optional | external number — ไม่ใช้ |
| `_GLItems` | 0..n | บรรทัด G/L (bank, fee ฯลฯ) |
| `_APARItems` | 0..n | บรรทัด customer / supplier |
| `_ProductTaxItems` | 0..n | บรรทัดภาษี (ถ้าเอกสารตัวอย่างมี) |

### `_GLItems`

| Field | หมายเหตุ |
|---|---|
| `GLAccountLineItem` | ลำดับบรรทัด `1`, `2`, … |
| `GLAccount` | บัญชี G/L (10 หลัก มี leading zero) |
| `DocumentItemText` | item text (SGTXT) |
| `AssignmentReference` | assignment (ZUONR) |
| `CostCenter` · `ProfitCenter` | ตามที่เอกสารตัวอย่างมี |
| `HouseBank` · `HouseBankAccount` | บรรทัด bank |
| `ValueDate` | value date ของบรรทัด bank |
| `TaxCode` | ถ้า G/L เป็น tax-relevant ต้องใส่ ไม่งั้น error |
| `_CurrencyAmount` | 1..n — ดูด้านล่าง |

### `_APARItems`

| Field | หมายเหตุ |
|---|---|
| `GLAccountLineItem` | ลำดับบรรทัด (นับต่อจาก `_GLItems`) |
| `Customer` / `Supplier` | ใส่อย่างใดอย่างหนึ่ง |
| `SpecialGLCode` | special G/L indicator — เว้นว่างสำหรับ payment ปกติ |
| `DocumentItemText` · `AssignmentReference` | |
| `ProfitCenter` | ถ้า splitting ไม่ derive ให้ |
| `PaymentTerms` · `DueCalculationBaseDate` | ไม่จำเป็นสำหรับ DZ |
| `_CurrencyAmount` | 1..n |

### `_CurrencyAmount` (อยู่ใต้ทุก item)

| Field | ค่า |
|---|---|
| `CurrencyRole` | `'00'` = transaction currency (ใส่แค่ตัวนี้พอ ระบบแปลงเอง) |
| `Currency` | `THB` |
| `JournalEntryItemAmount` | **เดบิต = บวก · เครดิต = ลบ** — ไม่มี DebitCreditCode แยก |

ผลรวมของทุกบรรทัดต้องเป็นศูนย์

## Incoming payment หน้าตาที่คาด

```
Header  : DZ · RFBU · company code / dates / reference จากเอกสารตัวอย่าง
_GLItems   [1]  Dr  bank clearing / cash G/L     +amount   (house bank, value date)
_APARItems [2]  Cr  customer                     −amount
```

posting key ที่ระบบสร้างให้: G/L เดบิต `40` · customer เครดิต `15` (payment) —
ต้องเทียบกับเอกสารตัวอย่างว่าออกมาเหมือนกันไหม

## ผลลัพธ์ที่ได้กลับมา

- `ls_mapped-journalentry[ 1 ]-%pid` → `CONVERT KEY OF i_journalentrytp FROM … TO …`
  ได้ `CompanyCode` + `AccountingDocument` + `FiscalYear`
- หรืออ่านจาก `COMMIT ENTITIES RESPONSE OF` แล้ว query
  `I_JournalEntry` / `I_OperationalAcctgDocItem` ด้วย `DocumentReferenceID` ที่ใส่ไปก็ได้

## สิ่งที่ API นี้ทำไม่ได้ (ต้องรู้ก่อนเอาไปทำต่อ)

- **ไม่ clear open item** — เอกสารที่ได้ค้างเป็น open item บน customer
  ต้องต่อด้วย Clearing API (`JournalEntryBulkClearingRequest_In`) เป็นอีกขั้น
- ไม่มี test-run flag → simulate ต้องทำในโค้ดเอง
- ไม่สร้าง one-time customer (SAP Note 3637431)
- บน on-premise / private edition SAP แนะนำให้ใช้ BAPI แทน — BO นี้เหมาะกับ public edition
