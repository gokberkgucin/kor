# FSM Metin Spesifikasyonu (Draw.io’dan çıkarım)

> Kaynak: `corum\_state\_machine\_v0\_2\_5\_timerfix\_ltencoded.drawio.xml` (A.YOLU ve B.YOLU sayfaları)
> Amaç: Codex ve insan okuyucu için state/event/transition’ları \*\*metne dökmek\*\*. Diyagram güncellenirse bu dosya da güncellenmeli.

## A.YOLU (A-path) State Machine

### Durumlar (states)

* **A\_BUILD\_LID\_QUEUE** — Kapak kuyruğu (build lid list/queue)
  sıra (order): 25→24→23→22→21→20 (fixed)
  seçim (selection): qr\_mask bits
* **A\_DECODE\_VALIDATE** — Decode + doğrula (decode \& validate)
  valid: 1–63
* **A\_ERROR** — Hata (error state)
  safe stop + log
* **A\_IDLE** — QR bekle (wait for QR scan)
* **A\_INTER\_DOOR\_DELAY** — Kapak arası bekle (inter-door delay)
  inter\_door\_delay\_sec=1s default, config (≥0)
* **A\_INTER\_SESSION\_DELAY** — Oturum arası (inter-session delay)
  inter\_session\_delay\_sec=0.5s default (config)
* **A\_INVALID\_QR** — Geçersiz QR (invalid QR)
  unlock yok (no unlock)
* **A\_SESSION\_DONE** — Oturum bitti (session done)
* **A\_SLEEP** — A.YOLU uyku (A-path sleep)
  15 dk (15 min) inaktif → sleep
  QR uyandırmaz (QR does not wake)
* **A\_UNLOCK\_LID** — Tek kapak aç (sequential unlock)
  unlock\_pulse\_duration\_sec=3s (default, config)
* **A\_UNLOCK\_MULTI** — Çoklu kapak aç (multi-lid unlock)
  unlock\_mode=SIMULTANEOUS (config)
  unlock\_stagger\_sec=0.3s (default)
  0=fully simultaneous

### Geçişler (transitions)

|From|Trigger / Guard|Actions|To|
|-|-|-|-|
|START|init||A\_IDLE|
|A\_IDLE|E\_A\_IDLE\_TIMEOUT\_15M (15 min idle)||A\_SLEEP|
|A\_SLEEP|E\_BUTTON32\_PRESS (A\_YOLU\_UYAN\_BUTTON)||A\_IDLE|
|A\_SLEEP|E\_QR\_SCANNED|ignore + buffer flush|A\_SLEEP|
|A\_IDLE|E\_QR\_SCANNED (scan)||A\_DECODE\_VALIDATE|
|A\_DECODE\_VALIDATE|\[invalid]||A\_INVALID\_QR|
|A\_INVALID\_QR|log \& return||A\_IDLE|
|A\_DECODE\_VALIDATE|\[valid]||A\_BUILD\_LID\_QUEUE|
|A\_BUILD\_LID\_QUEUE|\[unlock\_mode==SEQUENTIAL]||A\_UNLOCK\_LID|
|A\_BUILD\_LID\_QUEUE|\[unlock\_mode==SIMULTANEOUS]||A\_UNLOCK\_MULTI|
|A\_UNLOCK\_LID|after unlock\_pulse\_duration\_sec||A\_INTER\_DOOR\_DELAY|
|A\_INTER\_DOOR\_DELAY|\[more lids] after inter\_door\_delay\_sec||A\_UNLOCK\_LID|
|A\_INTER\_DOOR\_DELAY|\[queue empty]||A\_SESSION\_DONE|
|A\_UNLOCK\_MULTI|after all pulses complete (unlock\_pulse\_duration\_sec + (N-1)\*unlock\_stagger\_sec)||A\_SESSION\_DONE|
|A\_SESSION\_DONE|\[next session queued]||A\_INTER\_SESSION\_DELAY|
|A\_INTER\_SESSION\_DELAY|after inter\_session\_delay\_sec start next session||A\_DECODE\_VALIDATE|
|A\_SESSION\_DONE|\[no queued sessions]||A\_IDLE|
|A\_DECODE\_VALIDATE|E\_QR\_SCANNED|enqueue|A\_DECODE\_VALIDATE|
|A\_BUILD\_LID\_QUEUE|E\_QR\_SCANNED|enqueue|A\_BUILD\_LID\_QUEUE|
|A\_UNLOCK\_LID|E\_QR\_SCANNED|enqueue|A\_UNLOCK\_LID|
|A\_INTER\_DOOR\_DELAY|E\_QR\_SCANNED|enqueue|A\_INTER\_DOOR\_DELAY|
|A\_UNLOCK\_MULTI|E\_QR\_SCANNED|enqueue|A\_UNLOCK\_MULTI|
|A\_INTER\_SESSION\_DELAY|E\_QR\_SCANNED|enqueue|A\_INTER\_SESSION\_DELAY|
|A\_IDLE|E\_FATAL\_ERROR|safe stop + log|A\_ERROR|
|A\_DECODE\_VALIDATE|E\_FATAL\_ERROR|safe stop + log|A\_ERROR|
|A\_BUILD\_LID\_QUEUE|E\_FATAL\_ERROR|safe stop + log|A\_ERROR|
|A\_UNLOCK\_LID|E\_FATAL\_ERROR|safe stop + log|A\_ERROR|
|A\_UNLOCK\_MULTI|E\_FATAL\_ERROR|safe stop + log|A\_ERROR|

