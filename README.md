# MAISON Residence — ระบบจองห้องพักสวัสดิการพนักงานแรกเข้า TQM

> **ชื่อระบบ:** สุขี-สุกายะ  
> **Stack:** Google Apps Script (GAS) + Vanilla HTML/JS + Google Sheets  
> **ไม่มี backend server** — GAS ทำหน้าที่เป็น REST API ผ่าน `doPost` Web App  
> **imgbb** สำหรับ photo upload (browser-side, ไม่ผ่าน GAS)

---

## โครงสร้างไฟล์

```
design_handoff_maison_residence/
├── login.html          ← หน้า login + public stats
├── dashboard.html      ← พนักงาน: รายการจองของตัวเอง
├── booking.html        ← 3-step form จองห้อง
├── detail.html         ← รายละเอียดการจอง + ยกเลิก
├── profile.html        ← แก้ไขโปรไฟล์ + ผูก LINE
├── admin.html          ← แผงควบคุม HR (role: admin เท่านั้น)
├── report-ico.gif      ← icon สรุปการจอง
├── image-slot.js       ← helper upload รูปภาพ
├── support.js          ← shared utilities
├── avatars/            ← รูปตัวอย่าง
└── gas/
    ├── config.gs               ← CONFIG object (IDs, ชื่อ sheet, ค่า setting)
    ├── auth.gs                 ← login / PIN / LINE auth
    ├── api.gs                  ← doPost router + ทุก function ของระบบ
    ├── setup_spreadsheets.gs   ← สร้าง sheet ครั้งแรก (รัน 1 ครั้ง)
    ├── migrate_rooms.gs        ← แปลง roomId เก่า R001→CTM-xxx (รัน 1 ครั้ง)
    ├── migrate_bookings.gs     ← migrate data เก่า
    └── fix_migrated_rows.gs    ← แก้ไข row ที่ migrate ผิด
```

---

## Google Spreadsheet

ระบบใช้ **2 ไฟล์** แยกกัน:

| ตัวแปร | ชื่อไฟล์ | ID |
|---|---|---|
| `FILE_A_ID` | MASTER_EMPLOYEES | `1XCHEaV9jnKgxIDFJaA_eHe9efrcY59CdXbNReh-WpBE` |
| `FILE_B_ID` | ROOMBOOKING | `1OBEwxF3oSgf6rKBjCjXmND1uhLt6-4ccx558Qd_fGfg` |

### File A — MASTER_EMPLOYEES

**Sheet: Employees** (คอลัมน์ A–G)

| Col | Header | ตัวอย่าง |
|---|---|---|
| A | empId | EMP001 |
| B | firstName | สมชาย |
| C | lastName | ใจดี |
| D | dept | ฝ่ายขาย |
| E | position | เจ้าหน้าที่ |
| F | startDate | 2024-01-15 |
| G | status | active / inactive |

**Sheet: Auth** (คอลัมน์ A–I)

| Col | Header | หมายเหตุ |
|---|---|---|
| A | empId | |
| B | pinHash | SHA-256 ของ PIN 6 หลัก |
| C | role | `user` / `admin` |
| D | lineUid | จาก LINE Login OAuth |
| E | lineDisplayName | |
| F | isFirstLogin | TRUE = ยังไม่เคย set PIN |
| G | lastLogin | ISO datetime |
| H | createdAt | |
| I | linePhotoUrl | URL รูปโปรไฟล์ LINE |

---

### File B — ROOMBOOKING

**Sheet: Rooms** (คอลัมน์ A–K)

| Col | Header | หมายเหตุ |
|---|---|---|
| A | roomId | CTM-102, SWP-205 (PREFIX-roomNo) |
| B | location | จุฑาพิชญ์ แมนชั่น / สุวนันท์ อพาร์ทเม้นท์ |
| C | building | ใช้เมื่อ location มีหลายตึก (ตอนนี้ว่าง) |
| D | floor | 1, 2, 3 |
| E | roomNo | 101, 205 |
| F | occupant1 | bookingId ที่ครอง slot 1 |
| G | occupant2 | bookingId ที่ครอง slot 2 |
| H | roomType | พัดลม / ห้องแอร์ |
| I | photo1 | URL imgbb |
| J | photo2 | URL imgbb |
| K | photo3 | URL imgbb |

**Room ID Format:** `PREFIX-roomNo` เช่น `CTM-102`, `SWP-301`

- จุฑาพิชญ์ แมนชั่น → `CTM`
- สุวนันท์ อพาร์ทเม้นท์ → `SWP`
- ที่อยู่ใหม่ → admin กำหนด prefix เองใน modal "เพิ่มห้องพัก" (2–5 ตัวอักษร A–Z)

