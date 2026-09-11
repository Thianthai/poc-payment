# 04 — SQL สำหรับ export เอกสารตัวอย่าง

Released CDS view ที่ใช้ (เช็คแล้วบน tenant ตอน POC clearing 2026-09-09):

| View | ใช้ทำอะไร |
|---|---|
| `I_JournalEntry` | header ของเอกสาร |
| `I_OperationalAcctgDocItem` | **entry view** (BSEG) — บรรทัดที่ต้องส่งเข้า API |
| `I_JournalEntryItem` | **G/L view** (ACDOCA) — รวมบรรทัดที่ splitting สร้าง ใช้เทียบผลหลัง post |

Field name ของ `I_OperationalAcctgDocItem` ยืนยันแล้วจาก POC clearing ·
ของ `I_JournalEntry` ยังไม่เคย probe → รัน Q0 ก่อนถ้า Q1 compile ไม่ผ่าน

## วิธีรัน

ทุก query เป็น **ABAP SQL** — วางใน console class (`IF_OO_ADT_CLASSRUN`) แล้ว
`out->write( lt_xxx ).` หรือวางใน ADT SQL Console โดยตัด `INTO TABLE @DATA(...)` ออก

แทนค่า 3 ตัวนี้ก่อนรัน:

```
lc_company_code = '1000'          ← company code
lc_document     = '3300000017'    ← เลขเอกสาร payment ตัวอย่าง
lc_fiscal_year  = '2026'
```

---

## Q0 — probe ชื่อ field ของ `I_JournalEntry` (รันครั้งเดียวถ้า Q1 ไม่ผ่าน)

```abap
SELECT *
  FROM I_JournalEntry
  WHERE CompanyCode        = '1000'
    AND AccountingDocument = '3300000017'
    AND FiscalYear         = '2026'
  INTO TABLE @DATA(lt_probe).
```

## Q1 — header

```abap
SELECT CompanyCode,
       AccountingDocument,
       FiscalYear,
       AccountingDocumentType,
       DocumentDate,
       PostingDate,
       TransactionCurrency,
       DocumentReferenceID,
       AccountingDocumentHeaderText,
       ReferenceDocumentType,
       OriginalReferenceDocument,
       AccountingDocCreatedByUser,
       AccountingDocumentCreationDate
  FROM I_JournalEntry
  WHERE CompanyCode        = '1000'
    AND AccountingDocument = '3300000017'
    AND FiscalYear         = '2026'
  INTO TABLE @DATA(lt_header).
```

> `ReferenceDocumentType` (AWTYP) บอกว่าเอกสารตัวอย่างมาจากไหน — ถ้าเป็น `BKPFF`
> คือ post ตรง ถ้าเป็นอย่างอื่น (เช่น จาก bank statement) ให้บอกด้วย

## Q2 — entry view: บรรทัดที่ต้องส่งเข้า API (ตัวหลัก)

```abap
SELECT AccountingDocumentItem,
       PostingKey,
       FinancialAccountType,
       DebitCreditCode,
       GLAccount,
       Customer,
       Supplier,
       SpecialGLCode,
       AmountInTransactionCurrency,
       TransactionCurrency,
       AmountInCompanyCodeCurrency,
       ProfitCenter,
       CostCenter,
       DocumentItemText,
       AssignmentReference,
       HouseBank,
       HouseBankAccount,
       ValueDate,
       TaxCode,
       ClearingAccountingDocument,
       ClearingDate,
       IsOpenItemManaged
  FROM I_OperationalAcctgDocItem
  WHERE CompanyCode        = '1000'
    AND AccountingDocument = '3300000017'
    AND FiscalYear         = '2026'
  ORDER BY AccountingDocumentItem
  INTO TABLE @DATA(lt_items).
```

> ถ้า field ไหน compile ไม่ผ่าน (`HouseBank` / `HouseBankAccount` / `ValueDate` / `TaxCode`
> ยังไม่เคยใช้ใน POC ก่อน) ให้ตัดออกแล้วบอกว่าตัวไหนไม่มี

