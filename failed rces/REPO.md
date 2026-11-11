# R.E.P.O. — Remote Code Execution / Arbitrary File-Write Assessment

Game: R.E.P.O. (Steam, appid 3241660) — semiwork
Engine: Unity 6000.3.17f1, **Mono** scripting backend
Networking: Photon PUN 2 (Photon Realtime 4.1.8.21) over Steam-auth identities, Photon Voice
Build audited: current Steam build (Larva, Halloween/Arena race, recording) — `C:\Program Files (x86)\Steam\steamapps\common\REPO`
Scope: vanilla client, no BepInEx/Doorstop installed. Findings are what a **remote peer** (host or joining client) can cause on a stock install without local mods.
Class: Code-execution surface review
Verdict: **No vanilla remote code execution and no remote arbitrary file write** (high confidence, see section 9 for residual risk).
Severity: LOW (as an RCE claim) / MEDIUM as a DoS-and-trust paper

---

## 1. Executive summary

I enumerated every code-load, process-spawn, deserialization, archive, native-P/Invoke and network-to-disk sink in the shipped managed assembly and followed each one to whatever a remote peer can influence. There is no path where peer-supplied bytes become executed code on a stock client.

The only genuinely attacker-controlled native parser is Photon Voice's Opus decode path (`opus_egpv.dll`, libopus **1.1.1-beta-39**, i.e. pre-1.1.3). It is reachable end-to-end by a malicious voice peer, but the known bug in this version that is actually reachable (CVE-2017-0381, NLSF stabiliser OOB read) leaks/crashes rather than writes. There is also an unvalidated `FrameDurationUs` field in Photon's own voice-info handler that lets a peer force huge/negative buffer allocations and kill the voice renderer. Both are DoS, not exec.

The two chains that *look* like exec on paper and are not:

* `marketingMessage.link` → `Application.OpenURL` — the link really can be rewritten at runtime (Unity RemoteConfig `news_msg` + `JsonConvert.PopulateObject`), and `file:///C:/x.exe` would ShellExecute. But the URL comes from the developer's RemoteConfig, not from a peer, and it still requires a user click.
* Addressables — bundles do exist on disk, but the catalog is local-only (`StreamingAssets/aa`, zero `http` internal IDs) and the only `Addressables.LoadAssetAsync` call in game code loads cosmetic icon sprites. No remote catalog, no remote bundle fetch.

Everything else a peer touches (RPCs, room properties, voice packets) lands in managed data structures.

---

## 2. Trust boundary

Photon identity is Steam-verified, so `Owner.UserId` / `Owner.NickName` cannot be spoofed:

* `PlatformManager.cs:60-66` calls the Steam platform auth.
* `SteamPlatform.cs:72-86` → `GetAuthSessionTicket`.
* Photon picks up the ticket at `NetworkConnect.cs` connect flow.

Consequence: an actor can only issue RPCs as its own `PhotonView` owner, or from a view it created. Host (master client) is overwhelmingly authoritative — for most state objects only the master's RPCs are accepted. The `MasterOnlyRPC` / `OwnerOnlyRPC` / `MasterAndOwnerOnlyRPC` helpers in `SemiFunc.cs:3988 / :4005 / :4022` are applied correctly. A malicious **host** can therefore do almost anything to world state; a **joining client** is limited to its own-view RPCs plus the small ungated set (section 6).

Neither role reaches a code-load sink.

---

## 3. Why there is no RCE — proven sink list

These are the checks behind the verdict. Each is a negative result on the decompiled `Assembly-CSharp` (1,127 files, `C:\Users\pallu\AppData\Local\Temp\opencode\repo-src`).

