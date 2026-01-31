# PLANS.md — Uygulama Yeniden Yazım Planı (v1)

> **Source of truth:** `README_single.md`  
> **FSM geçişleri:** `FSM_SPEC_EXTRACT.md`  
> Bu plan, Codex CLI ile iteratif geliştirme için “yaşayan doküman” olarak tutulmalıdır.

---

## 0) Başarı Kriterleri (Definition of Done)

Aşağıdakiler sağlanınca v1 “tamamlandı” kabul edilir:

- [ ] **Tek kod tabanı** ile `cpu | hailo | imx500` inference backend seçimi yapılabiliyor (config üzerinden).
- [ ] Backend seçimi **IDLE state’te** güvenli biçimde uygulanıyor (hot-swap yok; gerekirse clean restart).
- [ ] **A.YOLU (QR)** akışı eksiksiz çalışıyor:
  - QR payload `1–63` integer parse/validate
  - bit mask → kapak kuyruğu (fixed priority)
  - `unlock_mode = SEQUENTIAL | SIMULTANEOUS` + unlock parametreleri: `unlock_pulse_duration_sec / inter_door_delay_sec / inter_session_delay_sec / unlock_stagger_sec`
  - session queue (aktif oturum varken gelen QR sıraya alınır)
- [ ] **B.YOLU (Touchscreen + Camera + AI)** akışı çalışıyor:
  - Screen akışı, confidence eşikleri, stabil frame şartları
  - Retry timeout politikası (count yok): `retry_inactivity_timeout_sec` + `max_retry_time_sec` + `unlocked_remove_timeout_sec` ve uyku (idle sleep) davranışı
- [ ] UI donmuyor (UI thread bloklanmıyor), inference ayrı yürütülüyor.
- [ ] Kalıcı kayıt (logging/persist) var: minimum event log + karar log.
- [ ] Testler:
  - FSM geçişleri için birim testler
  - QR decode/queue için birim testler
- [ ] Raspberry Pi 5 üzerinde “CPU backend” ile **uçtan uca** demo alınabiliyor.
- [ ] Hailo ve/veya IMX500 backend’leri için en azından:
  - adapter iskeleti + “device present / not present” durumunda düzgün degrade (fallback) davranışı
  - (mümkünse) gerçek inference demo

---



---

## 0.1) Dokümantasyon “Kilitleme” İşleri (Coding’e başlamadan önce)

Bu kısım, **kod yazmaya başlamadan önce** belirsizlikleri azaltmak için eklenmiştir.

- [ ] README_single.md içine **Data Flow + Dependency Rules + Dependency/Library Policy** bölümü eklendi (Mermaid diyagramları dahil).
- [ ] `docs/adr/` klasörü açıldı ve en kritik 3 karar için ADR yazıldı:
  - [ ] Kamera kütüphanesi seçimi (Picamera2 vs alternatif)
  - [ ] CPU inference runtime seçimi (Ultralytics/ONNXRuntime/OpenCV DNN)
  - [ ] Log formatı + saklama politikası (JSONL vs sqlite)
- [ ] `pyproject.toml` bağımlılık grupları tanımlandı (`dev`, opsiyonel backends).
- [ ] (Öneri) `pip-tools` ile lock’lı requirements akışı dokümante edildi ve CI’da çalışır hale getirildi.

**Kabul kriteri:** Dokümanlar “tek bakışta” şu soruları cevaplıyor:
- “Data flow nedir, kim kimi çağırır?”
- “Domain hangi katmanlara bağımlı olabilir?”
- “Pi üzerinde kurulum nasıl deterministik kalır?”


## 1) Kapsam (Scope)

### v1 kapsam içi
- Hexagonal + event-driven FSM
- PySide6 UI (B.YOLU ekranları)
- QR HID reader (A.YOLU)
- GPIO/relay sürme (unlock pulse) + güvenlik kuralları
- 3 backend mimarisi (CPU kesin; Hailo/IMX500 opsiyonel ama “tak-çıkar”)

### v1 kapsam dışı (şimdilik)
- Telefon uygulaması (Flutter) — dış bağımlılık
- Barkod→tür mapping yönetimi — dış bağımlılık
- “Görsel kayıt (image logging)” varsayılan açık değil
- Bulut / uzak telemetri (lokal log yeter)

---

## 2) Mimari İş Paketleri (Workstreams)

