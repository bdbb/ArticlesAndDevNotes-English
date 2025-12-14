# Totem Farm Development Notes

## Participants

- Youmiel

Assistance:

- Menggui233
- Qian_Ye_


## Development Motivation

1.21 doesn't yet have an adapted raid tower. Although witch drops (redstone, glowstone dust, etc.) can rely on witch towers, and emeralds can be obtained through villager trading, there's no source for totems. Additionally, developing a totem farm can serve as technical preparation for raid towers, hence this project was established.

## Concept (2024-07-08)

This design doesn't consider self-sustaining Ominous Bottles; it's built on the premise of having sufficient Ominous Bottle reserves. The entire tower needs to go through the following process: player uses Ominous Bottle - detect player using Ominous Bottle - wait 30s - migrate raid - teleport all mobs to the Nether - kill mobs - collect.

Because there's network latency between real clients and servers, after giving players Ominous Bottles, we need to actively detect Ominous Bottle consumption to determine the raid trigger moment. The detection method is having the player hold a stack of Ominous Bottles in their offhand, with nearby Ominous Bottle items and pressure plates detecting whether the player picks up items.

The 30s wait can directly use the long timer approach from bottle farms; the reset end can be used to solve the problem of multiple Ominous Bottle uses within 30s. The raid migration chain can directly use the single-villager migration chain from 40gt raid towers.

Since Totems of Undying don't require player kills to obtain, all mobs can be teleported directly to the Nether to fall to death. 1.21 allows passengers and vehicles to teleport across dimensions; just need to build Nether portals on the spawn platform.

Passengers can take fall damage while on vehicles, allowing direct fall-killing of riders without killing ravagers, shortening the fall tube length; just need to figure out how to separate ravagers afterward.


## Development Log (2024-07-09)

### Spawn Platform

The spawn platform uses two columns totaling 10 spawn positions, using boats to push mobs into Nether portals.

### Kill Chamber

Ravagers have 100 HP, requiring over 100 blocks of distance to fall-kill. Nether-side processing obviously can't accept too tall a tube—first, survival time is too long; second, breaking bedrock for one mob kill isn't worth it.

The fall structure referenced the design from slime farms using spaced trapdoors to separate large and small slimes. An additional sparser trapdoor layer was added above to push away surviving ravagers.

### Method of Controlling Player Ominous Bottle Use

When players press the use key, they'll prioritize interacting with blocks/entities, and only lastly use the item in hand. Therefore, you can place note blocks, weighted chests, armor stands, etc. in front of players to block the use action.


## Development Log (2024-07-10)

### Kill Chamber Improvements

Testing the kill chamber revealed that ravagers with riders can't be pushed by other entities, so the Nether-side entity ejection design was modified. Added a set of boats and pistons; boats push out regular raid mobs, pistons pull away ravagers with riders.

### Timer

Using the pressure plate's falling edge signal to determine when players used Ominous Bottles; weighted pressure plate detection interval is 10gt. Assuming 600gt intermediate delay, the most optimistic case is migrating the raid 601gt after player uses Ominous Bottle, worst case is 610gt. Subtracting the 32gt needed for players to use items, the remaining 568gt needs delay circuit handling; circuit directly borrowed from the 40-minute timer in Ominous Bottle farms.

After factoring, near 568gt, 572, 567, 560gt are relatively suitable as long timer parameters: 572 = 4 * 11 * 13, 567 = 9 * 9 * 7, 560 = 8 * 5 * 14 = 7 * 8 * 10. The actually used parameter is 40 * 14 = 560, leaving 40gt margin for player item use and circuit delay.

Controller timing sequence: Detect player picking up Ominous Bottle - (reset timer - dispense new bottle - detect if bottle is picked up by player, if yes loop) - timer completes - repeatedly block line of sight between player and note block to start next bottle - migrate raid 600gt after timer completes.

### Method of Controlling Player Ominous Bottle Use (Supplement)

Fake player long-press use can't be blocked by armor stands; needs to be changed to note blocks.

2024-07-28 supplement: Initially I thought fake players would ignore armor stands and interact directly with blocks behind them; actually fake players continue processing "use item in hand" logic when interaction with armor stand fails, unlike vanilla which directly skips.


## Development Log (2024-07-11)

### Item Collection

Testing showed farm efficiency as follows:

![rate](./img/totem_farm/rate.png)

Valuable products are redstone dust, gunpowder, glowstone dust, and Totems of Undying. Using high-throughput sorting to pack the first three, Totems of Undying and saddles are directly packed after non-stackable separation; modules are still ready-made.

Additionally, Ominous Bottle efficiency in the output is 77/h, raid trigger rate is about 120/h, proving the conclusion "expected Ominous Bottle output per raid wave is 2/3" is correct.



<br>
<br>
<br>
<br>

Totem Farm Development Notes © 2024 Author: Youmiel, licensed under CC BY-NC-SA 4.0. To view a copy of this license, visit http://creativecommons.org/licenses/by-nc-sa/4.0/.
