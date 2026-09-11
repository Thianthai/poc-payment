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

## โครงสร้าง `%param` — ยืนยันจาก tenant แล้ว (2026-09-11)

จาก behavior definition `I_JournalEntryTP`:

```
static factory save ( finalize, adjustnumbers ) action ( authorization : none ) Post
  deep parameter D_JournalEntryPostParameter [1];
```

= **save action** (เรียกใน console class ด้วย `MODIFY ENTITIES` + `COMMIT ENTITIES` ได้ ·
ใน RAP BO ของตัวเองเรียกได้เฉพาะ `finalize` / `adjust_numbers`)

### Header — `D_JournalEntryPostParameter` (root abstract entity)

| Field | DDIC | POC ใช้ |
|---|---|---|
| `CompanyCode` | `bukrs` | ✅ `1000` |
| `BusinessTransactionType` | `glvor` | ✅ `RFPI` (fallback `RFBU`) |
| `AccountingDocumentType` | `blart` | ✅ `DZ` |
| `DocumentDate` / `PostingDate` | `bldat` / `budat` | ✅ วันนี้ |
| `CreatedByUser` | `usnam` | ✅ `sy-uname` |
| `DocumentReferenceID` | `xblnr` | ✅ generate ต่อรอบ |
| `AccountingDocumentHeaderText` | `bktxt` | ว่าง (ตามตัวอย่าง) |
| `AccountingDocument` | `belnr_d` | external number — ไม่ใช้ |
| `LedgerGroup` | `fagl_ldgrp` | ไม่ใช้ |
| `InvoiceReferenceDocument` | `awkey_reb` | ไม่ใช้ (ไม่ clear) |
| `TaxReportingDate` / `TaxDeterminationDate` / `TaxFulfillmentDate` | | ไม่ใช้ |
| `InvoiceReceiptDate` · `ExchangeRateDate` · `IsNegativePosting` · `PostingFiscalPeriod` | | ไม่ใช้ |
| `Reference1InDocumentHeader` / `Reference2InDocumentHeader` | | ไม่ใช้ |
| `JrnlEntryCntrySpecificRef1..5` / `Date1..5` / `BP1..2` | | ไม่ใช้ |
| `ReversalReferenceDocumentKey` · `ReversalReason` · `PlannedReversalDate` | | ไม่ใช้ |
| `EntryViewPostingControl` | `fins_entry_view_postng_control` | ไม่ใช้ |

Composition (ชื่อจริง — **ไม่ใช่ `_APARItems` / `_ProductTaxItems` อย่างที่เดาไว้**):

| Node | Abstract entity | POC ใช้ |
|---|---|---|
| `_GLItems` [0..*] | `D_JournalEntryPostGLItemP` | ✅ 4 บรรทัด (bank, WHT, deferred tax, output tax) |
| `_ARItems` [0..*] | `D_JournalEntryPostARItemP` | ✅ 1 บรรทัด customer |
| `_APItems` [0..*] | `D_JournalEntryPostAPItemP` | ไม่ใช้ |
| `_TaxItems` [0..*] | `D_JournalEntryPostTaxItemP` | fallback ถ้า `_GLItems` + tax code ไม่ผ่าน |
| `_WithHoldingTaxItems` [0..*] | `D_JournalEntryPostWhgdItemP` | ไม่ใช้ (WHT line ของตัวอย่างเป็น G/L ธรรมดา) |
| `_OneTimeCustomerSupplier` [0..1] | `D_JournalEntryPostCPDP` | ไม่ใช้ |

> field ของ item node ด้านล่างยังเป็นค่าที่เดาไว้ — รอ paste `D_JournalEntryPostGLItemP` /
> `D_JournalEntryPostARItemP` จาก ADT

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

### `_ARItems`

| Field | หมายเหตุ |
|---|---|
| `GLAccountLineItem` | ลำดับบรรทัด (นับต่อจาก `_GLItems`) |
| `Customer` | เลข customer |
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
_ARItems   [2]  Cr  customer                     −amount
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
