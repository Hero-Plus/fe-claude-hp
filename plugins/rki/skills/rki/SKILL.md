---
name: rki
description: Cross-repo atlas for HeroPlus Remote Key Injection across heroplus-terminal (Sunmi P3), heroplus-a8 (Landi A8), be-heroplus (Rails KifService + SunmiRkiService), and heroplus-kif-bridge (AWS Lambda). Carries the Sunmi key_type enum, the kif-bridge key-type bug, P3 slot 1/2 KCV anchors, Landi A8 binding status, and the KCV-only hygiene rule. Use whenever the user mentions RKI, BDK, IPEK, KCV, DUKPT, KSN, KEK, KBPK, key injection, AWS Payment Cryptography / APC, kif-bridge, KifService, SunmiRkiService, RkiClient, Sunmi Partners portal, key_type / key_index, saveInitialKey / uploadKey / assignKey, RKIS protocol (initRemoteKeyLoadProc / authKMSCrt / checkCIDandSignData / loadRKISv2RemoteKey), KPay KDP, Landi RKIS, MYHSM, Utimaco, rlbdk, or any of the KCVs C41B33 / 2378EB / 2B8C2A / 8CA64DE9 / 4EAEFF / B26C87 / 2E3A98 / CCA41D7F. Lean toward loading — RKI threads weave across 4 repos and the matching code paths are hard to identify by grep alone.
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

From `be-heroplus/docs/02-INTEGRATION/sunmi-rki/README.md` and confirmed by the working `SunmiRkiService` upload path:

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

## kif-bridge bug (current state)

`heroplus-kif-bridge/src/lib/operations/export_dukpt_ipek_to_sunmi.rb:37` defines:

```ruby
SUNMI_KEY_TYPE_BDK = 1
```

Used at `:131` on the `uploadKey` call. **`1` is KEK, not DUKPT_BDK** (per the enum above). Every kif-bridge upload lands at Sunmi as `(KEK)` — visible in the Partners portal — so the IPEK never reaches a DUKPT slot. The portal-side Key Download Task succeeds; the device-side DUKPT slot enumeration cannot see the key because it's not stored as a DUKPT key.

The Lambda derives a per-terminal IPEK inside APC and uploads that pre-derived IPEK, so the matching value is `8` (DUKPT_IPEK). Alternative: redesign the Lambda to upload the BDK with `key_type: 7` (DUKPT_BDK) and let Sunmi derive — that mirrors the working `SunmiRkiService` path in BE. Pick is a BE call.

Empirical observation on `P3022556J0403`: two kif-bridge runs (12 May + 27 May 2026) at Key Index 1. Both portal-side rows show Key Type `(KEK)`, KCV `4EAEFF`, KSN `--`. Device-side slot 1 still holds the original Airwallex IPEK (KCV `2378EB`).

Secondary deviation: the Lambda sets `iin: ipek_kcv` (`:137`). The working `SunmiRkiService` shape sets `iin` to the BDK-ID nibbles extracted from the KSN (e.g. KSN `FFFF99F3BB0000100000` → `iin: "99F3BB"`). Either is accepted by Sunmi today, but the BDK-ID convention is what the portal expects semantically.

## kif-bridge sequencing hypothesis (current state)

The kif-bridge uploaded key's KCV is `4EAEFF` — which is **not** the canonical TDES-KCV of the Nomupay sandbox BDK (`rlbdk`, BDK-KCV `C41B33`). So even if `key_type` is fixed, the `BDK_ARN` the Lambda derived from almost certainly points at a different key in APC (a test/bootstrap key), not the real Nomupay BDK. Hedge: `BDK_ARN` is caller-supplied — BE owns the `rake` invocation that fed it. Asking BE for the literal `BDK_ARN` used in the 12 + 27 May runs would settle this; until then the inference rests on the KCV non-match.

