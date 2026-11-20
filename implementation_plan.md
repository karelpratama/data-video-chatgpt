# Blueprint Arsitektur: Automatic Story Drawing & Animated Explainer Video Creator

Dokumen ini merupakan rancangan arsitektur dan spesifikasi teknis lengkap sebelum implementasi kode dimulai. Aplikasi ini dirancang sebagai studio web full-stack yang mengubah audio narasi (MP3/WAV) dan file subtitle (SRT) milik pengguna menjadi video explainer animasi dinamis berformat MP4 melalui sutradara AI, mesin gambar prosedural berbasis vektor/Canvas, mesin animasi berbasis timeline deterministik, serta pipeline rendering FFmpeg di sisi server.

---

## Ringkasan Keputusan Kritis untuk Ditinjau

> [!IMPORTANT]
> Sebelum melanjutkan ke tahap implementasi kode, mohon tinjau tiga keputusan arsitektur inti di bawah ini. Seluruh keputusan telah dioptimalkan berdasarkan kemampuan environment runtime (`Node.js v22`, `FFmpeg 4.4.2` dengan `libx264` & `aac`, tanpa dependency browser Chromium).

1. **Mesin Gambar & Animasi Terpadu (Shared Canvas2D + SVG Path Engine)**
   - *Pilihan*: Menggunakan satu codebase mesin gambar & animasi berbasis standar HTML5 **Canvas 2D + SVG `Path2D`** di folder `shared/` yang berjalan identik di Browser (`<canvas>` preview) dan di Node.js server (`@napi-rs/canvas` berbasis native Skia).
   - *Alasan*: Menjamin hasil preview di browser **100% identik secara piksel, timing, dan animasi** dengan hasil render MP4 akhir di server, tanpa harus mengunggah ribuan frame PNG dari browser ke server dan tanpa memerlukan headless Chrome/Puppeteer.
2. **Model AI Sutradara Visual (`gemini-4-hydrogen`) dengan Structured Output**
   - *Pilihan*: Menggunakan `@google/genai` di backend Node.js dengan model `gemini-4-hydrogen` dan `responseSchema` ketat untuk menganalisis transkrip SRT lalu menyusun Storyboard JSON per scene.
   - *Alasan*: API Key tetap aman di server (`process.env.GEMINI_API_KEY`), hasil JSON terjamin valid sesuai skema mesin animasi, dan mendukung regenerasi per scene secara cepat.
3. **Penyimpanan Proyek Berbasis File System Terstruktur**
   - *Pilihan*: Menyimpan setiap proyek ke dalam direktori `projects/<projectId>/` di server (`project.json`, `storyboard.json`, file audio asli, file SRT, dan `output.mp4`).
   - *Alasan*: Memenuhi kebutuhan sistem project file untuk menyimpan progress, mendukung resume pekerjaan, edit per scene, dan menjaga aset media berukuran besar tetap terisolasi rapi.

---

## 1. Analisis Seluruh Kebutuhan Proyek

### 1.1 Kebutuhan Fungsional Utama

| Domain | Kebutuhan Spesifik |
| :--- | :--- |
| **Manajemen Proyek** | Dashboard daftar proyek, buat proyek baru, muat ulang proyek tersimpan, duplikasi/hapus proyek, dan auto-save setiap perubahan storyboard. |
| **Input & Validasi Media** | Upload audio narasi (`MP3`, `WAV`) dan subtitle (`SRT`). Aplikasi tidak menghasilkan audio/SRT sendiri; `ffprobe` membaca durasi audio aktual dan parser SRT memvalidasi timestamp serta keutuhan kalimat. |
| **Pengaturan Video & Style** | Rasio aspek (`16:9` Landscape, `9:16` Portrait, `1:1` Square), Resolusi (`720p`, `1080p`), FPS (`24`, `30`, `60`), toggle Subtitle ON/OFF, dan 6 preset gaya visual konsisten. |
| **Analisis AI & Segmentasi Scene** | AI memahami konteks, emosi, topik, dan pergantian pembahasan dari SRT; membagi cerita menjadi scene logis **tanpa memotong kalimat di tengah** dan menghindari scene terlalu singkat (< 2,5 detik) atau terlalu lama (> 18 detik tanpa variasi). |
| **Visual & Animasi Dinamis** | Bukan slideshow gambar statis. Setiap scene memiliki background kontekstual, karakter beranimasi (jalan, menunjuk, bereaksi, berpikir), objek utama & pendukung, diagram/ikon, efek gambar garis bertahap (*draw-on*), pop-up, slide, motion path, kamera zoom/pan, kinetic text, serta transisi antar-scene. |
| **Editor & Regenerasi Per Scene** | Pengguna dapat memilih scene pada timeline untuk preview terisolasi, edit teks/posisi/animasi/background/timing, atau regenerate visual scene tertentu via AI tanpa mengulang keseluruhan video. |
| **Preview & Rendering MP4** | Real-time interactive player tersinkronisasi dengan audio narasi (scrubbing, play/pause, subtitle toggle) + server-side render pipeline ke MP4 dengan progress bar real-time dan tombol download. |

### 1.2 Invarian Sinkronisasi & Konsistensi

- **Audio sebagai Master Clock**: Durasi total seluruh gabungan scene pada timeline **wajib tepat sama** dengan durasi file audio (`durationMs` dari `ffprobe`), sehingga audio tidak pernah terpotong dan video tidak memiliki dead-air kosong di akhir.
- **Zero Empty Scene**: Setiap scene dijamin memiliki minimal 1 background tema + 1 elemen visual utama (karakter/objek/diagram) + animasi kamera/elemen aktif.
- **Konsistensi Gaya Global**: Palet warna, ketebalan garis (*stroke weight*), gaya sudut (*lineCap/lineJoin*), dan proporsi karakter dikunci pada tingkat proyek melalui `StylePresetConfig`.

