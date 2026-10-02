<p align="right">
<a href="deepseek-mvp-design.zh_CN.md">简体中文</a> · <strong>English</strong>
</p>

# AI Usage Dashboard: DeepSeek MVP Design

> **Status:** Design discussion draft; not implementation approval. Discussion recorded 2026-10-02.

## 1. Purpose and scope

This note records a small first-version design for a personal AI Passport balance dashboard. It separates
confirmed requirements from recommendations and open decisions. It is not a visual design, firmware
specification, estimate, or claim that these features exist.

The MVP displays **DeepSeek account balance only**. The owner accepts entering and saving a DeepSeek API key
on the device. The preferred setup experience is to connect a phone to the Passport's temporary Wi-Fi access
point and configure router Wi-Fi using an ordinary phone browser, not a dedicated app. The Passport runs
on battery power and normally keeps its screen off; a function-key press lets the user view it. Brightness
and screen-off timeout should be adjustable without reflashing firmware.

This draft does not authorize firmware development or changes to source, configuration, partitions, or a
device.

### Out of scope for the MVP

- Balance or usage history, charts, and estimated remaining tokens.
- Other AI providers, subscriptions, or Coding Plan support.
- OTA, a required backend, a companion app, and guaranteed refresh rates, API costs, or freshness.

## 2. Confirmed requirements and recommendations

### Confirmed in discussion

- Personal AI Passport balance dashboard; first provider DeepSeek, balance only.
- The owner accepts storing an API key on the Passport.
- Ordinary phone-browser Wi-Fi setup is preferred if feasible; no dedicated app.
- Battery-powered use, normally-off screen, function-key view, and user-adjustable brightness and screen-off
  timeout without reflashing.
- The user reports an author-added eighth return-to-app entry in the currently installed third-party
  firmware. This is installation-specific behavior, not a universal OS setting.

### Recommended, not yet confirmed as product decisions

- **Direct device-to-DeepSeek HTTPS** is the recommended architecture. A required backend is unnecessary for
  the documented balance response, and the owner accepts on-device key storage. A backend may be considered
  later, but does not make undocumented account-history or usage APIs available. Separately collected
  observations or user-supplied official exports, if available, would be distinct inputs outside this MVP.
- Use a short-lived, password-protected SoftAP configuration session and stop its web services after success
  or timeout.
- Keep network service independent of dashboard screen lifecycle.

## 3. API and data meaning

DeepSeek documents this request:

```text
GET https://api.deepseek.com/user/balance
Authorization: Bearer <API_KEY>
```

The response documents `is_available` and a `balance_infos` list with `currency` (`CNY` or `USD`) and string
values `total_balance`, `granted_balance`, and `topped_up_balance`.

- `total_balance` is the current total for that currency.
- `granted_balance` is the remaining granted component.
- `topped_up_balance` is the **currently remaining top-up component**, not lifetime recharge.
- These are monetary decimals. Parse and format without binary floating-point rounding; keep CNY and USD
  separate and never add currencies.
- `is_available` means the balance suffices for API calls; it is not Wi-Fi status or an exact
  remaining-token count.

The scoped documentation review found a supported balance API, but no documented history or usage-statistics
endpoint. Use the public API, not private console pages, browser cookies, or model calls just to estimate
consumption. No official polling limit, endpoint cost, or freshness guarantee is asserted. An ordinary key
has no documented read-only scope: querying balance does not restrict what that key could authorize
elsewhere.

## 4. Proposed flow and runtime

### 4.1 Wi-Fi configuration

1. If no Wi-Fi credentials are saved (first setup), or after deliberate physical user entry, start a
   temporary, uniquely named, WPA2-protected access point.
2. Show its SSID, one-time password, and local page address on the device; never show the full API key on
   screen.
3. The user joins the AP and manually opens its local address in an ordinary browser. A captive-portal
   pop-up is optional convenience, never required; phones do not guarantee it, and HTTPS is not intercepted.
4. Explain that the setup AP has no Internet. The phone may warn or ask the user to stay connected.
5. Collect router credentials and the DeepSeek API key in separate fields. Bound and validate them, mask
   secrets, and never put credentials in URLs, logs, or responses.
6. Try the new Wi-Fi credentials and verify association to the target 2.4 GHz network and IP assignment.
   Persist only the new Wi-Fi credentials after success; preserve the old working credentials on failure.
