


t.me/monsterhigh | 4vz


# Hollow Hide: Untrusted Workshop Prefab Can Invoke Static Unity Methods, Leading to Potential RCE
## Summary
Hollow Hide automatically downloads the host-selected Steam Workshop map and instantiates its `Map` prefab without restricting the components or serialized callbacks contained in that prefab.
A malicious map can combine an auto-playing Timeline with a `SignalReceiver` and a persistent `UnityEvent` that calls the static method `UnityEngine.Application.OpenURL(string)`. On Windows, a `file://` URI can launch a local executable. The included harmless PoC uses Calculator as its payload.
This creates a potential remote-code-execution chain when a client downloads and loads an attacker's workshop map. The final bundle has not yet been executed in a live multiplayer client, so the issue should currently be treated as a strongly supported candidate rather than a completed end-to-end reproduction.
## Attack Chain
1. The attacker creates a Unity `6000.3.7f1` asset bundle containing a `Map` prefab.
2. The prefab contains the required `MapHiderSpawn` and `MapSeekerRoomAnchor` markers so that it passes the game's map validation.
3. The attacker adds a `PlayableDirector` with `playOnAwake = true`.
4. Its Timeline contains a `SignalEmitter` at time zero. The signal is marked retroactive and configured to emit once.
5. A `SignalReceiver` maps that signal to a serialized `UnityEvent`.
6. The event registers a static persistent listener for `Application.OpenURL(string)` with an attacker-controlled string argument. The harmless PoC uses `file:///C:/Windows/System32/calc.exe`.
7. The attacker publishes the map to Steam Workshop, hosts a lobby, and selects the map.
8. `NetworkRoomState.ServerSetWorkshopMap` replicates the map ID, name, and hash to connected clients.
9. When map preparation begins, `NetworkRoomState.OnPreparingMapChanged` calls `FetchMissingMap`, which calls `SteamWorkshopService.SubscribeAndDownloadAsync`. The client therefore retrieves the selected workshop item without a separate security prompt.
10. `WorkshopMapLoader.LoadRoutine` locates a `*.bundle`, checks its MD5 against the host's value, loads it with `AssetBundle.LoadFromFileAsync`, and instantiates the `Map` prefab.
11. The MD5 check establishes only that the client and host have the same file. It does not establish that the file is safe.
12. Instantiation activates the Timeline. `SignalReceiver.OnNotify` invokes the serialized `UnityEvent`, which resolves and calls `Application.OpenURL` with the supplied URI.
13. On Windows, the `file://` URI launches the selected executable through shell handling. In the PoC, the visible result is Calculator opening in the logged-in user's context.
## Proof of Concept
### Included files
- `Editor/BuildWorkshopOpenUrlPoc.cs`: builds the workshop map bundle.
- `Verify-OpenUrlChain.ps1`: validates the Unity callback primitive and harmless Windows executable-launch behavior.
### Build the map bundle
1. Open the map-authoring project in Unity `6000.3.7f1`, matching the game.
2. Ensure the game's `Assembly-CSharp.dll` and required dependencies are available so that `Donteco.MapHiderSpawn` and `Donteco.MapSeekerRoomAnchor` resolve.
3. Copy `Editor/BuildWorkshopOpenUrlPoc.cs` into the project's `Assets/Editor/` directory.
4. Select **Hollow Hide → Build Harmless OpenURL Workshop PoC**.
5. The builder writes `PocBundleOutput/workshop_openurl_poc.bundle`.
6. Upload it only to a private, researcher-controlled Workshop item.
7. In an isolated test environment, host a lobby and select that item. Connect a second test client and prepare/start the map.
8. Expected result: Calculator launches on the client when the prefab is instantiated.
Do not replace the Calculator URI with a destructive or unauthorized payload. Use only researcher-owned systems and test Workshop items.
### Run the local primitive verification
From the game directory:
```powershell
powershell -ExecutionPolicy Bypass -File poc/Verify-OpenUrlChain.ps1
```
The script passes when it:
- resolves `Application.OpenURL(string)` through the shipped `UnityEventBase.GetValidMethodInfo` implementation;
- constructs the shipped `CachedInvokableCall<string>` for the static callback; and
- launches a harmless marker executable through a `file://` URI and verifies its output file.
## Evidence and Current Limitations
Confirmed during analysis:
- The workshop loader instantiates arbitrary prefab components without an allowlist.
- Missing maps are automatically subscribed to and downloaded during map preparation.
- Timeline `SignalReceiver.OnNotify` invokes the matching serialized `UnityEvent`.
- The shipped Unity resolver accepts static `Application.OpenURL(string)` and constructs the corresponding invokable call.
- A harmless executable was successfully launched through a Windows `file://` URI.
- Direct `Process.Start(string)` was rejected by void-delegate construction and is not the reported vector.
Not yet confirmed:
- Building the final bundle with Unity Editor and executing it inside a live Hollow Hide multiplayer client.
- Whether the suspected incident used this exact technique. That requires the original Workshop item or bundle.
## Recommended Remediation
- Treat every Workshop bundle as untrusted active content, not passive map data.
- Before instantiation, inspect every component on the prefab and all children, including inactive objects. Enforce a strict allowlist of required map markers and explicitly reviewed visual components.
- Reject Timeline components, serialized `UnityEvent` callbacks, animation events, unknown scripts, and other callback-capable components. Do not instantiate first and sanitize afterward because lifecycle callbacks may already have run.
- Prefer a declarative map format containing only vetted geometry, transforms, and spawn markers instead of arbitrary Unity prefabs.
- Require clear user consent before automatically subscribing to and downloading a host-selected Workshop item.
- Continue checking host/client file consistency, but do not treat the MD5 value as a security control. If trusted publisher authenticity is required, use signed content from an authorized publisher rather than a host-supplied hash.