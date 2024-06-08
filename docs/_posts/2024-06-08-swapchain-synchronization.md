---
layout: post
title:  "Vulkan Swapchain synchronization explained"
date:   2024-06-08 12:56:22 +0200
categories: vulkan synchronization
permalink: /swapchain-sync-explained
---


# Vulkan Swapchain synchronization explained

Regular synchronization in Vulkan is relatively complex and takes a while to intuitively understand. This is made even more complex by the rather weird nature of synchronizing with a Swapchain. As I have seen many people struggling with this topic and using incorrect pipeline barriers, I decided to explain this fundamental topic in more detail.

Disclaimer: While I would say that I have a very thorough understanding of Vulkan synchronization and can intuitively use it, I cannot guarantee that I do not make any mistakes, so take the following chapters as my best effort to explain what I think is correct.

Do note that I expect a relatively basic understanding of regular Vulkan synchronization in order to follow this post.

## The Problem

Lets take the following basic operation that is used in pretty much any rendering loop:

```c++
// Acquire the next swapchain image
auto imageIndex = device.acquireNextImageKHR(swapchain, UINT64_MAX, acquireSemaphore, nullptr);

// Transition the image to COLOR_ATTACHMENT_OPTIMAL
auto imageBarrier = vk::ImageMemoryBarrier {
    ...
    .srcAccessMask = {},
    .dstAccessMask = vk::AccessFlagsBits::eColorAttachmentWrite,
    .oldLayout = vk::ImageLayout::eUndefined,
    .newLayout = vk::ImageLayout::eColorAttachmentOptimal,
    .image = swapchainImages[imageIndex],
};
commandBuffer.cmdPipelineBarrier(
    vk::PipelineStageFlagBits::eColorAttachmentOutput,
    vk::PipelineStageFlagBits::eColorAttachmentOutput,
    ...
    imageBarrier,
);

// Do rendering stuff
...

// transition to PRESENT_SRC
auto imageBarrier2 = vk::ImageMemoryBarrier {
    ...
    .srcAccessMask = vk::AccessFlagBits::eColorAttachmentWrite,
    .dstAccessMask = {},
    .oldLayout = vk::ImageLayout::eColorAttachmentOptimal,
    .newLayout = vk::ImageLayout::ePresentSrcKHR,
    .image = swapchainImages[imageIndex],
};
commandBuffer.cmdPipelineBarrier(
    vk::PipelineStageFlagBits::eColorAttachmentOutput,
    vk::PipelineStageFlagBits::eBottomOfPipe,
    ...
    imageBarrier2,
);

// submit command buffer
vk::PipelineStageFlags waitMask = vk::PipelineStageFlagBits::eColorAttachmentOutput;
auto submitInfo = vk::SubmitInfoKHR {
    .waitSemaphoreCount = 1,
    .pWaitSemaphores = &acquireSemaphore,
    .pWaitDstStageMask = &waitMask,
    .signalSemaphoreCount = 1,
    .pSignalSemaphores = &renderFinishedSemaphore,
    ...
};
graphicsQueue.submit({ submitInfo }, ...);

...

// present the image
auto presentInfo = vk::PresentInfoKHR {
    ...
    .waitSemaphoreCount = 1,
    .pWaitSemaphores = &renderFinishedSemaphore,
    .swapchainCount = 1,
    .pSwapchains = &swapchain,
    .pImageIndices = &imageIndex,
};
graphicsQueue.queuePresentKHR(presentInfo);
```

You've probably seen this kind of loop many times in many different tutorials and just took for granted that you have to use the specified pipeline stages and access flags. The problem is that I have never seen a tutorial that actually explains **why** you have to use these flags, as it isn't that obvious. It may seem like presenting an image is part of the COLOR_ATTACHMENT_OUTPUT stage, which is the stage we wait on before transitioning the image to COLOR_ATTACHMENT_OPTIMAL. However, this is not true. The swapchain operations do not operate in **any** pipeline stage.  
This means that synchronizing directly with the swapchain operations via pipeline barriers or subpass dependencies is impossible. Why these stages and flags are correct anyway is explained in the following chapters.

## Image acquisition

We acquire the next swapchain image with:

