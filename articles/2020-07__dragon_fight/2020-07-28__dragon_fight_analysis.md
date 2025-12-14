# Analyzing Minecraft Dragon Fight Mechanics from Source Code (Parts I to V)

*This article uses code decompiled from Minecraft 1.16.1 vanilla*


## Part I: Starting from the Dragon Cannon

The dragon cannon exploits a feature where the game only checks for Ender Dragon respawn conditions when a player places down a crystal, regardless of whether the crystal is placed on the altar. In the video, Fallen_Breath used pistons to push End Crystals to create respawn conditions, but since no detection was executed at that point, the dragon didn't respawn until a crystal was placed by a player.

![image1](./img/1~5/image1.png)

![image2](./img/1~5/image2.png)

The code controlling the respawn condition check is written in `minecraft\world\item\EndCrystalItem.java`, as shown below.

![image3](./img/1~5/image3.png)

When a player right-clicks with an End Crystal, the game checks whether the current state meets the placement conditions for an End Crystal—whether the selected block is obsidian or bedrock, and whether there are blocks or entities blocking above. After the placement check succeeds, the game creates the End Crystal entity, then performs the dragon respawn condition check (`tryRespawn` method).


## Part II: The Specific Process of Ender Dragon Respawn Condition Check

Note that the respawn condition check calls the `tryRespawn` method of the `EndDragonFight` instance `var14`, so let's look at the `EndDragonFight` class. The code is as follows:

![image4](./img/1~5/image4.png)

For now, we'll only discuss the content before `ArrayList var7 = Lists.newArrayList()`. This nested if statement handles 3 different cases for the dragon nest (i.e., the return portal):

1. level.dat records the dragon nest position—use that position directly;
2. level.dat doesn't record the nest position, but a nest-like structure can be found—use the block at a "specific position" in this structure as the nest position, and record it in level.dat;
3. level.dat doesn't record the nest position and no nest-like structure can be found—directly generate a new dragon nest.

Now let's explain the detailed mechanics of each case.

### (1) Dragon Nest Position Recorded in level.dat

The dragon nest position is stored in `Data.DragonFight.ExitPortalLocation`. Although since 1.9 the xz values of the nest position are always at (0,0), the file still records all three xyz coordinates. This means players can customize the nest position by modifying level.dat.

Also note that data in level.dat is read into memory when the save file is loaded. Mentions of "checking data in level.dat" or "writing data to level.dat" in this article are for convenience of expression—the program doesn't actually access the file for every operation.

### (2) Mechanism for Finding Nest-Like Structures

Finding the nest structure uses the `findExitPortal` method, with code as follows:

![image5](./img/1~5/image5.png)

In simple terms, this method does the following:

#### a) Marks all End Portal blocks within a 17×17 chunk area centered on End (0,0) chunk coordinates.

In practice, End Gateway blocks are also marked. Checking the code reveals that `TheEndPortalBlockEntity` is the parent class of `TheEndGatewayBlockEntity`, meaning the aforementioned code treats gateways the same as End portals.

![image6](./img/1~5/image6.png)

#### b) Uses these portal blocks as centers and compares surrounding structures with the `exitPortalPattern.find` method.

`exitPortalPattern` is of type `BlockPattern.BlockPatternMatch`, whose methods can be found in `\net\minecraft\world\level\block\state\pattern\BlockPattern.java`, which in turn calls the `matches` method.

![image7](./img/1~5/image7.png)

![image8](./img/1~5/image8.png)

So what does this code do? Simply put, it creates a Chebyshev circle centered on the input block coordinates with radius equal to the longest side of the structure to find minus one. Here the nest structure is 7×5×7, meaning the game compares blocks in a 13×13×13 cubic area centered on the portal block. The match method then rotates the structure to be compared in various ways before comparing again.

![image9](./img/1~5/image9.png)

Additionally, the bedrock portion of the nest structure is hardcoded directly in the `EndDragonFight` class constructor.

To increase reliability, I checked the code for summoning iron golems, which also uses the `find` method in the `BlockPattern.BlockPatternMatch` class to determine the summoning structure. The Wither works the same way.

![image10](./img/1~5/image10.png)

![image11](./img/1~5/image11.png)


For example, all patterns shown below (including structures after 90° horizontal rotation) can successfully summon an iron golem:

![image12](./img/1~5/image12.png)

#### c) After finding a nest-like structure, check whether the block at relative coordinate (3,3,3) is at the (0,0) coordinate. If so, write it to levels.dat as the portal position. In both upright and inverted cases, the selected block is at the gold block position shown. When the nest is placed sideways, the phenomenon is a bit hard to explain: the bedrock indicated by the arrow is at (0,0), but the portal coordinates written to the file are still for the block at the relative position of the gold block.

![image13](./img/1~5/image13.png)

![image14](./img/1~5/image14.png)

Additionally, if a sideways-placed nest contains End Portal blocks, it can cause a game error and throw a `NullPointerException`, reason unknown.

![image15](./img/1~5/image15.png)

![image16](./img/1~5/image16.png)


### (3) Mechanism for Generating the Dragon Nest

When level.dat doesn't record the nest position and no nest-like structure can be found, the `spawnExitPortal` method is called to generate a nest. This method checks again whether the nest position is recorded; if not, it searches top-down at the (0,0) coordinate column for the highest block excluding leaf blocks (and technically air blocks) to generate the nest, and records the position in level.dat. `var1` is a boolean variable controlling whether the generated nest includes portal blocks. Since this is for the respawn condition check and the dragon is by default killed, it generates a nest with portal blocks.

![image17](./img/1~5/image17.png)

Many of these explanations are hard to demonstrate directly in-game, but later sections will mention how to verify the above content through experiments.


