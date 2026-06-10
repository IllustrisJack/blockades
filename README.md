# Blockades

X4: Foundations mod. Adds a **Blockade** default behavior: ships hold a hemisphere formation on the hostile side of a target gate and engage hostiles transiting through it. AI factions at war auto-deploy blockade fleets at gates leading from enemy claimspace into their own.

Built for X4 9.0+.

---

## Player usage

1. Select a ship (or wing).
2. Open the Behavior dropdown.
3. Pick **Blockade**.
4. Choose a gate as the target.

Optional params (visible under "Advanced"):

| Param | Default | Notes |
|---|---|---|
| Distance from Gate | 6 km | Radius of the formation sphere. |
| Engagement Range | ship's radar range | Hostiles outside this are ignored. |
| Pursue Targets | off | If on, ships leave formation to chase. |
| Attack on Sight | on | If off, ships only react when attacked. |
| Face Away From Gate | off | Reverse the formation's lookat. |
| Side | auto | 0=auto (uses gate type), 1=destination-facing, -1=interior-facing. |
| Formation Size | auto | Total ships in formation; auto-derives from your wing. |
| Slot Index | auto | This ship's slot; auto-derives from position in wing. |
| Timeout | 0s | 0 = infinite. |

## Recommended leader

Use an L (destroyer) or M (frigate / corvette) as the wing leader. **XL pathfinding currently has issues** — XL hulls work fine as subordinates.

## Formation behavior

- **Solo ship** sits in slot 0 directly in front of the gate (offset slightly perpendicular to the highway centerline so XL/L hulls don't fight transiting mass traffic).
- **Wings** sort subordinates by class — L/XL fill the inner ring (θ=30°, closer to leader), then M, then S and others spill into the outer ring (θ=60°) and the flat ring (θ=80°) beyond that.
- **Leader is bastion-style**: holds position at slot 0, doesn't chase. Its weapons are flipped to `weaponmode.attackenemies` for the duration so turrets fire at any hostile that wanders into range. Existing `missiledefence` and `mining` turret assignments are preserved. Original modes are restored when the blockade ends.
- **Subordinates** hold their slot via vanilla `MoveWait`. They react to direct attacks via `MoveWait`'s built-in `InterruptAttack`. Once the leader has reached its slot, its hold loop scans for hostiles within `engagerange` of the gate (and of the leader itself); when one is found, each sub is dispatched to `AttackInRange` rallied on the leader's current position with `enforceradius`. Subs engage everything in range, then fall back to slot via their `MoveWait` default.
- **Leader positioning** uses vanilla `move.generic` called once on entry to the order — matches how vanilla `MoveWait` positions subs. After arrival the script holds via `stop_moving` + `stop_boost` + `commandaction.standingby` (the same tail vanilla `order.move.wait` uses). The hold loop refines facing in place via a rotation-only `move_to` each iteration. The loop only re-runs `move.generic` if the leader is knocked materially off slot (drift > `$exittol`).

When the leader's order is replaced or cancelled, the cleanup pass cancels each sub's current order and default order — switching gates won't leave a sub drifting back to the previous slot.

## AI behavior

Every 30 minutes (configurable in `md/blockade.xml`'s `VerifyVariables`), the mod scans every active gate. For each gate where:

- the sector-side owner is at war with the destination-side owner, and
- both factions are eligible (not Xenon/Khaak/civilian/player/etc.), and
- the defender has at least one shipyard, and
- no blockade fleet already covers this gate,

…a blockade fleet (1 destroyer + 3 frigates + 4 fighters by default) is spawned at the defender's nearest shipyard and dispatched. Each ship is assigned the Blockade order with a slot index, so they fan out into the hemisphere formation automatically.

Existing blockades are checked the same interval and disbanded when:

- the gate or its destination no longer exists,
- the destination sector flipped to the defender's faction,
- the war ended (relation > -0.1), or
- all ships in the blockade are dead.

When an AI blockade is established or disbanded, a player logbook entry is written under the **News** category (gated by `$NewsEnabled`, default on). Uses vanilla `add_player_log` — no dependency on any external news framework.

## Configuration

Tunables live in `md/blockade.xml` under the `VerifyVariables` cue:

```
$Enable               master toggle                    (true)
$ScanIntervalMin      minutes between scans            (30)
$MinFleetCapital      destroyers per blockade          (1)
$MinFleetEscort       frigates per blockade            (3)
$MinFleetFighter      fighters per blockade            (4)
$BlockadeDistance     order's distance param           (6km)
$NewsEnabled          player logbook entries           (true)
$DebugDetailed        per-event debug_text             (false)
$ExcludedFactions     list of factions to skip
```

To change a tunable for an in-progress save, edit the value via the in-game debug console:

```
md.Blockade.$BBVarTable.$Enable = false
```

Restart required after editing the XML itself (X4 only re-parses MD scripts on full process launch).

## Files

```
blockades/
├── content.xml
├── aiscripts/
│   └── order.move.blockade.xml      the order definition + behavior loop
├── libraries/
│   └── icons.xml                    map icon alias
├── md/
│   └── blockade.xml                 AI auto-blockade scan + dispatch
├── t/
│   └── 0001.xml                     localization (page 20810)
├── scripts/
│   └── publish.ps1                  Steam Workshop publish helper
└── README.md
```

No vanilla files are diffed. The mod is pure additions, so it doesn't conflict with anything.

## Compatibility

- X4 9.0+. The order delegates to vanilla `move.generic` for pathing/positioning and dispatches vanilla `AttackInRange` for sub engagement, so combat behavior tracks vanilla changes automatically.
- Leader weapon-mode override uses direct `<set_weapon_mode>` calls (one per combat weapon) and restores the prior modes on finish/abort — uninstalling the mod after a blockade ended is clean. Uninstalling **during** an active blockade will leave the leader's weapons on `attackenemies` until the player resets the mode manually.
- Leader main-batteries: L/XL forward-mounted weapons don't auto-fire under the Blockade order (vanilla X4 requires an `Attack`-style order context to aim them, and dispatching such an order would tear down our scan loop). Turrets engage via the `attackenemies` mode flip; main batteries stay idle. Active L/XL engagement is on the roadmap.
- All Egosoft DLCs supported (split, terran, pirate, boron, timelines).
- Save-safe: the mod stores state under `md.$BBVarTable` and re-initializes missing keys on load.
- Removing the mod mid-save will leave any spawned ships with an invalid order; they fall back to the previous default behavior.

## License

Dual-licensed under MIT or Apache-2.0, at your option. See `LICENSE-MIT` and `LICENSE-APACHE`.