| Sink | Present? | Evidence |
|---|---|---|
| `Process.Start` / `ProcessStartInfo` | **No** | zero hits in game code and in managed deps used by the game |
| `Assembly.Load` / `LoadFrom` / `LoadFile` / `Load(byte[])` | **No** | zero hits |
| `AppDomain.Load`, `AssemblyResolve` install | No | — |
| `AssetBundle.LoadFromFile` / `LoadFromStream` / `LoadFromMemory` | **No** | game code never calls it |
| Remote Addressables catalog / remote bundles | **No** | see 3.1 |
| `BinaryFormatter` / `NetDataContractSerializer` / `SoapFormatter` | **No** | saves go through ES3 + `JsonConvert`, see 3.3 |
| `Type.GetType` / `Activator.CreateInstance` / `TypeNameHandling.All` | **No** | — |
| `ZipFile` / `ZipArchive` / `Compression` / tar | **No** | no archive extraction anywhere |
| `unsafe` / `Marshal` write primitives | No | only `Marshal.SizeOf` |
| `DllImport` | only 3 groups | `user32.dll`, `steam_api64`, Photon Voice natives — section 4 |
| `Resources.Load` with network-supplied path | 1 path, master-only, still `Resources/` | `SemiAffectExploding.cs:301` → `SemiFunc.PrefabGet`, `MasterOnlyRPC` |
| `Application.OpenURL` | 4 fixed/one config path | section 5 |
| `Application.OpenURL` with `file:///` | only via RemoteConfig link | section 5 |

### 3.1 Addressables is local-only

