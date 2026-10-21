# zcode-bynara-plugin

Koleksi plugin [ZCode](https://zcode.ai) untuk Bynara API (`api-images.bynara.id`): generate gambar dan video langsung dari dalam ZCode.

## Isi marketplace

| Plugin | Versi | Fungsi |
|---|---|---|
| `bynara-image-generate` | 0.1.2 | Generate gambar dari teks atau edit gambar yang ada (`agnes-image-2.1-flash`, `agnes-image-2.0-flash`, `gpt-image-2`) |
| `bynara-video-generate` | 0.1.0 | Generate video MP4 pendek — text-to-video (**t2v**), image-to-video (**i2v**, gambar jadi frame pertama), reference-to-video (**r2v**, 1–3 gambar referensi digabung jadi scene baru) dengan `agnes-video-v2.0` |

## Instalasi

1. Di ZCode buka **Plugin Marketplace → Add → Add Plugin Marketplace**, lalu tempel path folder ini.
2. Pilih marketplace **zcode-bynara-plugin**, lalu **Install** plugin yang diinginkan.
3. Set API key Bynara, lalu restart ZCode:

   ```
   setx __BYNARA_API_KEY "sk-nry-..."
   ```

   Jangan pernah menyimpan API key di dalam repo ini — key dibaca dari environment variable `__BYNARA_API_KEY`.

## Pemakaian

| Command | Fungsi |
|---|---|
| `/bynara-image <prompt>` | Generate gambar dari teks |
| `/bynara-image-edit <gambar> <instruksi>` | Edit gambar yang ada |
| `/bynara-video <prompt>` | Generate video dari teks (selesai dalam 1–5 menit) |
| `/bynara-video-animate <gambar> [gambar2 gambar3] <prompt>` | Animasikan 1 gambar (jadi frame pertama) atau gabungkan 1–3 gambar referensi jadi scene baru |

Skill `bynara-image-generate` dan `bynara-video-generate` juga terpicu otomatis dari prompt bahasa natural (mis. "buatkan gambar...", "animasikan foto ini jadi video...").

## Catatan

- Semua request dibayar per pemakaian di akun Bynara Anda.
- Video: durasi 3–15 detik (billed per detik; 720p Rp100/detik, 1080p Rp250/detik). Hasil diunduh otomatis ke disk karena file di server expire 3 jam.
- Gambar: hasil diterima sebagai base64 dan disimpan ke folder `generated-images/`; video ke `generated-videos/`.
