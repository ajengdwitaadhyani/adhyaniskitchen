# Adhyani's Kitchen - Website Sistem Pre-Order

Selamat datang di repositori Adhyani's Kitchen. Ini adalah proyek website *Single-Page Application* (SPA) yang dirancang untuk mengelola pemesanan makanan *homemade* dengan sistem *pre-order*. Aplikasi ini sepenuhnya *serverless*, menggunakan Google Sheets sebagai database dan Google Apps Script sebagai backend, serta di-deploy secara aman ke GitHub Pages.

## ✨ Fitur Utama

  - **Katalog Produk Dinamis**: Menampilkan semua produk dari Google Sheet dengan sistem filter berdasarkan kategori.
  - **Penanganan Link Gambar Otomatis**: Secara otomatis mengubah link pratinjau Google Drive menjadi link gambar langsung, memudahkan manajemen produk.
  - **Manajemen Kuantitas**: Pengguna dapat menentukan jumlah item per produk langsung dari kartu produk.
  - **Keranjang Belanja Interaktif**: *Floating cart* menampilkan ringkasan pesanan, total harga, serta opsi untuk mengubah jumlah atau menghapus item.
  - **Backend Gratis & Serverless**: Pesanan otomatis disimpan ke Google Sheet melalui API dari Google Apps Script.
  - **Formulir Pemesanan Lengkap**: Mengumpulkan data nama, email, dan nomor WhatsApp aktif pelanggan.
  - **Verifikasi Pesanan Otomatis**:
      - **ID Pesanan Unik**: Setiap pesanan memiliki ID unik (contoh: `AK-1728391823`).
      - **Notifikasi Email**: Sistem mengirim email konfirmasi otomatis berisi detail pesanan.
  - **Integrasi WhatsApp**: Setelah pesanan terkirim, pelanggan diarahkan ke WhatsApp dengan pesan template untuk konfirmasi pembayaran.
  - **Deployment Aman (CI/CD)**: URL Google Apps Script dijaga kerahasiaannya menggunakan **GitHub Secrets** dan diinjeksikan secara otomatis saat proses deployment melalui **GitHub Actions**.

## 🛠️ Teknologi yang Digunakan

  - **Frontend**: HTML, Tailwind CSS, Vanilla JavaScript.
  - **Development Server**: Vite.
  - **Backend**: Google Sheets, Google Apps Script.
  - **CI/CD**: GitHub Actions.
  - **Hosting**: GitHub Pages.

## 🚀 Panduan Setup & Deployment

Ikuti panduan ini untuk melakukan setup proyek di lingkungan lokal dan mendeploy-nya ke produksi.

### Bagian 1: Konfigurasi Backend (Google Sheet & Apps Script)

Langkah ini hanya perlu dilakukan satu kali untuk menyiapkan API endpoint Anda.

#### **1.1 Buat dan Konfigurasi Google Sheet**

