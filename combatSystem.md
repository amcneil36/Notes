# Majesty: The Fantasy Kingdom Sim - Combat System

## Overview

Combat in **Majesty: The Fantasy Kingdom Sim** follows a three-phase system:

1. **Chance to Hit**
2. **Damage Calculation**
3. **Damage Reduction**

The system uses percentage rolls (1-100) to determine combat outcomes.

---

# Phase 1: Chance to Hit

## Melee Attacks (Heroes vs Heroes/Monsters)

### Step 1: Attacker Accuracy Roll

The attacker rolls against their **Hand-to-Hand** skill.

| Result | Outcome |
|---|---|
| Roll ≤ Hand-to-Hand | Attack proceeds |
| Roll > Hand-to-Hand | Attack misses |

### Step 2: Defender Parry Roll

If the attack connects, the defender rolls against **Parry**.

| Result | Outcome |
|---|---|
| Roll ≤ Parry | Attack completely avoided |
| Roll > Parry | Attack hits |

### Example

**Warrior attacks Goblin**

- Warrior Hand-to-Hand: 70
- Goblin Parry: 40

Rolls:

1. Warrior rolls 65  
   - 65 ≤ 70 → Attack proceeds
2. Goblin rolls 35  
   - 35 ≤ 40 → Parried

**Result:** No damage dealt.

---

# Ranged Attacks (Heroes vs Heroes/Monsters)

## Step 1: Attacker Accuracy Roll

The attacker rolls against their **Ranged** skill.

| Result | Outcome |
|---|---|
| Roll ≤ Ranged | Attack proceeds |
| Roll > Ranged | Attack misses |

## Step 2: Defender Dodge Roll

If the shot connects, the defender rolls against **Dodge**.

| Result | Outcome |
|---|---|
| Roll ≤ Dodge | Attack avoided |
| Roll > Dodge | Attack hits |

### Example

**Ranger attacks Vampire**

- Ranger Ranged: 75
- Vampire Dodge: 50

Rolls:

1. Ranger rolls 60  
   - Hit
2. Vampire rolls 55  
   - Dodge fails

**Result:** Damage calculation begins.

---

# Attacks Against Buildings and Lairs

Attacks against buildings and lairs:

- Automatically hit
- Require no accuracy roll

---

# Spell Attacks

Magic attacks automatically hit.

The defender rolls against **Magic Resistance**.

| Result | Outcome |
|---|---|
| Roll ≤ Magic Resistance | Spell resisted |
| Roll > Magic Resistance | Spell hits |

### Example

Wizard casts Fire Blast on Hell Bear:

- Hell Bear Magic Resistance: 30
- Roll: 45

45 > 30

**Result:** Spell hits.

---

# Phase 2: Initial Damage Calculation

# Hero Melee and Ranged Damage

Formula:
