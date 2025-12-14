# 1.21 Raid Farm Development Notes

## Participants

- Youmiel

Assistance:
- Nickid2018
- sfy_dr
- MizukiYuu

## Development Motivation

1.21 changed the raid trigger method from direct triggering to entering a village after using an ominous potion. This change makes previous raid tower farms no longer usable, requiring development of new farms.

## Concept (2024-07-12)

This design doesn't consider self-sustaining Ominous Bottles; it's built on the premise of having sufficient Ominous Bottle reserves. In the concept, a single farm operation cycle needs to complete the following actions: player uses Ominous Bottle - detect player using Ominous Bottle - push player to trigger raid at precise moment - calibrate tower operation cycle - drive tower mechanical structure at 20gt period - wait 30s - migrate raid - wait for mobs to spawn - kill mobs - collect.

\* The significance of triggering raids at precise moments is to facilitate aligning the tower's mechanical parts' operation cycle (e.g., pre-damage/centering)

Because detection and trigger-use-item circuits need to be arranged near the player, placing mob killing below the player is more convenient; can use the kill chamber from "Worry-Free AFK Raid Farm."

### Whether to Avoid Raid Recruitment

This depends on whether Ominous Bottles need to be recovered. Since we don't need to consider Ominous Bottle recovery rate now, raid recruitment can also be ignored.

### How to Support Real/Fake Players

Currently, real players and fake players differ in the following ways:

1. Game phases where actions are executed differ; currently doesn't appear to affect this farm design;
2. Real players have network latency, affecting attack, active movement, and potion use timing; design should note this;
3. Real player right-click item use can be blocked by armor stands, while fake players click through armor stands to blocks behind.

### How to Prevent Vexes

Currently there are two relatively reliable anti-vex methods:

1. Vex suppressor
2. After fall-damaging raiders, immediately use boats to push them into blocks to block line of sight, preventing witches from throwing potions

### How to Precisely Trigger Raids

Precise raid triggering technology was researched in the previous experimental "Worry-Free AFK Raid Farm" and can be directly used.


## Development Log (2024-07-15)

### Choosing Blocks to Block Right-Click Use

AFK players are positioned between two blocks, making iron trapdoors inconvenient for blocking right-clicks; need something that can be toggled by redstone and has no collision volume. Currently found option is powder snow, which can use pistons or dispensers. Using dispensers to place powder snow can't determine placement state, so using pistons is better.


## Development Log (2024-07-27)

Completed wiring for the bottle detection section. After careful thought, realized single-player operation actually doesn't need to AND the clock signal with the player bottle-use signal, because the 30s countdown time is enough for the previous wave of raids to finish spawning.

![2024-07-27](./img/raid_farm/2024-07-27_01.40.37.png)

### Boat-Suction Structure Repair

24w03a (1.20.5 snapshot) modified the judgment height for "entity's line of sight is in liquid," now 1/9 block above line of sight (credit: Nickid2018), causing the original boat-suction structure to no longer work. Now boats in this structure need to be lowered 1/9 block to continue working; the method is changing campfire (7 pixels high) to large amethyst bud (5 pixels high).

Note: When mobs dismount boats with no landing position, foot position will be at the center of the boat's collision box upper surface. So even after lowering 2 pixels, drops (4 pixels high) can still touch the water channel for collection. But if large amethyst bud is changed to trapdoor (3 pixels high), items will get stuck on boats because they can't touch water flow.


## Development Log (2024-07-28)

### Wiring

Removed the AND logic mentioned the previous day, completed 30s timer and player line-of-sight blocking section.

![2024-07-28_1](./img/raid_farm/2024-07-28_02.00.38.png)

Designed two 20gt comparator loops for driving the platform.

![2024-07-28_2](./img/raid_farm/2024-07-28_02.56.44.png)

### Fake Players

Discovered that fake players continue processing "use item in hand" logic when interaction with armor stand fails, unlike vanilla which directly skips. Therefore can't have fake players long-press use key and periodically attack armor stands to AFK this raid tower.


## Development Log (2024-07-29)

### Ghost Items

When player inventory is full and holding a stack of food, eating food while picking up same items from ground causes client-server desync. Client-side player item count doesn't increase from picking up items after decreasing. This issue prevents raid tower bottle replenishment from working.

No good way to update player held item count, so abandoned current detection-based approach and refactored to clock-based.


### Refactored Process

Designed as a 608gt clock-driven raid tower. Since 608gt = 8 * 76gt, still using comparator clock plus counter for long clock.

On startup: Toggle switch - reset clock - delayed start (for preparation) - calibrate spawn platform

Each cycle (each 76gt divided into a phase, can be handled exactly through signal level in latch):

1. Player starts using Ominous Bottle;
2. Push player into village section, replenish Ominous Bottle, previous cycle's raid spawns, start migration chain;
3. --
4. --
5. --
6. --
7. --
8. --

## Development Log (2024-07-29)

### Refactored Wiring

Completed refactoring, next test efficiency and stability.

Because this raid tower requires vanilla-behaving client to run correctly, invited a friend (sfy_dr) to AFK the raid tower.

## Development Log (2024-09-08)

### Modified Clock

When raid tower was driven at 20gt period, stability was poor; long-term AFK always spawned vexes, so changed to 40gt period. Comparator clock design from MizukiYuu:

![40gt Comparator Fader](<./img/raid_farm/40gt Comparator Fader by MizukiYuu.png>)

Wanted to maximize efficiency, so still chose 602gt raid generation clock.

### Efficiency

Tested over 800 minutes with real client; efficiency and stability were thoroughly tested.

![rate](./img/raid_farm/rate.png)



<br>
<br>
<br>
<br>

1.21 Raid Farm Development Notes © 2024 Author: Youmiel, licensed under CC BY-NC-SA 4.0. To view a copy of this license, visit http://creativecommons.org/licenses/by-nc-sa/4.0/.
