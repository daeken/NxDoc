Introduction
============

NVN (unknown what it stands for, but I like to think it's pronounced "in-veen") is the native graphics API for Switch games. It's at a similar level of abstraction to Metal, Vulkan, and D3D12, but tailored just to the Switch.

Comparisons
===========

***!NOTE: AI-Generated!***

Metal
-----

NVN shares several design philosophies with Metal, making it feel like a spiritual cousin:

**Similarities:**
- **Builder Pattern**: Both APIs use builders for object creation (NVN's `nvnTextureBuilderSet*` mirrors Metal's descriptor pattern with `MTLTextureDescriptor`)
- **Explicit Memory Management**: Both require explicit memory pool allocation and management, though NVN uses `NVNmemoryPool` while Metal uses `MTLHeap` and `MTLBuffer`
- **Command Buffer Model**: Both use command buffers (`NVNcommandBuffer` vs `MTLCommandBuffer`) that are recorded and submitted to queues
- **State Objects**: Both separate pipeline state from dynamic state, with NVN using explicit state objects (`NVNblendState`, `NVNdepthStencilState`) similar to Metal's `MTLRenderPipelineState`
- **Descriptor Pools**: NVN's `NVNtexturePool` and `NVNsamplerPool` are analogous to Metal's argument buffers and descriptor heaps
- **Platform-Specific**: Both are designed for specific hardware (Switch for NVN, Apple Silicon/Metal GPUs for Metal) allowing tight optimization

**Differences:**
- **Memory Ownership**: NVN requires games to allocate object memory themselves (opaque pointers), while Metal uses reference-counted objects with automatic memory management
- **Function Access**: NVN uses a unique `nvnDeviceGetProcAddress` mechanism similar to OpenGL's extension loading, while Metal uses direct linking
- **Synchronization**: NVN has simpler sync primitives (`NVNsync`, `NVNevent`) compared to Metal's `MTLFence` and `MTLEvent` system
- **Language Binding**: Metal is Objective-C/Swift native with C++ support, while NVN is pure C with opaque pointers
- **Debug Features**: NVN has extensive built-in debug support (`nvnDeviceInstallDebugCallback`, debug labels, debug domains) that feels more integrated than Metal's separate GPU frame capture tools
- **Texture Views**: NVN has lightweight `NVNtextureView` objects for aliasing textures, similar to but more explicit than Metal's texture views
- **Advanced Features**: NVN exposes Switch-specific features like tiled cache control (`nvnCommandBufferSetTiledCacheAction`), Z-cull data management, and hierarchical depth buffer features not present in Metal

**Overall**: NVN feels like a middle ground between Metal's ergonomics and Vulkan's explicitness, optimized specifically for Nintendo Switch hardware.

Vulkan
------

NVN and Vulkan share the low-level, explicit control philosophy but differ significantly in verbosity and complexity:

**Similarities:**
- **Explicit Resource Management**: Both require manual memory allocation, binding, and lifetime management through memory pools/heaps
- **Command Buffers**: Both use command buffer recording with primary/secondary (NVN doesn't appear to have secondary command buffers from the API surface)
- **Pipeline State Objects**: Both separate compile-time state from runtime state, though NVN's state objects are more granular
- **Descriptor Management**: NVN's texture/sampler pools map to Vulkan's descriptor sets and pools (`VkDescriptorPool`, `VkDescriptorSet`)
- **Queue-Based Submission**: Both use queues for command submission with explicit synchronization
- **Sparse Resources**: Both support sparse/virtual memory mapping (`nvnMemoryPoolMapVirtual` vs Vulkan's sparse binding)
- **Render Passes**: While not explicit in the header, NVN's `nvnCommandBufferSetRenderTargets` is analogous to Vulkan's render pass management

**Differences:**
- **Verbosity**: Vulkan requires extensive `VkCreateInfo` structures with explicit size/type fields (`sType`, `pNext` chains), while NVN uses builder objects with setter functions that feel more streamlined
- **Initialization**: Vulkan's multi-stage initialization (instance→physical device→logical device→queues) is collapsed in NVN to device builder→device→queue builder→queue
- **Memory Model**: Vulkan requires querying memory types and heaps explicitly (`vkGetPhysicalDeviceMemoryProperties`), while NVN abstracts this with `NVNmemoryPool` with flags
- **Synchronization**: Vulkan has semaphores, fences, and events as distinct primitives; NVN simplifies to `NVNsync` and `NVNevent`
- **Validation**: Vulkan uses validation layers as an external mechanism, NVN has built-in debug callbacks and extensive debug database walking (`nvnDeviceWalkDebugDatabase`)
- **Extensions**: Vulkan's extension system (`vkGetInstanceProcAddr`) is similar to NVN's `nvnDeviceGetProcAddress`, but Vulkan requires explicit extension enabling
- **Barriers**: Vulkan has explicit pipeline barriers for memory/layout transitions; NVN has a simpler `nvnCommandBufferBarrier` with flags
- **Shader Binding**: Vulkan uses pipeline layouts and descriptor set layouts explicitly; NVN appears to infer binding from shader data and uses simpler `nvnCommandBufferBind*` calls

**Unique NVN Features Not in Vulkan:**
- **Builder Pattern**: More ergonomic than Vulkan's info structs, avoiding repetitive structure initialization
- **Packaged Textures**: Direct support for proprietary texture formats (`nvnTextureBuilderSetPackagedTextureData`)
- **Fast Clear Registration**: Hardware-level optimization hints (`nvnDeviceRegisterFastClearColor`)
- **Tiled Rendering Control**: Explicit tile cache management for the Switch's tile-based architecture
- **Transform Feedback**: Built-in transform feedback support (Vulkan requires extensions)
- **Subroutines**: Shader subroutine support (`nvnProgramSetSubroutineLinkage`) not present in Vulkan
- **Coverage Modulation**: Advanced MSAA features like coverage modulation tables

**Overall**: NVN is significantly less verbose than Vulkan while maintaining similar low-level control. It feels like Vulkan's concepts filtered through a more pragmatic, console-focused lens where some choices (like memory types) are predetermined by the fixed hardware. The API trades Vulkan's cross-platform flexibility for Switch-specific optimizations and a cleaner, less ceremonious interface.

API Surface
===========

NVN does have a C++ API, in theory, but games don't actually use this as far as I can tell. Rather, everything is done indirectly via an exposed C API, with two exceptions: `nv::InitializeGraphics` and `nv::SetGraphicsAllocator`.

In order to access the C API, most symbols aren't directly accessed, but rather accessed via their equivalent of dlsym. Once nv is initialized, games call `nvnBootstrapLoader(const char*)` to request functions. However, only two functions are requested via this:

- `NVNboolean nvnDeviceInitialize(NVNdevice* device, const NVNdeviceBuilder* builder)`
- `void* nvnDeviceGetProcAddress(const NVNdevice* device, const char* name)`

It's unknown why they request `nvnDeviceInitialize` via the `nvnBootstrapLoader`, as all of the NVNdeviceBuilder-related functions (along with everything else) get requested via `nvnDeviceGetProcAddress`.

It should be noted that -- as far as I can tell -- every pointer type is purely treated as opaque, but is allocated by the game. (TODO: Figure out and document allocation sizes for all of these.)

Each category of functions relates to a different subset of NVN functionality and is split into its own section here.

Buffer
------

- `void nvnBufferBuilderSetDevice(NVNbufferBuilder* builder, NVNdevice* device)` -- AI-generated guess: Sets the device pointer for the given buffer builder
- `void nvnBufferBuilderSetDefaults(NVNbufferBuilder* builder)` -- AI-generated guess: Sets the buffer builder to default values
- `void nvnBufferBuilderSetStorage(NVNbufferBuilder* builder, NVNmemoryPool* pool, ptrdiff_t offset, size_t size)` -- AI-generated guess: Sets the memory pool, offset, and size for the buffer storage
- `NVNmemoryPool nvnBufferBuilderGetMemoryPool(const NVNbufferBuilder* builder)` -- AI-generated guess: Gets the memory pool associated with the buffer builder
- `ptrdiff_t nvnBufferBuilderGetMemoryOffset(const NVNbufferBuilder* builder)` -- AI-generated guess: Gets the offset within the memory pool for the buffer
- `size_t nvnBufferBuilderGetSize(const NVNbufferBuilder* builder)` -- AI-generated guess: Gets the size of the buffer
- `NVNboolean nvnBufferInitialize(NVNbuffer* buffer, const NVNbufferBuilder* builder)` -- AI-generated guess: Initializes a buffer object using the provided builder configuration
- `void nvnBufferSetDebugLabel(NVNbuffer* buffer, const char* label)` -- AI-generated guess: Sets a debug label for the buffer for debugging purposes
- `void nvnBufferFinalize(NVNbuffer* buffer)` -- AI-generated guess: Finalizes and destroys the buffer object, releasing its resources
- `void* nvnBufferMap(const NVNbuffer* buffer)` -- AI-generated guess: Maps the buffer memory for CPU access and returns a pointer to it
- `NVNbufferAddress nvnBufferGetAddress(const NVNbuffer* buffer)` -- AI-generated guess: Gets the GPU address of the buffer
- `void nvnBufferFlushMappedRange(const NVNbuffer* buffer, ptrdiff_t offset, size_t size)` -- AI-generated guess: Flushes a range of mapped buffer memory to make CPU writes visible to GPU
- `void nvnBufferInvalidateMappedRange(const NVNbuffer* buffer, ptrdiff_t offset, size_t size)` -- AI-generated guess: Invalidates a range of mapped buffer memory to make GPU writes visible to CPU
- `NVNmemoryPool* nvnBufferGetMemoryPool(const NVNbuffer* buffer)` -- AI-generated guess: Gets the memory pool associated with the buffer
- `ptrdiff_t nvnBufferGetMemoryOffset(const NVNbuffer* buffer)` -- AI-generated guess: Gets the offset within the memory pool where the buffer is located
- `size_t nvnBufferGetSize(const NVNbuffer* buffer)` -- AI-generated guess: Gets the size of the buffer in bytes
- `uint64_t nvnBufferGetDebugID(const NVNbuffer* buffer)` -- AI-generated guess: Gets a unique debug identifier for the buffer

CommandBuffer
-------------

- `NVNboolean nvnCommandBufferInitialize(NVNcommandBuffer* cmdBuf, NVNdevice* device)` -- AI-generated guess: Initializes a command buffer object for the given device
- `void nvnCommandBufferFinalize(NVNcommandBuffer* cmdBuf)` -- AI-generated guess: Finalizes and destroys the command buffer, releasing its resources
- `void nvnCommandBufferSetDebugLabel(NVNcommandBuffer* cmdBuf, const char* label)` -- AI-generated guess: Sets a debug label for the command buffer for debugging purposes
- `void nvnCommandBufferSetMemoryCallback(NVNcommandBuffer* cmdBuf, PFNNVNCOMMANDBUFFERMEMORYCALLBACKPROC callback)` -- AI-generated guess: Sets a callback function for command buffer memory management
- `void nvnCommandBufferSetMemoryCallbackData(NVNcommandBuffer* cmdBuf, void* callbackData)` -- AI-generated guess: Sets user data pointer to be passed to the memory callback
- `void nvnCommandBufferAddCommandMemory(NVNcommandBuffer* cmdBuf, const NVNmemoryPool* pool, ptrdiff_t offset, size_t size)` -- AI-generated guess: Adds memory from a pool for storing GPU commands
- `void nvnCommandBufferAddControlMemory(NVNcommandBuffer* cmdBuf, void* memory, size_t size)` -- AI-generated guess: Adds memory for command buffer control structures
- `size_t nvnCommandBufferGetCommandMemorySize(const NVNcommandBuffer* cmdBuf)` -- AI-generated guess: Gets the total size of command memory allocated
- `size_t nvnCommandBufferGetCommandMemoryUsed(const NVNcommandBuffer* cmdBuf)` -- AI-generated guess: Gets the amount of command memory currently used
- `size_t nvnCommandBufferGetCommandMemoryFree(const NVNcommandBuffer* cmdBuf)` -- AI-generated guess: Gets the amount of free command memory remaining
- `size_t nvnCommandBufferGetControlMemorySize(const NVNcommandBuffer* cmdBuf)` -- AI-generated guess: Gets the total size of control memory allocated
- `size_t nvnCommandBufferGetControlMemoryUsed(const NVNcommandBuffer* cmdBuf)` -- AI-generated guess: Gets the amount of control memory currently used
- `size_t nvnCommandBufferGetControlMemoryFree(const NVNcommandBuffer* cmdBuf)` -- AI-generated guess: Gets the amount of free control memory remaining
- `void nvnCommandBufferBeginRecording(NVNcommandBuffer* cmdBuf)` -- AI-generated guess: Begins recording commands into the command buffer
- `NVNcommandHandle nvnCommandBufferEndRecording(NVNcommandBuffer* cmdBuf)` -- AI-generated guess: Ends recording and returns a handle to the recorded commands
- `void nvnCommandBufferCallCommands(NVNcommandBuffer* cmdBuf, int numCommands, const NVNcommandHandle* handles)` -- AI-generated guess: Calls previously recorded command sequences
- `void nvnCommandBufferCopyCommands(NVNcommandBuffer* cmdBuf, int numCommands, const NVNcommandHandle* handles)` -- AI-generated guess: Copies previously recorded commands into the current command buffer
- `void nvnCommandBufferBindBlendState(NVNcommandBuffer* cmdBuf, const NVNblendState* blend)` -- AI-generated guess: Binds blend state for color blending operations
- `void nvnCommandBufferBindChannelMaskState(NVNcommandBuffer* cmdBuf, const NVNchannelMaskState* channelMask)` -- AI-generated guess: Binds channel mask state to control which color channels are written
- `void nvnCommandBufferBindColorState(NVNcommandBuffer* cmdBuf, const NVNcolorState* color)` -- AI-generated guess: Binds color state settings
- `void nvnCommandBufferBindMultisampleState(NVNcommandBuffer* cmdBuf, const NVNmultisampleState* multisample)` -- AI-generated guess: Binds multisample anti-aliasing state
- `void nvnCommandBufferBindPolygonState(NVNcommandBuffer* cmdBuf, const NVNpolygonState* polygon)` -- AI-generated guess: Binds polygon rasterization state (culling, fill mode, etc.)
- `void nvnCommandBufferBindDepthStencilState(NVNcommandBuffer* cmdBuf, const NVNdepthStencilState* depthStencil)` -- AI-generated guess: Binds depth testing and stencil testing state
- `void nvnCommandBufferBindVertexAttribState(NVNcommandBuffer* cmdBuf, int numAttribs, const NVNvertexAttribState* attribs)` -- AI-generated guess: Binds vertex attribute format and layout configuration
- `void nvnCommandBufferBindVertexStreamState(NVNcommandBuffer* cmdBuf, int numStreams, const NVNvertexStreamState* streams)` -- AI-generated guess: Binds vertex stream stride and divisor configuration
- `void nvnCommandBufferBindProgram(NVNcommandBuffer* cmdBuf, const NVNprogram* program, int stages)` -- AI-generated guess: Binds a shader program to specified pipeline stages
- `void nvnCommandBufferBindVertexBuffer(NVNcommandBuffer* cmdBuf, int index, NVNbufferAddress buffer, size_t size)` -- AI-generated guess: Binds a single vertex buffer at the specified index
- `void nvnCommandBufferBindVertexBuffers(NVNcommandBuffer* cmdBuf, int first, int count, const NVNbufferRange* buffers)` -- AI-generated guess: Binds multiple vertex buffers starting at the specified index
- `void nvnCommandBufferBindUniformBuffer(NVNcommandBuffer* cmdBuf, NVNshaderStage stage, int index, NVNbufferAddress buffer, size_t size)` -- AI-generated guess: Binds a single uniform buffer for a shader stage
- `void nvnCommandBufferBindUniformBuffers(NVNcommandBuffer* cmdBuf, NVNshaderStage stage, int first, int count, const NVNbufferRange* buffers)` -- AI-generated guess: Binds multiple uniform buffers for a shader stage
- `void nvnCommandBufferBindTransformFeedbackBuffer(NVNcommandBuffer* cmdBuf, int index, NVNbufferAddress buffer, size_t size)` -- AI-generated guess: Binds a single transform feedback buffer
- `void nvnCommandBufferBindTransformFeedbackBuffers(NVNcommandBuffer* cmdBuf, int first, int count, const NVNbufferRange* buffers)` -- AI-generated guess: Binds multiple transform feedback buffers
- `void nvnCommandBufferBindStorageBuffer(NVNcommandBuffer* cmdBuf, NVNshaderStage stage, int index, NVNbufferAddress buffer, size_t size)` -- AI-generated guess: Binds a single shader storage buffer for a shader stage
- `void nvnCommandBufferBindStorageBuffers(NVNcommandBuffer* cmdBuf, NVNshaderStage stage, int first, int count, const NVNbufferRange* buffers)` -- AI-generated guess: Binds multiple shader storage buffers for a shader stage
- `void nvnCommandBufferBindTexture(NVNcommandBuffer* cmdBuf, NVNshaderStage stage, int index, NVNtextureHandle texture)` -- AI-generated guess: Binds a single texture to a shader stage
- `void nvnCommandBufferBindTextures(NVNcommandBuffer* cmdBuf, NVNshaderStage stage, int first, int count, const NVNtextureHandle* textures)` -- AI-generated guess: Binds multiple textures to a shader stage
- `void nvnCommandBufferBindImage(NVNcommandBuffer* cmdBuf, NVNshaderStage stage, int index, NVNimageHandle image)` -- AI-generated guess: Binds a single image for shader read/write access
- `void nvnCommandBufferBindImages(NVNcommandBuffer* cmdBuf, NVNshaderStage stage, int first, int count, const NVNimageHandle* images)` -- AI-generated guess: Binds multiple images for shader read/write access
- `void nvnCommandBufferSetPatchSize(NVNcommandBuffer* cmdBuf, int i)` -- AI-generated guess: Sets the patch size for tessellation shaders
- `void nvnCommandBufferSetInnerTessellationLevels(NVNcommandBuffer* cmdBuf, const float* f)` -- AI-generated guess: Sets the inner tessellation levels for tessellation shaders
- `void nvnCommandBufferSetOuterTessellationLevels(NVNcommandBuffer* cmdBuf, const float* f)` -- AI-generated guess: Sets the outer tessellation levels for tessellation shaders
- `void nvnCommandBufferSetPrimitiveRestart(NVNcommandBuffer* cmdBuf, NVNboolean b, int i)` -- AI-generated guess: Enables/disables primitive restart and sets the restart index
- `void nvnCommandBufferBeginTransformFeedback(NVNcommandBuffer* cmdBuf, NVNbufferAddress buffer)` -- AI-generated guess: Begins transform feedback operation
- `void nvnCommandBufferEndTransformFeedback(NVNcommandBuffer* cmdBuf, NVNbufferAddress buffer)` -- AI-generated guess: Ends transform feedback operation
- `void nvnCommandBufferPauseTransformFeedback(NVNcommandBuffer* cmdBuf, NVNbufferAddress buffer)` -- AI-generated guess: Pauses transform feedback operation
- `void nvnCommandBufferResumeTransformFeedback(NVNcommandBuffer* cmdBuf, NVNbufferAddress buffer)` -- AI-generated guess: Resumes a paused transform feedback operation
- `void nvnCommandBufferDrawTransformFeedback(NVNcommandBuffer* cmdBuf, NVNdrawPrimitive mode, NVNbufferAddress buffer)` -- AI-generated guess: Draws primitives using transform feedback buffer data
- `void nvnCommandBufferDrawArrays(NVNcommandBuffer* cmdBuf, NVNdrawPrimitive mode, int first, int count)` -- AI-generated guess: Draws primitives from vertex arrays
- `void nvnCommandBufferDrawElements(NVNcommandBuffer* cmdBuf, NVNdrawPrimitive mode, NVNindexType type, int count, NVNbufferAddress indexBuffer)` -- AI-generated guess: Draws indexed primitives
- `void nvnCommandBufferDrawElementsBaseVertex(NVNcommandBuffer* cmdBuf, NVNdrawPrimitive mode, NVNindexType type, int count, NVNbufferAddress indexBuffer, int baseVertex)` -- AI-generated guess: Draws indexed primitives with a base vertex offset
- `void nvnCommandBufferDrawArraysInstanced(NVNcommandBuffer* cmdBuf, NVNdrawPrimitive mode, int first, int count, int baseInstance, int instanceCount)` -- AI-generated guess: Draws multiple instances of primitives from vertex arrays
- `void nvnCommandBufferDrawElementsInstanced(NVNcommandBuffer* cmdBuf, NVNdrawPrimitive mode, NVNindexType type, int count, NVNbufferAddress indexBuffer, int baseVertex, int baseInstance, int instanceCount)` -- AI-generated guess: Draws multiple instances of indexed primitives
- `void nvnCommandBufferDrawArraysIndirect(NVNcommandBuffer* cmdBuf, NVNdrawPrimitive mode, NVNbufferAddress buffer)` -- AI-generated guess: Draws primitives with parameters sourced from a buffer
- `void nvnCommandBufferDrawElementsIndirect(NVNcommandBuffer* cmdBuf, NVNdrawPrimitive mode, NVNindexType type, NVNbufferAddress buffer1, NVNbufferAddress buffer2)` -- AI-generated guess: Draws indexed primitives with parameters sourced from buffers
- `void nvnCommandBufferMultiDrawArraysIndirectCount(NVNcommandBuffer* cmdBuf, NVNdrawPrimitive mode, NVNbufferAddress buffer1, NVNbufferAddress buffer2, int i, ptrdiff_t o)` -- AI-generated guess: Draws multiple primitives with parameters and count from buffers
- `void nvnCommandBufferMultiDrawElementsIndirectCount( NVNcommandBuffer* cmdBuf, NVNdrawPrimitive mode, NVNindexType type, NVNbufferAddress buffer1, NVNbufferAddress buffer2, NVNbufferAddress buffer3, int i, ptrdiff_t o)` -- AI-generated guess: Draws multiple indexed primitives with parameters and count from buffers
- `void nvnCommandBufferClearColor(NVNcommandBuffer* cmdBuf, int index, const float* color, int mask)` -- AI-generated guess: Clears a color render target with floating-point color values
- `void nvnCommandBufferClearColori(NVNcommandBuffer* cmdBuf, int index, const int* color, int mask)` -- AI-generated guess: Clears a color render target with signed integer color values
- `void nvnCommandBufferClearColorui(NVNcommandBuffer* cmdBuf, int index, const uint32_t* color, int mask)` -- AI-generated guess: Clears a color render target with unsigned integer color values
- `void nvnCommandBufferClearDepthStencil(NVNcommandBuffer* cmdBuf, float depthValue, NVNboolean depthMask, int stencilValue, int stencilMask)` -- AI-generated guess: Clears depth and stencil buffers
- `void nvnCommandBufferDispatchCompute(NVNcommandBuffer* cmdBuf, int groupsX, int groupsY, int groupsZ)` -- AI-generated guess: Dispatches compute shader work groups
- `void nvnCommandBufferDispatchComputeIndirect(NVNcommandBuffer* cmdBuf, NVNbufferAddress buffer)` -- AI-generated guess: Dispatches compute shader with parameters from a buffer
- `void nvnCommandBufferSetViewport(NVNcommandBuffer* cmdBuf, int x, int y, int w, int h)` -- AI-generated guess: Sets the viewport rectangle
- `void nvnCommandBufferSetViewports(NVNcommandBuffer* cmdBuf, int first, int count, const float* ranges)` -- AI-generated guess: Sets multiple viewports
- `void nvnCommandBufferSetViewportSwizzles(NVNcommandBuffer* cmdBuf, int first, int count, const NVNviewportSwizzle* swizzles)` -- AI-generated guess: Sets viewport coordinate swizzling
- `void nvnCommandBufferSetScissor(NVNcommandBuffer* cmdBuf, int x, int y, int w, int h)` -- AI-generated guess: Sets the scissor rectangle for clipping
- `void nvnCommandBufferSetScissors(NVNcommandBuffer* cmdBuf, int first, int count, const int* rects)` -- AI-generated guess: Sets multiple scissor rectangles
- `void nvnCommandBufferSetDepthRange(NVNcommandBuffer* cmdBuf, float n, float f)` -- AI-generated guess: Sets the depth range mapping from normalized device coordinates
- `void nvnCommandBufferSetDepthBounds(NVNcommandBuffer* cmdBuf, NVNboolean enable, float n, float f)` -- AI-generated guess: Sets depth bounds testing range
- `void nvnCommandBufferSetDepthRanges(NVNcommandBuffer* cmdBuf, int first, int count, const float* ranges)` -- AI-generated guess: Sets multiple depth ranges
- `void nvnCommandBufferSetTiledCacheAction(NVNcommandBuffer* cmdBuf, NVNtiledCacheAction action)` -- AI-generated guess: Sets tiled cache behavior for tile-based rendering
- `void nvnCommandBufferSetTiledCacheTileSize(NVNcommandBuffer* cmdBuf, int w, int h)` -- AI-generated guess: Sets the tile size for tiled cache operations
- `void nvnCommandBufferBindSeparateTexture(NVNcommandBuffer* cmdBuf, NVNshaderStage stage, int i, NVNseparateTextureHandle handle)` -- AI-generated guess: Binds a texture separately from its sampler
- `void nvnCommandBufferBindSeparateSampler(NVNcommandBuffer* cmdBuf, NVNshaderStage stage, int i, NVNseparateSamplerHandle handle)` -- AI-generated guess: Binds a sampler separately from its texture
- `void nvnCommandBufferBindSeparateTextures(NVNcommandBuffer* cmdBuf, NVNshaderStage stage, int i1, int i2, const NVNseparateTextureHandle* handle)` -- AI-generated guess: Binds multiple textures separately from samplers
- `void nvnCommandBufferBindSeparateSamplers(NVNcommandBuffer* cmdBuf, NVNshaderStage stage, int i1, int i2, const NVNseparateSamplerHandle* handle)` -- AI-generated guess: Binds multiple samplers separately from textures
- `void nvnCommandBufferSetStencilValueMask(NVNcommandBuffer* cmdBuf, NVNface faces, int mask)` -- AI-generated guess: Sets stencil value mask for read operations
- `void nvnCommandBufferSetStencilMask(NVNcommandBuffer* cmdBuf, NVNface faces, int mask)` -- AI-generated guess: Sets stencil write mask
- `void nvnCommandBufferSetStencilRef(NVNcommandBuffer* cmdBuf, NVNface faces, int ref)` -- AI-generated guess: Sets stencil reference value for comparisons
- `void nvnCommandBufferSetBlendColor(NVNcommandBuffer* cmdBuf, const float* blendColor)` -- AI-generated guess: Sets the constant blend color
- `void nvnCommandBufferSetPointSize(NVNcommandBuffer* cmdBuf, float pointSize)` -- AI-generated guess: Sets the point size for point primitive rendering
- `void nvnCommandBufferSetLineWidth(NVNcommandBuffer* cmdBuf, float lineWidth)` -- AI-generated guess: Sets the line width for line primitive rendering
- `void nvnCommandBufferSetPolygonOffsetClamp(NVNcommandBuffer* cmdBuf, float factor, float units, float clamp)` -- AI-generated guess: Sets polygon depth offset parameters with clamping
- `void nvnCommandBufferSetAlphaRef(NVNcommandBuffer* cmdBuf, float ref)` -- AI-generated guess: Sets the alpha test reference value
- `void nvnCommandBufferSetSampleMask(NVNcommandBuffer* cmdBuf, int mask)` -- AI-generated guess: Sets which samples are enabled for multisample rendering
- `void nvnCommandBufferSetRasterizerDiscard(NVNcommandBuffer* cmdBuf, NVNboolean discard)` -- AI-generated guess: Enables/disables rasterization discard
- `void nvnCommandBufferSetDepthClamp(NVNcommandBuffer* cmdBuf, NVNboolean clamp)` -- AI-generated guess: Enables/disables depth clamping instead of clipping
- `void nvnCommandBufferSetConservativeRasterEnable(NVNcommandBuffer* cmdBuf, NVNboolean enable)` -- AI-generated guess: Enables/disables conservative rasterization
- `void nvnCommandBufferSetConservativeRasterDilate(NVNcommandBuffer* cmdBuf, float f)` -- AI-generated guess: Sets the dilation amount for conservative rasterization
- `void nvnCommandBufferSetSubpixelPrecisionBias(NVNcommandBuffer* cmdBuf, int i1, int i2)` -- AI-generated guess: Sets subpixel precision bias for rasterization
- `void nvnCommandBufferCopyBufferToTexture(NVNcommandBuffer* cmdBuf, NVNbufferAddress buffer, const NVNtexture* dstTexture, const NVNtextureView* dstView, const NVNcopyRegion* dstRegion, int flags)` -- AI-generated guess: Copies data from a buffer to a texture
- `void nvnCommandBufferCopyTextureToBuffer(NVNcommandBuffer* cmdBuf, const NVNtexture* srcTexture, const NVNtextureView* srcView, const NVNcopyRegion* srcRegion, NVNbufferAddress buffer, int flags)` -- AI-generated guess: Copies texture data to a buffer
- `void nvnCommandBufferCopyTextureToTexture(NVNcommandBuffer* cmdBuf, const NVNtexture* srcTexture, const NVNtextureView* srcView, const NVNcopyRegion* srcRegion, const NVNtexture* dstTexture, const NVNtextureView* dstView, const NVNcopyRegion* dstRegion, int flags)` -- AI-generated guess: Copies data from one texture to another
- `void nvnCommandBufferCopyBufferToBuffer(NVNcommandBuffer* cmdBuf, NVNbufferAddress src, NVNbufferAddress dst, size_t size, int flags)` -- AI-generated guess: Copies data from one buffer to another
- `void nvnCommandBufferClearBuffer(NVNcommandBuffer* cmdBuf, NVNbufferAddress buffer, size_t size, uint32_t i)` -- AI-generated guess: Clears a buffer with a uint32 value
- `void nvnCommandBufferClearTexture(NVNcommandBuffer* cmdBuf, const NVNtexture* dstTexture, const NVNtextureView* dstView, const NVNcopyRegion* dstRegion, const float* color, int mask)` -- AI-generated guess: Clears a texture with floating-point color values
- `void nvnCommandBufferClearTexturei(NVNcommandBuffer* cmdBuf, const NVNtexture* dstTexture, const NVNtextureView* dstView, const NVNcopyRegion* dstRegion, const int* color, int mask)` -- AI-generated guess: Clears a texture with signed integer color values
- `void nvnCommandBufferClearTextureui(NVNcommandBuffer* cmdBuf, const NVNtexture* dstTexture, const NVNtextureView* dstView, const NVNcopyRegion* dstRegion, const uint32_t* color, int mask)` -- AI-generated guess: Clears a texture with unsigned integer color values
- `void nvnCommandBufferUpdateUniformBuffer(NVNcommandBuffer* cmdBuf, NVNbufferAddress buffer, size_t size, ptrdiff_t o, size_t s, const void* p)` -- AI-generated guess: Updates uniform buffer data with new values
- `void nvnCommandBufferReportCounter(NVNcommandBuffer* cmdBuf, NVNcounterType counter, NVNbufferAddress buffer)` -- AI-generated guess: Reports a performance counter value to a buffer
- `void nvnCommandBufferResetCounter(NVNcommandBuffer* cmdBuf, NVNcounterType counter)` -- AI-generated guess: Resets a performance counter
- `void nvnCommandBufferReportValue(NVNcommandBuffer* cmdBuf, uint32_t value, NVNbufferAddress buffer)` -- AI-generated guess: Writes a uint32 value to a buffer
- `void nvnCommandBufferSetRenderEnable(NVNcommandBuffer* cmdBuf, NVNboolean enable)` -- AI-generated guess: Enables/disables rendering
- `void nvnCommandBufferSetRenderEnableConditional(NVNcommandBuffer* cmdBuf, NVNconditionalRenderMode mode, NVNbufferAddress addr)` -- AI-generated guess: Enables conditional rendering based on a value in a buffer
- `void nvnCommandBufferSetRenderTargets(NVNcommandBuffer* cmdBuf, int numColors, const NVNtexture* const* colors, const NVNtextureView* const* colorViews, const NVNtexture* depthStencil, const NVNtextureView* depthStencilView)` -- AI-generated guess: Sets the render targets for rendering operations
- `void nvnCommandBufferDiscardColor(NVNcommandBuffer* cmdBuf, int i)` -- AI-generated guess: Marks a color render target's contents as discardable
- `void nvnCommandBufferDiscardDepthStencil(NVNcommandBuffer* cmdBuf)` -- AI-generated guess: Marks depth/stencil buffer contents as discardable
- `void nvnCommandBufferDownsample(NVNcommandBuffer* cmdBuf, const NVNtexture* src, const NVNtexture* dst)` -- AI-generated guess: Downsamples a multisample texture to a single-sample texture
- `void nvnCommandBufferTiledDownsample(NVNcommandBuffer* cmdBuf, const NVNtexture* texture1, const NVNtexture* texture2)` -- AI-generated guess: Performs tiled downsampling of a texture
- `void nvnCommandBufferDownsampleTextureView(NVNcommandBuffer* cmdBuf, const NVNtexture* texture1, const NVNtextureView* view1, const NVNtexture* texture2, const NVNtextureView* view2)` -- AI-generated guess: Downsamples using texture views
- `void nvnCommandBufferTiledDownsampleTextureView(NVNcommandBuffer* cmdBuf, const NVNtexture* texture1, const NVNtextureView* view1, const NVNtexture* texture2, const NVNtextureView* view2)` -- AI-generated guess: Performs tiled downsampling using texture views
- `void nvnCommandBufferBarrier(NVNcommandBuffer* cmdBuf, int barrier)` -- AI-generated guess: Inserts a memory barrier for synchronization
- `void nvnCommandBufferWaitSync(NVNcommandBuffer* cmdBuf, const NVNsync* sync)` -- AI-generated guess: Waits for a sync object to be signaled
- `void nvnCommandBufferFenceSync(NVNcommandBuffer* cmdBuf, NVNsync* sync, NVNsyncCondition condition, int fence)` -- AI-generated guess: Inserts a fence sync command
- `void nvnCommandBufferSetTexturePool(NVNcommandBuffer* cmdBuf, const NVNtexturePool* pool)` -- AI-generated guess: Sets the active texture descriptor pool
- `void nvnCommandBufferSetSamplerPool(NVNcommandBuffer* cmdBuf, const NVNsamplerPool* pool)` -- AI-generated guess: Sets the active sampler descriptor pool
- `void nvnCommandBufferSetShaderScratchMemory(NVNcommandBuffer* cmdBuf, const NVNmemoryPool* pool, ptrdiff_t offset, size_t size)` -- AI-generated guess: Sets scratch memory for shader execution
- `void nvnCommandBufferSaveZCullData(NVNcommandBuffer* cmdBuf, NVNbufferAddress addr, size_t size)` -- AI-generated guess: Saves hierarchical Z-buffer culling data
- `void nvnCommandBufferRestoreZCullData(NVNcommandBuffer* cmdBuf, NVNbufferAddress addr, size_t size)` -- AI-generated guess: Restores hierarchical Z-buffer culling data
- `void nvnCommandBufferSetCopyRowStride(NVNcommandBuffer* cmdBuf, ptrdiff_t stride)` -- AI-generated guess: Sets the row stride for copy operations
- `void nvnCommandBufferSetCopyImageStride(NVNcommandBuffer* cmdBuf, ptrdiff_t stride)` -- AI-generated guess: Sets the image stride for copy operations
- `ptrdiff_t nvnCommandBufferGetCopyRowStride(const NVNcommandBuffer* cmdBuf)` -- AI-generated guess: Gets the current copy row stride
- `ptrdiff_t nvnCommandBufferGetCopyImageStride(const NVNcommandBuffer* cmdBuf)` -- AI-generated guess: Gets the current copy image stride
- `void nvnCommandBufferDrawTexture(NVNcommandBuffer* cmdBuf, NVNtextureHandle handle, const NVNdrawTextureRegion* region1, const NVNdrawTextureRegion* region2)` -- AI-generated guess: Draws a texture directly to the framebuffer
- `void nvnCommandBufferSetProgramSubroutines(NVNcommandBuffer* cmdBuf, NVNprogram* program, NVNshaderStage stage, const int i1, const int i2, const int* i3)` -- AI-generated guess: Sets active shader subroutines for a program
- `void nvnCommandBufferBindCoverageModulationTable(NVNcommandBuffer* cmdBuf, const float* f)` -- AI-generated guess: Binds a coverage modulation lookup table
- `void nvnCommandBufferResolveDepthBuffer(NVNcommandBuffer* cmdBuf)` -- AI-generated guess: Resolves a multisample depth buffer
- `void nvnCommandBufferPushDebugGroupStatic(NVNcommandBuffer* cmdBuf, uint32_t i, const char* description)` -- AI-generated guess: Pushes a debug group with static string for GPU debugging
- `void nvnCommandBufferPushDebugGroupDynamic(NVNcommandBuffer* cmdBuf, uint32_t i, const char* description)` -- AI-generated guess: Pushes a debug group with dynamic string for GPU debugging
- `void nvnCommandBufferPushDebugGroup(NVNcommandBuffer* cmdBuf, uint32_t i, const char* description)` -- AI-generated guess: Pushes a debug group for GPU debugging
- `void nvnCommandBufferPopDebugGroup(NVNcommandBuffer* cmdBuf)` -- AI-generated guess: Pops the current debug group
- `void nvnCommandBufferPopDebugGroupId(NVNcommandBuffer* cmdBuf, uint32_t i)` -- AI-generated guess: Pops debug groups up to a specific ID
- `void nvnCommandBufferInsertDebugMarkerStatic(NVNcommandBuffer* cmdBuf, uint32_t i, const char* description)` -- AI-generated guess: Inserts a debug marker with static string
- `void nvnCommandBufferInsertDebugMarkerDynamic(NVNcommandBuffer* cmdBuf, uint32_t i, const char* description)` -- AI-generated guess: Inserts a debug marker with dynamic string
- `void nvnCommandBufferInsertDebugMarker(NVNcommandBuffer* cmdBuf, const char* description)` -- AI-generated guess: Inserts a debug marker into the command stream
- `PFNNVNCOMMANDBUFFERMEMORYCALLBACKPROC nvnCommandBufferGetMemoryCallback(const NVNcommandBuffer* cmdBuf)` -- AI-generated guess: Gets the current memory callback function
- `void nvnCommandBufferGetMemoryCallbackData(const NVNcommandBuffer* cmdBuf)` -- AI-generated guess: Gets the current memory callback user data
- `NVNboolean nvnCommandBufferIsRecording(const NVNcommandBuffer* cmdBuf)` -- AI-generated guess: Checks if the command buffer is currently in recording state

Device
------

- `void nvnDeviceBuilderSetDefaults(NVNdeviceBuilder* builder)` -- AI-generated guess: Sets the device builder to default values
- `void nvnDeviceBuilderSetFlags(NVNdeviceBuilder* builder, int flags)` -- AI-generated guess: Sets flags for device creation
- `NVNboolean nvnDeviceInitialize(NVNdevice* device, const NVNdeviceBuilder* builder)` -- AI-generated guess: Initializes a device object using the provided builder configuration
- `void nvnDeviceFinalize(NVNdevice* device)` -- AI-generated guess: Finalizes and destroys the device, releasing its resources
- `void nvnDeviceSetDebugLabel(NVNdevice* device, const char* label)` -- AI-generated guess: Sets a debug label for the device for debugging purposes
- `PFNNVNGENERICFUNCPTRPROC nvnDeviceGetProcAddress(const NVNdevice* device, const char* s)` -- AI-generated guess: Gets a function pointer for a named NVN function
- `void nvnDeviceGetInteger(const NVNdevice* device, NVNdeviceInfo pname, int* v)` -- AI-generated guess: Queries device capabilities and limits as integer values
- `uint64_t nvnDeviceGetCurrentTimestampInNanoseconds(const NVNdevice* device)` -- AI-generated guess: Gets the current GPU timestamp in nanoseconds
- `void nvnDeviceSetIntermediateShaderCache(NVNdevice* device, int i)` -- AI-generated guess: Configures intermediate shader cache settings
- `NVNtextureHandle nvnDeviceGetTextureHandle(const NVNdevice* device, int textureID, int samplerID)` -- AI-generated guess: Gets a combined texture and sampler handle from descriptor IDs
- `NVNtextureHandle nvnDeviceGetTexelFetchHandle(const NVNdevice* device, int textureID)` -- AI-generated guess: Gets a texture handle for texel fetch operations (sampling without a sampler)
- `NVNimageHandle nvnDeviceGetImageHandle(const NVNdevice* device, int textureID)` -- AI-generated guess: Gets an image handle for shader read/write access from a texture descriptor ID
- `void nvnDeviceInstallDebugCallback(NVNdevice* device, const PFNNVNDEBUGCALLBACKPROC callback, void* callbackData, NVNboolean enable)` -- AI-generated guess: Installs a callback function for debug messages and validation
- `NVNdebugDomainId nvnDeviceGenerateDebugDomainId(const NVNdevice* device, const char* s)` -- AI-generated guess: Generates a unique debug domain ID from a string name
- `void nvnDeviceSetWindowOriginMode(NVNdevice* device, NVNwindowOriginMode windowOriginMode)` -- AI-generated guess: Sets the window coordinate origin mode (upper-left vs lower-left)
- `void nvnDeviceSetDepthMode(NVNdevice* device, NVNdepthMode depthMode)` -- AI-generated guess: Sets the depth range mode (0-1 vs -1-1)
- `NVNboolean nvnDeviceRegisterFastClearColor(NVNdevice* device, const float* color, NVNformat format)` -- AI-generated guess: Registers a floating-point color value for fast clear optimization
- `NVNboolean nvnDeviceRegisterFastClearColori(NVNdevice* device, const int* color, NVNformat format)` -- AI-generated guess: Registers a signed integer color value for fast clear optimization
- `NVNboolean nvnDeviceRegisterFastClearColorui(NVNdevice* device, const uint32_t* color, NVNformat format)` -- AI-generated guess: Registers an unsigned integer color value for fast clear optimization
- `NVNboolean nvnDeviceRegisterFastClearDepth(NVNdevice* device, float f)` -- AI-generated guess: Registers a depth value for fast clear optimization
- `NVNwindowOriginMode nvnDeviceGetWindowOriginMode(const NVNdevice* device)` -- AI-generated guess: Gets the current window coordinate origin mode
- `NVNdepthMode nvnDeviceGetDepthMode(const NVNdevice* device)` -- AI-generated guess: Gets the current depth range mode
- `uint64_t nvnDeviceGetTimestampInNanoseconds(const NVNdevice* device, const NVNcounterData* counterData)` -- AI-generated guess: Converts counter data to a timestamp in nanoseconds
- `void nvnDeviceApplyDeferredFinalizes(NVNdevice* device, int i)` -- AI-generated guess: Applies deferred resource finalization operations
- `void nvnDeviceFinalizeCommandHandle(NVNdevice* device, NVNcommandHandle handles)` -- AI-generated guess: Finalizes and releases a command handle
- `void nvnDeviceWalkDebugDatabase(const NVNdevice* device, NVNdebugObjectType debugObjectType, PFNNVNWALKDEBUGDATABASECALLBACKPROC callback, void* callbackData)` -- AI-generated guess: Iterates through debug database objects with a callback
- `NVNseparateTextureHandle nvnDeviceGetSeparateTextureHandle(const NVNdevice* device, int textureID)` -- AI-generated guess: Gets a texture handle for separate texture/sampler binding
- `NVNseparateSamplerHandle nvnDeviceGetSeparateSamplerHandle(const NVNdevice* device, int textureID)` -- AI-generated guess: Gets a sampler handle for separate texture/sampler binding
- `NVNboolean nvnDeviceIsExternalDebuggerAttached(const NVNdevice* device)` -- AI-generated guess: Checks if an external debugger is currently attached
- `void nvnDeviceWaitForError(NVNdevice* device)` -- AI-generated guess: Waits for and processes any pending GPU errors

Event
-----

- `void nvnEventBuilderSetDefaults(NVNeventBuilder* builder)` -- AI-generated guess: Sets the event builder to default values
- `void nvnEventBuilderSetStorage(NVNeventBuilder* builder, const NVNmemoryPool* pool, int64_t size)` -- AI-generated guess: Sets the memory pool and size for event storage
- `NVNboolean nvnEventInitialize(NVNevent* event, const NVNeventBuilder* builder)` -- AI-generated guess: Initializes an event object using the provided builder configuration
- `void nvnEventFinalize(NVNevent* event)` -- AI-generated guess: Finalizes and destroys the event, releasing its resources
- `uint32_t nvnEventGetValue(const NVNevent* event)` -- AI-generated guess: Gets the current value of the event
- `void nvnEventSignal(NVNevent* event, NVNeventSignalMode mode, uint32_t i)` -- AI-generated guess: Signals the event from the CPU with a specified value and mode

MemoryPool
----------

- `void nvnMemoryPoolBuilderSetDevice(NVNmemoryPoolBuilder* builder, NVNdevice* device)` -- AI-generated guess: Sets the device pointer for the given memory pool builder
- `void nvnMemoryPoolBuilderSetDefaults(NVNmemoryPoolBuilder* builder)` -- AI-generated guess: Sets the memory pool builder to default values
- `void nvnMemoryPoolBuilderSetStorage(NVNmemoryPoolBuilder* builder, void* memory, size_t size)` -- AI-generated guess: Sets the memory storage and size for the memory pool
- `void nvnMemoryPoolBuilderSetFlags(NVNmemoryPoolBuilder* builder, int flags)` -- AI-generated guess: Sets flags for memory pool creation (CPU/GPU access, physical/virtual, etc.)
- `void nvnMemoryPoolBuilderGetMemory(const NVNmemoryPoolBuilder* builder)` -- AI-generated guess: Gets the memory pointer from the memory pool builder
- `size_t nvnMemoryPoolBuilderGetSize(const NVNmemoryPoolBuilder* builder)` -- AI-generated guess: Gets the size of the memory pool
- `NVNmemoryPoolFlags nvnMemoryPoolBuilderGetFlags(const NVNmemoryPoolBuilder* builder)` -- AI-generated guess: Gets the flags for the memory pool
- `NVNboolean nvnMemoryPoolInitialize(NVNmemoryPool* pool, const NVNmemoryPoolBuilder* builder)` -- AI-generated guess: Initializes a memory pool object using the provided builder configuration
- `void nvnMemoryPoolSetDebugLabel(NVNmemoryPool* pool, const char* label)` -- AI-generated guess: Sets a debug label for the memory pool for debugging purposes
- `void nvnMemoryPoolFinalize(NVNmemoryPool* pool)` -- AI-generated guess: Finalizes and destroys the memory pool, releasing its resources
- `void* nvnMemoryPoolMap(const NVNmemoryPool* pool)` -- AI-generated guess: Maps the memory pool for CPU access and returns a pointer to it
- `void nvnMemoryPoolFlushMappedRange(const NVNmemoryPool* pool, ptrdiff_t offset, size_t size)` -- AI-generated guess: Flushes a range of mapped memory pool to make CPU writes visible to GPU
- `void nvnMemoryPoolInvalidateMappedRange(const NVNmemoryPool* pool, ptrdiff_t offset, size_t size)` -- AI-generated guess: Invalidates a range of mapped memory pool to make GPU writes visible to CPU
- `NVNbufferAddress nvnMemoryPoolGetBufferAddress(const NVNmemoryPool* pool)` -- AI-generated guess: Gets the GPU buffer address for the memory pool
- `NVNboolean nvnMemoryPoolMapVirtual(NVNmemoryPool* pool, int numRequests, const NVNmappingRequest* requests)` -- AI-generated guess: Maps virtual memory pages to physical memory within the pool
- `size_t nvnMemoryPoolGetSize(const NVNmemoryPool* pool)` -- AI-generated guess: Gets the size of the memory pool in bytes
- `NVNmemoryPoolFlags nvnMemoryPoolGetFlags(const NVNmemoryPool* pool)` -- AI-generated guess: Gets the flags for the memory pool

Program
-------

- `NVNboolean nvnProgramInitialize(NVNprogram* program, NVNdevice* device)` -- AI-generated guess: Initializes a program object for the given device
- `void nvnProgramFinalize(NVNprogram* program)` -- AI-generated guess: Finalizes and destroys the program, releasing its resources
- `void nvnProgramSetDebugLabel(NVNprogram* program, const char* label)` -- AI-generated guess: Sets a debug label for the program for debugging purposes
- `NVNboolean nvnProgramSetShaders(NVNprogram* program, int count, const NVNshaderData* stageData)` -- AI-generated guess: Sets the shader stages for the program from compiled shader data
- `NVNboolean nvnProgramSetSubroutineLinkage(NVNprogram* program, int i, const NVNsubroutineLinkageMapPtr* ptr)` -- AI-generated guess: Sets subroutine linkage mapping for the program

Queue
-----

- `NVNqueueGetErrorResult nvnQueueGetError(NVNqueue* queue, NVNqueueErrorInfo* info)` -- AI-generated guess: Gets error information from the queue
- `size_t nvnQueueGetTotalCommandMemoryUsed(NVNqueue* queue)` -- AI-generated guess: Gets the total amount of command memory used by the queue
- `size_t nvnQueueGetTotalControlMemoryUsed(NVNqueue* queue)` -- AI-generated guess: Gets the total amount of control memory used by the queue
- `size_t nvnQueueGetTotalComputeMemoryUsed(NVNqueue* queue)` -- AI-generated guess: Gets the total amount of compute memory used by the queue
- `void nvnQueueResetMemoryUsageCounts(NVNqueue* queue)` -- AI-generated guess: Resets the memory usage counters for the queue
- `void nvnQueueBuilderSetDevice(NVNqueueBuilder* builder, NVNdevice* device)` -- AI-generated guess: Sets the device pointer for the given queue builder
- `void nvnQueueBuilderSetDefaults(NVNqueueBuilder* builder)` -- AI-generated guess: Sets the queue builder to default values
- `void nvnQueueBuilderSetFlags(NVNqueueBuilder* builder, int flags)` -- AI-generated guess: Sets flags for queue creation
- `void nvnQueueBuilderSetCommandMemorySize(NVNqueueBuilder* builder, size_t size)` -- AI-generated guess: Sets the command memory size for the queue
- `void nvnQueueBuilderSetComputeMemorySize(NVNqueueBuilder* builder, size_t size)` -- AI-generated guess: Sets the compute memory size for the queue
- `void nvnQueueBuilderSetControlMemorySize(NVNqueueBuilder* builder, size_t size)` -- AI-generated guess: Sets the control memory size for the queue
- `size_t nvnQueueBuilderGetQueueMemorySize(const NVNqueueBuilder* builder)` -- AI-generated guess: Gets the total memory size required for the queue
- `void nvnQueueBuilderSetQueueMemory(NVNqueueBuilder* builder, void* memory, size_t size)` -- AI-generated guess: Sets the memory storage for the queue
- `void nvnQueueBuilderSetCommandFlushThreshold(NVNqueueBuilder* builder, size_t size)` -- AI-generated guess: Sets the threshold for automatic command buffer flushing
- `NVNboolean nvnQueueInitialize(NVNqueue* queue, const NVNqueueBuilder* builder)` -- AI-generated guess: Initializes a queue object using the provided builder configuration
- `void nvnQueueFinalize(NVNqueue* queue)` -- AI-generated guess: Finalizes and destroys the queue, releasing its resources
- `void nvnQueueSetDebugLabel(NVNqueue* queue, const char* label)` -- AI-generated guess: Sets a debug label for the queue for debugging purposes
- `void nvnQueueSubmitCommands(NVNqueue* queue, int numCommands, const NVNcommandHandle* handles)` -- AI-generated guess: Submits command buffers to the queue for GPU execution
- `void nvnQueueFlush(NVNqueue* queue)` -- AI-generated guess: Flushes pending commands to the GPU
- `void nvnQueueFinish(NVNqueue* queue)` -- AI-generated guess: Waits for all queued GPU commands to complete
- `void nvnQueuePresentTexture(NVNqueue* queue, NVNwindow* window, int textureIndex)` -- AI-generated guess: Presents a texture to a window for display
- `NVNqueueAcquireTextureResult nvnQueueAcquireTexture(NVNqueue* queue, NVNwindow* window, int* textureIndex)` -- AI-generated guess: Acquires a texture from a window for rendering
- `void nvnQueueFenceSync(NVNqueue* queue, NVNsync* sync, NVNsyncCondition condition, int flags)` -- AI-generated guess: Inserts a fence sync into the queue
- `NVNboolean nvnQueueWaitSync(NVNqueue* queue, const NVNsync* sync)` -- AI-generated guess: Waits on the queue for a sync object to be signaled

Sampler
-------

- `void nvnSamplerBuilderSetDevice(NVNsamplerBuilder* builder, NVNdevice* device)` -- AI-generated guess: Sets the device pointer for the given sampler builder
- `void nvnSamplerBuilderSetDefaults(NVNsamplerBuilder* builder)` -- AI-generated guess: Sets the sampler builder to default values
- `void nvnSamplerBuilderSetMinMagFilter(NVNsamplerBuilder* builder, NVNminFilter min, NVNmagFilter mag)` -- AI-generated guess: Sets the minification and magnification filter modes
- `void nvnSamplerBuilderSetWrapMode(NVNsamplerBuilder* builder, NVNwrapMode s, NVNwrapMode t, NVNwrapMode r)` -- AI-generated guess: Sets the texture coordinate wrap modes for s, t, and r coordinates
- `void nvnSamplerBuilderSetLodClamp(NVNsamplerBuilder* builder, float min, float max)` -- AI-generated guess: Sets the minimum and maximum level-of-detail clamp values
- `void nvnSamplerBuilderSetLodBias(NVNsamplerBuilder* builder, float bias)` -- AI-generated guess: Sets the level-of-detail bias for texture sampling
- `void nvnSamplerBuilderSetCompare(NVNsamplerBuilder* builder, NVNcompareMode mode, NVNcompareFunc func)` -- AI-generated guess: Sets the depth comparison mode and function for shadow sampling
- `void nvnSamplerBuilderSetBorderColor(NVNsamplerBuilder* builder, const float* borderColor)` -- AI-generated guess: Sets the border color with floating-point values
- `void nvnSamplerBuilderSetBorderColori(NVNsamplerBuilder* builder, const int* borderColor)` -- AI-generated guess: Sets the border color with signed integer values
- `void nvnSamplerBuilderSetBorderColorui(NVNsamplerBuilder* builder, const uint32_t* borderColor)` -- AI-generated guess: Sets the border color with unsigned integer values
- `void nvnSamplerBuilderSetMaxAnisotropy(NVNsamplerBuilder* builder, float maxAniso)` -- AI-generated guess: Sets the maximum anisotropic filtering level
- `void nvnSamplerBuilderSetReductionFilter(NVNsamplerBuilder* builder, NVNsamplerReduction filter)` -- AI-generated guess: Sets the reduction filter mode (min/max filtering)
- `void nvnSamplerBuilderSetLodSnap(NVNsamplerBuilder* builder, float f)` -- AI-generated guess: Sets the LOD snap value for texture sampling
- `void nvnSamplerBuilderGetMinMagFilter(const NVNsamplerBuilder* builder, NVNminFilter* min, NVNmagFilter* mag)` -- AI-generated guess: Gets the minification and magnification filter modes
- `void nvnSamplerBuilderGetWrapMode(const NVNsamplerBuilder* builder, NVNwrapMode* s, NVNwrapMode* t, NVNwrapMode* r)` -- AI-generated guess: Gets the texture coordinate wrap modes
- `void nvnSamplerBuilderGetLodClamp(const NVNsamplerBuilder* builder, float* min, float* max)` -- AI-generated guess: Gets the level-of-detail clamp values
- `float nvnSamplerBuilderGetLodBias(const NVNsamplerBuilder* builder)` -- AI-generated guess: Gets the level-of-detail bias
- `void nvnSamplerBuilderGetCompare(const NVNsamplerBuilder* builder, NVNcompareMode* mode, NVNcompareFunc* func)` -- AI-generated guess: Gets the depth comparison mode and function
- `void nvnSamplerBuilderGetBorderColor(const NVNsamplerBuilder* builder, float* borderColor)` -- AI-generated guess: Gets the border color as floating-point values
- `void nvnSamplerBuilderGetBorderColori(const NVNsamplerBuilder* builder, int* borderColor)` -- AI-generated guess: Gets the border color as signed integer values
- `void nvnSamplerBuilderGetBorderColorui(const NVNsamplerBuilder* builder, uint32_t* borderColor)` -- AI-generated guess: Gets the border color as unsigned integer values
- `float nvnSamplerBuilderGetMaxAnisotropy(const NVNsamplerBuilder* builder)` -- AI-generated guess: Gets the maximum anisotropic filtering level
- `NVNsamplerReduction nvnSamplerBuilderGetReductionFilter(const NVNsamplerBuilder* builder)` -- AI-generated guess: Gets the reduction filter mode
- `float nvnSamplerBuilderGetLodSnap(const NVNsamplerBuilder* builder)` -- AI-generated guess: Gets the LOD snap value
- `NVNboolean nvnSamplerInitialize(NVNsampler* sampler, const NVNsamplerBuilder* builder)` -- AI-generated guess: Initializes a sampler object using the provided builder configuration
- `void nvnSamplerFinalize(NVNsampler* sampler)` -- AI-generated guess: Finalizes and destroys the sampler, releasing its resources
- `void nvnSamplerSetDebugLabel(NVNsampler* sampler, const char* label)` -- AI-generated guess: Sets a debug label for the sampler for debugging purposes
- `void nvnSamplerGetMinMagFilter(const NVNsampler* sampler, NVNminFilter* min, NVNmagFilter* mag)` -- AI-generated guess: Gets the minification and magnification filter modes from the sampler
- `void nvnSamplerGetWrapMode(const NVNsampler* sampler, NVNwrapMode* s, NVNwrapMode* t, NVNwrapMode* r)` -- AI-generated guess: Gets the texture coordinate wrap modes from the sampler
- `void nvnSamplerGetLodClamp(const NVNsampler* sampler, float* min, float* max)` -- AI-generated guess: Gets the level-of-detail clamp values from the sampler
- `float nvnSamplerGetLodBias(const NVNsampler* sampler)` -- AI-generated guess: Gets the level-of-detail bias from the sampler
- `void nvnSamplerGetCompare(const NVNsampler* sampler, NVNcompareMode* mode, NVNcompareFunc* func)` -- AI-generated guess: Gets the depth comparison mode and function from the sampler
- `void nvnSamplerGetBorderColor(const NVNsampler* sampler, float* borderColor)` -- AI-generated guess: Gets the border color as floating-point values from the sampler
- `void nvnSamplerGetBorderColori(const NVNsampler* sampler, int* borderColor)` -- AI-generated guess: Gets the border color as signed integer values from the sampler
- `void nvnSamplerGetBorderColorui(const NVNsampler* sampler, uint32_t* borderColor)` -- AI-generated guess: Gets the border color as unsigned integer values from the sampler
- `float nvnSamplerGetMaxAnisotropy(const NVNsampler* sampler)` -- AI-generated guess: Gets the maximum anisotropic filtering level from the sampler
- `NVNsamplerReduction nvnSamplerGetReductionFilter(const NVNsampler* sampler)` -- AI-generated guess: Gets the reduction filter mode from the sampler
- `NVNboolean nvnSamplerCompare(const NVNsampler* sampler1, const NVNsampler* sampler2)` -- AI-generated guess: Compares two samplers for equality
- `uint64_t nvnSamplerGetDebugID(const NVNsampler* sampler)` -- AI-generated guess: Gets a unique debug identifier for the sampler

SamplerPool
-----------

- `NVNboolean nvnSamplerPoolInitialize(NVNsamplerPool* samplerPool, const NVNmemoryPool* memoryPool, ptrdiff_t offset, int numDescriptors)` -- AI-generated guess: Initializes a sampler descriptor pool in a memory pool at the specified offset
- `void nvnSamplerPoolSetDebugLabel(NVNsamplerPool* pool, const char* label)` -- AI-generated guess: Sets a debug label for the sampler pool for debugging purposes
- `void nvnSamplerPoolFinalize(NVNsamplerPool* pool)` -- AI-generated guess: Finalizes and destroys the sampler pool, releasing its resources
- `void nvnSamplerPoolRegisterSampler(const NVNsamplerPool* pool, int id, const NVNsampler* sampler)` -- AI-generated guess: Registers a sampler object into the pool at the specified descriptor ID
- `void nvnSamplerPoolRegisterSamplerBuilder(const NVNsamplerPool* pool, int id, const NVNsamplerBuilder* builder)` -- AI-generated guess: Registers a sampler from a builder into the pool at the specified descriptor ID
- `const NVNmemoryPool* nvnSamplerPoolGetMemoryPool(const NVNsamplerPool* pool)` -- AI-generated guess: Gets the memory pool associated with the sampler pool
- `ptrdiff_t nvnSamplerPoolGetMemoryOffset(const NVNsamplerPool* pool)` -- AI-generated guess: Gets the offset within the memory pool where the sampler pool is located
- `int nvnSamplerPoolGetSize(const NVNsamplerPool* pool)` -- AI-generated guess: Gets the number of sampler descriptors in the pool

State
-----

### BlendState

- `void nvnBlendStateSetDefaults(NVNblendState* blend)` -- AI-generated guess: Sets the blend state to default values
- `void nvnBlendStateSetBlendTarget(NVNblendState* blend, int target)` -- AI-generated guess: Sets which render target this blend state applies to
- `void nvnBlendStateSetBlendFunc(NVNblendState* blend, NVNblendFunc srcFunc, NVNblendFunc dstFunc, NVNblendFunc srcFuncAlpha, NVNblendFunc dstFuncAlpha)` -- AI-generated guess: Sets the blend function factors for RGB and alpha channels
- `void nvnBlendStateSetBlendEquation(NVNblendState* blend, NVNblendEquation modeRGB, NVNblendEquation modeAlpha)` -- AI-generated guess: Sets the blend equation (add, subtract, min, max) for RGB and alpha
- `void nvnBlendStateSetAdvancedMode(NVNblendState* blend, NVNblendAdvancedMode mode)` -- AI-generated guess: Sets advanced blend mode for effects like multiply, screen, overlay
- `void nvnBlendStateSetAdvancedOverlap(NVNblendState* blend, NVNblendAdvancedOverlap overlap)` -- AI-generated guess: Sets the overlap mode for advanced blending
- `void nvnBlendStateSetAdvancedPremultipliedSrc(NVNblendState* blend, NVNboolean b)` -- AI-generated guess: Sets whether source color is premultiplied for advanced blending
- `void nvnBlendStateSetAdvancedNormalizedDst(NVNblendState* blend, NVNboolean b)` -- AI-generated guess: Sets whether destination color is normalized for advanced blending
- `int nvnBlendStateGetBlendTarget(const NVNblendState* blend)` -- AI-generated guess: Gets which render target this blend state applies to
- `void nvnBlendStateGetBlendFunc(const NVNblendState* blend, NVNblendFunc* srcFunc, NVNblendFunc* dstFunc, NVNblendFunc* srcFuncAlpha, NVNblendFunc* dstFuncAlpha)` -- AI-generated guess: Gets the blend function factors
- `void nvnBlendStateGetBlendEquation(const NVNblendState* blend, NVNblendEquation* modeRGB, NVNblendEquation* modeAlpha)` -- AI-generated guess: Gets the blend equations
- `NVNblendAdvancedMode nvnBlendStateGetAdvancedMode(const NVNblendState* blend)` -- AI-generated guess: Gets the advanced blend mode
- `NVNblendAdvancedOverlap nvnBlendStateGetAdvancedOverlap(const NVNblendState* blend)` -- AI-generated guess: Gets the overlap mode for advanced blending
- `NVNboolean nvnBlendStateGetAdvancedPremultipliedSrc(const NVNblendState* blend)` -- AI-generated guess: Gets whether source is premultiplied for advanced blending
- `NVNboolean nvnBlendStateGetAdvancedNormalizedDst(const NVNblendState* blend)` -- AI-generated guess: Gets whether destination is normalized for advanced blending

### ChannelMaskState

- `void nvnChannelMaskStateSetDefaults(NVNchannelMaskState* channelMask)` -- AI-generated guess: Sets the channel mask state to default values
- `void nvnChannelMaskStateSetChannelMask(NVNchannelMaskState* channelMask, int index, NVNboolean r, NVNboolean g, NVNboolean b, NVNboolean a)` -- AI-generated guess: Sets which color channels are writable for a render target
- `void nvnChannelMaskStateGetChannelMask(const NVNchannelMaskState* channelMask, int index, NVNboolean* r, NVNboolean* g, NVNboolean* b, NVNboolean* a)` -- AI-generated guess: Gets which color channels are writable for a render target

### ColorState

- `void nvnColorStateSetDefaults(NVNcolorState* color)` -- AI-generated guess: Sets the color state to default values
- `void nvnColorStateSetBlendEnable(NVNcolorState* color, int index, NVNboolean enable)` -- AI-generated guess: Enables or disables blending for a render target
- `void nvnColorStateSetLogicOp(NVNcolorState* color, NVNlogicOp logicOp)` -- AI-generated guess: Sets the logical operation for color blending
- `void nvnColorStateSetAlphaTest(NVNcolorState* color, NVNalphaFunc alphaFunc)` -- AI-generated guess: Sets the alpha test function
- `NVNboolean nvnColorStateGetBlendEnable(const NVNcolorState* color, int index)` -- AI-generated guess: Gets whether blending is enabled for a render target
- `NVNlogicOp nvnColorStateGetLogicOp(const NVNcolorState* color)` -- AI-generated guess: Gets the logical operation for color blending
- `NVNalphaFunc nvnColorStateGetAlphaTest(const NVNcolorState* color)` -- AI-generated guess: Gets the alpha test function

### DepthStencilState

- `void nvnDepthStencilStateSetDefaults(NVNdepthStencilState* depthStencil)` -- AI-generated guess: Sets the depth/stencil state to default values
- `void nvnDepthStencilStateSetDepthTestEnable(NVNdepthStencilState* depthStencil, NVNboolean enable)` -- AI-generated guess: Enables or disables depth testing
- `void nvnDepthStencilStateSetDepthWriteEnable(NVNdepthStencilState* depthStencil, NVNboolean enable)` -- AI-generated guess: Enables or disables depth buffer writes
- `void nvnDepthStencilStateSetDepthFunc(NVNdepthStencilState* depthStencil, NVNdepthFunc func)` -- AI-generated guess: Sets the depth comparison function
- `void nvnDepthStencilStateSetStencilTestEnable(NVNdepthStencilState* depthStencil, NVNboolean enable)` -- AI-generated guess: Enables or disables stencil testing
- `void nvnDepthStencilStateSetStencilFunc(NVNdepthStencilState* depthStencil, NVNface faces, NVNstencilFunc func)` -- AI-generated guess: Sets the stencil comparison function for front/back faces
- `void nvnDepthStencilStateSetStencilOp(NVNdepthStencilState* depthStencil, NVNface faces, NVNstencilOp fail, NVNstencilOp depthFail, NVNstencilOp depthPass)` -- AI-generated guess: Sets stencil operations for stencil fail, depth fail, and depth pass cases
- `NVNboolean nvnDepthStencilStateGetDepthTestEnable(const NVNdepthStencilState* depthStencil)` -- AI-generated guess: Gets whether depth testing is enabled
- `NVNboolean nvnDepthStencilStateGetDepthWriteEnable(const NVNdepthStencilState* depthStencil)` -- AI-generated guess: Gets whether depth buffer writes are enabled
- `NVNdepthFunc nvnDepthStencilStateGetDepthFunc(const NVNdepthStencilState* depthStencil)` -- AI-generated guess: Gets the depth comparison function
- `NVNboolean nvnDepthStencilStateGetStencilTestEnable(const NVNdepthStencilState* depthStencil)` -- AI-generated guess: Gets whether stencil testing is enabled
- `NVNstencilFunc nvnDepthStencilStateGetStencilFunc(const NVNdepthStencilState* depthStencil, NVNface faces)` -- AI-generated guess: Gets the stencil comparison function for the specified faces
- `void nvnDepthStencilStateGetStencilOp(const NVNdepthStencilState* depthStencil, NVNface faces, NVNstencilOp* fail, NVNstencilOp* depthFail, NVNstencilOp* depthPass)` -- AI-generated guess: Gets the stencil operations for the specified faces

### MultisampleState

- `void nvnMultisampleStateSetDefaults(NVNmultisampleState* multisample)` -- AI-generated guess: Sets the multisample state to default values
- `void nvnMultisampleStateSetMultisampleEnable(NVNmultisampleState* multisample, NVNboolean enable)` -- AI-generated guess: Enables or disables multisampling
- `void nvnMultisampleStateSetSamples(NVNmultisampleState* multisample, int samples)` -- AI-generated guess: Sets the number of samples for multisampling
- `void nvnMultisampleStateSetAlphaToCoverageEnable(NVNmultisampleState* multisample, NVNboolean enable)` -- AI-generated guess: Enables or disables alpha-to-coverage
- `void nvnMultisampleStateSetAlphaToCoverageDither(NVNmultisampleState* multisample, NVNboolean enable)` -- AI-generated guess: Enables or disables dithering for alpha-to-coverage
- `NVNboolean nvnMultisampleStateGetMultisampleEnable(const NVNmultisampleState* multisample)` -- AI-generated guess: Gets whether multisampling is enabled
- `int nvnMultisampleStateGetSamples(const NVNmultisampleState* multisample)` -- AI-generated guess: Gets the number of samples
- `NVNboolean nvnMultisampleStateGetAlphaToCoverageEnable(const NVNmultisampleState* multisample)` -- AI-generated guess: Gets whether alpha-to-coverage is enabled
- `NVNboolean nvnMultisampleStateGetAlphaToCoverageDither(const NVNmultisampleState* multisample)` -- AI-generated guess: Gets whether alpha-to-coverage dithering is enabled
- `void nvnMultisampleStateSetRasterSamples(NVNmultisampleState* multisample, int samples)` -- AI-generated guess: Sets the number of rasterization samples
- `int nvnMultisampleStateGetRasterSamples(NVNmultisampleState* multisample)` -- AI-generated guess: Gets the number of rasterization samples
- `void nvnMultisampleStateSetCoverageModulationMode(NVNmultisampleState* multisample, NVNcoverageModulationMode mode)` -- AI-generated guess: Sets the coverage modulation mode
- `NVNcoverageModulationMode nvnMultisampleStateGetCoverageModulationMode(const NVNmultisampleState* multisample)` -- AI-generated guess: Gets the coverage modulation mode
- `void nvnMultisampleStateSetCoverageToColorEnable(NVNmultisampleState* multisample, NVNboolean enable)` -- AI-generated guess: Enables or disables coverage-to-color output
- `NVNboolean nvnMultisampleStateGetCoverageToColorEnable(const NVNmultisampleState* multisample)` -- AI-generated guess: Gets whether coverage-to-color is enabled
- `void nvnMultisampleStateSetCoverageToColorOutput(NVNmultisampleState* multisample, int i)` -- AI-generated guess: Sets which color output receives coverage data
- `int nvnMultisampleStateGetCoverageToColorOutput(const NVNmultisampleState* multisample)` -- AI-generated guess: Gets which color output receives coverage data
- `void nvnMultisampleStateSetSampleLocationsEnable(NVNmultisampleState* multisample, NVNboolean enable)` -- AI-generated guess: Enables or disables programmable sample locations
- `NVNboolean nvnMultisampleStateGetSampleLocationsEnable(const NVNmultisampleState* multisample)` -- AI-generated guess: Gets whether programmable sample locations are enabled
- `void nvnMultisampleStateGetSampleLocationsGrid(NVNmultisampleState* multisample, int* w, int* h)` -- AI-generated guess: Gets the sample locations grid dimensions
- `void nvnMultisampleStateSetSampleLocationsGridEnable(NVNmultisampleState* multisample, NVNboolean enable)` -- AI-generated guess: Enables or disables the sample locations grid
- `NVNboolean nvnMultisampleStateGetSampleLocationsGridEnable(const NVNmultisampleState* multisample)` -- AI-generated guess: Gets whether the sample locations grid is enabled
- `void nvnMultisampleStateSetSampleLocations(NVNmultisampleState* multisample, int i1, int i2, const float* f)` -- AI-generated guess: Sets custom sample locations

### PolygonState

- `void nvnPolygonStateSetDefaults(NVNpolygonState* polygon)` -- AI-generated guess: Sets the polygon state to default values
- `void nvnPolygonStateSetCullFace(NVNpolygonState* polygon, NVNface face)` -- AI-generated guess: Sets which polygon faces to cull (front, back, or none)
- `void nvnPolygonStateSetFrontFace(NVNpolygonState* polygon, NVNfrontFace face)` -- AI-generated guess: Sets the front face winding order (clockwise or counter-clockwise)
- `void nvnPolygonStateSetPolygonMode(NVNpolygonState* polygon, NVNpolygonMode polygonMode)` -- AI-generated guess: Sets the polygon fill mode (fill, line, or point)
- `void nvnPolygonStateSetPolygonOffsetEnables(NVNpolygonState* polygon, int enables)` -- AI-generated guess: Sets which polygon types have depth offset enabled
- `NVNface nvnPolygonStateGetCullFace(const NVNpolygonState* polygon)` -- AI-generated guess: Gets which polygon faces are culled
- `NVNfrontFace nvnPolygonStateGetFrontFace(const NVNpolygonState* polygon)` -- AI-generated guess: Gets the front face winding order
- `NVNpolygonMode nvnPolygonStateGetPolygonMode(const NVNpolygonState* polygon)` -- AI-generated guess: Gets the polygon fill mode
- `NVNpolygonOffsetEnable nvnPolygonStateGetPolygonOffsetEnables(const NVNpolygonState* polygon)` -- AI-generated guess: Gets which polygon types have depth offset enabled

### VertexAttribState

- `void nvnVertexAttribStateSetDefaults(NVNvertexAttribState* attrib)` -- AI-generated guess: Sets the vertex attribute state to default values
- `void nvnVertexAttribStateSetFormat(NVNvertexAttribState* attrib, NVNformat format, ptrdiff_t relativeOffset)` -- AI-generated guess: Sets the format and offset for a vertex attribute
- `void nvnVertexAttribStateSetStreamIndex(NVNvertexAttribState* attrib, int streamIndex)` -- AI-generated guess: Sets which vertex stream this attribute reads from
- `void nvnVertexAttribStateGetFormat(const NVNvertexAttribState* attrib, NVNformat* format, ptrdiff_t* relativeOffset)` -- AI-generated guess: Gets the format and offset for a vertex attribute
- `int nvnVertexAttribStateGetStreamIndex(const NVNvertexAttribState* attrib)` -- AI-generated guess: Gets which vertex stream this attribute reads from

### VertexStreamState

- `void nvnVertexStreamStateSetDefaults(NVNvertexStreamState* stream)` -- AI-generated guess: Sets the vertex stream state to default values
- `void nvnVertexStreamStateSetStride(NVNvertexStreamState* stream, ptrdiff_t stride)` -- AI-generated guess: Sets the stride between vertices in the stream
- `void nvnVertexStreamStateSetDivisor(NVNvertexStreamState* stream, int divisor)` -- AI-generated guess: Sets the instance divisor for instanced rendering
- `ptrdiff_t nvnVertexStreamStateGetStride(const NVNvertexStreamState* stream)` -- AI-generated guess: Gets the stride between vertices in the stream
- `int nvnVertexStreamStateGetDivisor(const NVNvertexStreamState* stream)` -- AI-generated guess: Gets the instance divisor for instanced rendering

Sync
----

- `NVNboolean nvnSyncInitialize(NVNsync* sync, NVNdevice* device)` -- AI-generated guess: Initializes a sync object for the given device
- `void nvnSyncFinalize(NVNsync* sync)` -- AI-generated guess: Finalizes and destroys the sync object, releasing its resources
- `void nvnSyncSetDebugLabel(NVNsync* sync, const char* label)` -- AI-generated guess: Sets a debug label for the sync object for debugging purposes
- `NVNsyncWaitResult nvnSyncWait(const NVNsync* sync, uint64_t timeoutNs)` -- AI-generated guess: Waits on the CPU for the sync object to be signaled with a timeout in nanoseconds

Texture
-------

- `void nvnTextureBuilderSetDevice(NVNtextureBuilder* builder, NVNdevice* device)` -- AI-generated guess: Sets the device pointer for the given texture builder
- `void nvnTextureBuilderSetDefaults(NVNtextureBuilder* builder)` -- AI-generated guess: Sets the texture builder to default values
- `void nvnTextureBuilderSetFlags(NVNtextureBuilder* builder, int flags)` -- AI-generated guess: Sets flags for texture creation
- `void nvnTextureBuilderSetTarget(NVNtextureBuilder* builder, NVNtextureTarget target)` -- AI-generated guess: Sets the texture target type (1D, 2D, 3D, cube, array, etc.)
- `void nvnTextureBuilderSetWidth(NVNtextureBuilder* builder, int width)` -- AI-generated guess: Sets the width of the texture
- `void nvnTextureBuilderSetHeight(NVNtextureBuilder* builder, int height)` -- AI-generated guess: Sets the height of the texture
- `void nvnTextureBuilderSetDepth(NVNtextureBuilder* builder, int depth)` -- AI-generated guess: Sets the depth of the texture (for 3D textures or array layers)
- `void nvnTextureBuilderSetSize1D(NVNtextureBuilder* builder, int size)` -- AI-generated guess: Sets the size for a 1D texture
- `void nvnTextureBuilderSetSize2D(NVNtextureBuilder* builder, int width, int height)` -- AI-generated guess: Sets the width and height for a 2D texture
- `void nvnTextureBuilderSetSize3D(NVNtextureBuilder* builder, int width, int height, int depth)` -- AI-generated guess: Sets the width, height, and depth for a 3D texture
- `void nvnTextureBuilderSetLevels(NVNtextureBuilder* builder, int numLevels)` -- AI-generated guess: Sets the number of mipmap levels
- `void nvnTextureBuilderSetFormat(NVNtextureBuilder* builder, NVNformat format)` -- AI-generated guess: Sets the pixel format of the texture
- `void nvnTextureBuilderSetSamples(NVNtextureBuilder* builder, int samples)` -- AI-generated guess: Sets the number of samples for multisampled textures
- `void nvnTextureBuilderSetSwizzle(NVNtextureBuilder* builder, NVNtextureSwizzle r, NVNtextureSwizzle g, NVNtextureSwizzle b, NVNtextureSwizzle a)` -- AI-generated guess: Sets the component swizzle mapping for texture reads
- `void nvnTextureBuilderSetDepthStencilMode(NVNtextureBuilder* builder, NVNtextureDepthStencilMode mode)` -- AI-generated guess: Sets the depth/stencil read mode for depth-stencil textures
- `size_t nvnTextureBuilderGetStorageSize(const NVNtextureBuilder* builder)` -- AI-generated guess: Gets the required storage size in bytes for the texture
- `size_t nvnTextureBuilderGetStorageAlignment(const NVNtextureBuilder* builder)` -- AI-generated guess: Gets the required storage alignment for the texture
- `void nvnTextureBuilderSetStorage(NVNtextureBuilder* builder, NVNmemoryPool* pool, ptrdiff_t offset)` -- AI-generated guess: Sets the memory pool and offset for texture storage
- `void nvnTextureBuilderSetPackagedTextureData(NVNtextureBuilder* builder, const void* data)` -- AI-generated guess: Sets prepackaged texture data pointer
- `void nvnTextureBuilderSetPackagedTextureLayout(NVNtextureBuilder* builder, const NVNpackagedTextureLayout* layout)` -- AI-generated guess: Sets the layout information for packaged texture data
- `void nvnTextureBuilderSetStride(NVNtextureBuilder* builder, ptrdiff_t stride)` -- AI-generated guess: Sets the row stride for linear textures
- `void nvnTextureBuilderSetGLTextureName(NVNtextureBuilder* builder, uint32_t name)` -- AI-generated guess: Sets an OpenGL texture name for interoperability
- `NVNstorageClass nvnTextureBuilderGetStorageClass(const NVNtextureBuilder* builder)` -- AI-generated guess: Gets the storage class (linear, tiled, etc.) for the texture
- `NVNtextureFlags nvnTextureBuilderGetFlags(const NVNtextureBuilder* builder)` -- AI-generated guess: Gets the flags for the texture
- `NVNtextureTarget nvnTextureBuilderGetTarget(const NVNtextureBuilder* builder)` -- AI-generated guess: Gets the texture target type
- `int nvnTextureBuilderGetWidth(const NVNtextureBuilder* builder)` -- AI-generated guess: Gets the width of the texture
- `int nvnTextureBuilderGetHeight(const NVNtextureBuilder* builder)` -- AI-generated guess: Gets the height of the texture
- `int nvnTextureBuilderGetDepth(const NVNtextureBuilder* builder)` -- AI-generated guess: Gets the depth of the texture
- `int nvnTextureBuilderGetLevels(const NVNtextureBuilder* builder)` -- AI-generated guess: Gets the number of mipmap levels
- `NVNformat nvnTextureBuilderGetFormat(const NVNtextureBuilder* builder)` -- AI-generated guess: Gets the pixel format of the texture
- `int nvnTextureBuilderGetSamples(const NVNtextureBuilder* builder)` -- AI-generated guess: Gets the number of samples for multisampled textures
- `void nvnTextureBuilderGetSwizzle(const NVNtextureBuilder* builder, NVNtextureSwizzle* r, NVNtextureSwizzle* g, NVNtextureSwizzle* b, NVNtextureSwizzle* a)` -- AI-generated guess: Gets the component swizzle mapping
- `NVNtextureDepthStencilMode nvnTextureBuilderGetDepthStencilMode(const NVNtextureBuilder* builder)` -- AI-generated guess: Gets the depth/stencil read mode
- `const void* nvnTextureBuilderGetPackagedTextureData(const NVNtextureBuilder* builder)` -- AI-generated guess: Gets the packaged texture data pointer
- `ptrdiff_t nvnTextureBuilderGetStride(const NVNtextureBuilder* builder)` -- AI-generated guess: Gets the row stride for linear textures
- `void nvnTextureBuilderGetSparseTileLayout(const NVNtextureBuilder* builder, NVNtextureSparseTileLayout* layout)` -- AI-generated guess: Gets the sparse tile layout information for the texture
- `uint32_t nvnTextureBuilderGetGLTextureName(const NVNtextureBuilder* builder)` -- AI-generated guess: Gets the OpenGL texture name
- `size_t nvnTextureBuilderGetZCullStorageSize(const NVNtextureBuilder* builder)` -- AI-generated guess: Gets the required storage size for Z-cull data
- `NVNmemoryPool nvnTextureBuilderGetMemoryPool(const NVNtextureBuilder* builder)` -- AI-generated guess: Gets the memory pool associated with the texture builder
- `ptrdiff_t nvnTextureBuilderGetMemoryOffset(const NVNtextureBuilder* builder)` -- AI-generated guess: Gets the offset within the memory pool for the texture
- `NVNboolean nvnTextureInitialize(NVNtexture* texture, const NVNtextureBuilder* builder)` -- AI-generated guess: Initializes a texture object using the provided builder configuration
- `size_t nvnTextureGetZCullStorageSize(const NVNtexture* texture)` -- AI-generated guess: Gets the required storage size for Z-cull data for the texture
- `void nvnTextureFinalize(NVNtexture* texture)` -- AI-generated guess: Finalizes and destroys the texture, releasing its resources
- `void nvnTextureSetDebugLabel(NVNtexture* texture, const char* label)` -- AI-generated guess: Sets a debug label for the texture for debugging purposes
- `NVNstorageClass nvnTextureGetStorageClass(const NVNtexture* texture)` -- AI-generated guess: Gets the storage class (linear, tiled, etc.) of the texture
- `ptrdiff_t nvnTextureGetViewOffset(const NVNtexture* texture, const NVNtextureView* view)` -- AI-generated guess: Gets the offset for a texture view within the texture
- `NVNtextureFlags nvnTextureGetFlags(const NVNtexture* texture)` -- AI-generated guess: Gets the flags for the texture
- `NVNtextureTarget nvnTextureGetTarget(const NVNtexture* texture)` -- AI-generated guess: Gets the texture target type
- `int nvnTextureGetWidth(const NVNtexture* texture)` -- AI-generated guess: Gets the width of the texture
- `int nvnTextureGetHeight(const NVNtexture* texture)` -- AI-generated guess: Gets the height of the texture
- `int nvnTextureGetDepth(const NVNtexture* texture)` -- AI-generated guess: Gets the depth of the texture
- `int nvnTextureGetLevels(const NVNtexture* texture)` -- AI-generated guess: Gets the number of mipmap levels
- `NVNformat nvnTextureGetFormat(const NVNtexture* texture)` -- AI-generated guess: Gets the pixel format of the texture
- `int nvnTextureGetSamples(const NVNtexture* texture)` -- AI-generated guess: Gets the number of samples for the texture
- `void nvnTextureGetSwizzle(const NVNtexture* texture, NVNtextureSwizzle* r, NVNtextureSwizzle* g, NVNtextureSwizzle* b, NVNtextureSwizzle* a)` -- AI-generated guess: Gets the component swizzle mapping of the texture
- `NVNtextureDepthStencilMode nvnTextureGetDepthStencilMode(const NVNtexture* texture)` -- AI-generated guess: Gets the depth/stencil read mode of the texture
- `ptrdiff_t nvnTextureGetStride(const NVNtexture* texture)` -- AI-generated guess: Gets the row stride for linear textures
- `NVNtextureAddress nvnTextureGetTextureAddress(const NVNtexture* texture)` -- AI-generated guess: Gets the GPU address of the texture
- `void nvnTextureGetSparseTileLayout(const NVNtexture* texture, NVNtextureSparseTileLayout* layout)` -- AI-generated guess: Gets the sparse tile layout information of the texture
- `void nvnTextureWriteTexels(const NVNtexture* texture, const NVNtextureView* view, const NVNcopyRegion* region, const void* p)` -- AI-generated guess: Writes texel data from CPU memory to the texture
- `void nvnTextureWriteTexelsStrided(const NVNtexture* texture, const NVNtextureView* view, const NVNcopyRegion* region, const void* p, ptrdiff_t o1, ptrdiff_t o2)` -- AI-generated guess: Writes texel data with custom stride from CPU memory to the texture
- `void nvnTextureReadTexels(const NVNtexture* texture, const NVNtextureView* view, const NVNcopyRegion* region, void* p)` -- AI-generated guess: Reads texel data from the texture to CPU memory
- `void nvnTextureReadTexelsStrided(const NVNtexture* texture, const NVNtextureView* view, const NVNcopyRegion* region, void* p, ptrdiff_t o1, ptrdiff_t o2)` -- AI-generated guess: Reads texel data with custom stride from the texture to CPU memory
- `void nvnTextureFlushTexels(const NVNtexture* texture, const NVNtextureView* view, const NVNcopyRegion* region)` -- AI-generated guess: Flushes a region of texture data to make CPU writes visible to GPU
- `void nvnTextureInvalidateTexels(const NVNtexture* texture, const NVNtextureView* view, const NVNcopyRegion* region)` -- AI-generated guess: Invalidates a region of texture data to make GPU writes visible to CPU
- `NVNmemoryPool nvnTextureGetMemoryPool(const NVNtexture* texture)` -- AI-generated guess: Gets the memory pool associated with the texture
- `ptrdiff_t nvnTextureGetMemoryOffset(const NVNtexture* texture)` -- AI-generated guess: Gets the offset within the memory pool where the texture is located
- `int nvnTextureGetStorageSize(const NVNtexture* texture)` -- AI-generated guess: Gets the storage size of the texture in bytes
- `NVNboolean nvnTextureCompare(const NVNtexture* texture1, const NVNtexture* texture2)` -- AI-generated guess: Compares two textures for equality
- `uint64_t nvnTextureGetDebugID(const NVNtexture* texture)` -- AI-generated guess: Gets a unique debug identifier for the texture

TexturePool
-----------

- `NVNboolean nvnTexturePoolInitialize(NVNtexturePool* texturePool, const NVNmemoryPool* memoryPool, ptrdiff_t offset, int numDescriptors)` -- AI-generated guess: Initializes a texture descriptor pool in a memory pool at the specified offset
- `void nvnTexturePoolSetDebugLabel(NVNtexturePool* pool, const char* label)` -- AI-generated guess: Sets a debug label for the texture pool for debugging purposes
- `void nvnTexturePoolFinalize(NVNtexturePool* pool)` -- AI-generated guess: Finalizes and destroys the texture pool, releasing its resources
- `void nvnTexturePoolRegisterTexture(const NVNtexturePool* pool, int id, const NVNtexture* texture, const NVNtextureView* view)` -- AI-generated guess: Registers a texture (with optional view) into the pool at the specified descriptor ID
- `void nvnTexturePoolRegisterImage(const NVNtexturePool* pool, int id, const NVNtexture* texture, const NVNtextureView* view)` -- AI-generated guess: Registers an image (texture for read/write access) into the pool at the specified descriptor ID
- `const NVNmemoryPool* nvnTexturePoolGetMemoryPool(const NVNtexturePool* pool)` -- AI-generated guess: Gets the memory pool associated with the texture pool
- `ptrdiff_t nvnTexturePoolGetMemoryOffset(const NVNtexturePool* pool)` -- AI-generated guess: Gets the offset within the memory pool where the texture pool is located
- `int nvnTexturePoolGetSize(const NVNtexturePool* pool)` -- AI-generated guess: Gets the number of texture descriptors in the pool

TextureView
-----------

- `void nvnTextureViewSetDefaults(NVNtextureView* view)` -- AI-generated guess: Sets the texture view to default values
- `void nvnTextureViewSetLevels(NVNtextureView* view, int baseLevel, int numLevels)` -- AI-generated guess: Sets the range of mipmap levels visible through the view
- `void nvnTextureViewSetLayers(NVNtextureView* view, int minLayer, int numLayers)` -- AI-generated guess: Sets the range of array layers visible through the view
- `void nvnTextureViewSetFormat(NVNtextureView* view, NVNformat format)` -- AI-generated guess: Sets the pixel format for the view (for format reinterpretation)
- `void nvnTextureViewSetSwizzle(NVNtextureView* view, NVNtextureSwizzle r, NVNtextureSwizzle g, NVNtextureSwizzle b, NVNtextureSwizzle a)` -- AI-generated guess: Sets the component swizzle mapping for the view
- `void nvnTextureViewSetDepthStencilMode(NVNtextureView* view, NVNtextureDepthStencilMode mode)` -- AI-generated guess: Sets the depth/stencil read mode for the view
- `void nvnTextureViewSetTarget(NVNtextureView* view, NVNtextureTarget target)` -- AI-generated guess: Sets the texture target type for the view
- `NVNboolean nvnTextureViewGetLevels(const NVNtextureView* view, int* baseLevel, int* numLevels)` -- AI-generated guess: Gets the range of mipmap levels from the view
- `NVNboolean nvnTextureViewGetLayers(const NVNtextureView* view, int* minLayer, int* numLayers)` -- AI-generated guess: Gets the range of array layers from the view
- `NVNboolean nvnTextureViewGetFormat(const NVNtextureView* view, NVNformat* format)` -- AI-generated guess: Gets the pixel format from the view
- `NVNboolean nvnTextureViewGetSwizzle(const NVNtextureView* view, NVNtextureSwizzle* r, NVNtextureSwizzle* g, NVNtextureSwizzle* b, NVNtextureSwizzle* a)` -- AI-generated guess: Gets the component swizzle mapping from the view
- `NVNboolean nvnTextureViewGetDepthStencilMode(const NVNtextureView* view, NVNtextureDepthStencilMode* mode)` -- AI-generated guess: Gets the depth/stencil read mode from the view
- `NVNboolean nvnTextureViewGetTarget(const NVNtextureView* view, NVNtextureTarget* target)` -- AI-generated guess: Gets the texture target type from the view
- `NVNboolean nvnTextureViewCompare(const NVNtextureView* view1, const NVNtextureView* view2)` -- AI-generated guess: Compares two texture views for equality

Window
------

- `void nvnWindowBuilderSetDevice(NVNwindowBuilder* builder, NVNdevice* device)` -- AI-generated guess: Sets the device pointer for the given window builder
- `void nvnWindowBuilderSetDefaults(NVNwindowBuilder* builder)` -- AI-generated guess: Sets the window builder to default values
- `void nvnWindowBuilderSetNativeWindow(NVNwindowBuilder* builder, NVNnativeWindow nativeWindow)` -- AI-generated guess: Sets the native window handle for the window
- `void nvnWindowBuilderSetTextures(NVNwindowBuilder* builder, int numTextures, NVNtexture* const* textures)` -- AI-generated guess: Sets the textures to use for the swapchain
- `void nvnWindowBuilderSetPresentInterval(NVNwindowBuilder* builder, int presentInterval)` -- AI-generated guess: Sets the present interval (vsync setting) for the window
- `NVNnativeWindow nvnWindowBuilderGetNativeWindow(const NVNwindowBuilder* builder)` -- AI-generated guess: Gets the native window handle from the builder
- `int nvnWindowBuilderGetPresentInterval(const NVNwindowBuilder* builder)` -- AI-generated guess: Gets the present interval from the builder
- `NVNboolean nvnWindowInitialize(NVNwindow* window, const NVNwindowBuilder* builder)` -- AI-generated guess: Initializes a window object using the provided builder configuration
- `void nvnWindowFinalize(NVNwindow* window)` -- AI-generated guess: Finalizes and destroys the window, releasing its resources
- `void nvnWindowSetDebugLabel(NVNwindow* window, const char* label)` -- AI-generated guess: Sets a debug label for the window for debugging purposes
- `NVNwindowAcquireTextureResult nvnWindowAcquireTexture(NVNwindow* window, NVNsync* textureAvailableSync, int* textureIndex)` -- AI-generated guess: Acquires a texture from the window swapchain for rendering
- `NVNnativeWindow nvnWindowGetNativeWindow(const NVNwindow* window)` -- AI-generated guess: Gets the native window handle from the window
- `int nvnWindowGetPresentInterval(const NVNwindow* window)` -- AI-generated guess: Gets the present interval (vsync setting) from the window
- `void nvnWindowSetPresentInterval(NVNwindow* window, int presentInterval)` -- AI-generated guess: Sets the present interval (vsync setting) for the window
- `void nvnWindowSetCrop(NVNwindow* window, int x, int y, int w, int h)` -- AI-generated guess: Sets the crop rectangle for window presentation
- `void nvnWindowGetCrop(const NVNwindow* window, NVNrectangle* rectangle)` -- AI-generated guess: Gets the crop rectangle from the window
