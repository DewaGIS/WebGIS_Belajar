# WebGIS Desa Srinanti — Script Day 3 (Measure, Search Bar, Deployment) + Eksperimen Multipage

---

### Struktur folder terbaru

```
webgis-srinanti/
├── index.html              <- halaman peta (Day 2 + Day 3)
├── style.css
├── script.js
├── index-beranda.html      <- isi homepage (dipakai di percobaan, bagian 8)
├── halaman.css             <- CSS navbar + homepage (percobaan)
├── beranda.js              <- statistik homepage (percobaan)
└── data/                   <- masukkan data di sini
    ├── landcover.tif
    ├── Data_Bangunan.geojson
    ├── Data_vegetasi.geojson
    ├── Data_LahanKosong.geojson
    └── Data_jalan.geojson
```

Cara menjalankan, dari **folder `webgis-srinanti/`**:

```bash
python -m http.server 8000
```

Lalu buka `http://localhost:8000`.

---

## 1. Langkah 1 — Memanggil CDN Plugin (Checkpoint 2, langkah "CDN LINK")

**📄 File: `index.html`** — di dalam `<head>`, sebelum `<link rel="stylesheet" href="style.css">`

```html
<!-- Plugin Day 3: Measure & Search (CSS) -->
<link
  rel="stylesheet"
  href="https://unpkg.com/leaflet-measure@3.1.0/dist/leaflet-measure.css"
/>
<link
  rel="stylesheet"
  href="https://unpkg.com/leaflet-search@4.0.0/dist/leaflet-search.min.css"
/>
```

**📄 File: `index.html`** — sebelum `</body>`, setelah `leaflet.js`, sebelum `script.js`

```html
<!-- Plugin Day 3: Measure & Search (JS) - harus setelah Leaflet JS -->
<script src="https://unpkg.com/leaflet-measure@3.1.0/dist/leaflet-measure.js"></script>
<script src="https://unpkg.com/leaflet-search@4.0.0/dist/leaflet-search.min.js"></script>
```

> Urutan penting: plugin harus dimuat **setelah** Leaflet dan **sebelum** `script.js`. Jika terbalik, muncul error `L.Control.Measure is not a constructor`. Versi plugin dikunci (`3.1.0` dan `4.0.0`) agar hasil demo stabil.

---

## 2. Langkah 2 — Measure Tool (Jarak & Luas)

**📄 File: `script.js`** — bagian 10 (akhir file)

```javascript
// Matikan auto-pan marker: penyebab peta melompat saat klik di mode ukur
L.Marker.mergeOptions({ autoPanOnFocus: false });

const measureControl = new L.Control.Measure({
  position: "topleft", // tambahan: agar tidak menumpuk dengan layer control
  primaryLengthUnit: "meters",
  secondaryLengthUnit: "kilometers",
  primaryAreaUnit: "sqmeters",
  secondaryAreaUnit: "hectares", // tambahan: sesuai slide, satuan hektar (ha)
  activeColor: "#6366f1",
  completedColor: "#10b981",
});

measureControl.addTo(map);

// Plugin menembakkan event ini saat pengukuran dimulai / selesai
map.on("measurestart", () => {
  sedangMengukur = true;
});
map.on("measurefinish", () => {
  sedangMengukur = false;
});
```

**📄 File: `script.js`** — bagian 4 (sudah diubah dari Day 2)

```javascript
let sedangMengukur = false; // diatur oleh plugin Measure (bagian 10)

map.on("click", function (e) {
  if (sedangMengukur) return; // pengaman: jangan tampilkan alert saat mengukur

  const lat = e.latlng.lat;
  const lng = e.latlng.lng;

  alert("Koordinat: " + lat + ", " + lng);
});
```

**Satuan yang didukung plugin:**

- Panjang: `meters`, `kilometers`, `feet`, `yards`, `miles`
- Luas: `sqmeters`, `hectares`, `acres`, `sqfeet`, `sqmiles`

**Cara memakai di browser:**

1. Klik ikon penggaris di kiri atas.
2. Pilih _Create a new measurement_.
3. Klik titik demi titik di peta. Dua titik menghasilkan **jarak**; tiga titik atau lebih menghasilkan **luas** dan keliling.
4. Klik _Finish measurement_ (atau klik dua kali pada titik terakhir) untuk menyelesaikan.

