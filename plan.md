Yes — **technically and practically, a plugin can do this**, and the best way to make it truly **global across RuneLite config profiles** is to **store the hotkey→profile bindings outside normal plugin config**. Here’s why. ([static.runelite.net][1])

RuneLite already gives plugins the pieces you need:

* `KeyManager` can register/unregister key listeners. ([static.runelite.net][1])
* `HotkeyListener` is the standard helper for binding a `Supplier<Keybind>` to `hotkeyPressed()` / `hotkeyReleased()`. ([static.runelite.net][2])
* `ProfileManager.Lock` lets you enumerate profiles and resolve them by name, id, or predicate. ([static.runelite.net][3])
* `ConfigManager.switchProfile(ConfigProfile)` is a real public method, and when it runs it saves current config, swaps profile data, emits `ConfigChanged` events for changed keys, and then emits `ProfileChanged`. ([GitHub][4])

The reason your original idea gets tricky is that RuneLite profiles are explicitly **separate sets of plugins and settings**, and `ConfigManager.getConfiguration(groupName, key)` / `setConfiguration(groupName, key, value)` operate on the current `configProfile`. The overloads with a non-null `profile` parameter target the separate RuneScape-profile store, not some global app-wide config bucket. So normal plugin config is **not** a stable global place to keep cross-profile hotkey mappings. ([GitHub][5])

On the “allowed” question: I did **not** find a published Plugin Hub rule that specifically forbids a plugin from keeping its own small data file. The published review language says Hub plugins are reviewed to ensure they are not malicious and do not break Jagex’s rules; it does not describe a ban on plugin-owned local storage. That means I would treat local file storage as **likely acceptable**, but still subject to review quality and scope. ([GitHub][6])

I also do not see a public “global plugin config” API in current RuneLite, while current RuneLite does expose stable filesystem locations such as `RuneLite.RUNELITE_DIR`, and official RuneLite discussions already reference plugin-specific subdirectories under that directory for plugin-managed assets. That makes a plugin-owned file under RuneLite’s data folder the cleanest fit for this use case. ([GitHub][7])

## Best solution

Use a **small external file** for the bindings, and use RuneLite’s profile API only for the actual switch.

Recommended design:

1. **Hotkeys**
   Register one or more `HotkeyListener`s through `KeyManager`. ([static.runelite.net][1])

2. **Global binding storage**
   Save bindings to something like:

```text
~/.runelite/profilehotkeys/bindings.json
```

or a plugin-namespaced equivalent under `RuneLite.RUNELITE_DIR`. RuneLite already uses that directory as its main local data area. ([GitHub][7])

3. **Profile resolution**
   Store both:

   * a stable lookup key you prefer (`profileId`)
   * a human fallback (`profileName`)

   Then resolve by id first, fallback to exact name. `ProfileManager.Lock` supports both forms. ([static.runelite.net][3])

4. **Switching**
   On hotkey:

   * acquire `ProfileManager.Lock`
   * find target profile
   * call `configManager.switchProfile(target)` ([static.runelite.net][3])

5. **Guard against recursion**
   Because switching a profile emits lots of config-change events and then `ProfileChanged`, your plugin should ignore its own transient reload/sync noise and avoid re-triggering a switch while one is already in progress. ([GitHub][4])

## Important caveat

For the switcher to remain usable **after** the profile change, your plugin must still be enabled in the target profile. Since profiles contain separate plugin states/settings, switching into a profile where your plugin is disabled will stop future hotkey listening there. ([GitHub][5])

So the practical rule is:

* **Plugin enabled in every target profile**
* **Bindings stored externally, not in normal plugin config**

That is the most robust version of a true global hotkey profile switcher.

## What I would avoid

I would **not**:

* store the bindings in ordinary plugin config
* rely on RuneScape-profile config overloads as a “global” store
* assume a profile name alone is always enough without a fallback strategy ([GitHub][4])

## Recommended plugin shape

A solid MVP architecture would be:

* `ProfileHotkeysPlugin`
* `ProfileHotkeysService`

  * load/save external JSON
  * resolve profiles
  * perform guarded switch
* `ProfileHotkeysPanel`

  * list profiles from `ProfileManager.Lock.getProfiles()`
  * let user assign hotkeys
  * write to JSON immediately ([static.runelite.net][3])

If you want, I can draft the exact class skeleton and data model for this plugin next.

[1]: https://static.runelite.net/runelite-client/apidocs/net/runelite/client/input/KeyManager.html?utm_source=chatgpt.com "KeyManager (RuneLite Client 1.12.11 API)"
[2]: https://static.runelite.net/runelite-client/apidocs/net/runelite/client/util/HotkeyListener.html "HotkeyListener (RuneLite Client 1.12.20 API)"
[3]: https://static.runelite.net/runelite-client/apidocs/net/runelite/client/config/ProfileManager.Lock.html "ProfileManager.Lock (RuneLite Client 1.11.16 API)"
[4]: https://github.com/runelite/runelite/blob/master/runelite-client/src/main/java/net/runelite/client/config/ConfigManager.java "runelite/runelite-client/src/main/java/net/runelite/client/config/ConfigManager.java at master · runelite/runelite · GitHub"
[5]: https://github.com/runelite/runelite/wiki/General-Features "General Features · runelite/runelite Wiki · GitHub"
[6]: https://github.com/runelite/plugin-hub "GitHub - runelite/plugin-hub: External plugins for RuneLite · GitHub"
[7]: https://github.com/runelite/runelite/blob/master/runelite-client/src/main/java/net/runelite/client/RuneLite.java "runelite/runelite-client/src/main/java/net/runelite/client/RuneLite.java at master · runelite/runelite · GitHub"