---

## 2. Arsitektur Aplikasi & Alur Data

Aplikasi menggunakan arsitektur **Full-Stack Monorepo Modular** dengan Express + Vite middleware pada port `3000`, di mana logika grafis dan timeline bersifat isomorfik (berbagi modul `shared/` antara frontend React dan backend renderer Node.js).

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                        FRONTEND (React + Tailwind v4)                   │
│  ┌──────────────┐  ┌──────────────────┐  ┌───────────────────────────┐  │
│  │  Dashboard   │  │  Studio Player   │  │   Storyboard & Scene      │  │
│  │  & Wizard    │  │  (<canvas> +     │  │   Inspector + Timeline    │  │
│  │  Upload/Cfg  │  │   HTML5 Audio)   │  │   (Edit / Regenerate)     │  │
│  └──────┬───────┘  └────────┬─────────┘  └─────────────┬─────────────┘  │
│         │                   │                          │                │
│         │                   ▼                          │                │
│         │      ┌──────────────────────────┐            │                │
│         │      │  SHARED VISUAL KERNEL    │◄───────────┘                │
│         │      │  • Animation Evaluator   │                             │
│         │      │  • Procedural Drawing    │                             │
│         │      │  • 6 Style Theme Tokens  │                             │
│         │      └────────────┬─────────────┘                             │
└─────────┼───────────────────┼───────────────────────────────────────────┘
          │ REST API / SSE    │ (Kode mesin yang sama dipakai di Server)
          ▼                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       BACKEND (Node.js + Express)                       │
│  ┌─────────────────┐  ┌──────────────────┐  ┌────────────────────────┐  │
│  │ Media Service   │  │ AI Director      │  │ Video Render Worker    │  │
│  │ • Multer Upload │  │ • @google/genai  │  │ • @napi-rs/canvas      │  │
│  │ • ffprobe check │  │ • gemini-4-      │  │   (Frame t=0..N RGBA)  │  │
│  │ • SRT Parser &  │  │   hydrogen       │  │ • Pipe stdin ke FFmpeg │  │
│  │   Sentence Grouper │ • JSON Validator │  │   (libx264 + aac MP4)  │  │
│  └────────┬────────┘  └────────┬─────────┘  └───────────┬────────────┘  │
│           │                    │                        │               │
│           ▼                    ▼                        ▼               │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │           PROJECT FILE STORAGE (./projects/<projectId>/)          │  │
│  │  project.json │ storyboard.json │ audio.mp3/wav │ subs.srt │ .mp4 │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### Alur Kerja End-to-End (14 Tahap)

1. Pengguna membuka **Dashboard Proyek** lalu klik **Create New Project**.
2. Pengguna mengisi nama proyek, memilih **Aspect Ratio**, **Resolution**, **FPS**, dan **Visual Style Preset**.
3. Pengguna mengunggah **Audio (MP3/WAV)** dan **Subtitle (SRT)**.
4. Server menjalankan `ffprobe` untuk membaca durasi audio presisi milidetik, serta mem-parsing SRT menjadi daftar segmen kalimat beserta timestamp validasinya.
5. Backend mengirim daftar segmen SRT + metadata proyek + katalog aset/pose prosedural yang tersedia ke **AI Director** (`gemini-4-hydrogen`).
6. AI mengelompokkan segmen SRT ke dalam **Scene Logis** (tanpa memutus kalimat) sambil menentukan mood, topik, kamera, transisi, background, karakter, objek, diagram, posisi, urutan kemunculan, dan animasi.
7. Server menormalisasi dan memvalidasi JSON storyboard (memastikan celah antar-scene tertutup rapat hingga akhir durasi audio).
8. Studio terbuka: **Drawing Engine** & **Animation Engine** langsung merender visual di `<canvas>` browser secara interaktif.
9. Audio HTML5 diputar sinkron dengan clock mesin animasi; pengguna melihat animasi *draw-on*, karakter bergerak, objek pop-up, kamera zoom/pan, dan subtitle ON/OFF.
10. Pengguna meninjau melalui **Scene Timeline** dan **Storyboard Editor**.
11. Pengguna dapat mengedit properti scene secara manual (ubah background, objek, animasi, kamera, durasi batas scene) atau klik **Regenerate Scene** untuk meminta AI menyusun ulang komposisi visual scene tersebut.
12. Pengguna menekan **Export Video (MP4)**.
13. Backend memicu **Render Job**, merender setiap frame deterministik `t = frameIndex / fps` memakai `@napi-rs/canvas` dan mengalirkannya langsung ke stdin `ffmpeg`, sambil mengirim update persentase & ETA ke UI via **Server-Sent Events (SSE)**.
14. Setelah muxing audio dan encoding H.264 selesai, pengguna dapat memutar hasil MP4 atau mengunduh file video final.

---

## 3. Pemilihan Teknologi & Alasan Teknis

