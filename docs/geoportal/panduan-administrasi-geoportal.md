# Panduan Administrasi & Pemeliharaan Geoportal

Halaman ini berisi kumpulan perintah operasional, skrip administrasi, dan langkah-langkah pemeliharaan (*maintenance*) untuk mengelola ekosistem Geoportal berbasis **GeoNode** dan **GeoServer** di lingkungan Docker.

---

## 1. Manajemen Pengguna & Keamanan

### A. Membuat & Mengubah Kata Sandi Administrator
Perintah berikut digunakan untuk mengelola akun dengan hak akses penuh (*superuser*) pada Django GeoNode.

* **Mengubah Kata Sandi Admin yang Sudah Ada:**
  ```bash
  
  docker exec -it django4geoportal python manage.py changepassword admin
  
  ```

* **Membuat Akun Superuser Baru:**
  ```bash

  docker exec -it django4geoportal python manage.py createsuperuser
  
  ```

### B. Membatasi Pendaftaran Pengguna
Untuk menjaga keamanan sistem agar tidak sembarang orang bisa mendaftar atau mengunggah layer, sesuaikan variabel berikut pada file konfigurasi environment (`settings.py` atau `.env`):

* **Menutup Pendaftaran Mandiri (Public Signup):**
  ```python
  ACCOUNT_OPEN_SIGNUP=False
  ```

* **Mewajibkan Moderasi Admin untuk Setiap Unggahan Data:**
  ```python
  ADMIN_MODERATE_UPLOADS=True
  ```

---

## 2. Pemeliharaan & Backup Data (Backup & Recovery)

Lakukan prosedur backup ini secara berkala untuk meminimalisasi risiko kehilangan data spasial maupun konfigurasi sistem.

### A. Backup Database PostgreSQL / PostGIS
Mengunduh seluruh *dump* basis data `geonode` dari container PostgreSQL ke dalam file `.sql` berformat tanggal hari ini:
```bash
docker exec -t db4geoportal pg_dump -U postgres geonode > backup_$(date +%F).sql
```

### B. Backup Volume Data GeoServer
Mengompres seluruh data konfigurasi dan *style* dari volume Docker `geoserver_data` menjadi berkas tarball (`.tar.gz`):
```bash
docker run --rm -v geoserver_data:/data -v $(pwd):/backup alpine tar czf /backup/geoserver_backup.tar.gz -C /data
```

---

## 3. Sinkronisasi Layer & Metadata

Gunakan perintah ini saat terdapat ketidaksesuaian data antara GeoServer, GeoNode, dan PostgreSQL, atau setelah melakukan *bulk update*.

### A. Sinkronisasi Data Internal
* **Sinkronisasi Dataset GeoNode:**
  ```bash
  docker exec -it django4geoportal python manage.py sync_geonode_datasets
  ```

* **Memperbarui Seluruh Layer (atau Layer Tertentu):**
  ```bash
  # Memperbarui seluruh layer
  docker exec -it django4geoportal python manage.py updatelayers

  # Memperbarui layer spesifik saja
  docker exec -it django4geoportal python manage.py updatelayers --filter=nama_layer_anda
  ```

* **Menyegarkan Seluruh Metadata Dataset:**
  ```bash
  docker exec -it django4geoportal python manage.py set_all_datasets_metadata
  ```

### B. Regenerasi XML Metadata & Pengaturan Izin Publik
Prosedur ini dilakukan jika file XML metadata corrupt, tidak tampil pada domain *production*, atau tidak dapat dibaca oleh aplikasi desktop GIS seperti QGIS tanpa login:

1. **Generate Ulang File XML Metadata:**
   ```bash
   # Regenerasi seluruh file XML
   docker exec -it django4geoportal python manage.py regenerate_xml

   # Regenerasi file XML untuk layer tertentu saja
   docker exec -it django4geoportal python manage.py regenerate_xml -l lahan_pemkot_update_190923
   ```