---

**Sheet: Bookings** (คอลัมน์ A–W, 23 คอลัมน์)

| Col | Header | หมายเหตุ |
|---|---|---|
| A | bookingId | BK-YYYYMMDD-XXXX |
| B | empId | รหัสพนักงาน **ผู้เข้าพัก** |
| C | firstName | |
| D | lastName | |
| E | nickname | |
| F | dept | |
| G | gender | |
| H | religion | |
| I | idCard | เก็บเป็น text (รักษา 0 นำหน้า) |
| J | phone | เก็บเป็น text |
| K | startDate | วันเริ่มงาน |
| L | checkIn | วันที่ต้องการเข้าพัก |
| M | checkOut | วันที่ออก (admin กรอก) |
| N | nights | จำนวนคืน (admin กรอก) |
| O | roomType | ประเภทห้องที่ขอ |
| P | purpose | ความประสงค์ |
| Q | status | pending / approved / active / checked_out / cancelled / rejected |
| R | assignedRoom | roomId ที่จัดให้ |
| S | roommate | รหัสหรือชื่อ roommate |
| T | photoUrl | URL รูปถ่าย (imgbb) |
| U | bookedBy | empId ผู้จองแทน (ถ้าจองเอง = empId เดียวกัน) |
| V | bookedByName | ชื่อผู้จองแทน |
| W | createdAt | ISO datetime |

**Sheet: Bookings_Archive** — โครงสร้างเหมือน Bookings ทุกอย่าง (ย้ายรายการเก่า > 3 เดือน)

---

## Booking Status Flow

```
pending ──(admin จัดห้อง)──► approved ──(admin check-in)──► active ──(admin check-out)──► checked_out
   │
   ├──(admin ปฏิเสธ)──► rejected
   │
   └──(user/admin ยกเลิก)──► cancelled
```

---

## GAS — config.gs

ค่าหลักทั้งหมดอยู่ใน `CONFIG` object:

```js
const CONFIG = {
  FILE_A_ID: "...",          // MASTER_EMPLOYEES Spreadsheet ID
  FILE_B_ID: "...",          // ROOMBOOKING Spreadsheet ID
  SHEET_EMPLOYEES: "Employees",
  SHEET_AUTH: "Auth",
  SHEET_BOOKINGS: "Bookings",
  SHEET_BOOKINGS_ARCHIVE: "Bookings_Archive",
  SHEET_ROOMS: "Rooms",
  ARCHIVE_MONTHS: 3,
  LINE_CHANNEL_ID: "",       // ใส่หลัง setup LINE Login
  LINE_CHANNEL_SECRET: "",
  ADMIN_EMAIL: "na.adisak@gmail.com",
  SESSION_EXPIRE_HOURS: 8,
};
```

---

## GAS — api.gs (ทุก Action)

### ไม่ต้อง auth
| action | params | หมายเหตุ |
|---|---|---|
| `checkEmployee` | `empId` | ตรวจว่ามีในระบบ + สถานะ PIN |
| `setPin` | `empId, pin` | ตั้ง PIN ครั้งแรก |
| `verifyPin` | `empId, pin` | login ด้วย PIN |
| `verifyLine` | `lineUid` | login ด้วย LINE |
| `getPublicStats` | — | สถิติสาธารณะสำหรับ login page |

### ต้อง `empId` (user + admin)
| action | params |
|---|---|
| `linkLine` | `empId, lineUid, lineDisplayName` |
| `unlinkLine` | `empId` |
| `getDepts` | `empId` |
| `getMyBookings` | `empId` |
| `createBooking` | `empId, data` |
| `getBookingById` | `empId, bookingId, role` |
| `cancelBooking` | `empId, bookingId` |
| `getRooms` | `empId` |
| `getMyProfile` | `empId` |
| `updateProfile` | `empId, data` |

### ต้อง role = `admin`
| action | params |
|---|---|
| `getAllBookings` | `empId, role` |
| `assignRoom` | `empId, role, bookingId, roomId, slot` |
| `unassignRoom` | `empId, role, bookingId, roomId, slot` |
| `checkIn` | `empId, role, bookingId` |
| `checkOut` | `empId, role, bookingId` |
| `rejectBooking` | `empId, role, bookingId` |
| `addRoom` | `empId, role, data` |
| `updateRoom` | `empId, role, data` |
| `deleteRoom` | `empId, role, roomId` |
| `getEmployees` | `empId, role` |
| `getUsers` | `empId, role` |
| `addEmployee` | `empId, role, data` |
| `editEmployee` | `empId, role, data` |
| `deleteEmployee` | `empId, role, targetEmpId` |
| `resetPin` | `empId, role, targetEmpId` |
| `updateRole` | `empId, role, targetEmpId, newRole` |
| `getArchiveStats` | `empId, role` |
| `archiveBookings` | `empId, role` |
| `getArchivedBookings` | `empId, role` |