| Lapisan | Teknologi Terpilih | Alasan Pemilihan |
| :--- | :--- | :--- |
| **Frontend UI** | React 19, TypeScript, Tailwind CSS v4, Lucide Icons, Motion | Sudah terpasang di workspace; komponen reaktif ideal untuk timeline scrubbing, inspector real-time, dan kontrol editor presisi tinggi. |
| **Backend Server** | Node.js v22, Express, `tsx`, `multer` | Arsitektur full-stack terintegrasi dengan Vite dev middleware pada port 3000; streaming file & SSE native tanpa kompleksitas tambahan. |
| **AI Director** | `@google/genai` (`gemini-4-hydrogen`) | Mendukung `responseSchema` terstruktur untuk menghasilkan Storyboard JSON valid secara konsisten, dengan API key terisolasi aman di server. |
| **Visual & Drawing Engine** | HTML5 Canvas 2D API + SVG `Path2D` | Kontrol penuh secara programatik per path, garis, kurva Bezier, transformasi matriks, dash-offset progressive drawing, dan komposisi layer. |
| **Server-Side Rasterizer** | `@napi-rs/canvas` (Skia native) | Implementasi Canvas 2D berkecepatan tinggi di Node.js (prebuilt Linux x64, tanpa dependency Cairo/GTK/Chromium), identik dengan Canvas browser. |
| **Video & Audio Encoder** | `ffmpeg` & `ffprobe` (CLI sistem) | Tersedia langsung di container (`/usr/bin/ffmpeg` 4.4.2 dengan `libx264` & `aac`); mendukung input `rawvideo` RGBA via pipe agar disk I/O tetap minimal. |
| **Progress Streaming** | Server-Sent Events (SSE) | Ringan, satu arah, andal untuk melaporkan progress frame rendering dan encoding FFmpeg secara real-time ke browser. |

---

## 4. Struktur Folder & Organisasi Modul

Struktur dipisahkan secara tegas menjadi tiga zona utama (`client`, `server`, `shared`) agar tidak ada file monolitik dan mesin grafis dapat digunakan ulang oleh Preview dan Renderer.

```text
/
├── server.ts                          # Entry point Express + Vite middleware + API mount
├── projects/                          # Penyimpanan persisten file proyek pengguna (gitignore)
│   └── <projectId>/
│       ├── project.json               # Metadata konfigurasi, status, path media
│       ├── storyboard.json            # Data storyboard terstruktur (source of truth)
│       ├── audio.<mp3|wav>            # Audio narasi asli pengguna
│       ├── subtitles.srt              # File SRT asli pengguna
│       └── output.mp4                 # Hasil render video final
│
├── shared/                            # Modul isomorfik (berjalan di Browser & Node.js)
│   ├── types/
│   │   ├── project.ts                 # Interface Project, VideoSettings, RenderJob
│   │   └── storyboard.ts              # Spesifikasi tipe Storyboard JSON, Scene, Layer, Animasi
│   ├── styles/
│   │   └── presets.ts                 # 6 Preset Visual (palet, ketebalan garis, wobble, font, paper texture)
│   ├── utils/
│   │   ├── srtParser.ts               # Parser SRT, validasi timestamp, grouping kalimat
│   │   └── easing.ts                  # Kurva easing (spring, bounce, cubic, Bezier path sampler)
│   └── engine/
│       ├── AnimationEngine.ts         # Evaluator state deterministik evaluateSceneAt(t)
│       ├── DrawingEngine.ts           # Renderer Canvas2D utama + kamera + transisi + subtitle
│       ├── primitives/
│       │   ├── pathUtils.ts           # Progressive draw-on (dash length), hand-drawn jitter/wobble
│       │   ├── backgrounds.ts         # Generator background kontekstual berlapis (parallax/props)
│       │   ├── characters.ts          # Rig karakter artikulasi (idle, walk, point, think, celebrate, explain)
│       │   ├── objects.ts             # Katalog 40+ objek/ikon/metafora komposit terprogram
│       │   └── diagrams.ts            # Diagram dinamis (flowchart, bar/line chart, mindmap, timeline, comparison)
│       └── index.ts                   # Facade renderFrame(ctx, storyboard, timeMs, options)
│
├── server/                            # Layanan backend khusus Node.js
│   ├── routes/
│   │   ├── projects.ts                # CRUD proyek & upload audio/SRT
│   │   ├── storyboard.ts              # Generate AI storyboard, update scene, regenerate 1 scene
│   │   └── render.ts                  # Trigger render MP4, SSE progress stream, cancel, download
│   ├── services/
│   │   ├── aiDirector.ts              # Integrasi @google/genai (gemini-4-hydrogen) + prompt sutradara + repair JSON
│   │   ├── mediaInspector.ts          # Wrapper ffprobe untuk durasi/sample rate audio
│   │   ├── projectStore.ts            # Read/write atomik ke ./projects/<id>/
│   │   └── videoRenderer.ts           # Loop frame @napi-rs/canvas -> pipe RGBA ke FFmpeg libx264+aac
│
└── src/                               # Frontend React Studio UI
    ├── App.tsx                        # Router tampilan (Dashboard vs Studio Workspace)
    ├── components/
    │   ├── TopBar.tsx                 # Header 3-zona (Wordmark, Navigasi proyek, Primary CTA Export)
    │   ├── DashboardView.tsx          # Grid/daftar proyek tersimpan + empty state + filter
    │   ├── CreateProjectModal.tsx     # Wizard upload Audio + SRT, pilih rasio/resolusi/FPS/style
    │   ├── StudioWorkspace.tsx        # Layout 3-panel Studio (Preview, Timeline, Inspector)
    │   ├── CanvasPreviewPlayer.tsx    # <canvas> live renderer + audio sync + scrub bar + subtitle toggle
    │   ├── SceneTimeline.tsx          # Track timeline proporsional durasi + klip scene + drag/select
    │   ├── StoryboardInspector.tsx    # Editor scene: edit narasi, style background, elemen, animasi, kamera, regenerate
    │   └── RenderExportModal.tsx      # Status render real-time via SSE, preview hasil MP4, dan download
    └── hooks/
        ├── useStudioPlayer.ts         # Sinkronisasi HTMLAudioElement <-> AnimationEngine
        └── useRenderStream.ts         # Subscriber SSE progress render
```

