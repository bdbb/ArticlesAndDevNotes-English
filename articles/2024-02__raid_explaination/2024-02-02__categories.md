# Classification of Stacking Raid Tower Applicability for Real Players

Thanks to @何为氕氘氚 and @Nickid2018 for their help in writing this article.

## Terminology

- Real player/Real human: Players connected to the world through a client and network, including single-player game players
- Fake player/Bot: Players generated using carpet mod's `/player` command
- All `Network Update` phases mentioned in this article should actually be divided into `Network` and `Player Action` phases, but are unified into one phase for easier understanding:
  - `Network` phase handles most player entity logic, such as fall damage, water flow pushing, potion effects, etc.
  - `Player Action` phase handles player actions, such as attacking, placing blocks, using items, etc.

## Overview

Stacking raid towers can be classified into four categories based on their acceptance of player actions:
1. Requires player behavior during the `Entity Update` phase / Machine timing depends on fake player's special properties
2. Requires stable player behavior, demanding players with no network latency/no packet loss/no frame drops
3. Allows some imprecision in player actions due to network latency
4. Completely requires no player action

\* If a non-stacking raid tower still doesn't support real players, what kind of unspeakable thing is it?


Determination method:
- Does it require player action? If no, then `4.`
- Are player actions idealized?
  - If no, then `3.`
  - If yes, then `2.`
- Can it only use `/player` command for AFK? If yes, then `1.`


## Detailed Explanation

### 1. Requires player behavior during `Entity Update` phase / Machine timing depends on fake player's special properties

