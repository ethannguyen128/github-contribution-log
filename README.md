
# Contribution [1]: Bug with Vulconus and Xoph's Blood

**Contribution Number:** 1
**Student:** Ethan Nguyen 
**Issue:** [[GitHub issue link]  ](https://github.com/PathOfBuildingCommunity/PathOfBuilding/issues/3062)
**Status:** Phase 1 Completed

---

## Why I Chose This Issue

This issue matches my skills and I was intrigued by thinking about the process to fix this issue.

---

## Understanding the Issue

### Problem Description

Path of Building models the dagger **Vulconus** with a config checkbox
("Do you have Avatar Of Fire?") that toggles whether the character currently
has the **Avatar of Fire** buff. That checkbox exists because Vulconus grants
Avatar of Fire only *temporarily* ("Every 8 seconds, gain Avatar of Fire for 4
seconds"), so the user has to tell PoB whether to assume it's active.

The bug appears when the character *also* wears **Xoph's Blood**, an amulet
that grants the **Avatar of Fire keystone permanently**. In that situation the
character *always* has Avatar of Fire, so the Vulconus checkbox should be
irrelevant, but PoB still lets the checkbox drive Vulconus's
Avatar-of-Fire-conditional stats.

### Expected Behavior

When any *permanent* source of Avatar of Fire is present (Xoph's Blood, the
passive tree keystone, an Avatar of Fire tattoo, a Hellscape/Recombinator mod,
etc.), the character should be treated as **always having Avatar of Fire**.
Vulconus's conditional lines should resolve accordingly and the checkbox should
have no effect:
- `50% of Physical Damage Converted to Fire while you have Avatar of Fire` → **always active**
- `(160-200)% increased Critical Strike Chance while you have Avatar of Fire` → **always active**
- `+2000 Armour while you do not have Avatar of Fire` → **never active**

### Current Behavior

The `Condition:HaveAvatarOfFire` flag is set **only** by the Vulconus config
checkbox. Having the Avatar of Fire *keystone* (from Xoph's Blood or anywhere
else) does not set that condition. So with the box unchecked, PoB wrongly
grants the +2000 Armour and withholds the crit/conversion bonuses, even though
the character permanently has Avatar of Fire. The result: Vulconus's armour and
damage flip based on a checkbox that shouldn't matter.

### Affected Components

- `src/Modules/ConfigOptions.lua` — defines the Vulconus checkbox (`conditionHaveVulconus`).
- `src/Modules/ModParser.lua` — maps the "while you (do not) have avatar of fire" mod text to `Condition:HaveAvatarOfFire`, and parses Vulconus's implicit into `Condition:HaveVulconus`.
- `src/Modules/CalcPerform.lua` — merges keystone modifiers; the natural place to derive a keystone-based condition.
- `src/Data/Uniques/dagger.lua`, `src/Data/Uniques/amulet.lua` — item definitions (reference only).

---

## Reproduction Process

### Environment Setup

- **App:** Runs on Windows as-is (bundled SimpleGraphic runtime) — no build step needed to open the GUI and reproduce.
- **Tests:** Use `busted` on LuaJIT (`busted --lua=luajit`). No Lua toolchain was installed locally, so I relied on the fork's GitHub Actions CI to run the test.
- **Local setup (next time):** Install LuaJIT + LuaRocks, then `luarocks install busted`; run a single case with `busted --lua=luajit spec/System/TestDefence_spec.lua --filter "issue #3062"`.

### Steps to Reproduce

1. Create an empty build and equip **Vulconus** (Current variant).
2. Equip **Xoph's Blood** (grants the Avatar of Fire keystone permanently).
3. In Configuration, toggle **"Do you have Avatar of Fire?"**.
4. **Observed:** the character's Armour, crit chance, and phys→fire conversion
   change with the checkbox, even though Avatar of Fire is permanent from
   Xoph's Blood.

### Reproduction Evidence

- **Commit showing reproduction:** [[Link to commit in your fork]](https://github.com/ethannguyen128/PathOfBuilding/commit/46183b79)
- **Screenshots/logs:** 
- **My findings:**
-  - `Condition:HaveAvatarOfFire` is written in exactly one place:
  `ConfigOptions.lua:1640` (the Vulconus checkbox `apply` function).
- The Avatar of Fire **keystone** modlist (granted by Xoph's Blood, the tree,
  tattoos, etc.) performs the phys/other→fire conversion and "deal no non-fire
  damage", but it **never sets `Condition:HaveAvatarOfFire`**.
- Therefore any item whose text is gated on "while you (do not) have Avatar of
  Fire" (currently only Vulconus) reads the condition purely from the checkbox,
  ignoring permanent keystone sources. This is the root cause.

---

## Solution Approach

### Analysis

The "have Avatar of Fire" game state is conflated with the Vulconus checkbox.
The checkbox is the *only* writer of `Condition:HaveAvatarOfFire`, so the
condition is blind to a permanently-granted Avatar of Fire keystone. The
condition should instead be **derived from keystone presence** and be forced
true whenever the keystone is active, independent of the checkbox.

### Proposed Solution
Derive the "has Avatar of Fire" character state directly from keystone presence instead of relying solely on the Vulconus config checkbox.

Vulconus grants Avatar of Fire only temporarily, so PoB gates its "while you (do not) have Avatar of Fire" modifiers on a Condition:HaveAvatarOfFire flag that was set exclusively by the Vulconus config checkbox. That misses the case where a permanent source of the Avatar of Fire keystone is equipped (Xoph's Blood, or a passive-tree allocation): the character always has Avatar of Fire, yet the condition stayed unset, leaving Vulconus's +2000 Armour while you do not have Avatar of Fire line incorrectly active.

The fix sets Condition:HaveAvatarOfFire whenever the Avatar of Fire keystone is actually active, keyed off env.keystonesAdded immediately after keystones are merged in calcs.perform. This reuses the existing keystone-detection idiom (the same env.keystonesAdded table already used in CalcDefence/CalcOffence) and represents the state as a Condition: flag consistent with the rest of the codebase. The config-checkbox path is left untouched — it still models Vulconus's temporary buff, and since Vulconus never grants the keystone permanently on its own, the two paths compose without conflict.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** With a permanent Avatar of Fire source equipped, PoB must
treat the character as always having Avatar of Fire and ignore the
Vulconus-specific checkbox.

**Match:** Keystone merging already runs in calcs.perform
(CalcPerform.lua:1102-1104) and populates env.keystonesAdded. Other
keystone/condition wiring (e.g. Condition:Have... flags in ModParser.lua)
shows the established pattern of representing character state as a
Condition: flag. Defensive keystone interactions are tested in
spec/System/TestDefence_spec.lua (e.g. Iron Reflexes armour), which is the
model for the new test.

**Plan:** [Step-by-step implementation plan]
1. In src/Modules/CalcPerform.lua, after modLib.mergeKeystones, set Condition:HaveAvatarOfFire when env.keystonesAdded["Avatar of Fire"] is true.
2. Verify the Vulconus checkbox still functions when no permanent source is equipped (it grants the keystone, which now implies the condition).
3. Add a system test to spec/System/TestDefence_spec.lua covering: Vulconus + a permanent Avatar of Fire source → +2000 Armour line absent and Condition:HaveAvatarOfFire set. Use the correct falsy assertion idiom (assert.is_nil) for the baseline Flag check.
4. Run the full Busted suite to confirm no regressions before committing.

**Implement:* (https://github.com/ethannguyen128/PathOfBuilding)

**Review:** Follow CONTRIBUTING.md. Use a conventional commit message
(e.g. fix: derive Avatar of Fire condition from keystone (#3062)). Keep the
change minimal and localized; no ModCache.lua regeneration is required
because the fix lives in calc code, not in parsed item/mod text (verified: the
ModCache CI check only diffs re-generated parse output).

**Evaluate:** Reproduce the original steps in the GUI and confirm the checkbox no longer
changes Armour/crit/conversion when Xoph's Blood is equipped, and still works
for a Vulconus-only build.

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: With Vulconus equipped and the config checkbox unchecked, Condition:HaveAvatarOfFire is unset and the +2000 Armour while you do not have Avatar of Fire line is active (Armour >= 2000).
- [ ] Test case 2: Permanent keystone source: Equipping an item with the bare Avatar of Fire line (as Xoph's Blood grants) sets Condition:HaveAvatarOfFire and drops the conditional armour by exactly 2000.
- [ ] Test case 3No config-path regression: Verified in code that Vulconus never grants the keystone permanently on its own, so the checkbox behavior (temporary buff) is unchanged.

### Integration Tests

- [ ] Full Busted system-test suite run end-to-end (build load → item creation → calc pipeline → defence output): 238 passed / 0 failed / 0 errors.
- [ ]  The #3062 test exercises the real calc path (calcs.perform) via runCallback("OnFrame"), confirming the keystone→condition link is respected by downstream armour calculations.

### Manual Testing

Automated system tests only, the #3062 test reproduces the exact reported scenario (Vulconus + a permanent Avatar of Fire source) through the full calc pipeline, so no separate GUI pass was performed. Recommended manual spot-check before merge: load a build with Vulconus + Xoph's Blood in the GUI and confirm the Calcs-tab breakdown no longer lists the +2000 Armour while you do not have Avatar of Fire modifier.

---

## Implementation Notes

### Week [7] Progress

Reproduced issue #3062 with a failing system test in TestDefence_spec.lua. Root-caused the bug: Condition:HaveAvatarOfFire was only ever set by the Vulconus config checkbox (which models Vulconus's temporary buff), so a permanent source of the Avatar of Fire keystone (Xoph's Blood, tree allocation) never flipped the condition — leaving +2000 Armour while you do not have Avatar of Fire incorrectly active.

### Code Changes

- **Files modified:** src/Modules/CalcPerform.lua — force Condition:HaveAvatarOfFire when the keystone is active, spec/System/TestDefence_spec.lua — corrected falsy assertion idiom
- **Key commits:** (https://github.com/PathOfBuildingCommunity/PathOfBuilding/commit/46183b79d52105e10e844d546970676590d9ff42)
https://github.com/PathOfBuildingCommunity/PathOfBuilding/commit/6ad1d1f18b0630cd2db49b96ef604199ca83d8c4 
- **Approach decisions:** Gated on env.keystonesAdded["Avatar of Fire"] (matches existing keystone-detection idiom) instead of adding a new mechanism.
Left the Vulconus config checkbox untouched, it still correctly models the temporary buff, and the two paths compose without conflict since Vulconus doesn't grant the keystone permanently.
Used a colon-style mod source ("Keystone:Avatar of Fire") consistent with existing "Tree:" / "Skill:" sources for clear attribution in tooltips.

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
