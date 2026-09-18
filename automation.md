# scene_config.cfg — automation architecture and FHEM/Perl gotchas

This documents how the automations in `scene_config.cfg` work internally, and every
non-obvious FHEM/Perl trap hit while building them. Device inventory and KNX GA
details live in `CLAUDE.md` / `knx_GA_map.md` / `floorplan.md` — this file is about
the *automation logic itself*.

## File map

| File | Role |
|---|---|
| `fhem.cfg` | includes the two files below, plus web/KNX connection setup |
| `device_config.cfg` | KNX device definitions (`LGT_*`, `SHU_*`, ...), `PosPreset_*` dummies |
| `scene_config.cfg` | everything documented here: automations + the preset-apply layer |

To apply any change: edit on rpi4b (or copy a locally-edited file over), `git commit`
in this repo, then `sudo systemctl restart fhem`. FHEM only re-reads config files on
restart — there's no live-reload.

## The one-shot `at`-timer pattern

None of the automations run their actions directly. Instead, each fires once at a
fixed daily time (`training`, `away_morning`) or at sunset (`away_evening`,
`always_on_evening`), and its body *schedules a cascade of one-shot `at` devices*
via `fhem("define <name> at +HH:MM:SS <cmd>")`, one per action, at increasing
offsets. This is what makes a single trigger unfold into a multi-step sequence over
minutes or hours instead of firing everything at once.

- Timer names are namespaced per automation so re-running one automation doesn't
  clobber another's in-flight timers: `e*` (`away_morning`), `m*` (`away_evening`),
  `a*` (`always_on_evening`).
- Each call **redefines** the same timer name every time the parent automation
  fires, so leftover timers from yesterday are simply overwritten today — no
  cleanup needed, no accumulation.
- **Busy-loop guard:** an offset of exactly `+00:00:00` makes FHEM refuse the timer
  with "Cowardly refusing to start a busy loop" (a safety guard against infinite
  recursion). Every generated offset here is guaranteed `>= 1` second — see the
  jitter ranges below, all of which have a positive floor.

## Automations

### `training` (09:25, at-home only)
Placeholder — logs only, no device commands. Extend inside the active branch if a
daily at-home routine is ever needed (e.g. an office light).

### `away_morning` (09:30, away only) — randomized daily
Opens shutters in a realistic wake-up order rather than all at once: parents
(bedroom → bathroom → landing) wake first, then the kids, then downstairs opens
room by room as the household moves toward breakfast and the workday. A second
phase ~3h later re-adjusts a subset of shutters for afternoon sun.

**Beats** (each beat's anchor = previous anchor + a randomized gap; every timer
*within* a beat is a fixed offset from that beat's anchor — see "Random jitter"
below):

| Beat | Anchor device | Members (fixed offset from anchor) |
|---|---|---|
| bedroom | `SHU_F2_bedroom` (+0) | bathroom (+185s), hall (+340s) |
| hannah | `SHU_F2_hannah` | nicolas (+150s) |
| wc | `SHU_F1_wc` | kitchen_window (+125s), kitchen_door (+225s), dining (+335s), living (+490s), office (+590s) |
| phase 2 | `SHU_F2_hall` @ ~+3h | bedroom(+2s), bathroom(+4s), hannah(+6s), nicolas(+8s), kitchen_door(+16s), dining(+18s), living(+20s) |

Target shutter percentages are unchanged from the original design (bedroom 50%,
bathroom 80%, hall 50%, hannah 80%, nicolas 50%; downstairs office/wc/kitchen_door
/dining/living all fully open; kitchen_window stays at 90% for privacy). Only
*order and timing* changed.

### `away_evening` (sunset, away only) — randomized daily
Simulates a full household evening: arrival at dusk → cooking while the kids play
→ a downstairs toilet break → family dinner → living-room TV → kids' bath/bedtime
→ parents' own bedtime. Six beats, same anchor/gap/fixed-delta mechanism as above:

| Beat | Anchor device | What happens |
|---|---|---|
| arrival | `LGT_F1_outside` | porch/hall lights, cellar C1/C3 close, porch light auto-off ~45min later |
| cooking | `SHU_F1_kitchen_window` | kitchen shutters →90% *before* kitchen light; kids play (shutters →90% first); wc break; office/wc/F2-hall close fully (unused rest of night) |
| dinner | `SHU_F1_diningroom` | dining shutter →90% before light, →100% once seated; kitchen/dining lights off after cleanup |
| TV | `SHU_F1_livingroom` | living shutter →90% before light, →100% once settled; first bathroom/shower use |
| kids bedtime | `LGT_F2_hall_stairs` | teeth brushing, bathroom/hannah/nicolas shutters fully close, bedtime-story lights off |
| parents bedtime | `LGT_F1_livingroom` off | final bathroom stop, bedroom shutter →90% before light, →100% once settled |

