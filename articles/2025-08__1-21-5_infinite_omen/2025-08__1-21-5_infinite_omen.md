# 1.21.5+ Infinite Bad Omen to Raid Omen Conversion Code Analysis — ConcurrentModificationException and Its Exploitation

**If you need to discuss this feature with others, I think "Infinite Raid Omen Conversion", "Infinite Raid Omen", "Endless Raid Omen", "infiniOmen", "OmenInfinitum" are all good names** (credit: Ryan 100c, Lemon_Iced, 清云流烟).

According to player feedback, some raid farm designs used in 1.21.5 and above exhibit a behavior where Bad Omen doesn't disappear after converting to Raid Omen, allowing free Raid Omen generation every 30 seconds for the 100-minute duration of the effect. This article will provide a code explanation of this feature and discuss extended applications of the related principles.

Special thanks to Lemon_Iced, Nickid2018, s_yh and others for their help during the research, listed in no particular order.


## TL;DR

If you're familiar with Java programming, here's a brief explanation of this feature:

During collection iteration, **Bad Omen's code logic modifies the collection structure** (adding Raid Omen), causing the hashmap's iterator to throw `ConcurrentModificationException` (CME) during `iterator.remove()`, **failing to remove the Bad Omen effect**. Not until the next game tick does Bad Omen successfully get removed when it converts to Raid Omen again.

*Note: `ConcurrentModificationException` doesn't necessarily require a multi-threaded environment to trigger; the exception in this problem is entirely created in a single-threaded environment.*


