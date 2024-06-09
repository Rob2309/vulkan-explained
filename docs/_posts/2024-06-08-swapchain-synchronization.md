---
layout: post
title:  "Vulkan Swapchain synchronization explained"
date:   2024-06-08 12:56:22 +0200
categories: vulkan synchronization
permalink: /swapchain-sync-explained
---


## No, the swapchain is not part of any pipeline stage

Synchronization in Vulkan is relatively complex and takes a while to intuitively understand.
An even more complex topic is properly synchronizing with a Swapchain, as it has quirks that
most tutorials do not properly explain.
As I have seen many people struggling with this topic and using incorrect synchronization,
I decided to explain this fundamental topic in more detail.
If you think that `vkQueuePresentKHR` is part of the `ColorAttachmentOutput` pipeline stage,
this post is for you.

**Disclaimer**: While I would say that I have a very thorough understanding of Vulkan
synchronization and can intuitively use it, I cannot guarantee that I do not make any mistakes,
so take the following chapters as my best effort to explain what I think is correct.

Do note that I expect a basic understanding of Vulkan synchronization in order to follow
this post.

## The question

Lets take the following basic operation that is used in pretty much any rendering loop:

```c++
// Acquire the next swapchain image
auto imageIndex = device.acquireNextImageKHR(swapchain, UINT64_MAX, acquireSemaphore, nullptr);

// Transition the image to COLOR_ATTACHMENT_OPTIMAL
vk::ImageMemoryBarrier2 toColorAttachment {
    ...
    .srcStageMask = vk::PipelineStageFlagBits::eColorAttachmentOutput,
    .srcAccessMask = {},
    .dstStageMask = vk::PipelineStageFlagBits::eColorAttachmentOutput,
    .dstAccessMask = vk::AccessFlagsBits::eColorAttachmentWrite,
    .oldLayout = vk::ImageLayout::eUndefined,
    .newLayout = vk::ImageLayout::eColorAttachmentOptimal,
    .image = swapchainImages[imageIndex],
};
commandBuffer.cmdPipelineBarrier2(...);

// Do rendering stuff
...
...

// transition to PRESENT_SRC
vk::ImageMemoryBarrier2 toPresentSrc {
    ...
    .srcStageMask = vk::PipelineStageFlagBits::eColorAttachmentOutput,
    .srcAccessMask = vk::AccessFlagBits::eColorAttachmentWrite,
    .dstStageMask = vk::PipelineStageFlagBits::eBottomOfPipe,
    .dstAccessMask = {},
    .oldLayout = vk::ImageLayout::eColorAttachmentOptimal,
    .newLayout = vk::ImageLayout::ePresentSrcKHR,
    .image = swapchainImages[imageIndex],
};
commandBuffer.cmdPipelineBarrier(...);

// submit command buffer
vk::PipelineStageFlags waitMask = vk::PipelineStageFlagBits::eColorAttachmentOutput;
vk::SubmitInfoKHR submitInfo {
    ...
    .waitSemaphoreCount = 1,
    .pWaitSemaphores = &acquireSemaphore,
    .pWaitDstStageMask = &waitMask,
    .signalSemaphoreCount = 1,
    .pSignalSemaphores = &renderFinishedSemaphore,
};
graphicsQueue.submit(...);

...

// present the image
vk::PresentInfoKHR presentInfo {
    ...
    .waitSemaphoreCount = 1,
    .pWaitSemaphores = &renderFinishedSemaphore,
    .swapchainCount = 1,
    .pSwapchains = &swapchain,
    .pImageIndices = &imageIndex,
};
graphicsQueue.queuePresentKHR(presentInfo);
```

You've probably seen this kind of rendering code many times in many different tutorials and
just took for granted that you have to use the specified pipeline stages and access flags.
The problem is that I have never seen a tutorial that actually explains **why** you have to
use these flags, as it isn't that obvious.

Let's first take a look at our `submitInfo`:
```c++
vk::PipelineStageFlags waitMask = vk::PipelineStageFlagBits::eColorAttachmentOutput;
vk::SubmitInfoKHR submitInfo {
    ...
    .waitSemaphoreCount = 1,
    .pWaitSemaphores = &acquireSemaphore,
    .pWaitDstStageMask = &waitMask,
    .signalSemaphoreCount = 1,
    .pSignalSemaphores = &renderFinishedSemaphore,
};
graphicsQueue.submit(...);
```