7. Stop AP, HTTP, and DNS services after success or timeout, and clean up their tasks and handlers.

This AP provides local configuration, not Internet access; NAT/router behavior is not needed. Do not assume
one phone can provide a hotspot while its Wi-Fi client joins the device's setup AP. The user may need to
preconfigure a network, use a second client, or switch the phone hotspot after setup; decide and test the
supported path. A temporary outage or unreachable saved access point must not automatically start the
configuration AP; preserve the saved Wi-Fi settings. The retry strategy remains to be decided.

### 4.2 Balance and display

After joining the saved router, call the documented endpoint over authenticated HTTPS. Keep network service
alive independently of screen creation/deletion. Run HTTP, storage, and retries in worker context; button
callbacks stay non-blocking. Non-LVGL tasks hold `bsp_lvgl_lock()` when accessing LVGL. Stop workers,
timers, and handlers before deleting a screen they may access.

On a function-key press, wake the display and show the latest known state; a background refresh may follow.
Mark offline cached values stale. Whether the first press only wakes the screen or also performs its
ordinary action remains to be decided.

Show separate total, granted, and remaining top-up amounts per currency. Exact compact layout is undecided.
Show last-success time only when the clock is trusted; otherwise report time as unknown/relative.
Distinguish connection, refresh, stale-data, and error states. Show battery only for a valid gauge reading;
otherwise mark unavailable or omit it. Offer manual refresh and consider conservative low-frequency polling.
Interval and retry policy remain open; no official API rate is implied.

## 5. Settings and persistence

Brightness and idle screen-off timeout should adjust at runtime and persist across normal restart, without
reflashing. A small store may also hold Wi-Fi credentials, the API key, and explicitly selected preferences.
Settings could be on-device, in the browser page, or both. Their placement, page, and exact key mappings are
undecided; do not treat this note as a finished UI design.

Ranges and defaults are open. 50% brightness and 30 seconds are examples only, not selected defaults.
Validate values against agreed bounds. Restoring defaults should, with explicit confirmation, clear only
this app's settings, not unrelated device data. Firmware updates or erases may affect saved data; no
migration from the current app is guaranteed. Decide whether the last balance survives reboot or stays in
RAM; do not assume an NVS write on every refresh.

The owner reports using a long OK press to enter settings in their current app. This is a candidate to
confirm, not established baseline settings functionality or a required dashboard gesture.

## 6. Hardware and reuse boundaries

Product specifications list ESP32-C3, 8 MB Flash, no PSRAM, 240 x 320 display, 520 mAh battery with CW2017
gauge, 2.4 GHz Wi-Fi, and UP/DOWN/OK keys on a GPIO0 ADC ladder. These constrain memory, radio activity, and
display-on time; they do not establish heap budgets or battery runtime.

Baseline `main` opens the `FoloToy` heading, mascot, and seven hardware-test entries: Display, Button, Audio,
Battery, Wi-Fi, BLE, and Low Power. The user confirms the currently installed third-party firmware has an
author-added eighth return-to-app entry. This is installation-specific; do not assume a universal settings
system or infer the exact image provenance or base commit. The
baseline Wi-Fi demo scans networks in RAM but does not connect or provision. The display demo offers 100%,
50%, and 10% and restores 100% on exit; it does not save preferences or implement idle timeout.

Derivative apps must implement their own UI; do not reuse the baseline test menu, demo pages, or visual
shell. BSP APIs and non-visual logic remain reusable. `bsp_display_backlight(0)` turns off the backlight,
not necessarily the whole display. The low-power demo tests timer-based light/deep sleep, not key wake from
deep sleep. That feature requires BSP work and board validation.

For an initial version, evaluate screen-off by turning off the backlight while the application continues
scanning buttons. Wi-Fi power save is also only a candidate. Automatic light sleep is an optional
experiment, not a guarantee. Postpone deep sleep and exact runtime goals until wake circuitry, reconnect,
energy use, and experience are tested.

## 7. Security and failures

The API key is a bearer credential that could authorize paid requests. Explain how to revoke it if the
device is lost; a dedicated key is a sensible precaution. Never put real keys, Wi-Fi passwords, or personal
data in docs, examples, URLs, or logs. NVS storage is not automatically encrypted or a secure vault.
Flash/NVS encryption and Secure Boot need separate planning; this design does not enable them. The default
partition layout remains unchanged.