## Q3 — G/L view: ทุกบรรทัดใน ACDOCA รวมที่ splitting สร้าง (ไว้เทียบผลหลัง post)

```abap
SELECT AccountingDocumentItem,
       LedgerGLLineItem,
       Ledger,
       GLAccount,
       Customer,
       DebitCreditCode,
       AmountInTransactionCurrency,
       TransactionCurrency,
       ProfitCenter,
       CostCenter,
       Segment,
       BusinessTransactionType,
       DocumentItemText,
       AssignmentReference
  FROM I_JournalEntryItem
  WHERE CompanyCode        = '1000'
    AND AccountingDocument = '3300000017'
    AND FiscalYear         = '2026'
    AND Ledger             = '0L'
  ORDER BY LedgerGLLineItem
  INTO TABLE @DATA(lt_acdoca).
```

> `BusinessTransactionType` ควรออกมาเป็น `RFBU` — ถ้าเป็นค่าอื่นให้บอก เพราะต้องใส่ค่าเดียวกันใน `%param`

---

## Invoice ต้นทาง — `9400000005` (requirement เพิ่ม 2026-09-11)

payment ต้อง derive จาก invoice → รัน **Q1 + Q2 + Q3 ข้างบนอีกรอบ** โดยแทน
`'3300000017'` ด้วย `'9400000005'` แล้วเพิ่ม Q4 / Q5 เพื่อดู field ภาษี / WHT ที่ยังไม่รู้ชื่อ

## Q4 — invoice · entry view ทุก field (หา WHT / tax base)

```abap
SELECT *
  FROM I_OperationalAcctgDocItem
  WHERE CompanyCode        = '1000'
    AND AccountingDocument = '9400000005'
    AND FiscalYear         = '2026'
  ORDER BY AccountingDocumentItem
  INTO TABLE @DATA(lt_inv_items_all).
```

## Q5 — invoice · G/L view ทุก field

```abap
SELECT *
  FROM I_JournalEntryItem
  WHERE CompanyCode        = '1000'
    AND AccountingDocument = '9400000005'
    AND FiscalYear         = '2026'
    AND Ledger             = '0L'
  ORDER BY LedgerGLLineItem
  INTO TABLE @DATA(lt_inv_acdoca_all).
```

> Q4/Q5 ได้ column เยอะ — paste มาทั้งหมดได้เลย Claude จะคัด field ที่เกี่ยวเอง

## Q7 — G/L master ของบัญชีที่ใช้ (tax category / posting without tax) — สำหรับหาทางใส่ DM/O1

```abap
SELECT *
  FROM I_GLAccountInCompanyCode
  WHERE CompanyCode = '1000'
    AND GLAccount IN ( '0011092001', '0011047003', '0021082005', '0021082003' )
  INTO TABLE @DATA(lt_gl_master).
```

## Q8 — tax code ที่เคยใช้จริงบน company code (หา 0% code สำหรับบรรทัด bank / WHT)

```abap
SELECT TaxCode,
       COUNT(*)                            AS ItemCount,
       SUM( AmountInTransactionCurrency )  AS TotalAmount
  FROM I_OperationalAcctgDocItem
  WHERE CompanyCode = '1000'
    AND TaxCode    <> ''
  GROUP BY TaxCode
  INTO TABLE @DATA(lt_tax_codes).
```

---

## ผลที่ได้ (export 2026-09-11 · ทุก query compile ผ่านทุก field)

เอกสารตัวอย่าง = **`3300000017` / 2026 / company code `1000`** — ตัวเดียวกับ payment
ที่ใช้ใน POC clearing

> `3300000017` ถูก **reverse แล้ว 2026-09-11** (reversal `3300000019`) ก่อน post จริงจาก
> `YCL_PAYMENT` เพื่อไม่ให้ลูกหนี้มี open item ซ้ำ — ข้อมูลด้านล่างยังใช้เป็น reference ได้

### Q1 — header

