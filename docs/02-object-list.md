# 02 — Object List

สถานะ ณ วันที่อัปเดตล่าสุด · Claude อัปเดตตารางนี้ตามที่เห็นใน `git log`
หลังผู้ใช้ push object ผ่าน abapGit

| # | Object | Type | ใครสร้าง | Status |
|---|---|---|---|---|
| 1 | `YPOC_PAYMENT` | Package | ผู้ใช้ (ADT) | ⬜ |
| 2 | `YCL_PAYMENT` | Class (console, `IF_OO_ADT_CLASSRUN`) | ผู้ใช้ (ADT) | ⬜ |

Legend: ⬜ ยังไม่สร้าง · 🟡 สร้างแล้วยังไม่ push · ✅ push ขึ้น repo แล้ว

ชื่อ confirm แล้ว 2026-09-11

## Config ที่ไม่ใช่ repository object

ไม่มี — `I_JournalEntryTP` เรียกในเครื่อง ไม่ต้องมี communication scenario / arrangement

## ลำดับการทำ

```
1. Object 1          สร้าง package บน ADT → link abapGit → push ให้ SAP serialize baseline
2. export data       รัน SQL ใน docs/04 กับเอกสารตัวอย่าง → ส่งผลกลับมา
3. Object 2          copy console class จาก chat → รัน F9 แบบ simulate
4. เทียบ payload      กับเอกสารตัวอย่าง จนตรงทุกบรรทัด
5. post จริง          gc_simulate = abap_false → เทียบเอกสารที่ได้กับตัวอย่าง
```

## Log การทดสอบ

| วันที่ | ทำอะไร | ผล |
|---|---|---|
| 2026-09-11 | เปิด project · confirm scope (post only, no clearing) + ชื่อ object | ✅ |

### ยังพิสูจน์ไม่ได้ (รออะไรอยู่)

| หัวข้อ | รอ |
|---|---|
| ชื่อ field จริงของ `%param` / `_GLItems` / `_APARItems` | เช็ค code completion ใน ADT |
| `Validate` มีบน tenant หรือไม่ | เช็คใน ADT |
| เอกสารตัวอย่างมีบรรทัดอะไรบ้าง | รอผล export จาก docs/04 |
| post จริงได้เลขเอกสาร | รอขั้น 5 |

## งานที่ยังค้าง

| # | เรื่อง | สถานะ |
|---|---|---|
| 1 | export เอกสารตัวอย่าง | ⬜ |
| 2 | เขียน `YCL_PAYMENT` | ⬜ รอข้อ 1 |
