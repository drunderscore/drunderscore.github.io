---
title: "TF2: The Sniper Dot may be Lying to You"
category: Reverse Engineering
tags:
- tf2
- networking
- source engine
excerpt: An investigation into why the Sniper Dot isn't true to itself, combined with a few quick lessons into the depth of the Source Engine.
---

Recently, the following clip was shared and I happened to stumble across it in a Discord chat. Take a look:
<iframe width='640' height='360' style='border: none;' src='https://medal.tv/clip/1ITHYuA2iKPQiZ/vpNnWAIp8?invite=cr-MSxwdmQsLA' allow='autoplay' allowfullscreen></iframe>

In this clip, you can see the sniper dot appear and disappear as the player walks between lobby and lower lobby near shutter on Snakewater, and the shutter is supposedly closed! This is a pretty significant issue for both teams: you either only _occasionally_ see the sniper dot (losing out on valuable information), or you're exposing the sniper dot when it shouldn't be visible (disclosing valuable information)! Want the hell is going on here?

To properly explain this, we'll have to quickly go over a few concepts that are implemented in game engines, but more importantly, how these techniques are used and implemented in the Source Engine.

## [PVS](https://developer.valvesoftware.com/wiki/PVS)
PVS stands for _Potentially Visible Set_, and is a generic technique used to calculate what may or may not be visible from a certain position.

In the Source Engine, PVS utilizes [visleaves](https://developer.valvesoftware.com/wiki/Visleaf) to quickly approximate what is visible from a certain position. This is all calculated at map compile-time by the `vvis` compiler. This implementation of PVS has two incredibly important uses within the Source Engine.

### World Culling
There is no reason to render what the player cannot see. In lieu of this, the Source Engine utilizes PVS to determine what can actually be seen. The entire world is not always rendered, but rather only sections of it that you could see from your position. Those sections that are not rendered are considered _culled_. This includes world brushes, as well as opaque and translucent entities.

### Network Transmission
For a similar reason as described above, there is no reason to transmit what the player cannot see (in fact, it's quite important in the realm of cheat prevention, such as an ESP to see player's through walls). One of the ways that Source Engine can determine if an entity should be transmitted is by using PVS. If you cannot see the entity, there is no reason to waste the network bandwidth to continuously update it. An entity that is no longer being transmitted is considered _dormant_.

## Frustum Culling
Alongside culling the world based on PVS, another technique is being employed here: _frustum culling_. The frustum is a shape that visualizes what the camera sees -- everything inside can be seen, and everything outside cannot be seen. Brushes and entities that would be rendered outside of the frustum are culled.

## Entities & Network Properties
Server-side entities are governed by the server, with their state being transmitted over the network, keeping all clients in sync with the world. This is done by using _data tables_.

A data table is a structure that contains all the properties for a particular entity, along with their types and other information required for de/serialization. Both the client and server have an identical copy, meaning they do not require any additional meta-information to begin communicating.

Skipping ahead a bit (and only speaking on what is relevant here), one of the types that can be transmitted are floating-point types, at a maximum of `32` bits. A developer can set arbitrary precision for floating-point properties, which can be used to reduce network bandwidth where full-precision is not necessary.

### The TF2 Player Data Table
The data table for the player, known as `DT_TFPlayer`, contains two special _embedded_ data tables: `DT_TFLocalPlayerExclusive` and `DT_TFNonLocalPlayerExclusive`.

The `DT_TFLocalPlayerExclusive` data table is only sent to the local player, whereas the `DT_TFNonLocalPlayerExclusive` data table is sent to everyone else.

This distinction is again used to reduce network bandwidth for information that is irrelevant if you are not the local player, but can also once again be considered a cheat prevention, by omitting data that isn't normally visible, but could be used by cheaters to gain an advantage. It's used elsewhere too, such as in the `DT_LocalPlayerExclusive` data table, which is _embedded_ in the `DT_BasePlayer` data table.

Within the `DT_TFNonLocalPlayerExclusive` data table exists an important property: `m_angEyeAngles`. This is a two-component vector which represents at which angles the player is looking (pitch and yaw). However, because these frequently update, and full-precision isn't truly necessary if you are not the local player, the developers decided to cut down the number of bits used to serialize these values. The pitch can only use `8` bits, whilst the yaw can only use `10`. This approximation is totally acceptable for _most_ situations, but still creates some odd scenarios -- for example, the pitch of non-local player's can't equal `0`, because the lack of precision makes it impossible.

## What is the Sniper Dot Anyhow?
The sniper dot is known as `env_sniperdot`. It is a simple, model-less entity that replaces it's drawing routine entirely with one that simply draws two sprites: the outer and inner dot (although seldom-known, the sniper dot will grow it's inner dot as the sniper rifle charge increases to 100%).

The position of the dot is calculated in 3 different ways:
- _If the dot's owner is the local player, then the position is directly in front of your face. **We don't touch on this at all.**_
- If the dot's owner _is not_ dormant, then the position is calculated by tracing a ray from the owner's eye position along their eye angles.
- If the dot's owner _is_ dormant, then the position is calculated by taking the position of the dot entity. This position is controlled entirely by the server, and calculated by a _similar_ (but not identical...) ray-trace from the owner's eye position along their eye angles.

