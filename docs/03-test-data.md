# 03 — Test Data ที่ต้อง export จากเอกสารตัวอย่าง

ค่าทั้งหมดใน `YCL_PAYMENT` ที่เขียนว่า `CHANGE_ME` ต้องแทนด้วยค่าจริงจาก
**เอกสาร incoming payment ตัวอย่าง** บน tenant ก่อนรัน

เป้าหมาย: post ให้ได้เอกสารใหม่ที่ **บรรทัดต่อบรรทัดเหมือนตัวอย่าง** (ยกเว้นเลขเอกสาร / วันที่)

## 1. ค่าระดับ header

| Constant ใน class | ความหมาย | มาจาก field |
|---|---|---|
| `gc_company_code` | company code | `CompanyCode` |
| `gc_document_type` | doc type — คาดว่า `DZ` | `AccountingDocumentType` |
| `gc_currency` | สกุลเงินของเอกสาร | `TransactionCurrency` |
| `gc_reference` | reference (XBLNR) | `DocumentReferenceID` |
| `gc_header_text` | header text (BKTXT) | `AccountingDocumentHeaderText` |
| `gc_business_transaction` | เอกสารตัวอย่างเป็น **`RFPI`** — ลองค่านี้ก่อน ถ้า API ปฏิเสธค่อยถอยไป `RFBU` | `BusinessTransactionType` (ACDOCA) |
| `gc_simulate` | `abap_true` = พิมพ์ payload เฉย ๆ · `abap_false` = post จริง | เริ่มที่ `abap_true` เสมอ |

> `DocumentDate` / `PostingDate` class ใช้วันปัจจุบัน ถ้าต้องการ fix วันที่เอง
> แก้ที่ method `build_entry`

## 2. บรรทัดของเอกสาร

ต้องได้ **ทุกบรรทัดใน entry view** (BSEG) ของเอกสารตัวอย่าง ทั้ง G/L และ customer

| Field | ใช้ทำอะไร | มาจาก field |
|---|---|---|
| `AccountingDocumentItem` | ลำดับบรรทัด | `AccountingDocumentItem` |
| `PostingKey` | บอกว่าเป็น bank (40) / customer (15) / อื่น ๆ | `PostingKey` |
| `FinancialAccountType` | `S` = G/L → `_GLItems` · `D` = customer → `_ARItems` | `FinancialAccountType` |
| `GLAccount` | บัญชี G/L ของบรรทัด | `GLAccount` |
| `Customer` | เลข customer (บรรทัด D) | `Customer` |
| `SpecialGLCode` | ต้องว่างสำหรับ payment ปกติ | `SpecialGLCode` |
| `AmountInTransactionCurrency` | ยอด (เครดิตติดลบ) | `AmountInTransactionCurrency` |
| `DebitCreditCode` | cross-check เครื่องหมาย | `DebitCreditCode` |
| `ProfitCenter` · `CostCenter` | account assignment | `ProfitCenter` · `CostCenter` |
| `DocumentItemText` | item text | `DocumentItemText` |
| `AssignmentReference` | assignment | `AssignmentReference` |
| `HouseBank` · `HouseBankAccount` | บรรทัด bank | `HouseBank` · `HouseBankAccount` |
| `ValueDate` | value date บรรทัด bank | `ValueDate` |
| `TaxCode` | ถ้ามี | `TaxCode` |

บรรทัดใน ACDOCA ที่ `AccountingDocumentItem = 000` คือบรรทัดที่ document splitting
สร้างเอง **ไม่ต้องส่งเข้า API** — แต่ export มาด้วยเพื่อเทียบผลหลัง post

## 3. ค่าจริงจากเอกสารตัวอย่าง `3300000017` (export แล้ว 2026-09-11)

ตารางเต็มอยู่ที่ [04-data-export-sql.md](04-data-export-sql.md) · สรุปที่จะ fix ลง `YCL_PAYMENT`:

```
company_code   : 1000
doc_type       : DZ
currency       : THB
bus_trans_type : RFPI  (fallback RFBU)
reference      : 09080007  → POC จะ generate ใหม่ให้ unique ต่อรอบ
header_text    : (ว่าง)

_GLItems
  1  0011092001  +6238.96  text 'รับผ่านช่องทาง mobile banking'  assignment 20260909
                           house bank BBL01 / CA001  value date = posting date
  2  0011047003   +179.97  assignment 20260909  value date = posting date
  3  0021082005   +419.93  tax code DM
  4  0021082003   -419.93  tax code O1  assignment 94000000052026003
_ARItems
  5  customer 0001000082  -6418.93
```

## 4. รูปแบบที่สะดวกที่สุดสำหรับส่งข้อมูลมา

รัน query ใน [04-data-export-sql.md](04-data-export-sql.md) แล้ว paste ผลจาก ADT
console / SQL console มาตรง ๆ ได้เลย (Q1 header + Q2 items + Q3 ACDOCA)
ไม่ต้องจัดรูปแบบ

หรือถ้าถนัด Fiori: app **Manage Journal Entries** → เปิดเอกสาร → tab *Line Items*
screenshot ทั้ง entry view และ general ledger view ก็ใช้ได้ แต่ SQL ได้ field ครบกว่า
