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
| `TaxDeterminationDate` | `txdat` | ✅ = posting date — **บังคับ** เพราะ company code เปิด time-dependent tax (เจอจริง 2026-09-11: `tax date has to be filled from caller`) |
| `TaxReportingDate` / `TaxFulfillmentDate` | | ไม่ใช้ |
| `InvoiceReceiptDate` · `ExchangeRateDate` · `IsNegativePosting` · `PostingFiscalPeriod` | | ไม่ใช้ |
| `Reference1InDocumentHeader` / `Reference2InDocumentHeader` | | ไม่ใช้ |
| `JrnlEntryCntrySpecificRef1..5` / `Date1..5` / `BP1..2` | | ไม่ใช้ |
| `ReversalReferenceDocumentKey` · `ReversalReason` · `PlannedReversalDate` | | ไม่ใช้ |
| `EntryViewPostingControl` | `fins_entry_view_postng_control` | ไม่ใช้ |

Composition (ชื่อจริง — **ไม่ใช่ `_APARItems` / `_ProductTaxItems` อย่างที่เดาไว้**):

| Node | Abstract entity | POC ใช้ |
|---|---|---|
| `_GLItems` [0..*] | `D_JournalEntryPostGLItemP` | ✅ ใบ 1: bank, WHT · ใบ 2: คู่ dummy `0011054001` |
| `_ARItems` [0..*] | `D_JournalEntryPostARItemP` | ✅ 1 บรรทัด customer |
| `_APItems` [0..*] | `D_JournalEntryPostAPItemP` | ไม่ใช้ |
| `_TaxItems` [0..*] | `D_JournalEntryPostTaxItemP` | ✅ ใบ 2: deferred → output tax (DM/O1) direct — post ผ่านเมื่อมีคู่ dummy `_GLItems` ในใบเดียวกัน |
| `_WithHoldingTaxItems` [0..*] | `D_JournalEntryPostWhgdItemP` | ✅ ใบ 1: 1 entry ต่อ WHT type ใน customer master (`MA`/`09`) amount 0 — ไม่มีจะ clear ไม่ได้ (rev 18) |
| `_OneTimeCustomerSupplier` [0..1] | `D_JournalEntryPostCPDP` | ไม่ใช้ |

### `_GLItems` — `D_JournalEntryPostGLItemP` (ยืนยันจาก tenant)

| Field | DDIC | POC ใช้ |
|---|---|---|
| `GLAccountLineItem` | `docln6` | ✅ `1`–`4` |
| `GLAccount` | `hkont` | ✅ |
| `DocumentItemText` | `sgtxt` | ✅ บรรทัด bank |
| `AssignmentReference` | `acpi_zuonr` | ✅ bank / WHT = posting date · output tax = key ของ tax line บน invoice |
| `TaxCode` | `mwskz` | ✅ บรรทัด deferred / output tax |
| `ValueDate` | `valut` | ✅ = posting date (ตามตัวอย่างทุกบรรทัด G/L) |
| `HouseBank` / `HouseBankAccount` | `hbkid` / `hktid` | ✅ บรรทัด bank |
| `CompanyCode` | `bukrs` | ไม่ใส่ (ใช้ของ header) |
| `ItemGroup` · `Reference1..3IDByBusinessPartner` · `OplAcctgDocItmCntrySpcfcRef1` | | ไม่ใช้ |
| `FinancialTransactionType` · `TaxJurisdiction` · `TaxItemAcctgDocItemRef` · `TaxCountry` | | ไม่ใช้ |
| `BusinessPlace` | `acpi_branch` | ✅ `0000` — Thai localization บังคับ (`Enter a business place.`) |
| `Plant` · `Material` · `BaseUnit` · `Quantity` · `IsNotCashDiscountLiable` · `PartnerCompany` | | ไม่ใช้ |
| `ProfitCenter` · `PartnerProfitCenter` · `Segment` · `PartnerSegment` · `CostCenter` · `CostCtrActivityType` | | ไม่ใส่ — ตัวอย่างว่าง ระบบ derive `DUMMY` เอง |
| `WBSElement` · `MasterFixedAsset` · `FixedAsset` · `SalesOrder(Item)` · `FunctionalArea` · `ServiceDocument*` · `PersonnelNumber` · `WorkItem` · `OrderID` · `JointVenture*` · `FinancialServices*` · `FinancialDataSource` | | ไม่ใช้ |
| `_CurrencyAmount` | association [0..*] → `D_JournalEntryPostCurrencyAmtP` | ✅ |
| `_ProfitabilitySupplement` | composition [0..1] → `D_JournalEntryPostCOPAP` | ไม่ใช้ |