| Field | ค่า |
|---|---|
| `AccountingDocumentType` | `DZ` |
| `DocumentDate` / `PostingDate` | `2026-09-09` |
| `TransactionCurrency` | `THB` |
| `DocumentReferenceID` | `09080007` |
| `AccountingDocumentHeaderText` | (ว่าง) |
| `ReferenceDocumentType` | `BKPFF` — post ตรง ไม่ได้มาจาก bank statement |
| `OriginalReferenceDocument` | `330000001710002026` |
| `AccountingDocCreatedByUser` | `CB9980000240` |

### Q2 — entry view (BSEG) · 5 บรรทัด · ผลรวม = 0

| Item | PK | Type | Account | Amount THB | Assignment | Text | House bank | Value date | Tax |
|---|---|---|---|---:|---|---|---|---|---|
| 001 | 40 | S | G/L `0011092001` | +6,238.96 | `20260909` | รับผ่านช่องทาง mobile banking | `BBL01` / `CA001` | 2026-09-09 | |
| 002 | 40 | S | G/L `0011047003` | +179.97 | `20260909` | | | 2026-09-09 | |
| 003 | 40 | S | G/L `0021082005` | +419.93 | | | | 2026-09-09 | `DM` |
| 004 | 50 | S | G/L `0021082003` | −419.93 | `94000000052026003` | | | 2026-09-09 | `O1` |
| 005 | 15 | D | Customer `0001000082` (recon `0011030001`) | −6,418.93 | | | | | |

- `ProfitCenter` / `CostCenter` / `SpecialGLCode` ว่างทุกบรรทัดใน entry view
- `IsOpenItemManaged = X` ทุกบรรทัด · `ClearingAccountingDocument` ว่าง = ยังไม่ถูก clear
  (ถูก clear ทีหลังโดย POC clearing)

อ่านความหมายของแต่ละบรรทัด (ยอด invoice สุทธิ 5,999.00):

| Item | คืออะไร | ที่มาของยอด |
|---|---|---|
| 001 | เงินเข้าบัญชีธนาคาร (bank clearing) | 6,418.93 − WHT 179.97 |
| 002 | ภาษีหัก ณ ที่จ่ายที่ลูกค้าหักไว้ (WHT receivable) | 3% × 5,999.00 |
| 003 | โอนออกจาก deferred output tax | 7% × 5,999.00 |
| 004 | เข้า output tax (assignment = invoice `9400000005`/2026/003) | 7% × 5,999.00 |
| 005 | ตัดลูกหนี้ | invoice เต็ม 6,418.93 |

### Q3 — G/L view (ACDOCA, ledger 0L) · 5 บรรทัด

ไม่มีบรรทัด `000` ที่ splitting สร้าง · ทุกบรรทัด `ProfitCenter = DUMMY` ·
`Segment = JASGROUP` (derive จาก PC) — entry view ไม่ได้ส่ง PC มา ระบบ derive เอง

**`BusinessTransactionType = RFPI`** ทุกบรรทัด — ไม่ใช่ `RFBU` ที่ SOAP doc บอกว่าเป็นค่าเดียวที่รับ
→ ต้องลองว่า BO interface รับ `RFPI` ไหม (ดู [03-test-data.md](03-test-data.md))

---

## ผลที่ได้ — invoice `9400000005` (export 2026-09-11 · Q1–Q5)

### Q1 — header

| Field | ค่า |
|---|---|
| `AccountingDocumentType` | `RV` (billing → FI) |
| `DocumentDate` / `PostingDate` | `2026-09-03` |
| `TransactionCurrency` | `THB` |
| `DocumentReferenceID` | `JA70000046` (= billing document) |
| `ReferenceDocumentType` / `OriginalReferenceDocument` | `VBRK` / `JA70000046` |

### Q2 + Q4 — entry view · 3 บรรทัด

