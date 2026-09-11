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

---

## ผลที่ได้ (export 2026-09-11 · ทุก query compile ผ่านทุก field)

เอกสารตัวอย่าง = **`3300000017` / 2026 / company code `1000`** — ตัวเดียวกับ payment
ที่ใช้ใน POC clearing

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
