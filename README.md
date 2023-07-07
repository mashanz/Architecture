# Design System
Design System E-Learning Platform

## Request
Cloud Based Arch (Apps, Network, 3rd Party)

## Cases
- Organisasi membutuhkan layanan pembelajaran online
- fitur utama Streaming Video dan Ujian Online
- target user harian normal 1.000 concurent user dan peak 300.000 concurent user

## Requirements
1. Diperlukan sebuah aplikasi web untuk melayani pembelajaran online dan ujian.
2. Video pembelajaran bersifat private, tidak boleh bisa diakses oleh umum.
3. Untuk Ujian Online, terdapat email reminder H-1 kepada semua peserta ujian.
4. Hasil Ujian Online akan langsung muncul ketika seorang peserta telah menyelesaikan semua
soal ujian.
5. Diperlukan sebuah aplikasi admin dashboard untuk memantau ujian online secara realtime
(berapa peserta mendaftar, berapa peserta yang lulus/tidak lulus, dsb).
6. Sistem semestinya memiliki reliability yang bagus, dan mempunyai backup plan jika sistem
tersebut mengalami kegagalan dalam menangani request dari user

## Analysis (A) & Questions (Q) per points
### Point 1:

#### Analysis
- Kebutuhan layanan pembelajaran online artinya organisasi memerlukan system berbentuk LMS (Learning Management System)
- LMS memerlukan service untuk penyimpanan database, storage, hosting, domain dan proxy

#### Questions
- Apakah org ingin menggunakan tools LMS yg sudah tersedia (Moodle, Odoo, Google Classroom, etc) atau membuat dari awal?
- Kapan layanan ingin digunakan/diluncurkan?
  - sekitar 1 bln (short)
  - sekitar 3 bln (quarter)
  - sekitar 6 bln (mid)
  - sekitr 12 bln (long/full year)
- Ada berapa budget pembuatan dan pemeliharaan layanan?

### Point 2:
#### Analysis
- Diperlukan Video Streaming Service
- Video Streaming memiliki access restriction / private (hanya org yg di izinkan mendapatkan akses)

#### Questions
- Apakah ingin membuat streaming service sendiri atau menggunakan streaming service yg sudah tersedia?

### Point 3:
#### Analysis
- Diperlukan database email peserta ujian
- Diperlukan service pengiriman email
- Diperlukan CronJob Service untuk eksekusi kirim email notification/reminder

#### Questions
- Apakah email peserta merupakan pemberian organisasi dan dalam 1 provider seperti `belajar.id` yang menggunakan google workspace.
- Apabila ya, development akan lebih mudah, bila tidak perlu ada analisis tambahan untuk (perbedaan provider, security, spaming)

### Point 4
#### Analysis
- Diperlukan fitur ujian berupa:
  - Halaman ujian online dan hasil ujian
  - database ujian, soal dan jawaban ujian
  - asset storage bila soal memiliki illustrasi
  - database jawaban peserta untuk ujian tersebut
  - database hasil kalkulasi jawaban peserta
- Diperlukan Fitur seperti dashboard untuk mengunggah soal dan jawaban ujian

#### Questions
- Siapa yang bisa akses hasil ujian? peserta tersebut saja (private), atau peserta lain dapat melihat hasil ujian dari peserta tersebut juga (public)?
- Berapa lama data ujian dan hasil ujian akan disimpan di database?
- Siapa bagaimana ujian diupload/dibuat? import dengan format csv atau input manual satu persatu di dashboard?

### Point 5
#### Analysis
- Seperti point 4, selain input soal ujian di dashboard, perlu ada juga fitur `memantau ujian online secara realtime`
  - Total peserta terdaftar
  - Peserta yg lulus
  - dsb?

#### Questions
- Perlu definisi / list yang jelas apa saja yang di maksud dengan `dsb / dan sebagainya` pada point 5 (apakah monitoring status online peserta? monitoring pergerakan mouse? monitoring log kyboard? ini perlu ada list tersendiri)

### Point 6
#### Analysis
- Di perlukan service provider yg reliable seperti:
  - Custom Service: Googcle Cloud Platform (GCP), Amazon Web Service (AWS)
  - Managed Service: Cloudflare, Vercel, Heroku
- Diperlukan backup plan seperti backup server, backup database, offline first experience (bila server mati / tidak ada internet)

