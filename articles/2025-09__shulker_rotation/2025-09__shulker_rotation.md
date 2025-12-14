# Code Is Not Magic, but "Chirality" Is Pseudoscience — How Shulker Horizontal Rotation Affects Dismount Position

*This article aims to research a shulker feature discovered by yellowxuuu in February 2022: [\[MC\] Chiral Shulkers](https://www.bilibili.com/video/BV1LY41157th). Since there was no public, clear explanation of the principle for many years after this feature was discovered, some players developed a phenomenon-based understanding of it, clearly misunderstanding its actual mechanism and scenarios.*


## TL;DR

The actual principle of "chirality" is that when a shulker rides a boat, its **horizontal facing** (`yaw` attribute) is constrained to within `[-105°, +105°]` **relative to the boat's horizontal facing**. After the **horizontal facing** is changed, the shulker's **dismount direction** also changes accordingly. This mechanism still exists as of 25w36b, applies to all boat-type vehicles (boats, chest boats, bamboo rafts), and can be returned to original orientation through specially designed structures. Using "chirality" to refer to this feature easily causes confusion with its original meaning; it's recommended to use "**hidden yaw**" instead (credit: Fallen_Breath).

*\*Readers who want to gradually understand the mechanism discovery process and related code can start reading from the next section; readers who want to understand yellowxuuu's device principle please go to ["IX. Original Device Operation Principle Analysis"](#ix-original-device-operation-principle-analysis); readers who want to directly obtain practical knowledge please go to ["IX. Original Device Operation Principle Analysis - Remaining Questions Answered"](#remaining-questions-answered); readers who want to learn the most appropriate terminology for this feature please go to ["XIII. What Would Be a More Appropriate Name"](#xiii-what-would-be-a-more-appropriate-name)*


## 0. Environment Setup

If you don't know where to obtain the source code but wish to follow along with this article, you can follow the steps in [Code Analysis of Special Raider Behavior in \[96, 112\) Range in 1.21.x](../2025-04__1-21_captain_replace/2025-04-09__1-21_captain_replace.md) to decompile the game source code.

This article's explanation is based on Minecraft version 1.18.1, using Yarn deobfuscation mapping. The analysis process may involve breakpoint debugging and other methods, so using a Fabric mod development environment is more recommended.


## I. Term Definitions

- **Horizontal rotation/Horizontal facing**: Here refers to the angle of entity rotation around the y-axis. The two terms have similar meanings and may be used interchangeably. In Minecraft, the y-axis is the vertical axis; related rotation angles are named `yRot` (Mojang mapping) and `yaw` (Yarn mapping) in code.
- **"Chirality"**: An arbitrarily named feature term, used by yellowxuuu to describe the feature covered in this article. It's unclear if there's only a "left" and "right" distinction; only used when referencing the original video or quoting others.


## II. Initial Analysis

According to the behavior shown in [BV1LY41157th](https://www.bilibili.com/video/BV1LY41157th), the shulker went through the following stages:

1. Spawn
2. First boarding, from the boat's stern
3. First dismount, to the left or right side
4. Several consecutive teleports
5. Random teleport to the left or right boat, boarding from the stern
6. Dismount position determined by its acquired "chirality"

The primary question is at which stage the shulker acquired "chirality." Currently, it appears that both left and right shulkers went through the same teleportation process, so teleportation probably doesn't affect shulker boarding/dismounting behavior. So the possible stages are 1, 2, or 3.

Additional questions: Why can shulkers dismount to different positions, but minecarts don't have a similar feature? Why doesn't the second boarding/dismounting give shulkers new "chirality"? Does this suggest boarding from the stern vs. from left/right sides have different properties? If the left shulker first dismounts to the boat's right side, and the right shulker first dismounts to the boat's left side, how would the situation change?


## III. Minecart vs. Boat Dismount Logic Differences

Consulting the [Minecraft wiki](https://minecraft.wiki), the pages for minecarts and boats respectively describe two dismount logics:

> [Minecart](https://minecraft.wiki/w/Minecart#Dismount): In order of right, left, back-right, back-left, front-right, front-left, back, front, it checks the eight horizontal directions (relative to direction of movement) for adjacent blocks. Then, if there is no suitable position adjacent to the minecart at the same height, the minecart searches one block above and below in sequence. For players, the minecart continues to check positions that can accommodate a crouching player. If still not found, the minecart chooses its own position as the mob's dismount position. After determining the dismount position, the minecart teleports the entity to the center of that position.

> [Boat](https://minecraft.wiki/w/Boat#Behavior): In Java Edition, the landing point when disembarking is determined by the player's current view horizontal direction. If there is no solid block immediately adjacent to the boat in the direction the player is facing, the landing point is the center of the boat.

This shows minecart dismount position is determined by the minecart's **movement state**, while boat dismount position is determined by the passenger's **horizontal facing**. The boat code confirms this:

```java
public class BoatEntity extends Entity {
    /* ... */
    public Vec3d updatePassengerForDismount(LivingEntity passenger) {
        Vec3d vec3d = getPassengerDismountOffset((double)(this.getWidth() * MathHelper.SQUARE_ROOT_OF_TWO), (double)passenger.getWidth(), passenger.getYaw());
        // ^ Calculate dismount offset based on passenger's horizontal rotation
        double d = this.getX() + vec3d.x;
        double e = this.getZ() + vec3d.z;
        BlockPos blockPos = new BlockPos(d, this.getBoundingBox().maxY, e);  // Predetermined position 1
        BlockPos blockPos2 = blockPos.down();  // Predetermined position 2
        if (!this.world.isWater(blockPos2)) {
            /* Check if dismount position meets dismount conditions, omitted */
        }
        return super.updatePassengerForDismount(passenger);
        // ^ Call parent class method, i.e., place passenger at center of own bounding box upper surface
    }
   /* ... */
}
```

Setting a breakpoint in the fabric mod development environment, we can see the call stack when a shulker dismounts:

```
BoatEntity.updatePassengerForDismount(LivingEntity)
LivingEntity.onDismounted(Entity)
LivingEntity.stopRiding()
ShulkerEntity.stopRiding()
Entity.removeAllPassengers()
BoatEntity.tick()
...
```

The above code and call stack only use the passenger's horizontal facing without modifying it, meaning horizontal facing is modified elsewhere, so we need to focus on factors affecting horizontal facing next.


## IV. Mob Rotation Property Overview

In `Entity` class:
```java
private float yaw;
private float pitch;
public float prevYaw;
public float prevPitch;
```

In `LivingEntity` class:
```java
public float bodyYaw;
public float prevBodyYaw;
public float headYaw;
public float prevHeadYaw;
// Some other properties that don't directly affect pose, not listed
```

We can see that shulkers inherit three horizontal rotation properties from parent classes, with the following meanings:
- `yaw`: Horizontal rotation that all entities have, including boats, item entities, and other non-mobs;
- `bodyYaw`: Mob body horizontal rotation, usually synced with `yaw` by mob's `tick()` or AI control code, but exceptions exist;
- `headYaw`: Mob head horizontal rotation.


## V. Eliminating Interference

### Does Teleportation Affect Horizontal Rotation

To be safe, first check if shulker teleportation affects horizontal rotation, to avoid interfering with subsequent analysis.

```java
public class ShulkerEntity extends GolemEntity implements Monster {
    /* ... */
    protected boolean tryTeleport() {
        if (!this.isAiDisabled() && this.isAlive()) {
            BlockPos blockPos = this.getBlockPos();

            for(int i = 0; i < 5; ++i) {
                BlockPos blockPos2 = blockPos.add(MathHelper.nextBetween(this.random, -8, 8), MathHelper.nextBetween(this.random, -8, 8), MathHelper.nextBetween(this.random, -8, 8));
                if (blockPos2.getY() > this.world.getBottomY() && this.world.isAir(blockPos2) && this.world.getWorldBorder().contains(blockPos2) && this.world.isSpaceEmpty(this, (new Box(blockPos2)).contract(1.0E-6))) {
                Direction direction = this.findAttachSide(blockPos2);
                    if (direction != null) {
                        this.detach();  // Detach from all passengers and vehicles, not in scope of discussion
                        this.setAttachedFace(direction);  // Modify attached facing, independent property, doesn't affect horizontal rotation
                        this.playSound(SoundEvents.ENTITY_SHULKER_TELEPORT, 1.0F, 1.0F);
                        this.setPosition((double)blockPos2.getX() + 0.5, (double)blockPos2.getY(), (double)blockPos2.getZ() + 0.5);  // Modify position, doesn't affect horizontal rotation
                        this.dataTracker.set(PEEK_AMOUNT, (byte)0);  // Modify shell opening degree
                        this.setTarget((LivingEntity)null);
                        // ^ Modify aggro target, most aggro-related AI only uses LookControl, modifying headYaw not yaw
                        return true;
                    }
                }
            }

            return false;
        } else {
            return false;
        }
    }
    /* ... */
}
```

### Does Shulker AI Affect Horizontal Rotation

Since the shulker AI part encountered earlier involves look control (`LookControl`), let's also check body control (`BodyControl`); regular mob AI modifies horizontal rotation in this part to coordinate movement and pathfinding. Shulkers use the special `ShulkerEntity.ShulkerBodyControl`, code as follows:

```java
static class ShulkerBodyControl extends BodyControl {
    public ShulkerBodyControl(MobEntity mobEntity) {
        super(mobEntity);
    }

    @Override
    public void tick() {
    }
}
```

That's right, it's empty, meaning shulker AI cannot cause any effect on its horizontal rotation. Additionally, shulker's `yaw` and `bodyYaw` are decoupled and won't sync. (Note: Shulker's `tick()` also doesn't contain code modifying horizontal rotation)


## VI. How Shulker's Own Code Handles Horizontal Facing

Points mentioned earlier won't be repeated; below are several more locations that set horizontal facing:

```java
public class ShulkerEntity extends GolemEntity implements Monster {
    /* ... */
    @Override
    public void stopRiding() {
        super.stopRiding();
        if (this.world.isClient) {
            this.prevAttachedBlock = this.getBlockPos();
        }
        // v When dismounting, set bodyYaw to 0, but doesn't affect yaw
        this.prevBodyYaw = 0.0F;
        this.bodyYaw = 0.0F;
    }

    @Override
    public EntityData initialize(
        ServerWorldAccess world, LocalDifficulty difficulty, SpawnReason spawnReason, @Nullable EntityData entityData, @Nullable NbtCompound entityNbt
    ) {
        // v When spawning, set yaw to 0
        this.setYaw(0.0F);
        this.headYaw = this.getYaw();
        this.resetPosition();
        return super.initialize(world, difficulty, spawnReason, entityData, entityNbt);
    }

    @Override
    public void updateTrackedPositionAndAngles(double x, double y, double z, float yaw, float pitch, int interpolationSteps, boolean interpolate) {
        this.bodyTrackingIncrements = 0;
        this.setPosition(x, y, z);
        // v Network-related code, sync rotation properties
        this.setRotation(yaw, pitch);
    }

    @Override
    public void readFromPacket(MobSpawnS2CPacket packet) {
        super.readFromPacket(packet);
        // v When client handles shulker spawn packet, set bodyYaw to 0
        this.bodyYaw = 0.0F;
        this.prevBodyYaw = 0.0F;
    }
    /* ... */
}
```
```java
public abstract class LivingEntity extends Entity {
    /* ... */
	protected LivingEntity(EntityType<? extends LivingEntity> entityType, World world) {
		super(entityType, world);
        /* ...other unrelated initialization logic... */
		this.setYaw((float)(Math.random() * (float) (Math.PI * 2)));    // Set yaw (unit is degrees) to random value in [0.0f, 2 * PI)
		this.headYaw = this.getYaw();
        /* ...other unrelated initialization logic... */
	}
    /* ... */
}
```

It's not hard to notice that the above code is either initialization or handling client-server sync logic, not subsequently changing horizontal facing.

Worth noting is in the `LivingEntity` constructor, mob horizontal facing is initialized to a random value in `[0.0f, 2 * PI)`, but this property is used in degrees, causing most mobs to roughly face the +z direction when created instead of other directions. For shulkers, `/summon` command and spawn egg spawning call `ShulkerEntity.initialize(...)`, making horizontal rotation strictly `0.0f`; while End City structure spawning and shulkers spawned by shulker bullet hits don't call `ShulkerEntity.initialize(...)`, making horizontal rotation a random value in `[0.0f, 2 * PI)`.


## VII. Does Boarding Change Horizontal Facing

Setting a breakpoint at `ShulkerEntity.startRiding(Entity, boolean)`, call stack is:

```
MobEntity.startRiding(Entity,boolean)
ShulkerEntity.startRiding(Entity,boolean)
Entity.startRiding(Entity)
BoatEntity.tick()
```

Among these, the ones with substantial content, `ShulkerEntity.startRiding(Entity,boolean)` and `MobEntity.startRiding(Entity,boolean)`, are as follows:

```java
// ShulkerEntity
public boolean startRiding(Entity entity, boolean force) {
    if (this.world.isClient()) {
        this.prevAttachedBlock = null;
        this.teleportLerpTimer = 0;
    }
    // v Set attached facing
    this.setAttachedFace(Direction.DOWN);
    return super.startRiding(entity, force);
}
```
```java
// MobEntity
public boolean startRiding(Entity entity, boolean force) {
    boolean bl = super.startRiding(entity, force);
    if (bl && this.isLeashed()) {
        this.detachLeash(true, true);
    }  // ^ Detach leash

    return bl;
}
```

This shows shulkers also don't directly change horizontal facing when boarding. Having eliminated the possibilities of horizontal facing changing during boarding, dismounting, and shulker's own calculations, the only possibility left is that the boat changed the shulker's horizontal facing.


## VIII. How the Boat Changes Shulker Horizontal Facing

The most convenient method to find when shulker horizontal facing is changed is to set a conditional breakpoint on `Entity.setYaw(float)` to trigger when shulker horizontal facing changes.

![shulker data](./img/shulker_data.png)

In yellowxuuu's example save, the shulker's horizontal rotation after going through the right device is `-71.450195f`, so use conditional expression `this instanceof net.minecraft.entity.mob.ShulkerEntity && yaw > -72 && yaw < -71`. Call stack when breakpoint triggers:

```
Entity.setYaw(float)
BoatEntity.copyEntityData(Entity)
BoatEntity.updatePassengerPosition(Entity)
Entity.tickRiding()
LivingEntity.tickRiding()
ServerWorld.tickPassenger(Entity,Entity)
ServerWorld.tickEntity(Entity)
```

This matches the earlier speculation very well. `BoatEntity.updatePassengerPosition(Entity)` as the name suggests contains the logic for the boat updating passenger position, which is indeed the case. `BoatEntity.copyEntityData(Entity)` is rather peculiar; in Mojang mapping its name is `clampRotation(Entity)`, meaning to limit passenger rotation angle, which better matches this method's content:

```java
public class BoatEntity extends Entity {
    /* ... */
    // clampRotation (Mojang mapping)
	protected void copyEntityData(Entity entity) {
		entity.setBodyYaw(this.getYaw());
        // v Calculate angle difference from boat's horizontal facing to passenger's horizontal facing, value range limited to [-180.0f, 180.f)
		float f = MathHelper.wrapDegrees(entity.getYaw() - this.getYaw());
		float g = MathHelper.clamp(f, -105.0F, 105.0F);  // Limit angle difference to [-105.0f, 105.0f]

        // v If g - f == 0, means previous angle difference didn't exceed range, no adjustment needed
		entity.prevYaw += g - f;
		entity.setYaw(entity.getYaw() + g - f);  // Adjust passenger's horizontal facing
		entity.setHeadYaw(entity.getYaw());
	}
    /* ... */
}
```

It's precisely this method that changes the shulker's horizontal facing, and since the shulker's calculation logic doesn't actively change horizontal facing, this changed value can persist until dismounting, manifesting as differences in dismount position.


## IX. Original Device Operation Principle Analysis

First understand Minecraft's coordinate system. y-axis points up, right-handed system, entity horizontal rotation is 0 when facing positive z-axis. In the figure, red, yellow, blue represent x, y, z axes respectively, consistent with F3 screen; subsequent screenshots will include coordinate reference.

![coordinate system](./img/coord_sys.png)

### 1. Shulker First Boarding

![set 01](./img/set_01.png)

Both boats face -z direction; the left device's boat has horizontal rotation `175.99316f`, the right device's boat has horizontal rotation `-176.4502f`. Shulker after boarding from left device has horizontal rotation adjusted to `70.993164f`, dismounting toward -x direction; after boarding from right device has horizontal rotation adjusted to `-71.450195f`, dismounting toward +x direction. Matches the `[-105.0f, 105.0f]` angle difference found in code, also matches passenger dismount mechanics.


### 2. Shulker Consecutive Teleports Through Scaffolding Passage

![set 02](img/set_02.png)

Code review confirmed that teleportation doesn't affect shulker horizontal facing.

### 3. Shulker Second Boarding

![set 03](img/set_03.png)

Both boats face -z direction; the left device's boat has horizontal rotation `176.15292f`, the right device's boat has horizontal rotation `-177.74106f`.

Shulker from lower left boat, with +70 ish horizontal rotation: going through left device, horizontal rotation becomes `71.15292f`, dismounts toward -x direction, then teleports to left platform; going through right device, horizontal rotation becomes `77.25894f`, tries to dismount toward -x direction but no suitable foothold in that direction, so lands on top of boat, **if later randomly teleporting to left device, can normally dismount toward -x direction**.

Shulker from lower right boat, with -70 ish horizontal rotation: going through right device, horizontal rotation becomes `-72.74106f`, dismounts toward +x direction, then teleports to right platform; going through left device, horizontal rotation becomes `-78.84708f`, tries to dismount toward +x direction but no suitable foothold in that direction, so lands on top of boat, **if later randomly teleporting to right device, can normally dismount toward +x direction**.

Therefore, the second-layer boat doesn't significantly change shulker horizontal rotation, more of a filtering function; as long as the boat roughly faces -z direction, it can separate shulkers with two types of horizontal rotation left and right. Left-right mirrored devices can be merged and still work normally, as shown:

![set 03 combine](img/set_03_combine.png)

### Remaining Questions Answered

At the [beginning of the article](#ii-initial-analysis), a series of questions about this phenomenon were raised. Now knowing the principle, we can explain the unanswered questions.

#### Q: Why can shulkers dismount to different positions, but minecarts don't have a similar feature?

A: Already answered in ["III. Minecart vs. Boat Dismount Logic Differences"](#iii-minecart-vs-boat-dismount-logic-differences).

#### Q: Why doesn't the second boarding/dismounting give shulkers new "chirality"?

A: Actually, shulkers still have their horizontal facing changed by the boat after the second boarding, but since the two boats they board have similar orientations, the shulker's horizontal facing change is small; or it happens to be within the allowed range with no change, so dismount direction appears unchanged.

#### Q: Does this suggest boarding from the stern vs. from left/right sides have different properties?

A: Already confirmed in ["VII. Does Boarding Change Horizontal Facing"](#vii-does-boarding-change-horizontal-facing) that boarding logic doesn't change passenger horizontal facing, so boarding direction has no effect.

#### Q: If the left shulker first dismounts to the boat's right side, and the right shulker first dismounts to the boat's left side, how would the situation change?

A: If "dismount to boat's side" means moving the foothold block to a different position, then besides shulker first dismount landing on top of the boat, horizontal facing change amount and second dismount position won't change. If "dismount to boat's side" means adjusting boat orientation to give shulker a different horizontal facing, then shulkers will ultimately transfer to diagonal platforms instead of same-side platforms.


## X. Misunderstood "Chirality"

yellowxuuu's video only demonstrated the separation device and used "chirality" to explain this phenomenon, without deeper explanation of "chirality." But the player community over many years of shulker farm design practice filled in the feature's rules, trigger conditions, etc. based solely on speculation, ultimately adding many baseless descriptions to this feature.

> **Chirality**
>
> When a shulker rides a boat for the first time after spawning, depending on its dismount direction, it will get a left or right chirality.
>
> Shulkers that attach to the left side of the boat on first dismount will get left chirality, and can only dismount from the left side when riding boats again, and vice versa. If there's no suitable block on the side corresponding to the chirality, the shulker will stay on the block where the boat is located.
>
> Once a shulker's chirality is determined it cannot be changed, but shulkers duplicated from existing chiral shulkers have no chirality.
>
> —— Minecraft wiki [Shulker#Chirality (edited by user Fight xing)](https://minecraft.wiki/w/Shulker?oldid=2440982#Chirality) (Note: English wiki content may differ)

The following sections will enumerate errors in the quoted paragraph and provide corrections.

### "...depending on its dismount direction, it will get..."

According to [previous analysis](#ix-original-device-operation-principle-analysis), shulker dismount position is determined by its horizontal facing, and how horizontal facing changes is determined by relative rotation between boat and shulker.

That is to say, it's not that which direction the shulker dismounts to that gives it "chirality," but its "chirality" determines which direction it dismounts to, unrelated to the environment around the boat. Even if it dismounts to directly on top of the boat, the horizontal facing interpreted as "chirality" was already modified while on the boat.

### "...left or right chirality..."

The actual property behind "chirality" is entity horizontal rotation, and this property's value range is typically `[-180.0f, 180.f)`. Even constrained by boat relative rotation range, `[-105.0f, 105.0f]` can correspond to seven dismount positions, including "forward" which belongs to neither left nor right.

### "...can only dismount from x side when riding boats again..."

As stated in ["III. Minecart vs. Boat Dismount Logic Differences"](#iii-minecart-vs-boat-dismount-logic-differences), where a passenger dismounts depends on passenger horizontal facing, not boat left/right side. A simple example: a shulker with horizontal facing 0 (+z direction) boards a boat facing -170 (-z slightly toward +x), horizontal facing becomes 65 (+x +z direction), dismounting from right side. Then let it board a boat facing 0 (+z direction), its own horizontal facing unchanged, but relative to the boat it now dismounts from the boat's left side.

### "...Once a shulker's chirality is determined it cannot be changed..."

Since shulker horizontal facing is changed by the boat's passenger facing constraint code, as long as the shulker boards another boat with relative rotation difference greater than 105, its horizontal facing can change again.


## XI. Restoring Original Horizontal Facing

Below is a prototype device for gradually adjusting shulkers with arbitrary horizontal facing back to +z facing (the facing when horizontal rotation is 0), design version 1.18.1. Only guarantees same facing, doesn't guarantee `yaw` numeric values are equal; might be `360.0f`, `-360.0f`, etc. Red nether brick is shulker initial position.

Boat horizontal facings from bottom to top are:
- (Acacia boat) arbitrary, simulates shulker being randomly rotated
- 115
- 45
- -25
- -95
- (Dark oak boat) -105

![rotate back](img/rotate_back_w_coord.png)

If player participation is allowed, an easier method is for the player to control the boat rotating 360° in any direction, then smoothly rotate to `yaw` equal to 105.0f or -105.0f.


## XII. Does This Feature Exist in Newer Versions

Yes, currently it appears that through 25w36b, the code causing the core effect hasn't changed. Additionally, this feature applies to all newly added boat-type vehicles (bamboo raft, chest boat, chest bamboo raft).


## XIII. What Would Be a More Appropriate Name?

This feature is actually just "when a shulker rides a boat, its horizontal facing is constrained and adjusted by the boat's facing," there's no independent, unchangeable property. Using "chirality" to describe it not only easily leads to misunderstanding it as a binary, irreversible state, but may also obscure the fact that its essence is "horizontal facing," a continuous variable.

Moreover, "chirality" as a professional term is widely used in science and education fields; casually borrowing it can easily confuse the word's meaning.

Therefore, it's recommended to use "hidden yaw" to describe this mechanism (credit: Fallen_Breath), which both mentions the "horizontal rotation" property and hints that this property's change isn't externally displayed. When specifically discussing the mechanism, directly using the property's original name "horizontal facing," `"yaw"`, or `"yRot"` is better, avoiding ambiguity and facilitating correspondence with code and game mechanics.

In summary, I hope this article's analysis can provide reference for related discussions.




<br>
<br>
<br>
<br>

"How Shulker Horizontal Rotation Affects Dismount Position" © 2025 Author: Youmiel, licensed under CC BY-NC-SA 4.0. To view a copy of this license, visit http://creativecommons.org/licenses/by-nc-sa/4.0/.
