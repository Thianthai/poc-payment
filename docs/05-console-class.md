# 05 — Console Class `YCL_PAYMENT` (snapshot)

source of truth คือ tenant · ไฟล์นี้เป็น snapshot ไว้อ่านเท่านั้น
**revision 18 (final)** · 2026-09-14 · `lv_simulate = abap_true` · rev 17 + `_WithHoldingTaxItems` type MA/09 บนบรรทัดลูกหนี้
· พิสูจน์แล้ว: post `3300000031` + `7200000002` → POC clearing clear ผ่านด้วย `3000000005`

## แนวคิด

```
read_invoice   SELECT I_OperationalAcctgDocItem (invoice) → AR line (D) + tax line (T)
build_entry    derive 2 เอกสาร → TABLE FOR ACTION IMPORT i_journalentrytp~post (2 entries)
print_entry    พิมพ์ payload + balance check ต่อใบ
post_entry     MODIFY ENTITIES … EXECUTE post (ทั้ง 2 ใบ) → COMMIT ENTITIES → SELECT เลขเอกสารด้วย reference
main           read → build → print → (lv_simulate = abap_false) post
```

| จาก invoice `9400000005` | ไป |
|---|---|
| AR line: `Customer`, +6,418.93 | **ใบ 1** `_ARItems[3]` −6,418.93 · bplace `0000` (ได้ PK 11) |
| tax line (`ItemType T`): base −5,999 × 3% | **ใบ 1** `_GLItems[2]` WHT `0011047003` +179.97 |
| ลูกหนี้ − WHT | **ใบ 1** `_GLItems[1]` bank `0011092001` +6,238.96 · `BBL01`/`CA001` |
| tax line: `TaxCode DM`, −419.93, base −5,999 | **ใบ 2** `_TaxItems[3]` `DM` +419.93 base +5,999 · `MWS` · `MWAS` · direct |
| (ยอดเดียวกัน) | **ใบ 2** `_TaxItems[4]` `O1` −419.93 base −5,999 · `MWS` · `MWAS` · direct |
| (ยอดเดียวกัน) | **ใบ 2** `_GLItems[1]/[2]` คู่ dummy `0011054001` ±419.93 — ให้ tax item derive business place |

ใบ 2 ยังมี Custom Logic `YY1_FIN_ACDOC_ITEM_SUBSTITUTIO` (BAdI `FIN_ACDOC_ITEM_SUBSTITUTION`) ใส่ assignment
ให้บรรทัด `O1` — อยู่นอก class

## ประวัติแก้ (ย่อ — รายละเอียดเต็มใน docs/02)

| Rev | เปลี่ยน | เพราะ |
|---|---|---|
| 1–2 | ส่งครั้งแรก · + `TaxDeterminationDate` header | `Time dependent taxes: tax date has to be filled from caller` |
| 3 | tax line → `_TaxItems` direct · + `BusinessPlace 0000` | `Tax statement item missing for tax code DM` · `Enter a business place.` |
| 4–5 | ตัด tax line → DZ 3 บรรทัด **post ผ่าน `3300000024`** · ตัด `CONVERT KEY` | `G/L account item without tax code in document with deferred taxes` · dump `BEHAVIOR_STATEMENT_ILLEGAL` |
| 6–9 | ใบเดียว 5 บรรทัด: `O0` บน WHT / เรียงลำดับใหม่ | ยังชน deferred check ที่ bank (tax category ว่าง) |
| 7–8 | 2 ใบ · ใบ 2 tax item ล้วน + `MWAS` | `KSCHL is empty` → แก้ · เหลือ `Enter a business place.` (tax item ไม่มี field) |
| 10–16 | tax line เป็น `_GLItems` ทุกแบบ (ไม่มี/มี tax code · + tax item amount 0 · item ref · เลขซ้ำ) | `requires a valid tax code` · `Tax statement item missing` · FF 817 · `tax base 0` · `Line item entered several times` |
| **17** | **ใบ 2 = คู่ dummy `_GLItems` `0011054001` ±419.93 + `_TaxItems` DM/O1 direct** | คู่ G/L ให้ tax item derive business place · **post ผ่าน `7200000001`** (2026-09-14) |
| **18** | **ใบ 1 + `_WithHoldingTaxItems`** (`GLAccountLineItem 3` · type `MA` · code `09` · amount/base 0 · manual flag) | POC clearing: `3300000026` clear ไม่ได้ (F5 787) — บรรทัดลูกหนี้ไม่มี WHT type ตาม customer master · **แก้แล้ว clear ผ่าน `3000000005`** |

