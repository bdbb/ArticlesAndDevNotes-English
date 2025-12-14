# 1.18+ Stacking Raid Farm Real Player Compatibility Test Results

For the classification of real player compatibility, see the other article.

This test only examines under what conditions these raid towers can reliably stack raids, not checking whether they spawn vexes, as that requirement takes too much time, unless vexes appeared during testing.

## List: Build Name + Author + Video Link

| **Build** | **Author** | **Video Link** | **Compatibility** | **Notes** |
|---|---|---|---|---|
| Tape Raid Tower V2 | anew_tape | https://www.bilibili.com/video/BV1N94y1z7Pc | 3. Allows some imprecision in player actions due to network latency | Would be perfect with armor stand replenishment and sweep detection to prevent signal inversion |
| Next-Gen Raid Farm Gen3 | CCS_Covenant | https://www.bilibili.com/video/BV1xB4y117sf | 3. Allows some imprecision in player actions due to network latency | Would be perfect with armor stand replenishment |
| 1.18+ Raid Farm 231k Emerald/h | DaveRooney | https://www.bilibili.com/video/BV1h84y1Q7zX | 1. Depends on fake player attack during Entity Update phase |  |
| Weak Loading Looting Stacking Raid Farm | GaRLic BrEd | https://www.bilibili.com/video/BV1Bu41127N7 | Hard to determine | Should be affected by the timing difference between real and fake player raid triggering |
| Chronos Raid Tower v3 | GaRLic BrEd | https://www.bilibili.com/video/BV1pF411P7nP | 1. Depends on fake player attack during Entity Update phase |  |
| Ultimate Stacking Raid Farm | GaRLic BrEd | https://www.bilibili.com/video/BV1zX4y117zk | Hard to determine | Should be affected by the timing difference between real and fake player raid triggering, piston moving player is in Block Entity phase, later than Entity Update phase |
| "Authentic Village Raid" 632k 10gt Stacking Raid Tower | MU_mushroom | https://www.bilibili.com/video/BV18C4y1r7Sx | No save available, appears to be "1. Depends on fake player attack during Entity Update phase" |  |
| Budget Raid Tower | Nash, ianxofour | https://www.bilibili.com/video/BV1Su411b7Nz | 2. Requires stable player behavior | Fake player AFK raid spawn rate is about 500/hour more than real player, because fake player attack during Entity Update phase bypasses raid recruitment |
| 20gt 446K Stacking Raid Farm | Nash/NashCasty | https://www.bilibili.com/video/BV1fN411W7Cj | 1. Depends on fake player attack during Entity Update phase |  |
| NuggTech Raid Tower | Nash/NashCasty | https://www.bilibili.com/video/BV1oX4y1x7yX | 1. Depends on fake player attack during Entity Update phase |  |
| NuggTech Raid Tower v2 | Nash/NashCasty | https://www.bilibili.com/video/BV1394y187zK | 37 villager version: 1. Depends on fake player attack during Entity Update phase <br>17 villager version: 1. Machine timing depends on fake player's raid trigger timing | Designer might not have realized static raid spawning can solve real player AFK issues |
| Universal Raid Tower | Nash/NashCasty | https://www.youtube.com/watch?v=yKnLcEiLci0 | 2. Requires stable player behavior | Fake player AFK raid spawn rate is about 150/hour more than real player, because fake player attack during Entity Update phase bypasses raid recruitment |
| 460K Raid Tower | qwert_JANG | https://www.bilibili.com/video/BV1tj411B7bi | 1. Depends on fake player attack during Entity Update phase | Wait, dude, how does your mob collection still get stuck? |
| High-Speed Raid Tower, Skyblock Compatible [413k/hour] | Scorpio天蝎君 | https://www.bilibili.com/video/BV1Ja411G7Ag | 1. Depends on fake player attack during Entity Update phase |  |
| 410k Firework Raid Farm | Youmiel | https://www.bilibili.com/video/BV1Hb4y1M7aQ | 3. Allows some imprecision in player actions due to network latency |  |
| 220k Worry-Free AFK Raid Farm | Youmiel | https://www.bilibili.com/video/BV1sN4y1s7DW | 3. Allows some imprecision in player actions due to network latency |  |
| 835k Emerald Farm | 何为氕氘氚 | https://www.bilibili.com/video/BV1cR4y1H7No | 1. Depends on fake player attack during Entity Update phase |  |
| 960k~967k Emerald Farm | 何为氕氘氚 | https://www.bilibili.com/video/BV1ha411J7b8 | 1. Depends on fake player attack during Entity Update phase |  |
| Stacking Raid Farm V6 | 何为氕氘氚 | https://www.bilibili.com/video/BV1TY4y1M7zt | 1. Depends on fake player attack during Entity Update phase |  |
| Stacking Raid Farm V7 | 何为氕氘氚 | https://www.bilibili.com/video/BV1bz4y1M7zM | 1. Machine timing depends on fake player's raid trigger timing |  |
| 5gt Stacking Raid Farm | 何为氕氘氚 | https://www.bilibili.com/video/BV1QT411q7UG | 1. Machine timing depends on fake player's raid trigger timing |  |
| Emerald Printer 2.0 Bubble Column Version | 黑山大叔 | https://www.bilibili.com/video/BV1f44y1x7vY | Hard to determine, but real player can stack | Works because clock is slow, distance between waves is greater than recruitment distance |
| High Version Raid Tower Modified (3.0) | 黑山大叔 | https://www.bilibili.com/video/BV1mP411J744 | Hard to determine, but real player can stack | Similar to 2.0 |
| Raid Tower Spawn Platform Traction Modified (3.5) | 黑山大叔 | https://www.bilibili.com/video/BV1RW4y1u7Mj | Hard to determine, but real player can stack | As mentioned earlier, using 28gt clock is actually because mob falling time is around 28gt, just enough to create sufficient distance |
| Raid Tower 4.0 | 黑山大叔 | https://www.bilibili.com/video/BV1WN411A7dM | 2. Requires stable player behavior | Works because clock is slow, distance between waves is greater than recruitment distance |
| 300k Printer, Modified from 小也睡醒了 | 黑山大叔 | https://www.bilibili.com/video/BV1bg4y127Ve | 1. Depends on fake player attack during Entity Update phase |  |
| Alpha Modified 400k Raid Tower | 黑山大叔, alpha-hhh | https://www.bilibili.com/video/BV16y4y1d7r2 | 1. Depends on fake player attack during Entity Update phase |  |
| 1.18+ 340k 4-Villager No-Vex Upward Migration Printer | 沫幽忧 | https://www.bilibili.com/video/BV1Gu411G79Y | No save available |  |
| No Vex No Snow 4-Villager 420k Printer V2 | 沫幽忧 | https://www.bilibili.com/video/BV1bu411G7tz | Hard to evaluate | Poor timing design, even following author's usage instructions easily spawns vexes |
| Sub-Era Stacking Raid Farm [430k/h] | 沫幽忧 | https://www.bilibili.com/video/BV1VX4y1x77z | No save available |  |
| Sub-Sub-Sub-Era Raid Farm [430k/h] | 沫幽忧 | https://www.bilibili.com/video/BV11m4y1p7ws | 2. Requires stable player behavior | Not using observer on tripwire might get to 3. |
| Raid-Monster Tower | 沫幽忧 | https://www.bilibili.com/video/BV1RG411f77y | No save available |  |
| Sub-Era Raid Flat Tower [260k/h] | 沫幽忧 | https://www.bilibili.com/video/BV1Su4y1e7g2 | 1. Machine timing depends on fake player's special properties | Piston on switch line got QC'd |
| 250k Installable Sub-Flat Raid Tower | 沫幽忧 | https://www.bilibili.com/video/BV1Vj41117fn | 1. Machine timing depends on fake player's special properties | Observer on tripwire, following T flip-flop will invert signal due to vex accidentally triggering tripwire |
| 350k Sub-Flat Raid Tower | 沫幽忧 | https://www.bilibili.com/video/BV1o34y1T7pL | No save available |  |
| 340k Cow Jet Tower | 沫幽忧 | https://www.bilibili.com/video/BV1V14y1C72X | No save available |  |
| Raid Farm ProMin [200k/h] | 沫幽忧 | https://www.bilibili.com/video/BV1Yp421Z7X6 | Hard to evaluate |  |
| Extremely Elegant Raid Tower | 丨半世丶浮殇丨 | https://www.bilibili.com/video/BV1Cw411s7ZV | 1. Depends on fake player attack during Entity Update phase |  |
| Extremely Elegant Raid Tower V2 | 丨半世丶浮殇丨 | https://www.bilibili.com/video/BV1Fw411W7nS | 2. Requires stable player behavior | Needs input side T flip-flop inversion prevention, automatic armor stand replenishment, and spawn-non-blocking platform |

