# 05 — Console Class `YCL_PAYMENT` (snapshot)

source of truth คือ tenant · ไฟล์นี้เป็น snapshot ไว้อ่านเท่านั้น
revision 10 · 2026-09-11 · `lv_simulate = abap_false` · เอกสารเดียว 5 บรรทัด · tax line เป็น `_GLItems` ระบุ G/L ตรง ๆ **ไม่มี tax code** (ไม่มี `_TaxItems`)

## แนวคิด

```
read_invoice   SELECT I_OperationalAcctgDocItem (invoice) → AR line (D) + tax line (T)
build_entry    derive 5 บรรทัด → TABLE FOR ACTION IMPORT i_journalentrytp~post
print_entry    พิมพ์ payload + balance check
post_entry     MODIFY ENTITIES … EXECUTE post → COMMIT ENTITIES → เลขเอกสาร
main           read → build → print → (gc_simulate = abap_false) post
```

| จาก invoice | ไป payment |
|---|---|
| AR line: `Customer`, amount +6,418.93 | **ใบ 1** `_ARItems[3]` amount −6,418.93 · `BusinessPlace 0000` |
| tax line (`ItemType T`): `TaxCode DM`, amount −419.93, base −5,999 | **ใบ 2** `_TaxItems[1]` `DM` +419.93 base +5,999 · `MWS` · direct |
| (ยอดเดียวกัน) | **ใบ 2** `_TaxItems[2]` `O1` −419.93 base −5,999 · `MWS` · direct |
| base 5,999 × 3% | **ใบ 1** `_GLItems[2]` `0011047003` +179.97 |
| ลูกหนี้ − WHT | **ใบ 1** `_GLItems[1]` `0011092001` +6,238.96 · `BBL01`/`CA001` |

ประวัติแก้:

