# ROBLOX-BETA-ANTICHEAT-


# Sentinel — Server-Authoritative Anti-Cheat for Roblox

**Created by 9okXE** | Open Source | Free to Use

---

## What is Sentinel?

Sentinel is a modular, server-authoritative anti-cheat framework for Roblox games. Instead of punishing players for a single suspicious moment (which could be lag), Sentinel collects evidence over time and only acts when there's a clear pattern of abuse.

### Key Features

- **Evidence-Based Scoring** — No single event triggers a ban. Risk accumulates over time, decays for minor issues, and only escalates with repeated or severe anomalies.
- **5 Detection Modules** — Movement (speed, teleport, fly, noclip), Combat (damage, range, cooldowns, line-of-sight), Economy (transaction validation), Remote Validation (schema-based payload checking), and Rate Limiting (per-player per-remote).
- **Honeypot Traps** — Fake RemoteEvents that no legitimate client ever fires. Exploit scripts that scan and fire them generate instant maximum-severity evidence. Zero false positives by design.
- **Server Reconciliation (Rubber-band)** — Instead of only filing evidence for movement hacks, the server snaps the player back to their last valid position. Cheaters are stuck in a loop.
- **Replay Recorder** — Server-side recording of flagged player positions, actions, and evidence for post-game admin review.
- **Admin Live Dashboard** — Real-time ScreenGui showing risk scores, evidence feeds, and manual Kick/Ban/Pardon buttons. Toggle with backtick key `` ` ``.
- **Escalation Ladder** — Warn → Rollback → Kick → Ban. Only bans with strong accumulated evidence. Bans persist via DataStore.
- **Configurable** — All thresholds live in a single Config module. Tune without touching detector code.
- **Zero Client Trust** — All detection runs server-side. No client-side anti-cheat code for exploiters to bypass.

---

## Installation (5 minutes)

### Method: Import .rbxm File

1. Download `Sentinel.rbxm`
2. Open your game in **Roblox Studio**
3. In the **Explorer** panel, right-click on the **root** (top-level "Workspace" area / game icon)
4. Click **Insert from File…**
5. Select the downloaded `Sentinel.rbxm`
6. The full folder structure will be imported automatically

### Verify the Structure

After importing, your Explorer should look like this:

```
ReplicatedStorage
  Shared
    Sentinel
      Types (ModuleScript)
      Config (ModuleScript)
      Net
        Remotes (Folder)
          HP_GiveCoins (RemoteEvent)
          HP_SetLevel (RemoteEvent)
          HP_SpawnItem (RemoteEvent)
          HP_GodMode (RemoteEvent)
          HP_UnlockAll (RemoteEvent)
          SentinelAdminSync (RemoteEvent)

ServerScriptService
  Server
    Sentinel
      Boot.server (Script)
      AntiCheatService (ModuleScript)
      EvidenceStore (ModuleScript)
      ActionPolicy (ModuleScript)
      Logger (ModuleScript)
      Reconciliation (ModuleScript)
      ReplayRecorder (ModuleScript)
      AdminBridge (ModuleScript)
      Detectors (Folder)
        MovementDetector (ModuleScript)
        CombatDetector (ModuleScript)
        RemoteValidationDetector (ModuleScript)
        EconomyIntegrityDetector (ModuleScript)
        RateLimitDetector (ModuleScript)
        HoneypotDetector (ModuleScript)

StarterPlayer
  StarterPlayerScripts
    Client
      SentinelClient.client (LocalScript)
      SentinelAdmin.client (LocalScript)
```

### Configure Admin Access

1. Open **ServerScriptService/Server/Sentinel/AdminBridge**
2. Find the `ADMIN_IDS` table near the top:
```lua
local ADMIN_IDS: {number} = {
    949032364,   -- Replace with your UserId
}
```
3. Replace with your own Roblox UserId (find it at `https://www.roblox.com/users/YOUR_ID/profile`)
4. Add any other admin UserIds on separate lines

### Enable DataStore (Required for Bans)

1. Go to **Game Settings → Security**
2. Toggle **Enable Studio Access to API Services** to ON
3. Make sure your game is **published** (File → Publish to Roblox)

### Test It

1. Click **Play** (F5)
2. Open the **Output** window (View → Output)
3. You should see:
```
[Sentinel] Anti-cheat system ONLINE — detectors active: 1 tick-based
[Sentinel] HoneypotDetector: 5 traps armed
[Sentinel] ReplayRecorder: initialized
[Sentinel] AdminBridge: online, 1 admins whitelisted
[Sentinel] Boot.server V2: initialization complete — all modules online
[Sentinel] Client companion loaded
[Sentinel] Admin dashboard loaded — press ` to toggle
```
4. Press the **` (backtick)** key to open the Admin Dashboard

---

## Connecting Your Game's Remotes

Sentinel doesn't protect remotes it doesn't know about. You need to register your game's RemoteEvents with schemas.

### Example: Register a Damage Remote

In **Boot.server**, add:

```lua
-- Create the remote
local myRemote = Instance.new("RemoteEvent")
myRemote.Name = "DealDamage"
myRemote.Parent = remotesFolder

-- Register with Sentinel (schema defines what's valid)
AntiCheatService.registerRemote({
    name = "DealDamage",
    rateLimit = 8,       -- max 8 fires per second
    args = {
        { argType = "number", min = 0, max = 100 },   -- damage
        { argType = "string" },                         -- weaponId
    },
})

-- Wire the handler with validation
myRemote.OnServerEvent:Connect(function(player, damage, weaponId)
    local valid, reason = AntiCheatService.validateRemote(
        player, "DealDamage", {damage, weaponId}
    )
    if not valid then return end -- rejected; evidence already filed

    -- Your game logic here (server-authoritative)
end)
```

### Grant Teleport Immunity

If your game has teleport pads or fast travel, grant immunity so players don't get flagged:

```lua
-- Call this BEFORE teleporting the player
AntiCheatService.setImmune(player, 3.0)  -- 3 seconds of immunity
```

---

## Tuning Thresholds

Open **ReplicatedStorage/Shared/Sentinel/Config** and adjust:

| Setting | Default | What It Controls |
|---------|---------|-----------------|
| `maxWalkSpeed` | 50 | Max horizontal speed before flagging (studs/sec) |
| `maxJumpPower` | 100 | Max jump power allowed |
| `teleportThreshold` | 80 | Studs moved in one sample = suspect teleport |
| `pingBuffer` | 0.2 | Latency tolerance (higher = fewer false positives) |
| `maxDamagePerHit` | 150 | Max single-hit damage allowed |
| `maxAttackRange` | 30 | Max attack distance (studs) |
| `warnThreshold` | 15 | Risk score to issue warning |
| `kickThreshold` | 40 | Risk score to kick |
| `banThreshold` | 80 | Risk score to ban |
| `decayRate` | 2 | Points decayed per interval (higher = more forgiving) |

If your game has speed boosts or dashes, increase `maxWalkSpeed`. If players have high ping, increase `pingBuffer`.

---

## Admin Dashboard Controls

- **Toggle:** Press `` ` `` (backtick key) during gameplay
- **Kick:** Removes player from server (they can rejoin)
- **Ban:** Removes player and persists ban to DataStore (permanent until pardoned)
- **Clear:** Resets player's risk score and evidence, removes any existing ban

### Unban a Player Manually

If you need to unban someone outside the dashboard, paste this in the **Command Bar** (server context):

```lua
local userId = 123456789 -- replace with their UserId
game:GetService("DataStoreService"):GetDataStore("SentinelBans"):RemoveAsync("ban_" .. tostring(userId))
print("Unbanned", userId)
```

---

## How Detection Works (Technical Summary)

1. **Five detectors** run independently on the server
2. Each detector can file **evidence** (severity 1–3) against a player
3. Evidence accumulates into a **risk score** (severity × 3 points per entry)
4. Risk score **decays** over time (2 points every 5 seconds by default)
5. When score crosses thresholds: **warn** (15) → **kick** (40) → **ban** (80)
6. Honeypot remotes generate **severity 3** evidence (9 points) with zero false positives
7. Movement violations trigger **rubber-banding** — player is snapped back to last valid position
8. Players above suspicion threshold (10) are **recorded** for replay review

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| No output on Play | Boot.server is a ModuleScript | Change to Script (right-click → check ClassName) |
| Client warning UI missing | SentinelClient.client is a Script | Change to LocalScript |
| Dashboard doesn't appear | Your UserId not in ADMIN_IDS | Edit AdminBridge, add your UserId |
| False noclip on normal walk | Raycast hitting own character | Update to latest MovementDetector (character filter) |
| Kicked immediately on join | Previous test ban persists | Run unban command in Command Bar (see above) |
| DataStore errors | API access not enabled | Game Settings → Security → Enable Studio Access |
| Remote fires rejected | Schema mismatch | Check argType matches typeof() return value exactly |

---

## License

**MIT License** — Free to use, modify, and distribute. Credit appreciated but not required.

```
MIT License

Copyright (c) 2025 9okXE

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

**Built by 9okXE** — Sentinel Anti-Cheat for Roblox