### WS-A: Repo & Geliştirici Deneyimi
- [ ] `pyproject.toml` + bağımlılıklar (PySide6, pytest, ruff, vs.)
- [ ] `config/app.toml` loader (tek kaynak)
- [ ] logging altyapısı (console + dosya; rotate opsiyonel)
- [ ] tek komutla çalıştırma: `python -m src.main` veya entrypoint

**Kabul kriteri:** boş iskelet çalışır, config okunur, log atar.

---

### WS-B: Domain (FSM + Event Model)
- [ ] Event tipleri: `QrScanned`, `InferenceResult`, `Timeout`, `ButtonPressed`, `HardwareFault`, ...
- [ ] State enum / değer nesneleri
- [ ] FSM motoru (senin yenilediğin FSM burada yaşayacak)
- [ ] Timer / timeout kavramı için `ClockPort` + deterministic test clock

**Kabul kriteri:** FSM, dış dünya olmadan “event replay” ile simüle edilebilir.

---

### WS-C: App Layer (Use-cases + Event Bus)
- [ ] Basit bir event bus / dispatcher (thread-safe)
- [ ] Use-case orkestrasyonu:
  - QR session queue runner
  - Inference lifecycle (start/stop)
  - Relay unlock orchestrator
  - UI update publisher

**Kabul kriteri:** Domain event → use-case → adapter çağrısı zinciri “tek noktadan” yönetilir.

---

### WS-D: Ports (Arayüzler)
- [ ] `InferencePort` (classify / start / stop / health)
- [ ] `CameraPort` (frame stream veya snapshot)
- [ ] `GpioRelayPort` (pulse/open)
- [ ] `QrScannerPort` (line/packet stream)
- [ ] `StoragePort` (append log, persist session summary)
- [ ] `TelemetryPort` (opsiyonel)

**Kabul kriteri:** Domain hiçbir adapter import etmez; sadece ports görür.

---

### WS-E: Adapters — QR (A.YOLU)
- [ ] USB-HID okuyucudan text buffer (Enter/newline olabilir; ignore trailing whitespace)
- [ ] `1–63` integer validation
- [ ] bitmask → lids queue (fixed order)
- [ ] session queue + “aktif oturum varken gelen QR” davranışı
- [ ] unlock: `unlock_mode=SEQUENTIAL` (pulse + `inter_door_delay_sec`) | `unlock_mode=SIMULTANEOUS` (`unlock_stagger_sec` ile opsiyonel kademeli tetikleme)

**Kabul kriteri:** gerçek QR okuyucuyla + test input ile aynı davranış.

---

### WS-F: Adapters — GPIO/Relay (Solenoid unlock)
- [ ] GPIO pin mapping (README_single.md’deki tabloya göre)
- [ ] “pulse” implementasyonu: süre dolunca otomatik kapanış
- [ ] Güvenlik: aynı röle kanalına çakışan pulse gelmez (per-channel lock/queue). Aynı anda birden çok röle tetikleme mümkündür (örn. `unlock_mode=SIMULTANEOUS`).
- [ ] “dry-run/mock” modu (hardware yokken geliştirme)

**Kabul kriteri:** mock modda test; gerçek hardware’de tek röle pulse doğrulaması.

---

### WS-G: UI (PySide6) — B.YOLU
- [ ] Ekran akışları (Screen1/2/3 vs.) ve event üretimi
- [ ] UI thread non-blocking: inference sonuçları signal/slot ile UI’ya geçer
- [ ] UX detayları:
  - “Low confidence” durumunda Screen2 seçim
  - Stabil frame şartı sağlandığında onay
  - Retry limitleri ve timeout mesajları
- [ ] “Service/Diagnostics” sayfası (backend seçimi, health, log path)

**Kabul kriteri:** CPU backend mock/gerçek ile ekranlar arası geçiş kararlı.

---

### WS-H: Inference Backends (3 seçenek)

#### H1 — CPU backend (önce bunu bitir)
- [ ] Model loader (Ultralytics / ONNXRuntime / OpenCV DNN — seçim yapılacak)
- [ ] Input preprocessing standardı (resize, normalize)
- [ ] Output standardı: `class_id`, `class_name`, `confidence`, `top_k`
- [ ] Performans: FPS limitleme, frame drop politikası

**Kabul kriteri:** CPU’da çalışan sınıflandırma ile B.YOLU uçtan uca demo.

