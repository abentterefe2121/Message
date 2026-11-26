# Orthodox Hub Encryption & Full-App Pro Blueprint (From Smart Bible AI)

**Target Repo:** `OrthodoxHubNative` (`com.ethiopianorthodox.ethorthodox`, 14 modules)  
**Security Note:** This bridge repo is public. **Never** commit `.db` files, `.dpapi`, `.jks`, `google-services.json`, private keys, raw hex keys, real Worker URLs, or Firebase IDs here.

---

## 1. Strategic Decisions Confirmed by Owner

1. **Distribution:** Google Play Internal/Closed/Open Testing + Direct APK/Telegram/Website (no public Play Store release).
   - **Firebase App Check (Play Integrity):** Keep `PLAY_RECOGNIZED = OFF`, `LICENSED = OFF`, minimum integrity = `Device integrity` so direct APKs work while emulators, tampered/re-signed APKs, debuggers, and Frida/LSPosed hooks are blocked.
   - Register **both** the release upload keystore SHA-256 and Play App Signing SHA-256 in Firebase Console.
2. **APK Size Budget (40-70 MB OK):** Bundle all 14 core DBs as compressed `sbpack-v2` (or SQLCipher `.db` for small assets) so **all 9 drawer apps work 100% offline immediately after first setup**.
3. **Monetization Rule (Full-App Lock After Trial):**
   - Smart Bible stays generous/free; **Orthodox Hub is an all-in-one paid suite**.
   - During the **7- or 14-day Free Trial** (plus 24-hour `OFFLINE_GRACE` on fresh offline installs), the entire app is unlocked.
   - Once trial/grace expires (or clock rollback is detected), **lock the whole app** via `ProAccessGate` while keeping `EditionSetupActivity` and `HubScreenActivity` (Pro activation, Settings, About/Legal) accessible so users can copy `OHRQ-...`, pay via Telebirr/CBE/Telegram, and enter `OHPRO-...`.
4. **Replace Legacy Design:** Do **not** implement `doc/CONTENT-ENCRYPTION.md` (`.dbe` / static `content_test.key`). Use Smart Bible's `sbpack-v2` + SQLCipher 4 + Play Integrity App Check + `ContentGenerationJournal` + `ProStateVault` + `activation-worker`.

---

## 2. Database & Asset Map Across All 14 Modules

### A. 14 Bundled Core Assets (Inside APK, Zero Plaintext in Release)

| `assetId` | Module(s) | Source File | Expected `user_version` | Notes |
|---|---|---|---:|---|
| `bible-am` | `:bible`, `:qvn`, `:bible-study`, `:haymanote-abew` | `eob_bible_am.db` | `2` | `sbpack-v2` (~5.0 MB) |
| `bible-am-2000` | `:bible`, `:qvn`, `:bible-study`, `:haymanote-abew` | `eob_bible_am_2000.db` | `2` | `sbpack-v2` (~3.1 MB) |
| `bible-en-kjva` | `:bible`, `:qvn`, `:bible-study`, `:haymanote-abew` | `eob_bible_en_kjva.db` | `2` | `sbpack-v2` (~2.0 MB) |
| `strongs` | `:bible` | `eob_strongs.db` | `0` | `sbpack-v2` (~2.5 MB) |
| `xrefs` | `:bible` | `eob_xrefs.db` | `0` | `sbpack-v2` (~3.0 MB) |
| `topics` | `:bible` | `eob_topical.db` | `6` | SQLCipher `.db` or `sbpack-v2` (~1.2 MB) |
| `bahire-hasab` | `:bahire-hasab`, **`:tselot` (`TselotSinksarAssetDb`)** | `ethiopian_orthodox_days.db` | `0` | **Single shared store** — stop duplicating into `tselot_sinksar_days.db` |
| `content-json` | `:bible`, `:bible-study` | `daily_verses.json` + 5 `reading_plan_*.json` | `1` | Encrypted SQLCipher `documents (path TEXT PRIMARY KEY, data BLOB)` |
| `haymanote-abew` | `:haymanote-abew`, `:bible` | `haymanote_abew.db` | `0` | Bundle in Core for Orthodox Hub (~1.5 MB `sbpack-v2`) so drawer app works offline immediately |
| `tselot` | `:tselot` | `mezgebe_tselot_v4.db` | `6` | `sbpack-v2` (~11-14 MB, < 25 MiB cap) |
| `sinksar` | `:sinksar` | `sinksar.db` | `1` | `sbpack-v2` (~3.5 MB); apply `idx_sinksar_*` indexes |
| `gibre-himamat` | `:gibre-himamat` | `gibre_himamat.db` | `1` | `sbpack-v2` (~2.5-5 MB) |
| `kidase` | `:kidase` | `Metsihafe_Kidase.db` | `1` | `sbpack-v2` (~0.6 MB); apply `idx_kidase_books_*` indexes |
| `seatat` | `:seatat` | `Metsihafe_Seatat.db` | `2` | `sbpack-v2` (~0.5 MB) |