## Table of Contents
- [TL;DR](#tldr)
- [0. Environment Setup](#0-environment-setup)
- [I. Prerequisite Knowledge: Overview of Minecraft 1.21.5 Status Effect System](#i-prerequisite-knowledge-overview-of-minecraft-1215-status-effect-system)
- [II. What is "ConcurrentModificationException"](#ii-what-is-concurrentmodificationexception)
- [III. Infinite Bad Omen to Raid Omen Conversion: The Most Direct Exploitation of CME](#iii-infinite-bad-omen-to-raid-omen-conversion-the-most-direct-exploitation-of-cme)
- [IV. Speculation: Why This Feature Wasn't Discovered Immediately, and How It Might Be Fixed](#iv-speculation-why-this-feature-wasnt-discovered-immediately-and-how-it-might-be-fixed)
- [V. Differences in CME Triggering Before and After 1.21.5](#v-differences-in-cme-triggering-before-and-after-1215)
- [VI. CME Skipping Status Effect Processing](#vi-cme-skipping-status-effect-processing)


## 0. Environment Setup

If you don't know where to obtain the source code but wish to follow along with this article, you can follow the steps in [Code Analysis of Special Raider Behavior in \[96, 112\) Range in 1.21.x](../2025-04__1-21_captain_replace/2025-04-09__1-21_captain_replace.md) to decompile the game source code.

This article's explanation is based on Minecraft versions 1.21 and 1.21.5, using Yarn deobfuscation mapping.


## I. Prerequisite Knowledge: Overview of Minecraft 1.21.5 Status Effect System

Mob status effect calculation is at the end of `LivingEntity::baseTick()`, named `LivingEntity::tickStatusEffects()`. Below is the main code of this method for reference.

```java
public abstract class LivingEntity extends Entity
implements Attackable, ServerWaypoint {
    /* ... */
    private final Map<RegistryEntry<StatusEffect>, StatusEffectInstance> activeStatusEffects = Maps.newHashMap();
    /* ... */

    protected void tickStatusEffects() {
        World world = this.getWorld();
        if (world instanceof ServerWorld) {
            ServerWorld lv = (ServerWorld)world;
            Iterator<Object> iterator = this.activeStatusEffects.keySet().iterator();
            // activeStatusEffects is a hashmap linking status effect registry entries to status effect instances on the mob
            // Instant effects are not added, such as Instant Damage and Instant Health
            try {
                while (iterator.hasNext()) {
                    RegistryEntry lv2 = (RegistryEntry)iterator.next();
                    StatusEffectInstance lv3 = this.activeStatusEffects.get(lv2);
                    if (!lv3.update(lv, this, () -> this.onStatusEffectUpgraded(lv3, true, null))) {
                        // ^ Update each status effect instance
                        iterator.remove();         // When update method returns false, call iterator's remove() method to remove this effect
                        this.onStatusEffectsRemoved(List.of(lv3));
                        continue;
                    }
                    if (lv3.getDuration() % 600 != 0) continue;
                    this.onStatusEffectUpgraded(lv3, false, null);
                }
            } catch (ConcurrentModificationException lv2) {
                // Caught ConcurrentModificationException, but does nothing
                // This exception is the protagonist of this article
            }
            /* ... */
            // Other code, not the focus, omitted
        } else {
            /* ... */
            // Client-side logic, not the focus, omitted
        }
    }
}
```

For status effect instances, the above code calls `StatusEffectInstance::update()`, whose contents are as follows:

```java
public class StatusEffectInstance implements Comparable<StatusEffectInstance> {
    public boolean update(ServerWorld world, LivingEntity entity, Runnable hiddenEffectCallback) {
        int i;
        if (!this.isActive()) {  // Check if this status effect is infinite or has remaining duration > 0, otherwise don't process subsequent logic
            return false;
        }
        int n = i = this.isInfinite() ? entity.age : this.duration;
        if (this.type.value().canApplyUpdateEffect(i, this.amplifier) &&
                !this.type.value().applyUpdateEffect(world, entity, this.amplifier)) {
            // ^ Check if this status effect instance needs processing this game tick
            // If yes, call applyUpdateEffect() to process the status effect, most status effect implementations are written in this method
            return false;
            // If this instance needs processing this game tick, and applyUpdateEffect() returns false, then return false
            // In tickStatusEffects(), code to remove current status effect instance will execute
        }
        this.updateDuration();   // Decrease status effect remaining duration by 1, i.e., calculate first then decrement timer
        if (this.tickHiddenEffect()) {
            hiddenEffectCallback.run();
            // Handle hidden status effects, e.g., having both "long duration, low level" and "short duration, high level" of same effect ID
        }
        return this.isActive();   // Same as above, i.e., control whether to clear current effect via remaining duration
    }
}
```


## II. What is "ConcurrentModificationException"

ConcurrentModificationException (hereinafter CME) is a runtime exception in Java, usually thrown when modifying the collection structure (e.g., adding/removing elements) while iterating over a collection. For a comprehensive understanding of CME triggering principles and scenarios, please read [java - Why is a ConcurrentModificationException thrown and how to debug it - Stack Overflow](https://stackoverflow.com/questions/602636/why-is-a-concurrentmodificationexception-thrown-and-how-to-debug-it). This article only lists the necessary items in this section.

### Why "ConcurrentModificationException" is Needed

Without this exception, modifying a collection during iteration could easily cause hard-to-debug problems. For example, the following JavaScript code:

```javascript
let arr = [1, 2, 3];
for (let i of arr) {
    arr.push(4); // Each loop iteration adds element "4" to the array end. No exception thrown, but may cause infinite loop or logic errors
}
```

In a NodeJS environment, executing this code causes an infinite loop, and programmers might not notice the loop is written incorrectly.

### Triggering Scenarios

`livingEntity.activeStatusEffect` is a HashMap. For HashMap, there are two scenarios that trigger CME:

1. **Directly modifying collection with collection methods during enhanced for loop / Iterator traversal** <br>
Enhanced for loops (like `for (T item : map.keySet())`) essentially still use Iterator to traverse collections. If `map.put(key, value)` or `map.remove(key)` adds/removes elements during traversal, it triggers an exception on the next call to `iterator.next()` or `iterator.remove()`.

2. **Using fail-fast non-thread-safe collections (like HashMap, ArrayList) for concurrent modification in multi-threaded environments** <br>
Since Minecraft's main game logic is single-threaded, status effect logic doesn't involve concurrent modification issues.


## III. Infinite Bad Omen to Raid Omen Conversion: The Most Direct Exploitation of CME

### Principle

Let's look at Bad Omen's implementation code:

```java
class BadOmenStatusEffect extends StatusEffect {
    protected BadOmenStatusEffect(StatusEffectCategory arg, int i) {
        super(arg, i);
    }

    @Override
    public boolean canApplyUpdateEffect(int duration, int amplifier) {
        return true;    // Every game tick needs to execute applyUpdateEffect()
    }

    @Override
    public boolean applyUpdateEffect(ServerWorld world, LivingEntity entity, int amplifier) {
        Raid lv2;
        ServerPlayerEntity lv;
        if (entity instanceof ServerPlayerEntity
                && !(lv = (ServerPlayerEntity) entity).isSpectator()
                && world.getDifficulty() != Difficulty.PEACEFUL
                && world.isNearOccupiedPointOfInterest(lv.getBlockPos())
                && ((lv2 = world.getRaidAt(lv.getBlockPos())) == null
                        || lv2.getBadOmenLevel() < lv2.getMaxAcceptableBadOmenLevel())) {
            // ^ Condition check for adding Raid Omen
            lv.addStatusEffect(new StatusEffectInstance(StatusEffects.RAID_OMEN, 600, amplifier));
            // ^ Here Raid Omen is added
            lv.setStartRaidPos(lv.getBlockPos());
            return false;    // Here returns false
        }
        return true;
    }
}
```

`BadOmenStatusEffect::applyUpdateEffect()` returns `false` after adding Raid Omen when conditions are met. Combined with the code description earlier, `StatusEffectInstance::update()` also returns `false`, and in `LivingEntity::tickStatusEffects()` it calls `iterator.remove()`. Because adding Raid Omen earlier changed the element count of `activeStatusEffect`, the subsequent `iterator.remove()` throws CME and fails to remove Bad Omen.

If on the next game tick, Raid Omen addition conditions are still met, code execution is the same as above, but `activeStatusEffect` already has Raid Omen, so adding again doesn't change element count, and subsequent `iterator.remove()` normally removes Bad Omen without throwing CME.

The above is the principle of infinite Bad Omen to Raid Omen conversion. Applied to raid farm design, this means having players meet Raid Omen addition conditions for only 1 game tick each time they acquire Raid Omen. Lemon_Iced's raid farm uses controllable POI claiming technology, and happens to make the village section exist for only 1 game tick, hence the infinite Raid Omen conversion phenomenon; if the village section persists, having players stay in the village section for only 1 game tick can also achieve the effect. Note that after acquiring Raid Omen, before Raid Omen converts to a raid, players cannot re-enter the village section, otherwise Bad Omen will also be removed.

### Timing Effects

For clarity, the following uses "normal trigger" to refer to the common method where players have Raid Omen while staying in a village section for more than 1 game tick; "infinite trigger" refers to the aforementioned method where players stay in the village section for only 1 game tick for infinite Raid Omen effect acquisition.

For normal trigger designs, the minimum trigger period is uncertain with two possible outcomes, specifically depending on the calculation order between Bad Omen and Raid Omen within the same game tick, and this calculation order isn't absolutely determined. Reading the code earlier, we can see `livingEntity.activeStatusEffect` is type `HashMap<RegistryEntry<StatusEffect>, StatusEffectInstance>`, and Mojang didn't implement `hashCode()` for `RegistryEntry.Reference<T>` (registry entry reference, implementation of `RegistryEntry<T>` interface). This means each program startup, the same set of status effects will have different traversal orders, and collection expansion (triggering rehash) will further affect their traversal order.

The above analysis can be verified with a mod; choose a mixin injection point where registry entries are initialized, construct a HashMap of the same type, store status effects in the map and output traversal results to verify.

#### Normal trigger, if Bad Omen is calculated before Raid Omen

Remaining time refers to the effect remaining time when `StatusEffect::canApplyUpdateEffect()` is called, same below.

- GT 0: Bad Omen adds Raid Omen, triggers CME, interrupts traversal; Raid Omen not calculated
- GT 1: Bad Omen adds Raid Omen again, removes itself; Raid Omen calculated, remaining time 600 gt
- GT 2: Raid Omen remaining time 599 gt
- ...
- ... (at some point, player obtains Bad Omen)
- ...
- GT 600: Raid Omen remaining time 1 gt, starts raid event, removes itself
- GT 601: Bad Omen can add Raid Omen again

Minimum raid generation period 601 gt.

#### Normal trigger, if Raid Omen is calculated before Bad Omen

- GT 0: Bad Omen adds Raid Omen, triggers CME, interrupts traversal; Raid Omen not calculated
- GT 1: Raid Omen calculated, remaining time 600 gt (decreases to 599 gt after calculation); Bad Omen adds Raid Omen again, removes itself, refreshes Raid Omen remaining time to 600 gt
- GT 2: Raid Omen remaining time 600 gt
- GT 3: Raid Omen remaining time 599 gt
- ...
- ... (at some point, player obtains Bad Omen)
- ...
- GT 601: Raid Omen remaining time 1 gt, starts raid event, removes itself
- GT 602: Bad Omen can add Raid Omen again

Minimum raid generation period 602 gt.

#### Infinite trigger

- GT 0: Bad Omen adds Raid Omen, triggers CME, interrupts traversal; Raid Omen not calculated
- GT 1: Raid Omen remaining time 600 gt
- GT 2: Raid Omen remaining time 599 gt
- ...
- ...
- GT 600: Raid Omen remaining time 1 gt, starts raid event, removes itself
- GT 601: Bad Omen can add Raid Omen again

Minimum raid generation period 601 gt.

#### Differences between real players and fake players

The description in [Classification of Stacking Raid Tower Applicability for Real Players](../2024-02__raid_explaination/2024-02-02__categories.md) regarding the phase difference in status effect calculation between real and fake players is still valid.


## IV. Speculation: Why This Feature Wasn't Discovered Immediately, and How It Might Be Fixed

### Why Neither Mojang Nor the Player Community Previously Discovered This Issue

#### 1. Stringent trigger conditions

Infinite Raid Omen conversion requires players to stay in the village section for only 1 game tick. Unless specifically constructed (using redstone circuits for control, fine operation under `/tick rate 1`, moving players at breakpoints in development environment, etc.), manual operation can hardly achieve 1 game tick stays. Even `/tick freeze` can't stop player status effects from continuing to calculate. Normal triggering methods only have a chance to delay Raid Omen end time by 1 game tick, which is hard to notice by eye alone.

#### 2. Mojang's lack of discipline during program development

Think about it—many inexplicable issues originate from Mojang's undisciplined development, for example:
1. [Dimension-dependent random redstone](https://www.bilibili.com/video/BV1Gv411t7sb?p=1): caused by Mojang using hashmap to store three dimensions but not implementing `hashCode` for dimensions
2. [Nether Fortress Nether Brick Mob Spawning Wander Issue in Minecraft 1.18.2+](https://blog.fallenbreath.me/2024/fortress-nether-bricks-pack-spawning-issue-1182/#more): caused by Mojang not implementing `equals` for `SpawnEntry`, this issue lingered for nearly three years before being discovered and accidentally fixed.

This time, Mojang's amateur mistakes include at least:
- Directly increasing element count during collection iteration
- Catching an exception and doing nothing
- Not implementing `hashCode()` for `RegistryEntry.Reference<T>`
- Using `HashMap`, a data structure with unstable traversal order, causing issues to be probabilistically masked by micro-timing

### How Mojang Might Fix This

Besides completing unimplemented methods and switching to more suitable data structures, the core issue is adding new status effects during status effect collection iteration, and the existing status effect system didn't consider such special usage. The solution is to avoid adding new status effects while iterating the collection (I think changing Bad Omen to directly generate raids would be great, right? `:P`), or simply refactor the status effect system for this requirement.

As for when it will be fixed, it depends on when this feature reaches Mojang, or when Mojang on a whim randomly refactors status effect code and maybe fixes it.


## V. Differences in CME Triggering Before and After 1.21.5

Actually, by 25w02a, Mojang was already consciously avoiding potion effect code triggering CME and modified some status effect processing logic. Next I'll explain what's different between status effect logic in versions 1.21 ~ 1.21.4 and versions after 1.21.5, and how to trigger CME.

### 1.21 Status Effect Code

First is `LivingEntity::tickStatusEffect()`, whose relevant code has basically not changed.

```java
protected void tickStatusEffects() {
    List<ParticleEffect> list;
    Iterator<RegistryEntry<StatusEffect>> iterator = this.activeStatusEffects.keySet().iterator();
    try {
        while (iterator.hasNext()) {
            RegistryEntry<StatusEffect> lv = iterator.next();
            StatusEffectInstance lv2 = this.activeStatusEffects.get(lv);
            if (!lv2.update(this, () -> this.onStatusEffectUpgraded(lv2, true, null))) {
                // Traversal method unchanged
                if (this.getWorld().isClient) continue;
                iterator.remove();   // Removal method unchanged
                this.onStatusEffectRemoved(lv2);
                continue;
            }
            if (lv2.getDuration() % 600 != 0) continue;
            this.onStatusEffectUpgraded(lv2, false, null);
        }
    } catch (ConcurrentModificationException lv) {
        // Caught ConcurrentModificationException, but does nothing
    }
    /* ... */
    // Other code, not the focus, omitted
}
```

Then `StatusEffectInstance::update()`

```java
public boolean update(LivingEntity entity, Runnable overwriteCallback) {
    if (this.isActive()) {
        int i;
        int n = i = this.isInfinite() ? entity.age : this.duration;
        if (this.type.value().canApplyUpdateEffect(i, this.amplifier)
                && !this.type.value().applyUpdateEffect(entity, this.amplifier)) {
            entity.removeStatusEffect(this.type);
            // After applyUpdateEffect() returns false, unlike 1.21.5 which continues returning false outward, it removes the effect on the spot
            // Internal implementation uses map.remove(item), won't throw CME
            // But will cause subsequent calls to iterator.next() etc. to throw CME
            // Since there's no direct return, subsequent logic still executes (duration decrease, hidden effect handling, etc.)
        }
        this.updateDuration();    // Calculate first then countdown, unchanged
        if (this.duration == 0 && this.hiddenEffect != null) {
            // Handle hidden status effects
            this.copyFrom(this.hiddenEffect);
            this.hiddenEffect = this.hiddenEffect.hiddenEffect;
            overwriteCallback.run();
        }
    }
    this.fading.update(this);
    return this.isActive();    // Compared to 1.21.5, only one return location
}
```

### 1.21 ~ 1.21.4 CME Creation Method Analysis

We can see 1.21's status effect processing logic won't call `iterator.remove()` when adding Raid Omen and throw CME, causing Bad Omen removal failure. However, we can still try to add/remove status effects before `iterator.next()` or `iterator.remove()` calls to throw CME.

There are two methods for adding/removing:
1. Bad Omen adds Raid Omen when trigger conditions are met
2. Try to make `applyUpdateEffect()` return `false` to remove itself; currently only Bad Omen adding Raid Omen, Raid Omen generating raid, and Absorption depleting yellow hearts return `false`

There are two locations that can throw CME:
1. `iterator.next()`: If the effect that was added/removed isn't the last one, called when iterating to the next effect
2. `iterator.remove()`: Only called when status effect duration ends

Combining these yields the following CME creation methods:
1. Fixed trigger when Raid Omen generates raid. Trigger location `iterator.remove()`
2. Bad Omen adds Raid Omen or Absorption depletes yellow hearts, and there are still effects not yet iterated. Trigger location `iterator.next()`
3. Absorption depletes yellow hearts while duration ends simultaneously. Trigger location `iterator.remove()`

Due to the uncertainty in status effect iteration order mentioned in [Section III](#timing-effects), CME triggering method `2.` is unstable.


## VI. CME Skipping Status Effect Processing

When CME triggers, the loop processing that player's status effects this game tick directly interrupts; effects not yet iterated not only don't have duration decreased, the effects they should have produced also don't occur—essentially skipping one game tick of processing.

For Bad Omen's behavior in versions 1.21.5 and above, because after adding Raid Omen, that branch only does `return false` and doesn't decrease duration, using the infinite trigger feature can also extend Bad Omen duration to some extent.

### 1.21 ~ 1.21.4 CME Timing Effects on Raid Farms

#### Bad Omen calculated before Raid Omen

- GT 0: Bad Omen meets conversion conditions, adds Raid Omen, removes Bad Omen, whether CME triggers doesn't matter
- GT 1: Raid Omen remaining time 600 gt
- GT 2: Raid Omen remaining time 599 gt
- ...
- ...(at some point, player obtains Bad Omen)
- ...
- GT 600: Raid Omen remaining time 1 gt, starts raid event, removes itself
- GT 601: Bad Omen meets conversion conditions, adds Raid Omen...

Minimum raid generation period 601 gt.

#### Raid Omen calculated before Bad Omen

- GT 0: Bad Omen meets conversion conditions, adds Raid Omen, removes Bad Omen, whether CME triggers doesn't matter
- GT 1: Raid Omen remaining time 600 gt
- GT 2: Raid Omen remaining time 599 gt
- ...
- ...(at some point, player obtains Bad Omen)
- ...
- GT 600: Raid Omen remaining time 1 gt, starts raid event, removes itself, triggers CME, doesn't process Bad Omen
- GT 601: Bad Omen meets conversion conditions, adds Raid Omen...

Minimum raid generation period 601 gt.


## VII. Summary

In summary, the 1.21.5 infinite Bad Omen to Raid Omen conversion is a new feature Mojang inadvertently created when refactoring code, (possibly) attempting to resolve ConcurrentModificationException. Exploiting ConcurrentModificationException can not only obtain Raid Omen for free but also skip some status effect calculations. Although it's unclear whether this can be meaningfully exploited, understanding its effects does somewhat help with Minecraft survival technology exploration.


<br>
<br>
<br>
<br>

"1.21.5+ Infinite Bad Omen to Raid Omen Conversion Code Analysis — ConcurrentModificationException and Its Exploitation" © 2025 Author: Youmiel, licensed under CC BY-NC-SA 4.0. To view a copy of this license, visit http://creativecommons.org/licenses/by-nc-sa/4.0/.
