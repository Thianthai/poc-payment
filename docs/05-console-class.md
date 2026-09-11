# 05 — Console Class `YCL_PAYMENT` (snapshot)

source of truth คือ tenant · ไฟล์นี้เป็น snapshot ไว้อ่านเท่านั้น
revision 4 · 2026-09-11 · `gc_simulate = abap_false` (อยู่ระหว่างทดสอบ post จริง)

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
| AR line: `Customer`, amount +6,418.93 | `_ARItems[3]` amount −6,418.93 · `BusinessPlace 0000` |
| tax line (`ItemType T`): base −5,999 | ใช้แค่ base ไปคิด WHT · ~~`_TaxItems` DM/O1~~ comment ไว้ (rev 4) |
| base 5,999 × `gc_wht_rate_percent` 3% | `_GLItems[2]` `0011047003` +179.97 |
| ลูกหนี้ − WHT | `_GLItems[1]` `0011092001` +6,238.96 · `BBL01`/`CA001` |

ประวัติแก้:

| Rev | เปลี่ยน | เพราะ |
|---|---|---|
| 1 | ส่งครั้งแรก · tax line เป็น `_GLItems` + `TaxCode` | — |
| 2 | + `TaxDeterminationDate` ใน header | `Time dependent taxes: tax date has to be filled from caller` |
| 3 | tax line → `_TaxItems` (`MWS`, direct) · + `BusinessPlace 0000` ทุก G/L / AR line · assignment `94000000052026003` หายไป (`_TaxItems` ไม่มี field) | `Tax statement item missing for tax code DM` · `Enter a business place.` |
| 4 | **Option A**: comment `_taxitems` ออก → 3 บรรทัด (bank / WHT / customer) · customer line = `'3'` | `G/L account item without tax code in document with deferred taxes` — บรรทัด DM/O1 ของตัวอย่างเป็นของที่ generate ตอน clear |

## Source

