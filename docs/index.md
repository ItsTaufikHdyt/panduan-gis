# Selamat Datang di Pusat Dokumentasi GIS & Geoportal

Website ini merupakan pusat panduan teknis, dokumentasi sistem, dan *knowledge base* pengelolaan **Geoportal** serta infrastruktur Sistem Informasi Geografis (GIS).

---

## 🚀 Ringkasan Sistem

Sistem Geoportal ini dibangun di atas arsitektur containerized berbasis **Docker**, mengintegrasikan beberapa layanan utama:

* **GeoNode / Django:** Manajemen dataset spasial, metadata, hak akses, dan antarmuka web.
* **GeoServer:** Layanan pemetaan (*Map Server*) untuk menyajikan WMS, WFS, WMTS, dan penataan *symbology* (SLD).
* **PostgreSQL / PostGIS:** Basis data spasial utama.
* **Nginx & Memcached:** Reverse proxy, penanganan SSL, dan manajemen cache performa web.

---

## 📚 Navigasi Dokumentasi

Pilih topik dokumentasi di bawah ini atau gunakan bilah pencarian (*search bar*) di bagian atas untuk menemukan perintah/panduan spesifik:

* [**Panduan Administrasi & Pemeliharaan**](geoportal/panduan-administrasi-geoportal.md)
  * Manajemen pengguna, superuser, dan pembatasan pendaftaran.
  * Prosedur backup database PostGIS dan volume GeoServer.
  * Sinkronisasi layer, regenerasi XML metadata, dan izin akses.
  * Integrasi CORS GeoServer dengan ArcGIS Online.
  * Pembersihan cache Nginx/Memcached dan pengumpulan berkas statis.

---

## 💡 Bantuan Cepat (Quick Commands)

Berikut adalah beberapa perintah operasional cepat yang paling sering digunakan:

```bash
# Cek status seluruh container Geoportal
docker compose ps

# Bersihkan cache web & gateway Nginx
docker restart memcached4geoportal nginx4geoportal

# Regenerasi XML metadata & izin baca publik
docker exec -it django4geoportal python manage.py regenerate_xml
docker exec -it django4geoportal python manage.py set_layers_permissions --permission=view
```