2. **Buka Akses Izin Baca (*View Permissions*):**
   ```bash
   docker exec -it django4geoportal python manage.py set_layers_permissions --permission=view
   ```

---

## 4. Konfigurasi GeoServer & Integrasi Layanan Spasial

### A. Aturan Penulisan SLD (Styled Layer Descriptor)
> **Penting:** Saat menyusun file XML SLD untuk *symbology* peta di GeoServer, tag properti wajib menggunakan format **huruf kecil (*lowercase*)**.

```xml
<ogc:PropertyName>nama_kolom</ogc:PropertyName>
```

### B. Menangani Kendala SSL / Reverse Proxy
Jika GeoServer mengalami kendala CSRF saat diakses melalui HTTPS (SSL), Anda dapat menambahkan properti parameter JVM berikut secara sementara:
```bash
-DGEOSERVER_CSRF_DISABLED=true
```

### C. Mengaktifkan CORS untuk Integrasi ArcGIS Online
Agar layanan peta WMS/WFS/WMTS dari GeoServer dapat dipanggil dan diakses langsung dari platform lain (seperti ArcGIS Online atau aplikasi pihak ketiga), tambahkan filter CORS ke dalam berkas `web.xml` GeoServer:

1. **Atur isi konfigurasi `web.xml`:**
   ```xml
   <filter>
       <filter-name>DockerGeoServerCorsFilter</filter-name>
       <filter-class>org.apache.catalina.filters.CorsFilter</filter-class>
       <init-param>
           <param-name>cors.allowed.origins</param-name>
           <param-value>[https://geoportal.bontangkota.go.id](https://geoportal.bontangkota.go.id),[https://bontangkota.maps.arcgis.com](https://bontangkota.maps.arcgis.com),[https://arcgis.com](https://arcgis.com)</param-value>
       </init-param>
       <init-param>
           <param-name>cors.support.credentials</param-name>
           <param-value>true</param-value>
       </init-param>
       <init-param>
           <param-name>cors.allowed.methods</param-name>
           <param-value>GET,POST,PUT,DELETE,HEAD,OPTIONS</param-value>
       </init-param>
   </filter>
   ```

2. **Terapkan perubahan ke dalam Container GeoServer:**
   ```bash
   # Salin file web.xml yang telah diperbarui ke dalam container
   docker cp ./web.xml geoserver4geoportal:/usr/local/tomcat/webapps/geoserver/WEB-INF/web.xml

   # Backup skrip entrypoint jika diperlukan
   docker cp geoserver4geoportal:/usr/local/tomcat/tmp/entrypoint.sh ./entrypoint.sh

   # Muat ulang container GeoServer
   docker compose up -d --force-recreate geoserver
   ```

---

## 5. Pembersihan Cache & Perbaikan Tampilan (Static Files)

Gunakan perintah ini ketika Anda baru saja mengubah kode template web, gambar, CSS/JS, atau ketika perubahan data pada peta tidak langsung terlihat di peramban (browser).

### A. Membersihkan Cache Server & Reverse Proxy
```bash
# Restart memori cache aplikasi Django
docker restart memcached4geoportal

# Restart Nginx web server untuk reload cache gateway
docker restart nginx4geoportal
```

### B. Memperbarui Berkas Statis & Migrasi Database
Jika tampilan UI berantakan atau ada pembaruan tabel Django:
```bash
# Kumpulkan ulang berkas statis (CSS, JS, Media)
docker exec -it django4geoportal python manage.py collectstatic --noinput --clear

# Eksekusi migrasi skema tabel jika ada perubahan struktur
docker exec -it django4geoportal python manage.py migrate
```

---

## 6. Referensi Perintah Django Manage

Untuk melihat seluruh daftar perintah (*command line utility*) bawaan Django & GeoNode yang tersedia pada sistem:
```bash
docker exec -it django4geoportal python manage.py help
```