### Notlar / Konfig Parametreleri (notes)

* A.YOLU (A-path) – QR/Scanner Side State Machine (v0.2.4)
* Not (note): QR tekrar okutulabilir (QR replay). Aktif oturumda yeni QR okunursa session queue (FIFO) içine ekle (enqueue).
* A.YOLU uyandırma (wake): • (32) A\_YOLU\_UYAN\_BUTTON (wake button) • A ve B unlock aynı anda olabilir (concurrent unlock allowed) • Kapak kapatma komutu yok (no close command): kapaklar mekanik kapanır (self-close)
* Config (configuration) – A.YOLU: • a\_sleep\_timeout\_min = 15 • unlock\_mode = SEQUENTIAL (default) | SIMULTANEOUS • unlock\_pulse\_duration\_sec = 3 • inter\_door\_delay\_sec = 1 (≥0) • inter\_session\_delay\_sec = 0.5 • unlock\_stagger\_sec = 0.3 (0=fully simultaneous)
* Global QR handler (global event handler): • A\_SLEEP hariç (except A\_SLEEP): E\_QR\_SCANNED → enqueue • A\_SLEEP: E\_QR\_SCANNED → ignore + buffer flush

## B.YOLU (B-path) State Machine

### Durumlar (states)

* **B\_CANCELLED** — İptal (cancel)
  timeout veya kullanıcı iptali (timeout or user cancel)
* **B\_ERROR** — Hata (error state)
  safe stop + log + ekran uyarısı
* **B\_SCREEN1** — Idle / Bekleme (idle)
  Sekil\_7\_Dokunmatik\_Ekran\_1
* **B\_SCREEN2** — Beyan + sonuç birleştirme (merge declaration + result)
  Sekil\_8\_Dokunmatik\_Ekran\_2
* **B\_SCREEN3** — Retry (retry classification)
  Sekil\_9\_Dokunmatik\_Ekran\_3
  retry\_inactivity\_timeout\_sec=15 default (config)
  max\_retry\_time\_sec=13 default (config)
* **B\_SLEEP** — Uyku modu (sleep mode)
  screen off, preview+inference stop
  QR uyandırmaz (QR does not wake)
* **B\_UNLOCKED\_WAIT\_REMOVE** — Kilit açıldı (unlocked)
  lid opened → wait E\_WEIGHT\_REMOVED
  unlocked\_remove\_timeout\_sec=5 default (config)

### Geçişler (transitions)