---

## Frontend — Session Storage

Login สำเร็จจะบันทึกลง `sessionStorage`:

```js
sessionStorage.setItem("empId", "EMP001");
sessionStorage.setItem("name",  "สมชาย ใจดี");
sessionStorage.setItem("dept",  "ฝ่ายขาย");
sessionStorage.setItem("role",  "user"); // หรือ "admin"
```

ทุก API call ส่ง `empId` + `role` ไปด้วยเสมอ:

```js
async function api(action, extra = {}) {
  const res = await fetch(GAS_URL, {
    method: "POST",
    body: JSON.stringify({
      action,
      empId: sessionStorage.getItem("empId"),
      role:  sessionStorage.getItem("role"),
      ...extra
    })
  });
  return res.json();
}
```

> **หมายเหตุ:** ไม่ได้ใช้ Content-Type header เพื่อ bypass CORS preflight บน GAS

---

## Photo Upload — imgbb

อัพโหลดรูปตรงจาก browser → imgbb → บันทึก URL ผ่าน GAS

```
API Key: 634f349492429106ef296d1d0bd1414b
Endpoint: https://api.imgbb.com/1/upload
```

```js
const form = new FormData();
form.append("key", IMGBB_KEY);
form.append("image", base64string);
const res = await fetch("https://api.imgbb.com/1/upload", { method: "POST", body: form });
const url = res.data.url; // บันทึก url นี้ลง sheet
```

---

## Login Flow

```
เปิด login.html
    │
    ▼
กรอก empId → checkEmployee
    │
    ├── isFirstLogin = true → กรอก PIN ใหม่ → setPin → เข้าระบบ
    │
    └── isFirstLogin = false
            │
            ├── กรอก PIN → verifyPin → เข้าระบบ
            │
            └── (ถ้าผูก LINE แล้ว) → LINE Login → verifyLine → เข้าระบบ
```

หลัง login สำเร็จ redirect ตาม role:
- `user` → `dashboard.html`
- `admin` → `admin.html`

---

## Admin Panel — Tabs

| Tab | เนื้อหา |
|---|---|
| **การจอง** | รายการจองทั้งหมด + filter + load more (20/page) |
| **จัดห้อง** | รายการ pending รอจัดห้อง |
| **ห้องพัก** | รายการห้อง (3 คอลัมน์ default, toggle 2/3) + เพิ่ม/แก้ไข + 3 รูป + load more 20/page |
| **ผู้ใช้งาน** | รายการพนักงาน + pagination 20/หน้า + line chart สรุปการจอง |
| **Archive** | รายการจองเก่า + สถิติ |

### Line Chart (Users tab)
- SVG-based, ไม่ใช้ library ภายนอก
- Toggle **รายวัน / รายเดือน**
- เลือก **ปี** และ **เดือน** (กรณี mode รายวัน)
- Toggle แสดง/ซ่อน **แต่ละ status**
- Colors: `pending=#D4A03B`, `approved=#3B9A5F`, `active=#2E82D4`, `checked_out=#8A8A9A`, `cancelled=#B0B0C0`, `rejected=#C84040`
- Icon: `report-ico.gif`

---

## Stat Cards (Admin Dashboard)

| Element ID | แสดง |
|---|---|
| `s-total` | รายการจองทั้งหมด |
| `s-pending` | รออนุมัติ |
| `s-approved` | จัดห้องแล้ว (approved) |
| `s-active` | กำลังพักอยู่ (active) |
| `s-checkout` | เช็คเอาต์แล้ว (checked_out) |
| `s-rooms` | ห้องพักทั้งหมด |

---

## Topbar Avatar

ทั้ง `admin.html` และ `dashboard.html` โหลด photo async หลัง page load:

```js
(async () => {
  try {
    const p = await api("getMyProfile");
    if (p.ok && p.photoUrl) {
      const av = document.getElementById("user-avatar");
      av.textContent = "";
      av.innerHTML = `<img src="${p.photoUrl}" style="width:100%;height:100%;object-fit:cover;border-radius:50%">`;
    }
  } catch(_) {}
})();
```

> photoUrl ไม่ได้เก็บใน sessionStorage — ต้อง fetch ทุกครั้งที่โหลดหน้า

---

## Login Page — Public Stats

