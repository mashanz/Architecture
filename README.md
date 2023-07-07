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

### A. Managed Service using Cloudflare
Apabila budget development dan maintanance terbatas seperti kurang nya developer, serta waktu untuk release harus segera. Diperlukan arsitektur dengan service yang reliable, bisa scalable, tidak rumit dan bisa langsung configurasi + deploy dalam hitungan menit. Kita dapat menggunakan ekosistem cloudflare untuk case di sini:

![CF](./Cloudflare%20Online%20Learning.svg)

#### Description
- Cloudflare DNS: Domain Management, SSL, Proxy, Email routing
  - Kelebihan:
    - DNS, SSL, Proxy, Email Routing bisa di manage di dashboard cloudflare
- Cloudflare Page: Web ujian dan dashboard (FE & BE) di jalankan disini.
  - Kelebihan:
    - Deploy lebih cepat dengan budget pas pasan
    - auto scalling, auto deployment, load balancing, bandwidth management semua sudah di handle cloudflare
  - Kekurangan:
    - limited build count untuk free plan, bisa di upgrade dengan paid plan
    - Perlu custom adapter/script untuk optimasi API Cloudflare Page (edge computing, auto scaling)
- Cloudflare KV: untuk menyimpan data berupa key-value (seperti login session, online status, frequently high through put data)
- Cloudflare D1: severless SQL service untuk menyimpan database User, Soal & Jawaban, Submit Hasil Ujian dsb
- Cloudflare R2: serverless object storage
- Cloudflare Stream: live streaming / on-demand video service
- Cloudflare Worker: serverless action untuk Cronjob dan eksekusi email service
- Cloudflare Queue: serverless email service (sent/recieve)
