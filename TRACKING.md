# TRACKING.md — Dokumentasi Tracking Skupy Bordir

Terakhir diperbarui: 27 Juli 2026

## Arsitektur

```
Klik / scroll pengunjung
        │
        ▼
  track(name, params)          ← satu fungsi pusat, di akhir <body> index.html
        │
        ├─► dataLayer.push({event: name, ...})   ← format GTM; menganggur sampai GTM dipasang
        └─► gtag('event', name, params)          ← aktif SEKARANG, selama SEND_DIRECT = true
                    │
                    ├─► GA4  (G-G9SMTY996S)
                    └─► Ads  (AW-18169243507)
```

**Status GTM: BELUM terpasang.** Butuh container ID (`GTM-XXXXXXX`) yang hanya bisa
dibuat pemilik akun di [tagmanager.google.com](https://tagmanager.google.com). Sampai
saat itu, semua event tetap berjalan penuh lewat gtag langsung — tidak ada fungsi
yang hilang. Lihat bagian "Migrasi ke GTM" di bawah.

## Daftar event

| Event | Pemicu | Parameter | Tujuan |
|---|---|---|---|
| `whatsapp_click` | Klik link `wa.me` mana pun (13 titik) | `cta_location`, `cta_text` | **Konversi utama** — jalur lead sesungguhnya |
| `phone_click` | Klik link `tel:` di footer | `cta_location` | Konversi sekunder |
| `hero_cta_click` | Klik CTA di dalam `#hero` (WA atau "Lihat Galeri") | `cta_text` | Efektivitas hero |
| `contact_button_click` | Klik WA / telepon / Instagram di footer `#kontak` | `cta_text`, `link_url` | Efektivitas footer |
| `nav_cta_click` | Klik tombol WA di navbar atau bar sticky mobile | `cta_location` (`navbar`/`mobile_cta`) | Efektivitas navigasi |
| `scroll_depth` | Mencapai 25 / 50 / 75 / 90% halaman (sekali per muatan) | `depth_percent` | Keterlibatan; proxy bounce |
| `portfolio_click` | Klik item galeri atau foto workshop (buka lightbox) | `item_name`, `cta_location` | Minat produk — item mana yang menarik |
| `faq_expand` | Membuka pertanyaan FAQ (hanya saat expand, bukan tutup) | `faq_question` | Keberatan apa yang paling dicari |
| `thank_you_view` | Halaman `/thank-you.html` dimuat | — | Penanda funnel selesai |
| `conversion` (Ads) | `/thank-you.html`, sekali per sesi (guard sessionStorage) | `send_to: AW-…/KmdjCMKGsa8cEPPO4tdD` | Konversi Google Ads bawaan |

**Tidak diimplementasikan:** `form_submit` — situs tidak punya form. Jika form dibuat
kelak, panggil `track('form_submit', {form_name: '...'})` di handler sukses form, dan
redirect ke `/thank-you.html`.

## Yang perlu dilakukan di GA4 (sekali, manual di antarmuka)

1. **Verifikasi event masuk:** Admin → DebugView (atau Realtime) → buka situs →
   klik tombol WA → event `whatsapp_click` harus muncul dalam beberapa detik.
2. **Tandai sebagai Key Event** (Admin → Events → toggle "Mark as key event"):
   - `whatsapp_click` ← paling penting
   - `phone_click`
   - `thank_you_view`
   Jangan tandai `scroll_depth` / `faq_expand` / `portfolio_click` — itu sinyal
   perilaku, bukan konversi.
3. **Daftarkan custom dimension** (Admin → Custom definitions) jika ingin laporan
   per-parameter: `cta_location`, `cta_text`, `faq_question`, `item_name`,
   `depth_percent`. Tanpa ini event tetap tercatat, hanya parameternya tidak bisa
   dipakai di laporan standar.

## Yang perlu dilakukan di Google Ads (sekali, manual)

Jalur yang disarankan — **import dari GA4** (bukan tag konversi baru):

1. Pastikan GA4 dan Google Ads tertaut (GA4 Admin → Product links → Google Ads).
2. Tandai `whatsapp_click` sebagai Key Event di GA4 (langkah di atas).
3. Di Google Ads → Goals → Conversions → New conversion action → Import →
   Google Analytics 4 → pilih `whatsapp_click`.
4. Set **Count = One** (satu klik WA per pengunjung = satu lead; klik kedua bukan
   lead baru).
5. Setelah data `whatsapp_click` stabil (±2 minggu), jadikan ia **primary
   conversion** dan turunkan konversi thank-you lama menjadi secondary — karena
   tanpa form, halaman thank-you hampir tidak menerima traffic.

**Anti-konversi-palsu:** homepage tidak memuat satu pun `gtag('event','conversion')`.
Event konversi Ads hanya ada di `/thank-you.html` dan diguard sessionStorage.
Jangan pernah menandai `scroll_depth` atau `page_view` sebagai konversi.

## Migrasi ke GTM (saat container sudah dibuat)

1. Buat container di tagmanager.google.com → dapat `GTM-XXXXXXX`.
2. Tempel dua snippet GTM resmi ke `index.html` dan `thank-you.html` pada
   komentar placeholder `<!-- Google Tag Manager: BELUM terpasang -->`.
3. Di GTM, buat: satu tag **Google Tag** (GA4 `G-G9SMTY996S`), triggers
   **Custom Event** untuk tiap nama event di tabel atas, dan tag GA4 Event
   untuk masing-masing.
4. **Penting:** setelah tag GTM terbukti jalan (Preview mode), ubah
   `SEND_DIRECT = true` menjadi `false` di skrip Skupy Tracking, dan hapus
   blok `gtag('config', ...)` dari <head> (pindah ke GTM). Kalau tidak,
   setiap event dan page_view akan terkirim DUA KALI.
5. Verifikasi ulang di GA4 DebugView bahwa tiap aksi hanya menghasilkan satu event.

## Menambah event baru

```js
// Di dalam skrip Skupy Tracking, atau dari mana pun setelahnya:
track('nama_event_baru', { param1: 'nilai' });
```

Konvensi nama: huruf kecil, snake_case, kata benda_kata kerja (`whatsapp_click`,
`faq_expand`). Maksimal 40 karakter (batas GA4). Parameter maksimal 25 per event.

## Catatan kualitas

- **Satu listener klik terdelegasi** untuk semua event klik — tidak ada listener
  per-elemen, tidak ada kebocoran memori.
- Listener scroll memakai `passive: true` + `requestAnimationFrame`, dan
  **melepas dirinya sendiri** setelah 90% tercapai.
- `transport_type: 'beacon'` dipakai agar event klik tetap terkirim meski
  halaman berpindah.
- Guard `sessionStorage` mencegah konversi Ads dobel saat refresh thank-you.
- gtag stub (`function gtag(){dataLayer.push(arguments)}`) membuat semua
  panggilan aman meski `gtag.js` belum/gagal termuat — event mengantre, tidak error.
