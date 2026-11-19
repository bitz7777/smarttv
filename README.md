# Lumen TV — Modulul 1 (nucleul TV)

Backend (FastAPI) + interfață Smart TV (HTML/CSS/JS) + servicii systemd pentru Debian 12.

## 1. Testare rapidă pe calculatorul tău (fără instalare de OS)

```bash
./tools/dev-run.sh          # creează venv, instalează FastAPI, pornește pe :8080
```
Deschide http://localhost:8080 într-un browser, pe tot ecranul (F11).
Taste: săgeți, Enter = OK, Esc/Backspace = înapoi, Home = acasă, Space = play/pauză, `,` `.` = -10s/+10s.

Teste backend: `cd backend && pip install pytest httpx && pytest -q`

Testare pairing de pe alt dispozitiv din rețea (înlocuiește IP-ul):
```bash
curl -s -X POST http://IP:8080/api/pair/start -H 'Content-Type: application/json' -d '{"device_name":"Test"}'
# pe ecran apare PIN-ul; apoi:
curl -s -X POST http://IP:8080/api/pair/confirm -H 'Content-Type: application/json' -d '{"pair_id":"...","pin":"123456"}'
curl -s -X POST http://IP:8080/api/control/key -H 'Authorization: Bearer TOKEN' -H 'Content-Type: application/json' -d '{"key":"right"}'
```
(De pe aceeași mașină, cererile pe localhost sunt tratate ca UI-ul TV și nu cer token. Testează de pe alt dispozitiv.)

## 2. Instalare pe laptop (Debian 12 minimal, proaspăt instalat)

```bash
git clone <repo> && cd smarttv
sudo ./tools/install.sh
sudo reboot
```
Laptopul pornește direct în interfața Lumen (cage + Chromium kiosk).
Loguri: `journalctl -u smarttv-ui -u smarttv-backend -b`

## 3. Ce este real și ce nu (la Modulul 1)

| Componentă | Stare |
|---|---|
| Pairing PIN + token, API de control, WebSocket, volum, power, stare Wi-Fi/BT/baterie | scrise; pairing testat automat, restul netestat pe hardware |
| UI: hero, rânduri, navigare cu săgeți, setări, afișare PIN, HUD volum | scris, **netestat într-un browser real** — așteaptă-te la mici ajustări vizuale |
| Redare video | **simulată** (bară de progres, play/pauză) ca să poți testa Now Playing din iOS. Redarea reală cu mpv vine în Modulul 3 |
| Conținutul din rânduri | titluri fictive din `backend/app/catalog.json`. Biblioteca reală (USB, rețea, metadate) vine în Modulul 3 |
| Pornirea aplicațiilor (Netflix etc.) | pornește Chromium separat în `--app`, cu profil propriu per serviciu. Tasta Home de pe iPhone o închide |
| Boot kiosk (`smarttv-ui.service`) | scris după rețeta standard `cage`, **netestat** — prima pornire poate cere ajustări |

## 4. Limite cunoscute

- **Securitate:** comunicarea e HTTP (fără TLS) în rețeaua locală; token-ul este protejat doar prin faptul că rețeaua e de încredere. TLS cu certificat propriu intră în Modulul 4. Cererile de pe localhost nu cer token (UI-ul TV rulează acolo).
- **DRM:** Chromium din Debian **nu include Widevine**, deci Netflix/Disney+/Prime/Max/SkyShowtime nu vor reda încă. Soluția (Google Chrome stable, care îl include) se rezolvă în Modulul 3, iar calitatea rămâne de obicei la 720p (Widevine L3 pe Linux). YouTube, Spotify web, Kodi, Plex, Jellyfin nu au această problemă.
- **Taste în aplicații externe:** cât rulează un serviciu extern (Netflix etc.), săgețile de pe iPhone nu ajung în aplicație. Vine în Modulul 3 prin dispozitiv virtual `uinput`.
- Tastatura de pe telefon: textul ajunge la UI (apare ca notificare); câmpurile de text în UI vin odată cu căutarea.

## 5. Contract API (pentru aplicația iOS, Modulul 2)

- Descoperire Bonjour: serviciu `_lumentv._tcp`, port 8080
- `GET /api/info` (public) · `POST /api/pair/start {device_name}` → `{pair_id}` · `POST /api/pair/confirm {pair_id, pin}` → `{token, device_id}`
- Cu `Authorization: Bearer <token>`:
  `POST /api/control/key {key}` (up, down, left, right, ok, back, home, playpause, ff, rw) ·
  `POST /api/control/text {text}` · `POST /api/control/volume {action: up|down|mute|set, value}` ·
  `POST /api/control/power {action: sleep|restart|shutdown}` ·
  `GET /api/apps` · `POST /api/apps/launch {id}` · `GET /api/state` · `GET|PATCH|DELETE /api/devices`
- `WS /ws/events` (header `Authorization`) trimite `state`, `now_playing`, `volume`