---

## 5. Rancangan UI/UX

Antarmuka dirancang bergaya **Editorial Production Studio** (gelap netral `#0F172A` / `#131B2E` untuk ruang kerja studio agar canvas video menjadi pusat fokus visual, dengan kontrol presisi bertipografi bersih `Plus Jakarta Sans` + `JetBrains Mono` untuk seluruh angka timecode).

### 5.1 Prinsip Tata Letak & Navigasi
- **Top Bar Contract (Satu Baris, Tiga Zona)**:
  - Kiri: Wordmark aplikasi + breadcrumb proyek aktif saat di dalam studio.
  - Tengah: Tab navigasi workspace (`Dashboard`, `Storyboard`, `Timeline`, `Export`).
  - Kanan: Satu tombol aksi utama kontekstual (`+ New Project` di Dashboard, atau `Export MP4` di dalam Studio).
- **Zero-Pill & Anti-Clutter Discipline**: Metadata durasi, rasio, FPS, dan jumlah scene ditampilkan sebagai teks tabular bersih dengan pemisah `·` (contoh: `16:9 · 1080p · 30 FPS · 02:14`). Tidak ada status dot berkedip palsu atau log telemetri hiasan.

### 5.2 Peta Layar & Interaksi Utama

1. **Halaman Dashboard Proyek**:
   - Header ringkas + filter status/rasio + tombol `New Project`.
   - Daftar proyek menampilkan thumbnail preview frame scene pertama yang dirender langsung oleh Drawing Engine, judul proyek, metadata audio/durasi, style preset, dan aksi cepat (`Open Studio`, `Duplicate`, `Delete`).
   - Empty state edukatif bila belum ada proyek.
2. **Modal / Wizard Create New Project**:
   - **Langkah 1 — Sumber Media**: Dropzone terpisah untuk Audio (`MP3/WAV`) dan Subtitle (`SRT`). Begitu diunggah, aplikasi langsung memvalidasi dan menampilkan durasi audio, jumlah baris subtitle, serta peringatan jika durasi SRT jauh melebihi durasi audio.
   - **Langkah 2 — Konfigurasi Visual & Video**: Pemilih Visual Style dengan *live mini-canvas thumbnail* untuk 6 preset, Aspect Ratio (`16:9`, `9:16`, `1:1`), Resolusi (`720p`, `1080p`), FPS (`24`, `30`, `60`), serta toggle Subtitle (`ON` / `OFF`).
   - Tombol `Analyze & Build Storyboard` memicu AI Director dan menampilkan progres tahap demi tahap (Upload -> Audio Analysis -> AI Scene Direction -> Visual Assembly).
3. **Studio Workspace (3-Panel Terintegrasi)**:
   - **Panel Atas-Kiri (65% lebar): Live Preview Stage**:
     - Viewport `<canvas>` yang otomatis menyesuaikan rasio (`16:9`, `9:16`, atau `1:1`) di dalam area panggung gelap.
     - Kontrol playback presisi: Play/Pause, Step Frame, Loop Scene Ini vs Play Full Video, Volume, Toggle Subtitle ON/OFF, serta tombol `Safe Margins / Grid`.
   - **Panel Kanan (35% lebar): Scene & Storyboard Inspector**:
     - Menampilkan detail scene aktif dengan tab terstruktur:
       - *Narasi & Timing*: Teks subtitle yang terikat pada scene ini, timestamp mulai–selesai (dapat disesuaikan tanpa celah antar-scene).
       - *Visual & Background*: Pilih tipe background, mood warna, karakter (aksi & ekspresi), objek utama/pendukung, serta diagram/teks highlight.
       - *Animasi & Kamera*: Atur efek masuk/keluar per elemen (`draw`, `pop`, `slide_left`, `bounce`, `fade`), gerakan kamera (`zoom_in`, `zoom_out`, `pan_left`, `pan_right`, `ken_burns_subtle`), dan transisi scene.
       - *Aksi Cepat Scene*: Tombol `Preview Scene` dan `Regenerate Scene with AI` (dengan kolom opsional catatan instruksi pengguna, misal: *"Ganti menjadi diagram alur 3 langkah dengan karakter menunjuk papan"*).
   - **Panel Bawah (Lebar Penuh): Multi-Track Scene Timeline**:
     - Skala waktu tabular (`00:00` s/d durasi audio).
     - Track 1: **Waveform / Audio Bar** referensi narasi.
     - Track 2: **Blok Scene** proporsional durasi dengan label judul scene, mood, dan indikator visual aktif; klik untuk melompat ke scene.
     - Track 3: **Segmen Subtitle SRT** di bawah blok scene agar pengguna dapat memverifikasi sinkronisasi kalimat secara visual.
4. **Modal Render Progress & Export MP4**:
   - Ringkasan spesifikasi akhir (`1920x1080 · 30 FPS · Audio AAC · Subtitle ON`).
   - Progress bar aktual (`Frame 1.420 / 3.600 · Encoding MP4 · Estimasi sisa 18 dtk`), tombol `Cancel Render`, dan pemutar `<video>` + tombol `Download MP4` saat selesai.

---

## 6. Spesifikasi Storyboard JSON

