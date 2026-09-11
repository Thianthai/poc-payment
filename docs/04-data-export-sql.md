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

## ผลที่ได้ (รอ export)

_ยังไม่มี — Claude จะเติมตารางตรงนี้หลังได้ผลจาก Q1–Q3_