## Source

```abap
CLASS ycl_payment DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC.

  PUBLIC SECTION.
    INTERFACES if_oo_adt_classrun.

  PRIVATE SECTION.
    TYPES: BEGIN OF ty_invoice_item,
             accountingdocumentitem      TYPE i_operationalacctgdocitem-accountingdocumentitem,
             accountingdocumentitemtype  TYPE i_operationalacctgdocitem-accountingdocumentitemtype,
             financialaccounttype        TYPE i_operationalacctgdocitem-financialaccounttype,
             glaccount                   TYPE i_operationalacctgdocitem-glaccount,
             customer                    TYPE i_operationalacctgdocitem-customer,
             taxcode                     TYPE i_operationalacctgdocitem-taxcode,
             transactioncurrency         TYPE i_operationalacctgdocitem-transactioncurrency,
             amountintransactioncurrency TYPE i_operationalacctgdocitem-amountintransactioncurrency,
             taxbaseamountintranscrcy    TYPE i_operationalacctgdocitem-taxbaseamountintranscrcy,
             clearingaccountingdocument  TYPE i_operationalacctgdocitem-clearingaccountingdocument,
           END OF ty_invoice_item.
    TYPES tt_invoice_item TYPE STANDARD TABLE OF ty_invoice_item WITH EMPTY KEY.
    TYPES tt_entry        TYPE TABLE FOR ACTION IMPORT i_journalentrytp~post.
    TYPES ty_assignment   TYPE c LENGTH 18.   " แทน acpi_zuonr (data element ไม่ released)
    TYPES ty_reference    TYPE c LENGTH 16.   " แทน xblnr

    METHODS read_invoice
      IMPORTING io_out       TYPE REF TO if_oo_adt_classrun_out
      EXPORTING es_ar_item   TYPE ty_invoice_item
                es_tax_item  TYPE ty_invoice_item
      RETURNING VALUE(rv_ok) TYPE abap_boolean.

    METHODS build_entry
      IMPORTING is_ar_item        TYPE ty_invoice_item
                is_tax_item       TYPE ty_invoice_item
      RETURNING VALUE(rt_entries) TYPE tt_entry.

    METHODS print_entry
      IMPORTING it_entries TYPE tt_entry
                io_out     TYPE REF TO if_oo_adt_classrun_out.

    METHODS post_entry
      IMPORTING it_entries TYPE tt_entry
                io_out     TYPE REF TO if_oo_adt_classrun_out.
ENDCLASS.



CLASS ycl_payment IMPLEMENTATION.

  METHOD if_oo_adt_classrun~main.
    DATA ls_ar_item  TYPE ty_invoice_item.
    DATA ls_tax_item TYPE ty_invoice_item.

    " abap_true = พิมพ์ payload อย่างเดียว · abap_false = post จริง (ได้เอกสารใหม่ทุกครั้งที่กด F9)
    DATA(lv_simulate) = abap_true.

    out->write( 'YCL_PAYMENT — post incoming payment from invoice 1000 / 9400000005 / 2026' ).

    DATA(lv_ok) = read_invoice( EXPORTING io_out      = out
                                IMPORTING es_ar_item  = ls_ar_item
                                          es_tax_item = ls_tax_item ).
    IF lv_ok = abap_false.
      RETURN.
    ENDIF.

    DATA(lt_entries) = build_entry( is_ar_item  = ls_ar_item
                                    is_tax_item = ls_tax_item ).

    print_entry( it_entries = lt_entries
                 io_out     = out ).

    IF lv_simulate = abap_true.
      out->write( 'SIMULATE mode — nothing posted. Set lv_simulate = abap_false to post.' ).
      RETURN.
    ENDIF.

    post_entry( it_entries = lt_entries
                io_out     = out ).
  ENDMETHOD.


  METHOD read_invoice.
    DATA lt_items     TYPE tt_invoice_item.
    DATA lv_ar_count  TYPE i.
    DATA lv_tax_count TYPE i.

    rv_ok = abap_false.
    CLEAR: es_ar_item, es_tax_item.

    " ---------- input: invoice ที่จะเอามา post payment ----------
    SELECT accountingdocumentitem,
           accountingdocumentitemtype,
           financialaccounttype,
           glaccount,
           customer,
           taxcode,
           transactioncurrency,
           amountintransactioncurrency,
           taxbaseamountintranscrcy,
           clearingaccountingdocument
      FROM i_operationalacctgdocitem
      WHERE companycode        = '1000'
        AND accountingdocument = '9400000005'
        AND fiscalyear         = '2026'
      ORDER BY accountingdocumentitem
      INTO CORRESPONDING FIELDS OF TABLE @lt_items.

    IF lt_items IS INITIAL.
      io_out->write( 'ERROR: invoice 9400000005 / 2026 not found in company code 1000' ).
      RETURN.
    ENDIF.

    io_out->write( |--- Invoice 9400000005: { lines( lt_items ) } line(s) ---| ).
    LOOP AT lt_items INTO DATA(ls_item).
      io_out->write( |  { ls_item-accountingdocumentitem } type { ls_item-financialaccounttype }/{ ls_item-accountingdocumentitemtype } | &&
                     |G/L { ls_item-glaccount } cust { ls_item-customer } tax { ls_item-taxcode } | &&
                     |{ ls_item-amountintransactioncurrency } { ls_item-transactioncurrency } | &&
                     |base { ls_item-taxbaseamountintranscrcy } clrg { ls_item-clearingaccountingdocument }| ).

      IF ls_item-financialaccounttype = 'D'.              " customer line
        lv_ar_count += 1.
        es_ar_item = ls_item.
      ELSEIF ls_item-accountingdocumentitemtype = 'T'.    " tax line (deferred)
        lv_tax_count += 1.
        es_tax_item = ls_item.
      ENDIF.
    ENDLOOP.

    " POC รองรับแค่เคส 1 customer line + 1 tax line (แบบ 9400000005)
    IF lv_ar_count <> 1 OR lv_tax_count <> 1.
      io_out->write( |ERROR: POC supports exactly 1 customer line + 1 tax line, found { lv_ar_count } / { lv_tax_count }| ).
      RETURN.
    ENDIF.

    IF es_ar_item-clearingaccountingdocument IS NOT INITIAL.
      io_out->write( |ERROR: customer line already cleared by { es_ar_item-clearingaccountingdocument }| ).
      RETURN.
    ENDIF.

    rv_ok = abap_true.
  ENDMETHOD.


  METHOD build_entry.
    DATA lv_customer_amount TYPE wrbtr.
    DATA lv_tax_amount      TYPE wrbtr.
    DATA lv_tax_base        TYPE wrbtr.
    DATA lv_wht_amount      TYPE wrbtr.
    DATA lv_bank_amount     TYPE wrbtr.

    DATA(lv_today)     = cl_abap_context_info=>get_system_date( ).
    DATA(lv_now)       = cl_abap_context_info=>get_system_time( ).
    DATA(lv_currency)  = is_ar_item-transactioncurrency.                " THB
    DATA(lv_date_text) = CONV string( lv_today ).
    DATA(lv_user)      = cl_abap_context_info=>get_user_technical_name( ).

    " ยอด: เดบิต = บวก · เครดิต = ลบ → กลับเครื่องหมายจาก invoice
    lv_customer_amount = - is_ar_item-amountintransactioncurrency.       " Cr ลูกหนี้      −6,418.93
    lv_tax_amount      = - is_tax_item-amountintransactioncurrency.      " Dr deferred tax   +419.93
    lv_tax_base        = - is_tax_item-taxbaseamountintranscrcy.         " base            +5,999.00
    lv_wht_amount      = lv_tax_base * 3 / 100.                          " Dr WHT 3%         +179.97
    lv_bank_amount     = - lv_customer_amount - lv_wht_amount.           " Dr bank         +6,238.96

    " assignment ตามเอกสารตัวอย่าง: bank/WHT = posting date
    DATA(lv_assignment_date) = CONV ty_assignment( lv_today ).

    " reference ไม่ซ้ำต่อรอบ ไว้ query เอกสารกลับมา (16 chars)
    " ใบ 1 = POC0911113752 · ใบ 2 = POC0911113752T
    DATA(lv_reference)     = CONV ty_reference( |POC{ lv_date_text+4(4) }{ lv_now }| ).
    DATA(lv_reference_tax) = CONV ty_reference( |{ lv_reference }T| ).

    " ---------------------------------------------------------------------------------------
    " ทำไมต้อง 2 ใบ (functional ยอมรับ 2026-09-14):
    "   ใบเดียว 5 บรรทัดชนกฎ "เอกสารที่มี deferred tax code ทุก G/L line ต้องมี tax code"
    "   ที่บรรทัด bank (tax category ว่าง ใส่ tax code ไม่ได้) → แยกบรรทัดโอน DM→O1 ออกเป็นใบ SA
    " ---------------------------------------------------------------------------------------
    rt_entries = VALUE #(

      " ================= ใบ 1: payment DZ (bank / WHT / customer) → เช่น 3300000026 =================
      ( %cid   = |PAY{ lv_now }|
        %param = VALUE #(
          companycode             = '1000'
          businesstransactiontype = 'RFPI'                             " ตามเอกสารตัวอย่าง (API รับ)
          accountingdocumenttype  = 'DZ'
          documentdate            = lv_today
          postingdate             = lv_today
          documentreferenceid     = lv_reference
          taxdeterminationdate    = lv_today                           " time-dependent tax เปิดอยู่ → บังคับ
          createdbyuser           = lv_user

          _glitems = VALUE #(
            " [1] เงินเข้าธนาคาร
            ( glaccountlineitem   = '1'
              glaccount           = '0011092001'
              documentitemtext    = 'รับผ่านช่องทาง mobile banking'
              assignmentreference = lv_assignment_date
              valuedate           = lv_today
              housebank           = 'BBL01'
              housebankaccount    = 'CA001'
              businessplace       = '0000'                             " Thai: head office — บังคับ
              _currencyamount     = VALUE #( ( currencyrole           = '00'   " transaction currency
                                               currency               = lv_currency
                                               journalentryitemamount = lv_bank_amount ) ) )
            " [2] ภาษีหัก ณ ที่จ่ายที่ลูกค้าหักไว้ — ไม่ใส่ tax code (ตามตัวอย่าง)
            ( glaccountlineitem   = '2'
              glaccount           = '0011047003'
              assignmentreference = lv_assignment_date
              valuedate           = lv_today
              businessplace       = '0000'
              _currencyamount     = VALUE #( ( currencyrole           = '00'
                                               currency               = lv_currency
                                               journalentryitemamount = lv_wht_amount ) ) ) )

          _aritems = VALUE #(
            " [3] ตัดลูกหนี้ — ไม่ใส่ GLAccount ให้ระบบ derive reconciliation account เอง
            "     ได้ PK 11 (credit memo) ไม่ใช่ 15 — ข้อจำกัดของ API
            ( glaccountlineitem = '3'
              customer          = is_ar_item-customer                    " 0001000082
              businessplace     = '0000'
              _currencyamount   = VALUE #( ( currencyrole           = '00'
                                             currency               = lv_currency
                                             journalentryitemamount = lv_customer_amount ) ) ) )

          " WHT info ของบรรทัดลูกหนี้ — ต้องมีครบทุก type ตาม customer master ไม่งั้น clearing ปฏิเสธ
          " ("open items display different withholding tax information from the business partner master record")
          " customer 0001000082 มี 1 type: MA (A/R at payment) code 09 (Service 3%)
          " amount/base = 0 + manual flag → แค่ให้ข้อมูล type ติดไปกับบรรทัด ไม่ให้ระบบคำนวณ/สร้างบรรทัด WHT ซ้อน
          " (บรรทัด WHT 179.97 ส่งเป็น G/L 0011047003 อยู่แล้ว ตามเอกสารตัวอย่าง)
          _withholdingtaxitems = VALUE #(
            ( glaccountlineitem             = '3'                        " ผูกกับบรรทัดลูกหนี้ [3]
              withholdingtaxtype            = 'MA'
              withholdingtaxcode            = '09'
              whldgtaxisenteredmanually     = abap_true
              whldgtaxbaseisenteredmanually = abap_true
              _currencyamount               = VALUE #( ( currencyrole           = '00'
                                                         currency               = lv_currency
                                                         journalentryitemamount = 0
                                                         taxbaseamount          = 0 ) ) ) ) ) )

      " ================= ใบ 2: โอน deferred tax DM → O1 (SA) → เช่น 7200000001 =================
      ( %cid   = |TAX{ lv_now }|
        %param = VALUE #(
          companycode             = '1000'
          businesstransactiontype = 'RFBU'                             " G/L posting ธรรมดา
          accountingdocumenttype  = 'SA'
          documentdate            = lv_today
          postingdate             = lv_today
          documentreferenceid     = lv_reference_tax
          taxdeterminationdate    = lv_today
          createdbyuser           = lv_user

          " คู่ dummy บน 0011054001 (net 0 · ไม่ใช่ open item managed · tax category ว่าง)
          " หน้าที่: ให้ tax item มี G/L line ในใบเดียวกันไว้ derive business place
          " (ใบที่มีแต่ _TaxItems ได้ "Enter a business place." เพราะ tax item ไม่มี field นี้)
          _glitems = VALUE #(
            ( glaccountlineitem   = '1'
              glaccount           = '0011054001'
              assignmentreference = lv_assignment_date
              valuedate           = lv_today
              businessplace       = '0000'
              _currencyamount     = VALUE #( ( currencyrole           = '00'
                                               currency               = lv_currency
                                               journalentryitemamount = lv_tax_amount ) ) )   " +419.93
            ( glaccountlineitem   = '2'
              glaccount           = '0011054001'
              assignmentreference = lv_assignment_date
              valuedate           = lv_today
              businessplace       = '0000'
              _currencyamount     = VALUE #( ( currencyrole           = '00'
                                               currency               = lv_currency
                                               journalentryitemamount = - lv_tax_amount ) ) ) )   " −419.93

          " tax item แบบ direct — G/L derive จาก TaxCode + TaxItemClassification MWS
          "   DM → 0021082005 · O1 → 0021082003
          " ConditionType ต้องใส่ (ไม่งั้น "KSCHL is empty") · TaxDeterminationDate บน item ห้ามใส่ (doc)
          " ส่งเป็น _GLItems + tax code ไม่ได้ (base line → "Tax statement item missing")
          " assignment บนบรรทัด O1 มาจาก Custom Logic YY1_FIN_ACDOC_ITEM_SUBSTITUTIO (BAdI) ไม่ใช่จากตรงนี้
          _taxitems = VALUE #(
            " [3] โอนออกจาก deferred output tax (tax code DM จาก invoice)
            ( glaccountlineitem     = '3'
              taxcode               = is_tax_item-taxcode                " DM
              taxitemclassification = 'MWS'
              conditiontype         = 'MWAS'
              isdirecttaxposting    = abap_true
              _currencyamount       = VALUE #( ( currencyrole           = '00'
                                                 currency               = lv_currency
                                                 journalentryitemamount = lv_tax_amount      " +419.93
                                                 taxbaseamount          = lv_tax_base ) ) )  " +5,999
            " [4] เข้า output tax
            ( glaccountlineitem     = '4'
              taxcode               = 'O1'                              " target ของ DM
              taxitemclassification = 'MWS'
              conditiontype         = 'MWAS'
              isdirecttaxposting    = abap_true
              _currencyamount       = VALUE #( ( currencyrole           = '00'
                                                 currency               = lv_currency
                                                 journalentryitemamount = - lv_tax_amount    " −419.93
                                                 taxbaseamount          = - lv_tax_base ) ) ) ) ) ) ).
  ENDMETHOD.


  METHOD print_entry.
    DATA lv_total TYPE wrbtr.

    LOOP AT it_entries INTO DATA(ls_entry).
      io_out->write( |--- Payload (%cid { ls_entry-%cid }) ---| ).
      io_out->write( |Header: CoCd { ls_entry-%param-companycode } DocType { ls_entry-%param-accountingdocumenttype } | &&
                     |BTType { ls_entry-%param-businesstransactiontype } DocDate { ls_entry-%param-documentdate } | &&
                     |PostDate { ls_entry-%param-postingdate } TaxDate { ls_entry-%param-taxdeterminationdate } | &&
                     |Ref { ls_entry-%param-documentreferenceid } CreatedBy { ls_entry-%param-createdbyuser }| ).

      CLEAR lv_total.

      LOOP AT ls_entry-%param-_glitems INTO DATA(ls_gl).
        READ TABLE ls_gl-_currencyamount INTO DATA(ls_gl_amount) INDEX 1.
        lv_total += ls_gl_amount-journalentryitemamount.
        io_out->write( |  _GLItems[{ ls_gl-glaccountlineitem }] G/L { ls_gl-glaccount } | &&
                       |{ ls_gl_amount-journalentryitemamount } { ls_gl_amount-currency } | &&
                       |tax { ls_gl-taxcode } assign { ls_gl-assignmentreference } value { ls_gl-valuedate } bplace { ls_gl-businessplace } | &&
                       |bank { ls_gl-housebank }/{ ls_gl-housebankaccount } text { ls_gl-documentitemtext }| ).
      ENDLOOP.

      LOOP AT ls_entry-%param-_taxitems INTO DATA(ls_tax).
        READ TABLE ls_tax-_currencyamount INTO DATA(ls_tax_amount) INDEX 1.
        lv_total += ls_tax_amount-journalentryitemamount.
        io_out->write( |  _TaxItems[{ ls_tax-glaccountlineitem }] tax { ls_tax-taxcode } class { ls_tax-taxitemclassification } | &&
                       |cond { ls_tax-conditiontype } direct { ls_tax-isdirecttaxposting } | &&
                       |{ ls_tax_amount-journalentryitemamount } { ls_tax_amount-currency } base { ls_tax_amount-taxbaseamount }| ).
      ENDLOOP.

      LOOP AT ls_entry-%param-_aritems INTO DATA(ls_ar).
        READ TABLE ls_ar-_currencyamount INTO DATA(ls_ar_amount) INDEX 1.
        lv_total += ls_ar_amount-journalentryitemamount.
        io_out->write( |  _ARItems[{ ls_ar-glaccountlineitem }] customer { ls_ar-customer } bplace { ls_ar-businessplace } | &&
                       |{ ls_ar_amount-journalentryitemamount } { ls_ar_amount-currency }| ).
      ENDLOOP.

      LOOP AT ls_entry-%param-_withholdingtaxitems INTO DATA(ls_wht).
        READ TABLE ls_wht-_currencyamount INTO DATA(ls_wht_amount) INDEX 1.
        io_out->write( |  _WithHoldingTaxItems[{ ls_wht-glaccountlineitem }] type { ls_wht-withholdingtaxtype } | &&
                       |code { ls_wht-withholdingtaxcode } { ls_wht_amount-journalentryitemamount } | &&
                       |base { ls_wht_amount-taxbaseamount } manual { ls_wht-whldgtaxisenteredmanually }/{ ls_wht-whldgtaxbaseisenteredmanually }| ).
      ENDLOOP.

      io_out->write( |Balance check: { lv_total } (must be 0)| ).
    ENDLOOP.
  ENDMETHOD.


  METHOD post_entry.
    io_out->write( |--- Posting { lines( it_entries ) } document(s) via I_JournalEntryTP~Post ---| ).

    MODIFY ENTITIES OF i_journalentrytp
      ENTITY journalentry
      EXECUTE post FROM it_entries
      MAPPED   DATA(ls_mapped)
      FAILED   DATA(ls_failed)
      REPORTED DATA(ls_reported).

    LOOP AT ls_reported-journalentry INTO DATA(ls_reported_entry).
      io_out->write( |  MSG [{ ls_reported_entry-%cid }]: { ls_reported_entry-%msg->if_message~get_text( ) }| ).
    ENDLOOP.

    " fail ใบใดใบหนึ่ง = rollback ทั้งหมด ไม่มีใบไหนค้าง
    IF ls_failed-journalentry IS NOT INITIAL.
      LOOP AT ls_failed-journalentry INTO DATA(ls_failed_entry).
        io_out->write( |  FAILED: %cid { ls_failed_entry-%cid }| ).
      ENDLOOP.
      io_out->write( 'FAILED in EXECUTE post — rolling back, nothing posted' ).
      ROLLBACK ENTITIES.
      RETURN.
    ENDIF.

    COMMIT ENTITIES
      RESPONSE OF i_journalentrytp
      FAILED   DATA(ls_commit_failed)
      REPORTED DATA(ls_commit_reported).
    DATA(lv_commit_subrc) = sy-subrc.

    LOOP AT ls_commit_reported-journalentry INTO DATA(ls_commit_msg).
      io_out->write( |  MSG (commit) [{ ls_commit_msg-%pid }]: { ls_commit_msg-%msg->if_message~get_text( ) }| ).
    ENDLOOP.

    IF lv_commit_subrc <> 0 OR ls_commit_failed-journalentry IS NOT INITIAL.
      io_out->write( 'FAILED in COMMIT ENTITIES — nothing posted' ).
      RETURN.
    ENDIF.

    " %pid จาก late numbering — แค่พิมพ์ไว้ดู
    " ห้ามใช้ CONVERT KEY ตรงนี้ → dump BEHAVIOR_STATEMENT_ILLEGAL (ใช้ได้เฉพาะ save phase ของ RAP)
    LOOP AT ls_mapped-journalentry INTO DATA(ls_mapped_entry).
      io_out->write( |COMMITTED: %cid { ls_mapped_entry-%cid } %pid { ls_mapped_entry-%pid }| ).
    ENDLOOP.

    " เลขเอกสารจริง: query ด้วย reference ของทุกใบในรอบนี้ (POC…, POC…T)
    DATA(lv_pattern) = |{ it_entries[ 1 ]-%param-documentreferenceid }%|.
    SELECT companycode, accountingdocument, fiscalyear, accountingdocumenttype, postingdate, documentreferenceid
      FROM i_journalentry
      WHERE companycode         = '1000'
        AND documentreferenceid LIKE @lv_pattern
      ORDER BY documentreferenceid
      INTO TABLE @DATA(lt_posted).

    IF lt_posted IS INITIAL.
      io_out->write( |Reference { lv_pattern } not found in I_JournalEntry yet — check Manage Journal Entries| ).
    ENDIF.
    LOOP AT lt_posted INTO DATA(ls_posted).
      io_out->write( |POSTED: { ls_posted-accountingdocument } / { ls_posted-fiscalyear } type { ls_posted-accountingdocumenttype } posted { ls_posted-postingdate } ref { ls_posted-documentreferenceid }| ).
    ENDLOOP.
  ENDMETHOD.

ENDCLASS.
```