Why this is likely: `be-heroplus`'s `KifService#import_key` (`app/services/kif_service.rb:161`) — the MYHSM→APC TR-31 / ECDH import — has **no committed caller**. The only `@client.import_key` reference outside that method is a trust-anchor cert import at `:106`, not the BDK. So in committed BE code, the Nomupay BDK was never imported into HP-APC; the working slot-2 path (`SunmiRkiService`) **bypasses APC** by reconstructing the BDK in Rails-process memory from three components in credentials. Counter-evidence to watch for: an unmerged BE branch, or a hand-run `rails console` invocation of `KifService#import_key`, that did the import without committing — only BE can confirm or refute.

Per the BE-side email thread (Jason ↔ Nomupay ↔ Utimaco): the MYHSM → HP-APC ECDH bootstrap is still in sandbox setup as of late May 2026 — sandbox `.pem` chain exchanged (`hp_ecdh_root_sandbox.pem`, `hp_ecdh_leaf_sandbox.pem`, ECC P-521, chain KCV `CCA41D7F`), ceremony pending.

So the full fix chain has two parts:

1. **Run the MYHSM → HP-APC import ceremony**, getting `rlbdk` (KCV `C41B33`) imported into HP-APC. Then `KifService` has something to point kif-bridge at.
2. **Fix `SUNMI_KEY_TYPE_BDK` in the Lambda** (or restructure to BDK-upload-mode with `key_type: 7`).

Both are required. Either alone leaves the system broken.

## Device slot maps (current state)

### Sunmi P3 (`P3022556J0403`)

| Slot | Acquirer | IPEK-KCV (CHK0) | KSN counter | Path that injected it       |
| ---- | -------- | --------------- | ----------- | --------------------------- |
| 1    | AWX      | `2378EB`        | `B010…0015` | Sunmi Partners portal (Airwallex BDK enrollment, ~April 2026) |
| 2    | Nomupay  | `2B8C2A`        | `FFFF99F3BB0000100000` | `SunmiRkiService` (local-component reconstruction) |

Both slots are working — slot 2 binds cleanly to `rlbdk` and PIN-encrypt round-trips through Nomupay's HSM.

The kif-bridge `(KEK)` uploads at Key Index 1 (12 + 27 May) did NOT replace slot 1's AWX IPEK — KEKs land in the KEK keystore, not the DUKPT slot. So slot 1 remains the Airwallex IPEK indefinitely unless either (a) the kif-bridge bug is fixed and a proper DUKPT_BDK/DUKPT_IPEK upload supersedes it, or (b) a `SunmiRkiService`-style local-component path is run for slot 1.

### Landi A8 (`216PCA8C2654`)

| Slot | Acquirer | Pad `calcKCV` | Binding identity                  |
| ---- | -------- | ------------- | --------------------------------- |
| 2    | Nomupay  | `8CA64DE9`    | **OPEN** — not yet round-tripped  |

`calcKCV` on this firmware returns a 4-byte non-canonical value (not `TDES_ECB(0…0, K)[:3]`); see `heroplus-a8/docs/research/ocg-integration/10-open-questions.md` Q44. The 4-step Landi RKIS protocol (`initRemoteKeyLoadProc` → `authKMSCrt` → `checkCIDandSignData` → `loadRKISv2RemoteKey`) completes end-to-end and writes a key into the slot, but whether the key is `rlbdk` (vs a KPay-side test/bootstrap key) is unconfirmed until a PIN-encrypt → Nomupay-HSM decrypt round-trip succeeds.

A8's BDK path is **independent** of kif-bridge — it goes through KPay's RKIS endpoint (`demokdp.unimarspay.net:30000` for sandbox), driven by `heroplus-a8/features/rki/RkiClient.kt`. So fixing the kif-bridge bug does not affect A8.

## Identity / KCV registry (safe to record)

