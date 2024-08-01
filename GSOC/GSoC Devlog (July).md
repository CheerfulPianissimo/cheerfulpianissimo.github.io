# Weeks 1-2
Most of this time was spent catching up on OpenGL, EGL and the interactions thereof.
Some useful resources I perused:
- https://learnopengl.com/ for learning the foundational OpenGL concepts along with a simple triangle demo.
- https://registry.khronos.org/EGL/ the EGL specs. EGL doesn't have as many resources as OpenGL to learn from so often you have to look at the base specs themselves. The particular extensions I had to use in my project were described by:
	- https://registry.khronos.org/EGL/extensions/EXT/EGL_EXT_image_dma_buf_import.txt for converting a GBM BufferObject into an EGLImage
	- https://registry.khronos.org/OpenGL/extensions/OES/OES_EGL_image_external.txt : for importing the EGLImage as a usable GL texture
Also invaluable was the [Exchanging pixel buffers](https://www.kernel.org/doc/html//latest/userspace-api/dma-buf-alloc-exchange.html#exchanging-pixel-buffers "Permalink to this heading") document which helped me make sense of the various parameters demanded by the above functions. 

# Weeks 3-5
## DMABUF-> OpenGL Texture Pipeline MVP Implementation
I decided to tackle this part of the project in the same manner that I tackled the first phase, by first building an MVP to get the pipeline working and then extracting the work back into libwayshot. For this phase, my mentor @ShinyZenith's educational project [wayland-egl-ctx](https://github.com/Shinyzenith/wayland-egl-ctx) proved to be invaluable. It already had a lot of the finicky EGL and OpenGL setup code for me to build off of. 

With this solid foundation in place, I modified the demo to build a rectangle rather than a triangle.![[Pasted image 20240730174837.png]]
As my first experience dealing directly with C code under unsafe blocks, I messed up the variable types and passed a wrong array size to some of the GL code. Debugging this was a good learning experience.

With the rectangle rendering in place, all that was left was turning the screen-capture into a texture and binding it to this rectangle. 

The crux of this step is the [eglCreateImage](https://docs.rs/khronos-egl/latest/khronos_egl/api/trait.EGL1_5.html#tymethod.eglCreateImage) function. Pass the right constants and parameters to it and returns an EGLImage containing the required data. 

Actually implementing that step proved to be challenging. I quickly got the function call compiling but it kept failing with a undescriptive "BadParameter" error message.

After trying out various permutations and combinations of the parameters for hours I gave up and decided to properly debug it via GDB. Tunneling down the function call stack finally gave me the root of the error: a failing ioctl call deep in libdrm. It had an errno of 9 "Bad file descriptor" which clued me into what the actual issue was. I was using .as_raw_fd() to obtain the dmabuf file descriptors but this caused the fd to be invalid by the time it was used in eglCreateImage. Using into_raw_fd() immediately solved the issue at the cost of a lot of time spent debugging. The debugging experience I gained in this phase was entirely worth it.

With the EGLImage in hand, it was finally time to convert it and bind it to a texture. Doing this required the somewhat verbose [EGLImageTargetTexture2DOES](https://registry.khronos.org/OpenGL/extensions/OES/OES_EGL_image_external.txt)function. The name is actually pretty comprehensible if you look at it for a while:
- EGLImage is the source
- It Targets a GL Texture
	- This GL texture is 2D
- OES stands for OpenGL ES 
This step thankfully went smoothly and before I knew I had the MVP working:
![[Pasted image 20240730180003.png]]
Messing around in the fragment shader gave me color inversion:
![[Pasted image 20240730180043.png]]
With the MVP now functional, I committed it to the main project under the examples folder in libwayshot. 

I also decided to try out continuously refreshing the image in the texture to create a live screen mirror. It worked but I the performance was disappointing. I was not entirely surprised by this outcome - as discussed in  last month's devlog, libwayshot is still a screen shot library rather than a screen capture library and changing this will require some deeper changes in it's structure to reduce repetitive allocations and setup. This topic merits some discussion with the mentors.

## What Next?
With one month of GSoC remaining, it's time to move on to the the final stage of this project: implementing the dmabuf version of the high-level API. There are also a few other todos:
- Proper DRM modifier handling, the present code only works for single planar formats, it also fails on the wlroots Vulkan renderer for unclear reasons. I'm presently going along with whatever format GBM chooses as it's default but to be correct we may need to query the available formats via linux_dmabuf_feedback and pass that list to GBM. linux_dmabuf_feedback also has the concept of "main device" and "tranche device" which I haven't fully gotten my head around. One issue here is that I don't actually have the hardware to test other formats and modifiers.
- Merging textures from multiple displays in a way analogous to how it is done with DynamicImage. I'm not sure we should actually be doing this but I think this can be done after doing the Dmabuf->GL  texture conversion API for the individual displays.
- Verifying memory safety for GL, EGL and GBM handling code. I'll try to be careful but ultimately I have little experience dealing with Rust FFI and more experienced eyes will be very helpful for this.