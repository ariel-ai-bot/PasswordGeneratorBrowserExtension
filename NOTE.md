# PasswordGeneratorBrowserExtension - Analysis Notes

> Analyzed by 愛莉兒 (Ariel) on 2026-05-06

## Architecture Overview

```
PasswordGeneratorBrowserExtension/
├── config/                    # Browser-specific manifests
│   ├── chrome/manifest.json   # MV3, Service Worker
│   └── firefox/manifest.json  # MV3, legacy scripts + gecko settings
├── src/
│   ├── assets/               # Icons, images, loading spinner
│   ├── background/
│   │   └── background.js     # Service Worker: StateManager, message routing, password gen
│   ├── modules/
│   │   ├── bitpacker.js      # Binary packing/unpacking (bit/bool/byte/int/int64/string/timestamp)
│   │   ├── constants.js      # Default values, autofill option maps
│   │   ├── heartbeat.js      # Periodic heartbeat to keep service worker alive
│   │   ├── history.js        # HistoryItem: packs history entries using BitPacker + SchemaCompressor
│   │   ├── password.js       # PWG: HMAC-SHA512 based slave password generation
│   │   ├── schema.js         # SchemaCompressor: compresses type schema to bits
│   │   ├── startup.js        # UIStartUp: waits for background SW to initialize
│   │   ├── state.js          # StateManagerClient (SMW): state/settings/history API via chrome.runtime
│   │   └── storage.js        # StorageManager: local/cloud sync, settings, history persistence
│   ├── options/
│   │   ├── options.html      # Settings tabs: Settings / History / Export-Import / About
│   │   └── options.js        # OptionsManager: tab switching, CRUD for settings
│   ├── popup/
│   │   ├── popup.html        # Main popup UI
│   │   └── popup.js          # PopupManager: event handling, UI state, password generation trigger
│   └── _locales/
│       ├── en/messages.json
│       └── zh_TW/messages.json
├── tests/                    # Jest tests (history, password, schema, bitpacker)
├── webpack.config.js         # Main webpack config (entry: background, options, popup)
├── webpack.chrome.js         # Chrome-specific overrides (currently empty)
├── package.json
└── .gitignore
```

## Data Flow

```
User clicks popup
  → popup.js (PopupManager)
    → chrome.runtime.sendMessage({type: 'GENERATE_PASSWORD', mode})
  → background.js (StateManager)
    → if (master_password === ''): generatePassword()  [random]
    → else: generateSlavePassword() → PWG.generateSlavePassword() [HMAC-SHA512]
    → sends back {type: 'PASSWORD_GENERATED', data: {password, checksum, need_copy}}
  → popup.js onMessage listener
    → fills password field, shows checksum, auto-copies if mode===2
```

## Storage Model

```json
{
  "s": "<base64 packed settings>",    // settings (10 fields, BitPacker)
  "u": "<schema|base64 packed UI state>", // UI state (10 fields + schema)
  "h": "<item1,item2,...>",           // history (comma-separated packed items)
  "t": "<base64 packed timestamp>"    // last write timestamp
}
```

- Stored in `chrome.storage.local` and optionally synced to `chrome.storage.sync`
- Sync decision based on timestamp comparison (local vs cloud)

## Password Generation Algorithms

### Random Mode (no master password)
- Uses `crypto.getRandomValues()` for each character
- **Issue**: Modulo bias — `byte[0] % charset.length` is not uniform

### Slave Mode (with master password)
- HMAC-SHA512 based deterministic generation
- Process:
  1. HMAC-SHA512(master_password) → master_hash_hex (first 8 hex chars as checksum)
  2. For each char position: HMAC-SHA512(key, salt + accumulated_binary) → binary string
  3. Extract `charset_need_bit` bits, mod charset_length to pick character
  4. Slide window approach to avoid regenerating from scratch
- Salt format: `{version}:{salt_string}`

---

## Identified Issues

### 🔴 Critical

1. **Modulo Bias in Random Password Generation** (`background.js:145`)
   ```javascript
   var randomIndex = byte[0] % charset.length;
   ```
   `byte[0]` ranges 0-255. If charset.length is not a divisor of 256, some characters are more likely. E.g., charset_length=10 → digits 0-4 get probability 26/256, digits 5-9 get 25/256.