Here we tell the GPU to not execute the `ColorAttachmentOutput` stage before the `acquireSemaphore` has been signalled.
Sounds pretty logical, as this is the stage were we want to write to our color attachment.
However, we still have to transition the image to the correct layout for rendering, thus we need a pipeline barrier:
```c++
// Transition the image to COLOR_ATTACHMENT_OPTIMAL
vk::ImageMemoryBarrier2 toColorAttachment {
    ...
    .srcStageMask = vk::PipelineStageFlagBits::eColorAttachmentOutput,
    .srcAccessMask = {},
    .dstStageMask = vk::PipelineStageFlagBits::eColorAttachmentOutput,
    .dstAccessMask = vk::AccessFlagsBits::eColorAttachmentWrite,
    .oldLayout = vk::ImageLayout::eUndefined,
    .newLayout = vk::ImageLayout::eColorAttachmentOptimal,
    .image = swapchainImages[imageIndex],
};
commandBuffer.cmdPipelineBarrier2(...);
```

Obviously our `dstStageMask` is `ColorAttachmentOutput`, as we want to transition the image to the correct layout **before** writing to it in our rendering commands.
But why are we specifying `srcStageMask` as `ColorAttachmentOutput` as well?
Is the presentation engine part of `ColorAttachmentOutput`?
This would sound pretty logical at first, but this is actually completely wrong!
So let's dive into the situation with Vulkan Swapchain synchronization.

## The answer

We acquire the next swapchain image with:

```c++
auto imageIndex = device.acquireNextImageKHR(swapchain, UINT64_MAX, acquireSemaphore, nullptr);
```

After this, we obviously need to transition the image to the ColorAttachmentOptimal layout in order to allow rendering to it. But first, we have to wait for the presentation engine to have finished reading the image.  
As the spec states [[1]]:
> The presentation engine may not have finished reading from the image at the time it is acquired, so the application must use semaphore and/or fence to ensure that the image layout and contents are not modified until the presentation engine reads have completed.

[1]: https://registry.khronos.org/vulkan/specs/1.3-extensions/html/chap34.html#vkAcquireNextImageKHR

This means that we have to wait on the semaphore before transitioning the image (and then writing to it when rendering).
But how do we tell our layout transition to not start before the semaphore is signalled, if there is no proper pipeline stage to wait on?
The answer is: *dependency chains*!

A dependency chain occurs when a synchronization operation A has a destination stage mask that has stages in common with the source stage mask of a following synchronization operation B.
When such a chain occurs, anything in the srcStage of A *happens before* anything in the dstStage of B [[2]].
This sounds pretty logical at first but is actually a very important guarantee in the spec for our case.
Let's take a look at a simple example:

[2]: https://registry.khronos.org/vulkan/specs/1.3-extensions/html/chap7.html#synchronization-dependencies

```
Synchronization A: srcStage=TRANSFER, dstStage=COLOR_ATTACHMENT_OUTPUT
Synchronization B: srcStage=COLOR_ATTACHMENT_OUTPUT, dstStage=COMPUTE_SHADER
```

In this case, since A.dstStage has common bits with B.srcStage, we guarantee that anything in TRANSFER happens before COMPUTE_SHADER.
This is a fundamental piece of the specification that we need to understand for the image acquisition synchronization.

Let's take a look at the acquisition again:

```c++
auto imageIndex = device.acquireNextImageKHR(swapchain, UINT64_MAX, acquireSemaphore, nullptr);

// Transition the image to COLOR_ATTACHMENT_OPTIMAL
vk::ImageMemoryBarrier2 toColorAttachment {
    ...
    .srcStageMask = vk::PipelineStageFlagBits::eColorAttachmentOutput,
    .srcAccessMask = {},
    .dstStageMask = vk::PipelineStageFlagBits::eColorAttachmentOutput,
    .dstAccessMask = vk::AccessFlagsBits::eColorAttachmentWrite,
    .oldLayout = vk::ImageLayout::eUndefined,
    .newLayout = vk::ImageLayout::eColorAttachmentOptimal,
    .image = swapchainImages[imageIndex],
};
commandBuffer.cmdPipelineBarrier2(...);

// submit command buffer
vk::PipelineStageFlags waitMask = vk::PipelineStageFlagBits::eColorAttachmentOutput;
vk::SubmitInfoKHR submitInfo {
    ...
    .waitSemaphoreCount = 1,
    .pWaitSemaphores = &acquireSemaphore,
    .pWaitDstStageMask = &waitMask,
    .signalSemaphoreCount = 1,
    .pSignalSemaphores = &renderFinishedSemaphore,
};
graphicsQueue.submit(...);
```

Can you spot the dependency chain here? The semaphore wait operation of `graphicsQueue.submit` forms synchronization operation **A** with `srcStage=???` and `dstStage=eColorAttachmentOutput`.
`???` is the imaginary pipeline stage that the presentation engine operates on.  
The pipeline barrier `toColorAttachment` forms synchronization operation **B** with `srcStage=eColorAttachmentOutput` and `dstStage=eColorAttachmentOutput`.
As you can probably see, since `A.dstStage` and `B.srcStage` are identical, a dependency chain is formed.  
With this we ensure that the presentation engine is finished with reading the swapchain image (semaphore is signalled), before we transition the image for further rendering (pipeline barrier). We have achieved this without needing to specify the non-existing hypothetical stage "PRESENTATION_ENGINE".  
This is, in fact, the only way to properly synchronize with the swapchain acquisition operation and the spec also states this [[3]]:

> Note  
> When the presentable image will be accessed by some stage S, the recommended idiom for ensuring correct synchronization is:
> 
> The VkSubmitInfo used to submit the image layout transition for execution includes vkAcquireNextImageKHR::semaphore in its pWaitSemaphores member, with the corresponding element of pWaitDstStageMask including S.
> 
> The synchronization command that performs any necessary image layout transition includes S in both the srcStageMask and dstStageMask.

As you can see, this is exactly what we do in the code above. Do note that the actual stage that we use as semaphore wait stage and pipeline barrier source stage in `toColorAttachment` is actually irrelevant for the sake of correctness, as we only use this stage to setup a dependency chain, but as the spec recommends this setup, we will use it in this way.

[3]: https://registry.khronos.org/vulkan/specs/1.3-extensions/html/chap34.html#vkAcquireNextImageKHR

## And what about `vkQueuePresentKHR`?

The second part of the swapchain synchronization is thankfully much less complicated. We only need to ensure that we have finished writing to the swapchain image and transitioned it to PRESENT_SRC layout before the presentation image tries to read from it. Let's take a look at the presentation part of the code again:

```c++
// transition to PRESENT_SRC
vk::ImageMemoryBarrier2 toPresentSrc {
    ...
    .srcStageMask = vk::PipelineStageFlagBits::eColorAttachmentOutput,
    .srcAccessMask = vk::AccessFlagBits::eColorAttachmentWrite,
    .dstStageMask = vk::PipelineStageFlagBits::eBottomOfPipe,
    .dstAccessMask = {},
    .oldLayout = vk::ImageLayout::eColorAttachmentOptimal,
    .newLayout = vk::ImageLayout::ePresentSrcKHR,
    .image = swapchainImages[imageIndex],
};
commandBuffer.cmdPipelineBarrier(...);

// submit command buffer
vk::SubmitInfoKHR submitInfo {
    ...
    .signalSemaphoreCount = 1,
    .pSignalSemaphores = &renderFinishedSemaphore,
};
graphicsQueue.submit(...);

...

// present the image
vk::PresentInfoKHR presentInfo {
    ...
    .waitSemaphoreCount = 1,
    .pWaitSemaphores = &renderFinishedSemaphore,
    .swapchainCount = 1,
    .pSwapchains = &swapchain,
    .pImageIndices = &imageIndex,
};
graphicsQueue.queuePresentKHR(presentInfo);
```

As you can see, with the `toPresentSrc` barrier, we ensure that the transition only happens after we have finished rendering (barrier `srcStageMask` is `eColorAttachmentOutput`).
Since we rely on the `renderFinishedSemaphore` to be signalled, we do not need to specify a specific destination stage for the barrier, thus we just use `eBottomOfPipe`.  
But we do have to make our writes **visible** to the presentation engine, which we would usually do with `dstAccessMask` of the image memory barrier.
However, since the presentation engine operates in **no** pipeline stage, there also is no access mask to specify.
How do we solve this problem?
The answer luckily is: we don't have to.  
The spec states for `vkQueuePresentKHR` [[4]]:
> Any writes to memory backing the images referenced by the pImageIndices and pSwapchains members of pPresentInfo, that are available before vkQueuePresentKHR is executed, are automatically made visible to the read access performed by the presentation engine. This automatic visibility operation for an image happens-after the semaphore signal operation, and happens-before the presentation engine accesses the image.

[4]: https://registry.khronos.org/vulkan/specs/1.3-extensions/html/chap34.html#vkQueuePresentKHR

This means, we only need to make our writes available, which is done via the `srcAccessMask`.
The presentation engine automatically makes sure the any **available** writes are made **visible**.

## Conclusion

I hope I could make the actual mechanisms behind swapchain synchronization more clear, as it is really not that trivial when starting out with Vulkan and synchronization.
There are a lot of tutorials out there that do not properly explain the reasoning behind the synchronization choices, resulting in many people making incorrect assumptions.

To summarize swapchain synchronization:
- Swapchain operations do not operate in any pipeline stage and have no corresponding access flags, no matter what any tutorial/redditor tells you.
- `vkAcquireNextImageKHR` can only be waited on via dependency chains, which is done via `waitDstStages` of `VkSubmitInfo` and `srcStageMask` of a pipeline barrier.
These fields must have stages in common.
- `vkQueuePresentKHR` makes all **available** writes automatically **visible**, meaning we do not have to specify a specific `dstStage` and `dstAccessMask`

