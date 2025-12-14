# Analyzing Minecraft Dragon Fight Mechanics from Source Code (Part VI) — The Mysteriously Respawning Ender Dragon

*\*This article uses code decompiled from Minecraft 1.16.1 vanilla*


A few days ago, MIZUkiYuu shared a feature in OSTC Lab: after destroying all portal blocks in the End and re-entering the save, the Ender Dragon respawns. The respawned dragon drops 12000 experience upon defeat and regenerates the dragon egg. After several days of code digging, the conclusions are basically finalized, and now I can share the exploration process.

First, since this is dragon fight related content, let's head straight to `net.minecraft.world.level.dimension.end.EndDragonFight`. The first thing that caught my attention was `EndDragonFight.createNewDragon()`, which creates an Ender Dragon at End (0, 128, 0). This method is called by `EndDragonFight.setRespawnStage(DragonRespawnAnimation)` and `EndDragonFight.findOrCreateDragon()`. The former creates a dragon after the respawn animation ends, which is obviously not what we're looking for. The latter is called in `EndDragonFight.tick()` and has a dragon death check and existence check—this seems interesting.

![image1](./img/6/image1.png)

Then when I reproduced the dragon regeneration phenomenon, when the player entered the End, the backend output this:

![image2](./img/6/image2.png)

If you've run your own server, this message should be familiar. Here we've clearly already killed the Ender Dragon, yet the game determines the dragon isn't dead. Tracing this message, I found the `EndDragonFight.scanState()` method.

![image3](./img/6/image3.png)

The `EndDragonFight.scanState()` method does the following: determines whether this is the first dragon kill (more precisely, whether the dragon has been killed before), and determines whether the dragon was dead when the save was last exited, performing error correction. The determination of whether this is the first dragon kill depends on the result of `hasActiveExitPortal()`, so let's trace that code.

![image4](./img/6/image4.png)

The gist of this code is to traverse all End Portal/Gateway blocks in a 17×17 chunk area centered on the (0,0) chunk (see Part II for why gateways are included). If at least one block exists, it considers the dragon to have been killed before. Here's the problem: in normal gameplay, before the player's first dragon kill, there are indeed no portal blocks in the End (return portal not open, gateways not generated). But the special case now is that the player manually destroyed all portal/gateway blocks, so during the check, the game can't find any such blocks and therefore concludes the dragon hasn't been killed before.

Subsequently, the code `if(!this.previouslyKilled && this.dragonKilled) {this.dragonKilled = false;}` compares the two variables recording the dragon's death status. If the dragon hasn't been killed before, the game determines the dragon isn't dead.

Now returning to `EndDragonFight.tick()`, everything becomes clear: the game determines the dragon isn't dead, but there's no dragon in the End at this point, so 1200 ticks after the player enters the End, the game performs a self-check and directly spawns an Ender Dragon.

![image1](./img/6/image1.png)

**But two questions remain: why was there a boss bar initially when no dragon was present? And why does the dragon drop 12000 experience and generate a dragon egg?**

The part that displays the Ender Dragon boss bar is written in the first line of `EndDragonFight.tick()`: `this.dragonEvent.setVisible(!this.dragonKilled);`. It controls the boss bar visibility through the `dragonKilled` variable, which determines whether the dragon is dead, rather than checking for the dragon's existence.

The experience drop part is written in the `net.minecraft.world.entity.boss.enderdragon.EnderDragon.tickDeath()` method, which queries the `previouslyKilled` variable in the `EndDragonFight` class instance to determine whether this is the first dragon kill, then sets the experience drop amount accordingly. Since the dragon generation earlier already determined this was the first kill, the experience dropped is the first-kill amount.

![image5](./img/6/image5.png)

The dragon egg generation part is written in the `net.minecraft.world.level.dimension.end.EndDragonFight.setDragonKilled()` method, which also checks the `previouslyKilled` variable before deciding whether to generate the dragon egg. When generating the egg, it finds the highest block at (0,0) and spawns the dragon egg one block above it.

![image6](./img/6/image6.png)

Now the entire process is clear. Here's a unified summary:

First, the player manually destroys all End Portal/Gateway blocks within the End main island (within 17×17 chunks centered on the (0,0) chunk), then exits the save from any location;

Then the player reopens the save, enters the End, and triggers the dragon fight check. The game concludes the dragon hasn't been killed before, and also that the dragon isn't dead, beginning to execute dragon fight related code;

Because the dragon is determined not dead, the boss bar is displayed, but there's no dragon in the End at this point;

After 1200 ticks, the game doesn't detect an Ender Dragon, so it regenerates one. The loot from defeating it is the same as the first dragon kill.

Note that the End dragon fight check is triggered only once per save opening, or once per server startup for servers. This is because the `needStateScanning` variable is set to true during game initialization and set to false after executing the dragon fight check, with no other places modifying this variable's value.

![image7](./img/6/image7.png)

If you want to use this feature for an Ender Dragon experience farm, it might not be suitable for servers since you need to restart after each dragon kill, which isn't convenient for servers.

Additionally, based on the principles described above, you can extend other gameplay:

1. Only destroy the gateways then respawn the dragon. After the dragon respawns, re-enter the save and kill the dragon to get first-kill loot. This requires less work than destroying all portals and avoids the 1200 tick idle period after entering the End;

2. Before the first dragon kill, use cheats to generate End Portal/Gateway blocks on the main island, then re-enter the save. Killing the dragon will not yield first-kill loot.


<br>
<br>
<br>
<br>

"Analyzing Minecraft Dragon Fight Mechanics from Source Code (Part VI)" © 2020 Author: Youmiel, licensed under CC BY-NC-SA 4.0. To view a copy of this license, visit http://creativecommons.org/licenses/by-nc-sa/4.0/.
