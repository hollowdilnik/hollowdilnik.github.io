---
layout: post
title: "Fixing aliasing / flickering widgets in UE4"
---
> TL;DR:

> Enable `bAutoGenerateMips` for the `RenderTarget` of the `UWidgetComponent` and set its `Filter` to `TF_Trilinear`.

> Call `RenderTarget->UpdateResourceImmediate(false)` after **every** update of the widget.

> In the widget material, change the `Sampler Source` for `SlateUI` to `From texture asset` or enable trilinear / aniso-linear filtering for `TEXTUREGROUP_World`.

[UMG](https://docs.unrealengine.com/4.27/en-US/InteractiveExperiences/UMG/) is great for UI widgets.
However, `UWidgetComponent`&#x200B;s that display these widgets in the world do not look good, especially in VR.
They are rendered at their designed resolution with no filtering, resulting in extreme aliasing and flickering.
This can be fixed by generating mipmaps and filtering, but enabling it in UE4 is non-trivial *(why is it not a single checkbox?)*.

This note is mostly targeted towards VR and Oculus Quest headsets, but is still relevant to other devices and applications that use `UWidgetComponent`&#x200B;s.

<!-- TODO: GIF THAT SHOWS BEFORE VS AFTER -->

## Generating mipmaps for UMG widgets
The render target is created inside `UWidgetComponent::UpdateRenderTarget`.
Right after [its creation](https://github.com/EpicGames/UnrealEngine/blob/4.26/Engine/Source/Runtime/UMG/Private/Components/WidgetComponent.cpp#L1696) is the right place to setup the render target parameters:

```cpp
...
RenderTarget->bAutoGenerateMips = true;
RenderTarget->Filter = TF_Trilinear;
...
```

The simplest way to do it is to override this function in a child class, copy its implementation and add the setup there.
Further, you can expose these parameters to blueprints, making the original `UWidgetComponent` completely obsolete.
Exposing `RenderTarget->LODGroup` is also a good idea in case you want to setup filtering in texture groups instead.

Changing these parameters in a different place (or during the object lifetime) is messy.
With the current implementation of render targets, the number of mips is not updated after changing `bAutoGenerateMips`, so you'd need to create a new render target object.
You would also need to update the `SlateUI` parameter of the material instance if you change the render target.

Simply setting `bAutoGenerateMips = true` is not enough!
While the render target will be updated by the widget component, the mips would not be regenerated.
We need to trigger mip generation manually after each update of the widget.

The widget is drawn to the render target inside the `UWidgetComponent::DrawWidgetToRenderTarget` method.
We just need to call `UpdateResourceImmediate` here in order to regenerate mips:

```cpp
void UFilteredWidgetComponent::DrawWidgetToRenderTarget(float DeltaTime)
{
	Super::DrawWidgetToRenderTarget(DeltaTime);
	
	if (RenderTarget && RenderTarget->bAutoGenerateMips)
	{
		RenderTarget->UpdateResourceImmediate(false);
	}
}
```

I am not 100% confident this is the right place to call it, because drawing widgets and regenerating mips are async operations.
However, I believe that both will be executed on the same rendering thread, and as long as `UpdateResourceImmediate` is called after `DrawWidgetToRenderTarget`, mipmap generation should execute after the widget is drawn to the render target.
Although this approach has worked fine for me, I don't have much knowledge of UE4 rendering pipeline, so please correct me if I am wrong here.

Finally, make sure to add `"Slate"` to the `PublicDependencyModuleNames` in your `.Build.cs` file, because the source code of `UWidgetComponent::UpdateRenderTarget` depends on it.

## Enabling filtering
These modifications of `UpdateRenderTarget` and `DrawWidgetToRenderTarget` are enough to enable mipmapping and filtering of the render target.
However, filtering would only work if the material is set up correctly.
The default widget material (&#x200B;`Widget3DPassThrough`&#x200B;) uses a shared sampler for the `SlateUI` texture (&#x200B;`Sampler Source` is set to `Shared: Clamp`&#x200B;).
This would override the filtering settings we set up before, and would use default filtering from `TEXTUREGROUP_World`.
Unless you want to change the default filtering for the whole project to trilinear / aniso-linear, set the `Sampler Source` to `From texture asset`.

## Caveats
**Performance:** this approach can get pretty heavy.
Generating mipmaps for fifteen 1k render targets costs me about 9ms on Oculus Quest 2 (0.6ms per widget).
Updating one widget every frame is probably fine if it is really needed, but anything more than that can quickly become too expensive.
If you need to update a lot of widgets, you should probably spread out the updates across multiple frames.
A couple of widgets with manual redraw are not going to be a big deal though.

**Unreal Engine version:** Currently I use UE 4.26.2 from the launcher.
I did not check if these modifications work the same way in other engine versions.
Even if they are broken, it should be straightforward to do something similar in other engine versions.

**Blend Mode:** This approach works great with opaque, translucent, additive and AlphaComposite materials.
However, masked materials do not look great; the opacity mask gets messed up by filtering.
It might be possible to do some shader magic to make masked materials look good, but it is out of scope of this note.

<!-- TODO: GIFS WITH DIFFERENT FILTERING -->

<!-- TODO: GIFS WITH DIFFERENT BLEND MODES -->

## Possible extensions
Not sure if it is worth it, but it might be possible to gain additional performance by limiting the number of mips generated for the render target.
It would, however, require some modification of the engine code since neither `NumMips` nor `GetNumMips()` can be set / overridden in child classes.

Also, mipmap generation on Oculus Quest currently uses compute shaders instead of `glGenerateMipmap` or `vkCmdBlitImage`, but I have no idea if using them would be faster than compute shaders or whether it is even possible on Quest.

## Relevant links
1. A shorter writeup on generating mipmaps for render targets in UE4 [https://alanedwardes.com/blog/posts/generating-mipmaps-for-render-textures-ue4/](https://alanedwardes.com/blog/posts/generating-mipmaps-for-render-textures-ue4/)
1. Oculus blog post discussing aliasing in textures with a lot of relevant tips [https://developer.oculus.com/blog/common-rendering-mistakes-how-to-find-them-and-how-to-fix-them/](https://developer.oculus.com/blog/common-rendering-mistakes-how-to-find-them-and-how-to-fix-them/)
1. Docs on texture groups, including filtering parameters [https://docs.unrealengine.com/4.27/en-US/RenderingAndGraphics/Textures/SupportAndSettings/](https://docs.unrealengine.com/4.27/en-US/RenderingAndGraphics/Textures/SupportAndSettings/)