2. **Missing `clipboardRead` Permission** — The extension uses `navigator.clipboard.writeText()` which should work, but the `clipboardWrite` permission is declared. However, `chrome.tabs.query` for activeTab URL may need `tabs` permission in MV3.

3. **Service Worker Lifecycle** — `chrome.runtime.sendMessage` from popup may fail if SW has been killed. The heartbeat (every 10s) helps, but there's a race condition window.

### 🟡 Important

4. **Typo: `storge_cloud_sync`** — In `storage.js:33` and everywhere it's referenced (should be `storage_cloud_sync`)

5. **Typo: `is_initial_valuies`** — In `history.js:28` (should be `initial_values`)

6. **Inefficient Binary String Concatenation** in `password.js:38`
   ```javascript
   combine_binary_password = combine_binary_password.substring(diff_offset);
   ```
   Creates a new string each iteration. For 40-char passwords with 7-bit charset, this could do 5+ iterations with strings up to 3500+ bits.

7. **`toastr` Dependency Unused** — Imported in `background.js` and `options.js` but only used in `options.js`. Background imports it but never calls `toastr.success()` etc.

8. **Fire-and-forget `chrome.runtime.sendMessage`** in `popup.js:17`
   ```javascript
   let response = chrome.runtime.sendMessage({ type: 'GENERATE_PASSWORD', mode: mode });
   return response;  // response is undefined! Not a Promise
   ```
   The function is `async` but returns `undefined` because `sendMessage` is not awaited.

9. **`getKeywordFromUrl` strips `www.` then replaces all dots** — `facebook.com` → `facebook-com`, but `api.google.co.uk` → `api-google-co-uk` (loses domain structure)

10. **`getPureURL` strips port and query** — `https://example.com:8080/path?foo=bar` → `https://example.com/path` (may cause salt mismatch)

### 🟢 Minor

11. **Inconsistent `var`/`let`/`const`** — `popup.js` uses `var` on line 13, `background.js` uses `var` on line 11.

12. **`localStorage` for theme not persisted in settings** — Theme preference lives in `localStorage` but isn't part of the settings schema, so cloud sync doesn't carry it.

13. **No password strength indicator** — Users can't see if their generated password is strong enough.

14. **`autocomplete="off"` not enough** — Modern browsers ignore `autocomplete="off"` on password fields. Should use `autocomplete="new-password"`.

15. **`webpack.chrome.js` is empty** — Chrome and Firefox share the same output directory structure; no Chrome-specific optimizations.

16. **No CI/CD** — No GitHub Actions, no automated testing on PR.

17. **No LICENSE file** — Package.json says `"license": "ISC"` but the repo has GPL-3.0 from the parent.

18. **`chrome.runtime.onSuspend` in options.html theme.js** — Theme toggle uses IIFE but isn't a module, loads via `<script>` tag.

---

## Planned Improvements (to be committed)

1. ~~Fix modulo bias in `generatePassword()` — use rejection sampling~~ ✅
2. ~~Fix fire-and-forget `send_generate_password()` return value~~ ✅
3. ~~Fix typos (`storge_cloud_sync`, `is_initial_valuies`)~~ ✅
4. ~~Change `autocomplete` to `new-password`~~ ✅
5. ~~Add password strength indicator~~ ✅
6. ~~Remove unused `toastr` import from background.js~~ ✅
7. ~~Add `.editorconfig`~~ ✅
8. Add Chrome-specific webpack config ⬜
9. Add basic GitHub Actions workflow ⬜
10. Improve binary string efficiency in password.js ⬜

---

## Build & Test Verification (2026-05-06)

```
npm run build:chrome    → ✅ webpack 5.98.0 compiled successfully in 4002 ms
npm run build:firefox   → ✅ webpack 5.98.0 compiled successfully in 4022 ms
npm test                → ✅ 4 suites, 10 tests passed
dist/chrome/            → ✅ All expected files present
dist/firefox/           → ✅ All expected files present
```
