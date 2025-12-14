Consolidating my scattered research. Some things I consider undefined features will be temporarily omitted to avoid being patched. Given Mojang's current update habits, even things that seem fine might be changed in some minor version.


## 2024-06-14 Update, MC 1.21

1. The drop condition for Ominous Bottles is <not in a raid (meaning no RaidID)> AND is a <raid captain> AND is a <pillager>. Vindicators cannot drop Ominous Bottles;

2. Not every raid (meaning the entire raid event including all waves) can drop Ominous Bottles. Within a wave, vindicators spawn before pillagers. If a wave contains vindicators, a pillager cannot possibly be captain. According to code definition, the base composition of each wave is as follows. Extra waves spawned by Bad Omen above level 1 are roughly the composition of the last wave <at current difficulty>, specifics don't matter for this topic:

![each_wave](./img/note/each_wave.png)

3. Each wave additionally spawns some vindicators and pillagers. Easy and Normal difficulty may spawn 0~1 vindicators (Easy difficulty has higher probability of 0), Hard difficulty may spawn 0~2 vindicators. So waves 1 and 3 from (2) with potential also don't necessarily spawn a pillager captain.

4. Raid migration mechanics unchanged.


Current testing with budget raid tower shows frequent situations where Ominous Bottles income doesn't cover consumption, making self-sustainability quite difficult. Possible solutions:

1. Specifically modify an outpost to obtain Ominous Bottles

2. Use raid mob AI that actively picks up banners to fill captain vacancy to replace vindicator captains with pillager captains. Since MC-247440 isn't fixed yet, difficulty further increases


## 2024-06-28 Update, MC 1.21