|From|Trigger / Guard|Actions|To|
|-|-|-|-|
|START|init||B\_SCREEN1|
|B\_SCREEN1|E\_IDLE\_TIMEOUT\_1H (no touch|weight)|B\_SLEEP|
|B\_SLEEP|E\_TOUCH (wake)||B\_SCREEN1|
|B\_SLEEP|E\_WEIGHT\_PRESENT (wake + start)|new session\_id; reset: decl\_done=false; user\_type=unset; prediction\_valid=false; clear class/conf; stable\_count=0; require\_improvement=false; conf\_old=unset; placement\_done=true; start camera+preview+inference|B\_SCREEN2|
|B\_SCREEN1|E\_WEIGHT\_PRESENT (place-first)|new session\_id; reset: decl\_done=false; user\_type=unset; prediction\_valid=false; clear class/conf; stable\_count=0; require\_improvement=false; conf\_old=unset; placement\_done=true; start camera+preview+inference|B\_SCREEN2|
|B\_SCREEN1|E\_TOUCH (touch-first)|new session\_id; reset: decl\_done=false; user\_type=unset; placement\_done=false; prediction\_valid=false; clear class/conf; stable\_count=0; require\_improvement=false; conf\_old=unset; start preview (inference waits)|B\_SCREEN2|
|B\_SCREEN2|E\_DECLARATION\_DONE|decl\_done=true; store user\_type|B\_SCREEN2|
|B\_SCREEN2|E\_WEIGHT\_PRESENT|placement\_done=true; start inference (if needed)|B\_SCREEN2|
|B\_SCREEN2|E\_WEIGHT\_REMOVED (before unlock)|stop inference; placement\_done=false; prediction\_valid=false; clear class/conf; stable\_count=0; prompt 'atık yok'|B\_SCREEN2|
|B\_SCREEN2|E\_PREDICTION\_UPDATE \[decl\_done \&\& placement\_done \&\& class\_new==user\_type \&\& conf\_new>=CONF\_ACCEPT\_EKRAN2]|unlock (type lid)|B\_UNLOCKED\_WAIT\_REMOVE|
|B\_SCREEN2|E\_PREDICTION\_UPDATE \[decl\_done \&\& placement\_done \&\& class\_new==user\_type \&\& conf\_new<CONF\_ACCEPT\_EKRAN2]|enter retry; require\_improvement=true; conf\_old=conf\_new; stable\_count=0; retry\_start\_ts=now; reset retry\_inactivity\_timer|B\_SCREEN3|
|B\_SCREEN2|E\_PREDICTION\_UPDATE \[placement\_done \&\& !decl\_done]|update class\_new, conf\_new, stable\_count; prediction\_valid=true|B\_SCREEN2|
|B\_SCREEN3|E\_DECLARATION\_DONE|decl\_done=true; update user\_type; reset retry\_inactivity\_timer|B\_SCREEN3|
|B\_SCREEN3|E\_WEIGHT\_PRESENT|placement\_done=true; continue inference; retry\_start\_ts=now (restart max\_retry\_time); reset retry\_inactivity\_timer|B\_SCREEN3|
|B\_SCREEN3|E\_WEIGHT\_REMOVED (before unlock)|stop inference; placement\_done=false; prediction\_valid=false; clear class/conf; stable\_count=0; prompt 'atık yok'; retry\_start\_ts=unset (pause max\_retry\_time); reset retry\_inactivity\_timer|B\_SCREEN3|
|B\_SCREEN3|E\_PREDICTION\_UPDATE \[G\_EKRAN3\_SUCCESS(require\_improvement)]|unlock (type lid); stable\_count=0|B\_UNLOCKED\_WAIT\_REMOVE|
|B\_SCREEN3|E\_RETRY\_MAXTIME\_EXPIRED (max\_retry\_time\_sec=13 default, config) \[placement\_done==true]|unlock non-recyclable (6); stable\_count=0|B\_UNLOCKED\_WAIT\_REMOVE|
|B\_SCREEN3|E\_PREDICTION\_UPDATE \[placement\_done \&\& NOT(G\_EKRAN3\_SUCCESS(require\_improvement))]|update class\_new, conf\_new, stable\_count; prediction\_valid=true; reset retry\_inactivity\_timer|B\_SCREEN3|
|B\_SCREEN3|E\_RETRY\_INACTIVITY\_TIMEOUT (retry\_inactivity\_timeout\_sec=15 default, config)|stop inference; reset flags; clear prediction; return Screen1|B\_SCREEN1|
|B\_UNLOCKED\_WAIT\_REMOVE|E\_WEIGHT\_REMOVED (after unlock)|end session; reset flags; clear prediction|B\_SCREEN1|
|B\_SCREEN2|T=T1\_FIRST\_WINDOW\_SEC|prompt missing step(s)|B\_SCREEN2|
|B\_SCREEN2|T=T1+T2 \& (missing\_decl OR missing\_place)|cancel; stop inference+preview; reset flags; clear prediction|B\_CANCELLED|
|B\_CANCELLED|return to idle||B\_SCREEN1|
|B\_SCREEN2|E\_PREDICTION\_UPDATE \[placement\_done==false]|ignore (late result)|B\_SCREEN2|
|B\_SCREEN2|E\_PREDICTION\_UPDATE \[decl\_done \&\& placement\_done \&\& class\_new!=user\_type]|enter retry; require\_improvement=false; stable\_count=0; retry\_start\_ts=now; reset retry\_inactivity\_timer|B\_SCREEN3|
|B\_SCREEN3|E\_PREDICTION\_UPDATE \[placement\_done==false]|ignore (late result)|B\_SCREEN3|
|B\_UNLOCKED\_WAIT\_REMOVE|E\_UNLOCKED\_REMOVE\_TIMEOUT (unlocked\_remove\_timeout\_sec=5 default, config)|end session (forced); reset flags; clear prediction|B\_SCREEN1|
|B\_SCREEN2|E\_FATAL\_ERROR|safe stop + log + show error|B\_ERROR|
|B\_SLEEP|E\_FATAL\_ERROR|safe stop + log + show error|B\_ERROR|
|B\_SCREEN2|E\_FATAL\_ERROR|safe stop + log + show error|B\_ERROR|
|B\_SCREEN3|E\_FATAL\_ERROR|safe stop + log + show error|B\_ERROR|
|B\_UNLOCKED\_WAIT\_REMOVE|E\_FATAL\_ERROR|safe stop + log + show error|B\_ERROR|
|B\_CANCELLED|E\_FATAL\_ERROR|safe stop + log + show error|B\_ERROR|
|B\_SCREEN2|E\_DECLARATION\_CLEARED (beyan sil/clear)|decl\_done=false; user\_type=unset; prediction\_valid=false; clear class/conf; stable\_count=0|B\_SCREEN2|
|B\_SCREEN3|E\_CANCEL\_SESSION\_BUTTON (oturumu iptal/cancel)|cancel; stop inference+preview; reset flags; clear prediction|B\_CANCELLED|