**Catatan:**

- Tampilan teks plugin berbahasa Inggris. Paket plugin ini tidak menyertakan terjemahan Bahasa Indonesia.
- Saya memeriksa kode plugin: saat mengukur, klik ditangkap oleh lapisan penangkap milik plugin. Jadi `alert` dari Day 2 kemungkinan memang tidak muncul saat mengukur. Flag `sedangMengukur` saya tambahkan sebagai **pengaman tambahan**. Saya belum mencoba perilaku ini di browser sungguhan, jadi tolong dicek saat latihan.

---

## 3. Langkah 3 — Search Bar

### 3.1 Masalah: atribut `nama_fasilitas` tidak ada di data

Slide memakai `propertyName: 'nama_fasilitas'`, tetapi data pelatihan tidak punya kolom itu (atributnya `Nama_Obj` dan `Keterangan`). Karena itu `nama_fasilitas` **dibuat oleh script** dari data yang ada, sehingga kode Search tetap persis seperti di slide.

| Layer        | Sumber `nama_fasilitas`                                              | Contoh                                                |
| ------------ | -------------------------------------------------------------------- | ----------------------------------------------------- |
| Bangunan     | `Keterangan`, hanya yang `Nama_Obj` bukan `Pemukiman` (47 fasilitas) | `Kopi Goy`, `Kantor Desa Srinanti`                    |
| Vegetasi     | `Keterangan`                                                         | `Kebun Kelapa Sawit Blok - 7`, `Kebun Tanaman Pangan` |
| Lahan kosong | `Nama_Obj`                                                           | `Lapangan`, `Semak Belukar`                           |

144 rumah warga (`Pemukiman`) sengaja tidak dimasukkan karena semuanya bernama sama. Jalan juga tidak dimasukkan (15 ruas bernama sama: `Jalan Desa`).

**Nama kembar:** plugin menyimpan hasil pencarian per nama, sehingga nama yang sama akan saling menimpa. Script memberi nomor: misalnya `Warung Sembako` dan `Warung Sembako (2)`, `Warung Makan` dan `Warung Makan (2)`, `Penginapan` sampai `Penginapan (5)`.

### 3.2 Kode

**📄 File: `script.js`** — bagian 7 (setiap fetch sekarang disimpan ke variabel promise)

```javascript
const pBangunan = fetch('data/Data_Bangunan.geojson')
    .then(res => res.json())
    .then(data => { ... })
    .catch(err => ...);
// pola sama untuk pVegetasi, pLahanKosong, pJalan
```

**📄 File: `script.js`** — bagian 11 (akhir file)

```javascript
/* 11.1 Buat atribut nama_fasilitas */
const namaTerpakai = {};

function buatNamaUnik(nama) {
  if (!namaTerpakai[nama]) {
    namaTerpakai[nama] = 1;
    return nama;
  }
  namaTerpakai[nama]++;
  return `${nama} (${namaTerpakai[nama]})`;
}

function untukSetiapFitur(group, callback) {
  group.eachLayer((layer) => {
    if (layer.feature) callback(layer);
    else if (layer.eachLayer) untukSetiapFitur(layer, callback);
  });
}

function isiNamaFasilitas(group, ambilNama) {
  untukSetiapFitur(group, (layer) => {
    const nama = ambilNama(layer.feature.properties);
    if (nama) layer.feature.properties.nama_fasilitas = buatNamaUnik(nama);
  });
}

/* 11.2 Pasang Search Control setelah semua GeoJSON selesai dimuat */
Promise.all([pBangunan, pVegetasi, pLahanKosong]).then(() => {
  isiNamaFasilitas(layerBangunan, (p) =>
    p.Nama_Obj !== "Pemukiman" ? p.Keterangan : null,
  );
  isiNamaFasilitas(layerVegetasi, (p) => p.Keterangan);
  isiNamaFasilitas(layerLahanKosong, (p) => p.Nama_Obj);

  const layerCari = L.layerGroup([
    layerBangunan,
    layerVegetasi,
    layerLahanKosong,
  ]);

  const searchControl = new L.Control.Search({
    layer: layerCari,
    propertyName: "nama_fasilitas",
    zoom: 17,
    initial: false,
    marker: false,
    textPlaceholder: "Cari nama fasilitas...",
  });

  map.addControl(searchControl);

  // Tambahan: buka popup fitur yang ditemukan
  searchControl.on("search:locationfound", function (e) {
    if (e.layer && e.layer.openPopup) e.layer.openPopup();
  });
});
```