| Item | PK | Type | Account | THB | Tax | field สำคัญจาก Q4 |
|---|---|---|---|---:|---|---|
| 001 | 01 | D | Customer `0001000082` (recon `0011030001`) | +6,418.93 | `DM` | **`WithholdingTaxCode = XX`** · WHT base/amount = 0 (ยังไม่คำนวณ ณ invoice) · payment terms `ZP00` · `NetDueDate 2026-09-03` |
| 002 | 50 | S | G/L `0021060006` revenue | −5,999.00 | `DM` | `TaxItemGroup 001` |
| 003 | 50 | S | G/L `0021082005` deferred output tax | −419.93 | `DM` | **`AccountingDocumentItemType = T`** · `TaxType A` · `TransactionTypeDetermination MWS` · `TaxBaseAmountInTransCrcy −5,999.00` · `TaxItemGroup 001` · open item managed |

`ClearingAccountingDocument` ว่างทุกบรรทัด (item 001/003 ถูก reset clearing วันนี้ —
`LastChangeDateTime 2026-09-11 03:39 UTC`) → invoice **open พร้อมใช้เป็น input**

### Q3 / Q5 — G/L view

3 บรรทัด ไม่มี splitting line · `ProfitCenter 0000010002` · `Segment JASGROUP` ·
`BusinessTransactionType = SD00` (มาจาก SD ไม่เกี่ยวกับ payment)

### mapping invoice → payment ที่อ่านได้

| Payment `3300000017` | derive จาก invoice | ค่าที่ต้อง fix (ไม่มีบน invoice) |
|---|---|---|
| 005 customer −6,418.93 | item 001: `Customer` + `AmountInTransactionCurrency` กลับเครื่องหมาย | — |
| 003 deferred tax `DM` +419.93 | item 003 (`ItemType = T`): `GLAccount` + `TaxCode` + amount กลับเครื่องหมาย | — |
| 004 output tax `O1` −419.93 · assignment `94000000052026003` | amount = item 003 · assignment = `AccountingDocument + FiscalYear + AccountingDocumentItem` ของ item 003 | G/L `0021082003` + tax code `O1` (target ของ `DM` จาก deferred-tax config) |
| 002 WHT +179.97 | base = `TaxBaseAmountInTransCrcy` ของ item 003 (5,999.00) · trigger = `WithholdingTaxCode XX` บน item 001 | อัตรา 3% + G/L `0011047003` (จาก WHT config ของ code `XX`) |
| 001 bank +6,238.96 | = customer − WHT | house bank `BBL01` / `CA001` + G/L `0011092001` |

### ยังไม่รู้ — ต้อง export payment `3300000017` แบบ Q4 (`SELECT *`)

| คำถาม | ดูจาก field |
|---|---|
| บรรทัด 002 (WHT) ถูกระบบสร้างเองจาก WHT code หรือ user ใส่ G/L ตรง ๆ | `IsAutomaticallyCreated` · `AccountingDocumentItemType` ของ item 002 · `WithholdingTaxCode` / `WithholdingTaxAmount` / `WithholdingTaxBaseAmount` ของ item 005 |
| บรรทัด 003/004 (tax) เป็น tax item (`T`) หรือ G/L ธรรมดา | `AccountingDocumentItemType` · `TaxItemGroup` · `TransactionTypeDetermination` · `TaxBaseAmountInTransCrcy` ของ item 003/004 |

คำตอบนี้ตัดสินว่า class ต้องส่งเป็น `_GLItems` ตรง ๆ หรือ `_TaxItems` / WHT node
แล้วให้ระบบสร้างบรรทัดเอง

## Q6 — payment · entry view ทุก field

```abap
SELECT *
  FROM I_OperationalAcctgDocItem
  WHERE CompanyCode        = '1000'
    AND AccountingDocument = '3300000017'
    AND FiscalYear         = '2026'
  ORDER BY AccountingDocumentItem
  INTO TABLE @DATA(lt_pay_items_all).
```

### ผล Q6 (2026-09-11) — **ทุกบรรทัดของ payment ถูกใส่เอง ไม่มีบรรทัดที่ระบบสร้าง**