Normal player actions are executed during the `Network Update` phase, but fake players and `/player` commands process player behaviors that should be handled in the `Network Update` phase during the `Entity Update` phase instead (for introduction to game phases, see Fallen_Breath's article: ["Deep Dive into Minecraft #1 Game Flow"](https://www.bilibili.com/read/cv4122124)). This affects raid tower design in many aspects:

  - Without handling raid recruitment, only fake players and `/player` commands and similar damage methods between `Raid Calculation` and `Entity Update` phases can kill captains to obtain Bad Omen at the appropriate time. Detailed micro-timing analysis:
     1. `Raid Calculation`
        1. Remove raid mobs more than 112 blocks from raid center
        2. Spawn raid mobs
     2. `Entity Update`, processed in order of entity joining the dimension
        1. Fake player joined dimension before raid mobs, so fake player attack is processed first, killing captain, **obtaining Bad Omen**
        2. Just-spawned raid mobs are still in the raid, AI calculation runs, raid recruitment calculation pulls other living raid mobs back into the raid
     3. Followed by `Block Entity` and `Network Update` phases
  - Fake player entities (and any player controlled by `/player` command) interaction with other entities is affected by the order they joined the world (spawned or loaded). Many raid towers now use armor stands to detect player sweep attacks, which affects when armor stands pop up.
    - If the detection armor stand joined the world later than the fake player entity, the armor stand will pop up **in the same gt** as the fake player action
    - If the detection armor stand joined the world earlier than the fake player entity, the armor stand will pop up **1gt after** the fake player action
  - Because there's a `Block Entity` phase between `Entity Update` and `Network Update`, fake player actions can also affect some circuits containing block entities, such as [【MC Redstone】Pressure Plate that Distinguishes Players from Mobs @Joker_小狼](https://www.bilibili.com/video/BV1as411v7Xn). Depending on raid tower design, some circuits may have situations where only one of fake player or real player works as expected. <br>![EU,NU Discriminator](./img/EU,NU识别器.png)

Additionally, besides fake player attack/use actions, other fake player calculations are also in the `Entity Update` phase, such as potion effects. Raid triggering relies on Bad Omen effect detecting nearby village sections. When the execution method isn't sword sweep but other entities (like TNT, fireworks), fake players trigger raids 1gt later than real players. Micro-timing analysis:

- Fake player, single dimension:
  1. GT 0: `Entity Update` phase
      1. Fake player joined current dimension before TNT/firework, fake player calculated first, but no Bad Omen effect so nothing happens
      2. TNT/firework explodes, kills raid mob, fake player **obtains Bad Omen effect**
  2. GT 1: `Entity Update` phase
      1. Calculate fake player's potion effects, Bad Omen **triggers raid**
- Fake player, dual dimension. Overworld spawning, Nether execution:
  1. GT 0:
      1. Overworld - `Entity Update` phase: Fake player calculates, but no Bad Omen effect so nothing happens
      2. Nether - `Entity Update` phase: TNT/firework explodes, kills raid mob, fake player **obtains Bad Omen effect**
  2. GT 1:
      1. Overworld - `Entity Update` phase: Calculate fake player's potion effects, Bad Omen **triggers raid**
- Real player, single or dual dimension:
  1. GT 0:
      1. Some dimension - `Entity Update` phase: TNT/firework explodes, kills raid mob, player **obtains Bad Omen effect**
      2. `Network Update` phase: Calculate player's potion effects, Bad Omen **triggers raid**

Real players and fake players also differ in the relative order between processing actions and calculating potion effects. Real players calculate potion effects before processing actions, while fake players calculate potion effects after processing actions. Real humans controlled by `/player` command are a fusion of both: actions in `Entity Update` phase, potion effects calculated in `Network Update` phase. Micro-timing analysis for these three:

- Real player:
   1. GT 0: `Network Update` phase
      1. Calculate player's potion effects, but no Bad Omen effect so nothing happens
      2. Process player action, kill captain, **obtain Bad Omen effect**
   2. GT 1: `Network Update` phase
      1. Calculate player's potion effects, Bad Omen **triggers raid**
- Fake player:
   1. GT 0: `Entity Update` phase
      1. Process fake player action, kill captain, **obtain Bad Omen effect**
      1. Calculate fake player's potion effects, Bad Omen **triggers raid**
- Real human controlled by `/player` command:
   1. GT 0: `Entity Update` phase
      1. Process `/player` command action, kill captain, **obtain Bad Omen effect**
   2. GT 0: `Network Update` phase
      1. Calculate player's potion effects, Bad Omen **triggers raid**

Carpet mod's fake players were designed to calculate in `Entity Update` phase only because author gnembon thought it shouldn't cause problems, but this view is clearly problematic now: <br>![gnembon quote: fake players are like pigs](./img/假玩家是猪.jpg)

### 2. Requires stable player behavior

Some raid towers' spawn platforms use pistons to push/pull the floor. During piston movement, the platform has no positions where raid parties can spawn. This requires players to avoid certain time periods when triggering raids, otherwise many raids will attempt to spawn parties exactly during piston push/pull, causing raid interruption. In regular raid towers where players are within village sections, they immediately trigger raids upon obtaining Bad Omen effect, with no restriction on trigger timing. If player attack intervals deviate, the aforementioned spawn failures will occur.

In fact, using tweakeroo on local server/single player still can't achieve absolutely stable operations—this is determined by the inherent nature of `client-server` architecture. Using local server/single player availability as a criterion isn't appropriate, but we can examine whether the design itself idealizes player actions. For example: using an observer watching tripwire + T flip-flop when detecting sweeps with armor stands. This idealizes player attack intervals, assuming players won't swing again within 11-14gt after the first swing, without considering that observers can't detect again within 4gt after detecting a block state change.

### 3. Allows some imprecision in player actions due to network latency

This category is reached after solving problems in `Category 2`. This doesn't mean real player AFK efficiency necessarily equals fake player AFK efficiency. Since raid tower efficiency relates to raids triggered per unit time, and triggering raids requires players to first kill raid captains to get Bad Omen effect, when players miss attacks due to lag, raid tower efficiency decreases.

### 4. Completely requires no player action

Currently there are no raid towers that don't require player action, so why have this category? Can only say "stay tuned" `:P`

<br>
<br>
<br>

Classification of Stacking Raid Tower Applicability for Real Players © 2024 Author: Youmiel, licensed under CC BY-NC-SA 4.0. To view a copy of this license, visit http://creativecommons.org/licenses/by-nc-sa/4.0/.