Storyboard JSON adalah kontrak data tunggal antara **AI Director**, **Storyboard Editor**, **Animation Engine**, dan **Rendering Engine**. Semua koordinat menggunakan sistem ternormalisasi **`0.0` – `1.0`** relatif terhadap kanvas aktif sehingga satu storyboard otomatis beradaptasi ke `16:9`, `9:16`, maupun `1:1` (dengan *safe-zone layout helper* per rasio).

```json
{
  "version": "1.0",
  "projectId": "proj_8f92a1",
  "title": "Cara Kerja Energi Surya",
  "settings": {
    "aspectRatio": "16:9",
    "resolution": "1080p",
    "fps": 30,
    "stylePreset": "educational_explainer",
    "subtitlesEnabled": true,
    "totalDurationMs": 45200
  },
  "consistencyProfile": {
    "palette": {
      "backgroundBase": "#F8FAFC",
      "primary": "#0F172A",
      "accent": "#2563EB",
      "secondary": "#F59E0B",
      "highlight": "#10B981"
    },
    "protagonist": {
      "id": "char_guide",
      "archetype": "educator",
      "hairStyle": "short_modern",
      "outfitColor": "#2563EB",
      "skinTone": "#E8B89A"
    }
  },
  "scenes": [
    {
      "sceneId": "scene_01",
      "index": 0,
      "startMs": 0,
      "endMs": 8400,
      "subtitleIndices": [1, 2],
      "narrationText": "Setiap hari, matahari memancarkan energi yang cukup untuk memenuhi kebutuhan listrik seluruh dunia.",
      "summary": "Pembukaan dengan ilustrasi matahari bersinar di atas kota modern dan panel surya.",
      "mood": "inspiring",
      "background": {
        "type": "sky_city_horizon",
        "skyTone": "morning_bright",
        "elements": ["clouds_drifting", "distant_city_silhouette", "rolling_hills"]
      },
      "camera": {
        "type": "zoom_in",
        "startZoom": 1.0,
        "endZoom": 1.12,
        "startPan": { "x": 0.0, "y": 0.0 },
        "endPan": { "x": 0.03, "y": -0.02 },
        "easing": "easeInOutCubic"
      },
      "transitionIn": { "type": "fade", "durationMs": 400 },
      "transitionOut": { "type": "slide_left", "durationMs": 450 },
      "elements": [
        {
          "id": "el_sun",
          "role": "main_object",
          "kind": "object",
          "assetType": "sun_radiant",
          "position": { "x": 0.22, "y": 0.24 },
          "scale": 1.15,
          "zIndex": 2,
          "enterAtMs": 300,
          "enterDurationMs": 900,
          "enterAnimation": "draw_then_pulse",
          "idleAnimation": "rotate_slow_rays",
          "exitAnimation": "fade_out"
        },
        {
          "id": "el_panel",
          "role": "supporting_object",
          "kind": "object",
          "assetType": "solar_panel_array",
          "position": { "x": 0.62, "y": 0.68 },
          "scale": 1.1,
          "zIndex": 3,
          "enterAtMs": 1400,
          "enterDurationMs": 1100,
          "enterAnimation": "draw",
          "idleAnimation": "subtle_shimmer",
          "motionPath": null
        },
        {
          "id": "el_char",
          "role": "character",
          "kind": "character",
          "characterId": "char_guide",
          "action": "walk_in_and_point",
          "expression": "enthusiastic",
          "facing": "right",
          "pointTarget": { "x": 0.62, "y": 0.60 },
          "position": { "x": 0.32, "y": 0.72 },
          "scale": 1.0,
          "zIndex": 4,
          "enterAtMs": 800,
          "enterDurationMs": 900,
          "enterAnimation": "slide_left",
          "idleAnimation": "breathing_gestures"
        },
        {
          "id": "el_photon_flow",
          "role": "supporting_object",
          "kind": "particle_stream",
          "assetType": "energy_particles",
          "from": { "x": 0.26, "y": 0.28 },
          "to": { "x": 0.60, "y": 0.62 },
          "zIndex": 3,
          "enterAtMs": 2200,
          "enterDurationMs": 1800,
          "enterAnimation": "motion_path_flow"
        },
        {
          "id": "el_callout_text",
          "role": "kinetic_text",
          "kind": "text",
          "text": "173.000 Terawatt Energi Surya",
          "variant": "highlight_badge",
          "position": { "x": 0.64, "y": 0.26 },
          "zIndex": 5,
          "enterAtMs": 3200,
          "enterDurationMs": 600,
          "enterAnimation": "pop_up",
          "idleAnimation": "float_gentle"
        }
      ]
    }
  ]
}
```

### Aturan Validasi Skema Storyboard
- `scene[0].startMs === 0`, `scene[i].endMs === scene[i+1].startMs`, dan `scene[last].endMs === totalDurationMs`.
- Setiap `subtitle` pada file SRT harus dimiliki tepat oleh satu `sceneId` secara berurutan tanpa pemecahan kalimat di tengah segmen.
- Setiap `assetType` divalidasi terhadap registri komponen di `DrawingEngine`; jika AI mengusulkan nama objek tak dikenal, mapper semantik memetakannya ke objek terdekat atau *compound shape builder* dinamis berbasis ikon + label agar tidak pernah ada scene kosong.

---

## 7. Rancangan Drawing Engine

Drawing Engine dibangun berbasis **Canvas 2D + SVG Path2D Vector Engine** sehingga setiap garis, bentuk, karakter, background, dan diagram digambar secara programatik per frame—bukan memuat gambar bitmap statis.

