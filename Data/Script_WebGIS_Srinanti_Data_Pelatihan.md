# WebGIS Desa Srinanti — Script untuk Data Pelatihan

Versi script Day 2 yang disesuaikan dengan data pelatihan. `landcover.tif` **dibaca langsung** di browser (tanpa konversi ke PNG). Datanya tidak disertakan di project; kamu tinggal memasukkannya ke folder `data/`. Isi tiga file kode di bawah sama persis dengan yang ada di folder `webgis-srinanti/`.

## 1. Struktur Folder

```
webgis-srinanti/
├── index.html
├── style.css
├── script.js
└── data/                          <- masukkan data di sini
    ├── landcover.tif
    ├── Data_Bangunan.geojson
    ├── Data_vegetasi.geojson
    ├── Data_LahanKosong.geojson
    └── Data_jalan.geojson
```

Nama file di `data/` harus persis seperti di atas (huruf besar/kecil berpengaruh), karena script memanggilnya lewat `fetch('data/...')`.

**Cara menjalankan** (di dalam folder `webgis-srinanti/`):

```bash
python -m http.server 8000
```

lalu buka `http://localhost:8000`. Atau pakai VS Code Live Server. Tidak bisa dibuka lewat klik dua kali karena memakai `fetch()`. Komputer juga perlu terhubung internet untuk memuat Leaflet, library GeoTIFF, dan tile OSM dari CDN.

## 2. Ringkasan Data

| File                       | Geometri        | Jumlah         | Atribut                                     | Catatan                                                                               |
| -------------------------- | --------------- | -------------- | ------------------------------------------- | ------------------------------------------------------------------------------------- |
| `Data_Bangunan.geojson`    | MultiPolygon    | 191            | `id, area, Nama_Obj, Keterangan, Kelas_Obj` | 21 jenis `Nama_Obj` (144 Pemukiman, sisanya fasilitas). `area` = luas (ha)            |
| `Data_vegetasi.geojson`    | MultiPolygon    | 26             | `id, Nama_Obj, Kelas_Obj, Keterangan, area` | 25 Kelapa Sawit (Blok 1-25) + 1 KWT Sari Mukti. `area` = luas (ha), rentang 0,38-4,07 |
| `Data_LahanKosong.geojson` | MultiPolygon    | 3              | `id, Nama_Obj, Kelas_Obj, Keterangan, Area` | Lapangan, Semak Belukar, Lahan Kosong. Perhatikan **`Area` huruf kapital A**          |
| `Data_jalan.geojson`       | MultiLineString | 15             | `id, Nama_Obj, Kelas_Obj, Keterangan, area` | Atribut `area` di sini sebenarnya **panjang jalan (km)**, bukan luas                  |
| `landcover.tif`            | Raster          | 6448 x 4262 px | 1 band uint8, nodata = 0                    | UTM 50N (EPSG:32650), piksel 10 m, terkompresi LZW                                    |

Semua GeoJSON memakai CRS84 (urutan `[lon, lat]`), jadi langsung cocok dengan `L.geoJSON()`. Area data ada di sekitar lat 4,155 / lon 117,279, sehingga pusat peta tidak lagi `-6.9175, 107.6191` seperti di slide.

## 3. Penyesuaian dari Slide

| Slide                                          | Di versi ini                                                                                        |
| ---------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| Center `[-6.9175, 107.6191]`, zoom 13          | Center `[4.1550, 117.2787]`, zoom 16 (area data hanya sekitar 1 km)                                 |
| `maxZoom: 18`                                  | `maxZoom: 19` (tile OSM tersedia sampai zoom 19)                                                    |
| Leaflet dari `unpkg.com/leaflet/...`           | Dikunci ke versi `1.9.4` agar hasil demo stabil                                                     |
| Data titik + `L.circleMarker` (`pointToLayer`) | Data ini poligon dan garis, jadi memakai opsi **`style`**, bukan `pointToLayer`                     |
| Styling per kategori (`category === 'Health'`) | Warna per kategori dari atribut `Nama_Obj` (lookup `kategoriBangunan`)                              |
| Data-driven styling pada `pop`                 | Dipakai pada `area` (luas, ha) data vegetasi: > 2 ha, 1-2 ha, < 1 ha                                |
| `map.on('click')` + alert                      | Tetap alert. Ditambah `bubblingMouseEvents: false` pada style agar klik di fitur tidak memicu alert |
| (tidak ada)                                    | Tambahan: raster GeoTIFF, `L.control.layers` (nyala/matikan layer), dan legenda                     |