```c++
auto imageIndex = device.acquireNextImageKHR(swapchain, UINT64_MAX, acquireSemaphore, nullptr);
```

After this, we obviously need to transition the image to the ColorAttachmentOptimal layout in order to allow rendering to it. But first, we have to wait for the presentation engine to have finished reading the image.  
As the spec states [[1]]:
> The presentation engine may not have finished reading from the image at the time it is acquired, so the application must use semaphore and/or fence to ensure that the image layout and contents are not modified until the presentation engine reads have completed.

[1]: https://registry.khronos.org/vulkan/specs/1.3-extensions/html/chap34.html#vkAcquireNextImageKHR

This means that we have to wait on the semaphore before transitioning the image (and then writing to it when rendering). But which stage do we wait on, if the swapchain acquisition operates on **no** pipeline stage? The answer is: *dependency chains*!

A dependency chain occurs when a synchronization operation A has a destination stage mask that shares stages with the source stage mask of a following synchronization operation B. When such a chain occurs, anything in the srcStage of A *happens before* anything in the dstStage of B [[2]].

[2]: https://registry.khronos.org/vulkan/specs/1.3-extensions/html/chap7.html#synchronization-dependencies

```
Synchronization A: srcStage=TRANSFER, dstStage=COLOR_ATTACHMENT_OUTPUT
Synchronization B: srcStage=COLOR_ATTACHMENT_OUTPUT, dstStage=COMPUTE_SHADER
```

In this case, since A.dstStage has common bits with B.srcStage, we guarantee that anything in TRANSFER happens before COMPUTE_SHADER. This is a fundamental piece of the specification that we need to understand the image acquisition synchronization.

Let's take a look at the acquisition again:

```c++
auto imageIndex = device.acquireNextImageKHR(swapchain, UINT64_MAX, acquireSemaphore, nullptr);

// Transition the image to COLOR_ATTACHMENT_OPTIMAL
auto imageBarrier = vk::ImageMemoryBarrier {
    ...
    .srcAccessMask = {},
    .dstAccessMask = vk::AccessFlagsBits::eColorAttachmentWrite,
    .oldLayout = vk::ImageLayout::eUndefined,
    .newLayout = vk::ImageLayout::eColorAttachmentOptimal,
    .image = swapchainImages[imageIndex],
};
commandBuffer.cmdPipelineBarrier(
    vk::PipelineStageFlagBits::eColorAttachmentOutput,
    vk::PipelineStageFlagBits::eColorAttachmentOutput,
    ...
    imageBarrier,
);

// submit command buffer
vk::PipelineStageFlags waitMask = vk::PipelineStageFlagBits::eColorAttachmentOutput;
auto submitInfo = vk::SubmitInfoKHR {
    .waitSemaphoreCount = 1,
    .pWaitSemaphores = &acquireSemaphore,
    .pWaitDstStageMask = &waitMask,
    .signalSemaphoreCount = 1,
    .pSignalSemaphores = &renderFinishedSemaphore,
    ...
};
graphicsQueue.submit({ submitInfo }, ...);
```

Can you spot the execution dependency here? The semaphore signal wait operation of `graphicsQueue.submit` forms synchronization operation **A** with `srcStage=???` and `dstStage=eColorAttachmentOutput`. The pipeline barrier forms synchronization operation **B** with `srcStage=eColorAttachmentOutput` and `dstStage=eColorAttachmentOutput`. As you can probably see, since `A.dstStage` and `B.srcStage` are identical, an execution dependency chain is formed.  
With this we ensure that the presentation engine is finished with reading the swapchain image (semaphore is signalled), before we transition the image for further rendering (pipeline barrier). We have achieved this without needing to specify the non-existing hypothetical stage "PRESENTATION_ENGINE".  
This is the only way to properly synchronize with the swapchain acquisition operation and the spec also states this [[3]]:

> Note  
> When the presentable image will be accessed by some stage S, the recommended idiom for ensuring correct synchronization is:
> 
> The VkSubmitInfo used to submit the image layout transition for execution includes vkAcquireNextImageKHR::semaphore in its pWaitSemaphores member, with the corresponding element of pWaitDstStageMask including S.
> 
> The synchronization command that performs any necessary image layout transition includes S in both the srcStageMask and dstStageMask.

