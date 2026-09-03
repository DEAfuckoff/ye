# Custom Punchy-compatible animations — progress log

Goal: replace the Punchy mod's animation clips with our own hand-authored ones so the pack
is original work. Only Punchy's *functional* hooks (clip names, bone names, timeline effects)
are kept so the mod still triggers the right clip at the right time — every pose and every
keyframe is written from scratch, not copied or numerically tweaked from Punchy's files.

## Format / rules we follow

- Bedrock animation JSON, `format_version: "1.8.0"`, one clip per `*.animation.json`.
- Bones: `right_arm`, `left_arm` (arm swing) and `itemgrip_right`, `itemgrip_left` (the held item).
- Keys are sparse and hand-placed on a 24 fps grid (`0.0417` steps) with `lerp_mode: "catmullrom"`
  so Blockbench/Minecraft draw a smooth spline between a handful of poses instead of us baking
  a value on every frame (Punchy's originals are dense per-frame data — ours are not).
- Punchy hooks kept verbatim because they're what makes the clip *work*, not how it *looks*:
  `dual_handed`, `left_switch_item_<item>`, `hide_left_switch_item_<item>`, `start_mesh_animation`,
  `anim_speed_<x>`, `load_bow` sound, `loop: true` / `loop: "hold_on_last_frame"`.
- Clips that chain (eat_start → eat_loop → eat_end) must share the exact hand-off pose on
  their boundary frames, otherwise the arm pops.

## Style direction

"Next level" but readable: clear anticipation → action → settle, slight overshoot on big moves,
asymmetric timing (fast out, slow in), and tiny micro-motion in held poses so nothing looks frozen.
Squash/stretch is only used on the item grip (food), never on the arms.

## Log

### PR #1 — `eat_loop` and `use_bow_right` rebuilt from scratch (merged)

**`eat_loop.animation.json`** — original was a 1.125 s per-frame dump with mirrored arms.
Rewritten as a 1.29 s loop with a story per cycle:

1. bite — food pushed into the mouth, grip squashed (`1.16 / 0.85` scale)
2. tear — arm swings ~11° down/back with a rotation kick on the grip
3. three chew bobs, each slightly smaller, food slowly turning
4. return to the mouth with a small overshoot
5. nibble accent at the end, then loop

First and last frame = `eat_start` last frame = `eat_end` first frame
(`right_arm` rot `[2.01, -62.01, -17.18]`, pos `[1.9, 3.5, 0]`, grip rot `[0, 6, -92]`).
Timeline `anim_speed_1.2`.

**`use_bow_right.animation.json`** — original was a straight two-hand raise.
Rewritten as a 3.42 s `hold_on_last_frame` clip with four beats:

1. **raise (0 – 0.67 s)** — right arm swings the bow up canted (~28° → 77° Z), overshoots and
   settles to ~70°; grip counter-rotates a few degrees so the bow "whips".
2. **quiver reach (0 – 0.46 s)** — left arm starts pointing back over the shoulder (`-87.5°` X),
   comes forward in an arc to the string.
3. **nock (0.83 s)** — left hand meets the bow; timeline fires `start_mesh_animation` +
   `hide_left_switch_item_minecraft:arrow` here, `load_bow` sound at 0.92 s.
4. **draw (0.83 – 2.0 s)** — fast-out/slow-in pull to the cheek (left arm Z 20° → 70°,
   position Z −1.5 → 24), elbow flares outward as the string comes back.
5. **strain hold (2.0 – 3.42 s)** — ±0.3–0.5° / ±0.1 unit tremor on both arms so the pose breathes.

Timeline `0.0: dual_handed;left_switch_item_minecraft:arrow;`.

### PR #2 — `eat_loop` softened (merged)

In-game feedback: loop felt "hard" / jerky. Changes, all in `eat_loop.animation.json`:

| what            | before          | after        |
|-----------------|-----------------|--------------|
| loop length     | 1.2917 s        | 1.5 s        |
| `anim_speed`    | 1.2             | 1            |
| tear swing      | ~11° X, −1.0 Z  | ~6.5° X, −0.5 Z |
| chew bobs       | ±0.35 / ±3°     | ±0.15 / ±1.5° |
| squash/stretch  | 1.16 / 0.85     | 1.08 / 0.92  |
| grip snaps      | every chew      | removed — food just drifts a few degrees |
| key spacing     | some 2-frame gaps | evenly spaced, no spline overshoot |

Boundary pose unchanged so it still chains with `eat_start` / `eat_end`.

## Testing

- Pack: put the `*.animation.json` files in `assets/minecraft/punchy/animations/` inside a
  resource pack, enable it, `F3+T` to reload.
- Bow: bow in main (right) hand, arrows anywhere in inventory, hold right-click.
- Blockbench: open Punchy's player template model, Animate → Import Animations → pick the JSON.
  Parent a placeholder cube (or the vanilla `bow.json` item model) to `itemgrip_right` and a thin
  cube to `itemgrip_left` to see where the bow/arrow sit. Item switch/hide effects are runtime-only.

## Next up (not started)

- `eat_start`, `eat_end` restyled to match the new loop
- `use_bow_left` (mirror of bow_right with its own timing)
- remaining clips: drink, punch, mining, attack_*, hand_in/out, swim, row, ...