## The Problem(s)
Unfortunately for Valve (and us), there isn't just a single problem at hand here...

- Due to the reduced precision of eye angles from non-local players as described above, when the client calculates the position of the dot, it's result is somewhat different than what the server calculates using it's full-precision eye angles. This results in deviation when tracing the ray from the player's eye position, and becomes even further disjointed with the server as the distance increases. It may even collide with a brush or entity prematurely on the client, where it hasn't on the server.
[![A pink cube and a red cube against a backdrop. The pink cube is positioned above the red cube, quite a bit higher.](/images/2023-12-13/reduced_angle_precision.png)](/images/2023-12-13/reduced_angle_precision.png)
> The pink cube is the server's position, whilst the red cube is the position the sniper dot is actually being rendered.

- Now, because Valve, the supposed "render" position of the entity isn't actually located where the sprite is being rendered, but rather is identical to the position of the entity itself. This creates a new problem: culling! Culling is entirely based on the render position of the entity -- if it is not within your frustum, it is not rendered. However, because the render position is incorrect, it can easily become the case where the sprite is positioned within your frustum, but the entity is not. This causes the entity to be culled, and will not be drawn, even though it would have been visible on the screen.
<video width="640" height="360" loop controls src="/images/2023-12-13/incorrect_render_origin.mp4"></video>
> The red cube (and therefore, sniper dot) disappear as the pink cube goes out of view, even though the dot was still visible.

- **Now**, (once again) because Valve, when the client and server both go to ray-trace the sniper dot, to see which brush or entity it should be positioned up against, they end up differing. The client considers certain solids that the server does not, resulting in further desync from the entity's position and the rendered sprite's position. This makes it trivially easy for the issue above to appear.
[![Snakewater lower lobby, near shutter, with lower lobby and shutter in view. The red cube is positioned at the shutter, whilst the pink cube is positioned at the back of the wall perpendicular to the shutter, next to the ramp to lower.](/images/2023-12-13/differing_ray_traces.png)](/images/2023-12-13/differing_ray_traces.png)
> The client's ray-trace properly stopped at the shutter, drawing the sniper dot there. However, the server's ray-trace passed through the shutter, hitting the back of the wall.

- **AND NOW** comes another problem: If you recall, there was another local-player only data table I mentioned: `DT_LocalPlayerExclusive`. This table contains an important property named `m_vecViewOffset`. This is a 3-component vector that holds the view offset of the player. This changes when the offset of their view changes, such as when crouching. It is combined with the player's position to calculate the eye position. The keen-eyed readers may have already realized the issue: This is _local-player only_, and yet other clients are attempting to use it when calculating the eye position, which is then in turn used as the starting position of the ray-trace for the sniper dot! This results in the player's position being combined with a zero-vector, resulting in the wrong position when the player crouches, and thus the dot never moves even though it totally has.
<video width="640" height="360" loop controls src="/images/2023-12-13/vec_view_offset_missing.mp4"></video>
> As the player crouches, the sniper dot's position also moves (and so would the shot). Despite that, the rendered sniper dot does not move.

_Finally_, if you combine the miscalculated server ray-trace WITH the change in position relative to the dormancy of the owner, you get this: the sniper dot being rendered THROUGH the shutter door even though it's closed! I am almost certain this is what is observed in the introduction video.
[![Snakewater lower lobby, from the perspective of upper lobby, facing the back wall. The shutter is not in view, but it is closed. The BLU sniper dot is seen on that back wall.](/images/2023-12-13/sniper_dot_through_shutter.png)](/images/2023-12-13/sniper_dot_through_shutter.png)

## Conclusion
I'm not sure what else you expect me to say here other than: Yeah it's broken. Multiple issues come together to make it an incredibly inaccurate piece of visualization.

As for the solution, my suggestion to Valve (or to whomever is developing TF2 these days) are these two options:
- **Do away with the prediction entirely**. The client simply doesn't have enough information to accurately predict this (imprecise eye angles, no view offset), and it would make the sniper dot perfectly accurate -- as long as the server's ray-trace masks and filters are corrected to match what the client currently does. This _would_ make prior demos somewhat inaccurate (the sniper dot's position from the ray-trace is already wrong), but personally I'm not sure how much I would care about a sniper dot in a demo -- especially compared to live gameplay.
- **Keep the prediction**, but fix and improve the accuracy.
  - Increase the precision of `m_angEyeAngles`
  - Move `m_vecViewOffset` out of `DT_LocalPlayerExclusive`
  - Correct the render origin of the sniper dot
  - Fix the server's mask and filters when ray-tracing

### Thanks
- [Dogdayboy](https://rgl.gg/Public/PlayerProfile?p=76561198874506663) for the original clip (in collaboration with [Fireball](https://rgl.gg/Public/PlayerProfile?p=76561198087251749) apparently).
- [M17](https://rgl.gg/Public/PlayerProfile?p=76561198124771378) for sharing the clip.
