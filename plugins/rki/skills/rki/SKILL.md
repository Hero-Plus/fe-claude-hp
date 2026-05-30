---
name: rki
description: Cross-repo atlas for HeroPlus Remote Key Injection across heroplus-terminal (Sunmi P3), heroplus-a8 (Landi A8), hero-plus (Rails KifService + SunmiRkiService), and heroplus-kif-bridge (AWS Lambda). Carries the Sunmi key_type enum, the kif-bridge key-type fix, P3 slot 1/2 KCV anchors, Landi A8 binding status, and the KCV-only hygiene rule. Use whenever the user mentions RKI, BDK, IPEK, KCV, DUKPT, KSN, KEK, KBPK, key injection, AWS Payment Cryptography / APC, kif-bridge, KifService, SunmiRkiService, RkiClient, Sunmi Partners portal, key_type / key_index, saveInitialKey / uploadKey / assignKey, RKIS protocol (initRemoteKeyLoadProc / authKMSCrt / checkCIDandSignData / loadRKISv2RemoteKey), KPay KDP, Landi RKIS, MYHSM, Utimaco, rlbdk, or any of the KCVs C41B33 / 2378EB / 2B8C2A / 8CA64DE9 / 4EAEFF / B26C87 / 2E3A98 / CCA41D7F. Lean toward loading — RKI threads weave across 4 repos and the matching code paths are hard to identify by grep alone.
---

# HeroPlus Remote Key Injection — cross-repo atlas

This skill is the source of truth on conflict for HeroPlus RKI state across four repos. Read this first, then drill into in-repo docs for depth. Never copy load-bearing facts (KCVs, slot maps) into other docs — link back here instead, so a fix lands in one place.

## The KCV-only rule

Record only **KCVs** (3-byte check values, `TDES_ECB(0…0, K)[:3]`) and public **resource identifiers** (AWS `KeyArn`, certificate KCVs, KSN, key-name labels). Never commit:

- Plaintext key material (raw key bytes, hex strings)
- TR-31 / TR-34 wrapped key blocks (even though the wrap is under a KEK held by the counterparty)
- OAEP cryptograms
- PIN block ciphertext / decrypt outputs

If material is committed by mistake, git history retains it permanently — rotation of the affected key is the only clean remedy. Sandbox key rotation is low-cost; do it rather than leaving a wrapped block in history.

This rule applies to every artifact: source files, doc files, commit messages, PR descriptions, Slack/WhatsApp paste-throughs preserved into docs, screenshots committed to the repo.

## Sunmi `key_type` enum (load-bearing)

From `hero-plus/docs/02-INTEGRATION/sunmi-rki/README.md` and confirmed by the working `SunmiRkiService` upload path:

| Value | Meaning |
|---|---|
| 1 | KEK (Key Encrypt/Exchange Key) |
| 2 | TMK |
| 3 | PIK (PIN Key) |
| 4 | MAK (MAC Key) |
| 5 | TDK (Track Encryption Key) |
| 7 | DUKPT_BDK |
| 8 | DUKPT_IPEK |
| 9 | KBPK |

`uploadKey` requires `ksn` when `key_type` is 7 or 8. The Sunmi Partners portal renders these as `(KEK)Key Encrypt/Exchange Key` and `(DUKPT_BDK)DUKPT Base Derivation Key` in the "Key Type" column.

## kif-bridge key-type bug (FIXED 2026-05-29)

`heroplus-kif-bridge/src/lib/operations/export_dukpt_ipek_to_sunmi.rb` originally defined `SUNMI_KEY_TYPE_BDK = 1`, used on the `uploadKey` call. **`1` is KEK, not a DUKPT type** (per the enum above), so the 12 May + 27 May uploads landed at Sunmi as `(KEK)`, KCV `4EAEFF`, KSN `--` — the IPEK never reached a DUKPT slot even though the portal-side Key Download Task showed "Succeeded." That asymmetry is the standing lesson: **portal "Succeeded" is not proof of in-slot state** — only a device-side read is.

**Fixed:** the Lambda uploads a pre-derived IPEK, so the correct `key_type` is `8` (DUKPT_IPEK). PR `Hero-Plus/heroplus-kif-bridge#1` changed `1 → 8`; PR `#2` added the `ksn:` parameter that `key_type: 8` requires (`SunmiClient#upload_key` previously sent none, which `key_type: 8` rejects with "IPEK key invalid"). Lambda redeployed. The 28 May upload now classifies as `(DUKPT_IPEK)` with a bound KSN. A local clone may lag — HEAD `ba61ef3` still shows `= 1`; trust the deployed Lambda + merged PRs over the checkout.

