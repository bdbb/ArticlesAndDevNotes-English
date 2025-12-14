# 1.21 Self-Sustaining Raid Farm Development Notes

## Participants

- Youmiel

Assistance:
- acaciachan
- Hydrogen_PDC

## Development Motivation

After 1.21's major changes to raid trigger mechanics, Ominous Bottle recovery rate in conventional raid towers became an issue. According to the [research notes](../../articles/2024-06__1-21_raid/mechanic_note.md), directly killing raid mobs leads to Ominous Bottle consumption exceeding income. This design attempts to utilize the captain replacement mechanic to replace vindicator captains with pillager captains.

## Concept (2024-09-18)

Structure for capturing captains can reference:
[【MC】Stacking Bad Omen Storage【Ajvej & Bread】](https://www.bilibili.com/video/BV1ve4y1Q7LT), capturing the first 3 mobs likely clears out vindicators.

The ideal solution is completing captain capture and killing (using TNT) within 96 blocks of raid center. Due to MC-247440, the captain can't be removed from the raid before death, otherwise captain replacement can't continue. Then figure out how to let remaining pillagers pick up the banner to become captain, then kill them to drop Ominous Bottles. Can use fall tube height difference to remove raid mobs; this requires moving the raid center up instead of the traditional downward.

Currently, content upcoming in 1.21.2 will modify raid party spawn point height difference to within 96 blocks, change raid mob refresh method, and fix MC-247440, so this tower should try to be compatible with 1.21.2 during design.

## Development Log (2024-09-18)

Completed upward migration chain design, completed spawn platform layout.

Need to quickly kill captain before next raid wave spawns; currently thought method is using minecarts to transport mobs next to TNT duplicator to blow up, while recovering minecarts. Because spawned current wave mobs are less than 96 blocks from raid center, next wave won't spawn until mobs pass through the fall tube.

TNT captain killing section needs to use sea pickles or candles to achieve sufficient one-shot explosive damage.

## Development Log (2024-09-19)

### Item Collection

Item collection uses face-to-face observers to save wiring space and self-recover after misalignment. Learned this from TIS dual-dimension slime kill chamber.

![collect](img/self-sustained_raid_farm/2024-09-20_11.32.43.png)

### Raid Triggering

This time's migration chain migrates raids upward; raid triggering needs to change from "push player down" to "push player up." Wiring is much simpler because no need to consider blocks above player affecting player pose, plus gravity helps.

![raid_trigger](img/self-sustained_raid_farm/raid_trigger.png)

### Wiring Colors

Too many flying wires; to prevent not understanding the circuit when modifying later, now using four different colored blocks for wiring: white, light gray, yellow, light blue.

### Timer and Basic Control Flow

Timer and control flow use the 602gt timer and control flow developed in [1.21 Raid Farm Development Notes](./raid_farm_dev_note.md).


## Development Log (2024-09-20)


### Issue 1: Cannot Pick Up Banner

Already met conditions for raid mobs to pick up banner, but they seem unable to pick it up.

![cannot_pick](./img/self-sustained_raid_farm/cannot_pick.png)

Cause analysis:
1. When comparing drops and items raid mobs can pick up, code uses `ItemStack.matches()`, which compares item count. Therefore banners with stack count not equal to 1 can't be picked up by raid mobs.
2. Drops need to be at positions raid mobs can pathfind to; specific mechanism unclear.

### Spawn Platform Improvements

![platform](./img/self-sustained_raid_farm/2024-09-20_22.45.46.png)

Has problem of taking away ravagers.

## Development Log (2024-09-21)

### Spawn Platform Modification .1

![platform](./img/self-sustained_raid_farm/2024-09-21_15.12.09.png)

Has problem of second wave minecart taking away new captain.

### Powder Snow Buffer

![platform](./img/self-sustained_raid_farm/2024-09-21_15.31.25.png)

Stacking blocks less than 0.2 blocks thick on powder snow still makes powder snow's buffer effective. Some mobs in kill chamber didn't take fall damage because of this, so frequently observed non-one-shot-kill situations.

### Spawn Platform Modification .2

![platform](./img/self-sustained_raid_farm/2024-09-21_17.26.11.png)

After all the modifications it still ended up back to Ajvej's spawn platform, just moved the previous banner re-drop to the rails.

### Timing Adjustment

Changed spawn platform operation cycle to 30gt. Adjusted timing in many places; too many to remember.


## Development Log (2024-09-22)

### Stability Testing

Currently discovered captain-catching minecarts deplete over time; guessing because Totems of Undying entering the loop system causes cars to be dispatched one extra, causing cart to be blown up before entering explosion chamber.

### Timing Optimization

Circuit has clock alignment logic; every 602gt resets the 60gt clock driving spawn platform once to adapt to the 2gt per cycle raid spawn phase offset. Initial timing was roughly estimated; often had raid mob spawns fail because platform was blocked; after readjusting timing, positioned raid first spawn moment at 6gt before first action of each spawn platform cycle, greatly increasing spawn success rate (85% -> 95%).


## Development Log (2024-09-23)

### Minecart Separation

Changed non-stackable item separation at minecart collection to minecart separation, avoiding interference from mixed-in Totems of Undying.

### Vex Suppressor

This tower body is very narrow, can't fit boat-suction structure, vex problem hard to solve. After various attempts (pre-damage, timing adjustment, using blue sheep turning red AI), switched to vex suppressor.


## Development Log (2024-10-06)

### Theoretical Efficiency

Real player AFK might disconnect, inconvenient for efficiency testing; this is theoretical value tested using fake player AFK with command block giving Bad Omen effect inserted in circuit:

![rate 602gt](./img/self-sustained_raid_farm/rate_602gt.png)

### Collection - Regular Items

Regular item packing uses pyra's design; the high-throughput item sorting at top was modified by me to a neater design.

![Collection Packing](./img/self-sustained_raid_farm//2024-10-06_23.55.09.png)

### Collection - Totem Sorting

Totem sorting consists of multiple components: stackable/non-stackable separation, iron axe/crossbow filter, potion separation, packer. Iron axe/crossbow filter design from acaciachan and commandLeo; other parts relatively simple, designed on-site.

![Totem Sorting](./img/self-sustained_raid_farm/2024-10-06_23.52.41.png)


### Collection - Empty Shulker Box Replenishment

Empty shulker box replenishment uses design jointly from Mangelious, 萌萌的小公举, viomm, and acaciachan; since box demand isn't high, only extracted the base part separately.

![Empty Box Replenishment](./img/self-sustained_raid_farm/2024-10-06_23.56.51.png)


### Collection - Experience Separation

Experience orbs occasionally fall into item collection water channel; separating them out might be useful. Controlling item speed can let items pass over gaps in water channel while letting experience fall out through gaps.

![Experience Separation](./img/self-sustained_raid_farm/2024-10-08_00.41.10.png)


## Development Log (2024-10-08)

### Collection - Single-Speed High-Throughput Sorting - Nine-to-One - Packing - AB Single Slice

Redstone dust and emeralds both have less than single hopper speed efficiency, but packing without crafting into blocks takes a lot of space, so I made the AB single slice collection shown to solve this need. High-throughput sorting and packer still inherited from previous collection system; nine-to-one crafting section uses mixed packing detection approach.

Left design verified to be extremely unstable; should not be used.

![Single-Speed High-Throughput Sorting - Nine-to-One - Packing](./img/self-sustained_raid_farm/2024-10-08_12.26.48.png)


## Development Log (2024-11-19)

### Collection - Bottle Sorting

Collection system used 5 sorting slices to collect 5 different types of Ominous Bottles; I think this wastes space. After asking for help, got this 5-sorting minecart sorter designed by acaciachan, occupies 2 wide water channel, works better with slow water channel.

![multi-item-sorter](./img/self-sustained_raid_farm/multi-item-sorter_acaciachan.png)

After this replacement, bottle collection originally taking 5 wide plus extra signal isolation space can be changed to 4+1 format (levels 2~5 as one group, level 1 sorted separately), reducing 3~4 blocks width. Since this minecart multi-item sorting structure is very simple, it can even be sandwiched between different collection slices, utilizing space originally reserved for signal isolation.

![storage](./img/self-sustained_raid_farm/2024-11-19_storage.png)


## Development Log (2024-12-05)

### 1.21.2+ Compatibility

Testing showed this tower isn't compatible with 1.21.2+; main issues are still 1.21.2+'s significantly changed raider refresh logic, and 1.21.2+ has updates to banner pickup logic; better to redesign.

### Possible 1.21.2 Improvements

1. Need to adapt new refresh mechanics
2. After captain is removed from raid (not death), other members can normally pick up banner, so captain replacement can be more efficient; Ominous Bottle farm can be improved accordingly


## Development Log (2024-12-10)

### Bottle Sorting Bug Fix

During test operation, discovered the design used in [Bottle Sorting](#collection---bottle-sorting) doesn't work properly in single-player worlds; after testing found this design doesn't work properly at certain specific coordinates.

After original author acaciachan's debugging, discovered that at certain moments, one face of hopper minecart collision box exactly coincides with block grid face. Due to floating point precision issues, hopper's entity selection box can't select such minecarts at some coordinates but can at others, so sometimes two hopper minecarts are in pickup range. As well known, when hoppers try to take items from multiple container minecarts, they randomly select one then check contents; if empty minecart is selected instead of the minecart that needs items taken, one item is missed.

![float precision](./img/self-sustained_raid_farm/2024-12-10_minecart.png)

Fix method is to adjust hopper minecart position to avoid its collision box exactly coinciding with block grid; acaciachan's fix is changing the block above the diagonal powered rail to waterlogged trapdoor.

![fix](./img/self-sustained_raid_farm/2024-12-10_fix.png)



<br>
<br>
<br>
<br>

1.21 Self-Sustaining Raid Farm Development Notes © 2024 Author: Youmiel, licensed under CC BY-NC-SA 4.0. To view a copy of this license, visit http://creativecommons.org/licenses/by-nc-sa/4.0/.
