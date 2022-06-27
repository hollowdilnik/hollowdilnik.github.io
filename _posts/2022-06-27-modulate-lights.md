---
layout: post
title: "Making modulate lights work without mobile HDR"
---
>**TL;DR:** Change `BF_Zero` to `BF_SourceColor` in [this line](https://github.com/EpicGames/UnrealEngine/blob/1e5926084bbf386041103735ed6c2ab27bc1c1ee/Engine/Source/Runtime/Renderer/Private/MobileBasePass.cpp#L430)  and rebuild the engine. Divide the emission of your modulate material by half. Disable "Render after DOF". Now your modulate materials will work without mobile HDR.

![Lights](/assets/img/lights.jpg){:class="img-responsive" max-width=500}

If you have tried to make cheap fake lights for mobile VR using a modulate material in Unreal Engine, you've probably noticed that your modulate materials don't work and wondered why. So what's going on?

Modulate materials take the current scene color and multiply it by the shader output to darken or brighten whatever was rendered behind them. This is implemented on a very low level by specifying the [blend operation](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/man/html/VkBlendOp.html#_description) for the current material. There are five basic blend operations that are supported on most hardware: add, subtract, reverse subtract, min and max. But wait, where is the multiplication? Turns out that these blend operations allow for some flexibility. For example, the blend operation "add" actually performs a linear combination of the source and destination values:
```
DestColor := SourceColor * SourceBlendFactor + DestColor * DestBlendFactor
```
where `SourceColor` is the output of your shader, `DestColor` is the color in the render target, and the corresponding blend factors can be chosen from a [list of possible blend factors](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/man/html/VkBlendFactor.html#_description).

UE implements its modulate materials by setting the `SourceBlendFactor` to `DestColor` and `DestBlendFactor` to zero:
```
DestColor := SourceColor * DestColor + DestColor * 0
```
which achieves exactly what we want.

Unfortunately, `SourceColor`, `DestColor` and the corresponding blend values are getting clamped to [0, 1] if your render target is using a fixed-point format (i.e. if you have mobile HDR turned off in UE). So if we try to make modulate lights, and set `SourceColor` to something like (3,3,3,1) to brighten things up, it would get clamped and we will see `DestColor := 1 * DestColor`, which doesn't get us anywhere. Our lights are completely invisible, as if they are not supported on this hardware.

So is that it? We've hit a hardware limitation and all hope is lost?.. Of course not!

Remember the expression for the add blend operation? UE sets the `DestBlendFactor` to zero because we don't need it under normal circumstances. Well, developing for mobile VR is far from "normal circumstances"! If we set the `DestBlendFactor` to `SourceColor`, we can get this blend operation instead:
```
DestColor := SourceColor * DestColor + DestColor * SourceColor
```
which is equivalent to
```
DestColor := (2 * SourceColor) * DestColor
```
We have doubled our emission value, which is effectively equivalent to clamping the `SourceColor` to [0, 2] instead of [0, 1]. Now we can make the scene brighter, we just need to change the blend factor. Unfortunately, the easiest way to do it is to modify the engine source code. Find [this line](https://github.com/EpicGames/UnrealEngine/blob/1e5926084bbf386041103735ed6c2ab27bc1c1ee/Engine/Source/Runtime/Renderer/Private/MobileBasePass.cpp#L430) in your version of Unreal Engine (I'm using 4.27) and change `BF_Zero` to `BF_SourceColor`. The line should now look like this:
```cpp
DrawRenderState.SetBlendState(TStaticBlendState<CW_RGB, BO_Add, BF_DestColor, BF_SourceColor>::GetRHI());
```
Rebuild the engine and enjoy working (but still limited) modulate materials that do not require mobile HDR!

---

There are a couple of points left to discuss:

You need to disable "Render after DOF" in the material properties, or the material wouldn't work on Quest.

Since we've effectively doubled the emission on mobile, we need to scale it back in the material graph (multiply by 0.5). Note that now 0.5 is the value resulting in zero opacity.

The easiest way to make the lights work regardless of where you are relative to the light mesh, is to use a mesh with inwards-facing faces (like `Sphere_inversenormals` from the engine content, but probably something with less triangles) and disable depth testing in the material. This way there will always be a surface to render the light on if you are seeing the light, so it won't flicker or randomly disappear. This is an easy trick, but it is probably not ideal for performance. If this is a problem, you might need to come up with a way to cull these lights manually.

"Being able to render modulate materials with emissive value 2 is cool and all, but I want more power!". The only way I see to boost the emission even further is to stack these lights on top of each other, increasing overdraw. The good thing is that this emission boost is exponential, as we can double the brightness after each draw. So if you have 1 light, you get up to 2x brightness, 2 overlapping lights give you up to 4 brightness, and to get 16x brightness, you need 4 overlapping lights. There is also a way to make an essentially free "brightness boosting" fragment shader, so the only additional cost would come from repeated blending and not from fragment shader cost.

I am not aware of any cheaper way to achieve the same effect without using mobile HDR. However, the `VK_B10G11R11_UFLOAT_PACK32` color format sounds like a promising alternative to the usual RGBA8 color format. We get floating point values packed into the same 32 bits as the LDR values. Is this going to be the key to mobile HDR for the cost of mobile LDR?

## Relevant links
1. Possible blend operations in Vulkan: [https://www.khronos.org/registry/vulkan/specs/1.3-extensions/man/html/VkBlendOp.html#_description](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/man/html/VkBlendOp.html#_description)
2. Possible blend factors in Vulkan: [https://www.khronos.org/registry/vulkan/specs/1.3-extensions/man/html/VkBlendFactor.html#_description](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/man/html/VkBlendFactor.html#_description)
3. The line that defines the blending of modulate materials on mobile: [https://github.com/EpicGames/UnrealEngine/blob/1e5926084bbf386041103735ed6c2ab27bc1c1ee/Engine/Source/Runtime/Renderer/Private/MobileBasePass.cpp#L430](https://github.com/EpicGames/UnrealEngine/blob/1e5926084bbf386041103735ed6c2ab27bc1c1ee/Engine/Source/Runtime/Renderer/Private/MobileBasePass.cpp#L430)
4. Normal reconstruction from depth: [https://wickedengine.net/2019/09/22/improved-normal-reconstruction-from-depth/](https://wickedengine.net/2019/09/22/improved-normal-reconstruction-from-depth/)