Post-fix, device-confirmed on `P3022556J0403` (2026-05-29): the 28 May `(DUKPT_IPEK)` upload reached the device — `dukptCurrentKSN(1)` returns `rc=0`, KSN `FFFF9876543210E00001`, matching the upload. The full pipeline is validated (see "sequencing — resolved" below); the earlier `(KEK)` rows never reached the slot, as expected.

## kif-bridge sequencing — RESOLVED (2026-05-29)

The earlier hypothesis here held that `4EAEFF` was a placeholder/test key, not `rlbdk`, because `4EAEFF ≠ C41B33`. **That was wrong** — it compared an IPEK-KCV to a BDK-KCV, which never match (an IPEK is `derive(BDK, IKSN)`, a different key). Offline ANSI X9.24-1 derivation confirms `4EAEFF` IS the IPEK of the real Nomupay BDK: `derive(Nomupay-BDK/C41B33, IKSN FFFF9876543210E0)` → IPEK-KCV `4EAEFF`, self-validated by reproducing slot-2's `2B8C2A` from the same BDK under KSN `FFFF99F3BB…`. So the Lambda's `BDK_ARN` points at the real Nomupay BDK.

It follows that **HP-APC demonstrably holds the Nomupay sandbox BDK (`C41B33`)** — the IPEK could not derive to `4EAEFF` otherwise — which disproves the earlier "APC may not yet hold the BDK" worry. The code observation still stands as a fact (`KifService#import_key` at `kif_service.rb:161` has no committed caller, and `SunmiRkiService` bypasses APC for slot 2), but the BDK reached APC by some path regardless; exactly how it was imported (ceremony completed, or a hand-run import) is unconfirmed and no longer load-bearing.