**Cara memakai di browser:** klik ikon kaca pembesar di kiri atas, ketik teks, lalu pilih dari daftar.

**Contoh pencarian untuk demo:**

| Ketik      | Hasil                                                           |
| ---------- | --------------------------------------------------------------- |
| `kopi`     | `Kopi Goy`                                                      |
| `warung`   | 10 hasil (termasuk `Warung Makan (2)` dan `Warung Sembako (2)`) |
| `blok - 7` | `Kebun Kelapa Sawit Blok - 7`                                   |
| `lapangan` | `Lapangan` (lahan kosong)                                       |

---

## 4. Deployment: GitHub dan Vercel

### 4.1 Sebelum mulai

- Pastikan folder `data/` sudah berisi 5 file data. Jika tidak ikut di-push, situs publik tampil tanpa data.
- Hapus `data/PUT_DATA_HERE.txt` bila tidak diperlukan.
- Uji dulu semua poin Pre-flight Check (bagian 7).

### 4.2 GitHub (Step-by-Step)

1. Buat repository baru di **github.com** (mis. `webgis-app`). Tidak perlu mencentang README atau .gitignore bila ingin push pertama yang bersih.
2. Buka terminal **di dalam folder project** (`webgis-srinanti/`).
3. Jalankan perintah berikut:

```bash
# 1. Inisialisasi Repository Lokal
git init

# 2. Simpan Semua File Proyek
git add .
git commit -m "Tes Publish WebGIS"

# 3. Hubungkan & Push ke GitHub
git branch -M main
git remote add origin https://github.com/username/repo_name.git
git push -u origin main

# 4. Update perubahan
git add .
git commit -m "Jelaskan perubahan yang terjadi"
git push origin main
```

Hal yang perlu diperhatikan:

- Ganti `username` dan `repo_name` dengan akun dan nama repository sendiri. URL `git remote add origin ...` harus ditulis **dalam satu baris**. Pada slide terlihat terpotong ke dua baris karena lebar layar.
- Jika Git belum pernah dikonfigurasi di komputer, jalankan sekali (ganti dengan data sendiri) sebelum `git commit`:
  ```bash
  git config --global user.name "Nama Anda"
  git config --global user.email "email@contoh.com"
  ```
- Saat `git push`, GitHub meminta autentikasi. Kata sandi akun tidak lagi diterima untuk push lewat HTTPS; gunakan login lewat browser/credential manager Git atau Personal Access Token.
- Setiap pembaruan berikutnya cukup: `git add .`, `git commit -m "pesan"`, `git push`.

### 4.3 Vercel (Live URL)

1. Login ke **vercel.com** menggunakan akun GitHub.
2. Klik **Add New Project**, lalu **Import** repo GitHub.
3. Pilih repository `webgis-app` (atau nama repo yang dibuat).
4. Klik **Deploy**.

Hasil akhir: alamat publik berbentuk `https://nama-project.vercel.app`, dengan HTTPS dan deploy ulang otomatis setiap `git push`.

Catatan:

- Project ini situs statis (tanpa proses build), jadi umumnya tidak perlu pengaturan tambahan seperti pada slide. Tampilan antarmuka Vercel dapat berubah; jika ada pilihan _Framework Preset_, pilih **Other** dan biarkan _Root Directory_ kosong. Saya tidak bisa memeriksa antarmuka Vercel saat ini, jadi sesuaikan dengan tampilan yang muncul.
- Nama alamat `webgis-app.vercel.app` di slide hanya contoh; Vercel yang menentukan alamat sebenarnya (bisa ada tambahan kode jika nama sudah dipakai).
- Peta memuat tile dari server OpenStreetMap. Cocok untuk demo pelatihan; untuk situs dengan pengunjung banyak, perhatikan kebijakan penggunaan tile OpenStreetMap.
- Setelah deploy, ulangi Pre-flight Check pada alamat Vercel, terutama Console dan Network, karena file data kini diakses dari server publik.

---