```js
// getPublicStats ไม่ต้อง auth — เรียกตอนโหลดหน้า login
const res = await api("getPublicStats");
// res.rooms.avail      → ห้องว่าง
// res.rooms.total      → ห้องทั้งหมด
// res.bookings.active  → กำลังพักอยู่
// res.bookings.pending → รออนุมัติ
```

IDs ใน HTML: `hs-rooms`, `hs-total`, `hs-active`, `hs-pending`

---

## loadAll() — ลำดับการโหลด (สำคัญ)

```js
async function loadAll() {
  await loadBookings();                          // ต้องเสร็จก่อน!
  await Promise.all([loadRooms(), loadUsers()]);
  renderUsers();   // booking count บน user card จึงถูกต้อง
  loadArchiveStats();
  lcInit();        // init line chart
}
```

> **ห้าม** `Promise.all([loadBookings(), loadRooms(), loadUsers()])` พร้อมกัน  
> race condition ทำให้ `allBookings = []` เมื่อ `renderUsers()` รัน → booking count = 0

---

## Deployment

### ครั้งแรก (Initial Setup)

1. เปิด [script.google.com](https://script.google.com) → สร้างโปรเจกต์ใหม่
2. วางไฟล์ทั้งหมดใน `gas/` (แต่ละไฟล์เป็น tab แยก)
3. แก้ไข `FILE_A_ID` และ `FILE_B_ID` ใน `config.gs`
4. รัน `setup_spreadsheets()` → สร้าง sheet + header ทั้งหมด
5. **Deploy → New deployment**
   - Type: **Web App**
   - Execute as: **Me**
   - Who has access: **Anyone**
6. คัดลอก Deployment URL → วางแทน `GAS_URL` ใน HTML ทุกไฟล์

### Re-deploy หลัง code เปลี่ยน

1. GAS Editor → **Deploy → Manage deployments**
2. เลือก deployment → **Edit (ดินสอ)** → Version: **New version** → Deploy
3. URL เดิมยังใช้ได้ ไม่ต้องเปลี่ยนใน HTML

### Migration Rooms (รันครั้งเดียว)

```
1. รัน checkDuplicateRooms() → ตรวจ roomId ซ้ำ
2. แก้ไขข้อมูลซ้ำด้วยตนเองใน Sheet ก่อน
3. รัน migrateRooms() เพื่อ:
   - เพิ่ม header photo1/2/3 (col I/J/K)
   - แปลง R001→CTM-xxx, R007→SWP-xxx ฯลฯ
   - อัพเดท assignedRoom ใน Bookings sheet
```

---

## Room Prefix System

| Location | Prefix อัตโนมัติ |
|---|---|
| สุวนันท์ อพาร์ทเม้นท์ (หรือ สวนันท์) | SWP |
| จุฑาพิชญ์ แมนชั่น | CTM |
| อื่นๆ | 3 ตัวอักษรแรก English / กำหนดเองใน modal |

Frontend modal "เพิ่มห้อง":
1. ดู prefix จากห้องที่มีอยู่แล้วใน location เดียวกัน
2. ถ้าไม่มี → ใช้ known mapping
3. ถ้าไม่ตรง → ใช้ initials จากชื่อ English

GAS `addRoom()` รับ `data.prefix` จาก frontend (fallback ถึง `_getRoomPrefix()` ถ้าไม่ส่งมา)

---

## Known Issues / Pending

- [ ] **Redeploy GAS** หลังเพิ่ม `getPublicStats`, `updateRoom`, `addRoom` (prefix), `getRooms` (photos)
- [ ] **รัน `migrateRooms()`** หลังแก้ข้อมูลซ้ำ R001/R002
- [ ] **LINE Login** — ยังไม่ได้ใส่ `LINE_CHANNEL_ID` / `LINE_CHANNEL_SECRET` ใน config.gs
- [ ] `building` field ใน Rooms ว่างเปล่า — รองรับกรณี location มีหลายตึกในอนาคต
- [ ] Auth เป็น empId+role trust-based — production จริงควรเปลี่ยนเป็น signed token

---

## Design Tokens (CSS Variables)

```css
--bg:           #EDE7DA   /* พื้นหลังหลัก */
--surface:      #FFFDF8   /* card / panel */
--ink:          #23201B   /* ข้อความหลัก */
--ink-soft:     #4A453C
--muted:        #857D6E
--pine:         #1E3A34   /* สีเขียวเข้ม primary */
--pine-deep:    #16302A
--brass:        #A8834F   /* สีทอง accent */
--brass-soft:   #C9A876
--line-color:   #E7DFCE   /* เส้นขอบ */
```

Font: **Kodchasan** (Google Fonts) — weights 300/400/500/600/700

---

*อัพเดทล่าสุด: 2026-07-11*