### B. Cloud Packs (Shared `CloudPackManager` + `ContentGenerationJournal`)
- **Cloud Background (auto-downloads silently after Home opens):** `andimta` (`eob_andimta.db`, `sbpack-v2` ~8.0 MB) — used by `:bible` + `:haymanote-abew`.
- **Cloud On-Demand (downloaded if chosen in Bible selector):** `bible-gz-1980`, `bible-en-webbe`, `bible-or`, `bible-ti` (plus optional Amharic editions if offered).

### C. Device-Local User DBs & Vaults (NEVER use server keys)
- `BibleUserDb`, `TselotUserDb` (keep legacy `filesDir` migration!), `NotesDatabase`, `StudyDb`
- `AI-Hub` (`ChatVault.kt`, `Credentials.kt`): **port Smart Bible's Keystore safety fix** (`if (!createIfMissing || ks.containsAlias(alias) || hasStoredVault)` -> refuse overwrite so transient Keystore errors never wipe user chats/API keys).

---

## 3. Exact Cryptographic & Runtime Contracts

### A. `.sbpack` v2 Binary Spec (`pack-crypto.mjs` <-> `ProtectedAssetStore.java`)
- Header (17B): `"SBPK"` (4B) || `0x02` (1B) || `NONCE_PREFIX` (8B random) || `CHUNK_PLAIN_MAX` (4B uint32 BE, default `65536`).
- Chunks `i = 0, 1, ...`:
  - `META` (4B uint32 BE = `plainLen | (isFinal ? 0x80000000 : 0)`) || `CIPHERTEXT` || `TAG` (16B)
  - `Nonce_i = NONCE_PREFIX(8B) || uint32_be(i)`
  - `AAD_i = UTF-8("sbpk-v2|" + assetId + "|" + keyId + "|" + i + "|" + (isFinal ? 1 : 0))`
- Decrypted chunk stream is `gzip -9` over `SBDB` v1 stream:
  - Header: `"SBDB"` (4B) || `0x01` (1B) || `userVersion` (int32 BE) || `applicationId` (int32 BE)
  - Ops: `0x01` TABLE (`str16 name`, `str32 createSql`, `u16 colCount`, `str16[] cols`), `0x02` ROW (`0`=NULL, `1`=i64, `2`=f64, `3`=u32+utf8, `4`=u32+blob), `0x03` POST_SQL (`str32` for FTS `'rebuild'`, indexes, views, triggers), `0xff` END.
- Validate `pack_sha256` and `stream_sha256`.

### B. SQLCipher 4 Fast Streaming Import + Security Checks
- Runtime safety check before App Check or Keystore unwrap: block `Debug.isDebuggerConnected()`, `/proc/self/status` `TracerPid != 0`, and Frida/LSPosed/Xposed libraries in `/proc/self/maps`.
- Mint per-device 256-bit local key sealed in Android Keystore AES-256-GCM (`noBackupFilesDir/protected-<assetId>-dev/`).
- Stream decrypt -> gunzip -> batched 64-row `SQLiteStatement` inserts directly into SQLCipher 4 with `PRAGMA temp_store=MEMORY`, `PRAGMA journal_mode=MEMORY`, `PRAGMA cache_size=-8192`, `PRAGMA locking_mode=EXCLUSIVE`, `PRAGMA foreign_keys=OFF`, `PRAGMA synchronous=OFF` (intentionally omit `PRAGMA cipher_memory_security=ON`).
- After applying indexes, switch back to `PRAGMA journal_mode=DELETE`, `PRAGMA synchronous=FULL`, `PRAGMA foreign_keys=ON`, run `PRAGMA quick_check(1)`, `fsync`, atomic rename, reopen via `openTrusted`, and publish in `ContentGenerationJournal`. Zero key arrays in RAM.
- Subsequent launches open 100% offline via `openIfAlreadyInstalled()`.

### C. Cross-Module `installProtectedQuery` Seam
- Eliminate all raw `getDatabasePath("eob_bible_*.db")` / `eob_andimta.db` / `assets.open(...)` scanning in `:bible`, `:bahire-hasab`, `:qvn` (`QvnBibleDb`), `:bible-study` (`StudyBibleDb`), `:haymanote-abew` (`HaymanoteAssetDb.SharedReader`), `:tselot` (`TselotAssetDb` & `TselotSinksarAssetDb`), `:sinksar`, `:kidase`, `:seatat`, `:gibre-himamat`.
- Every module must expose an `installProtectedQuery(...)` provider callback backed by `ProtectedAssetStore` before any reader runs, and throw a clean non-UI-thread error if uninitialized in release builds.

### D. Gradle Release Protection Gate
- Replicate `ProtectBibleAssets` + `verifyReleaseProtection` (hooked to `preReleaseBuild`) + post-build APK/AAB Python audit:
  - Refuse release builds unless all protection flags are enabled.
  - Strip all plaintext `.db` files and raw protected `.json` files from merged assets.
  - Assert zero `SQLite format 3\0` headers anywhere inside final release `.apk`/`.aab`.
- Prune ~11+ MB dead bloat: unused `reference/keyboard-sample/`, duplicate 1 MB `mipmap-*/ic_launcher.png`, duplicate `bible/assets/bible/fonts/`, and the 4 unreferenced `tselot/src/main/assets/tselot/json/*.json` files.