### 7.1 Arsitektur 6 Style Preset Konsisten

Setiap preset mengendalikan parameter rendering global pada `DrawingContext`:

| Style Preset | Permukaan & Kertas | Karakteristik Garis | Pewarnaan & Isian |
| :--- | :--- | :--- | :--- |
| `hand_drawn` | Kertas tekstur hangat grain tipis (`#FAF8F4`) | Garis organik multi-pass (*subtle wobble* deterministik per seed) + `round` cap | Cat air datar transparan (*soft offset fill*) di balik outline tinta |
| `whiteboard` | Papan putih bersih dengan sudut halus (`#FFFFFF`) | Garis spidol tebal konsisten (`3.5px`), ujung membulat + simulasi ujung spidol aktif saat *draw-on* | Minimalis hitam/abu dengan aksen warna spidol biru/merah/hijau terbatas |
| `minimalist_cartoon` | Latar pastel lembut terkurasi | Outline bersih tebal (`3px`) dengan kontras tinggi dan sudut ekspresif | Blok warna solid cerah + bayangan sel (*cel-shading* 2 tingkat) |
| `educational_explainer` | Kanvas studio netral (`#F1F5F9` / `#0F172A` adaptif mood) | Garis geometris presisi (`2.2px`), sudut terukur, aksen grid tipis | Palet semantik terstruktur untuk kejelasan informasi & diagram |
| `sketch_animation` | Kertas sketsa bertekstur (*crosshatch*) | Garis pensil tipis berlapis + arsiran diagonal (*hatching*) dinamis | Monokrom grafit dengan 1 warna sorotan (*selective color accent*) |
| `flat_illustration` | Latar gradien lembut berlapis (*layered depth*) | Tanpa outline luar (*borderless vector*) atau stroke sangat halus | Bentuk geometris modern dengan gradien tonal & bayangan lembut |

### 7.2 Sistem Penggambaran Prosedural Berbasis Layer

1. **Progressive Stroke Draw-On (`drawProgress: 0..1`)**:
   - Setiap elemen vektor mendefinisikan urutan sub-path (`Path2D` / array titik kurva Kubik Bezier).
   - Mesin menghitung total panjang segmen lalu memotong panjang gambar memakai `ctx.setLineDash([drawnLength, totalLength])` dan menggambar *fill* secara bertahap (`fade-in` halus dari `progress 0.65 -> 1.0`) setelah kerangka garis hampir selesai—menghasilkan efek nyata "ilustrasi digambar dari nol".
   - Khusus mode `whiteboard` dan `sketch_animation`, koordinat ujung pena aktif (`penTipPos`) diekspor untuk merender ikon tangan/pena/spidol kecil yang mengikuti ujung garis secara presisi.
2. **Rig Karakter Artikulasi (`CharacterRig`)**:
   - Karakter bukan gambar diam, melainkan struktur hierarki sendi (*joint hierarchy*): `pelvis -> torso -> head (eyes, brows, mouth) + leftArm/rightArm (upper, forearm, hand) + leftLeg/rightLeg`.
   - Mendukung arketipe (*educator, tech_worker, student, scientist, entrepreneur*) yang mengunci proporsi, warna rambut, dan pakaian di seluruh video (`consistencyProfile`).
   - Aksi terprogram: `idle_breathe`, `walk` (siklus langkah sinusoid + ayunan lengan), `point` (lengan mengarah ke koordinat `pointTarget` objek/diagram), `explain` (gestur tangan bergantian + ekspresi bicara), `think` (tangan di dagu + ikon gelembung pikiran), `react_surprised`, `celebrate`, dan `sit_at_desk`.
3. **Katalog Objek Komposit & Metafora Visual (`40+ Procedural Assets`)**:
   - Dibangun dari gabungan bentuk primitif (rect, arc, polygon, Bezier): Bumi/globe berputar, roket meluncur, lampu ide menyala, jam berdetak, buku terbuka, laptop/kode, server/cloud, kaca pembesar, roda gigi saling bertaut, kunci/perisai keamanan, kota, pepohonan, matahari, baterai terisi, uang/grafik naik, otak/jaringan saraf, dll.
   - Setiap objek memiliki *sub-parts* yang dapat dianimasikan terpisah (mis. baling-baling berputar, jarum jam bergerak, api roket berdenyut).
4. **Diagram & Infografis Terprogram**:
   - `flowchart`: Node kotak/lingkaran muncul berurutan dihubungkan panah yang tergambar bertahap.
   - `bar_chart` & `line_chart`: Sumbu X/Y tergambar lebih dulu, lalu batang tumbuh / kurva garis meluncur naik sesuai narasi.
   - `comparison_split`: Kanvas terbelah dua dengan ikon kontras (`Before vs After` atau `A vs B`).
   - `cycle_diagram` & `timeline_steps`: Langkah 1-2-3-4 berotasi atau menyala mengikuti progres waktu scene.
5. **Background Kontekstual Berlapis**:
   - Latar belakang tidak pernah polos kosong: selalu terdiri atas lapisan *Backdrop Gradient/Paper* + *Midground Environment* (kantor, laboratorium, langit & kota, ruang angkasa, papan tulis kelas, lanskap alam, jaringan abstrak geometris) dengan gerakan *parallax* halus saat kamera melakukan pan/zoom.

---

## 8. Rancangan Animation Engine

