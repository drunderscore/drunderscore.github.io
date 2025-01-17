---
title: 'The Spongebob SquarePants Movie: Bowl Storage'
category: Reverse Engineering
tags:
- writeup
excerpt: A technical explanation of "Bowl Storage" -- a memory corruption exploit present in the original Xbox release. Utilized in speedrunning during "Mindy Skip", this was presented to the community to increase understanding and consistency.
---
In the lead-up to Awesome [Games Done Quick](https://gamesdonequick.com/) 2025 (AGDQ 2025), I had a cursory look at the schedule. What a surprise to see a blast from a long-ago past: _The Spongebob SquarePants Movie_!
A younger self had played this poorly, yet what a wonderful feeling it was to see again. My interested was further piqued by two major factors:
- The speedrunner appearing at this GDQ has a Twitch channel named [BubbleBastard](https://twitch.tv/BubbleBastard)
- One of the techniques used in the speedrun, named Mindy Skip, had the potential to _crash the game_.

Anything that can crash your game is exploitable. Something with only the _potential_ to crash your game is even more exploitable! It was the perfect storm of prior interest and new-found curiosity that lead me to investigate, reverse engineer, and gather a more complete than ever picture of the technical details surrounding this exploit.

This writeup was the result of that investigation.

## Dangling Pointer
After initializing the scene, whilst initializing the player, there is a code path that will initialize a global containing the incrediball model instance: `xModelInstance* s_incrediball_model_instance`. It achieves this by searching for a loaded asset with the hash `incrediball_ball`. If it is found, it will allocate an `xModelInstance` from it and store it into `s_incrediball_model_instance`. If it is not found, `s_incrediball_model_instance` will be set to `null`.

The aforementioned code path is ONLY taken when the scene player mapping contains at least one of the player tags listed in the table below. This means that for scenes with _none_ of these player tags, `s_incrediball_model_instance` will not be initialized.
- `PLS1`
- `PLS2`
- `PLS3`
- `PLS4`
- `PLS5`
- `PLS6`
- `PLS7`

I have no idea how costumes work in this game (sorry!), so I don't know what these actually end up doing, but theoretically would also correctly initialize the pointer:
- `PLSA`
- `PLSB`
- `PLSC`
- `PLSD`

This `xModelInstance` is allocated via `xModelInstanceAllocReuse`, which use a set of mem pools to store the backing memory for the object. These pools are re-initialized during each scene init. However, if the conditions above are met, meaning `s_incrediball_model_instance` is not re-initialized, it's old value will remain, pointing to an allocation made during the previous scene, from mem pools which have now been freed. This creates a dangling pointer -- it is referring to different memory than it thinks is there.

**NOTE**: This dangling pointer problem occurs _all the time_ throughout the game -- it just so happens that the memory corruption aspect is only reached in a particular, and entirely separate, manner.

## Memory Corruption
The memory corruption is triggered in tandem with the dangling pointer, dependent on the value of a certain global flag: `bool s_bubble_bowl_explosion_effect_active`. This flag is set whilst the explosion effect of Spongebob's Macho Spongebowl is active. The flag is _only_ cleared when the explosion effect decays and disappears. When switching scenes whilst this flag is set, it is _not_ cleared -- it will never be clear until another bowl explosion effect is active. The result is the ability to travel multiple scenes whilst the flag remains set, so long as another bowl explosion is not made active.

So, if the `s_incrediball_model_instance` pointer is dangling, and the `s_bubble_bowl_explosion_effect_active` flag is set, then an arbitrary memory write will occur every update(?). The values written are as follows:
```c
*(int*)(s_incrediball_model_instance + 0x38) = 0; // Probably always zero.
*(int*)(s_incrediball_model_instance + 0x58) = 0x40833333; // Actually a floating point, 4.1f
*(int*)(s_incrediball_model_instance + 0x5C) = 0x40833333; // Actually a floating point, 4.1f
*(int*)(s_incrediball_model_instance + 0x60) = 0x40833333; // Actually a floating point, 4.1f
```
(This also invokes `xModelBucket_Add(s_incrediball_model_instance)`, but I haven't investigated that any further.)

Due to the usage of mem pools, the value of `s_incrediball_model_instance` is actually quite determinant, and the outcomes then so -- the same setup should almost always result in the same outcome.

Though, the two most common outcomes are simply:
- Overwrite something important, crash later when other code finds out.
- Overwrite something not important, probably some mem pool backing memory that wasn't given out and no one is referencing, and nothing happens.

So _why_ and _how_ could Mindy Skip ever work? Miracles, probably.

## Mindy Skip
- Throw a macho bowl in No Cheese
- Pause whilst explosion effect is active
- Warp to Shell City Spongeball Challenge (`BL03`)
- Warp to Knucklehead (`PT02`)
- Spam buttons to OOB during a cutscene/taskbox _(sic)_
  - This quirk is completely unrelated to all of this, but it noramlly just result in a scene reset where Mindy continues talking, preventing progression.
- Mindy shuts up, race begins. Token requirement no longer required, even after warping back in.

Bowling in No Cheese is _just_ to set the `s_bubble_bowl_explosion_effect_active` flag for later. Warping to `BL03` (which has the scene player mapping `PLY4 PLS3` (Spongeball, Spongebob) meaning no dangling pointer occurs) is just to get a different `s_incrediball_model_instance` pointer that is more favorable. Warping now to `PT02` causes `s_incrediball_model_instance` to become dangling from the previous scene in `BL03`. As soon as the scene finishing loading and the transition in begins, the memory corruption is triggered.

The dangling pointer `s_incrediball_model_instance` is overlapping an `xEntAsset` for a random `SIMP` entity in some obtuse way (which asset?). It has over overwritten key parts of this structure:
- `xBaseAsset::baseType = 0x33; // Known as eBaseTypeTextBox. Fitting yes, relevant no.`
- `xBaseAsset::linkCount = 0x33;`
- `xBaseAsset::baseFlags = 0x4083; // This represents Enabled and Persistent`
- `xEntAsset::flags = 0x33;`
- `xEntAsset::subtype = 0x33;`
- `xEntAsset::pflags = 0x83;`
- `xEntAsset::moreFlags = 0x40;`
- `xEntAsset::surfaceID = 0x40833333;`

Although the memory corruption has occurred, nothing catastrophic has yet to happen. Going out-of-bounds and _resetting the scene_ is a crucial step in actualizing the corrupted values.

While resetting the scene, execution has ended up in `xBaseReset`. This is resetting a real live `xBase` to match it's `xBaseAsset` counterpart -- and this one is corrupted. The following corrupted values are overwritten:
- `xBase::baseFlags = 0x4087; // This represents Enabled, Persistent and Valid`
Now, winding down into `xEntReset`...
- `xEnt::flags = xEntAsset::flags;`
- `xEnt::moreFlags = xEntAsset::moreFlags;`

NOTE: The `xBase::baseType` is left _unmodified_. For Mindy Skip, it happens to be a `SIMP` asset and entity: `xBase::baseType = 0x0B;`

Execution proceeds past this, resetting everything remaining as normal. It eventually comes time to load the save data -- specifically, the persisted data. It does _just so happen_ that this corrupted entity has now been incidentally marked as `Persistent` in it's base flags. This is a problem, as the entity was _certainly_ not marked as such before, and thus it's unexpected appearance in this code path will cause an issue.

While the games goes through all objects `Persistent` objects, it comes across the corrupted one. As the base type `SIMP` is an entity, it will load two unintended bits from the data stream:
- `Enabled`
- `IsVisible`

This is a non-problem for this entity, but causes issues for _everything following_: the data stream has now been misaligned, and the rest of the persisted data will misinterpreted, setting unexpected values into other objects. Down the line, some `game_object:task_box`s associated with Mindy's dialog are reached. These persist an important value in the data stream: `ztaskbox::state_enum`. Normally at this point, these states would be either `0x01` (`STATE_DESCRIPTION`), or `0x02` (`STATE_REMINDER`). However, these misaligned reads end up with the values `0x40` or `0x80`! This is larger than the largest expected `0x06` (`MAX_STATE`), causing some code to be skipped as execution falls straight through, which will "complete" the taskbox, by:
- Setting it's state to `0xFFFFFFFF` (`STATE_INVALID`)
- Marking it as not enabled
- Dispatching the `eEventTaskBox_OnComplete` event

As the load finishes up, and the taskboxes disappear, the drive begins -- all while everything proceeds as normal. Almost. There was quite a bit of fallout damage from all the other things that went wrong during the use of the misaligned data stream. So, by simply re-warping to `PT02`, the remaining issues can be cleared up. Re-entering without any requirement works because the taskbox states, now completed, are saved before warping out, permanently removing them for that save.

## TL;DR
- There are actually two bugs working together to cause Bowl Storage, one of which is happening constantly throughout the game, just with no side effects.
- Mindy Skip uses save data misalignment to cause her taskboxes to be read as complete.
- You must warp to at least ONE scene that does NOT have a scene player mapping containing one of the player tags in the table at the top for the dangling pointer (and thusly, the memory corruption) to occur.
  - Example: Going back and forth between No Cheese, SD101, and Depression will never cause this.
  - Stages that will trigger the dangling pointer
    - `BB01`
    - `DE02`
    - `TT02`
    - `B101`
    - `TR01`
    - `B201`
    - `GG02`
    - `SC02`
    - `PT02`
    - `FB01`
    - `FB02`
    - `FB03`
    - `FB04`
    - `AM04`
- Mindy Skip is incredibly lucky, and it's discovery is very impressive.

# Addendum I
## Heap Allocation
The game's allocation strategy revolves around one central heap, allocated once from the underlying system at the very start of the game. This heap is divided into 12 "depths", of which the game pushes and pops in a particular order, such that all succeeding allocations become apart of that depth. This creates what are known as *arena allocator*s, where individual allocations to be freed are not tracked, but the underlying memory is freed all at once.

**NOTE: Although there are 12 depths, the game likely only ever makes use of 9 of them.**

The following table lists important depths with known-useful names
|  Depth  |  Name  |
|---------|--------|
|6|HIP Player Start|
|7|Scene Start     |
|8|HIP Start       |

## Memory Leak
During `zPlayerLoadHIP`, an allocation of `0x2B8` bytes (`696` bytes) is made for some(?) purpose. This allocation is made _just_ before the next depth is pushed, to `HIP Player Start`, which means this allocation lives at the previous depth (usually `5`).

However, memory allocated at depth `5` is _never_ reclaimed, because it is never popped off the depth stack. This means that for every `zPlayerLoadHIP` invocation, `0x2B8` bytes of memory leak. This call happens every `zSceneInit` -- ever scene load.

This causes all following allocation for that depth and following depths to no longer be deterministic in nature of the address handed out.

It should be the case that, _relatively_ speaking, any allocations following this would have an equal offset from each other every time, no matter if the address itself has shifted, because these allocations are still deterministic in order. However, due to alignment and underlying block size restrictions, occasionally there is slightly more (wasted) memory given to an allocation attempting to achieve a particular alignment.

This heap misalignment occurs on the 5th `zSceneInit` invocation, and every 4 invocations following, _always_.

## Rule of 4s
This heap misalignment, and the manipulation of it, is the _Rule of 4s_ -- every 4 invocations past the 5th will be misaligned. If the number of scene loads, minus one, is divisible by 4, then the heap is misaligned.

### Counting scene loads
- Reaching the main menu _ever_ counts as a scene load.
  - This includes during boot, after the FMVs.
- Seeing the loading screen counts as a load.

Being misaligned is the _uncommon_ case, whilst being aligned is the _common_ case.

## Mindy Skip Revisited
The original text had failed to go over an important possibility of this trick: complete failure, and crashing soon after the race beginning. Now known, this was the result of heap misalignment.

To ensure with absolute certainty that Mindy Skip will not fail, the Rule of 4s must be employed to ensure performance for the _common_ case -- the _uncommon_ case will cause the failure and latent crash. This means, that upon reaching `PT02` to perform Mindy Skip, your scene load count, minus one, must **NOT** be divisible by 4.