## 5. Kode Lengkap Versi Akhir

### `index.html` (peta, sebelum percobaan bagian 8)

```html
<!DOCTYPE html>
<html lang="id">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>WebGIS Desa Srinanti</title>

    <!-- Leaflet CSS -->
    <link
      rel="stylesheet"
      href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"
    />

    <!-- Plugin Day 3: Measure & Search (CSS) -->
    <link
      rel="stylesheet"
      href="https://unpkg.com/leaflet-measure@3.1.0/dist/leaflet-measure.css"
    />
    <link
      rel="stylesheet"
      href="https://unpkg.com/leaflet-search@4.0.0/dist/leaflet-search.min.css"
    />

    <!-- Our CSS -->
    <link rel="stylesheet" href="style.css" />
  </head>
  <body>
    <h1>WebGIS Desa Srinanti</h1>

    <!-- Map Container -->
    <div id="map"></div>

    <!-- Leaflet JavaScript -->
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

    <!-- Pembaca GeoTIFF + layer raster untuk Leaflet (untuk landcover.tif) -->
    <script src="https://unpkg.com/georaster@1.6.0"></script>
    <script src="https://unpkg.com/georaster-layer-for-leaflet@4.1.2"></script>

    <!-- Plugin Day 3: Measure & Search (JS) - harus setelah Leaflet JS -->
    <script src="https://unpkg.com/leaflet-measure@3.1.0/dist/leaflet-measure.js"></script>
    <script src="https://unpkg.com/leaflet-search@4.0.0/dist/leaflet-search.min.js"></script>

    <!-- Our JavaScript -->
    <script src="script.js"></script>
  </body>
</html>
```

### `style.css`

```css
body {
  margin: 0;
  font-family: Arial;
}

h1 {
  margin: 10px 15px;
  font-size: 22px;
}

#map {
  height: 500px;
}

/* Legenda (dipakai di script.js, bagian Legenda) */
.legend {
  background: white;
  padding: 8px 10px;
  border-radius: 6px;
  box-shadow: 0 0 6px rgba(0, 0, 0, 0.3);
  font-size: 12px;
  line-height: 18px;
}

.legend b {
  display: block;
  margin: 4px 0 2px;
}

.legend i {
  display: inline-block;
  width: 14px;
  height: 14px;
  margin-right: 6px;
  vertical-align: middle;
  border: 1px solid #555;
}
```

### `script.js`