### `_TaxItems` — `D_JournalEntryPostTaxItemP` (ยืนยันจาก tenant)

| Field | DDIC | POC ใช้ |
|---|---|---|
| `GLAccountLineItem` | `docln6` | ✅ |
| `TaxCode` | `mwskz` | ✅ `DM` (จาก invoice) / `O1` (target ของ DM) |
| `TaxItemClassification` | `ktosl` | ✅ `MWS` — account key → derive G/L (`DM` → `0021082005` · `O1` → `0021082003`) · doc: mandatory ถ้าไม่ให้ `ConditionType` |
| `ConditionType` | `kschl` | ✅ **`MWAS`** (output tax) — ไม่ใส่ได้ `KSCHL is empty` |
| `IsDirectTaxPosting` | `bapi_flg_dir` | ✅ `X` — post ภาษีตรงโดยไม่มี base line |
| `TaxDeterminationDate` | `acpi_txdat` | ❌ doc ProductTaxItem: **"Do not use"** — ใส่ที่ header พอ |
| `TaxRate` | `msatz_f05l` | ไม่ใส่ (derive จาก tax code + date) |
| `TaxItemAcctgDocItemRef` | `taxps` | ❌ **ห้ามใส่บน TH** — FF 817 `Taxes by item is not activated` (line-by-line tax เฉพาะ US/CA/BR) (rev 12) |
| `TaxJurisdiction` · `TaxJurisdictionLevel` · `LowestLevelTaxJurisdiction` · `TaxCountry` | | ไม่ใช้ (US/CA/BR · RITA) |
| `_CurrencyAmount` | association [0..*] → `D_JournalEntryPostCurrencyAmtP` | ✅ `JournalEntryItemAmount` + `TaxBaseAmount` (เครื่องหมายเดียวกัน) |

> **ไม่มี `GLAccount` / `AssignmentReference` / `BusinessPlace`** — G/L มาจาก config ·
> assignment `94000000052026003` ของตัวอย่างจึงใส่ไม่ได้ ·
> business place ของ tax item **derive จากบรรทัด G/L/AR ในเอกสารเดียวกัน** → เอกสารที่มีแต่ tax item
> ได้ `Enter a business place.` และไม่มีที่ให้ใส่ → **ทางแก้ที่ผ่าน**: ใส่คู่ dummy `_GLItems` net 0
> (`0011054001` ±ยอด · มี business place · บัญชีไม่ใช่ OIM) ไว้ในใบเดียวกัน → `7200000001` (2026-09-14)
>
> ใส่ tax item ในใบ payment ไม่ได้ → ชน `G/L account item without tax code in document with deferred
> taxes` ที่ bank line (tax category ว่าง) — check นี้ไม่ฟ้องในใบ SA ที่ไม่มี AR line

### `_WithHoldingTaxItems` — `D_JournalEntryPostWhgdItemP` (ยืนยันจาก tenant)

| Field | DDIC | POC ใช้ |
|---|---|---|
| `GLAccountLineItem` | `docln6` | ✅ = เลข item ของบรรทัดลูกหนี้ที่ WHT ผูกด้วย (`3`) |
| `WithholdingTaxType` | `witht` | ✅ `MA` (A/R at Payment & No cert. numbering 1) — จาก customer master |
| `WithholdingTaxCode` | `wt_withcd` | ✅ `09` (Service 3%) — จาก customer master |
| `WhldgTaxIsEnteredManually` | `wt_amnman` | ✅ `X` — ให้ค่าเอง ไม่ให้ระบบคำนวณ |
| `WhldgTaxBaseIsEnteredManually` | `wt_basman` | ✅ `X` |
| `_CurrencyAmount` | association [0..*] → `D_JournalEntryPostCurrencyAmtP` | ✅ `JournalEntryItemAmount 0` · `TaxBaseAmount 0` (ตามตัวอย่าง `3300000017/005`: base/amount 0) |