`REPO_Data\StreamingAssets\aa\` contains `settings.json`, `catalog.json`, `catalog.bin`, `link.xml` and 30 `*.bundle` files (cosmetics, localization, `*_monoscripts.bundle`, one small default group). It is a real, populated Addressables profile — but:

* `settings.json` `m_CatalogLocations[0].m_InternalId` is `{UnityEngine.AddressableAssets.Addressables.RuntimePath}/catalog.bin`, i.e. resolved under `StreamingAssets/aa` (local).
* Of 1363 `m_InternalIds` in `catalog.json`, every asset id resolves either to the local `RuntimePath` or to an `Assets/...` editor path. Zero contain `http`.
* The entire game codebase contains exactly **one** Addressables call: `Addressables.LoadAssetAsync<Sprite>(cosmeticAsset.iconRef)` at `MenuElementCosmeticButton.cs:357`, guarded by `RuntimeKeyIsValid()`; the field is `AssetReferenceSprite` (`CosmeticAsset.cs:37`).
* No `LoadContentCatalogAsync`, no `CheckForCatalogUpdates`, no `UpdateCatalogs`, no `Addressables.InstantiateAsync` anywhere.

So even though 1363 things are marked addressable (levels, items, valuables), at runtime the game only pulls sprites, and only from the local catalog. Swapping a bundle on disk is a local install action, not a remote one.

### 3.2 Prefab spawn path never takes a network string

`PhotonNetwork.Instantiate*` always receives a constant from `PrefabRef.ResourcePath`. The single RPC that *does* accept a resource path — `SemiAffectExplodingTriggerRPC` / snapshot `_resourcePath` → `PrefabGet` → `Resources.Load` (`SemiAffectExploding.cs:301`) — is `MasterOnlyRPC`-gated and still only reads `Resources/`. A hostile host can point it at any built-in prefab, never at a file or an assembly.

### 3.3 Save data is typed and safe

ES3 writes three string fields on a `[JsonObject(MemberSerialization.OptIn)]` type and reads them back with `JsonConvert`. No polymorphic type tags, no `TypeNameHandling`, no formatter. Save **filenames** come from the local UI text field, so path traversal there is local-only (you can create `..\foo` names yourself, but a peer cannot).

### 3.4 Resources / StreamingAssets / TTS

`StreamingAssets\` beyond `aa` is `BakedIcons\` (local PNGs), `Localizations\` (TextAsset tables) and two licence files — nothing executable. Icon baking reads `streamingAssetsPath` through `UnityWebRequestTexture`; the path is fixed. TTS is managed `Strobotnik.Klattersynth` (`TTSVoice.cs`) — no native voice engine, no SSML, no SSML `<audio>` fetch.

### 3.5 Mods are out of scope by definition

`.dll`/`.bundle` next to `BepInEx` are loaded by BepInEx, not by the game. The game only *detects* mods (`ModdedCheck.cs` reflects already-loaded assemblies for `MelonInfoAttribute`) and knows the `AssetBundles/` prefix in `LevelGenerator.cs:1437`. A client is never shipped a host's bundle, so a modded host's custom level just fails to generate for vanilla joiners (`MultiplayerPool.cs:187` falls back to `Resources.Load<GameObject>(prefabId)`).
---

## 4. The one real native attack surface: Photon Voice

This is the only place attacker bytes enter native code.

* `Photon.Voice.Windows` binds `opus_egpv` — `lib_name = "opus_egpv"`, `opus_decode` / `opus_decode_float`, `CallingConvention.Cdecl` (`pv-src\PhotonVoice.API.decompiled.cs:1361-1379`).
* On disk: `REPO_Data\Plugins\x86_64\opus_egpv.dll` (440,832 B), banner `libopus v1.1.1-beta-39-g976c8ad` (mid-2016, **before** the 1.1.3 fix set).
* The packet path is fully peer-controlled: `PhotonTransportProtocol.createVoiceInfoFromEventPayload` reads `SamplingRate=(int)h[2]`, `Channels=(int)h[3]`, `FrameDurationUs=(int)h[4]` (`:9660-9700`) → `RemoteVoiceOptions.setOutput` (`:5692`) → `new OpusCodec.Decoder<T>` → `OpusDecoder` ctor (`:959-978`).

Reachable defects:

### 4.1 Frame-size mismatch: tested live, guard holds

The peer's `FrameDurationUs` sizes the managed output array (`FrameDurationSamples => (int)((long)SamplingRate * (long)FrameDurationUs / 1000000)` at `:7890`, then `new T[frameSamples * channels]` in `OpusDecoder`), while the packet's own encoded duration is independently attacker-chosen. The obvious exploit would be a packet longer than the declared frame, written by native code into the pinned array.

I compiled a harness against the shipped `opus_egpv.dll` and ran both directions of the mismatch.

Long packet (60 ms, 2880-sample) decoded with a too-small `frame_size`:

```
encoded bytes=392
frame_size=48   -> ret=-2
frame_size=960  -> ret=-2
frame_size=1920 -> ret=-2
frame_size=2880 -> ret=2880
```

Normal 20 ms (960-sample) packet decoded with the `frame_size` a hostile `FrameDurationUs` would produce, replaying the exact managed post-processing:

```
20ms packet bytes=127
fs=0   ret=-1 -> Array.Clear(buf,-1,1)   -> IndexOutOfRangeException
fs=48  ret=-2 -> Array.Clear(buf,-2,48)  -> IndexOutOfRangeException
fs=120 ret=-2 -> Array.Clear(buf,-2,120) -> IndexOutOfRangeException
fs=960 ret=960 -> no clear
```

So libopus 1.1.1-beta **does** return `OPUS_BUFFER_TOO_SMALL (-2)` and does not overwrite. The bug is in the wrapper, not libopus:

* `decodePacket` (`:982-998`) treats any non-zero return as success: `num2 = num * channels; if (num2 < buffer.Length) Array.Clear(buffer, num2, buffer.Length - num2);`
* `num = -2` (or `-1` when `frameSamples = 0`) -> `num2 = num * channels` -> `Array.Clear(buffer, negative, ...)` -> `IndexOutOfRangeException` (confirmed with a two-line .NET test; it is `IndexOutOfRangeException`, not `ArgumentOutOfRangeException`).
* That throws on the Photon decode thread (`decodeThread` → `decodeQueue` → `Decoder.Input`), where it is caught once and the `RemoteVoice` is disposed. Net effect: the attacker's voice is **permanently dead on every listener** (logged `Exception in decode thread`), not a game crash.

Reachable values:
1. `FrameDurationUs = 2500` (or `0`) with normal 20 ms packets -> every decode returns -1/-2 -> voice killed. The ctor accepts these because it only validates rate/channels.
1. `FrameDurationUs = 2500` (or `0`) with normal 20 ms packets → every decode returns -2 → voice killed. The ctor accepts these because it only validates rate/channels.
2. `FrameDurationUs = int.MaxValue` → ~103 M samples → ~206 MB (mono) / ~412 MB (stereo) array; ctor allocation is inside `Decoder.Open`'s try/catch, so it either succeeds very slowly or the voice dies with `OutOfMemoryException`.
3. Negative `FrameDurationUs` → negative array length → `OverflowException`, again caught in `Open`; voice dead.
4. Unboxing / shape: `onVoiceInfo` casts `payload` to `object[]`, each element to `Dictionary<byte,object>`, and `createVoiceInfoFromEventPayload` casts `(int)h[2..4]`, `(byte)h[1]`, `(byte)h[11]` directly, with mandatory-key lookups at `h[2]`, `h[3]`, `h[4]`, `h[5]`, `h[12]`. `onVoiceEvent` has no try/catch, so a voice event with the wrong container, field type or a missing mandatory key throws `InvalidCastException` / `KeyNotFoundException` inside the Photon event callback.

### 4.2 CVE-2017-0381

OOB read in `silk/NLSF_stabilize.c`, fixed in 1.1.3, present in this build. Triggered by a crafted Opus payload on the SILK path. Effect: info leak / crash, **no write primitive**.

Why none of this is RCE: no reachable libopus bug lands a heap write, the buffer-length guard holds (tested), and the caller-side overflows published against libopus (PJSIP, FreeRDP) depend on their resampling/layout code, which Photon does not use.

PoC: modded Photon Voice client, join room, emit voice-info with `h[4] = 2500`, then transmit normal audio. Listeners lose that voice and log `Exception in decode thread`.

---

## 5. The nearest thing to exec: RemoteConfig → OpenURL

`MenuPageMain.cs:478` does `Application.OpenURL(MenuManager.instance.marketingMessage.link)` when the marketing banner is clicked. `marketingMessage` is a ScriptableObject, but RemoteConfig overwrites it live:

* `RemoteConfigManager.cs:140-143` — `GetJson("news_msg")` then `JsonConvert.PopulateObject(json, marketingMessage)`.
* `MenuPageMain.cs:484-487` — title/message text refreshed from the same object.

`Application.OpenURL` on Windows hands the URL to the shell, i.e. `ShellExecute`. So a `news_msg` of

```json
{ "title": "Update", "message": "Click", "link": "file:///C:/Users/pallu/Downloads/payload.exe" }
```

executes that local file on click; an `https://` link opens the default browser and can download-and-run with the usual prompt. This is developer/RemoteConfig-controlled (or whoever can alter that RemoteConfig environment), **not** reachable by a peer in a lobby, and it needs one click.