**Design invariant — shutter before light:** a room's shutter is always driven to
`>=90%` closed *before or at* the same moment that room's light first turns on. An
illuminated room behind a wide-open shutter is an obvious "nobody's home" tell to
an outside observer. This is encoded as a **fixed offset within a beat** (e.g.
kitchen shutter at beat+0/+3, kitchen light at beat+10) — since only the *gaps
between beats* are randomized, not offsets within a beat, this invariant holds
for every possible random draw, not just the nominal case.

**Manual test trigger** (fires now instead of waiting for real sunset):
```
set SCN_Away on
set away_evening modifyTimeToTrigger +00:00:05
```
Runs the away branch in ~5s using the already-defined command; the
`*{sunset("REAL")}` recurrence re-arms itself for the next real sunset afterward.
`set SCN_Away off` again once done. The same technique works for `away_morning`.

### `always_on_evening` (sunset, at-home only)
Closes common-area shutters (hall, kitchen_window, kitchen_door, diningroom,
livingroom, cellar C1/C3 — no other rooms) in two stages: 25 min after sunset to
80% (kitchen_door to 60%), held for 1 hour, then fully closed to 100%. Bedroom,
children's rooms, bathroom, office and WC are deliberately excluded — personal
spaces aren't auto-closed just because someone's home.

### Random jitter (`away_morning`, `away_evening`)
Both automations compute their offsets instead of using literal `+HH:MM:SS`
strings, so the exact timing differs every day:

- A **local `$fmt` closure** (`sub { sprintf("+%02d:%02d:%02d", ...) }`, redefined
  each time the automation fires) converts a second-count into the `+HH:MM:SS`
  string `at` needs.
- Each beat's anchor = the previous beat's anchor + `int(rand($width)) + $floor`
  seconds (a fresh random draw every firing). Ranges were chosen so the total
  runtime stays plausible: `away_morning` phase 1 spans ~24–33 min, phase 2 stays
  within +2h56–3h04; `away_evening` spans ~1h45–2h45 sunset-to-lights-out.
- Offsets *within* a beat are always a **fixed** delta from that beat's anchor —
  this is what makes the shutter-before-light invariant (and all same-room
  on/off ordering) hold regardless of what the random draw produces.

## FHEM/Perl traps hit while building this

These cost real debugging time and, in one case, produced *no visible error at the
point of failure* — worth reading before editing any `at`/`notify` Perl block here.

### 1. FHEM splits on any bare `;`, ignoring `{}` nesting entirely
FHEM's command-chain parser (`AnalyzeCommandChain`) treats **every unescaped `;`
in the whole command as a split point**, even when it's lexically deep inside a
`{ perl block }`, `sub {...}`, or an `if {...} else {...}`. It does *not* track
brace depth to decide "is this semicolon actually top-level." The very first
jitter implementation here used ordinary multi-statement Perl (`my $x = ...; my
$y = ...; fhem(...);`) and every single `;` inside it caused FHEM to log a burst
of `Unknown command my, try help.` / `Unknown command fhem("define, try help.` —
each fragment landing as a bogus standalone command.

**Fix:** double every real semicolon to `;;`. FHEM's parser treats `;;` as one
literal escaped `;` and doesn't split on it; Perl, once it actually gets to
`eval` the de-escaped text, treats the doubled separator as a harmless empty
statement, so program logic is unaffected. This is *why* the `PosPresetApply_*`
notifies (see below) are written as a single semicolon-free statement each —
avoiding the need for `;;` entirely was a deliberate choice, not an oversight.

### 2. `;;` behaves differently for `at` vs `notify` — verified 2026-09-18
This is the sharpest edge here, and it's why it's worth writing down precisely.
Prior work on the `PosPresetApply_*` notifies found that **`notify_Exec` re-runs
`AnalyzeCommandChain` on the stored command at *every* trigger**, not just at
define time — so a `;;`-escaped notify body can parse cleanly once (`list
<notify>` shows a perfectly normal single-`;` DEF, since the escape is consumed
on the first pass) and then silently fragment into no-ops on every subsequent
firing, with nothing but a clean-looking `list` output to go on. That finding is
why every `PosPresetApply_*`/`PosPresetDefault` notify body here is a single
semicolon-free statement (postfix `for (...)`, regex `/r` flag) — not because
`;;` "doesn't work," but because it doesn't survive `notify`'s repeated
re-parsing.