As you can see, this is exactly what we do in the code above. Do note that the actual stage that we use as semaphore wait stage and pipeline barrier source stage is actually irrelevant for the sake of correctness, as we only use this stage to setup a dependency chain, but as the spec recommends this setup, we will use it in this way.

[3]: https://registry.khronos.org/vulkan/specs/1.3-extensions/html/chap34.html#vkAcquireNextImageKHR

## Image Presentation

The second part of the swapchain synchronization is thankfully much less complicated. We only need to ensure that we have finished writing to the swapchain image and transitioned it to PRESENT_SRC layout before the presentation image tries to read from it. Let's take a look at the presentation part of the code again:

```c++
// transition to PRESENT_SRC
auto imageBarrier2 = vk::ImageMemoryBarrier {
    ...
    .srcAccessMask = vk::AccessFlagBits::eColorAttachmentWrite,
    .dstAccessMask = {},
    .oldLayout = vk::ImageLayout::eColorAttachmentOptimal,
    .newLayout = vk::ImageLayout::ePresentSrcKHR,
    .image = swapchainImages[imageIndex],
};
commandBuffer.cmdPipelineBarrier(
    vk::PipelineStageFlagBits::eColorAttachmentOutput,
    vk::PipelineStageFlagBits::eBottomOfPipe,
    ...
    imageBarrier2,
);

// submit command buffer
vk::PipelineStageFlags waitMask = vk::PipelineStageFlagBits::eColorAttachmentOutput;
auto submitInfo = vk::SubmitInfoKHR {
    .waitSemaphoreCount = 1,
    .pWaitSemaphores = &acquireSemaphore,
    .pWaitDstStageMask = &waitMask,
    .signalSemaphoreCount = 1,
    .pSignalSemaphores = &renderFinishedSemaphore,
    ...
};
graphicsQueue.submit({ submitInfo }, ...);

...

// present the image
auto presentInfo = vk::PresentInfoKHR {
    ...
    .waitSemaphoreCount = 1,
    .pWaitSemaphores = &renderFinishedSemaphore,
    .swapchainCount = 1,
    .pSwapchains = &swapchain,
    .pImageIndices = &imageIndex,
};
graphicsQueue.queuePresentKHR(presentInfo);
```

As you can see, with the second image barrier, we ensure that the transition only happens after we have finished rendering (barrier source stage is `eColorAttachmentOutput`). Since we rely on the `renderFinishedSemaphore` to be signalled, we do not need to specify a specific destination stage for the barrier, thus we just use `eBottomOfPipe`.  
But we do have to make our writes **visible** to the presentation engine, which we would usually do with `dstAccessMask` of the image memory barrier. However, since the presentation engine operates in **no** pipeline stage, there also is no access mask to specify. How do we solve this problem? The answer luckily is: we don't have to.  
The spec states for `vkQueuePresentKHR` [[4]]:
> Any writes to memory backing the images referenced by the pImageIndices and pSwapchains members of pPresentInfo, that are available before vkQueuePresentKHR is executed, are automatically made visible to the read access performed by the presentation engine. This automatic visibility operation for an image happens-after the semaphore signal operation, and happens-before the presentation engine accesses the image.

[4]: https://registry.khronos.org/vulkan/specs/1.3-extensions/html/chap34.html#vkQueuePresentKHR

This means, we only need to make our writes visible, which is done via the `srcAccessMask`. The presentation engine automatically makes sure the any **available** writes are made **visible**.

## Conclusion

I hope I could make the actual mechanisms behind swapchain synchronization more clear, as it is really not that trivial when starting out with Vulkan synchronization. There are a lot of tutorials out there that do not properly explain the reasoning behind the synchronization choices, resulting in many people making incorrect assumptions.

To summarize swapchain synchronization:
- `vkAcquireNextImageKHR` can only be waited on via dependency chains, which is done via `waitDstStages` of `VkSubmitInfo` must share bits with `srcStageMask` of the corresponding image transition barrier.
- `vkQueuePresentKHR` makes all **available** writes automatically **visible**, meaning we do not have to specify a specific `dstStage` and `dstAccessMask`