Animation Engine bersifat **100% Deterministik Berbasis Waktu** (`Pure Function: state = evaluateAt(storyboard, timeMs)`), bukan berbasis akumulasi `dt` real-time. Hal ini menjamin:
- Scrubbing mundur/maju di browser selalu menampilkan posisi animasi yang presisi.
- Frame ke-`N` di browser dan frame ke-`N` di FFmpeg render server selalu identik.

### 8.1 Pipeline Evaluasi Per Frame (`timeMs`)

```text
timeMs ──► Temukan Scene Aktif (& Scene Sebelumnya jika dalam window Transisi)
             │
             ├──► Hitung localTimeMs = timeMs - scene.startMs
             ├──► Evaluasi Kamera Scene (Zoom + Pan + Subtle Drift) -> Matriks Transformasi
             ├──► Evaluasi Transisi Background & Transisi Antar-Scene (Crossfade / Wipe / Slide / Iris)
             ├──► Untuk setiap Elemen (diurutkan berdasarkan zIndex):
             │      ├── Fase Masuk (enterAtMs .. enterAtMs + enterDurationMs)
             │      │     └─ draw | pop_up | fade_in | slide_{dir} | scale_up | rotate_in | bounce_in
             │      ├── Fase Aktif / Idle (setelah masuk s/d akhir scene)
             │      │     └─ float_gentle | pulse | rotate_continuous | character_cycle | motion_path
             │      └── Fase Keluar (jika elemen punya exitAtMs sebelum scene berakhir)
             │            └─ fade_out | slide_out | scale_down
             └──► Evaluasi Lapisan Kinetic Text & Subtitle Overlay (jika subtitlesEnabled = true)
```

### 8.2 Spesifikasi Kemampuan Animasi

- **Draw Animation**: Interpolasi progres garis `0 -> 1` menggunakan kurva `easeInOutCubic` diikuti opacity fill.
- **Pop-Up & Bounce Ringan**: Kurva pegas (*back-out / spring overshoot* `scale: 0 -> 1.12 -> 0.96 -> 1.0`) agar objek terasa hidup dan kenyal.
- **Slide In / Out**: Pergerakan dari luar batas *viewport* (kiri, kanan, atas, bawah) dengan *easeOutQuart* + sedikit *squash-and-stretch* saat berhenti.
- **Motion Path**: Evaluator Kurva Kubik Bezier (`P0, P1, P2, P3`) + orientasi tangen opsional (mis. pesawat/roket/partikel energi mengikuti lintasan melengkung dari objek A ke objek B).
- **Kamera (Zoom & Pan)**: Transformasi matriks global pada layer *World Space* (di bawah layer Subtitle agar teks subtitle selalu stabil terbaca di layar). Mendukung `zoom_in`, `zoom_out`, `pan_left`, `pan_right`, `focus_target` (kamera menggeser dan memperbesar ke objek yang sedang dibahas narator).
- **Scene Transitions**: `fade`, `cross_dissolve`, `slide_left`, `slide_up`, `wipe_diagonal`, dan `iris_circle` dengan cara merender *dual-buffer* singkat selama `transitionDurationMs` (350–550ms) di batas pergantian scene.
- **Kinetic Typography**: Teks kata kunci penting muncul dengan animasi per kata (*staggered pop-up*) atau efek *highlighter sweep* di belakang teks.

---

## 9. Rancangan Rendering Engine (MP4 Pipeline)

Rendering video final dilakukan sepenuhnya di **Backend Node.js** menggunakan `@napi-rs/canvas` dan `ffmpeg` tanpa menyimpan ribuan file gambar sementara ke disk.

### 9.1 Resolusi & Dimensi Output

| Aspect Ratio | 720p (`WxH`) | 1080p (`WxH`) |
| :--- | :--- | :--- |
| `16:9` (Landscape) | `1280 x 720` | `1920 x 1080` |
| `9:16` (Portrait / Reels / Shorts) | `720 x 1280` | `1080 x 1920` |
| `1:1` (Square) | `720 x 720` | `1080 x 1080` |

### 9.2 Pipeline Pipe Langsung (`@napi-rs/canvas` -> `ffmpeg`)

1. **Inisialisasi Kanvas & Proses FFmpeg**:
   - Buat 1 instance kanvas Skia (`createCanvas(width, height)`) dan kanvas cadangan untuk transisi antar-scene.
   - Spawn proses `ffmpeg`:
     ```bash
     ffmpeg -y \
       -f rawvideo -pix_fmt rgba -s {W}x{H} -r {FPS} -i pipe:0 \
       -i {AUDIO_PATH} \
       -c:v libx264 -preset medium -crf 18 -pix_fmt yuv420p \
       -c:a aac -b:a 192k \
       -shortest -movflags +faststart \
       {OUTPUT_MP4_PATH}
     ```
2. **Loop Frame Deterministik + Kontrol Backpressure**:
   - Total frame dihitung tepat: `totalFrames = Math.ceil((audioDurationMs / 1000) * fps)`.
   - Untuk setiap `frame = 0 .. totalFrames - 1`:
     - Hitung `timeMs = (frame / fps) * 1000`.
     - Panggil `renderFrame(ctx, storyboard, timeMs, renderOptions)` dari modul `shared/engine`.
     - Ekstrak buffer piksel mentah (`canvas.data()` / `ctx.getImageData` -> RGBA `Uint8Array`).
     - Tulis ke `ffmpeg.stdin.write(buffer)`. Jika `write()` mengembalikan `false` (buffer penuh), tunggu event `'drain'` sebelum merender frame berikutnya agar penggunaan RAM konstan (~150–250 MB) tidak melonjak untuk video berdurasi panjang.