```javascript
console.log("WebGIS loaded!");

/* ============================================
   1. MAP ENGINE
   ============================================ */
const map = L.map("map", {
  center: [4.155, 117.2787], // pusat area data (lat, lng)
  zoom: 16,
  zoomControl: false,
  minZoom: 5,
  maxZoom: 19,
});

/* ============================================
   2. BASEMAP
   ============================================ */
const osm = L.tileLayer("https://tile.openstreetmap.org/{z}/{x}/{y}.png", {
  maxZoom: 19,
  attribution: "&copy; OpenStreetMap contributors",
}).addTo(map);

/* ============================================
   3. MAP CONTROLS
   ============================================ */
L.control
  .zoom({
    position: "topright",
  })
  .addTo(map);

L.control
  .scale({
    metric: true,
    imperial: false,
    position: "bottomleft",
  })
  .addTo(map);

/* ============================================
   4. EVENT LISTENER (KLIK)
   Klik pada fitur tidak memicu alert (bubblingMouseEvents: false),
   alert hanya muncul saat klik di area kosong peta.
   ============================================ */
let sedangMengukur = false; // diatur oleh plugin Measure (bagian 10)

map.on("click", function (e) {
  if (sedangMengukur) return; // pengaman: jangan tampilkan alert saat mengukur

  const lat = e.latlng.lat;
  const lng = e.latlng.lng;

  alert("Koordinat: " + lat + ", " + lng);
});

/* ============================================
   5. LANDCOVER (RASTER GeoTIFF)
   data/landcover.tif dibaca langsung di browser:
   fetch -> arrayBuffer -> parseGeoraster -> GeoRasterLayer.
   Projeksi UTM 50N (EPSG:32650) dibaca otomatis dari file TIF.
   Layer dibuat setelah file selesai diunduh & dibaca, lalu
   ditambahkan ke peta dan ke layer control (bagian 8).
   ============================================ */

// Warna per kelas piksel (kelas 0 = nodata -> transparan)
const kelasLandcover = {
  1: { nama: "Air", warna: "#419bdf" },
  2: { nama: "Pohon", warna: "#397d49" },
  4: { nama: "Vegetasi tergenang", warna: "#7a87c6" },
  5: { nama: "Tanaman pertanian", warna: "#e49635" },
  7: { nama: "Area terbangun", warna: "#c4281b" },
  8: { nama: "Lahan terbuka", warna: "#a59b8f" },
  10: { nama: "Awan", warna: "#616161" },
  11: { nama: "Rangeland", warna: "#e3e2c3" },
};

function warnaLandcover(values) {
  const kelas = kelasLandcover[values[0]];
  return kelas ? kelas.warna : null; // null = tidak digambar
}

let layerLandcover = null;

fetch("data/landcover.tif")
  .then((res) => res.arrayBuffer())
  .then((arrayBuffer) => parseGeoraster(arrayBuffer))
  .then((georaster) => {
    console.log("Landcover georaster:", georaster);

    layerLandcover = new GeoRasterLayer({
      georaster: georaster,
      opacity: 0.6,
      resolution: 256,
      pixelValuesToColorFn: warnaLandcover,
    });

    layerLandcover.addTo(map);
    layerLandcover.bringToBack(); // di bawah layer vektor
    layersControl.addOverlay(layerLandcover, "Landcover (raster 10 m)");
  })
  .catch((err) => console.error("Gagal load landcover.tif:", err));

/* ============================================
   6. STYLE & FUNGSI BANTU
   ============================================ */

/* 6.1 Bangunan: warna berdasarkan kategori (dari atribut Nama_Obj) */
const kategoriBangunan = {
  Pemukiman: "Hunian",
  Toko: "Perdagangan & Jasa",
  Warung: "Perdagangan & Jasa",
  "Warung Makan": "Perdagangan & Jasa",
  Cafe: "Perdagangan & Jasa",
  Pasar: "Perdagangan & Jasa",
  Bank: "Perdagangan & Jasa",
  Bengkel: "Perdagangan & Jasa",
  Penginapan: "Wisata & Penginapan",
  Atraksi: "Wisata & Penginapan",
  Sekolah: "Pendidikan",
  Masjid: "Ibadah",
  Musholla: "Ibadah",
  Gereja: "Ibadah",
  Puskesmas: "Kesehatan",
  "Kantor Desa": "Pemerintahan",
  "Kantor Kecamatan": "Pemerintahan",
  "Gedung Kecamatan": "Pemerintahan",
  "Kantor Polisi": "Pemerintahan",
  KopDes: "Pemerintahan",
  "Kantor BUMDes": "Pemerintahan",
};

const warnaKategori = {
  Hunian: "#9ca3af",
  "Perdagangan & Jasa": "#f59e0b",
  "Wisata & Penginapan": "#a855f7",
  Pendidikan: "#3b82f6",
  Ibadah: "#10b981",
  Kesehatan: "#ef4444",
  Pemerintahan: "#1e3a8a",
};

function getKategori(namaObj) {
  return kategoriBangunan[namaObj] || "Hunian";
}

function styleBangunan(feature) {
  const kategori = getKategori(feature.properties.Nama_Obj);
  return {
    fillColor: warnaKategori[kategori],
    fillOpacity: 0.85,
    color: "#333333",
    weight: 0.8,
    bubblingMouseEvents: false,
  };
}

/* 6.2 Vegetasi: data-driven (numeric threshold) berdasarkan luas (ha) */
function getColor(luas) {
  if (luas > 2) return "#166534"; // luas  > 2 ha
  if (luas > 1) return "#22c55e"; // 1 - 2 ha
  return "#bbf7d0"; // < 1 ha
}

function styleVegetasi(feature) {
  return {
    fillColor: getColor(feature.properties.area),
    fillOpacity: 0.7,
    color: "#14532d",
    weight: 1,
    bubblingMouseEvents: false,
  };
}

/* 6.3 Lahan kosong */
function styleLahanKosong(feature) {
  return {
    fillColor: "#d6b98c",
    fillOpacity: 0.7,
    color: "#8b6b3d",
    weight: 1.5,
    dashArray: "5, 4",
    bubblingMouseEvents: false,
  };
}

/* 6.4 Jalan */
function styleJalan(feature) {
  return {
    color: "#dc2626",
    weight: 4,
    opacity: 0.9,
    bubblingMouseEvents: false,
  };
}

/* ============================================
   7. LOAD GEOJSON (FETCH API + L.geoJSON)
   Semua layer dimasukkan ke variabel agar bisa diatur di layer control.
   ============================================ */
const layerBangunan = L.layerGroup();
const layerVegetasi = L.layerGroup();
const layerLahanKosong = L.layerGroup();
const layerJalan = L.layerGroup();

/* 7.1 Bangunan (MultiPolygon, 191 fitur) */
const pBangunan = fetch("data/Data_Bangunan.geojson")
  .then((res) => res.json())
  .then((data) => {
    L.geoJSON(data, {
      style: styleBangunan,
      onEachFeature: (feature, layer) => {
        const p = feature.properties;
        layer.bindPopup(
          `<b>${p.Nama_Obj}</b><br>` +
            `Kategori: ${getKategori(p.Nama_Obj)}<br>` +
            `Keterangan: ${p.Keterangan}<br>` +
            `Luas: ${p.area} ha`,
        );
        // Tooltip hanya untuk fasilitas (bukan rumah warga) agar peta tidak penuh
        if (p.Nama_Obj !== "Pemukiman") {
          layer.bindTooltip(p.Nama_Obj);
        }
      },
    }).addTo(layerBangunan);
  })
  .catch((err) => console.error("Gagal load Data_Bangunan.geojson:", err));

/* 7.2 Vegetasi (MultiPolygon, 26 fitur) */
const pVegetasi = fetch("data/Data_vegetasi.geojson")
  .then((res) => res.json())
  .then((data) => {
    L.geoJSON(data, {
      style: styleVegetasi,
      onEachFeature: (feature, layer) => {
        const p = feature.properties;
        layer.bindPopup(
          `<b>${p.Nama_Obj}</b><br>` +
            `${p.Keterangan}<br>` +
            `Luas: ${p.area} ha`,
        );
        layer.bindTooltip(p.Keterangan);
      },
    }).addTo(layerVegetasi);
  })
  .catch((err) => console.error("Gagal load Data_vegetasi.geojson:", err));

/* 7.3 Lahan Kosong (MultiPolygon, 3 fitur) */
const pLahanKosong = fetch("data/Data_LahanKosong.geojson")
  .then((res) => res.json())
  .then((data) => {
    L.geoJSON(data, {
      style: styleLahanKosong,
      onEachFeature: (feature, layer) => {
        const p = feature.properties;
        layer.bindPopup(
          `<b>${p.Nama_Obj}</b><br>` +
            `Kelas: ${p.Kelas_Obj}<br>` +
            `Luas: ${p.Area} ha`,
        );
        layer.bindTooltip(p.Nama_Obj);
      },
    }).addTo(layerLahanKosong);
  })
  .catch((err) => console.error("Gagal load Data_LahanKosong.geojson:", err));

/* 7.4 Jalan (MultiLineString, 15 fitur)
   Catatan: atribut "area" pada data jalan sebenarnya panjang jalan (km). */
const pJalan = fetch("data/Data_jalan.geojson")
  .then((res) => res.json())
  .then((data) => {
    L.geoJSON(data, {
      style: styleJalan,
      onEachFeature: (feature, layer) => {
        const p = feature.properties;
        layer.bindPopup(
          `<b>${p.Nama_Obj}</b><br>` +
            `${p.Keterangan}<br>` +
            `Panjang: ${p.area} km`,
        );
        layer.bindTooltip(p.Nama_Obj);
      },
    }).addTo(layerJalan);
  })
  .catch((err) => console.error("Gagal load Data_jalan.geojson:", err));

/* Urutan tampil: landcover (bawah, ditambahkan saat TIF selesai dibaca) -> lahan kosong -> vegetasi -> jalan -> bangunan (atas) */
layerLahanKosong.addTo(map);
layerVegetasi.addTo(map);
layerJalan.addTo(map);
layerBangunan.addTo(map);

/* ============================================
   8. LAYER CONTROL
   ============================================ */
const layersControl = L.control
  .layers(
    { OpenStreetMap: osm },
    {
      "Lahan Kosong": layerLahanKosong,
      Vegetasi: layerVegetasi,
      Jalan: layerJalan,
      Bangunan: layerBangunan,
    },
    { position: "topright", collapsed: window.innerWidth < 600 },
  )
  .addTo(map);

/* ============================================
   9. LEGENDA
   ============================================ */
const legend = L.control({ position: "bottomright" });

legend.onAdd = function () {
  const div = L.DomUtil.create("div", "legend");

  let html = "<b>Bangunan</b>";
  for (const kategori in warnaKategori) {
    html += `<i style="background:${warnaKategori[kategori]}"></i>${kategori}<br>`;
  }

  html +=
    "<b>Vegetasi (luas)</b>" +
    '<i style="background:#166534"></i>&gt; 2 ha<br>' +
    '<i style="background:#22c55e"></i>1 - 2 ha<br>' +
    '<i style="background:#bbf7d0"></i>&lt; 1 ha<br>';

  html +=
    "<b>Lainnya</b>" +
    '<i style="background:#d6b98c"></i>Lahan Kosong<br>' +
    '<i style="background:#dc2626"></i>Jalan<br>';

  html += "<b>Landcover</b>";
  [2, 7, 5, 11].forEach((k) => {
    html += `<i style="background:${kelasLandcover[k].warna}"></i>${kelasLandcover[k].nama}<br>`;
  });

  div.innerHTML = html;
  return div;
};

legend.addTo(map);

/* ============================================
   10. MEASURE (UKUR JARAK & LUAS)  - DAY 3
   Plugin: leaflet-measure (CSS & JS dimuat di index.html)
   ============================================ */
// Matikan auto-pan marker: penyebab peta melompat saat klik di mode ukur
L.Marker.mergeOptions({ autoPanOnFocus: false });

const measureControl = new L.Control.Measure({
  position: "topleft", // tambahan: agar tidak menumpuk dengan layer control
  primaryLengthUnit: "meters",
  secondaryLengthUnit: "kilometers",
  primaryAreaUnit: "sqmeters",
  secondaryAreaUnit: "hectares", // tambahan: sesuai slide, satuan hektar (ha)
  activeColor: "#6366f1",
  completedColor: "#10b981",
});

measureControl.addTo(map);

// Saat mengukur, geser dengan klik kiri dimatikan
map.on("measurestart", () => {
  sedangMengukur = true;
  map.dragging.disable();
});
map.on("measurefinish", () => {
  sedangMengukur = false;
  map.dragging.enable();
});

/* ============================================
   11. SEARCH BAR (CARI FASILITAS)  - DAY 3
   Plugin: leaflet-search (CSS & JS dimuat di index.html)
   Pencarian berdasarkan atribut `nama_fasilitas`.
   Atribut ini dibuat dari data (bagian 11.1) karena data pelatihan
   belum punya kolom `nama_fasilitas`.
   ============================================ */

/* 11.1 Buat atribut nama_fasilitas
   - Bangunan: hanya fasilitas (Nama_Obj bukan 'Pemukiman'), nama dari Keterangan
   - Vegetasi: nama dari Keterangan (mis. "Kebun Kelapa Sawit Blok - 1")
   - Lahan kosong: nama dari Nama_Obj
   Nama yang kembar diberi nomor, mis. "Warung Makan (2)", karena plugin
   menyimpan hasil pencarian per nama (nama kembar akan saling menimpa). */
const namaTerpakai = {};

function buatNamaUnik(nama) {
  if (!namaTerpakai[nama]) {
    namaTerpakai[nama] = 1;
    return nama;
  }
  namaTerpakai[nama]++;
  return `${nama} (${namaTerpakai[nama]})`;
}

// Jalankan callback untuk setiap fitur di dalam layer group (termasuk L.geoJSON di dalamnya)
function untukSetiapFitur(group, callback) {
  group.eachLayer((layer) => {
    if (layer.feature) callback(layer);
    else if (layer.eachLayer) untukSetiapFitur(layer, callback);
  });
}

function isiNamaFasilitas(group, ambilNama) {
  untukSetiapFitur(group, (layer) => {
    const nama = ambilNama(layer.feature.properties);
    if (nama) layer.feature.properties.nama_fasilitas = buatNamaUnik(nama);
  });
}

/* 11.2 Pasang Search Control setelah semua GeoJSON selesai dimuat */
Promise.all([pBangunan, pVegetasi, pLahanKosong]).then(() => {
  isiNamaFasilitas(layerBangunan, (p) =>
    p.Nama_Obj !== "Pemukiman" ? p.Keterangan : null,
  );
  isiNamaFasilitas(layerVegetasi, (p) => p.Keterangan);
  isiNamaFasilitas(layerLahanKosong, (p) => p.Nama_Obj);

  // Layer pencarian = gabungan layer yang punya nama_fasilitas
  const layerCari = L.layerGroup([
    layerBangunan,
    layerVegetasi,
    layerLahanKosong,
  ]);

  const searchControl = new L.Control.Search({
    layer: layerCari,
    propertyName: "nama_fasilitas",
    zoom: 17,
    initial: false, // cocokkan teks di mana saja, bukan hanya awal kata
    marker: false,
    textPlaceholder: "Cari nama fasilitas...",
  });

  map.addControl(searchControl);

  // Tambahan: buka popup fitur yang ditemukan
  searchControl.on("search:locationfound", function (e) {
    if (e.layer && e.layer.openPopup) e.layer.openPopup();
  });
});
```