### Notlar / Konfig Parametreleri (notes)

* B.YOLU (B-path) – Touchscreen + Camera State Machine (v0.2.4)
* 10s+10s kuralı (two-stage timeout): • Screen2’ye girişte (on entry to Screen2) timer başlar (touch-first + place-first) • T1 = T1\_FIRST\_WINDOW\_SEC (default 10s) → prompt missing step(s) • T2 = T2\_SECOND\_WINDOW\_SEC (default 10s) • T1+T2 sonunda (end): - missing\_decl OR missing\_place → cancel + Screen1 - decl\_done AND placement\_done → karar (decision) akışı devam
* Event (event) – AI sonucu (AI result): • E\_PREDICTION\_UPDATE: class\_new, conf\_new güncellendi (updated) • Screen2/3 kararları bu event ile tetiklenir (triggered)  Screen3 başarı koşulu (success guard): • G\_EKRAN3\_SUCCESS(require\_improvement): - require\_improvement=false → (class\_new==user\_type) AND (conf\_new>=CONF\_ACCEPT\_EKRAN3) AND (stable\_count>=N\_STABLE\_FRAMES) - require\_improvement=true → yukarıdakiler + (conf\_new>=conf\_old+CONF\_MIN\_IMPROVEMENT)  Güvenlik (safety): • E\_WEIGHT\_REMOVED (before unlock) → placement\_done=false, prediction\_valid=false, stable\_count=0 • placement\_done=false iken gelen E\_PREDICTION\_UPDATE → ignore (late result) • Retry max süre (max): E\_RETRY\_MAXTIME\_EXPIRED (max\_retry\_time\_sec=13 default, config) → sadece placement\_done=true iken fallback unlock (guard); B\_SCREEN3'te E\_WEIGHT\_PRESENT ile retry\_start\_ts=now (restart) • Weight eventleri (weight events) E\_WEIGHT\_PRESENT/E\_WEIGHT\_REMOVED: polling (polling) ile türetilir → state girişinde (on state entry) mevcut ağırlık (current weight) kontrol edilerek event kaçırma (missed event) riski azaltılır. • Screen2: E\_DECLARATION\_CLEARED (beyan sil/clear) → decl\_done=false; user\_type=unset; prediction\_valid=false
* Log (logging) – saha denemesi (field trial) için öneri (recommended): Her event + her transition için kayıt (log record). Format: JSONL (json lines). Ortak alanlar (common fields): timestamp\_utc, device\_id, path(A/B), session\_id, state\_from, state\_to, event, guard\_result, actions B alanları: decl\_done, placement\_done, user\_type, weight\_raw, class\_new, conf\_new, stable\_count, require\_improvement A alanları: qr\_raw, qr\_mask, selected\_lids, unlock\_mode, unlock\_cmds Versiyon (version): app\_version, config\_hash, model\_hash
* Not (note): A ve B kullanıcıları birbirinin kapaklarını görmez (users cannot see each other’s lids) v2.5 (no weight sensor): E\_WEIGHT\_PRESENT = Button\_koy ile simüle (simulated), E\_WEIGHT\_REMOVED = Button\_cek ile simüle (simulated).İsimler aynı kalır (names unchanged); sadece event kaynağı (event source) sensör (sensor) yerine buton (button). A\_SLEEP ve B\_SLEEP bağımsız olabilir (independent sleeps).
