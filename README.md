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
| Distance from Gate | 10 km | Radius of the formation sphere. Keeps the ring clear of the gate-traffic envelope so ships actually settle. |
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
- **Leader is bastion-style**: holds position at slot 0, doesn't chase. Its weapons are flipped to `weaponmode.attackenemies` for the duration so turrets fire at any hostile that wanders into range. Existing `missiledefence` and `mining` turret assignments are preserved. Original modes are restored when the blockade ends. (The leader's forward main batteries stay idle — giving the leader a full Attack order would preempt the Blockade order itself; subordinates do the main-gun work.)
- **Subordinates** hold their slot via vanilla `MoveWait` — perfectly still while the area is quiet; nothing in the script touches a holding sub. The leader's hold loop scans its gravidar (vanilla `find_gravidar_contact` + `maybeattackedby`, the same primitive `order.fight.attack.inrange` uses) for hostiles within `engagerange` of the gate. When any are found, every sub still on `MoveWait` gets a full vanilla **`Attack`** order — primary = the hostile closest to the gate, the rest as secondaries, `allowothertargets` for whatever shows up next — bounded to the engagement area (`radius = engagerange` anchored at the gate, `enforceradius`). The full Attack context is what brings subordinate L/XL main batteries to bear. When a sub's Attack ends it falls back to its `MoveWait` default and flies home; if enemies remain — or new ones arrive while it's still flying back — the leader re-dispatches it, at most once per 30 s per sub. When the leader's scan comes up empty it actively recalls every sub it dispatched (orders are tagged `internalorder`; player-issued Attack orders are never touched, and recall is skipped when Pursue Targets is on).
- **Late joiners**: ships attached to the wing mid-blockade are automatically assigned the next free slot on the leader's next loop tick — no need to reissue the order. Launched drones are excluded from slots and combat dispatch (the carrier's drone logic owns them).
- **Leader positioning** uses vanilla `move.generic` called once on entry to the order — matches how vanilla `MoveWait` positions subs. After arrival the script holds via `stop_moving` + `stop_boost` + `commandaction.standingby` (the same tail vanilla `order.move.wait` uses). The hold loop refines facing in place via a rotation-only `move_to` each iteration (the `order.move.recon` idiom). The loop only re-runs `move.generic` if the leader is knocked materially off its **arrival position** (safepos offset from the geometric slot is expected and doesn't count as drift).

When the leader's order is replaced or cancelled, the cleanup pass cancels each sub's current order and default order — switching gates won't leave a sub drifting back to the previous slot.

## AI behavior

Deployment is owned by X4's **job system**, not by MD spawn-cheating. `libraries/jobs.xml` adds, per major race (Argon, Paranid, Teladi, Split, Terran, Boron):

- `blockade_<race>_l` — the wing leader. L destroyer, runs the `Blockade` order. Built at the faction's own shipyards via `<environment buildatshipyard="true"/>` and `<location ... faction="<race>" relation="self"/>` so each faction only deploys these in its own claimspace.
- `blockade_<race>_esc_m` — M frigate / corvette subordinate (`wing="3"`), runs `Escort` as default; once the leader's Blockade init enumerates its subordinates, MoveWait is dispatched to each at its slot.
- `blockade_<race>_esc_s` — S heavy-fighter subordinate (`wing="4"`).

Quotas: `galaxy="6" cluster="1"` per race — at most 6 blockade fleets across the galaxy per faction, and at most 1 per cluster. The job system queues these at shipyards over real time, respecting the faction's economy.

The Blockade order's init (in `aiscripts/order.move.blockade.xml`) **auto-picks its target gate** when none was passed in — it scans the ship's current sector via `find_gate active="true"` and selects the first active gate whose destination owner is actively hostile to the ship's owner (`mayattack` AND `relationto ≤ -0.25`). Same hostile-pair test as the previous MD scan, but evaluated per-ship at spawn time instead of by a galaxy-wide MD timer.

When the hostile relationship ends (relation thaws, owner change, gate destroyed), the order's interrupt handlers fire and the ship's Blockade order finishes naturally. The job system then decides on its next tick whether to re-queue a replacement based on the current claimspace state.

`md/blockade.xml` is now a thin state-table holder — the scan/dispatch cues that previously spawned ships were removed when jobs took over. Legacy save data is harmless (just unused entries).

## Files

```
blockades/
├── content.xml
├── aiscripts/
│   └── order.move.blockade.xml      the order definition + behavior loop
├── libraries/
│   ├── icons.xml                    map icon alias (additive)
│   └── jobs.xml                     blockade jobs (additive, per race)
├── md/
│   └── blockade.xml                 state-table holder (deployment owned by jobs.xml)
├── t/
│   └── 0001.xml                     localization (page 20810)
├── scripts/
│   ├── publish.ps1                  Steam Workshop publish helper
│   └── ...
└── README.md
```

All diffs are additive (`<add>` selectors against vanilla `<jobs>` and `<icons>` roots) — no vanilla files are replaced. Conflicts with other mods are unlikely.

## Compatibility

- X4 9.0+. The order delegates to vanilla `move.generic` for pathing/positioning and dispatches vanilla `Attack` orders for sub engagement, so combat behavior tracks vanilla changes automatically.
- Leader weapon-mode override uses direct `<set_weapon_mode>` calls (one per combat weapon) and restores the prior modes on finish/abort — uninstalling the mod after a blockade ended is clean. Uninstalling **during** an active blockade will leave the leader's weapons on `attackenemies` until the player resets the mode manually. Subordinate turrets are also flipped to `attackenemies` on dispatch and are **not** auto-restored (no per-sub backup table); reconfigure manually if needed after the blockade ends.
- Subordinate L/XL ships fire their main batteries via the dispatched `Attack` orders. The **leader's** mains stay idle — a self-Attack would preempt the Blockade order itself and dismantle the ring. Leader turrets engage via the `attackenemies` mode flip.
- Player-placed blockades are intent-respecting: you can blockade a *neutral* gate (no auto-cancel when the destination isn't formally hostile) and stack onto a gate another faction is already blockading. AI blockades end when their war ends so the job system can re-task the fleet.
- All Egosoft DLCs supported (split, terran, pirate, boron, timelines).
- Save-safe: the mod stores state under `md.$BBVarTable` and re-initializes missing keys on load.
- Removing the mod mid-save will leave any spawned ships with an invalid order; they fall back to the previous default behavior.

## License

Dual-licensed under MIT or Apache-2.0, at your option. See `LICENSE-MIT` and `LICENSE-APACHE`.