> ทำไมต้องส่ง: clearing (POC clearing) เช็คว่า WHT type บน open item ต้องตรงกับ customer master
> ทุก type — Fiori เติมให้เองจาก master แต่ API ไม่เติม (`3300000026` clear ไม่ได้)
> · WHT จริง 179.97 ยังส่งเป็น G/L `0011047003` ตามตัวอย่าง (ทางเลือกที่ยังไม่ลอง: ให้ระบบคำนวณเองจาก base)
> · `XX` ที่เห็นใน `I_OperationalAcctgDocItem.WithholdingTaxCode` เป็น classic WHT field ไม่ใช่ตัวนี้

### `_ARItems` — `D_JournalEntryPostARItemP` (ยืนยันจาก tenant)

| Field | DDIC | POC ใช้ |
|---|---|---|
| `GLAccountLineItem` | `docln6` | ✅ `5` |
| `Customer` | `kunnr` | ✅ จาก AR line ของ invoice |
| `GLAccount` | `hkont` | ไม่ใส่ — ระบบ derive reconciliation account เอง |
| `DocumentItemText` · `AssignmentReference` | | ไม่ใส่ (ตัวอย่างว่าง) |
| `SpecialGLCode` | `acpi_umskz` | ไม่ใส่ |
| `PaymentTerms` · `DueCalculationBaseDate` · `CashDiscount*` · `NetPaymentDays` | | ไม่ใส่ |
| `PaymentMethod` · `PaymentMethodSupplement` · `SEPAMandate` · `PaymentReference` · `PaymentBlockingReason` · `PaymentServiceProvider` · `PaymentRefByPaytSrvcProvider` | | ไม่ใช้ |
| `HouseBank` / `HouseBankAccount` | | ไม่ใส่ (อยู่บรรทัด G/L bank แทน) |
| `TaxCode` · `TaxJurisdiction` · `TaxCountry` · `VATRegistration` · `ReportingCountry` · `IsEUTriangularDeal` | | ไม่ใช้ |
| `BusinessPlace` | `acpi_branch` | ✅ `0000` |
| `Reference1..3IDByBusinessPartner` · `OplAcctgDocItmCntrySpcfcRef1` · `BranchAccount` · `BusinessSectionCode` | | ไม่ใช้ |
| `SalesOrder(Item)` · `JointVenture*` · `CreditControlArea` · `PaymentReason` · `DigitalPaymentType` · `PaymentByDigitalPaymentService` · `DunningKey` · `DunningBlock` · `StateCentralBankPaymentReason` | | ไม่ใช้ |
| `_CurrencyAmount` | association [0..*] → `D_JournalEntryPostCurrencyAmtP` | ✅ |

> **ไม่มี field WHT** ใน `_ARItems` — `WithholdingTaxCode XX` ที่เห็นบนตัวอย่างระบบ
> derive จาก customer master เอง ตรงกับที่วิเคราะห์ไว้ใน docs/04

### `_CurrencyAmount` — `D_JournalEntryPostCurrencyAmtP` (ยืนยันจาก tenant)

| Field | DDIC | POC ใช้ |
|---|---|---|
| `CurrencyRole` | `curtp` | ✅ `'00'` = transaction currency |
| `Currency` | `waers` | ✅ `THB` (จาก invoice) |
| `JournalEntryItemAmount` | `wrbtr` | ✅ **เดบิต = บวก · เครดิต = ลบ** |
| `TaxBaseAmount` | `fwbas` | ✅ บรรทัดภาษี: `+5,999` (deferred) / `−5,999` (output) ตามตัวอย่าง |
| `TaxAmount` | `wmwst` | ไม่ใส่ |
| `CashDiscountBaseAmount` | `wskto` | ไม่ใส่ |
| `ExchangeRate` · `IndirectQuotedExchangeRate` | | ไม่ใส่ (THB → THB) |

ผลรวม `JournalEntryItemAmount` ของทุกบรรทัดต้องเป็นศูนย์

## Payload ที่ `YCL_PAYMENT` ส่ง (rev 17 final · derive จาก invoice `9400000005`)

