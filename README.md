# YPOC_PAYMENT — POC: Post Incoming Payment ผ่าน BO Interface `I_JournalEntryTP`

POC เรียก **released BO interface `I_JournalEntryTP`** (action `Post`) ด้วย EML
จาก ABAP Cloud console class บน **SAP S/4HANA Cloud Public Edition**
เพื่อ post เอกสาร incoming payment ให้ได้หน้าตาเดียวกับเอกสารตัวอย่างที่ post จาก Fiori

- BO interface บน SAP Business Accelerator Hub:
  <https://api.sap.com/bointerface/I_JOURNALENTRYTP>
- ขอบเขต: console class ตัวเดียว รัน posting ครั้งละ 1 เอกสาร
  ข้อมูลทดสอบ fix ไว้ใน code ทั้งหมด (ไม่มี UI / ไม่มี RAP ของตัวเอง)

## ต่างจาก POC clearing ยังไง

POC ก่อนหน้า ([`poc-clearing`](https://github.com/Thianthai/poc-clearing)) ต้องประกอบ SOAP
แล้วยิง HTTP กลับเข้า tenant ตัวเอง เพราะ Clearing API มีแค่แบบ inbound SOAP

รอบนี้ **ไม่ต้องยิง HTTP** — `I_JournalEntryTP` เป็น BO interface ที่ SAP release ให้
ABAP Cloud เรียกตรง ๆ ด้วย `MODIFY ENTITIES … EXECUTE Post` + `COMMIT ENTITIES`
จึงไม่มี communication scenario / arrangement / credential ให้ตั้งเลย

```
YCL_PAYMENT ──MODIFY ENTITIES OF i_journalentrytp──▶ I_JournalEntryTP~Post ──▶ ACDOCA / BSEG
 (console class)      EXECUTE post + COMMIT ENTITIES         (in-process)       เอกสาร DZ
```

## ขอบเขตที่ตกลงแล้ว (2026-09-11)

| | |
|---|---|
| ทำ | post เอกสาร incoming payment (doc type `DZ`) — Dr bank / Cr customer |
| Input | **invoice** (company code + เลขเอกสาร + ปี) — class อ่านบรรทัดของ invoice จาก CDS แล้ว derive บรรทัด payment เอง (เพิ่ม 2026-09-11 · ตัวอย่าง: invoice `9400000005` → payment `3300000017`) |
| **ไม่ทำ** | **clear open item ของ invoice** — `I_JournalEntryTP~Post` ไม่ clear ให้ (SAP ยืนยันเอง) ถ้าจะ clear ต้องต่อด้วย Clearing API จาก POC ก่อนหน้าเป็นอีกขั้น |
| Split | **functional ยอมรับ 2 ใบ** (2026-09-14): DZ payment 3 บรรทัด + ใบโอน deferred tax DM→O1 แยก — เพราะใบเดียว 5 บรรทัดชนกฎ deferred tax ที่บรรทัด bank |
| Simulate | `gc_simulate = abap_true` พิมพ์ payload ออก console ไม่ยิง `EXECUTE post` · เปลี่ยนเป็น `abap_false` เมื่อจะ post จริง (`Post` ไม่มี test-run flag ในตัว) |

ผลที่ได้จาก POC นี้คือเอกสาร DZ ที่ **ค้างเป็น open item บน customer** ไม่ได้ผูกกับ invoice

## ผลลัพธ์ POC — ✅ สำเร็จ · post 2 ใบ แล้ว clear ได้ด้วย POC clearing (2026-09-14)

`YCL_PAYMENT` อ่าน invoice `9400000005` แล้ว post **2 ใบใน commit เดียว** จากนั้น
[`poc-clearing`](https://github.com/Thianthai/poc-clearing) clear ทั้งหมดด้วย clearing document **`3000000005`**

| ใบ | Type | เอกสาร | บรรทัด |
|---|---|---|---|
| 1 payment | `DZ` `RFPI` | **`3300000031`** | bank `0011092001` +6,238.96 · WHT `0011047003` +179.97 · customer `0001000082` −6,418.93 (+ WHT item type `MA` code `09`) |
| 2 deferred tax transfer | `SA` `RFBU` | **`7200000002`** | dummy `0011054001` +419.93 / −419.93 · `DM` +419.93 base 5,999 · `O1` −419.93 base −5,999 (+ BAdI ใส่ assignment) |

clearing `3000000005` คลุม: invoice 001 (+6,418.93) ↔ payment 003 (−6,418.93) · invoice 003 deferred (−419.93) ↔ SA 003 (+419.93)

รอบก่อนหน้า: `3300000024` (ใบ 1 อย่างเดียว) · `3300000026` + `7200000001` (2 ใบ แต่ไม่มี WHT item → clear ไม่ได้ F5 787 · reverse แล้ว)

### เทียบกับตัวอย่าง `3300000017` (Post Incoming Payments)

| บรรทัด | ตัวอย่าง | API | สถานะ |
|---|---|---|---|
| bank +6,238.96 | PK 40 · house bank · value date · text · bplace | เหมือนทุก field | ✅ |
| WHT +179.97 | PK 40 | เหมือนทุก field | ✅ |
| customer −6,418.93 | **PK 15** · `WithholdingTaxCode XX` | **PK 11** credit memo · `XX` (จาก `_WithHoldingTaxItems`) | ⚠️ PK ต่าง แต่ clearing ยืนยันว่า clear ได้เหมือนกัน |
| deferred `DM` / output `O1` | อยู่ในใบ payment · `TTD = MWS` | อยู่**ใบ SA แยก** · `TTD` ว่าง · + คู่ dummy `0011054001` | ⚠️ functional ยอมรับ split · clear ได้ |
| assignment บน `O1` | `94000000052026003` | `94000000052026001` จาก Custom Logic (fix ค่า) | ⚠️ 3 หลักท้ายให้ functional ยืนยัน |

### สิ่งที่พิสูจน์ได้

- ABAP Cloud console class เรียก `I_JournalEntryTP~Post` ได้ตรง ๆ ไม่ต้องมี communication arrangement
- post หลายใบใน `MODIFY ENTITIES` + `COMMIT ENTITIES` เดียว — fail ใบใดใบหนึ่ง rollback ทั้งหมด
- บรรทัดโอน deferred tax post ผ่าน API ได้ **เฉพาะเมื่อแยกใบ** และมีคู่ dummy G/L net 0 ในใบเดียวกัน
  ให้ tax item derive business place — 16 รูปแบบอื่นชน check มาตรฐานของ FI ทั้งหมด (docs/02)
- **บรรทัดลูกหนี้ต้องมี `_WithHoldingTaxItems` ครบทุก type ตาม customer master** (amount 0 ได้)
  ไม่งั้น clearing ปฏิเสธ F5 787 — Fiori เติมให้เอง API ไม่เติม
- เอกสารจาก API (PK 11) clear กับ invoice ผ่าน Clearing API ได้ — PK 15 ไม่ใช่เงื่อนไข

### ข้อจำกัด / สิ่งที่ต้องทำต่อก่อน production

- WHT type/code ต้องอ่านจาก customer master ของลูกค้าแต่ละราย (หลาย type → หลาย entry · ไม่มี → ไม่ส่ง)
- Custom Logic `YY1_FIN_ACDOC_ITEM_SUBSTITUTIO` ใส่ assignment เป็นค่า fix — ต้อง derive จาก invoice
- ไม่ derive `WithholdingTaxCode` เอง · ไม่ clear (เป็นงานของ Clearing API) · PK 15 ทำไม่ได้
- คู่ dummy `0011054001` โผล่ใน line item ของบัญชีนั้น (net 0 · ไม่ใช่ OIM)
- reference ของใบ SA ถูก substitution ทับเป็น `0906xxxx` — หาเลขเอกสารจาก MSG commit แทน

## Object บน repo

abapGit serialize ด้วย `FOLDER_LOGIC = FULL` · `STARTING_FOLDER = /src/`

| Object | Type | Status |
|---|---|---|
| `YPOC_PAYMENT` | Package | ✅ [`src/package.devc.xml`](src/package.devc.xml) |
| `YCL_PAYMENT` | Class (console, `IF_OO_ADT_CLASSRUN`) | 🟡 rev 18 final อยู่ใน docs/05 · รอ push abapGit |

รายละเอียด + log อยู่ที่ [docs/02-object-list.md](docs/02-object-list.md)

## เอกสาร

| ไฟล์ | เนื้อหา |
|---|---|
| [docs/01-api-reference.md](docs/01-api-reference.md) | โครงสร้าง parameter ของ `Post`, ข้อจำกัด, จุดที่ต้องระวัง |
| [docs/02-object-list.md](docs/02-object-list.md) | รายการ ABAP object + สถานะ + log การทดสอบ |
| [docs/03-test-data.md](docs/03-test-data.md) | ช่องข้อมูลที่ต้อง export จากเอกสารตัวอย่างมาเติม |
| [docs/04-data-export-sql.md](docs/04-data-export-sql.md) | ABAP SQL ดึงเอกสารตัวอย่างจาก released CDS view |
| [docs/05-console-class.md](docs/05-console-class.md) | snapshot source code ของ console class |

## การแบ่งงาน push

- **ABAP object** (package, class) → ผู้ใช้สร้างใน ADT แล้ว push ผ่าน abapGit เอง
- **เอกสารทั้งหมดใน repo นี้** → Claude เป็นคน push

source of truth ของ ABAP object คือ tenant เสมอ
`docs/05-console-class.md` เป็นแค่ snapshot ไว้อ่าน ไม่ใช่ตัวจริง