--------------------------------

## Strange Phenomenon in 1.18+ Stacking Raid Farm Design: Fake Player Dependency

1.18+ stacking raid towers—remove carpet and lose half, require efficiency not to drop significantly and lose another half, the remaining usable ones are countable on fingers. Once you see a raid tower that's <u>the simplest downward migration chain + spawn platform without downward acceleration or no nearby villagers + player always in village section</u>, you can basically determine this is a raid tower only for carpet fake players, unless the tower's clock is slow enough.

Ask yourself: among the current "technical Minecraft" community, how many people default that fake players equal real players under ideal network conditions? But as the previous article stated, fake player behavior is completely different from real players, even real humans controlled by `/player` command aren't the same. It's precisely this default that makes many designers never think that real player compatibility needs verification. Those so-called "simple raid towers" mostly just removed parts needed for real player AFK but not for fake player AFK, then claim they saved materials. However, they inadvertently hid the most valuable part: doesn't support real players, must be used with carpet mod.

I believe many people have repeatedly emphasized to others "Paper isn't vanilla, Forge isn't vanilla, we should use Fabric," and know that the starting point of playing vanilla redstone is "my designs can be used by others even without any mods installed." Looking back at those raid towers only for carpet fake players, do they still live up to these words? If I haven't been clear enough, you can also read [Overly Long Dynamics EP003: mod VS Tech](https://www.bilibili.com/read/cv16970398)

## Conclusion

If you've read this far and still think it's natural that stacking raid towers can only use fake players, then allow me to conclude: You just want the convenience brought by litematica and fake players, and don't care about how you obtain that convenience. In that case, I suggest you try the following two options: use more convenient commands like `/give`; or add more interesting and convenient mods like Create, Industrial Craft, Applied Energistics, etc. Vanilla content is too scarce and production methods too primitive—it's not suitable for your gameplay.

<br>
<br>
<br>

1.18+ Stacking Raid Farm Real Player Compatibility Test Results © 2024 Author: Youmiel, licensed under CC BY-NC-SA 4.0. To view a copy of this license, visit http://creativecommons.org/licenses/by-nc-sa/4.0/.
