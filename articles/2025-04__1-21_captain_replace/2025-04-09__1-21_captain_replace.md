# Code Analysis of Special Raider Behavior in \[96, 112\) Range in 1.21.x

This article provides code analysis for [BV1HHZhYGE7U](https://www.bilibili.com/video/BV1HHZhYGE7U). The video author discovered that when raiders are 96 to 112 blocks from the raid, pillager captains can drop Ominous Bottles, and raid party members can pick up ominous banners.


## 0. Environment Setup

If you don't know where to obtain the source code but wish to follow along with this article, the following may help. To avoid potential licensing risks, this article uses the Yarn deobfuscation mapping maintained by FabricMC. You can obtain decompiled game source code using Yarn through two methods:

1. Use [FabricMC/yarn](https://github.com/FabricMC/yarn), either by downloading as a package or using `git clone` for the corresponding branch. After ensuring you have a suitable Java version in your system environment variables, execute `./gradlew decompileCFR` in the project root directory. The decompiled results will be in `build/namedSrc/`.

2. (Requires Git and a Java IDE like IntelliJ IDEA) Use Git to clone [FabricMC/fabric-example-mod](https://github.com/FabricMC/fabric-example-mod) locally, then open the project with the IDE. Switch to the commit for the game version you need, then follow the instructions in [Setting up a Development Environment | Fabric Docs](https://docs.fabricmc.net/develop/getting-started/setting-up-a-development-environment) to configure the project. You can find Minecraft in the dependencies, which is the decompiled game code.

Some game logic doesn't exist as code but as built-in resource packs and data packs. You can find these by opening `versions/[game version]/[game version].jar` in the game directory with a compression file manager; `assets/` is the resource pack directory and `data/` is the data pack directory.

This article's explanation is based on Minecraft version 1.21.


## I. Prerequisite Knowledge - Code Meaning of Raiders Being "In a Raid"

We often say raiders are "in a raid," but in code, raid and raider objects each store references to each other. "In a raid" depending on context may mean "the raid contains a reference to the raider" or "the raider contains a reference to its belonging raid."

```java
public class Raid {
    // Raid contains mappings of each wave's raiders and raid captains
    private final Map<Integer, RaiderEntity> waveToCaptain;
    private final Map<Integer, Set<RaiderEntity>> waveToRaiders;
    /* ... */
}
```

```java
public abstract class RaiderEntity extends PatrolEntity {
    @Nullable
    protected Raid raid;  // The raid this raider belongs to
    /* ... */
}
```

In `RaiderEntity`, there are several methods that check raid existence:

```java
// Get the raid instance this raider belongs to
@Nullable
public Raid getRaid() {
    return this.raid;
}

public boolean hasRaid() {
    World world = this.getWorld();
    if (!(world instanceof ServerWorld)) {
        return false;
    }
    ServerWorld lv = (ServerWorld)world;
    return this.getRaid() != null || lv.getRaidAt(this.getBlockPos()) != null;
    // Criterion for raider being in a raid: raid reference is not null OR no raid exists within 96 blocks of current position
    // lv.getRaidAt() - finds a raid within 96 blocks of the specified coordinates in the dimension (depends on HashMap traversal order)
}

public boolean hasActiveRaid() {
    return this.getRaid() != null && this.getRaid().isActive();
    // Check if raider belongs to an active raid
    // this.getRaid().isActive() - raid is in ONGOING state and raid center position is loaded
}

@Override
public boolean hasNoRaid() {
    return !this.hasActiveRaid();
    // Check if raider is not in a raid (inverse of previous check)
}
```

Another way of saying "the raider contains a reference to its belonging raid" is "the raider has a RaidId," because in the entity data serialization to NBT code, the key `RaidId` shows the raid's numeric ID. This can be viewed using the `/data` command and is a convenient inspection method in practice.

```java
@Override
public void writeCustomDataToNbt(NbtCompound nbt) {
    super.writeCustomDataToNbt(nbt);
    nbt.putInt("Wave", this.wave);
    nbt.putBoolean("CanJoinRaid", this.ableToJoinRaid);
    if (this.raid != null) {
        nbt.putInt("RaidId", this.raid.getRaidId());
    }
}
```


## II. Prerequisite Knowledge - When Is a Raider Considered a "Captain"

`RaiderEntity` extends `PatrolEntity`, and both have similar "check if captain" methods:

```java
public abstract class PatrolEntity extends HostileEntity {
    /* ... */
    private boolean patrolLeader;
    /* ... */
    // Whether this is a "patrol leader"
    public boolean isPatrolLeader() {
        return this.patrolLeader;
    }
    /* ... */

    // this.patrolLeader is saved in NBT under the key PatrolLeader
    public void writeCustomDataToNbt(NbtCompound nbt) {
        super.writeCustomDataToNbt(nbt);
        if (this.patrolTarget != null) {
            nbt.put("patrol_target", NbtHelper.fromBlockPos(this.patrolTarget));
        }
        nbt.putBoolean("PatrolLeader", this.patrolLeader);
        nbt.putBoolean("Patrolling", this.patrolling);
    }
}
```

```java
public abstract class RaiderEntity extends PatrolEntity {
    // Whether this is a raid captain
    public boolean isCaptain() {
        // Check if there's an ominous banner on the head
        ItemStack lv = this.getEquippedStack(EquipmentSlot.HEAD);
        boolean bl = !lv.isEmpty() && ItemStack.areEqual(lv, Raid.getOminousBanner(this.getRegistryManager().getWrapperOrThrow(RegistryKeys.BANNER_PATTERN)));
        // Check if this is a "patrol leader"
        boolean bl2 = this.isPatrolLeader();
        return bl && bl2;
    }
}
```

Worth noting is that the `RaiderEntity::isCaptain(DamageSource)` method, added in 24w13a, is only used for the loot table predicate `RaiderPredicate`. All captain-checking logic in the original raid mechanics uses `PatrolEntity::PatrolLeader()`. `RaiderPredicate` is currently only used for pillager loot tables to determine whether to drop Ominous Bottles.

```java
public record RaiderPredicate(boolean hasRaid, boolean isCaptain) implements EntitySubPredicate
{
    /* ... */
    // Requirements for dropping Ominous Bottle: hasRaid = false, isCaptain = true
    public static final RaiderPredicate CAPTAIN_WITHOUT_RAID = new RaiderPredicate(false, true);

    /* ... */
    @Override
    public boolean test(Entity entity, ServerWorld world, @Nullable Vec3d pos) {
        if (entity instanceof RaiderEntity) {
            RaiderEntity lv = (RaiderEntity)entity;
            // Predicate implementation logic, a simple AND operation
            return lv.hasRaid() == this.hasRaid && lv.isCaptain() == this.isCaptain;
        }
        // Always returns false for non-raiders
        return false;
    }
}
```

Additionally, under Yarn mapping, `PillagerEntity::isRaidCaptain(ItemStack)` is actually a method that checks "whether the pillager wants to pick up this item." In Mojang mapping it's called `wantsItem`, and it's not a captain check.


## III. Prerequisite Knowledge - When Will Raiders Pick Up Banners

Raiders picking up banners is controlled by `PickupBannerAsLeaderGoal` (note: "behavior" here has nothing to do with Minecraft's "memory behavior" AI system, and raiders use the "goal" AI system. For the operating principles of mob AI systems in MC, see [Minecraft Wiki - Mob AI](https://minecraft.wiki/w/Mob#AI)). For the basic framework of this type of AI, see the `net.minecraft.entity.ai.goal.Goal` class.

```java
public class PickupBannerAsLeaderGoal<T extends RaiderEntity> extends Goal {
    /* ... */

    // canStart() helps determine whether this AI goal can run
    @Override
    public boolean canStart() {
        List<ItemEntity> list;
        Raid lv = ((RaiderEntity)this.actor).getRaid();
        if (!((RaiderEntity)this.actor).hasActiveRaid()
            || ((RaiderEntity)this.actor).getRaid().isFinished()
            || !((PatrolEntity)this.actor).canLead()
            || ItemStack.areEqual(
                ((MobEntity)this.actor).getEquippedStack(EquipmentSlot.HEAD),
                Raid.getOminousBanner(((Entity)this.actor).getRegistryManager().getWrapperOrThrow(RegistryKeys.BANNER_PATTERN)))) {
            // Early return when requirements aren't met
            // Conditions: executor doesn't belong to an active raid,
            //      or belongs to a finished raid (victory or defeat),
            //      or this mob cannot become captain (depends on canLead() override, false for witches/ravagers, true otherwise),
            //      or this mob already has an ominous banner in helmet slot
            return false;
        }
        RaiderEntity lv2 = lv.getCaptain(((RaiderEntity)this.actor).getWave());
        if (!(lv2 != null
              && lv2.isAlive()
              || (list = ((Entity)this.actor).getWorld().getEntitiesByClass(
                  ItemEntity.class,
                  ((Entity)this.actor).getBoundingBox().expand(16.0, 8.0, 16.0),
                  OBTAINABLE_OMINOUS_BANNER_PREDICATE)
                  ).isEmpty())) {
            // Further conditions:
            // No captain exists for executor's raid and wave, or captain is dead, and nearby ominous banner item exists (see code for detection range)
            return ((MobEntity)this.actor).getNavigation().startMovingTo(list.get(0), 1.15f);
        }
        return false;
    }
    /* ... */
}

```


## IV. Interim Summary

With the prerequisite knowledge, we can now summarize the discussion environment.

First, when raiders are 96~112 blocks from the raid, and the raid is ongoing, and chunks are force-loaded, at this point the raid's `waveToCaptain` and `waveToRaiders` should both have mappings, and the raider's `raid` field should properly reference its belonging raid. So the return values of the four methods checking raid existence in `RaiderEntity` should be:

- `getRaid()`: current raid
- `hasRaid()`:
  - `this.getRaid() != null`: `raid` field is not null, so `true`
  - `lv.getRaidAt(this.getBlockPos()) != null`: current position is outside raid's 96 block range, can't get it, so `false`
  - OR operation on above two, result is `true`
- `hasActiveRaid()`: raid is ongoing, chunks force-loaded, so `true`
- `hasNoRaid()`: negation of previous method, so `false`


For captain checking methods, whether it's a spawned raid captain or a raider that picked up a banner to become captain, both are created through normal means, `patrolLeader` field is `true`, so:

- `PatrolEntity::isPatrolLeader()`: directly associated with the field, so result is `true`
- `RaiderEntity::isCaptain()`: has ominous banner in helmet slot, and `isPatrolLeader()` returns `true`, so result is `true`


For non-captain raider banner pickup goal, assuming captain is dead:

- Executor belongs to an active raid: yes
- Executor belongs to an unfinished raid: yes
- This mob type can become captain: not witch/ravager, so yes
- Executor has no ominous banner in helmet slot: yes
- No captain exists for executor's raid and wave, or captain is dead: yes
- Nearby ominous banner item exists: captain drops banner, so yes

From this we can conclude that raiders 96~112 blocks from the raid can normally trigger the banner pickup AI goal.


Now look at the `RaiderPredicate` used for pillager Ominous Bottle drops:

- `lv.hasRaid() == false`: `true == false`, result is `false`
- `lv.isCaptain() == true`: `true == true`, result is `true`
- AND operation on above two, result is `false`

According to this reasoning, pillager captains shouldn't drop Ominous Bottles, which contradicts the facts, so there must be an issue somewhere in the code.


## V. Raider Death Logic

Since drops aren't as expected, the loot table logic must be abnormal. Now focus on the raider's `onDeath()` method. `PillagerEntity`'s inheritance chain is `Entity` -> `LivingEntity` -> `MobEntity` -> `PathAwareEntity` -> `HostileEntity` -> `PatrolEntity` -> `RaiderEntity` -> `PillagerEntity`, of which only `RaiderEntity` overrides the `onDeath(DamageSource)` method from `LivingEntity`.

```java
public abstract class RaiderEntity extends PatrolEntity {
    /* ... */
    @Override
    public void onDeath(DamageSource damageSource) {
        if (this.getWorld() instanceof ServerWorld) {
            Entity lv = damageSource.getAttacker();
            Raid lv2 = this.getRaid();
            if (lv2 != null) {
                if (this.isPatrolLeader()) {
                    // If captain, remove self from raid's waveToCaptain map
                    lv2.removeLeader(this.getWave());
                }
                if (lv != null && lv.getType() == EntityType.PLAYER) {
                    lv2.addHero(lv);
                }
                lv2.removeFromWave(this, false);    // Remove self from raid
            }
        }
        super.onDeath(damageSource);    // Execute base class method of same name
    }
    /* ... */
}
```

```java
// Continuing from above, in the logic for removing raider from raid, the raider's raid property is set to null
// This way neither side retains a reference to the other
public class Raid {
    public void removeFromWave(RaiderEntity entity, boolean countHealth) {
        boolean bl2;
        Set<RaiderEntity> set = this.waveToRaiders.get(entity.getWave());
        if (set != null && (bl2 = set.remove(entity))) {    // Remove raider from waveToRaiders map
            if (countHealth) {
                this.totalHealth -= entity.getHealth();
            }
            entity.setRaid(null);    // Set this raider's raid property to null
            this.updateBar();
            this.markDirty();
        }
    }
}
```

```java
public abstract class LivingEntity extends Entity implements Attackable {
    /* ... */
    public void onDeath(DamageSource damageSource) {
        if (this.isRemoved() || this.dead) {
            return;
        }
        Entity lv = damageSource.getAttacker();
        LivingEntity lv2 = this.getPrimeAdversary();
        if (this.scoreAmount >= 0 && lv2 != null) {
            lv2.updateKilledAdvancementCriterion(this, this.scoreAmount, damageSource);
        }
        if (this.isSleeping()) {
            this.wakeUp();
        }
        if (!this.getWorld().isClient && this.hasCustomName()) {
            LOGGER.info("Named entity {} died: {}", (Object)this, (Object)this.getDamageTracker().getDeathMessage().getString());
        }
        this.dead = true;
        this.getDamageTracker().update();
        World world = this.getWorld();
        if (world instanceof ServerWorld) {
            ServerWorld lv3 = (ServerWorld)world;
            if (lv == null || lv.onKilledOther(lv3, this)) {
                this.emitGameEvent(GameEvent.ENTITY_DIE);
                this.drop(lv3, damageSource);    // Handle drops
                this.onKilledBy(lv2);
            }
            this.getWorld().sendEntityStatus(this, EntityStatuses.PLAY_DEATH_SOUND_OR_ADD_PROJECTILE_HIT_PARTICLES);
        }
        this.setPose(EntityPose.DYING);
    }
    /* ... */

    protected void drop(ServerWorld world, DamageSource damageSource) {
        boolean bl;
        boolean bl2 = bl = this.playerHitTimer > 0;
        if (this.shouldDropLoot() && world.getGameRules().getBoolean(GameRules.DO_MOB_LOOT)) {
            this.dropLoot(damageSource, bl);                // Process loot table, drop loot
            this.dropEquipment(world, damageSource, bl);    // Drop equipment
        }
        this.dropInventory();                       // Drop inventory contents
        this.dropXp(damageSource.getAttacker());    // Drop experience
    }
    /* ... */
}
```

Because the raider's `onDeath()` method first executes this class's processing logic, then calls the base class's `onDeath()` method, when actually processing the loot table, the killed pillager captain has already been removed from the raid. So the result of `RaiderEntity::hasRaid()` changes:

- `hasRaid()`:
  - `this.getRaid() != null`: raider has been removed from raid, `raid` field is null, so `false`
  - `lv.getRaidAt(this.getBlockPos()) != null`: unchanged, still `false`
  - OR operation on above two, result is `false`

Correspondingly, the `RaiderPredicate` result also changes:

- `lv.hasRaid() == false`: `false == false`, result is `true`
- `lv.isCaptain() == true`: `true == true`, result is `true`
- AND operation on above two, result is `true`

Therefore, pillager `RaiderPredicate` passes, can drop Ominous Bottle.

From the above analysis we can also see that when a raider dies, the `this.getRaid() != null` part of `RaiderEntity::hasRaid()` always returns `false`, the return value completely depends on `lv.getRaidAt(this.getBlockPos()) != null`. So the condition for pillagers dropping Ominous Bottles can be simplified to `raider.getRaidAt(raider.getBlockPos()) != null && raider.isCaptain() == true`, i.e., its death position is not within 96 blocks of any raid, and it is a raid captain.

## VI. Summary

In summary, it's precisely because the distance for removing raiders from a raid (112 blocks) is greater than the distance for pillager captains to drop Ominous Bottles (96 blocks) that makes the \[96, 112\) special range where pillager captains can drop Ominous Bottles and raid party members can pick up ominous banners.


<br>
<br>
<br>
<br>

"Code Analysis of Special Raider Behavior in \[96, 112\) Range in 1.21.x" © 2025 Author: Youmiel, licensed under CC BY-NC-SA 4.0. To view a copy of this license, visit http://creativecommons.org/licenses/by-nc-sa/4.0/.