| Item | KCV | Where |
|---|---|---|
| Nomupay sandbox BDK (`rlbdk`, TDES_2KEY) | `C41B33` | KPay KDP + Sunmi Partners portal + HP-APC (intended) |
| Airwallex sandbox BDK | `94C0E9` | Sunmi Partners portal (slot 1 source) |
| AWX IPEK @ P3 slot 1 | `2378EB` | KSN `B0100000000000000015` |
| Nomupay IPEK @ P3 slot 2 | `2B8C2A` | KSN `FFFF99F3BB0000100000`, IIN `99F3BB` |
| TR-34 KEK HP-APC ↔ KPay-APC | `B26C87` | KPay-APC `KeyArn …rjgodpksadtvewjc`, TR31_K1_KEY_BLOCK_PROTECTION_KEY, TDES_3KEY |
| HP-APC ECDH sandbox cert chain | `CCA41D7F` | Nomupay-MYHSM bootstrap (pending) |
| Sunmi Initial Key (P3 transport KEK) | `2E3A98` | Sunmi Cloud, healthy |
| kif-bridge uploaded key (mis-typed as KEK) | `4EAEFF` | P3 Partners portal — placeholder/test key, NOT `rlbdk` |
| Pad `calcKCV(2)` on A8 `216PCA8C2654` | `8CA64DE9` | Non-canonical 4-byte format on this firmware |

KSN width: 10 bytes / 20 hex (ANSI X9.24-1 TDES-DUKPT). The Sunmi portal screenshot truncates display to 19 hex chars; the actual KSN is 20.

## Per-repo file index

The atlas is thin by design — drill into these files for depth. The atlas is authoritative on conflict between the docs.

### `be-heroplus` (Rails, the orchestrator)

| Concern | File |
|---|---|
| Working path (slot 2 today) | `app/services/sunmi_rki_service.rb` — `reconstruct_bdk:455` XORs three components from Rails credentials; uploads with `key_type: 7` at `:340` |
| BDK components source | Rails credentials at `nomupay.pos.bdk_component_1/2/3` (see `sunmi_rki_service.rb:558-573`) |
| APC orchestration (broken end-to-end) | `app/services/kif_service.rb` — `import_key:161` (no caller), `export_dukpt_ipek_to_sunmi:282`, `inject_dukpt_ipek_to_terminal:314` |
| TR-31 usage constants (correct in BE) | `kif_service.rb:53-55` — `KEY_USAGE_BDK = 'TR31_B0_BASE_DERIVATION_KEY'`, `KEY_USAGE_KEK = 'TR31_K1_KEY_BLOCK_PROTECTION_KEY'` |
| Sunmi key_type enum reference | `docs/02-INTEGRATION/sunmi-rki/README.md` |
| Test spec asserting BDK identity | `spec/services/sunmi_rki_service_spec.rb:36` (`combined_bdk_kcv = 'C41B33'`) |

### `heroplus-kif-bridge` (AWS Lambda)

| Concern | File |
|---|---|
| The bug | `src/lib/operations/export_dukpt_ipek_to_sunmi.rb:37` (`SUNMI_KEY_TYPE_BDK = 1`) used at `:131` |
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
- "View Key Derivation Record" link appears only on DUKPT_BDK rows (KEKs have no BDK→IPEK derivation chain to display).
- "KSN(IPEK)" column shows `--` for KEK rows; real DUKPT IPEKs always carry a KSN.
- Key Index is a real routing field, not parsed from key-name substrings — BE explicitly sets it. The kif-bridge runs targeted Index 1 correctly; the upload mistype is `key_type`, not `key_index`.
- "More" dropdown on a kif-bridge KEK row offers only `Delete` — no portal-side "Reclassify as DUKPT_BDK". The fix lives in the Lambda, not in the portal.

## Cross-references

- `heroplus-terminal/.claude/docs/acquirer-handoff.md` — separate concern (AID/CAPK contamination between Nomupay ↔ AWX), but adjacent to RKI in the multi-acquirer architecture
- `heroplus-a8/.claude/rules/docs-research-style.md` — replace-don't-annotate rule; **read first** before editing any file under `heroplus-a8/docs/research/`
- `heroplus-terminal/modules/sunmipos/.claude/rules/sunmipos-paysdk.md` — Sunmi PaySDK fragile patterns (DUKPT KSN advancement, EMV CVM fallback, etc.)