| Rev | เปลี่ยน | เพราะ |
|---|---|---|
| 1 | ส่งครั้งแรก · tax line เป็น `_GLItems` + `TaxCode` | — |
| 2 | + `TaxDeterminationDate` ใน header | `Time dependent taxes: tax date has to be filled from caller` |
| 3 | tax line → `_TaxItems` (`MWS`, direct) · + `BusinessPlace 0000` ทุก G/L / AR line · assignment `94000000052026003` หายไป (`_TaxItems` ไม่มี field) | `Tax statement item missing for tax code DM` · `Enter a business place.` |
| 4 | **Option A**: comment `_taxitems` ออก → 3 บรรทัด (bank / WHT / customer) · customer line = `'3'` | `G/L account item without tax code in document with deferred taxes` — บรรทัด DM/O1 ของตัวอย่างเป็นของที่ generate ตอน clear · **post ผ่าน → `3300000024`** |
| 5 | ตัด `CONVERT KEY` ออกจาก `post_entry` (พิมพ์ `%pid` + `SELECT` ด้วย reference แทน) · `gc_simulate` กลับเป็น `abap_true` | dump `BEHAVIOR_STATEMENT_ILLEGAL` หลัง commit — `CONVERT KEY` ใช้ได้เฉพาะ save phase ของ RAP |
| 6 | คืน `_TaxItems` DM/O1 · WHT line [2] ใส่ `TaxCode O0` · customer กลับเป็น `'5'` | ทดลองสมมติฐาน: check deferred tax ดูเฉพาะบัญชี tax-relevant (bank ใส่ tax code ไม่ได้อยู่แล้ว) |
| 6b | ถอด `CONSTANTS` ทั้งหมด → literal ตรงจุดที่ใช้ · `gc_simulate` → `lv_simulate` ใน `main` | ผู้ใช้ขอให้อ่านง่ายตอน investigate (logic ไม่เปลี่ยน) |
| 7 | `build_entry` คืน 2 entries: DZ (bank/WHT/customer · WHT ไม่มี tax code) + `SA` `RFBU` มีแต่ `_TaxItems` DM/O1 · reference ใบ 2 = ใบ 1 + `T` · `post_entry` query `LIKE` ได้ทั้งคู่ | เอกสารเดียวชนกฎ deferred tax ที่บรรทัด bank → แยกใบ |
| 8 | tax item: + `ConditionType = 'MWAS'` · ลบ `TaxDeterminationDate` | `KSCHL is empty` · doc ProductTaxItem บอก TaxDeterminationDate "Do not use" · isolate ให้เหลือ blocker business place |
| 9 | กลับเป็นเอกสารเดียว 5 บรรทัด · `GLAccountLineItem` = customer 1 · WHT 2 · DM 3 · O1 4 · bank 5 · WHT ไม่มี tax code | ผู้ใช้ขอทดสอบว่าลำดับบรรทัดมีผลกับ check deferred tax ไหม |
| 10 | ไม่มี `_TaxItems` · บรรทัด DM/O1 เป็น `_GLItems` ระบุ `0021082005` / `0021082003` ตรง ๆ **ไม่ใส่ tax code** (มี comment ให้เปิดถ้าจะ re-test แบบมี tax code = rev 2) | ผู้ใช้ขอทดสอบ direct posting ไป tax account |

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

    " abap_true = พิมพ์ payload อย่างเดียว · abap_false = post จริง
    DATA(lv_simulate) = abap_false.

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
      ELSEIF ls_item-accountingdocumentitemtype = 'T'.    " tax line
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

    " assignment ตามเอกสารตัวอย่าง: bank/WHT = posting date · output tax = key ของ tax line บน invoice
    DATA(lv_assignment_date) = CONV ty_assignment( lv_today ).
    DATA(lv_tax_assignment)  = CONV ty_assignment( |9400000005{ '2026' }{ is_tax_item-accountingdocumentitem }| ).

    " reference ไม่ซ้ำต่อรอบ ไว้ query เอกสารกลับมา (16 chars)
    DATA(lv_reference) = CONV ty_reference( |POC{ lv_date_text+4(4) }{ lv_now }| ).

    " rev 10: ไม่มี _taxitems — บรรทัดภาษีเป็น _glitems ระบุ G/L ตรง ๆ (ไม่ใส่ tax code)
    "   แบบมี tax code = rev 2 → Tax statement item missing (base line) · เปิด comment 2 บรรทัด taxcode ถ้าจะ re-test
    rt_entries = VALUE #(
      ( %cid   = |PAY{ lv_now }|
        %param = VALUE #(
          companycode             = '1000'
          businesstransactiontype = 'RFPI'                             " ตามเอกสารตัวอย่าง
          accountingdocumenttype  = 'DZ'
          documentdate            = lv_today
          postingdate             = lv_today
          documentreferenceid     = lv_reference
          taxdeterminationdate    = lv_today
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
              businessplace       = '0000'                             " Thai: head office
              _currencyamount     = VALUE #( ( currencyrole           = '00'   " transaction currency
                                               currency               = lv_currency
                                               journalentryitemamount = lv_bank_amount ) ) )
            " [2] ภาษีหัก ณ ที่จ่ายที่ลูกค้าหักไว้
            ( glaccountlineitem   = '2'
              glaccount           = '0011047003'
              assignmentreference = lv_assignment_date
              valuedate           = lv_today
              businessplace       = '0000'
              _currencyamount     = VALUE #( ( currencyrole           = '00'
                                               currency               = lv_currency
                                               journalentryitemamount = lv_wht_amount ) ) )
            " [3] โอนออกจาก deferred output tax — G/L ตรง ๆ
            ( glaccountlineitem   = '3'
              glaccount           = is_tax_item-glaccount                 " 0021082005
*              taxcode             = is_tax_item-taxcode                  " DM — เปิดถ้าจะ re-test แบบมี tax code
              valuedate           = lv_today
              businessplace       = '0000'
              _currencyamount     = VALUE #( ( currencyrole           = '00'
                                               currency               = lv_currency
                                               journalentryitemamount = lv_tax_amount
                                               taxbaseamount          = lv_tax_base ) ) )
            " [4] เข้า output tax — G/L ตรง ๆ
            ( glaccountlineitem   = '4'
              glaccount           = '0021082003'
*              taxcode             = 'O1'                                " เปิดถ้าจะ re-test แบบมี tax code
              assignmentreference = lv_tax_assignment
              valuedate           = lv_today
              businessplace       = '0000'
              _currencyamount     = VALUE #( ( currencyrole           = '00'
                                               currency               = lv_currency
                                               journalentryitemamount = - lv_tax_amount
                                               taxbaseamount          = - lv_tax_base ) ) ) )

          _aritems = VALUE #(
            " [5] ตัดลูกหนี้ — ไม่ใส่ GLAccount ให้ระบบ derive reconciliation account เอง
            ( glaccountlineitem = '5'
              customer          = is_ar_item-customer                    " 0001000082
              businessplace     = '0000'
              _currencyamount   = VALUE #( ( currencyrole           = '00'
                                             currency               = lv_currency
                                             journalentryitemamount = lv_customer_amount ) ) ) ) ) ) ).
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
                       |direct { ls_tax-isdirecttaxposting } { ls_tax_amount-journalentryitemamount } { ls_tax_amount-currency } | &&
                       |base { ls_tax_amount-taxbaseamount } taxdate { ls_tax-taxdeterminationdate }| ).
      ENDLOOP.

      LOOP AT ls_entry-%param-_aritems INTO DATA(ls_ar).
        READ TABLE ls_ar-_currencyamount INTO DATA(ls_ar_amount) INDEX 1.
        lv_total += ls_ar_amount-journalentryitemamount.
        io_out->write( |  _ARItems[{ ls_ar-glaccountlineitem }] customer { ls_ar-customer } bplace { ls_ar-businessplace } | &&
                       |{ ls_ar_amount-journalentryitemamount } { ls_ar_amount-currency }| ).
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

    " %pid จาก late numbering — แค่พิมพ์ไว้ดู (ห้าม CONVERT KEY ตรงนี้ → BEHAVIOR_STATEMENT_ILLEGAL)
    LOOP AT ls_mapped-journalentry INTO DATA(ls_mapped_entry).
      io_out->write( |COMMITTED: %cid { ls_mapped_entry-%cid } %pid { ls_mapped_entry-%pid }| ).
    ENDLOOP.

    " เลขเอกสารจริง: query ด้วย reference ของทุกเอกสารในรอบนี้
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