Both halves of the old fix chain are effectively done: the BDK is in APC, and `key_type` is fixed (PRs #1/#2). The pipeline `APC → kif-bridge → Sunmi Cloud → device DUKPT slot` is validated end-to-end on `P3022556J0403` (index 1, used as a deliberate scratch slot for the test — see slot map). The one remaining caveat is operational, not a pipeline defect: the test run used KSN `FFFF9876543210E00001` (IIN `987654`, a sequential placeholder); PIN blocks under it would not decrypt at Nomupay unless `987654` is a registered BDK-ID, so a real injection should use the terminal's registered IIN `99F3BB`.

## Device slot maps (current state)

### Sunmi P3 (`P3022556J0403`)

| Slot | Acquirer | IPEK-KCV | Current KSN (device) | Path that injected it       |
| ---- | -------- | --------------- | ----------- | --------------------------- |
| 1    | (was AWX) | `4EAEFF`       | `FFFF9876543210E00001` | kif-bridge / APC — **scratch-slot validation test, 2026-05-29** |
| 2    | Nomupay  | `2B8C2A`        | `FFFF99F3BB…` (counter advances with use) | `SunmiRkiService` (local-component reconstruction) |

Slot 2 is the live Nomupay key — the device reads `PIN_KEY_INDEX = 2`; it binds to the Nomupay sandbox BDK (`C41B33`) and PIN-encrypt round-trips through Nomupay's HSM.

As of 2026-05-29, slot 1 holds the kif-bridge APC validation IPEK (`4EAEFF`, IIN `987654`), device-confirmed via `dukptCurrentKSN(1)` (`rc=0`). This was a deliberate end-to-end pipeline test using index 1 as a scratch slot; it overwrote the prior Airwallex IPEK (`2378EB`, KSN `B010…0015`). Both AWX and Nomupay keys are to be re-injected to their intended slots now that the APC pipeline is validated.

### Landi A8 (`216PCA8C2654`)

| Slot | Acquirer | Pad `calcKCV` | Binding identity                  |
| ---- | -------- | ------------- | --------------------------------- |
| 2    | Nomupay  | `8CA64DE9`    | **OPEN** — not yet round-tripped  |

`calcKCV` on this firmware returns a 4-byte non-canonical value (not `TDES_ECB(0…0, K)[:3]`); see `heroplus-a8/docs/research/ocg-integration/10-open-questions.md` Q44. The 4-step Landi RKIS protocol (`initRemoteKeyLoadProc` → `authKMSCrt` → `checkCIDandSignData` → `loadRKISv2RemoteKey`) completes end-to-end and writes a key into the slot, but whether the key is `rlbdk` (vs a KPay-side test/bootstrap key) is unconfirmed until a PIN-encrypt → Nomupay-HSM decrypt round-trip succeeds.

A8's BDK path is **independent** of kif-bridge — it goes through KPay's RKIS endpoint (`demokdp.unimarspay.net:30000` for sandbox), driven by `heroplus-a8/features/rki/RkiClient.kt`. So fixing the kif-bridge bug does not affect A8.

## Identity / KCV registry (safe to record)

**Naming.** `rlbdk` is **KPay's KDP console name** for the key with KCV `C41B33` (per `heroplus-a8/docs/research/rki-study.md` — "KPay's name for this BDK in their console"). The same key bytes are the **Nomupay sandbox BDK** in HP infrastructure (HP-APC, Sunmi Partners portal, Sunmi P3 slot 2 — which arrives via `SunmiRkiService`, never through KPay). Use "Nomupay sandbox BDK" in HP / Sunmi / Nomupay contexts; reserve `rlbdk` for KPay-KDP and Landi-A8 contexts, where the key is enrolled under that name. A KCV match proves identical key bytes, not shared provenance — so the name is what tells you whose copy you hold and which injection path delivered it.

| Item | KCV | Where |
|---|---|---|
| Nomupay sandbox BDK (TDES_2KEY) — KPay's KDP console names the same bytes `rlbdk` | `C41B33` | HP-APC + Sunmi Partners portal (slot-2 source); KPay KDP holds it under the name `rlbdk`; intended for Landi A8 slot 2 (binding unverified — see slot map). Confirmed in HP-APC — `4EAEFF` IPEK derives from it |
| Airwallex sandbox BDK | `94C0E9` | Sunmi Partners portal (slot 1 source) |
| AWX IPEK @ P3 slot 1 | `2378EB` | KSN `B0100000000000000015` |
| Nomupay IPEK @ P3 slot 2 | `2B8C2A` | KSN `FFFF99F3BB0000100000`, IIN `99F3BB` |
| TR-34 KEK HP-APC ↔ KPay-APC | `B26C87` | KPay-APC `KeyArn …rjgodpksadtvewjc`, TR31_K1_KEY_BLOCK_PROTECTION_KEY, TDES_3KEY |
| HP-APC ECDH sandbox cert chain | `CCA41D7F` | Nomupay-MYHSM bootstrap (pending) |
| Sunmi Initial Key (P3 transport KEK) | `2E3A98` | Sunmi Cloud, healthy |
| kif-bridge IPEK (derived from the Nomupay sandbox BDK; was mis-typed as KEK) | `4EAEFF` | P3 slot 1 — `derive(Nomupay-BDK/C41B33, KSN FFFF9876543210E00001)`; IIN `987654` is a test value |
| Pad `calcKCV(2)` on A8 `216PCA8C2654` | `8CA64DE9` | Non-canonical 4-byte format on this firmware |

KSN width: 10 bytes / 20 hex (ANSI X9.24-1 TDES-DUKPT). The Sunmi portal screenshot truncates display to 19 hex chars; the actual KSN is 20.

## Per-repo file index

The atlas is thin by design — drill into these files for depth. The atlas is authoritative on conflict between the docs.

### `hero-plus` (Rails, the orchestrator)

Branch model: `develop` auto-deploys to dev; `master` auto-deploys to production. RKI changes currently live on `develop` — default to that branch when grep'ing for paths in this section or running `git log`; `master` will lag.

| Concern | File |
|---|---|
| Working path (slot 2 today) | `app/services/sunmi_rki_service.rb` — `reconstruct_bdk:455` XORs three components from Rails credentials; uploads with `key_type: 7` at `:340` |
| BDK components source | Rails credentials at `nomupay.pos.bdk_component_1/2/3` (see `sunmi_rki_service.rb:558-573`) |
| APC orchestration (validated end-to-end 2026-05-29) | `app/services/kif_service.rb` — `import_key:161` (no committed caller, yet APC holds the BDK), `export_dukpt_ipek_to_sunmi:282`, `inject_dukpt_ipek_to_terminal:314` |
| TR-31 usage constants (correct in BE) | `kif_service.rb:53-55` — `KEY_USAGE_BDK = 'TR31_B0_BASE_DERIVATION_KEY'`, `KEY_USAGE_KEK = 'TR31_K1_KEY_BLOCK_PROTECTION_KEY'` |
| Sunmi key_type enum reference | `docs/02-INTEGRATION/sunmi-rki/README.md` |
| Test spec asserting BDK identity | `spec/services/sunmi_rki_service_spec.rb:36` (`combined_bdk_kcv = 'C41B33'`) |

### `heroplus-kif-bridge` (AWS Lambda)

| Concern | File |
|---|---|
| key-type fix | `src/lib/operations/export_dukpt_ipek_to_sunmi.rb` — was `SUNMI_KEY_TYPE_BDK = 1` (KEK); now `key_type: 8` + `ksn:` via PRs #1/#2 (local clone may lag at `ba61ef3`) |
| Sunmi REST client | `src/lib/sunmi_client.rb` (POST `/key/saveInitialKey`, `/key/uploadKey`) |
| APC client | `src/lib/apc_client.rb` (`export_dukpt_ipek_oaep`) |
| Architecture diagram | `README.md` |

### `heroplus-terminal` (Sunmi P3 device-side)

| Concern | File |
|---|---|
| Per-acquirer slot constants | `modules/sunmipos/android/src/main/java/expo/modules/sunmipos/nomupay/CardReaderModule.kt:55` (`PIN_KEY_INDEX = 2`); AWX side uses index 1 in `CardAwxModule.kt` |
| Slot map + injection paths (descendant doc) | `modules/sunmipos/CLAUDE.md` § "DUKPT key slots" |
| AID/CAPK handoff between Nomupay ↔ AWX | `.claude/docs/acquirer-handoff.md` (separate concern — EMV-layer, not RKI-layer) |
| Sunmi SDK error code reference | `modules/sunmipos/.claude/reference/sunmi-paysdk-demo/app/libs/.../SPErrorCode.java` |

### `heroplus-a8` (Landi A8 device-side, separate RKI path)

| Concern | File |
|---|---|
| 4-step RKIS client | `app/src/main/kotlin/com/heroplusgroup/a8/features/rki/RkiClient.kt` + `RkiConfig.kt` |
| Pad probe / diagnostic | `app/src/main/kotlin/com/heroplusgroup/a8/features/hardware/landi/BdkKcvDiagnostic.kt` |
| RKI study guide | `docs/research/rki-study.md` |
| APC architecture (BE-side, A8-relevant slice) | `docs/research/payment-cryptography-architecture.md` |
| RKI + key-management deep dive | `docs/research/ocg-integration/06-rki-and-key-management.md` |
| Open questions | `docs/research/ocg-integration/10-open-questions.md` (Q1, Q2, Q17, Q32, Q33, Q39, Q44) |
| Doc style hook | `.claude/rules/docs-research-style.md` (replace-don't-annotate; PostToolUse-enforced) |

### External source materials (not committed, on local disk)

| Concern | Path |
|---|---|
| Working slot-2 Postman injection trace | `/Users/spring/Code/job/rki/README_nomupay.ts` |
| Email thread (Jason ↔ Nomupay ↔ Utimaco) | `~/Desktop/jason_rki.pdf` |
| WhatsApp thread (Kai @ KPay) | `~/Downloads/KPay_Heroplus_Jason_Whatsapp/_chat.txt` |

## Sunmi 6-step RKI flow (canonical)

The working `SunmiRkiService` path follows:

1. `getProtectionKey` — fetch Sunmi session protection key (RSA public, used to wrap step 2)
2. `saveInitialKey` — upload a transport KEK (PKCS1 v1.5 under the protection key); Sunmi returns `initial_key_id` for use in step 3
3. `uploadKey` — upload the operational key encrypted under the transport KEK from step 2, with `key_type` (use 7 for BDK, 8 for IPEK) + `key_index` + `ksn` (required for type 7/8) + `iin` (BDK-ID from KSN) + `package_name`
4. `assignKey` — bind the uploaded key to a device SN
5. `createTask` — create a download task for the device
6. Device unlocked + online — task fires, Sunmi pushes the key into the device's DUKPT slot

The kif-bridge Lambda implements steps 2 + 3 (`saveInitialKey` + `uploadKey`); steps 1, 4, 5 are owned by `KifService` in BE. Step 6 is operational.

## Sunmi Partners portal facts

- "Key Type" column shows `(KEK)Key Encrypt/Exchange Key` for `key_type=1` rows, `(DUKPT_BDK)DUKPT Base Derivation Key` for `key_type=7`. Pre-derived IPEK uploads (`key_type=8`) display similarly under DUKPT.
- "View Key Derivation Record" link appears only on DUKPT_BDK (`key_type: 7`) rows. KEK rows and pre-derived DUKPT_IPEK (`key_type: 8`) rows have no BDK→IPEK chain to show — so the portal cannot reveal the BDK behind a `key_type: 8` upload; only a device-side KSN read + offline derivation confirms its lineage.
- "KSN(IPEK)" column shows `--` for KEK rows; real DUKPT IPEKs always carry a KSN.
- Key Index is a real routing field, not parsed from key-name substrings — BE explicitly sets it. The kif-bridge runs targeted Index 1 correctly; the original mistype was `key_type` (now fixed), never `key_index`.
- "More" dropdown on a kif-bridge KEK row offers only `Delete` — no portal-side "Reclassify as DUKPT_BDK". The fix lives in the Lambda, not in the portal.

## Cross-references

- `heroplus-terminal/.claude/docs/acquirer-handoff.md` — separate concern (AID/CAPK contamination between Nomupay ↔ AWX), but adjacent to RKI in the multi-acquirer architecture
- `heroplus-a8/.claude/rules/docs-research-style.md` — replace-don't-annotate rule; **read first** before editing any file under `heroplus-a8/docs/research/`
- `heroplus-terminal/modules/sunmipos/.claude/rules/sunmipos-paysdk.md` — Sunmi PaySDK fragile patterns (DUKPT KSN advancement, EMV CVM fallback, etc.)
