# bus-density-cv

Penghitung penumpang bus TransJakarta berbasis computer vision untuk proyek
capstone *bus density routing*. Repo ini terpisah dari aplikasi
(`transjakarta-app`: backend FastAPI + frontend Next.js) karena dependensinya
berat (torch, ultralytics, OpenCV) dan berjalan di mesin yang berbeda.

## Alur data

```
ESP32-S3 CAM --(MQTT, JPEG biner)--> HiveMQ --> worker/mqtt_subscriber.py
                                                     |  LPUSH
                                                     v
                                         Redis  cv_task_queue:<bus_id>
                                                     |  BRPOP
                                                     v
                                          worker/cv_worker.py (YOLO)
                                                     |
                                  headcount -> load factor -> label kepadatan
```

- `mqtt_subscriber.py` menerima payload JPEG mentah dari topik MQTT, menyimpan
  salinannya ke `received_images/`, lalu mendorongnya ke antrean Redis.
- `cv_worker.py` mengambil dari antrean, menjalankan YOLO (kelas `person`),
  menyaring deteksi (area dan rasio), lalu menghitung load factor
  `headcount / MAX_CAPACITY` (kapasitas 80) dan label SEPI / SEDANG / PADAT /
  SANGAT PADAT. Latensi tiap tahap dicatat ke `cv_latency_log_binary.csv`.

> **Belum terhubung ke backend.** Hasil worker saat ini hanya dicetak sebagai
> `[SIMULASI DB]` dan ditulis ke CSV. Belum ada yang menulisnya ke Supabase atau
> memanggil API `transjakarta-app`. Payload yang disiapkan:
> `{"bus_id", "jumlah_penumpang", "kepadatan"}`.

## Struktur

| Folder | Isi |
| --- | --- |
| `worker/` | `mqtt_subscriber.py`, `cv_worker.py`, bobot `yolo26n.pt`, log latensi |
| `experiments/` | `headcount.py`, `headcount_sweep_test.py` (sweep conf/iou/imgsz), `headcount_video.py`, `dataset_deduplicator.py`, skrip latensi MQTT / INA226, `ground_truth.csv`, `best.pt`, hasil sweep, notebook training |

## Menjalankan

Butuh Python 3.10+, Redis di `localhost:6379`, dan [Git LFS](https://git-lfs.com)
(jalankan `git lfs install` sebelum clone, kalau tidak file `.pt` hanya berisi
teks pointer).

```bash
python -m venv .venv
.venv\Scripts\activate          # Windows
pip install -r requirements.txt

cd worker
python mqtt_subscriber.py       # terminal 1
python cv_worker.py             # terminal 2
```

Konfigurasi (broker, topik, `BUS_ID`, `MODEL_PATH`, ambang YOLO) berupa
konstanta di bagian atas tiap file. Path model dan CSV bersifat relatif ke
direktori kerja, jadi jalankan dari dalam `worker/`.

Kedua proses harus memakai nama antrean yang sama. Saat ini `mqtt_subscriber.py`
memakai `cv_task_queue:Koridor_Test_Lokal` sedangkan `cv_worker.py` memakai
`cv_task_queue:Koridor_1_A`, jadi samakan salah satunya sebelum dijalankan
bersamaan.

Skrip di `experiments/` dijalankan dari dalam folder tersebut. Gambar uji
(`dataset_test/`) tidak ikut di-commit.
