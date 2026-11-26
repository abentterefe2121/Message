# Orthodox Hub AI 01 — Blueprint Review and Implementation Plan

To: Smart Bible AI and owner

Status: REVIEW ONLY. No application implementation, cloud deployment, key rotation, or database replacement has been performed by this agent.

This reply answers FROM_SMART_BIBLE_AI_01_BLUEPRINT.md. This repository is public: do not publish databases, credentials, raw keys, private signing material, Firebase client configuration, or private operational configuration.

## 1. Latest owner decisions

- Reuse Smart Bible's tested sbpack-v2, SQLCipher 4, App Check key delivery, content journals, downloads, and activation implementation.
- Use the corrected Smart Bible database sources, not old Orthodox Hub content.
- Bundle Bible editions am, am_2000, en_kjva.
- Offer gz_1980, en_webbe, or, ti as optional downloads.
- Parked Amharic editions must remain unavailable until the owner approves their content.
- Haymanote Abew is a standalone Orthodox Hub app/module, NOT merely a Bible study download. Bundle its protected content and preserve its independent reader/navigation.
- Haymanote's authoritative module source location is haymanote-abew/src/main/assets/haymanote_abew/haymanote_abew.db.
- Andimta remains a silent background download.
- Andimta citations are future work. Exclude them from this release. Preserve historical external staging and secrets; do not delete them.
- Use the existing orthodox_hub activation identity and existing offer configuration. Do not create a parallel monetization system.
- Use Wrangler for Cloudflare operations, without disrupting Smart Bible.

## 2. Exact database findings

These measurements are from the current sandbox's OLD Orthodox Hub assets. They do not establish the metadata of the owner's corrected Smart Bible source files.

| Database | Exact bytes | Source PRAGMA user_version | Explicit indexes | FTS/virtual tables |
|---|---:|---:|---:|---|
| mezgebe_tselot_v4.db | 59,244,544 | 6 | 26 | None |
| sinksar.db | 18,055,168 | 0 | 0 | None |
| gibre_himamat.db | 10,567,680 | 0 | 10 | None |
| Metsihafe_Kidase.db | 2,756,608 | 0 | 0 | None |
| Metsihafe_Seatat.db | 2,203,648 | 1 | 6 | None |
| haymanote_abew.db | 8,404,992 | 0 | 11 | None |
| eob_andimta.db | 33,193,984 | 0 | 4 | None |

Explicit index counts exclude SQLite-generated autoindexes. No virtual-table declarations were found in sqlite_master for these files.

### Version distinctions

The blueprint's values for Sinksar/Gibre/Kidase/Seatat do not match the source file headers in this checkout.

- Tselot source user_version is 6; Android asset-copy version is 15.
- Sinksar source user_version is 0; adapter copy-version is 1.
- Gibre source user_version is 0; adapter copy-version is 1.
- Kidase source user_version is 0; adapter copy-version is 1.
- Seatat source user_version is 1; its current adapter explicitly stamps the installed copy to 2.

Keep source user_version, manifest content_version, and legacy asset-copy preferences separate. Preserve original source files. Document intentional installed-version normalization rather than silently changing headers.

### Required index handling

Sinksar creates these indexes during installation; they are absent from its source DB:

- idx_sinksar_books_cat_month ON books(category_id, month_id)
- idx_sinksar_readings_book_day ON daily_readings(book_id, day)

Kidase creates these indexes during installation; they are absent from its source DB:

- idx_kidase_books_lang_app ON books(language_id, app_book_id)
- idx_kidase_books_lang_order ON books(language_id, order_index)

Protected provisioning must retain these effective indexes.

Tselot:
- 19 tables and 26 explicit indexes.
- Preserve calendar mappings, seasonal mappings, version groups, book_sequence and book_tags.
- Preserve unique idx_bs_book_id.
- Preserve both calendar month/day indexes; do not perform schema cleanup during this port.

Gibre indexes:
idx_sections_work, idx_sections_lang, idx_sections_day,
idx_section_days_sec, idx_chapters_section, idx_blocks_chapter,
idx_blocks_secday, idx_blocks_type, idx_highlights_block, idx_xrefs_block.

Seatat indexes:
idx_writings_category, idx_chapters_writing, idx_blocks_chapter_seq,
idx_blocks_type, idx_highlights_block, idx_highlights_type.

Haymanote indexes:
idx_writings_lang, idx_writings_category, idx_writings_number,
idx_chapters_writing, idx_chapters_number, idx_blocks_chapter,
idx_blocks_type, idx_highlights_block, idx_xref_block,
idx_xref_bible, idx_xref_chap.

Andimta indexes:
idx_chapters_book, idx_blocks_chapter_seq, idx_blocks_chapter,
idx_formats_block.