3. **Pelaporan Progress & Kontrol Pembatalan**:
   - Setiap 10–15 frame, kirim event SSE `{ phase: "rendering", frame, totalFrames, percent, etaMs }` ke client.
   - Jika pengguna menekan `Cancel Render`, loop berhenti, `ffmpeg.kill('SIGTERM')` dipanggil, dan file parsial dibersihkan.
4. **Sinkronisasi Audio & Subtitle**:
   - Subtitle dirender langsung di atas kanvas dalam *safe-zone* bawah dengan *contrast scrim* bersih apabila `subtitlesEnabled === true` (memastikan tampilan subtitle di preview dan MP4 100% sama, tanpa bergantung pada filter `libass` eksternal).
   - Durasi video dikunci persis pada durasi audio dari `ffprobe` (`-af apad` jika perlu atau presisi total frame sesuai `audioDurationMs`) agar kalimat terakhir narasi tidak pernah terpotong.

---

## 10. Identifikasi Keterbatasan Teknologi & Risiko

| Risiko / Keterbatasan | Dampak Jika Diabaikan | Strategi Mitigasi Teknis |
| :--- | :--- | :--- |
| **1. Konsistensi & Validitas Output AI Storyboard** | AI bisa menghasilkan nama aset fiktif, timestamp tumpang tindih, atau memotong kalimat narasi di tengah. | Gunakan `responseSchema` ketat di `@google/genai`, berikan daftar enum aset/pose yang valid dalam system prompt, lalu jalankan *Deterministic Storyboard Sanitizer* di backend untuk menutup celah timestamp, merapatkan durasi ke `audioDurationMs`, dan fallback ke komposisi objek/diagram valid jika ada properti tidak cocok. |
| **2. Performa Render & Penggunaan Memori pada Video Panjang (1080p @ 60fps)** | 3 menit 1080p60 = 10.800 frame. Menyimpan PNG ke disk menghabiskan gigabyte storage dan memperlambat I/O; tanpa backpressure RAM bisa OOM. | Gunakan streaming `rawvideo` RGBA langsung ke `stdin` FFmpeg tanpa menulis frame PNG ke disk, pakai ulang 1–2 buffer canvas, dan terapkan mekanisme `stdin.drain` backpressure. Gunakan `-preset veryfast` atau `medium` yang seimbang serta `60fps` sebagai opsi sadar kecepatan. |
| **3. Ketersediaan Font antara Browser & Linux Container (`@napi-rs/canvas`)** | Jika font di browser berbeda dengan font sistem Linux di server, posisi wrapping teks atau kinetic typography di MP4 bisa bergeser. | Bundel file font `.ttf` lokal (mis. `Plus Jakarta Sans` dan `JetBrains Mono`) di direktori `shared/assets/fonts/`, daftarkan ke `@napi-rs/canvas` via `GlobalFonts.registerFromPath()` di server dan `@font-face` di CSS browser agar metrik teks (`measureText`) identik. |
| **4. Monotonisasi Visual untuk Narasi Panjang (> 5 menit)** | Scene yang terlalu mirip secara berurutan terasa membosankan atau seperti slideshow. | AI Director diwajibkan menerapkan *Visual Pacing Rules*: larang 2 scene berturut-turut memakai tipe background + gerakan kamera yang sama; variasikan komposisi (karakter+objek -> diagram alur -> close-up metafora -> split comparison). |
| **5. Ketidakcocokan Durasi Audio vs File SRT** | File SRT pengguna mungkin memiliki timestamp akhir melebihi durasi MP3/WAV atau memiliki celah hening panjang di awal/akhir. | Saat upload, `mediaInspector` membandingkan durasi audio `ffprobe` vs timestamp akhir SRT, menampilkan warning informatif di UI, serta otomatis meng-clamp timestamp scene terakhir agar tepat sinkron dengan durasi audio asli. |
| **6. Batas Timeout Koneksi / Long-Running Render** | Request HTTP biasa akan timeout jika rendering berlangsung > 60 detik. | Endpoint `POST /api/projects/:id/render` langsung mengembalikan `202 Accepted` berisi `jobId` dan berjalan asynchronous di background; frontend memantau progres melalui `GET /api/render/:jobId/stream` (SSE). |

---

## Rencana Tahapan Implementasi (Setelah Blueprint Disetujui)

Apabila rancangan di atas sudah disetujui, pembangunan akan dilakukan secara bertahap dan terverifikasi pada setiap fase:
1. **Fase 1 — Fondasi Proyek, Media Upload & SRT Parser**: Setup server Express + `tsx`, penyimpanan proyek `./projects/`, upload audio+SRT, integrasi `ffprobe`, dan parser SRT.
2. **Fase 2 — Shared Storyboard Schema, Drawing Engine & Animation Engine**: Implementasi 6 style preset, progressive draw-on, karakter artikulasi, 40+ objek & diagram, kamera, transisi, dan evaluator deterministik `evaluateSceneAt(t)`.
3. **Fase 3 — AI Director (`gemini-4-hydrogen`)**: Implementasi analisis narasi + SRT menjadi Storyboard JSON terstruktur + fitur regenerate per scene.
4. **Fase 4 — Studio Frontend UI**: Dashboard, Create Project Wizard, Studio Workspace (Canvas Live Preview sinkron audio, Timeline multi-track, Storyboard Inspector lengkap).
5. **Fase 5 — Rendering Engine MP4 & Verifikasi End-to-End**: Integrasi `@napi-rs/canvas` -> FFmpeg pipe, SSE progress, export & download MP4, serta uji coba kompilasi dan alur kerja lengkap.