| Item | `ItemType` | `IsAutomaticallyCreated` | `TaxCode` | `TaxType` / `TTD` | `TaxItemGroup` | `TaxBase` | `WHT code` | อื่น ๆ |
|---|---|---|---|---|---|---:|---|---|
| 001 bank | (ว่าง) | (ว่าง) | | | 000 | 0 | | `PlanningLevel B0` · `HouseBank BBL01` · `HouseBankAccount CA001` |
| 002 WHT | (ว่าง) | (ว่าง) | | | 000 | 0 | | G/L ธรรมดา — **ไม่ใช่ WHT item** |
| 003 deferred | (ว่าง) | (ว่าง) | `DM` | `A` / `MWS` | 001 | +5,999.00 | | tax line แบบใส่ตรง (ต่างจาก invoice ที่เป็น `T`) |
| 004 output | (ว่าง) | (ว่าง) | `O1` | `A` / `MWS` | 002 | −5,999.00 | | assignment `94000000052026003` |
| 005 customer | (ว่าง) | (ว่าง) | | | 000 | 0 | `XX` | WHT base / amount = **0** · `IsUsedInPaymentTransaction X` · `NetPaymentAmount −6,418.93` |

สรุป:

- WHT line 002 เป็น **G/L ธรรมดา** ใส่เอง → class ส่งเป็น `_GLItems` ตรง ๆ ได้
- `WithholdingTaxCode XX` บน customer line มีทั้งบน invoice (จาก SD) และ payment
  โดย base/amount = 0 ทั้งคู่ → เป็นค่าที่ระบบ derive จาก customer master เอง
  ไม่ได้คำนวณ WHT → **class ไม่ต้องส่ง WHT node**
- tax line 003/004 ไม่ใช่ `ItemType T` แต่มี `TaxType A` + `MWS` + base
  → เป็น G/L line ที่ใส่ tax code แล้วระบบ enrich ให้ = ทางที่จะลองก่อนคือ
  `_GLItems` + `TaxCode` · ถ้า API สร้างบรรทัดภาษีงอกมาค่อยย้ายไป `_TaxItems`

---

## ผลที่ได้ — เอกสารที่ POC post: `3300000024` (Q2 + Q3 + Q6 · 2026-09-11)

post จาก `YCL_PAYMENT` rev 4 (option A · 3 บรรทัด) · reference `POC0911…`

| Item | PK | Type | Account | THB | Assignment | Text | House bank | Value date | bplace | อื่น ๆ |
|---|---|---|---|---:|---|---|---|---|---|---|
| 001 | 40 | S | G/L `0011092001` | +6,238.96 | `20260911` | รับผ่านช่องทาง mobile banking | `BBL01` / `CA001` | 2026-09-11 | `0000` | `PlanningLevel B0` |
| 002 | 40 | S | G/L `0011047003` | +179.97 | `20260911` | | | 2026-09-11 | `0000` | |
| 003 | **11** | D | Customer `0001000082` (recon `0011030001`) | −6,418.93 | | | | | `0000` | `InvoiceReference V` · `IsSalesRelated X` · `WithholdingTaxCode` ว่าง · `IsUsedInPaymentTransaction` ว่าง |

ACDOCA: 3 บรรทัด · `ProfitCenter DUMMY` · `Segment JASGROUP` · `BusinessTransactionType RFPI` · ไม่มี splitting line

### เทียบกับ `3300000017`

| หัวข้อ | `3300000017` (F-28) | `3300000024` (API) | สถานะ |
|---|---|---|---|
| bank line 001 | | เหมือนทุก field | ✅ |
| WHT line 002 | | เหมือนทุก field | ✅ |
| deferred/output tax 003/004 | มี (generate ตอน clear) | ไม่มี | ⚠️ ตัดออกตาม option A — ต้องเกิดจาก clearing / Transfer Deferred Tax |
| customer posting key | `15` incoming payment | `11` credit memo (+ `InvoiceReference V`) | ❌ ข้อจำกัด API: AR credit = 11 เท่านั้น |
| `IsUsedInPaymentTransaction` | `X` | ว่าง | ❌ ตามมากับ PK 11 |
| `WithholdingTaxCode` | `XX` (base/amount 0) | ว่าง | ❌ API ไม่ derive จาก customer master — อาจลอง `_WithHoldingTaxItems` |
| header: DZ / RFPI / BKPFF / business place / PC / segment | | เหมือน | ✅ |