## Part III: Code Explanation for Two-Crystal Dragon Respawn

This content is basically a math problem, and it uses many different classes. Jumping between these classes while reading was quite tedious, so I'll only present the final conclusions without detailed explanation of the intermediate calculations. Main code is attached at the end.

![image18](./img/1~5/image18.png)

Every time a player places a crystal, the program calls the `trySpawn` method to check whether the Ender Dragon respawn conditions are met. It checks in north-east-south-west order whether there is an End Crystal collision box overlapping with the four 1×1×1 regions marked by glass blocks. Considering the End Crystal's volume, the block range where crystals for respawning can be placed spans two layers total, marked with light blue glass. The red glass positions indicate where a crystal can cover two detection positions at once:

![image19](./img/1~5/image19.png)

Here's the source code:

`tryRespawn` method

![image20](./img/1~5/image20.png)

Partial definition of the `Direction` class

![image21](./img/1~5/image21.png)

`Plane` enum type in the `Direction` class

![image22](./img/1~5/image22.png)

`getEntitiesOfClass` method in `LevelChunk`

![image23](./img/1~5/image23.png)

`AABB` (likely due to incomplete deobfuscation) (*Now known to stand for Axis-Aligned Bounding Box, no need to point this out—note added during re-editing*) method, which outlines a block region

![image24](./img/1~5/image24.png)

Data fields and constructor of the `BlockPos` class

![image25](./img/1~5/image25.png)

`relative` method of `BlockPos`

![image26](./img/1~5/image26.png)

Data fields and constructor of `Vec3i`, parent class of `BlockPos`

![image27](./img/1~5/image27.png)

`relative` method of `Vec3i`

![image28](./img/1~5/image28.png)

## Part IV: Part of the Dragon Respawn Process

(Here comes my favorite part)

After the respawn check succeeds, the game calls the `respawnDragon` method to respawn the Ender Dragon, which mainly goes through the following processes:

1. Check conditions: if the dragon has been killed and is not currently being respawned, proceed;
2. Use the `findExitPortal` method and 4 nested for loops to find all dragon nests within 17×17 chunks centered on the (0,0) chunk;
3. Fill all bedrock and End Portal blocks within the 7×5×7 framework of found nests with End Stone;
4. Schedule the respawn animation, etc.;
5. After processing all nests, call `spawnExitPortal` to directly generate a nest without End Portal at the position recorded in `exitPortalLocation`. This won't search top-down for nest generation position again, because meeting the respawn conditions means crystals are placed at the required positions on the nest, confirming the nest exists.

![image29](./img/1~5/image29.png)

Since the respawn process also calls the `findExitPortal` method and subsequently replaces these positions with End Stone, we can observe the characteristics of the `findExitPortal` method by respawning the dragon. For example, we can build the bedrock structure of the nest ourselves, then add End Portal/Gateway blocks nearby to verify whether `findExitPortal` really uses these blocks to find nest positions.

![image30](./img/1~5/image30.png)

I initially entrusted this part of the research to 异形龙虾 (Alien Lobster), who made important contributions to this problem. Special thanks to him.


## Part V: Some Strange Phenomena

Now let's briefly explain the mechanisms behind some strange phenomena:

### Double End Portals / Controlling End Portal Generation Position

I recall seeing this situation several times. In the TIS Redstone Tech Exchange group (formerly TIS Folk Science group), 四维鱼2019 (4D Fish 2019) mentioned the End portal duplication phenomenon; CarrotLee's survival video ([BV11J411k7NB](https://www.bilibili.com/video/BV11J411k7NB)) also showed two End portals; the BCP server I occasionally visit also had a similar situation; and I recently heard that the TGIM server also had two dragon nests.

![image31](./img/1~5/image31.png)

![image32](./img/1~5/image32.png)

![image33](./img/1~5/image33.png)

Now through reading the source code, we can clearly explain this phenomenon: First, the dragon nest coordinates in the server's level.dat were accidentally lost (usually due to server software changes, like switching from Paper to vanilla without merging level.dat information). Then, a player placed a crystal in the End dimension for some reason (respawning dragon/PvP, etc.), triggering the dragon respawn attempt. Since most technical Minecraft servers build sand duplication machines after defeating the dragon, the bedrock structure of the nest gets destroyed. With no coordinates and the nest structure destroyed, it obviously falls into case 3 mentioned earlier, so the game directly generates a new dragon nest—this is why double End portals appear.

Additionally, when exitPortalLocation data is lost, you can customize the nest generation position. Refer to the mechanisms explained earlier—you can build a pillar with non-leaf blocks at (0,0) or build the bedrock structure of the nest. Using bedrock to build a horizontal nest can even shift the nest position away from (0,0). Since there's currently no universally available method to obtain bedrock items across all versions, and it's unclear under what circumstances level.dat data loss occurs, this idea currently has no practical survival applicability.

### Bedrock to End Stone Conversion

I originally discovered this feature in mid-May 2020 when randomly filling bedrock accidentally produced this phenomenon. You can see it in my video [BV1854y1D7cd](https://www.bilibili.com/video/BV1854y1D7cd). This applies the mechanism described in Part IV. Following the mechanism explanation above, it's not hard to understand. Note that this behavior causes significant lag—the test server at the time lagged for over 20 minutes.

Similarly, because bedrock cannot be obtained in large quantities, this End Stone conversion method has no survival applicability beyond being interesting. (Maybe it could be used for creative mode technical builds)

<br>
<br>
Work of Youmiel
<br>
<br>

"Analyzing Minecraft Dragon Fight Mechanics from Source Code (Parts I to V)" © 2020 Author: Youmiel, licensed under CC BY-NC-SA 4.0. To view a copy of this license, visit http://creativecommons.org/licenses/by-nc-sa/4.0/.