1.  Buka [Google Sheets](https://sheets.google.com) dan buat **Blank spreadsheet**. Beri nama, misalnya `Database Adhyani's Kitchen`.

2.  Buat **dua sheet** di dalam file tersebut:

      * Sheet pertama, ganti namanya menjadi **`Pesanan Masuk`**.
      * Sheet kedua, buat baru dan beri nama **`Daftar Produk`**.

3.  **Setup Sheet `Pesanan Masuk`**:
    Buat header kolom pada baris pertama. **Urutan dan penamaan harus sama persis seperti di bawah** agar skrip berfungsi dengan benar.

| Kolom | Header | Catatan |
| :--- | :--- | :--- |
| A1 | `orderId` | ID unik untuk setiap pesanan. |
| B1 | `timestamp` | Waktu pesanan dibuat. |
| C1 | `nama` | Nama pelanggan. |
| D1 | `email` | Email pelanggan. |
| E1 | `whatsapp` | **BARU**: Nomor WhatsApp pelanggan (format `wa.me/...`). |
| F1 | `pesanan` | Rangkuman item pesanan. |
| G1 | `jumlah` | Rangkuman jumlah per item. |
| H1 | `totalPembayaran`| Total tagihan. |
| I1 | `statusPembayaran` | Buat *dropdown* dengan opsi `Belum Dibayar`, `Dibayar`. |
| J1 | `statusPesanan` | Buat *dropdown* dengan opsi `Diproses`, `Dikirim`, `Selesai`. |

4.  **Setup Sheet `Daftar Produk`**:
    Sheet ini adalah sumber data untuk produk Anda. Isi dengan beberapa contoh. Anda bisa menggunakan link pratinjau Google Drive biasa untuk `imageUrl`.

| id | name | price | category | description | imageUrl |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | Cimol Keju | 10000 | cimol | Isi 15 butir per porsi | `https://drive.google.com/file/d/FILE_ID_ANDA/view?usp=sharing` |
| 2 | Cireng Isi | 15000 | cireng | Isi 5pcs per porsi | `https://placehold.co/600x400/FFE4B5/A0522D?text=Cireng+Isi` |
| 3 | Risol Mayo | 20000 | risol | Isi 5pcs per porsi | `https://placehold.co/600x400/F5DEB3/964B00?text=Risol+Mayo` |

#### **1.2 Buat Google Apps Script**

1.  Dari Google Sheet Anda, buka **Extensions \> Apps Script**.
2.  Hapus semua kode default, lalu salin dan tempel **seluruh kode di bawah ini**. Kode ini sudah diperbarui untuk menangani input nomor WhatsApp.

<!-- end list -->

```javascript
const sheetPesanan = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("Pesanan Masuk");
const sheetProduk = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("Daftar Produk");

/**
 * Mengubah berbagai jenis URL gambar menjadi URL gambar langsung (direct link).
 * Tidak lagi menggunakan konversi Base64.
 * @param {string} url - URL gambar dari Google Sheet.
 * @returns {string} URL gambar yang bisa langsung ditampilkan di <img>.
 */
function getDirectImageUrl(url) {
  if (typeof url !== 'string' || url === '') return "[https://placehold.co/600x400/CCCCCC/FFFFFF?text=Invalid+URL](https://placehold.co/600x400/CCCCCC/FFFFFF?text=Invalid+URL)";

  // 1. Menangani link halaman Imgur (contoh: [https://imgur.com/abcdef](https://imgur.com/abcdef))
  const imgurMatch = url.match(/imgur\.com\/([a-zA-Z0-9]+)$/);
  if (imgurMatch && imgurMatch[1]) {
    return `https://i.imgur.com/${imgurMatch[1]}.jpg`;
  }

  // 2. Menangani link Google Drive
  const driveMatch = url.match(/drive\.google\.com\/file\/d\/([a-zA-Z0-9_-]+)/);
  if (driveMatch && driveMatch[1]) {
    return `https://drive.google.com/uc?export=view&id=${driveMatch[1]}`;
  }

  // 3. Jika sudah merupakan link gambar langsung, kembalikan apa adanya.
  if (url.match(/\.(jpeg|jpg|gif|png)$/) != null) {
    return url;
  }

  // Fallback jika format tidak dikenali
  return url;
}


