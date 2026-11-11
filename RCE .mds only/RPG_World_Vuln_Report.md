# Security Report — RPG World (Steam appid 492150)

**Product:** RPG World - Action RPG Maker (buildid 5461577, Unity 5.4.5p5 / Mono, Photon PUN1)
**Date:** 2026-11-11 · **Reporter:** authorized security research · **CVSS (Chain A): 8.8 (AV:N/AC:L/PR:N/UI:R/S:C/C:H/I:H/A:H)**

## Summary
Multiple vulnerabilities allow a remote attacker to achieve arbitrary code execution on any player's machine via normal gameplay (joining a multiplayer room) or a single Workshop subscription. The 2016-era Photon cloud service for this title is still live, making the multiplayer vector internet-reachable today.

## Vulnerabilities

**VULN-1 — Path traversal in archive extraction (CWE-22, Critical)**
The native extractor `libzipw.dll` (minizip 0.15, 1998) used for all world archives concatenates each archive entry name onto the output path with no `..` sanitization and writes via `_wfopen(path, "wb")` (`zipEX` @ `0x18000704a`). Reachable from two remote paths: (a) Photon event-25/27 world stream (`MyWorld.cs:8330/8390`), sender-unvalidated (`MyWorld.cs:7114`, registered `:1882`); (b) Steam Workshop world subscription (`FileFormats.cs:2718`). Impact: arbitrary file write outside the intended directory → overwrite `Managed/Assembly-CSharp.dll` → attacker code executes on next game launch. **Proven:** traversal, directory-creation escape, and byte-for-byte file replacement all demonstrated against the shipped DLL; full chain (attacker room on live public cloud → hostile world stream → escaped write) confirmed end-to-end against a byte-faithful simulator of the game's handler.

**VULN-2 — Heap buffer overflow in `zipEX` (CWE-122, High)**
Extraction path construction concatenates output path (≤260 wchars) and entry name (≤259) into a `malloc(0x208)` buffer with no combined-length check (`0x180008d30-0x180008df4`). Entry names ≥256 chars (≥~186 with the game's real output path) corrupt the heap. **Proven:** 4/4 reproducible `STATUS_HEAP_CORRUPTION` on the shipped DLL; same delivery vector as VULN-1.

**VULN-3 — Insecure deserialization of world/UGC data (CWE-502, Medium)**
`BinaryFormatter.Deserialize` of remote/workshop-supplied `start.nam`/`game.nam`/`.world` (`SteamWorkshop.cs:376-424`, `FileFormats.cs:2824, 3040-3059`), firing on mere subscription enumeration. Classic .NET gadgets are absent from the shipped profile; Mono-specific gadget not yet identified. Exec potential conditional.

**VULN-4 — Unvalidated native image decode (CWE-119, Low/Medium)**
World-controlled filenames fed to `Texture2D.LoadImage` over local files (`MyWorld.cs:2353, 24967, 25122`), reaching bundled 2016-era image decoders with attacker bytes.

## Attack scenario (VULN-1, multiplayer)
1. Attacker joins or hosts any Photon room (public cloud, alive, verified) or publishes a Workshop world.
2. Victim enters multiplayer or subscribes. The hostile world archive streams to the victim and is auto-extracted with `../` entries.
3. Attacker files land in `Managed/`, replacing game code; execution on next launch. No victim action beyond normal play.

## Recommendations
1. Sanitize archive entry names (reject `..`/absolute/separator-containing names) before extraction in `libzipw.dll`; or replace minizip 1998 with a maintained library.
2. Validate combined path length against the destination buffer in `zipEX`.
3. Replace `BinaryFormatter` with a safe serializer for all UGC.
4. Restrict world asset loading to an allow-listed directory.
5. If the Photon AppId is decommissioned, disable multiplayer or migrate it; note the current cloud accepts these clients in 2026.

## Disclosure status
Reporter-hold; no vendor contact yet. Full technical evidence ( PoC scripts, archives, decompilation references) available on request.