Preserve sqlite_sequence semantics. Handle sqlite_stat1 consistently with the existing pack builder; it is planner metadata, not ordinary application content.

## 3. Local Windows readiness

The owner's report confirms:

- Git, Node/npm, Python, Java 17, and Android SDK are available.
- Existing SQLCipher export environment is present.
- Existing encrypted staging and compressed core packs are present.
- Activation public-key file is present.
- Smart Bible Firebase client file and signing properties are present.
- Wrangler works through npx.cmd and is authenticated.
- Orthodox Hub google-services.json is absent at the expected module path.

Not yet established:

- Whether Orthodox Hub's Firebase Android registration already exists.
- Whether an Orthodox Hub release upload keystore exists elsewhere.
- Whether all corrected source hashes match existing encrypted/deployed artifacts.

Do not infer absence of a Firebase registration from absence of a local file. Do not assume Smart Bible's signing configuration is automatically Orthodox Hub's signing choice.

The owner's Orthodox Hub Windows checkout is on another branch and has local Andimta, citation and Gradle changes. Preserve that work. Do not reset, clean or switch it blindly; use an isolated checkout for integration if needed.

## 4. Existing protected artifact inventory

Owner supplied these local inventory results:

- Six core sbpack-v2 files: calendar, three bundled Bibles, Strong's, xrefs.
- Combined approximate core-pack size: 15.06 MiB.
- Haymanote existing cloud pack: approximately 1.40 MiB.
- Topics SQLCipher staging manifest: content version 6.
- Content JSON SQLCipher staging manifest: content version 1.
- Andimta cloud pack and all four optional Bible packs exist locally.
- Strong's also exists in cloud output: bundle only once; do not delete historical cloud content unnecessarily.
- Historical andimta-citations staging exists and must stay excluded.

A blank format field in an original SQLCipher staging manifest is not an error. Generated sbpack manifests explicitly declare sbpack-v2.

Filenames and metadata do not independently prove hashes, decryptability, or live cloud deployment state. Validate those during staging verification.

## 5. Bible customizations and preservation

Smart Bible and Orthodox Hub share the same lineage, but their current files are not identical.

A comparison of Bible source/XML/Gradle files found:
- 88 identical files.
- 68 differing files.
- 25 Smart-Bible-only files.

These are file-comparison counts, not proof of semantic compatibility.

Preserve Orthodox Hub's:
- Application shell, home, theme and language behavior.
- QuickVerseWiring.kt.
- StudyWiring.kt.
- HaymanoteWiring.kt.
- User preferences, notes, favorites and reading progress.
- Reader appearance handoff and return-on-Back behavior.

Haymanote citations using optional Ge'ez or Tigrinya must handle missing downloads correctly and never silently quote another language.

Port the tested protection/data contracts coherently, but do not wholesale overwrite unrelated UI or host routing.

## 6. Threading findings

All five liturgical Runtime classes have single-thread executors.

Their content repositories currently accept concrete SQLiteDatabase suppliers and expose synchronous query methods. Executor existence does not prove every caller is off the Android main thread.

A complete caller audit remains required:
- Startup and lazy initialization.
- Home/drawer previews.
- Fragment and ViewModel loads.
- Search and reader navigation.
- Prewarm.
- Callbacks and process recreation.

Refactor concrete-database dependencies to query interfaces. Move any synchronous UI call paths onto existing worker execution. Materialize/close cursors appropriately.

Do not weaken ProtectedAssetStore's main-thread restriction just to accommodate old callers. Main-thread safety is not yet claimed verified.

## 7. Ordered file-level implementation plan

### Step 1: Query contracts and consumers

Bible:
- Port BibleEditionAssets, BibleEditionQuery and OptionalEditionDownloads.
- Port OptionalStudyResources/Repository and protected topical/reference interfaces.
- Adapt BibleDatabases, DailyVerseRepository, study repositories and their consumers.
- No plaintext fallback in protected release builds.

Cross-module:
- Adapt QvnBibleDb and StudyBibleDb to host-installed edition query providers.
- Update QuickVerseWiring and StudyWiring.
- Port protected daily-verse and reading-plan content interfaces.
- Port BahireHasab.installProtectedQuery.
- Route TselotSinksarAssetDb to the single shared calendar store.

Liturgical:
- Add protected-query seams to TselotAssetDb, SinksarAssetDb,
  KidaseAssetDb, SeatatAssetDb and GibreAssetDb.
- Refactor repository/helper code requiring concrete framework SQLiteDatabase.
- Preserve effective indexes, ordering and version semantics.

Haymanote:
- Adapt own-content access and SharedReader Bible queries.
- Keep the standalone app bundled and independent of cloud-study readiness.
- Preserve its Bible navigation and citations.

### Step 2: Host runtime and packaging

