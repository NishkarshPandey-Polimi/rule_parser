# Why 6.txt Failed (and How We Fixed It)

## The Original Rule That Failed ❌

```
While a friendly TYRANIDS unit is within 6" of this model, ranged weapons
equipped by models in that unit have the [ASSAULT] and [LETHAL HITS] abilities.
```

**Error Message:**

```
No terminal matches 'a' in the current parser context, at line 1 col 126
 models in that unit have the [ASSAULT] and [LETHAL HITS] abilities.
                                        ^
Expected: ABILITY
```

---

## Two Problems

### Problem 1: Multiple Abilities ❌

```
have the [ASSAULT] and [LETHAL HITS] abilities
                  ^^^
              Parser chokes here
```

**Why it fails:**

- Parser expects: `have the [ABILITY] ability` (singular)
- We gave it: `have the [ASSAULT] and [LETHAL HITS] abilities` (multiple + "and")

The grammar has **no rule** for handling multiple abilities.

---

### Problem 2: TYRANIDS Keyword Filter ❌

```
a friendly TYRANIDS unit
           ^^^^^^^^
           This causes serialization errors
```

**What happens:**

1. ✅ Parser reads it fine
2. ✅ Creates IR successfully
3. ❌ **Serialization fails** - can't convert to output code

The code generator is missing a handler for `HasKeyword` when it's used inside conditions.

**Output looks like:**

```python
if owned_by(var6, Player::you) and <class 'rule_parser.dialect.HasKeyword'>
                                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                                   Should be actual code, not a class name!
```

---

## Working Version 1: Leadership Pattern ✅

```
While this model is leading a unit, ranged weapons equipped by models
in that unit have the [LETHAL HITS] ability.
```

**Why it works:**

- ✅ Single ability: `[LETHAL HITS]`
- ✅ No keyword filter: Just checks leadership
- ✅ All operations fully supported

**Generated code:**

```python
def evaluate_weapon_abilities(self_model, Unit self_unit, evaluated_model, Unit evaluated_unit):
    ref var4 = self_model.is_leading_unit()
    ref var5 = unit_of(self_model)
    ref var6 = var5.contain(evaluated_model)
    ref var7 = var6 and var4
    if var7:
        add_ability(evaluated_model, WeaponAbility(WeaponAbilityKind::lethal_hits, 0),
                    WeaponQualifierKind::ranged)
```

---

## Working Version 2: Distance Pattern ✅

```
While this model is within 6" of a friendly unit, ranged weapons equipped
by models in that unit have the [LETHAL HITS] ability.
```

**Why it works:**

- ✅ Single ability: `[LETHAL HITS]`
- ✅ No keyword filter: Just checks "friendly" (ownership), not "TYRANIDS"
- ✅ Distance check fully supported

**Generated code:**

```python
def evaluate_weapon_abilities(self_model, Unit self_unit, evaluated_model, Unit evaluated_unit):
    ref var4 = all_units()
    ref var5 = []
    for ref var6 in var4:
        if owned_by(var6, Player::you):
            var5.append(var4)

    for ref var7 in var5:
        ref var8 = self_model.is_within_range(var7, 6)
        ref var9 = var7.contain(evaluated_model)
        ref var10 = var9 and var8
        if var10:
            add_ability(evaluated_model, WeaponAbility(WeaponAbilityKind::lethal_hits, 0),
                        WeaponQualifierKind::ranged)
```

---

## Quick Comparison

| Feature       | Original (Failed)        | V1 (Passed)    | V2 (Passed)      |
| ------------- | ------------------------ | -------------- | ---------------- |
| **Abilities** | ASSAULT + LETHAL HITS ❌ | LETHAL HITS ✅ | LETHAL HITS ✅   |
| **Condition** | TYRANIDS keyword ❌      | Leadership ✅  | Distance only ✅ |
| **Works?**    | No                       | Yes            | Yes              |

---

## The Rules for Making Text Pass

### ✅ DO:

- Use **one ability** per rule: `have the [ABILITY] ability`
- Keep conditions simple: `"a friendly unit"`, `"this model is leading"`
- Use distance checks: `"within 6\""`

### ❌ DON'T:

- Use multiple abilities: `"[ABILITY1] and [ABILITY2]"`
- Filter by keywords in conditions: `"a friendly TYRANIDS unit"`

---

## Bottom Line

**Original failed because:**

1. Can't handle multiple abilities with "and"
2. Can't handle keyword filters in conditions

**Fix:** Simplify to one ability and remove keyword filtering.

If you need the TYRANIDS keyword or multiple abilities, you'd have to split it into separate rules or modify the parser code.