function doGet(e) {
  try {
    const productsData = sheetProduk.getDataRange().getValues();
    const headers = productsData.shift(); // Ambil header
    
    const imageUrlIndex = headers.indexOf('imageUrl');

    const productsJson = productsData.map(row => {
      let product = {};
      headers.forEach((header, index) => {
        // Jika kolomnya adalah imageUrl, proses dengan fungsi baru
        if (index === imageUrlIndex) {
          product[header] = getDirectImageUrl(row[index]);
        } else {
          product[header] = row[index];
        }
      });
      return product;
    });
    
    return ContentService.createTextOutput(JSON.stringify(productsJson))
      .setMimeType(ContentService.MimeType.JSON);
  } catch (error) {
    return ContentService.createTextOutput(JSON.stringify({ error: error.toString() }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

// --- Fungsi doPost dan lainnya tetap sama ---
function doPost(e) {
  try {
    const orderData = JSON.parse(e.parameter.items);
    const customerName = e.parameter.nama;
    const customerEmail = e.parameter.email;
    const customerWhatsapp = e.parameter.whatsapp;
    const orderId = e.parameter.orderId;

    const products = getProductsAsObject();
    
    let totalPembayaran = 0;
    let pesananDetails = [];
    let jumlahDetails = [];

    for (const itemId in orderData) {
      const product = products[itemId];
      const quantity = orderData[itemId];
      if (product) {
        totalPembayaran += product.price * quantity;
        pesananDetails.push(product.name);
        jumlahDetails.push(`${quantity}x`);
      }
    }

    const timestamp = new Date().toLocaleString("id-ID", { timeZone: "Asia/Jakarta" });
    sheetPesanan.appendRow([
      orderId,
      timestamp,
      customerName,
      customerEmail,
      customerWhatsapp,
      pesananDetails.join(', '),
      jumlahDetails.join(', '),
      totalPembayaran,
      "Belum Dibayar",
      "Diproses"
    ]);

    sendConfirmationEmail(orderId, customerName, customerEmail, pesananDetails, jumlahDetails, totalPembayaran);

    return ContentService.createTextOutput(JSON.stringify({
      result: "success",
      data: {
          totalPembayaran: totalPembayaran,
          pesanan: pesananDetails.join(', '),
          jumlah: jumlahDetails.join(', ')
      }
    })).setMimeType(ContentService.MimeType.JSON);

  } catch (error) {
    return ContentService.createTextOutput(JSON.stringify({
      result: "error",
      message: error.toString()
    })).setMimeType(ContentService.MimeType.JSON);
  }
}

function getProductsAsObject() {
  const productsData = sheetProduk.getDataRange().getValues();
  const headers = productsData.shift();
  const productsObj = {};
  productsData.forEach(row => {
    let product = {};
    headers.forEach((header, index) => {
        product[header] = row[index];
    });
    productsObj[product.id] = product;
  });
  return productsObj;
}

function sendConfirmationEmail(orderId, nama, email, pesanan, jumlah, total) {
  var subject = "Konfirmasi Pesanan Adhyani's Kitchen #" + orderId;
  let itemDetailsHtml = "";
  pesanan.forEach((item, index) => {
      itemDetailsHtml += `${jumlah[index]} ${item}<br>`;
  });
  var htmlBody = `
    <html>
      <body style="font-family: Arial, sans-serif; color: #333;">
        <h2>Terima Kasih Telah Memesan, ${nama}!</h2>
        <p>Pesanan Anda dengan ID <strong>${orderId}</strong> telah kami terima.</p>
        <hr>
        <h3>Detail Pesanan:</h3>
        <p>${itemDetailsHtml}</p>
        <h3>Total: Rp ${parseInt(total).toLocaleString('id-ID')}</h3>
        <hr>
        <p>Silakan lanjutkan konfirmasi dan pembayaran melalui WhatsApp.</p>
        <p>Terima kasih,<br><strong>Adhyani's Kitchen</strong></p>
      </body>
    </html>
  `;
  MailApp.sendEmail({ to: email, subject: subject, htmlBody: htmlBody });
}
```

3.  Simpan proyek Apps Script dengan nama `API Pesanan Website`.

#### **1.3 Deploy Script sebagai Web App**

1.  Klik **Deploy \> New deployment**.
2.  Pilih tipe **Web app** dan atur konfigurasi:
      * **Description**: `API untuk website Adhyani's Kitchen v2`.
      * **Execute as**: `Me`.
      * **Who has access**: `Anyone`.
3.  Klik **Deploy**. Lakukan otorisasi jika diminta oleh Google.
4.  Salin **Web app URL** yang muncul. Simpan URL ini baik-baik.

-----

### Bagian 2: Pengembangan Lokal (Local Development)

Gunakan lingkungan ini untuk testing dan pengembangan fitur baru sebelum di-deploy.

#### **2.1 Prasyarat**

  * Pastikan Anda telah menginstal [Node.js](https://nodejs.org/) (versi 18 atau lebih baru).

#### **2.2 Setup Proyek Lokal**

1.  Clone repositori ini ke komputer Anda.

2.  Buat file bernama `.env` di *root folder* proyek. File ini **tidak akan** di-upload ke GitHub dan hanya digunakan untuk pengembangan lokal.

3.  Isi file `.env` dengan URL API Anda dari Langkah 1.3. **Penting**: Nama variabel harus diawali dengan `VITE_`.

    ```env
    # .env
    # Variabel ini akan dibaca oleh Vite saat menjalankan 'npm run dev'
    VITE_GOOGLE_SCRIPT_URL="URL_WEB_APP_ANDA_DARI_LANGKAH_1.3"
    ```

4.  Buka terminal di folder proyek, lalu instal semua dependensi yang dibutuhkan:

    ```bash
    npm install
    ```

5.  Jalankan server pengembangan lokal:

    ```bash
    npm run dev
    ```

    Buka URL `http://localhost:xxxx` yang ditampilkan di terminal pada browser Anda. Website akan berjalan dan mengambil data dari Google Sheet Anda.

-----

### Bagian 3: Deployment ke Produksi (GitHub Pages)

Proses ini mengotomatiskan deployment website Anda ke GitHub Pages setiap kali ada perubahan di branch `main`.

#### **3.1 Konfigurasi GitHub Secrets**

URL Google Apps Script Anda adalah informasi sensitif. Kita akan menyimpannya dengan aman di GitHub Secrets.

1.  Di repositori GitHub Anda, buka tab **Settings \> Secrets and variables \> Actions**.
2.  Klik **New repository secret** dan tambahkan:
      * **Name**: `GOOGLE_SCRIPT_URL`
      * **Secret**: Tempel **Web app URL** yang Anda dapatkan dari Langkah 1.3.

#### **3.2 Pastikan Workflow CI/CD Ada**

Pastikan file workflow berikut ada di dalam repositori Anda pada path `.github/workflows/deploy.yml`. Skrip ini akan secara otomatis mengganti placeholder di `index.html` dengan secret Anda saat deployment.

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches:
      - main

# [PERBAIKAN] Menambahkan izin agar Actions dapat melakukan push ke branch gh-pages
permissions:
  contents: write

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout 🛎️
        uses: actions/checkout@v3

      - name: Replace Google Script URL 🔄
        run: |
          sed -i "s|__GOOGLE_SCRIPT_URL__|${{ secrets.GOOGLE_SCRIPT_URL }}|g" index.html
        env:
          GOOGLE_SCRIPT_URL: ${{ secrets.GOOGLE_SCRIPT_URL }}

      - name: Deploy 🚀
        uses: JamesIves/github-pages-deploy-action@v4
        with:
          branch: gh-pages
          folder: .
```

#### **3.3 Aktifkan GitHub Pages**

1.  Di repositori Anda, buka **Settings \> Pages**.
2.  Pada bagian "Build and deployment", di bawah "Source", pilih **GitHub Actions**. Opsi ini memungkinkan workflow CI/CD untuk mem-publish situs Anda.

#### **3.4 Lakukan Push Perdana**

Commit dan push semua perubahan Anda ke branch `main`. Ini akan memicu GitHub Action untuk pertama kalinya.

```bash
git add .
git commit -m "feat: setup initial project with deployment workflow"
git push origin main
```

Setelah workflow selesai berjalan (sekitar 1-2 menit), website Anda akan tersedia di URL publik yang ditampilkan pada tab **Settings \> Pages**.