1. Ominous Bottles only droppable by pillagers confirmed as intended design (Work As Intended, WAI). Related bug report MC-270014 processed as WAI (https://bugs.mojang.com/browse/MC-270014). Additionally, 1.21 changelog states "Ominous Bottles can be found uncommonly in any Vaults, and are dropped by Raid Captains which are defeated outside a Raid," contradicting actual behavior, but Mojang doesn't seem to care much (https://bugs.mojang.com/browse/MC-273895).



## 2024-07-04 Update, MC 1.21

1. Bad Omen doesn't convert to Raid Omen when current position is within a level 5 raid's range.

2. Pillager captain dropped Ominous Bottle level is random within 1~5, equal probability for each level.


## 2024-07-09 Update, MC 1.21

1. Client long-press item use is controlled by two network packets:
   - ServerboundUseItemPacket: Notifies server when player starts using item
   - ServerboundPlayerActionPacket: When action field is RELEASE_USE_ITEM or SWAP_ITEM_WITH_OFFHAND, terminates player item use


## 2024-08-14 Update, MC 1.21

1. In MC 1.21, expected drops per raid wave is 180.765 (without ravager handling) or 192.1675 (with ravager handling), assuming Hard difficulty, raid level above 2, using Looting 3 and all mobs killed by player, not counting saddles and potions. Witch drops expected at 93.75/wave, redstone dust expected at 56.25/wave. Remaining content in table, data recalculated based on @何为氕氘氚's work before 1.21.

![Drops](img/note/drops.png)

2. Based on above data, 1.21 single player raid tower limit efficiency is 21691.8/h (no ravager handling) or 23060.1/h (with ravager handling, same below), emerald efficiency 7920/h or 9000/h, redstone efficiency 6750/h.


## 2024-12-05 Update, MC 1.21.4

Summarizing changes to raids after 1.21.2:

1. Raid spawn zone is still a ring, but ring radius and center offset relate to raid wave spawn timer (referred to as variable `t` below). Mechanism:
   1. `t` decreases by 1 each gt (countdown). When `t == 0`, spawn raid party, and `t` only resets to 300
   2. Get scaling factor from `t`: `s = 0.22 * t / 20 - 0.24`
   3. Ring radius is `32 * s`, center random offset from raid center on xz axes is `(0 ~ 2) * floor(s)` each

2. Each gt raid randomly selects spawn points on ring several times and checks if selected position is suitable;

3. If selected spawn position doesn't meet following conditions, reselect:
   1. Y difference from raid center greater than 96
   2. Selected position is village section and `t > 140`
   3. Current position can't spawn mobs and below isn't snow layer. For whether a block allows mob spawning, see [wiki](https://minecraft.wiki/w/Spawn#Spawn_conditions)
   4. ... (other unimportant conditions, e.g., chunk must be force loaded)

4. When no suitable spawn position found, each raid tries 8 times per gt (was 3 before 1.21.2). If `t == 0`, attempts become 20 × 6 (was 20 × 4 before 1.21.2);

5. Secondary conclusion: When current raid wave is removed from raid for non-death reasons, the next wave spawned (only exists when `t = 0`) will be on a ring with radius 7~8.

## 2024-12-29 Update, MC 1.21.4

Corrected spawn attempt count when `t = 0` above.

## 2025-04-01 Update, MC 1.21.5

Following is brief analysis of video [MC1.21.2~1.21.4 Simple Self-Sustaining Raid Tower](https://www.bilibili.com/video/BV1HHZhYGE7U/):

Pillager captain dropping Ominous Bottle only requires no raid within 96 blocks, regardless of whether captain is in a raid; while the AI behavior of picking up banner to fill captain vacancy requires raider to actually be in a raid.

Above features should apply since 1.21. For detailed analysis see: [Code Analysis of Special Raider Behavior in \[96, 112\) Range in 1.21.x](../2025-04__1-21_captain_replace/).


## 2025-08-02 Update, Supplementing Some (Former) Secret Features
### 2024-06-28 Update, MC 1.21

Ominous Bottle doesn't require player kill of pillager captain to drop. Fall damage/cramming/sourceless explosion all work.

### 2024-07-06 Update, MC 1.21

Bad Omen triggering into Raid Omen, and Raid Omen converting to Raid, can cause `ConcurrentModificationException` due to adding/removing from collection during iteration. Status effects after the gt where exception occurs won't update. (Credit: Nickid2018, QWERTY770)

### 2024-08-16 Update, MC 1.21.2 Snapshot

[MC-247440](https://bugs.mojang.com/browse/MC-247440) has been effectively fixed, but Mojang only marked [MC-195754](https://bugs.mojang.com/browse/MC-195754) as "Fixed," unclear if they forgot or what. Now `raidToLeaderMap` correctly updates when captain is removed from raid.

Thus, my former idea of using Nether portal to teleport raid captain away, letting raid members pick up banner to create captain storage can now be implemented (which also led to [BV1DL4y1E7nH](https://www.bilibili.com/video/BV1DL4y1E7nH)), perhaps could be improved into a better Ominous Bottle farm.

### 2025-07-24 Update, MC 1.21.5

`ConcurrentModificationException` location after 1.21.5 differs from 1.21 ~ 1.21.4, because Mojang modified some status effect system code. Current version, when player doesn't have Raid Omen but has Bad Omen, entering village section triggers `ConcurrentModificationException` causing Bad Omen removal to fail that time. In same situation, if player has Raid Omen, Bad Omen removes normally without triggering `ConcurrentModificationException.`

### 2025-08-01 Update, MC 1.21 ~ 1.21.4

1. In 1.21 ~ 1.21.4, `ConcurrentModificationException` besides being triggered by aforementioned raid-related status effects, can also be created by Absorption. Specific method is depleting Absorption health 1gt before Absorption effect ends. Next gt when Absorption effect updates, it removes twice (effect depleted + timer ended). During code execution, `HashMap::remove()` is called first then `Iterator::remove()`, latter throws `ConcurrentModificationException` during execution because collection structure was modified.

2. Status effects on mobs (and players) are stored in a `HashMap<Holder<MobEffect>, MobEffectInstance>`, but Mojang didn't implement `hashcode()` method for `Holder<MobEffect>`, and uses iterator to traverse this hashmap directly when calculating and updating status effects. Therefore, calculation order between a mob's various status effects isn't stable. Factors like game process restart, `HashMap` expansion (expansion nodes: effect count 12, 24, vanilla total effects not enough for next expansion) all change status effect calculation order. This feature also applies to versions after 1.21.5, unclear how to meaningfully exploit it.
