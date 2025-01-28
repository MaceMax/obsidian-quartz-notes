Vulkan has no global state, unlike OpenGL. `VkDevice` and `VkInstance` is passed as a parameter to every Vulkan API function call.

## The Vulkan API
### Validation layers
The Vulkan API is vast and as a result it is easy to make mistakes, but this is where Validation Layers comes to the rescue.

Validation Layers is an optional feature in Vulkan that detects and reports incorrect usage of the API.

Validation Layers work by intercepting your function calls and performs validation on the data. If all data is correctly validated a call to the driver will be performed. It should be noted that intercepting functions and running validation comes with a performance loss.

The Validation Layers are useful for catching errors such as using incorrect configurations, using wrong enums, synchronization issues, and object lifetimes. It is recommended to enable Validation Layers while developing to ensure that your code follows the specification requirements. Running your application without any reported validation errors is a good sign, however this should not be used as an indicator for how well your application will run on different hardware. We recommend testing your application on as many different devices as possible.

It is important to note that the Validation Layers does not catch bugs like uninitialized memory and bad pointers. It is highly recommended to use tools such as Address Sanitizer and Valgrind. Be aware that graphics drivers often create false positives while using these tools, making their output quite noisy.

### `VkInstance`
Represents the Vulkan API Context. When creating a `VkInstance`, we can enable validation layers, extensions and a logger. In general, we only need to create a single `VkInstance` during the entire program duration.

### `VkPhysicalDevice`
A reference to a specific GPU. We query `VkInstance` for what GPUs that are available in the system. `VkPhysicalDevice` also allows us to query the features of the GPU such as memory size and what extensions are available. 

### `VkDevice`
We create a `VkDevice` from the `VkPhysicalDevice`. This is the GPU driver of on the GPU hardware and is the interface that allows us to communicate with the GPU. Most Vulkan commands require the `VkDevice`.

### `Swapchain`
A `swapchain` allows us to perform rendering to the screen. It is a OS/windowing provided structure with some images that we can draw to and display on screen. Not a core part of the Vulkan specification, and are unique to different platforms. **For compute shaders, we do not need to set up a swapchain**. A swapchain is created on a given size, and if the window resizes, you will have to recreate the swapchain again.

## Workflow 
In the Vulkan API, almost everything is designed around objects that you create manually and then use. This is not only for the actual GPU resources such as Images/Textures and Buffers (for memory or things like Vertex data) but also for a lot of “internal” configuration structures.

For example, things like GPU fixed function, like rasterizer mode, are stored into a Pipeline object which holds shaders and other configuration. In OpenGL and DX11, this is calculated “on the fly” while rendering. When you use Vulkan, you need to think if it’s worth to cache these objects, or create them while rendering on-the-fly. Some objects like Pipelines are expensive to create, so it’s best to create them on load screens or background threads. Other objects are cheaper to create, such as DescriptorSets and it’s fine to create them when you need them during rendering.

Because everything in Vulkan is “pre-built” by default, it means that most of the state validation in the GPU will be done when you create the object, and the rendering itself does less work and is faster. Good understanding of how these objects are created and used will allow you to control how everything executes in a way that will make the frame rate smooth.

When doing actual GPU commands, all of the work on the GPU has to be recorded into a CommandBuffer, and submitted into a Queue. You first allocate a command buffer, start encoding things on it, and then you execute it by adding it to a Queue. When you submit a command buffer into a queue, it will start executing on the GPU side. You have tools to control when that execution has finished. If you submit multiple command buffers into different queues, it is possible that they execute in parallel.

There is no concept of a frame in Vulkan. This means that the way you render is entirely up to you. The only thing that matters is when you have to display the frame to the screen, which is done through a swapchain. But there is no fundamental difference between rendering and then sending the images over the network, or saving the images into a file, or displaying it into the screen through the swapchain.

This means it is possible to use Vulkan in an entirely headless mode, where nothing is displayed to the screen. You can render the images and then store them onto the disk (very useful for testing!) or use Vulkan as a way to perform GPU calculations such as a raytracer or other compute tasks.

### Vulkan command execution

![[vkcommands.png]]

In Vulkan commands go through a command buffer, and are executed Queue. The general flow to execute commands is: 
- Allocate a `VkCommandBuffer` from a `VkCommandPool`
- Record commands into the command buffer using `VkCmdXXXXX` functions.
- Submit the command buffer into a `VkQueue`, which starts executing the commands. 
The same command buffer can submitted multiple times. 

### `VkQueue`
Queues in Vulkan are an "execution port" for GPUs. Every GPU has multiple queues available, and can be used at the same time to execute different  command streams. Commands submitted to separate queues may execute at once. 

All queues in Vulkan come from a Queue family. A queue family is the type of queue it is and what type of commands it supports. 

### `VkCommandPool`
A `VkCommandPool` is created from the `VkDevice`. You have to specify the index of the queue family that the command pool will create commands from. `VkCommandPool` can be thought of as the allocator for a `VkCommandBuffer`. You can allocate as many `VkCommandBuffer` as you want from a given pool, but you can only record commands from one thread at a time. If you want multi-threaded command recording, you need more `VkCommandPool` objects. For that reason, we will pair a command buffer with its command allocator.

### `VkCommandBuffer`
All GPU commands are recorded into a `VkCommandBuffer`. All functions that will execute GPU work, won't do anything until the `VkCommandBuffer` is submitted to the GPU through a `VkQueueSubmit` call. 

Command buffers initially start in a **Ready** state. When in a **Ready** state, we call `vkBeginCommandBuffer()` to put it into a recording state, where we can start inputting commands with `vkCmdXXXX` functions. Once done recording commands, we call `vkEndCommandBuffer()` to finish recording and put the buffer into an **Executable** state where it is ready to be submitted to the GPU using `vkQueueSubmit()`. Do not reset the command buffer until the GPU has finished executing all the commands from the command buffer. Reset the command buffer with `vkResetCommandBuffer()`.

## Synchronization