`at`-type devices (`away_morning`, `away_evening`, and every one-shot `eN`/`mN`/
`aN` timer they spawn) do **not** have this problem — verified empirically by
defining throwaway `at` devices with `;;`-escaped multi-statement bodies via the
FHEMWEB command API and firing them repeatedly: correct computed values every
time, no fragmentation, even across multiple re-fires. `at_Exec` evidently
evaluates its stored Perl block directly rather than re-running it through the
chain-splitter the way `notify_Exec` does. **Conclusion: `;;`-escaped
multi-statement Perl blocks are safe in `at` DEFs (including the ones these two
automations dynamically define via `fhem("define ...")`), but must be avoided —
in favor of single-statement bodies — in `notify` DEFs.** If a future edit ever
converts one of these `at`s into a `notify`, this stops being safe and needs
re-testing, not just re-reading.

### 3. Don't name Perl variables `$a` / `$b`
While verifying the above, a test block using `my $a` / `my $b` produced *silent*
`uninitialized value` warnings and empty results, despite the assignments
looking correct. `$a` and `$b` are Perl's special package globals reserved for
`sort` comparators; using them as ordinary lexicals is a well-known footgun that
can misbehave in ways unrelated to whatever a surrounding library call does
internally. None of the real automation code uses these names (variables are
named for what they hold — `$bedroom`, `$beat1`, etc.) — keep it that way.

### 4. Bare `{ }` blocks don't need a separating `;` between them
Perl allows a bare block (`{ ... }`) as a complete statement with no trailing
semicolon required, and several such blocks can simply follow one another with
nothing but whitespace between them: `{fhem("...")}{fhem("...")}` is entirely
valid Perl (two sequential statements), and since there's no `;` anywhere, FHEM's
chain-splitter has nothing to trip over. This is the original style throughout
`scene_config.cfg` for command sequences that don't need cross-statement shared
state. It stops being viable the moment you need a variable computed once and
reused across multiple `fhem(...)` calls (each bare block is *also* its own
lexical scope, so a `my` declared in one is invisible in the next) — which is
exactly why the jittered blocks in `away_morning`/`away_evening` are each ONE
bare block containing many `;;`-separated statements, rather than many small
blocks.

### 5. Safe way to test a new Perl block before touching the real automations
Because trap #2 above means a clean `list <device>` is not proof a block will
behave correctly when it actually fires, don't trust config-load success alone
for anything nontrivial. Test in isolation first, against a harmless `dummy`
device, via the FHEMWEB command API (or telnet on :7072):
```
define zTestDummy dummy
define zTest at +00:00:03 {\
	{\
		my $x = 20 + int(rand(131));;\
		fhem("define zd1 at +00:00:05 set zTestDummy on");;\
	}\
}
```
Wait for it to fire, check the log for `Unknown command` noise, `list zd1` to
confirm the nested define landed with the expected computed value, then `delete`
every `zTest*`/`zd*`/`zTestDummy` device. None of this touches `scene_config.cfg`
— it's pure runtime state, gone on the next FHEM restart even if you forget to
clean up (though cleaning up is still good practice). This is exactly the method
used to verify traps #1–#3 above.

### 6. FHEMWEB command API needs a CSRF token
`curl` against `/fhem?cmd=...` returns `400 Bad Request` without one. Fetch it
first from any response header, then pass it back as a query param:
```
TOKEN=$(curl -sD - "http://localhost:8086/fhem" -o /dev/null | grep -i x-fhem-csrftoken | tr -d '\r' | cut -d' ' -f2)
curl -s --data-urlencode "cmd=list away_evening" --data-urlencode "fwcsrf=$TOKEN" "http://localhost:8086/fhem?XHR=1"
```
(Plain `nc localhost 7072 < commands.txt` against the telnet port works too and
skips the CSRF dance entirely, but line-buffers less predictably for multi-line
DEFs — prefer writing multi-line commands to a file first either way rather than
fighting shell quoting through several layers of `ssh`.)

## Not covered here

FHEMWEB-specific gotchas unrelated to automation logic — `devStateIcon` HTML
formatting rules, icon file extensions, CSS specificity against `brightstyle.css`,
`sortby` for group-box row order — apply to `device_config.cfg` / `mystyle.css`,
not the scene logic above, and aren't repeated here.