### `index-beranda.html` (isi homepage; di-rename jadi `index.html` pada percobaan bagian 8)

```html
<!DOCTYPE html>
<html lang="id">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Beranda - WebGIS Desa Srinanti</title>

    <!-- CSS dasar project + CSS beranda/navbar -->
    <link rel="stylesheet" href="style.css" />
    <link rel="stylesheet" href="halaman.css" />
  </head>
  <body>
    <!-- Navbar -->
    <header class="navbar">
      <a class="brand" href="index.html">WebGIS Desa Srinanti</a>
      <nav>
        <a href="index.html" class="aktif">Beranda</a>
        <a href="peta.html">Peta</a>
      </nav>
    </header>

    <!-- Hero -->
    <section class="hero">
      <h1>WebGIS Desa Srinanti</h1>
      <p>
        Peta interaktif berisi bangunan, fasilitas umum, vegetasi, lahan kosong,
        jalan, dan tutupan lahan (landcover). Dibuat dalam Pelatihan WebGIS
        menggunakan HTML, CSS, JavaScript, dan Leaflet.
      </p>
      <a class="tombol" href="peta.html">Buka Peta Interaktif &rarr;</a>
    </section>

    <script src="beranda.js"></script>
  </body>
</html>
```

### `halaman.css`

```css
/* CSS tambahan untuk eksperimen beranda + peta (Day 3).
   Dimuat SETELAH style.css. */

* {
  box-sizing: border-box;
}

body {
  background: #f8fafc;
  color: #1f2937;
}

/* ---------- Navbar (dipakai di index.html dan peta.html) ---------- */
.navbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 56px;
  padding: 0 20px;
  background: #14532d;
  color: white;
}

.navbar .brand {
  color: white;
  font-weight: bold;
  font-size: 18px;
  text-decoration: none;
}

.navbar nav a {
  color: #d1fae5;
  text-decoration: none;
  margin-left: 18px;
  padding: 6px 2px;
  border-bottom: 2px solid transparent;
}

.navbar nav a:hover,
.navbar nav a.aktif {
  color: white;
  border-bottom-color: #86efac;
}

/* ---------- Halaman Peta ---------- */
body.halaman-peta #map {
  height: calc(100vh - 56px);
  height: calc(100dvh - 56px); /* browser mobile modern */
}

/* ---------- Halaman Beranda ---------- */
.hero {
  padding: 56px 20px;
  text-align: center;
  background: linear-gradient(135deg, #166534, #22c55e);
  color: white;
}

.hero h1 {
  margin: 0 0 10px;
  font-size: 34px;
}

.hero p {
  max-width: 640px;
  margin: 0 auto 24px;
  line-height: 1.6;
}

.tombol {
  display: inline-block;
  padding: 12px 26px;
  background: white;
  color: #14532d;
  font-weight: bold;
  text-decoration: none;
  border-radius: 999px;
}

.tombol:hover {
  background: #dcfce7;
}
```