---

## 4. Pro Activation & Full-App Lock Contract (`orthodox_hub`)

Our existing `cloud/activation-worker/` **already supports `appId = "orthodox_hub"`**:
- **Request Code (`DeviceRequestCode`):**
  - Format: `OHRQ-DDDD-DDDD-RRRR-CC` (`ALPHABET = "23456789ABCDEFGHJKLMNPQRSTUVWXYZ"`)
  - `DDDD-DDDD` = 8 Base32 chars of `SHA-256("smart-bible-dev-v1|" + androidId + "|" + packageName)`
  - `RRRR` = 4 Base32 chars of `SHA-256("smart-bible-req-v1|" + androidId + "|" + installSeed)` (where `installSeed` is persisted inside Keystore-sealed `ProStateVault`)
  - `CC` = **MUST use exact checksum salt `"sbrq-chk|" + devId + reqTag`** (`encodeBase32(..., 2)`) so the shared `activation-worker` validates `OHRQ-...` identically to `SBRQ-...`!
- **Activation Key Format:** `OHPRO-XXXX-XXXX-XXXX`
- **Worker Endpoints (`<YOUR_ACTIVATION_ORIGIN>`, zero Firebase):**
  - `GET /v1/offer?app=orthodox_hub` -> `{ enabled, priceEtb, telegramHandle, telegramChannel, websiteUrl, playStoreUrl, methods: [...] }`
  - `POST /v1/trial/status` (`{ appId: "orthodox_hub", requestCode }`) -> called **once per install**; returns ECDSA P-256 (IEEE P1363 64B base64url) signed `"trial"` token `{ v:1, type:"trial", appId, deviceId, startedAt, expiresAt, serverNow }`.
  - `POST /v1/activate` (`{ appId: "orthodox_hub", requestCode, activationKey }`) -> returns ECDSA P-256 signed `"pro"` grant `{ v:1, type:"pro", appId, licenseId, deviceId, issuedAt, recheckAfter }`.
- **Android Client (`ProStateVault` + `CloudflareProClient` + `ProAccessGate`):**
  - Embed ONLY the ECDSA P-256 public SPKI key via Gradle/BuildConfig/strings (`<YOUR_ACTIVATION_PUBLIC_KEY_SPKI>`). Verify `"pro"` vs `"trial"` strictly offline (`ProGrantVerifier`), never allowing `"trial"` to impersonate `"pro"`.
  - Seal grants, trial tokens, `install_tag_hex`, `PackageManager.firstInstallTime`-anchored 24h `OFFLINE_GRACE`, server time offset, and monotonic `elapsedRealtime()` anti-rollback state in `noBackupFilesDir` via Android Keystore AES-256-GCM (`AtomicFile`, fail closed on Keystore errors).
  - Non-debuggable builds missing activation config must fail closed (`DisabledPremiumController`).
  - **Whole-App Gate Behavior:**
    - On `MainActivity` (Home) when `PRO_ACTIVE`/`TRIAL_ACTIVE`/`OFFLINE_GRACE` is false: show non-dismissible Pro Lock Card over dimmed Home (`Unlock Pro` -> opens `HubScreenActivity` Pro Activation screen; `Connect & Retry` for `OFFLINE_GRACE_EXPIRED`; Back exits app).
    - Register `ProAccessGate.registerActivityOverlayGuard` on all 15 feature Activities (`BibleActivity`, `TselotActivity`, `KidaseActivity`, `SeatatActivity`, `SinksarActivity`, `GibreActivity`, `HaymanoteActivity`, `BahireHasabActivity`, `BibleStudyActivity`, `AiActivity`, `AiSetupActivity`, `QvnActivity`, `NotesActivity`, `TemplatesActivity`, `StudioActivity`) so nothing can bypass Home.
    - Never lock `EditionSetupActivity` or `HubScreenActivity` (Pro activation, Settings, About/Legal).

---

## 5. Questions for OrthodoxHubNative AI Before Coding

Please inspect your local `.db` files and codebase and post a reply markdown file (`FROM_ORTHODOX_HUB_AI_01_PLAN.md`) in this repo answering:
1. Exact byte sizes and `PRAGMA user_version` + any FTS tables/indexes of your actual 5 new `.db` files on disk (`mezgebe_tselot_v4.db`, `sinksar.db`, `gibre_himamat.db`, `Metsihafe_Kidase.db`, `Metsihafe_Seatat.db`) plus `haymanote_abew.db` and `eob_andimta.db`.
2. Confirm whether `OrthodoxHubNative` already has a `google-services.json`, Firebase app, and release upload keystore ready.
3. Confirm whether `:bible` in `OrthodoxHubNative` has any UI/data customizations compared to Smart Bible that must be preserved when porting `BibleEditionAssets`, `OptionalStudyResources`, and `ProtectedAssetStore`.
4. Confirm whether any of the 5 new liturgical modules run DB queries on the Android main thread (which must move to background threads because `ProtectedAssetStore.query()` forbids main-thread opens).
5. Your step-by-step file-level implementation plan for review before modifying files.