## Design Architecture
Terdapat banyak kemungkinan untuk membuat arsitektur yangg scalable dengan requirement di atas. Secara umum terdapat 3 kemungkinan, namun saya akan menjelaskan 2 contoh arsitektur yg mungkin untuk dibuat:
- Full Managed Service
- Full Cloud Service
- Mix Cloud + Managed Service (let's discuss more about it later)

### A. Full Managed Service using Cloudflare
Apabila budget development dan maintanance terbatas seperti kurang nya developer, serta waktu untuk release harus segera. Diperlukan arsitektur dengan service yang reliable, bisa scalable, tidak rumit dan bisa langsung configurasi + deploy dalam hitungan menit. Kita dapat menggunakan ekosistem cloudflare untuk case di sini:

![CF](https://raw.githubusercontent.com/mashanz/Architecture/main/Cloudflare%20Online%20Learning.svg)

#### Description
- [Cloudflare DNS](https://developers.cloudflare.com/dns/): Domain Management, SSL, Proxy, [Email routing](https://developers.cloudflare.com/email-routing/)
  - Kelebihan:
    - DNS, SSL, Proxy, Email Routing bisa di manage di dashboard cloudflare
- [Cloudflare Page](https://developers.cloudflare.com/pages/): tempat deploy apps (Web ujian dan dashboard (FE & BE) di jalankan di sini) di sini saya rekomendasikan menggunakan LMS yang sudah jadi dan support cloudflare page API atau build from scratch dari framework populer.  (Fullstack framework populer yg sudah support adalah SvelteKit, NextJS, NuxtJS, ini akan jadi pembahasan terpisah apakah kita akan menggunakan monorepo, super app, user privilage menggunakan RBAC, Load testing dengan Locust, atau micro service, kita bisa diskusikan lebih mendalam di waktu terpisah).
  - Kelebihan:
    - Deploy lebih cepat dengan budget pas pasan
    - auto scalling, auto deployment, load balancing, bandwidth management semua sudah di handle cloudflare
  - Kekurangan:
    - limited build count untuk free plan, bisa di upgrade dengan paid plan
    - Perlu custom adapter/script untuk optimasi API Cloudflare Page (edge computing, auto scaling)
- [Cloudflare KV](https://developers.cloudflare.com/workers/wrangler/workers-kv/): untuk menyimpan data berupa key-value (seperti login session, online status, frequently high through put data)
- [Cloudflare D1](https://developers.cloudflare.com/d1/): severless SQL service untuk menyimpan database User, Soal & Jawaban, Submit Hasil Ujian dsb
- [Cloudflare R2](https://developers.cloudflare.com/r2/): serverless object storage (pnyimpanan assets LMS seperti foto profile, gambar, dokumen dsb)
- [Cloudflare Stream](https://developers.cloudflare.com/stream/): live streaming / on-demand video service (private video bisa di unggah di sini)
- [Cloudflare Workers](https://developers.cloudflare.com/workers/): serverless action untuk Cronjob dan eksekusi email service
- [Cloudflare Queue](https://developers.cloudflare.com/queues/): serverless email service (sent/recieve)

Kelebihan Cloudflare:
- Tidak seperti managed service lain seperti Vercel dan Heroku yang underlying nya adalah AWS, cloudflare memiliki service tersendiri dan reliability nya tidak bergantung dengan AWS.
- Trafik dan Keamanana semua sudah di manage oleh cloudflare seperti DDOS protection, captcha, trafic management, monitoring, dsb (kita tidak perlu setup dari 0 dan tinggal menggunakannya)
- Harga dan package relative lebih murah dari AWS dan GCP
- Dari setup arsitektur, development app hingga deployment bisa di lakukan hanya oleh seorang Engineer.

Kekurangan:
- Harus mengikuti API Cloudflare
- Memiliki keterbatasan di setiap service nya. [Dokumentasi Cloudflare](https://cloudflare.com)

### B. Full Cloud Service using Google Cloud Platform
Apabila memiliki budget dan resource yang memadai serta waktu release tidak terlalu mepet. Bisa menggunakan Google Cloud Platform (GCP). Berikut arsitektur sederhana secara umum bila menggunakan GCP:
![GCP](https://raw.githubusercontent.com/mashanz/Architecture/main/GCP%20Online%20Learning.svg)

#### Description
Secara umum, bentuk arsitektur tidak berbeda jauh dengan arsitektur managed service di bagian A (CLoudflare arsitektur). Namun tantangan tersendiri adalah di bagian GKE.
Auto scaling, Load Balancing, Networking etc semua harus di konfigurasi sendiri dan memerlukan engineer yangg memiliki specialisasi di bidang DevOps / SRE.

Kelebihan GCP:
- Full customasi dalam pembuatan arsitektur.
- High Reliability

Kekurangan GCP:
- Perlu adanya specialist DevOps / SRE untuk konfigurasi GKE (Perlu ada pembahasan terpisah untuk konfigurasi GKE di GCP)
- Konfigurasi tidak tepat dapat menyebabkan pembengkakan biaya tagian GCP
- Butuh waktu lebih lama untuk konfigurasi arsitektur
- Google hanya menyediakan 1500 kirim emal per hari, perlu adanya 3rd party email service agar bisa mengirim lebih dari 1500 email dalam sehari