#### H2 — Hailo backend (opsiyonel hızlandırma)
- [ ] “device present” detection + graceful failure
- [ ] HEF model yükleme + inference çağrısı (HailoRT)
- [ ] CPU backend ile aynı output sözleşmesi

**Kabul kriteri:** aynı UI/FSM ile backend değişimi çalışır; Hailo varsa hızlanır, yoksa CPU’ya düşer.

#### H3 — IMX500 backend (opsiyonel hızlandırma)
- [ ] “sensor model deployed” kontrolü
- [ ] IMX500 inference çıktısını okuma
- [ ] Output sözleşmesi eşleme

**Kabul kriteri:** IMX500 ile inference alınabiliyorsa aynı UI/FSM çalışır; alınamıyorsa düzgün hata + fallback.

> Not: H2/H3’te “tek model dosyası” yok. Backend’e göre ayrı artefact gerekir.

---

### WS-I: Persist / Log / Telemetry
- [ ] Event log (jsonl veya sqlite)
- [ ] Session summary (qr_mask, chosen class, confidence, backend, timestamps)
- [ ] Log rotate / disk kullanım limiti (opsiyonel ama önerilir)

**Kabul kriteri:** debug edilebilirlik: “neden kapak açıldı/açılmadı” logdan anlaşılır.

---

### WS-J: Test & Simülasyon
- [ ] FSM transition testleri (FSM_SPEC_EXTRACT.md’ye göre)
- [ ] A.YOLU QR parse/queue testleri (edge cases: whitespace, 0, 64, non-numeric)
- [ ] Adapter contract testleri (mock ports)
- [ ] “Event replay” simülatörü (headless) → UI olmadan logic test

**Kabul kriteri:** CI’da (yerelde) deterministik test.

---

### WS-K: Deployment (Pi üzerinde)
- [ ] systemd service (auto-start)
- [ ] config dosyasının yeri (örn. `/etc/app/app.toml`) + override mekanizması
- [ ] log path + permission

**Kabul kriteri:** Pi boot sonrası otomatik açılış + recover.

---

## 3) Geliştirme Sırası (Önerilen)

Bu sıra “en az sürpriz” üretir:

1. WS-A (repo/config/log)  
2. WS-B (FSM + event model)  
3. WS-C + WS-D (use-case + ports)  
4. WS-E (QR) + WS-F (GPIO) → A.YOLU demo  
5. WS-G (UI) + H1 (CPU inference) → B.YOLU demo  
6. WS-I + WS-J (persist + test/sim)  
7. WS-K (deployment)  
8. H2/H3 (Hailo/IMX500) → opsiyonel hızlandırma

---

## 4) Açık Kararlar (TBD)

- [ ] Model seçimi: YOLO11n-cls mi YOLOv8n-cls mi?
- [ ] CPU inference runtime: Ultralytics native mi, ONNXRuntime mı?
- [ ] Hailo toolchain/export pipeline hangi versiyon?
- [ ] IMX500 export/deploy akışı: hangi format + otomasyon?
- [ ] Uyku (sleep) davranışının UI/preview warm ile ilişkisi

---

## 5) Riskler ve Mitigasyonlar (Sürpriz Avcılığı)

- **Quantization/compile sürprizleri (Hailo/IMX500):**
  - Mitigasyon: CPU backend’i her zaman çalışır tut; hızlandırmalar opsiyonel.
- **Threading/UI donması:**
  - Mitigasyon: UI thread sadece render + event emit; inference başka thread/process.
- **Hardware race conditions (relay):**
  - Mitigasyon: tek-atımlı pulse + per-channel lock/queue; A/B bağımsız kapaklar. `unlock_mode=SIMULTANEOUS` desteklenir, global kısıt yok.
- **Konfig dağınıklığı:**
  - Mitigasyon: tek config kaynağı + runtime override politikası.
- **Disk dolması (log/image):**
  - Mitigasyon: log rotate + image logging default kapalı.

---

## 6) Codex ile Kullanım Notu

Codex’e büyük işleri verirken önce şunu yaptır:

1) Bu PLANS.md’de ilgili maddeyi netleştir  
2) Kapsam dışını yaz  
3) Adım adım plan + kabul kriteri  
4) Sonra kod yazdır

Her iş bitince:
- [ ] ilgili checkbox’ı işaretle
- [ ] “Açık Kararlar (TBD)” listesini güncelle
- [ ] gerekiyorsa README_single.md’yi güncelle (source of truth)