`SemiFunc.OpenFile` (`SemiFunc.cs:5134`) is the other `file:///` producer (`Path.GetFullPath` + requires the file to exist). All four of its callers use fixed paths: `/logs` (`Player.log`), `/saves` (save folder), `MenuElementSaveFile.cs:57` (save folder), `SemiIconMaker.cs:512` (icons folder) — so it is local-only.

---

## 6. Secondary peer-reachable findings (not RCE, context for the trust model)

Collected while tracing; useful if the paper needs to say "the client is authoritative-but-wrong, the host is not code".

* `PlayerTumble.TumbleForceRPC` (`:705`) and `TumbleTorqueRPC` (`:724`) accept raw `Vector3` with no finite/clamp check → `tumbleForce += _force` → `rb.AddForce(..., Impulse)` (`:436`) = arbitrary launch/teleport. A NaN makes `magnitude > 0f` permanently false, silently killing all impulses until the tumble resets.
* `PlayerHealth.HurtOtherRPC` / `HealOtherRPC` are `MasterAndOwnerOnlyRPC` and take arbitrary `damage`, bypassing `Hurt`'s `IsMine` gate → an owner can self-kill or self-heal at will. `UpdateHealthRPC` (`:516`) writes `health` / `maxHealth` and `StatsManager.SetPlayerHealth` unclamped → fake HP everywhere.
* `ValuableObject.Discover` fires ungated `DiscoverRPC` (`:334` broadcast, `:342` body, `RpcTarget.All`) from a local raycast → any client reveals every valuable on everyone's map.
* `PlayerExpressionSetRPC` / `StopRPC` update an unchecked index → uncaught per-frame exception on all clients.
* `ItemEquippable.RPC_StartUnequip` is ungated → strip any player's held item at any range.
* `MultiplayerPool.InstantiateFailed` unguarded pre-generation → kicks everyone to the menu.
* `MenuPageServerList.OnRoomListUpdate:130` does `(string)GetValueOrDefault("server_name","")` — a non-string room prop aborts server-list refresh.
* Room properties are client-writable → `RoomName`, `Region`, `BuildName`, `HasPassword` are strings and can only cause join/display desync; the Halloween revision can be overridden.
* `TruckScreenText.MessageSendCustomRPC` (`:1184`) drops typing messages (`isTyping`) — cosmetic.

