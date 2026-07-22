# LEGO Legacy: Heroes Unboxed — Gate Map & Revival Plan
### Case 027 · from static analysis of the iOS build's managed assemblies

## Build identity

Despite the "(AS)" filename, this IPA is the **Netflix** release, not the standalone:

- Bundle ID: `com.netflix.NGP.LegoLegacyHeroesUnboxed`
- Version: `1.5.1` (Netflix versioning restarted from the standalone's 1.x line)
- Custom URL scheme: `netflix83`
- Embeds `NetflixGames.framework` (30 MB native binary + Hawkins data)

So the Netflix identity layer is physically present in the boot path. Whether it is *load-bearing* for reaching gameplay is the one open question the native binary will settle (see "Open questions").

## The backend it talks to

The whole online stack is **Gameloft's Online Framework**, exposed to managed code through a native binding layer (`Oizys.dll` = P/Invoke stubs into `libnativeengine`). The subsystems, by their internal codenames:

| Codename | Role | Maps to gate |
|---|---|---|
| **CRM** (`CRMSystem`, `FUNC_GetConfigFromServer`) | remote config / feature flags | Gate 1 (online-config) |
| **Gaia** (`GaiaAuthorize`, `GaiaSystem`) | Gameloft identity / account | Gate 2 (login) |
| **Janus** (`RefreshJanusToken`, `GetJanusScopes`) | the auth token Gaia issues | Gate 2 |
| **Hermes** (`RegisterEndpointToHermes`, `hermesInbox`) | **service discovery** — hands the client its downstream URLs | Gate 1→3 glue |
| **TorNet** (`ClientServerInterface`, `OnlineConfig`) | binary realtime session transport | Gate 3 (session) |
| **GVS\*** actions (in `Game.dll`) | server-authoritative game mutations | Gate 4 (player-state) |
| **GameConfig** (`Oizys`) | baked-in IDs: ClientID, GameSpace, IGPCode, GGI, JanusScopes, FedAPIVersion, DeltaKey, MobAppID… | feeds all gates |

Note: `Gameloft.VSSFramework.dll` is the **audio** system (VoxSound), a naming collision with the case file's use of "VSS" for the action framework. The actual game actions (`GVS*`) live in `Game.dll`.

## Two levers that make this tractable

1. **Plain-HTTP is allowed for Gameloft.** `Info.plist` → `NSAppTransportSecurity` grants `gameloft.com` (and all subdomains) `NSExceptionAllowsInsecureHTTPLoads: true`. The client will talk to Gameloft endpoints over cleartext HTTP. Gates 1, 3, and 4 can be redirected with a DNS override to a local HTTP server — **no TLS interception, no cert-pinning fight.** (Netflix auth domains are *not* in the exception list, so gate 2 stays HTTPS.)

2. **Hermes is service discovery.** The client doesn't hardcode most endpoints — it registers with Hermes and uses the URLs Hermes returns. Control the CRM/Hermes response and you steer every downstream call to your server without enumerating them by hand.

## The four gates

### Gate 1 — online-config (CRM / Oizys)
- **Mechanism:** native `FUNC_GetConfigFromServer` → CRM config; `CRMSystem_RegisterConfigRefreshedCallback` fires when it lands.
- **Redirect:** DNS-override the config host (cleartext OK) → local HTTP server returns a CRM JSON payload.
- **Difficulty:** Low. The blocker is knowing the exact JSON field set the client requires; reconstruct from the `GameConfig_Get*` surface + a captured/archived response.

### Gate 2 — Gaia login → Janus token (IdentityModel.OidcClient)
- **Mechanism:** `GaiaAuthorizeAsync` runs an OIDC flow (`/authorize`, `client_id`, `scope`) and gets a **Janus** token with scopes from `GetJanusScopes`. Console markers: `[WAIT] Gaia Init` → `[OK] Gaia Init`.
- **Redirect:** stand up a fake OIDC provider; if the client validates against a JWKS you serve, sign with your own key.
- **Open risk:** the Netflix layer (`netflix83`, NetflixGames.framework) may wrap or gate this. Confirm from the native binary whether a Gaia-only path is reachable.
- **Difficulty:** Medium (High if Netflix entitlement is load-bearing).

### Gate 3 — TorNet session handshake
- **Mechanism:** `ClientServerInterface` is constructed with client-supplied delegates: `GetServiceURLFunction`, **`EncryptTokenFunction`**, `ReceivedTransmissionsFunction`, `StateChangedFunction`, plus an `OperationMode`. Session params come from `OnlineConfig`: Host, Port, ClientId, Credential, UserId, AuthScopes, DataCenter, ServerType, ConnectionType, CustomLoginData.
- There is an observed relative path **`/token/encrypt`** — the Janus token is exchanged/encrypted into a TorNet credential. `EncryptToken` runs through a static callback (`EncryptToken_StaticCallback` / `CallEncryptTokenDelegate`).
- **Difficulty:** High — this is the binary-protocol gate. The wire framing and the encrypt step must be reversed from the native `lego` binary (the managed side is only the delegate shims).

### Gate 4 — player-state + GVS\* actions
- **Mechanism:** a paired request/response RPC set. **59 request actions, 50 response types** recovered from `Game.dll` metadata (full list below). Once the TorNet session is up, the server persists player state and answers each action with a valid delta.
- **Boot-critical / earliest needed:** `GVSCompleteAgeGate`, `GVSCompleteTutorialStep`, `GVSClaimDailyLoginReward`, `GVSGetStoreConfig`, `GVSRequestProfileEdit`.
- **First-playable loop:** `GVSStartBattle` → `GVSCompleteBattle` → `GVSLootBattle`; then gacha (`GVSClaimGacha` / `GVSBoughtGacha`) and progression (`GVSLevelUpMinifig`, `GVSUnlockMinifig`, `GVSUnlockCampaign`).
- **Difficulty:** Medium but broad — mechanical once the transport works; each action's schema reversed incrementally.

## Full GVS\* action surface (59)

```
Account/boot:  CompleteAgeGate, CompleteTutorialStep, RequestProfileEdit,
               ClaimDailyLoginReward, ClaimTimedGift
Battle:        StartBattle, CompleteBattle, LootBattle, RefreshCampaignNode,
               UnlockCampaign, ViewedCampaign, CompleteFlaggedEvent
PvP/Arena:     StartPvpBattle, CompletePvpBattle, ArenaRefresh, ArenaCooldownSkip
Tournament:    TournamentBattleStarted, TournamentBattleEnded, TournamentDefenseTeam
Brixpedition:  BrixpeditionBattleEnded, BrixpeditionHealMinifig, BrixpeditionMapReset,
               SendMinifigsForExcavation
Gacha:         ClaimGacha, ClaimMultipleGachas, BoughtGacha, TGachaCampaign
Minifig:       LevelUpMinifig, LevelDownMinifig, RankUpMinifig, RankDownMinifig,
               UpgradeMinifigAbility, DowngradeMinifigAbility, UnlockMinifig,
               ConvertMinifigShards, ConvertExtraCharacterTiles,
               AddFavouriteMinifig, RemoveFavouriteMinifig, UpdateSavedTeams
Gear:          EquipMinifigGear, EquipMinifigGearLevelTarget, CombineMinifigGear,
               EquipAndCombineMinifigGear, HandleCraftItem, UseBrickSeparator
LEGO sets:     UnlockLegoSet, RankUpLegoSet, UpgradeSetAbility
Store/econ:    VisitStore, RefreshStore, GetStoreConfig, PurchaseItem, BoughtItem,
               PurchaseExtraEnergy, ResetEventEnergy
Achievements:  ClaimAchievement, ClaimMultipleAchievements, GetGuildAchievementProgress
Town:          CompleteSetTownSquareVisualProfile
```

## Recommended first milestone: "boot to TorNet wait"

The first visible win is getting the client past config + login to where it blocks on the TorNet session. Steps:

1. DNS-redirect the Gameloft config host → local HTTP server (cleartext, per ATS).
2. Serve a minimal CRM/Hermes response; iterate against the client until `[OK] Gaia Init`-adjacent logging shows config accepted.
3. Fake the Gaia OIDC `/authorize` + token issue; get a Janus token the client accepts.
4. Watch it advance to the TorNet handshake and stall there — that stall is the milestone. It proves gates 1–2 are satisfied and isolates the remaining work to gate 3.

Then gate 3 (reverse TorNet framing + `/token/encrypt`) is the real research problem, and gate 4 is incremental grind after it.

## Open questions / what's still needed

Everything above came from the managed assemblies, which give the client's *view* of each gate. Still required, all sourced from the **native `lego` Mach-O** (64 MB, not in the code slice) and the `Data/` folder:

- **Bootstrap host(s)** — the initial config URL is compiled into the native binary.
- **Baked GameConfig values** — exact ClientID, GameSpace, IGPCode, GGI, JanusScopes, FedAPIVersion, DeltaKey (likely in native or a `Data/` config blob).
- **TorNet wire format** — packet framing, opcodes, the encrypt step.
- **Server-side response *values*** — the code defines shapes, not the exact JSON the real servers returned. Archived captures (HAR/pcap) from the community would shortcut this; otherwise reconstruct from client requirements.

---

## UPDATE 2 — findings from the native `lego` binary (arm64 Mach-O, 64 MB)

The native binary **retained its C++ symbols with source paths**
(`/Users/jog.builder/Projects/LegoNetflix/FuWorkspace/lego/Engine/Externals/…`).
This upgrades gate 3 from "opaque binary" to "reversible with a disassembler" — every
Online Framework class and method is named.

### Confirmed hosts
| Host | Role |
|---|---|
| `gameoptions.gameloft.com` (+ `gameoptions-staging.`) | **Gate 1 config** — GameOptions system |
| `201205igp.gameloft.com` | IGP (game-product) host; **`201205` is the game's IGP code** |
| `201205igt.gameloft.com` *(truncated in strings)* | IGT companion host |
| `sink.gameloft.com` | analytics sink (ignore for revival) |
| `eve.game…` | EveLink (`GameConfig_GetEveLink`) |

### Gate 1 — now concrete
- Endpoints: **`/config/`** and **`/configs/users/me`** on `gameoptions.gameloft.com`.
- The client **ships a fallback config**: `GameOptionsInitial.json` (bundled in `Data/`),
  caches server config to `GameOptions_saved.json`, and does etag validation via
  `GameOptionsEtag.t`. Native API: `GameOptionsSystem` + `oizys::GaiaSystem::GaiaInterface::GetConfig(oli::online::ConfigRequestArgs)`.
- **Implication:** `GameOptionsInitial.json` is very likely a near-complete template of the
  server response. Serving a lightly-edited copy from the fake host may satisfy gate 1 outright.

### Gate 2 — versions & flow pinned
- **Gaia V2, lib version `15.3.0`** (`GaiaV2_LibVersion_15.3.0`). This is a specific,
  cross-title Online Framework version — protocol should match other 15.3.x Gameloft games.
- Flow symbols: `JanusLogin` → `GetJanusToken` → `GetJanusFedID` (Federation), with
  `AuthorizeResponse` / `IAuthorizer::ErrorCode` as result types. `KairosSystem` is a
  sibling service (social/push) — not boot-critical.
- **Gate 2→3 handoff (from GaiaManager log strings):** a successful Gaia auth returns a
  **Fed(eration) Access token**, a **Data Center Id**, and an ETS token. The fed access
  token + data center id are exactly what the TorNet connect needs — the fake Gaia response
  must supply both. Request params carry `client_id`, `gamespace`, `promoted_gamespace`,
  and `janus_scopes` (literal values still pending the bundled `Data/` JSON).

### Eve — datacenter directory (gate-3 dependency)
`https://eve.game…` hosts `ListDataCenters`, which backs TorNet's `XListDatacentersCallback`
/ `AutoSelectDatacenter`. Boot path for the session: **Eve lists datacenters → one is
selected (or handed over by Gaia's Data Center Id) → TorNet connects there.** The fake stack
must answer Eve's list with at least one datacenter pointing back at the local TorNet server.

### Noise to ignore (not the game backend)
The templated `%@` URLs (`adrevenue`, `skadsdk`, `onelink`, `register`, `inapps`, `viap`,
`ars`, `monitorsdk`, `conversions`, …) are the **AppsFlyer** analytics/ad-attribution SDK.
`sink.gameloft.com` + the ETS/GGI tracking strings are analytics. None are boot-critical;
they can fail harmlessly or be null-routed.

### Gate 3 — wire format is enumerable
`TorNet_Client_DotNet_Binding` adapter surface:
- **Session:** `ClientServerInterface`: `StartUp`, `ShutDown`, `PostTransmission`,
  `Update_Internal`, `GetState`, `Purge`, `SetCSharpHandle`, background/foreground notifies,
  and transmission batching (`Begin/End/Purge/IsBatchingTransmissions`).
- **Wire unit:** `DataTransmission` = length-prefixed binary buffer with typed primitives
  (`ReadString`, `ReadUInt16`, `GetRead/WriteCapacity`, `GetRead/WriteSize`, …).
  Reconstruct the format by enumerating every `Read*`/`Write*` primitive.
- **Config:** `CreateBasicOnlineConfig`, `ServerType`, `Datacenter`.
- **Datacenter discovery:** `XListDatacentersCallback` + `AutoSelectDatacenter` — TorNet's
  host is discovered/selected at runtime, so it's steerable via the config/Hermes response
  rather than hardcoded.

### Revised next step — highest-value artifact
Pull the **bundled config JSON and any online-config data** from the IPA `Data/` folder;
this likely contains the literal `ClientID`, `GameSpace`, Janus scopes, host templates, and
a response template for gate 1:
```
cd ~/Downloads
unzip llhu.ipa 'Payload/lego.app/Data/GameOptions*'            -d cfg_slice
unzip llhu.ipa 'Payload/lego.app/Data/*.json'                  -d cfg_slice 2>/dev/null
# list what config-ish data ships:
unzip -l llhu.ipa | grep -iE 'GameOptions|config|online|gaia|janus|hermes|\.json' | grep -iv '3d/'
```
Then zip `cfg_slice` (should be small) and share it. With the config template + the hosts
above, the gate-1 fake server can be scaffolded for real.

### After that: standing up the boot chain
1. Local DNS override: point `gameoptions.gameloft.com`, `201205igp.gameloft.com`,
   `201205igt.gameloft.com` at a local box (cleartext HTTP OK per ATS).
2. Serve gate-1 config (from `GameOptionsInitial.json` template) → watch for config accepted.
3. Fake Gaia V2 15.3.0 `/authorize` + Janus token issue → watch for `[OK] Gaia Init`.
4. Client advances to TorNet datacenter-list + handshake and stalls → **milestone reached.**
5. Reverse `DataTransmission` framing from the native symbols; implement the session
   handshake + `/token/encrypt`; then grind gate-4 GVS* actions.

### Recommended tooling for the native reversing (gate 3)
Because symbols survived, load `lego` into Ghidra (free) or IDA and jump straight to the
`ClientServerInterface` / `DataTransmission` / `oli::online` symbols — no manual function ID
needed. This is the one part that needs a real disassembler rather than string extraction.

---

## UPDATE 3 — config files pulled from the IPA (`Data/Config/`)

Extracted directly from the release IPA via HTTP range requests (no full download). The
config folder and what each file turned out to be:

| File | State | Content |
|---|---|---|
| **`OizysConfig.json`** | **encrypted** | The Online Framework config — ClientID, GameSpace, scopes, Gaia hosts. **This is the file with the literal identifiers.** Encrypted with a Gameloft scheme; key is in the native binary. |
| `snsconfig.json` | plaintext | Login-provider matrix (see below) |
| `GameOptionsInitial.json` | plaintext | **Graphics/memory tier presets only** (GPU/MEM profiles) — *not* the online config |
| `EngineConfig.json` | plaintext | Trivial engine flags (orientation, console output) |
| `glue.json` | plaintext | UI/font glue |
| `maintenance_message.json` | plaintext | Localized maintenance strings |
| `DefaultSettings.tx`, `Settings.tx` | binary (`GCBF`) | Gameloft binary settings format |

### Login providers (from `snsconfig.json`, iPhone)
`GLLive`, **`Netflix`**, `AppleSignIn`, `Facebook`, `GameCenter`, `EmailPhonebook`,
`NumberPhonebook`. Facebook app ID `623207384557200` is in the clear.

**Key implication for gate 2:** Netflix is only one of several Gaia login backends.
**`GLLive` (Gameloft Live) is a separate provider present in the client** — so a
non-Netflix auth path likely exists, and the Netflix entitlement layer may be avoidable
entirely. Prefer targeting the GLLive/Gaia path for the revival.

### The two remaining gating unknowns (both need the native binary in a disassembler)
1. **Decrypt `OizysConfig.json`** → literal ClientID, GameSpace, Janus scopes, Gaia host
   templates. The decrypt key/routine is in `lego` (look near `GameConfig_GetDeltaKey`,
   `Libpsy`, and the Oizys config loader).
2. **TorNet `DataTransmission` wire format** → gate 3 handshake + `/token/encrypt`.

Everything else (hosts, endpoints, login flow, datacenter discovery, the full GVS* action
surface) is now known. Recon is effectively complete; the project moves to building.

---

## UPDATE 4 — protocol reference (from full native symbol/string dump)

### OizysConfig.json field schema (what the encrypted config decrypts to)
`igp_code` (=201205), `bundleId`, `dowloadSource` [sic], `useFedOnCloud`, `eveLink`,
`fedAPIVersion`, `janus_scopes`, `dlcProfile`, `autoDatacenterSelect`, `lockCountry`,
`crmRefreshPeriod`, `gaiaRetryTimer`, `deltaDNAKey`, `appsflyerKey`, `appsflyerAppId`,
`ads*AppId`. Config encryption uses Apple **CommonCrypto (`CCCrypt`)** with a **derived**
key (`AESKeyForPassword:salt:` present) — key is computed at runtime, so it must be
recovered by tracing the loader in Ghidra, not by string search. (The two 16-char strings
`Gxxkv8tKy0Wwc2bI` / `BNzWZbSGMNF3wdOG` are NOT this key — tested, they're TorNet-side.)

### TorNet login flow (gate 3), in order
1. `getServiceURL` resolves the `auth` service URL (fails loudly if unavailable).
2. GET `/authorize` with `access_token_only` → obtains access token.
3. Retrieve `matchmaker` / controller URL via `getServiceURL`.
4. **Controller login**: request nonce (**"Anubis" nonce** challenge), then login with
   encrypted token (`EncryptToken` callback — fails if `encryptTokenFunction` is null),
   `username`, `client_dc`, `client_ver`.
5. Controller responds with `controller_host`, `controller_tcp_port`, `room_id`,
   `server_type` → client opens the TCP game session ("connect game").

### TorNet service types (resolved via Hermes / getServiceURL)
`auth`, `lobby`, `storage`, `config`, `social`, `leaderboard`, `leaderboard_ro`.
Matchmaking variants: `/singletons` (singleton_id), automatch (filter/score_min/score_max/
isolated/midgame_join), quicklaunch.

### Candidate TorNet keys (for the encrypt/handshake step)
`Gxxkv8tKy0Wwc2bI`, `BNzWZbSGMNF3wdOG` — test against the `EncryptToken` / DataTransmission
crypto once that routine is located in Ghidra.

### Pinned component versions (cross-title protocol matching)
GaiaV2 `15.3.0` · GLOTv3 `17.0.4` · OfflineItemsV2 `6.1.0` · PopupsLib `13.7.1` ·
IGBLib `16.0.0`. Oizys systems present: Gaia, Hermes, Kairos, Popups, Social, Tracking,
GameOptions, OfflineItems, OnlineFramework, OnlineLibsUtils, CRM, IGB, IGP.

### Identity bootstrap
`identifiers::Identifiers` calls `/assign_global_id` (source `Identifiers_6.0.0`), caches a
GDID in `GDID.bin`. GLLive username format is `gllive:myuser`. Anonymous id `_GAIA_ANON_GLUID`.
