# Plan — Multi-part SVG (ala blush.design)

Status: **rencana**, belum diimplementasikan.
Branch: `feat/svg-multipart` (dibuat dari `master` setelah PR #17 merge).

---

## 1. Keputusan yang sudah disepakati

| # | Keputusan | Catatan |
|---|---|---|
| 1 | **Hanya menerima SVG** (Illustrator / CorelDRAW) untuk part | Ibis Paint & SAI di luar scope v1 — dikomunikasikan di UI |
| 2 | Kreator menentukan **karya tentang apa** (judul/deskripsi/kategori) lalu **daftar slot** yang dipakai | Slot tidak global, per karya |
| 3 | Setiap slot punya **flag `swappable`** → hanya itu yang bisa diganti pengunjung | Permintaan inti: "part yang memang diizinkan untuk diganti" |
| 4 | **Tanpa editor koordinat.** Semua part wajib punya `viewBox` identik | Kreator menyusun koordinat di Illustrator/Corel |
| 5 | Yang diatur kreator di web hanya **urutan tumpukan (z-index)** | Cukup integer, bukan drag di kanvas |
| 6 | Satu slot boleh punya **banyak varian**, satu ditandai default | Menggantikan `latestParts()` |

Belum diputuskan: ganti warna (palet), jumlah slot minimal, batas varian, sikap saat health check gagal. Lihat §13.

---

## 2. Kondisi kode saat ini (fakta)

| Bagian | Isi sekarang | Dampak |
|---|---|---|
| `PART_KATEGORI` | `["head","body","background"]` di `src/controllers/karya.controller.ts` | Harus jadi dinamis per karya |
| `karya_parts` | `id, karya_id, kategori, media_path, media_type, created_at` | Tambah `slot_id`, `view_box` |
| `composition.ts` | `latestParts()` (part terbaru per `kategori`), `partsByCategory()` | Ganti ke "default part per slot" |
| Posisi layer | **hardcoded** di 6 view: `w-2/5` kepala, `w-3/5` badan, `object-cover` background | Diganti `inset-0 w-full h-full` + `z-index` |
| Lokasi hardcode | `homepage/karya`, `homepage/index`, `partials/karya-card`, `studio/karya/show`, `studio/publications`, `admin/works/show` | Semua ikut berubah |
| `customizer.js` | `layers` map 3 kategori + export canvas hardcode posisi | Digeneralisasi baca `data-*` |
| Upload | 1 file per submit, 1 slot per submit, `saveImage()` (jpeg/png/gif/webp ≤20MB) | Tambah jalur SVG + multi-file |
| Validasi | `PART_KATEGORI.includes(kategori)` | Ganti: slot harus milik karya itu |
| Owner check | `isOwner(item, userId)` di semua route admin karya | **Pertahankan** |

Data lokal saat ini (untuk konteks pengujian, bukan bagian kode):

- 4 akun: 1 `superadmin`, 2 `admin`, 1 `member`
- 3 karya: `Makara Adventure`, `Campus Poster Series`, `Welcome Asset` (semua 2D)
- 6 part: 2 `background`, 2 `body`, 2 `head` — semua PNG

---

## 3. Aturan produk v1

1. Kreator membuat karya → menentukan **daftar slot** (nama + label).
2. Per slot: upload **beberapa varian SVG**, tandai satu sebagai **default**.
3. Per slot: centang **Swappable** (boleh diganti pengunjung).
4. Slot disusun urut (z-index) — determines tumpukan.
5. Semua SVG dalam satu karya **wajib** `viewBox` sama dengan karya.
6. Publik: tray hanya muncul untuk slot `swappable`; slot non-swappable selalu tampil dengan varian default.
7. Export: **SVG** (gabungan) dan **PNG** (canvas).

Contoh isi slot yang diharapkan:

```
Makara Character          viewBox: 0 0 400 500
├─ background   z=0   swappable ✅  2 varian
├─ legs         z=10  swappable ✅  2 varian
├─ body         z=20  swappable ❌  1 varian   ← badan utama, terkunci
├─ head         z=30  swappable ✅  3 varian
└─ hair         z=40  swappable ✅  2 varian (nested di atas kepala)
```

---

## 4. Model data & migrasi

### 4.1 Skema baru

```sql
-- Karya menyimpan kanvas acuan
ALTER TABLE karya ADD COLUMN view_box TEXT;           -- "0 0 400 500"
ALTER TABLE karya ADD COLUMN preview_path TEXT;       -- hasil komposit untuk kartu

-- Slot per karya (metadata ringan)
CREATE TABLE karya_slots (
  id TEXT PRIMARY KEY,
  karya_id TEXT NOT NULL,
  slot_key TEXT NOT NULL,            -- 'head' | 'hair' | 'kepala' (bebas, slug)
  label TEXT NOT NULL,               -- 'Head', 'Hair', 'Kepala'
  z_index INTEGER NOT NULL DEFAULT 0,
  swappable INTEGER NOT NULL DEFAULT 1,   -- 0/1
  default_part_id TEXT,              -- varian default (nullable → fallback terbaru)
  created_at TEXT DEFAULT (datetime('now'))
);
CREATE UNIQUE INDEX idx_slots_karya_key ON karya_slots(karya_id, slot_key);
CREATE INDEX idx_slots_karya_z ON karya_slots(karya_id, z_index);

-- Part sekarang menunjuk slot, bukan kategori bebas
ALTER TABLE karya_parts ADD COLUMN slot_id TEXT;
ALTER TABLE karya_parts ADD COLUMN view_box TEXT;
ALTER TABLE karya_parts ADD COLUMN svg_path TEXT;     -- null untuk part PNG lama
```

`karya_parts.kategori` **tidak dihapus** (dipakai untuk backfill & kompatibilitas), tapi setelah migrasi tidak lagi dibaca logika baru.

### 4.2 Backfill (idempoten, di `migrate()`)

Untuk setiap karya yang sudah punya part dengan `kategori` lama:

```
slot_key  = kategori lama ('head' | 'body' | 'background')
label     = Head | Body | Background
z_index   = 10 (body), 20 (background dipaksa 0), head 30
swappable = 1
default_part_id = part terbaru pada kategori itu (perilaku lama dipertahankan)
view_box  = "0 0 1000 1250" (rasio default, hanya agar tidak null)
```

Semua dalam satu transaksi; aman dijalankan berulang karena hanya menyentuh baris `slot_id IS NULL`.

### 4.3 Dampak ke query

| Fungsi | Sebelum | Sesudah |
|---|---|---|
| `latestParts(parts)` | terbaru per `kategori` | `defaultParts(parts, slots)` → `default_part_id`, fallback terbaru |
| `partsByCategory` | `Record<kategori, Part[]>` | `partsBySlot` → `Record<slot_key, Part[]>` terurut |
| `withComposition` | tambah `parts: {head,body,background}` | tambah `slots: [{key,z,swappable,parts,default}], parts: {slot_key: path}` |

Kartu grid cukup memakai `parts` (map) + `slots` untuk urutan.

---

## 5. Keamanan SVG (gerbang wajib — PR 1)

SVG adalah XML yang bisa mengeksekusi JavaScript. Tanpa langkah ini fitur **tidak boleh rilis**.

### 5.1 Sanitasi saat upload

Allowlist elemen & atribut, buang seluruhnya:

| Dibuang | Alasan |
|---|---|
| `<script>`, `<foreignObject>`, `<iframe>`, `<use href="http…">` | XSS |
| Semua atribut `on*` (`onload`, `onclick`, …) | XSS |
| `<style>` bebas, `@import`, `url(javascript:)` | Eksfiltrasi / CSS override |
| `xlink:href` / `href` ke luar (http/https/data) | SSRF / data URI |
| Komentar pihak ketiga yang menyisipkan markup | Vektor klasik |

Yang diizinkan: `svg, g, path, rect, circle, ellipse, line, polyline, polygon, text, tspan, defs, clipPath, mask, linearGradient, radialGradient, stop, use(#internal)` + atribut presentasi (`fill`, `stroke`, `stroke-width`, `opacity`, `transform`, `d`, `points`, `viewBox`, `id`, `class`).

Implementasi: parser XML eksplisit (mis. `fast-xml-parser` dengan allowlist kita sendiri) — **jangan** regex. Hasil normalisasi disimpan ulang, bukan file mentah.

### 5.2 Penyajian

- Serve `/uploads/*.svg` dengan `Content-Type: image/svg+xml`, `X-Content-Type-Options: nosniff`, `Content-Security-Policy: default-src 'none'; style-src 'unsafe-inline'` (lihat `src/app.ts`).
- v1 memakai `<img src>` (tidak execute script, tidak bisa di-recolor). Inline SVG **hanya** setelah sanitasi lolos + kalau fitur recolor dikerjakan.

### 5.3 Test keamanan (wajib ada)

Fixture jahat (tolak semua):
```
<script>alert(1)</script>
<svg onload="alert(1)">
<foreignObject><iframe src="//evil"></iframe></foreignObject>
<a xlink:href="javascript:alert(1)"><text>x</text></a>
<image href="https://evil/x.png"/>
<style>@import url(//evil/x.css)</style>
```

Fixture valid (terima, tidak berubah bentuk): path polos Illustrator, `#hex` fills, `clipPath`.

---

## 6. Validasi & health check

Dijalankan saat upload; gagal → flash error spesifik (bukan "upload gagal").

| Aturan | Pesan |
|---|---|
| Ekstensi `.svg` | "Only SVG files are accepted for parts." |
| XML valid + root `<svg>` | "This file is not a valid SVG." |
| `viewBox` ada | "SVG has no viewBox — export with artboard = viewBox." |
| `viewBox` == karya `view_box` | "All parts must share the same viewBox (0 0 W H)." |
| Tidak ada `<image>` raster tertanam | "Flattened/embedded images are not allowed — export vectors only." |
| Ukuran ≤ 2 MB | "SVG is too large (max 2MB)." |
| Jumlah slot ≤ 12 | "A work can have at most 12 slots." |
| Varian per slot ≤ 20 | "This slot is full (max 20 variants)." |
| Nama slot unik per karya | "A slot with this name already exists." |

Health check **ditolak** (bukan warning) — supaya tidak ada karya setengah jadi di publik. Kalau ternyata terlalu galak, gampang dilunakkan nanti.

---

## 7. Server: rute & controller

Prefix tetap `/studio` (admin) dan `/` (publik). Tetap form POST + PRG, **tanpa JSON API**.

### 7.1 Route baru

| Route | Fungsi |
|---|---|
| `POST /studio/karya/:id/slots` | tambah slot (label, swappable) |
| `POST /studio/karya/:id/slots/:sid/edit` | ubah label / swappable |
| `POST /studio/karya/:id/slots/:sid/delete` | hapus slot (+ part-nya, konfirmasi) |
| `POST /studio/karya/:id/slots/reorder` | simpan urutan z-index (hidden input array) |
| `POST /studio/karya/:id/slots/:sid/parts` | **upload multi-file** varian SVG |
| `POST /studio/karya/:id/slots/:sid/parts/:pid/default` | tandai varian default |
| `POST /studio/karya/:id/slots/:sid/parts/:pid/delete` | hapus varian |
| `POST /studio/karya/:id/viewbox` | set `view_box` karya (sebelum upload part pertama) |
| `GET /karya/:id?v=<slot>:<part>,…` | kombinasi dari URL (share) |

Semua route admin: `requireAuth` + **`isOwner`** + validasi bahwa `sid`/`pid` milik karya tersebut.

### 7.2 File yang disentuh

```
src/db.ts                          migrasi §4
src/models/karya.model.ts          slot CRUD, parts per slot, defaultPart, reorder
src/models/…  (moderation)         takedown part? (lihat §12)
src/controllers/karya.controller.ts  hapus PART_KATEGORI, route slot & upload SVG
src/composition.ts                 defaultParts / partsBySlot / withComposition
src/svg.ts  (baru)                 sanitizeSvg(), parseViewBox(), extractFills()
src/thumbnail.ts (baru)            komposit default → preview_path (resvg)
public/customizer.js               generik: baca slot dari DOM (data-*), export
views/studio/karya/show.eta.html   UI slot + upload + reorder + default
views/homepage/karya.eta.html      tray per slot swappable
views/partials/karya-card.eta.html tumpuk per slot + preview
views/homepage/index.eta.html, views/studio/publications.eta.html,
views/admin/works/show.eta.html    ikut skema baru
package.json                       dependency resvg (jika disetujui)
```

---

## 8. UI

### 8.1 Studio — kelola part (ganti halaman `studio/karya/show`)

```
┌─ Canvas ──────────────────┐  ┌─ Slots ─────────────────────────────┐
│  [live preview 4:5]       │  │ ⠿ Background   z0  swappable ✅     │
│                           │  │   ├ 2 varian · default ★            │
│  viewBox: 0 0 400 500     │  │   └ [upload multi-file]             │
│                           │  │ ⠿ Head         z30 swappable ✅     │
└───────────────────────────┘  │ ⠿ Body         z20 locked 🔒        │
                               └─────────────────────────────────────┘
```

- Form upload pakai `<input type="file" multiple accept=".svg">`
- Reorder: tombol ▲▼ (lebih aman daripada drag untuk aksesibilitas & tanpa JS berat)
- Preview live: klik varian → layer itu berganti (memakai pola `preview-pick` yang sudah ada)

### 8.2 Publik

- Tray hanya untuk slot `swappable`, dikelompokkan per slot (label tampil)
- Toggle mata per slot (visibilitas) tetap ada
- Tombol **Download SVG** (gabungkan `<svg>` jadi satu dokumen) & **Download PNG** (canvas)
- URL membawa pilihan varian → bisa dibagikan

---

## 9. Thumbnail kartu (keputusan dependency)

Masalah: grid 20 kartu × 5 SVG = 100+ request dan layout berat.

Solusi: saat publish (dan saat default berubah), server mengomposit → `karya.preview_path` (PNG/WebP). Alternatif teknis:

| Opsi | Plus | Minus |
|---|---|---|
| **`@resvg/resvg-js`** | Cepat, native, tanpa browser | Dependency biner (prebuilt tersedia) |
| `sharp` (librsvg) | Populer, sekaligus resize | Build native lebih berat |
| Render tanpa thumbnail | Nol dependency | Kartu berat/jelek |

Rekomendasi: `@resvg/resvg-js`. Fallback sementara jika ditolak: kartu merender maksimal 3 layer teratas + badge "+N".

---

## 10. Rencana PR bertahap

### PR 1 — Fondasi SVG + slot (harus satu paket)
- [ ] `src/svg.ts`: sanitasi allowlist + `parseViewBox()` + `extractFills()`
- [ ] `src/db.ts`: tabel `karya_slots`, kolom baru, backfill idempoten
- [ ] `karya.model.ts`: slot CRUD, part per slot, default part, reorder
- [ ] `karya.controller.ts`: route slot + upload multi-SVG + validasi §6
- [ ] `composition.ts`: `defaultParts` / `partsBySlot`
- [ ] `studio/karya/show.eta.html`: UI slot
- [ ] `test/svg.test.ts` + `test/web.test.ts` (upload, owner guard, viewBox mismatch)
- [ ] Sanitasi diuji dengan fixture jahat

### PR 2 — Customizer publik + export
- [ ] View publik & kartu pakai slot/z-index (6 file)
- [ ] `customizer.js` generik (data-*, tanpa hardcode 3 kategori)
- [ ] Export SVG gabungan + PNG canvas
- [ ] URL share kombinasi

### PR 3 — Thumbnail & performa
- [ ] `src/thumbnail.ts` + `@resvg/resvg-js`
- [ ] Generate saat publish / ubah default; fallback bila gagal
- [ ] Lazy loading + `width/height` untuk hindari layout shift

### PR 4 (opsional) — Palet warna ala Blush
- [ ] `extractFills()` → daftar warna unik per karya
- [ ] Color picker per warna; render ulang SVG
- [ ] **Butuh inline SVG** → hanya setelah sanitasi terbukti (test menyeluruh)

---

## 11. Test plan

| Level | Kasus |
|---|---|
| Unit (`svg.test.ts`) | sanitasi 6 payload jahat, SVG valid utuh, `parseViewBox`, `extractFills` |
| Unit (`karya.model.test.ts`) | slot CRUD, reorder mengubah tumpukan, default part, hapus slot menghapus part |
| Web | upload multi-file → N varian; viewBox beda → ditolak; slot milik karya lain → 404/403; member lain upload → ditolak (owner check) |
| Web | hapus karya → semua file slot terhapus; hapus slot → file terhapus |
| Web | publik: slot non-swappable tidak punya tray; slot swappable bisa diganti; kartu memakai default |
| Security | SVG `<script>` tidak tersimpan; `/uploads/*.svg` punya header nosniff + CSP |
| Regresi | karya lama (PNG, 3 kategori) tetap tampil setelah migrasi |

---

## 12. Risiko & mitigasi

| Risiko | Dampak | Mitigasi |
|---|---|---|
| SVG berisi script | XSS tersimpan | Sanitasi + header + test (§5) |
| Ekspor Corel rusak (tanpa viewBox, ada `<image>`) | Upload gagal terus | Health check dengan pesan jelas + panduan ekspor |
| Semua part beda `viewBox` | Komposisi berantakan | Ditolak sejak upload |
| Preview hilang (resvg gagal) | Kartu kosong | Fallback render 3 layer |
| Karya lama rusak setelah migrasi | Regresi | Backfill transaksional + test regresi |
| Kreator bingung alur baru | Adopsi rendah | Panduan "How to export from Illustrator" di studio |
| Part bermasalah tapi karya lolos | Moderasi tidak presisi | Tambah `karya_parts.status` (takedown per part) — bisa di PR 2 |

---

## 13. Pertanyaan yang masih terbuka

1. **Ganti warna** (extract `fill` → color picker): masuk PR 4 atau ditunda?
2. **Minimal slot** per karya untuk boleh publish (mis. ≥ 2 slot swappable)?
3. **Batas varian** per slot: 10 atau 20?
4. Jika health check gagal, **tolak** (rekomendasi) atau **terima dengan peringatan**?
5. Moderasi **per part** perlu sekarang atau nanti?
6. **Thumbnail** `@resvg/resvg-js` disetujui? (dependency baru)

---

## 14. Sample data untuk dicoba

Setelah PR 1 & 2 jalan, tersedia seed untuk mencoba alur lengkap.

### 14.1 Command

```bash
# di branch fitur
bun run seed:svg-demo "<username>"
```

`scripts/seed-svg-demo.ts` akan:
1. Mencari user berdasarkan username (harus sudah register)
2. Membuat karya **"Makara Character"** dengan `view_box = "0 0 400 500"`
3. Membuat 5 slot sesuai §3
4. Menulis 10 file SVG contoh ke `uploads/` lalu mengisi `karya_parts`
5. Mengomposit `preview_path`

### 14.2 Isi seed (bisa dicoba manual juga)

Semua contoh memakai kanvas yang sama: `viewBox="0 0 400 500"`.

**background (2 varian)**

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 400 500">
  <rect width="400" height="500" fill="#fdf3d7"/>
</svg>
```
```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 400 500">
  <rect width="400" height="500" fill="#cfe3ff"/>
  <circle cx="200" cy="180" r="140" fill="#ffffff" opacity="0.6"/>
</svg>
```

**legs (2 varian, `z=10`)**

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 400 500">
  <rect x="160" y="330" width="30" height="140" rx="12" fill="#1f2a44"/>
  <rect x="210" y="330" width="30" height="140" rx="12" fill="#1f2a44"/>
</svg>
```
```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 400 500">
  <rect x="150" y="330" width="34" height="140" rx="12" fill="#e8a100"/>
  <rect x="216" y="330" width="34" height="140" rx="12" fill="#e8a100"/>
</svg>
```

**body (1 varian, `z=20`, `swappable = 0`)**

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 400 500">
  <rect x="130" y="220" width="140" height="130" rx="24" fill="#2c3a5c"/>
</svg>
```

**head (3 varian, `z=30`)**

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 400 500">
  <circle cx="200" cy="150" r="70" fill="#ffd9b3"/>
  <circle cx="178" cy="140" r="8" fill="#1f2a44"/>
  <circle cx="222" cy="140" r="8" fill="#1f2a44"/>
</svg>
```
```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 400 500">
  <circle cx="200" cy="150" r="70" fill="#e8a100"/>
  <circle cx="178" cy="140" r="8" fill="#1f2a44"/>
  <circle cx="222" cy="140" r="8" fill="#1f2a44"/>
  <path d="M170 180 q30 20 60 0" stroke="#1f2a44" stroke-width="6" fill="none"/>
</svg>
```
```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 400 500">
  <rect x="130" y="80" width="140" height="140" rx="40" fill="#3a4a70"/>
</svg>
```

**hair (2 varian, `z=40`)**

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 400 500">
  <path d="M120 150 q80 -110 160 0 q-20 -40 -80 -40 q-60 0 -80 40z" fill="#1f2a44"/>
</svg>
```
```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 400 500">
  <path d="M120 150 q80 -110 160 0 q-20 -40 -80 -40 q-60 0 -80 40z" fill="#c98a00"/>
</svg>
```

### 14.3 Langkah mencoba

1. `bun run dev` + `bun run dev:css`
2. Login sebagai user pemilik seed (register dulu bila belum ada)
3. Jalankan `bun run seed:svg-demo <username>`
4. Buka `/studio/publications` → **Makara Character**
   - cek daftar slot, urutan, tanda ★ default, badge 🔒 pada `Body`
5. Buka `/karya/<id>` (publik):
   - tray hanya muncul untuk Background, Legs, Head, Hair
   - `Body` tidak bisa diganti (sesuai `swappable = 0`)
   - coba **Download SVG** dan **Download PNG**
6. Salin URL dengan kombinasi (`?v=head:2,hair:1`) → buka di tab lain, kombinasi harus sama

### 14.4 Kasus uji negatif (manual)

| Aksi | Harapan |
|---|---|
| Upload SVG `viewBox="0 0 100 100"` ke Makara | Ditolak: "All parts must share the same viewBox" |
| Upload PNG ke slot | Ditolak: "Only SVG files are accepted" |
| Upload SVG berisi `<script>` | Ditolak; tidak tersimpan di `uploads/` |
| User lain upload ke karya ini | Ditolak (owner check) |
| Hapus slot `head` | Konfirmasi, lalu 3 varian + filenya hilang |
| Hapus karya | Semua file SVG ikut terhapus |

---

## 15. Estimasi

| PR | Perkiraan |
|---|---|
| PR 1 (SVG + slot + migrasi + test keamanan) | besar — inti pekerjaan |
| PR 2 (customizer publik + export + 6 view) | sedang |
| PR 3 (thumbnail resvg) | kecil–sedang |
| PR 4 (palet warna) | sedang, opsional |

Rekomendasi: kerjakan **PR 1 dan 2 sebagai satu rangkaian sebelum dirilis**, karena PR 1 saja mengubah data tanpa UI pengganti yang lengkap.