```
ใบ 1  DZ · RFPI · TaxDeterminationDate · ref POC…            → 3300000026
_GLItems  [1]  0011092001  +6,238.96  bank    text/BBL01/CA001/assignment/value date/bplace 0000
_GLItems  [2]  0011047003    +179.97  WHT     assignment/value date/bplace 0000
_ARItems  [3]  0001000082  −6,418.93  customer · bplace 0000  (ได้ PK 11 ไม่ใช่ 15)

ใบ 2  SA · RFBU · TaxDeterminationDate · ref POC…T           → 7200000001
_GLItems  [1]  0011054001    +419.93  dummy · bplace 0000   ┐ net 0 · ให้ tax item derive business place
_GLItems  [2]  0011054001    −419.93  dummy · bplace 0000   ┘
_TaxItems [3]  DM  +419.93  MWS · MWAS · direct · TaxBaseAmount +5,999  (G/L derive → 0021082005)
_TaxItems [4]  O1  −419.93  MWS · MWAS · direct · TaxBaseAmount −5,999  (G/L derive → 0021082003)
```

posting key ที่ระบบสร้าง: G/L เดบิต `40` / เครดิต `50` · customer เครดิต **`11`** (credit memo) —
ตัวอย่างจาก Post Incoming Payments เป็น `15`

## ผลลัพธ์ที่ได้กลับมา

- `MAPPED` ให้แค่ `%cid` + `%pid` (late numbering) — **`CONVERT KEY` ใช้ใน console ไม่ได้**
  (`BEHAVIOR_STATEMENT_ILLEGAL` · เจอจริง 2026-09-11)
- วิธีที่ใช้ได้: query `I_JournalEntry` ด้วย `DocumentReferenceID` ที่ generate ไว้หลัง `COMMIT ENTITIES`

## สิ่งที่ API นี้ทำไม่ได้ (ต้องรู้ก่อนเอาไปทำต่อ)

- **ไม่ clear open item** — เอกสารที่ได้ค้างเป็น open item บน customer
  ต้องต่อด้วย Clearing API (`JournalEntryBulkClearingRequest_In`) เป็นอีกขั้น
- ไม่มี test-run flag → simulate ต้องทำในโค้ดเอง
- ไม่สร้าง one-time customer (SAP Note 3637431)
- บน on-premise / private edition SAP แนะนำให้ใช้ BAPI แทน — BO นี้เหมาะกับ public edition

## API ทางเลือกบน Public Edition (สำรวจ 2026-09-11 หลังพบว่า PK 15 ทำไม่ได้)

ไม่มี API "post incoming payment" ตรง ๆ — community ลงเอยที่ `I_JournalEntryTP` เหมือนกัน
([Q&A 2025](https://community.sap.com/t5/financial-management-q-a/sap-s-4-public-cloud-post-incoming-payment-integration/qaq-p/14060266)) ·
SAP เองจัด `I_JournalEntryTP` / Journal Entry – Post ไว้สำหรับ "ad-hoc payment on account"
([blog 2025-11](https://community.sap.com/t5/financial-management-blog-posts-by-sap/sap-s-4hana-cloud-for-customer-payments-transition-and-future-with-apis/ba-p/14242704))

| API | bank line | customer PK | clear | โอน deferred tax | หมายเหตุ |
|---|---|---|---|---|---|
| `I_JournalEntryTP` | ✅ | **11** | ❌ | ❌ | POC นี้ |
| Journal Entry – Post SOAP (`JournalEntryBulkCreationRequest_In` async / `JournalEntryCreateRequestConfirmation_In` sync) | ✅ | **11** | ❌ | ❌ | **เครื่องยนต์เดียวกับ `I_JournalEntryTP`** — doc ระบุ DebtorItem credit = 11 · ต่างแค่เรียกผ่าน SOAP + `SAP_COM_0002` |
| Journal Entry – Clearing (SOAP) | ❌ | — | ✅ | ตอน clear | POC clearing |
| **Bank Statement** (`SAP_COM_0316`, scope 1EG) | ✅ | **15** | ✅ | ✅ | ทางมาตรฐานสำหรับเงินโอนเข้า bank · ต้อง config EBS posting rule |
| Manage Automatic Payments (F110) | ✅ | 15 | ✅ | | direct debit เท่านั้น |
| EBPP Payment Request (2508.02+) | | | | | portal + Digital Payments add-on |