Use HTTPS with certificate-chain and hostname validation and a trustworthy current date; never skip
certificate checks for a bad clock. ESP-IDF 5.5.3 has `MBEDTLS_HAVE_TIME_DATE` off by default, so
certificate date checks and trusted time (for example, SNTP before requests) need explicit future
configuration and testing. Bound response bytes and JSON, set finite timeouts, back off retries, throttle
refreshes, and allow redirects only under an explicit safe policy.

Distinguish AP/browser, Wi-Fi authentication/IP, DNS/time/TLS, HTTP authorization, malformed-response, and
temporary provider/network failures. DeepSeek authentication errors or outages must not invalidate a
successful Wi-Fi connection or roll back valid Wi-Fi credentials. Mark cached
offline balances stale and keep their actual last-success time when known. The local page also needs bounded
requests/fields, no credential echo, malformed-input rejection, and complete service cleanup. WPA2 does not
encrypt NVS, and local HTTP is not end-to-end encrypted.

## 8. Decisions before implementation

1. Confirm direct device access (recommended) or define a concrete optional backend and its
   responsibilities.
2. Choose UI language and fonts. Default Montserrat lacks Chinese glyphs; Chinese UI requires font/glyph
   checks and on-device rendering validation.
3. Agree brightness/timeout ranges and defaults, plus conservative polling and retry policy.
4. Decide device vs browser settings, first-key wake behavior, and whether a long OK press is desired.
5. Decide whether the last balance persists across reboot and how stale data is labeled and replaced.

Long-term runtime and deep-sleep wake are future goals, not decisions justified by current hardware
documentation.

## 9. Future acceptance outline (not performed)

A separately approved implementation should include host tests for parsing, exact decimal formatting,
currency separation, errors, stale-state transitions, input bounds, retries, and button timing; an ESP-IDF
build plus merged-image and partition verification; and device tests for browser setup, wrong-password
preservation, AP cleanup, reconnect, HTTPS time/certificates, offline cache, refresh, key wake, settings
persistence, and battery-read failures. Measure heap, stacks, radio/display power, and practical battery
 life before making resource/runtime claims. Verify credentials never appear in logs or unintended UI
output. The one-time setup-AP password may be deliberately displayed during setup, but never display the
full provider API key or router password. Ensure oversized local requests fail safely.

For this documentation task, Build: NOT RUN; Host tests: NOT RUN; Device tests: NOT RUN. No device, account,
or API requests or credential checks were made.

## 10. References

Repository sources:

- [Product specifications](../../../docs/hardware-design/specifications.md)
- [Button, display, and I2C hardware definitions](../../../components/bsp/include/bsp_pins.h)
- [Baseline application and menu](../../../main/main.c)
- [Display/backlight demo](../../../main/demo_display.c)
- [Timer-only low-power demo](../../../main/demo_low_power.c)
- [Derivative UI and runtime rules](../../../docs/development/ai-guide.md)
- [Wi-Fi provisioning reference and scan-only status](../../../docs/development/engineering/wifi-provisioning.md)
- [SoftAP provisioning/resource-budget lessons](../../../docs/reference/phoenixzhc/softap-provisioning-and-resource-budget.md)
- [Documentation conventions](../../../docs/contribution/doc-conventions.md)
- [Fork documentation placement](../../../docs/fork-guide.md)

Public references:

- [DeepSeek: Get User Balance](https://api-docs.deepseek.com/api/get-user-balance)
- [DeepSeek API overview](https://api-docs.deepseek.com/api/deepseek-api)
- [ESP-IDF 5.5.3 ESP32-C3 sleep modes](https://docs.espressif.com/projects/esp-idf/en/v5.5.3/esp32c3/api-reference/system/sleep_modes.html)
- [ESP-IDF 5.5.3 HTTP client](https://docs.espressif.com/projects/esp-idf/en/v5.5.3/esp32c3/api-reference/protocols/esp_http_client.html)
- [Upstream BLUFI Wi-Fi provisioning reference](https://github.com/FoloToy/ai-passport/tree/demo/blufi-provisioning) — Wi-Fi logic reference, not a ready-made browser setup page.