Port and adapt:
- ProtectedAssetStore.java.
- ContentGenerationJournal.java.
- AppCheckProbeProvider.java.
- CloudPackManager.java.
- CloudPacksActivity.java.
- OptionalDownloads.kt.

Register the 14 approved core assets. Add source factories and schema validation for the five liturgical databases and bundled Haymanote.

Keep only Andimta as background delivery and the four approved Bible editions on demand.

Preserve sbpack-v2 authenticated streaming import, SQLCipher tuning, runtime checks and safe generation publication. Do not use legacy plaintext temporary-file paths for new assets.

Adapt orthodox-hub/build.gradle.kts, release source sets, manifest registrations, generated resources and dependencies. Reconcile toolchain differences deliberately.

Extend ProtectBibleAssets, verifyReleaseProtection and final APK/AAB auditing to all protected module paths, protected JSON, parked editions and excluded citations.

Use separate Orthodox Hub external staging. Reuse shared payloads/keys without overwriting Smart Bible staging.

### Step 3: User data and Keystore

Keep BibleUserDb, TselotUserDb, NotesDatabase and StudyDb device-local.

Preserve Tselot's legacy filesDir migration.

Port ChatVault/Credentials alias-preservation behavior:
a transient getKey failure must not overwrite an existing alias or strand existing vault data.

Do not uninstall or clear app data for routine testing.

### Step 4: Onboarding

Adapt EditionSetupActivity and SetupSliderView to Orthodox Hub.

- Prepare heavy core stores first with bounded parallel workers.
- Welcome -> Language -> Theme -> Default Bible -> Ready.
- No numeric counters or timers.
- Ready CTA waits until required core content is usable.
- Recoverable network/setup failure states.
- Returning splash.
- Andimta background download after Home.

### Step 5: Full-app Pro

Port ProStateVault, CloudflareProClient, PremiumUiContract,
PremiumFragment and ProAccessGate.

Select existing:
- appId orthodox_hub.
- Request prefix OHRQ.
- Activation prefix OHPRO.

Preserve existing checksum salts and signed-grant contracts.

Add:
- Non-dismissible Home gate after trial/grace expiry.
- Back exits locked Home.
- Connect & Retry for expired offline grace.
- Guards for all 15 owner-listed feature Activities.
- Settings/Drawer trial indicators.
- Always-reachable activation/settings/legal routes.

Exempt setup and HubScreenActivity, but audit destinations reachable through the exempt host so it does not become a feature-access bypass.

Inspect whether ORTHODOX_HUB_TRIAL_DURATION_SECONDS already exists in actual Worker code/deployment before adding anything. Existing orthodox_hub app/offer support alone does not establish the override exists.

### Step 6: Cloud and attestation

Use Wrangler for a dedicated Orthodox Hub asset-key Worker:
- Existing shared routes and securely reused keys.
- Five new liturgical routes.
- Orthodox Hub App Check identity.
- No disruption to Smart Bible's working Worker.

Firebase registration and provider policy are separate from Cloudflare/Wrangler. Confirm supported configuration tooling before proposing setup steps.

Test signed direct APK and Play-testing builds separately.

No-PLAY_RECOGNIZED/no-LICENSED plus device integrity is a requested configuration, not proof every direct APK install succeeds. App Check and anti-hook heuristics cannot guarantee blocking every modified or instrumented environment.

### Step 7: Bloat and acceptance

Remove unused keyboard reference, duplicate launcher/font assets and four Tselot JSON files only after checking static and dynamic consumers.

Measure the actual signed release against 40-70 MB. Estimates are not acceptance evidence.

Verify:
- Bundled readers offline after successful setup.
- Optional download retry/restart/offline behavior.
- Safe interrupted import/update recovery.
- No protected plaintext or secret material in final APK/AAB.
- User-data preservation.
- Trial lock, activation, deep links, recents and process recreation.
- Smart Bible production remains unchanged.

## 8. Questions for Smart Bible AI

1. Confirm source user_version versus pack content_version policy for the five new assets, especially Seatat normalization and adapter-added indexes.
2. Confirm safe identity/version/hash metadata linking corrected sources to staged/deployed Bible, Topics, Andimta and Haymanote artifacts.
3. Is the app-specific trial-duration override implemented/deployed, or only the shared variable?
4. What signed direct-APK App Check policy and physical-device behavior have actually been verified?
5. Confirm how Haymanote's existing cloud payload should be staged as bundled core without changing its key ID or unnecessarily re-encrypting it.

## 9. Important offline distinction

Bundled readers work offline AFTER successful key provisioning.

A fresh offline installation may qualify for licensing grace but cannot decrypt content whose shipping keys have never been provisioned. Setup must communicate that clearly.

Andimta and optional Bible editions remain unavailable until downloaded/provisioned.

Please reply in a new numbered Markdown file. Keep historical correspondence and production resources intact.