## 4. Tentang landcover.tif (dibaca langsung)

Leaflet sendiri tidak bisa membaca GeoTIFF, jadi `index.html` menambah dua library dari CDN:

```html
<script src="https://unpkg.com/georaster@1.6.0"></script>
<script src="https://unpkg.com/georaster-layer-for-leaflet@4.1.2"></script>
```

Alurnya di `script.js` (bagian 5): `fetch('data/landcover.tif')` → `arrayBuffer()` → `parseGeoraster()` → `new GeoRasterLayer({...})` → `addTo(map)`. Pola ini mirip `fetch` + `L.geoJSON` di slide, hanya parser dan layer-nya berbeda.

- **Proyeksi:** file ini UTM 50N, bukan WGS84. Library membacanya dari header TIF dan mereproyeksi sendiri, jadi tidak perlu `proj4` terpisah.
- **Warna kelas:** piksel berisi kode kelas (1, 2, 4, 5, 7, 8, 10, 11), diwarnai lewat tabel `kelasLandcover` dan fungsi `pixelValuesToColorFn`. Nilai 0 (nodata) otomatis transparan. Kelas yang muncul di file ini: Air, Pohon, Vegetasi tergenang, Tanaman pertanian, Area terbangun, Rangeland.
- **Sumber landcover:** kode kelas dan warnanya cocok dengan skema 10 m land cover Esri/Impact Observatory (1 Air, 2 Pohon, 4 Vegetasi tergenang, 5 Tanaman pertanian, 7 Area terbangun, 8 Lahan terbuka, 10 Awan, 11 Rangeland). Itu dugaan saya dari nilai kelasnya, jadi konfirmasi dulu sumbernya sebelum dipresentasikan.
- **Waktu muat:** file TIF seluruhnya (±770 KB, 27 juta piksel, cakupan sekitar 64 x 42 km) diunduh dan didekompresi di browser. Di komputer saya, proses pembacaannya sekitar 1 detik (Node.js). Di browser peserta bisa beberapa detik, dan layer landcover baru muncul setelah itu. Layer lain tampil duluan.
- **Cakupan luas:** bounds raster jauh lebih besar dari desa. Jika peserta zoom keluar, landcover terlihat meluas ke sekitar desa. Itu normal.

**Status pengujian:** saya memastikan library membaca file TIF kamu dengan benar (proyeksi 32650, nodata 0, kelas piksel, dan bounds hasil reproyeksi mencakup area desa). Namun saya belum bisa menjalankannya di browser sungguhan, jadi **tampilan warna di peta belum saya lihat sendiri**. Tolong coba dulu di komputer trainer sebelum sesi, dan cek Console (F12) bila landcover tidak muncul.

## 5. Kode

### `index.html`

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

    <!-- Our JavaScript -->
    <script src="script.js"></script>
  </body>
</html>
```

### `style.css`

```css
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
map.on("click", function (e) {
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
  };
}

/* 6.4 Jalan */
function styleJalan(feature) {
  return {
    color: "#dc2626",
    weight: 4,
    opacity: 0.9,
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
fetch("data/Data_Bangunan.geojson")
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
fetch("data/Data_vegetasi.geojson")
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
fetch("data/Data_LahanKosong.geojson")
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
fetch("data/Data_jalan.geojson")
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
    { position: "topright", collapsed: false },
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
```

## 6. Catatan untuk Trainer

- **Bangunan banyak (191 poligon):** tooltip hanya dipasang pada fasilitas, bukan rumah warga (`Pemukiman`), supaya peta tidak penuh label.
- **Urutan layer:** landcover paling bawah, lalu lahan kosong, vegetasi, jalan, dan bangunan paling atas. Vektor ditentukan oleh urutan `.addTo(map)`, landcover diletakkan di belakang dengan `bringToBack()`.
- **Atribut `Area` vs `area`:** data lahan kosong memakai `p.Area`, sedangkan bangunan, vegetasi, dan jalan memakai `p.area`. JavaScript membedakan huruf besar dan kecil, jadi kalau tertukar nilainya muncul `undefined`.
- **Jika layer tidak muncul:** buka Console (F12). Pesan `Gagal load ...` biasanya berarti server lokal belum jalan, path/nama file salah, atau tidak ada internet untuk memuat library dari CDN.
