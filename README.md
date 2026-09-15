# XHouse IoT Controller for Home Assistant

[![hacs_badge](https://img.shields.io/badge/HACS-Custom-41BDF5.svg)](https://hacs.xyz/)

A Home Assistant custom integration for XHouse / Giigle IoT devices (gate controllers, smart switches, covers, etc.).

## Supported Devices

- **XH-SGC01** — WiFi smart garage controller
- **EGA18, EGA15, EGB18 & EGB1900** — Gate controllers
- Other WiFi switch devices discovered on the account

## Installation

### HACS (Recommended)

   [![Open your Home Assistant instance and open a repository inside the Home Assistant Community Store.](https://my.home-assistant.io/badges/hacs_repository.svg)](https://my.home-assistant.io/redirect/hacs_repository/?owner=BenJamesAndo&repository=XHouse-IoT-Controller&category=integration)

1. Open HACS in Home Assistant
2. Click the three dots in the top right → **Custom repositories**
3. Add this repository URL and select **Integration** as the category
4. Click **Download**
5. Restart Home Assistant

### Manual

1. Copy the `custom_components/xhouse` folder into your Home Assistant `config/custom_components/` directory
2. Restart Home Assistant

## Setup

1. Go to **Settings → Devices & Services → Add Integration**
2. Search for **XHouse IoT Controller**
3. Enter your XHouse account email and password. Use a second account linked to your device to prevent log outs on your app when using Home Assistant.

## Configuration

After setup, click **Configure** on the integration to adjust:

- **Refresh interval** — Polling frequency in seconds (minimum 5s, default 30s)
- **Debug mode** — Enable verbose logging

## Platforms

| Platform | Description |
|----------|-------------|
| `cover` | Gate/barrier controllers (open, close, stop) |
| `switch` | On/off switch devices |
| `button` | Momentary action buttons |

## Links

- [Community thread](https://community.home-assistant.io/t/sgc01-smart-wifi-garage-opener/457208)

## Screenshot
<img width="989" height="271" alt="image" src="https://github.com/user-attachments/assets/6e97540f-8d7e-4aaa-ad4b-ceabdecd9298" />

## Troubleshooting: cloud API quirks (patched in this fork)

### JSON responses with a non-standard `Content-Type`

The XHouse cloud (`iemp.giigleiot.net`) returns valid JSON but with a
non-standard `Content-Type: text/json;charset=utf-8` header. `aiohttp`'s
`resp.json()` only accepts `application/json` by default and raises
`ContentTypeError` (`Attempt to decode JSON with unexpected mimetype`) on
every response. Each failed poll marks the coordinator as failed, which
makes **all entities unavailable and blocks every command until Home
Assistant is restarted**.

**Fix:** `await resp.json(content_type=None)` — decode JSON regardless of
the reported MIME type.

### Dead session with no re-login

The integration only recognized an expired token by the literal message
`"token invalid"`. Any other server wording ("token expired",
"please login", etc.) meant the integration never re-logged in and stayed
broken until restart. On top of that, the command path called
`int(api.user_id)` after the user id had already been cleared, raising an
unhandled `TypeError`.

**Fix:** session errors are now detected via a set of markers; commands go
through a coordinator helper that re-logins first and retries the command
once after re-login, so the integration recovers by itself.

Patch is compatible with integration version 1.2.1.

формулировка сервера = интеграция никогда не перелогинивается. Плюс при нажатии кнопки
код обращался к int(api.user_id) с уже очищенным user_id → необработанный TypeError.
→ Теперь: широкий набор признаков «сессия протухла», автоперелогин перед командой,
повтор команды после перелогина, таймауты и битый JSON оборачиваются в нормальную ошибку.
