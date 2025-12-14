# Analyzing Minecraft Dragon Fight Mechanics from Source Code (Part VII) — More Features

*\*This article uses code decompiled from Minecraft 1.16.1 vanilla*

Before starting, here's a supplement to the previous article:

When analyzing the mysterious Ender Dragon resurrection phenomenon, I forgot to write about the constructors of `ServerLevel` and `EndDragonFight` (if you don't know what a constructor is, you can simply think of it as initialization work before the dragon fight mechanism starts running).

![image1](./img/7/image1.png)
![image2](./img/7/image2.png)

The constructor mainly does three things in sequence:

- Initializes the boss event for the End dragon fight (boss bar color, boss fog, etc.);
- Reads dragon fight information from file (i.e., level.dat): nest position, dragon alive status, dragon's UUID, list of unopened gateways. And handles exceptional situations;
- Constructs nest structure information through hardcoding, used for generating and finding nests.

However, regardless of how the constructor reads file information, the subsequent misjudgment caused by `scanState()` can still overwrite the dragon's alive status.

---------------------------------------------------------------------------------------------------------------------------------

When I re-read the code while analyzing the feature discovered by MIZUkiYuu, I found some content I had previously overlooked, so here's a supplement.

First, let's parse the flow of the `EndDragonFight.tick()` method. This method is called in `ServerLevel.tick()` and controls the entire End dragon fight process. The general flow is:
- Every 20 ticks, scan the list of players in the End
- If there are players in the End:
    - Add a "dragon" type loading ticket to the End (0,0) chunk (mechanism newly added in 1.16);
    - Ender Dragon alive status check, once and only once per save opening;
    - Monitor dragon respawn progress / advance respawn process;
    - Various checks during the dragon fight:
        - Check dragon existence every 1200 ticks;
        - Check remaining crystal count every 100 ticks;
- If no players in the End, remove the "dragon" loading ticket;

![image3](./img/7/image3.png)

First, about loading tickets. ~~The 1.16 "dragon" type loading ticket is added when a player enters the End, which differs from the previous requirement of needing an active dragon fight (see [cv3395631](https://www.bilibili.com/read/cv3395631)). Even if the player leaves the main island through a gateway, the main island remains loaded due to the dragon loading ticket. This feature can be understood as the End's "spawn chunks."~~ The "dragon" type loading ticket only requires a player to be within the spherical range, which differs from what [cv3395631](https://www.bilibili.com/read/cv3395631) said about needing an active dragon fight (I don't know why, but I trust my conclusion is correct). Previously people often joked that End concrete curing machines without Carpet mod fake players could summon a dragon for loading, but now it only requires someone to be in the End dimension.

Additionally, the dragon existence check every 1200 ticks can not only mysteriously resurrect the Ender Dragon but also create situations with N dragons existing simultaneously. The method is to move the dragon to an unloaded chunk so the game can't find it when searching. Then another dragon spawns at (0,128,0). Repeat as needed.

The subsequent various checks don't seem very interesting so I won't go into detail. It's just these two lines:

```java
this.level.getChunkSource().addRegionTicket(TicketType.DRAGON, new ChunkPos(0, 0), 9, Unit.INSTANCE);
boolean var1 = this.isArenaLoaded();
```

After adding the loading ticket, why does it check whether the 17×17 chunk area centered on the main island (0,0) chunk is loaded...?

Then there's the `EndDragonFight.scanState()` mentioned in Part VI, but this time we're focusing on the middle portion:

![image4](./img/7/image4.png)

The middle portion searches the entire dimension for all dragons and creates a list. When no dragon is found, it determines the dragon has been killed. After finding a dragon, if the nest doesn't have a return portal (actually meaning no End Portal/Gateway blocks exist within 17×17 chunks centered on the (0,0) chunk, as emphasized multiple times before), it deletes the first dragon in the recorded dragon list. In actual experiments, the dragon is deleted and then regenerates at (0,128,0), giving the player the impression that the dragon's health and position were reset. Generally, this phenomenon only occurs during the first dragon fight, but such dragon deletion behavior is quite puzzling. A possible explanation is that the game designers intentionally increased the difficulty of the first dragon fight (though it's trivial for experienced players).

-----

Combining the various features described above, here's a **rough idea**: a special Ender Dragon experience farm.

First, destroy all End Portal/Gateway blocks near the main island, then re-enter the save. Use [**some method**] to lead the respawned dragon to an unloaded chunk and trap it in [**some stable unload-resistant Ender Dragon cage**]. Wait for the game to spawn a new dragon, and repeat to "cache" several dragons. Once enough are cached, kill the last dragon to create a return portal and restore dimension travel.

When experience is needed, first destroy all End Portal/Gateway blocks near the main island, then travel to the [**Ender Dragon cage**], re-enter the save. At this point one dragon should be deleted while a new dragon spawns at End (0,128,0). If there's [**a fast method to kill dragons**], obtaining large amounts of experience shouldn't be too difficult. For multiple extractions, you can remove nest portal blocks by re-entering the save during the dragon respawn process. However, this method requires all gateways to be opened and destroyed, meaning travel to the End outer islands would need to be via elytra or End teleportation.


<br>
<br>
<br>
<br>

"Analyzing Minecraft Dragon Fight Mechanics from Source Code (Part VII)" © 2020 Author: Youmiel, licensed under CC BY-NC-SA 4.0. To view a copy of this license, visit http://creativecommons.org/licenses/by-nc-sa/4.0/.