---

## 7. If you want true RCE on this target

Three doors, all requiring something beyond vanilla:

1. **Local install write** — drop a DLL into `REPO_Data\Managed` (or install BepInEx and add a plugin). Vanilla loads it at startup. This is the realistic answer for "make REPO run my code".
2. **Developer supply chain** — control the `news_msg.link` RemoteConfig value and get a click. Same exposure as any browser link.
3. **Native memory corruption** — find a write primitive in libopus 1.1.1-beta (or the Photon marshaller) that survives the voice decode path; nothing public does today.

---

## 8. Reference index

Managed (temp decompile) — `C:\Users\pallu\AppData\Local\Temp\opencode\repo-src\`:
`SemiFunc.cs:3988/4005/4022/5134`, `PlayerTumble.cs:436/705/724`, `PlayerHealth.cs:493-568`, `ValuableObject.cs:319-346`, `MultiplayerPool.cs:187`, `MenuPageServerList.cs:130`, `TruckScreenText.cs:1184`, `MenuPageMain.cs:466-487`, `RemoteConfigManager.cs:136-164`, `MarketingMessage.cs`, `MenuElementCosmeticButton.cs:357`, `CosmeticAsset.cs:37`, `SemiAffectExploding.cs:301`, `SemiIconMaker.cs:512`, `MenuElementSaveFile.cs:57`, `ModdedCheck.cs`, `LevelGenerator.cs:1437`, `NetworkConnect.cs`, `SteamLobbyManager.cs`.

Photon Voice - `C:\Users\pallu\AppData\Local\Temp\opencode\pv-src\PhotonVoice.API.decompiled.cs:959-978/982-998/1361-1379/5692/6236-6265/7890/9660-9700`.
Natives — `REPO_Data\Plugins\x86_64\opus_egpv.dll`, `webrtc-audio.dll`, `AudioIn.dll` (mic capture), `steam_api64.dll`.
Addressables — `REPO_Data\StreamingAssets\aa\settings.json`, `catalog.json`.
---

## 9. Residual surface and confidence

Deliberately not claiming an absolute 100%. This is what was checked after the first pass, and what would remain.

Checked and cleared in this pass:

* **All 186 managed DLLs** were string-scanned for `Assembly.Load*`, `Process.Start*`, `Activator.CreateInstance`, `BinaryFormatter`, `TypeNameHandling`, `CompileAssemblyFromSource`, `DownloadFile`. Only framework-owned copies appear; the game's own code has zero of each (recursive scan of all 1,127 decompiled files).
* **Hot Reload** (`SingularityGroup.HotReload.Runtime.Public.dll`, `Domain_Reload.dll`, `UnityEngine.HotReloadModule.dll`): only public attributes and a marker class are shipped, no `SingularityGroup.HotReload.Runtime` assembly, and nothing in game code references it. It is not listener-active in this build.
* **`websocket-sharp.dll`** and **`PhotonChat.dll`** are both dead — only hanging `using` directives, no `WebSocket`/Photon.Chat instantiation.
* **ES3 / Easy Save 3** (compiled into `Assembly-CSharp-firstpass.dll`): typed loads only; `ES3Cloud` is never used, so no HTTP save fetch. No `ZipFile`, `ZipArchive`, `GZipStream`, `DeflateStream`, `XmlSerializer` or `DataSet` anywhere.
* **Newtonsoft `JsonConvert.PopulateObject`** call sites: default serializer settings (`TypeNameHandling.None`), so a `$type` in the RemoteConfig JSON is ignored. `MarketingMessage.ImportClipboardButton` is editor-only; only RemoteConfig feeds it at runtime.
* **`libonigwrap.dll`**: no managed assembly references it at all (checked all 186). Orphan native.
* **`webrtc-audio.dll`**: attached via `GetComponent<WebRtcAudioDsp>()` and only wired when `localVoiceAudioShort != null`, i.e. local capture/monitor path (`PlayerVoiceChat.cs:526`). It never sees the remote decoder's buffer.
* **Chat dev commands**: `PlayerAvatar.ChatMessageSend` checks `_message.StartsWith('/')` and calls `DebugCommandHandler.Execute` on the **sending** machine only, before the RPC (`PlayerAvatar.cs:1908`). Remote chat cannot invoke commands. The commands themselves (`/logs`, `/saves`, `/configreload`, `/item`, `/enemy`, `/valuable`, `/cosmetic import`) use `Resources`/in-memory dictionaries, not disk loads.
* **`AssetBundle.LoadFromFile`/`LoadFromStream`**: zero. The only bundle code is `GetAllLoadedAssetBundles()` + `LoadAssetAsync` in `LevelGenerator.cs:1437-1494`, so a bundle must already be loaded (i.e. a BepInEx mod on that client).
* **Steam Cloud / `SteamRemoteStorage`**: not used.
* **`Unity.Services.Authentication.PlayerAccounts`** runs a loopback `HttpListener` for the OAuth hand-off — local token injection only, not code.

Remaining risk, ranked:

1. **libopus 1.1.1-beta internals beyond CVE-2017-0381.** Public CVEs for that era are reads/DoS. A private write primitive in SILK/CELT decode is possible but would need fuzzing or RE of the exact DLL.
2. **Photon `VoiceClient.processFrame` fragment reassembly** (`pv-src:6096-6234`). `num2 = (length-1) * fragmentCount` and the pool sizing are managed, so overflow there is `IndexOutOfRange`/OOM, not memory corruption — but worth fuzzing for a harder crash than the voice-kill.
3. **`TransportProtocol` payload unboxing.** Every `(byte)array[n]` / `(int)h[n]` is an unchecked unbox; wrong Photon types give `InvalidCastException` in the event callback. DoS only.
4. **Native `discord_partner_sdk.dll` / `steam_api64.dll`.** Both local IPC peers (named pipes / Steam client). Compromising the local Steam or Discord client is out of band.

Confidence: **high that a vanilla remote peer cannot execute code on a stock install**; the managed surface is small and fully enumerated here. The gap is a private, non-CVE memory-corruption bug in the bundled native audio code or the Photon marshaller, which only dynamic testing would settle.