```abap
CLASS ycl_payment DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC.

  PUBLIC SECTION.
    INTERFACES if_oo_adt_classrun.

  PRIVATE SECTION.
    " ---------- input: invoice ที่จะเอามา post payment ----------
    CONSTANTS gc_company_code TYPE bukrs   VALUE '1000'.
    CONSTANTS gc_invoice      TYPE belnr_d VALUE '9400000005'.
    CONSTANTS gc_fiscal_year  TYPE gjahr   VALUE '2026'.

    " ---------- guard: abap_true = พิมพ์ payload อย่างเดียว ไม่ post ----------
    CONSTANTS gc_simulate TYPE abap_boolean VALUE abap_false.

    " ---------- header ----------
    CONSTANTS gc_document_type        TYPE blart VALUE 'DZ'.
    CONSTANTS gc_business_transaction TYPE glvor VALUE 'RFPI'.  " ตามเอกสารตัวอย่าง · ถ้า API ปฏิเสธ → 'RFBU'
    CONSTANTS gc_currency_role        TYPE curtp VALUE '00'.    " transaction currency

    " ---------- ค่าที่ไม่ได้มาจาก invoice (ตามเอกสารตัวอย่าง 3300000017) ----------
    CONSTANTS gc_gl_bank            TYPE hkont VALUE '0011092001'.
    CONSTANTS gc_house_bank         TYPE hbkid VALUE 'BBL01'.
    CONSTANTS gc_house_bank_account TYPE hktid VALUE 'CA001'.
    CONSTANTS gc_bank_text          TYPE sgtxt VALUE 'รับผ่านช่องทาง mobile banking'.
    CONSTANTS gc_gl_wht             TYPE hkont VALUE '0011047003'.
    CONSTANTS gc_wht_rate_percent   TYPE p LENGTH 3 DECIMALS 2 VALUE '3.00'.
    CONSTANTS gc_tax_code_output    TYPE mwskz VALUE 'O1'.       " target tax code ของ DM (G/L derive จาก code + MWS)
    CONSTANTS gc_tax_classification TYPE c LENGTH 3 VALUE 'MWS'. " account key → derive G/L ของ tax item
    CONSTANTS gc_business_place     TYPE c LENGTH 4 VALUE '0000'. " Thai: head office (ตามตัวอย่างทุกบรรทัด)

    " ---------- ตัวแยกบรรทัด invoice ----------
    CONSTANTS gc_account_type_customer TYPE i_operationalacctgdocitem-financialaccounttype       VALUE 'D'.
    CONSTANTS gc_item_type_tax         TYPE i_operationalacctgdocitem-accountingdocumentitemtype VALUE 'T'.

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

    out->write( |YCL_PAYMENT — post incoming payment from invoice { gc_company_code } / { gc_invoice } / { gc_fiscal_year }| ).

    " inline DATA( ) ใน IMPORTING ของ functional call ที่อยู่ใน IF ใช้ไม่ได้ → แยกประกาศ
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

    IF gc_simulate = abap_true.
      out->write( 'SIMULATE mode — nothing posted. Set gc_simulate = abap_false to post.' ).
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
      WHERE companycode        = @gc_company_code
        AND accountingdocument = @gc_invoice
        AND fiscalyear         = @gc_fiscal_year
      ORDER BY accountingdocumentitem
      INTO CORRESPONDING FIELDS OF TABLE @lt_items.

    IF lt_items IS INITIAL.
      io_out->write( |ERROR: invoice { gc_invoice } / { gc_fiscal_year } not found in company code { gc_company_code }| ).
      RETURN.
    ENDIF.

    io_out->write( |--- Invoice { gc_invoice }: { lines( lt_items ) } line(s) ---| ).
    LOOP AT lt_items INTO DATA(ls_item).
      io_out->write( |  { ls_item-accountingdocumentitem } type { ls_item-financialaccounttype }/{ ls_item-accountingdocumentitemtype } | &&
                     |G/L { ls_item-glaccount } cust { ls_item-customer } tax { ls_item-taxcode } | &&
                     |{ ls_item-amountintransactioncurrency } { ls_item-transactioncurrency } | &&
                     |base { ls_item-taxbaseamountintranscrcy } clrg { ls_item-clearingaccountingdocument }| ).

      IF ls_item-financialaccounttype = gc_account_type_customer.
        lv_ar_count += 1.
        es_ar_item = ls_item.
      ELSEIF ls_item-accountingdocumentitemtype = gc_item_type_tax.
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
    DATA(lv_currency)  = is_ar_item-transactioncurrency.
    DATA(lv_date_text) = CONV string( lv_today ).

    " ยอด: เดบิต = บวก · เครดิต = ลบ → กลับเครื่องหมายจาก invoice
    lv_customer_amount = - is_ar_item-amountintransactioncurrency.       " Cr ลูกหนี้      −6,418.93
    lv_tax_amount      = - is_tax_item-amountintransactioncurrency.      " Dr deferred tax   +419.93 (ไม่ได้ใช้ใน option A)
    lv_tax_base        = - is_tax_item-taxbaseamountintranscrcy.         " base            +5,999.00
    lv_wht_amount      = lv_tax_base * gc_wht_rate_percent / 100.        " Dr WHT            +179.97
    lv_bank_amount     = - lv_customer_amount - lv_wht_amount.           " Dr bank         +6,238.96

    " assignment ตามเอกสารตัวอย่าง: bank/WHT = posting date
    DATA(lv_assignment_date) = CONV ty_assignment( lv_today ).

    " reference ไม่ซ้ำต่อรอบ ไว้ query เอกสารกลับมา (16 chars)
    DATA(lv_reference) = CONV ty_reference( |POC{ lv_date_text+4(4) }{ lv_now }| ).

    rt_entries = VALUE #(
      ( %cid   = |PAY{ lv_now }|
        %param = VALUE #(
          companycode             = gc_company_code
          businesstransactiontype = gc_business_transaction
          accountingdocumenttype  = gc_document_type
          documentdate            = lv_today
          postingdate             = lv_today
          documentreferenceid     = lv_reference
          taxdeterminationdate    = lv_today                          " time-dependent tax เปิดอยู่ → ต้องส่ง
          createdbyuser           = cl_abap_context_info=>get_user_technical_name( )

          _glitems = VALUE #(
            " [1] เงินเข้าธนาคาร
            ( glaccountlineitem   = '1'
              glaccount           = gc_gl_bank
              documentitemtext    = gc_bank_text
              assignmentreference = lv_assignment_date
              valuedate           = lv_today
              housebank           = gc_house_bank
              housebankaccount    = gc_house_bank_account
              businessplace       = gc_business_place
              _currencyamount     = VALUE #( ( currencyrole           = gc_currency_role
                                               currency               = lv_currency
                                               journalentryitemamount = lv_bank_amount ) ) )
            " [2] ภาษีหัก ณ ที่จ่ายที่ลูกค้าหักไว้
            ( glaccountlineitem   = '2'
              glaccount           = gc_gl_wht
              assignmentreference = lv_assignment_date
              valuedate           = lv_today
              businessplace       = gc_business_place
              _currencyamount     = VALUE #( ( currencyrole           = gc_currency_role
                                               currency               = lv_currency
                                               journalentryitemamount = lv_wht_amount ) ) ) )

          " ---------- Option A (2026-09-11): ตัด tax line ออก ----------
          " บรรทัดโอน DM→O1 ในเอกสารตัวอย่างเป็นของที่ Post Incoming Payments generate ตอน clear
          " ถ้าใส่เองจะชน "G/L account item without tax code in document with deferred taxes"
          " (ทุก G/L line ต้องมี tax code เมื่อเอกสารมี deferred tax code)
          " → deferred tax transfer ให้เกิดตอน clear / job Transfer Deferred Tax แทน
*          _taxitems = VALUE #(
*            " [3] โอนออกจาก deferred output tax (tax code จาก invoice)
*            ( glaccountlineitem     = '3'
*              taxcode               = is_tax_item-taxcode
*              taxitemclassification = gc_tax_classification
*              isdirecttaxposting    = abap_true
*              taxdeterminationdate  = lv_today
*              _currencyamount       = VALUE #( ( currencyrole           = gc_currency_role
*                                                 currency               = lv_currency
*                                                 journalentryitemamount = lv_tax_amount
*                                                 taxbaseamount          = lv_tax_base ) ) )
*            " [4] เข้า output tax
*            ( glaccountlineitem     = '4'
*              taxcode               = gc_tax_code_output
*              taxitemclassification = gc_tax_classification
*              isdirecttaxposting    = abap_true
*              taxdeterminationdate  = lv_today
*              _currencyamount       = VALUE #( ( currencyrole           = gc_currency_role
*                                                 currency               = lv_currency
*                                                 journalentryitemamount = - lv_tax_amount
*                                                 taxbaseamount          = - lv_tax_base ) ) ) )

          _aritems = VALUE #(
            " [3] ตัดลูกหนี้ — ไม่ใส่ GLAccount ให้ระบบ derive reconciliation account เอง
            ( glaccountlineitem = '3'
              customer          = is_ar_item-customer
              businessplace     = gc_business_place
              _currencyamount   = VALUE #( ( currencyrole           = gc_currency_role
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
                       |assign { ls_gl-assignmentreference } value { ls_gl-valuedate } bplace { ls_gl-businessplace } | &&
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
    io_out->write( '--- Posting via I_JournalEntryTP~Post ---' ).

    MODIFY ENTITIES OF i_journalentrytp
      ENTITY journalentry
      EXECUTE post FROM it_entries
      MAPPED   DATA(ls_mapped)
      FAILED   DATA(ls_failed)
      REPORTED DATA(ls_reported).

    LOOP AT ls_reported-journalentry INTO DATA(ls_reported_entry).
      io_out->write( |  MSG: { ls_reported_entry-%msg->if_message~get_text( ) }| ).
    ENDLOOP.

    IF ls_failed-journalentry IS NOT INITIAL.
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
      io_out->write( |  MSG (commit): { ls_commit_msg-%msg->if_message~get_text( ) }| ).
    ENDLOOP.

    IF lv_commit_subrc <> 0 OR ls_commit_failed-journalentry IS NOT INITIAL.
      io_out->write( 'FAILED in COMMIT ENTITIES — nothing posted' ).
      RETURN.
    ENDIF.

    " เลขเอกสารจาก late numbering
    LOOP AT ls_mapped-journalentry INTO DATA(ls_mapped_entry).
      IF ls_mapped_entry-%pid IS NOT INITIAL.
        CONVERT KEY OF i_journalentrytp FROM ls_mapped_entry-%pid TO DATA(ls_key).
        io_out->write( |POSTED: { ls_key-companycode } { ls_key-accountingdocument } { ls_key-fiscalyear }| ).
      ELSE.
        io_out->write( |POSTED: { ls_mapped_entry-companycode } { ls_mapped_entry-accountingdocument } { ls_mapped_entry-fiscalyear }| ).
      ENDIF.
    ENDLOOP.

    " cross-check ด้วย reference ที่ generate ไว้
    DATA(lv_reference) = it_entries[ 1 ]-%param-documentreferenceid.
    SELECT companycode, accountingdocument, fiscalyear, accountingdocumenttype, postingdate
      FROM i_journalentry
      WHERE companycode         = @gc_company_code
        AND documentreferenceid = @lv_reference
      INTO TABLE @DATA(lt_posted).

    IF lt_posted IS INITIAL.
      io_out->write( |Reference { lv_reference } not found in I_JournalEntry yet — check Manage Journal Entries| ).
    ENDIF.
    LOOP AT lt_posted INTO DATA(ls_posted).
      io_out->write( |Verified: { ls_posted-accountingdocument } / { ls_posted-fiscalyear } type { ls_posted-accountingdocumenttype } posted { ls_posted-postingdate } ref { lv_reference }| ).
    ENDLOOP.
  ENDMETHOD.

ENDCLASS.
```
