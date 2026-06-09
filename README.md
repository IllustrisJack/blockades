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
| Side | auto | 0=auto (uses gate type), 1=destination-facing, -1=interior-facing. |
| Formation Size | auto | Total ships in formation; auto-derives from your wing. |
| Slot Index | auto | This ship's slot; auto-derives from position in wing. |

Solo ships sit directly in front of the gate. Wings of 2–7 ships use one inner ring (theta=30°); 8–13 add an outer ring (theta=60°); 14+ add a flat ring (theta=80°). All on the hostile-facing hemisphere.

## AI behavior

Every 30 minutes (configurable in `md/blockade.xml`'s `Init`), the mod scans every active gate. For each gate where:

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

## Configuration

Tunables live in `md/blockade.xml` under the `VerifyVariables` cue:

```
$Enable               master toggle
$ScanIntervalMin      minutes between scans            (30)
$MinFleetCapital      destroyers per blockade          (1)
$MinFleetEscort       frigates per blockade            (3)
$MinFleetFighter      fighters per blockade            (4)
$BlockadeDistance     order's distance param           (6km)
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
├── md/
│   └── blockade.xml                 AI auto-blockade scan + dispatch
├── t/
│   └── 0001.xml                     localization (page 20810)
└── README.md
```

No vanilla files are diffed. The mod is pure additions, so it doesn't conflict with anything.

## Compatibility

- X4 9.0+. The order delegates to vanilla's `move.generic` and `move.seekenemies` aiscripts for path-finding and engagement, so it inherits vanilla combat behavior changes automatically.
- All Egosoft DLCs supported (split, terran, pirate, boron, timelines).
- Save-safe: the mod stores state under `md.$BBVarTable` and re-initializes missing keys on load.
- Removing the mod mid-save will leave any spawned ships with an invalid order; they fall back to the previous default behavior.

## License

Dual-licensed under MIT or Apache-2.0, at your option. See `LICENSE-MIT` and `LICENSE-APACHE`.
