# 1.21 Multiplayer AFK Raid Farm Development Notes

## Participants

- Youmiel

## Development Motivation

After the single-player AFK raid farm was completed overall in 1.21, I wanted to try whether multiplayer rotation potion-drinking AFK could be integrated into the existing circuits, which could improve the resource return from building one tower.

## Concept (2024-08-10)

The single-player AFK raid tower already implements logic for having players drink potions first, wait sufficient time, then move players to precisely trigger raids. By extending the existing structure, it should be possible to increase the number of players providing Bad Omen; just need to integrate these players' AFK positions into one module to conveniently switch between single/multiplayer AFK.

### Difficulty: Operation Cycle

Testing revealed that the shortest cycle for single-player AFK continuous raid generation is 601gt, not 600gt. This number isn't a multiple of 20gt; after multiple cycles, "phase drift" occurs between the raid tower and raid spawn interval, causing raids to attempt spawning mobs while the platform is blocked, thereby reducing efficiency.

Perhaps should adopt 610gt or 620gt operation cycles to better align with raid spawn intervals.

Prime factorization of 610gt and 620gt:
- 610gt = 2 * 5 * 61 gt
- 620gt = 2 * 2 * 5 * 31 gt = 10 * 62 gt

Clearly, for comparator loop + counter combinations, 620gt is easier to make.


## Development Log (2024-08-11)

### Bad Omen Generation Module Design

Gathering players together easily causes item-stealing issues, so this design must maintain some distance between droppers while ensuring compactness. Using powder snow to slow down items dispensed by droppers.

![2024-08-11_03.07.53](img/multiplayer_raid_farm/2024-08-11_03.07.53.png)

![2024-08-11_03.08.28](img/multiplayer_raid_farm/2024-08-11_03.08.28.png)


## Development Log (2024-08-13)

Connected the Bad Omen generation module to the raid tower; drive signal taken directly from the clock driving the spawn platform. Limited to one chunk width, only added 11 AFK positions; actually adding more isn't a problem.

![Alt text](img/multiplayer_raid_farm/2024-08-14_02.49.13.png)

## Development Log (2024-09-09)

### Efficiency

Divided into 11-player AFK and 11+1 player AFK efficiency, i.e., the difference of whether the player in the kill chamber is AFK.

![11](./img/multiplayer_raid_farm/rate_1.png)

![11 + 1](./img/multiplayer_raid_farm/rate_2.png)
<br>
<br>
<br>
<br>

1.21 Multiplayer AFK Raid Farm Development Notes © 2024 Author: Youmiel, licensed under CC BY-NC-SA 4.0. To view a copy of this license, visit http://creativecommons.org/licenses/by-nc-sa/4.0/.
