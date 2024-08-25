---
tags:
  - gsoc
---
## Protocol Selection
First, we need some context around how wlroots-based wayland compositors handle screen capture. There are mainly two protocols that enable this:
1.  wlr-screencopy-unstable-v1
2.  wlr-export-dmabuf-unstable-v1

Libwayshot in its present state uses only wlr-screencopy. The major difference between these two is in the type of buffers each of them deal with:

- In wlr-screencopy the compositor will notify the client of upto two different types of available buffer types: wl_shm shared memory buffers and linux dmabuf based DMA-BUF buffers. The client must create one of these and pass it to the compositor for it to copy screen content into.
- libwayshot presently uses only the wl_shm mode of wlr-screencopy.
- wlr-screencopy is capable of copy on damage functionality: the compositor signals a filled buffer only when the captured region has been damaged.
- wlr-screencopy can choose to capture certain sub-regions of outputs.
- Buffers here must be created by the client: wl_shm buffers can be created by memfd_create and dmabuf buffers can be created by the linux_dmabuf wayland protocol.

- A copy is unavoidable here. However, VRAM->CPU copies can still be avoided in dmabuf mode as dmabuf buffer copies occur entirely on the GPU. These are relatively more efficient depending on the hardware and drivers.

- wlr-export-dmabuf deals solely with dmabuf buffers. In this protocol the compositor passes off dmabuf handles to the client for each frame. No copying will be involved here.

- The client can choose which output to capture but it cannot select regions under this output.
-  No copy with damage is available.
- The client need not create dmabuf buffers of its own.

Overall the tradeoffs between these approaches are as follows:
-   wlr-screencopy needs to perform dmabuf copies but these are pretty efficient and any performance losses here compared to wlr-export-dmabuf here should be somewhat offset by the availability of copy with damage requiring less data to be processed overall. Ultimately benchmarking will be required to understand the performance gaps.
-   wlr-screencopy is more complex to implement than wlr-export-dmabuf but this is offset by the fact that most of this complexity is already present in libwayshot while wlr-export-dmabuf will have to be implemented from scratch.

-   The overall trade off appears to be between the flexibility and universal availability of wlr-screencopy versus the simplicity and (arguably) performance of wlr-export-dmabuf.

After consideration of the above tradeoffs and discussion with organization contributors, I have decided to implement the wlr-screencopy version of the dmabuf backend for this project.
## Design
Here’s a diagram containing high level overview of what I propose to initially build. The components in dotted lines will be implemented as part of GSoC.
![[proposal_image1.png]]

The main steps in implementing this are:
1. Obtaining a GPU Handle
2. Finding parameters to be used for the dmabuf object
3. dmabuf backed wl-buffer creation
4. Passing the wl-buffer to compositor for copy
5. Returning the resulting buffer to the library user

Let’s have a high-level overview for each of these steps:
### Obtaining A GPU handle
For creating a dmabuf, we need a DRM render node to create it on. 
I plan to use [drm-rs](https://crates.io/crates/drm) , bindings for libdrm, for this purpose. This library, belonging to the Smithay project also has a sister library [GBM crate](https://crates.io/crates/gbm) which can be used to create backing dmabufs for wayland-rs's dmabuf object.

drm-rs requires a file name to the render node in order to open it, usually located in `/dev/dri`:

>$ > la /dev/dri
total 0
drwxr-xr-x  2 root root         80 May 24 10:16 by-path/
crw-rw----+ 1 root video  226,   1 May 24 20:15 card1
crw-rw-rw-  1 root render 226, 128 May 24 10:16 renderD128

There are a few ways to obtain the render node path:
- From the user: the user can pass the path as an string argument
- Using the [drm-lease-v1 protocol](https://wayland.app/protocols/drm-lease-v1#wp_drm_lease_request_v1) , this protocol can be used to lease a DRM device for high performance purposes.
- Fallback to /dev/renderD128 if drm-lease protocol is unavailable

Once we have a drm-rs `Card` object available, we're ready to create a DMA-BUF on it via GBM. Create a `GBMDevice` using the `Card` object. All of this can be done when the `WayshotConnection` object is initialized and the `GBMDevice` object can be stored in that struct.
### Finding parameters to be used for the DMA-BUF
The key data structure that deals with individual frames in libwayshot is the `CaptureFrameState` struct. It is this struct that interacts with the wayland-rs screencopy API to capture individual frames. Here's how each frame capture happens:
- The `capture_output_frame_get_state` function in `WayshotConnection` initializes a [screencopy_manager](https://wayland.app/protocols/wlr-screencopy-unstable-v1#zwlr_screencopy_manager_v1) with appropriate region selection parameters and then uses it to create a CaptureFrameState struct along with an associated [screencopy frame](https://wayland.app/protocols/wlr-screencopy-unstable-v1#zwlr_screencopy_frame_v1)
- The wayland-rs `Dispatch` implementation for `CaptureFrameState` in `dispatch.rs` listens for a `buffer` event that contains information that can be used to create a [wl_shm](https://wayland.app/protocols/wayland#wl_shm) object to be passed to the compositor for it to copy the frame data into.
	- This includes things like frame height, frame width, stride and the fourcc color format for the buffer.
	- For our project, we'll also have to listen for `linux_dmabuf` events that do the same thing for dmabufs.
- The function `capture_output_frame_inner` is what initializes the `wl_shm` buffer and calls the screencopy [copy](https://wayland.app/protocols/wlr-screencopy-unstable-v1#zwlr_screencopy_frame_v1:request:copy) request to ask the compositor to copy the frame into the buffer.
	- For our project, we'll instead have to create a dmabuf object on the GPU we initialized earlier and pass that instead. This is detailed next.
### DMA-BUF Backed wl-buffer Creation
Creating a DMA-BUF is a somewhat more involved process than creating shm buffers. While the latter can be created using a few system calls, the general mechanism for creating DMA-BUFs in wayland is by using the stable linux-dmabuf protocol.

The general steps involved are:
- Obtain the ZwpLinuxDmabufV1 interface from the compositor.
	- The wayland global registry can be queried to do this.
-  Use ZwpLinuxDmabufV1 to create a ZwpLinuxBufferParamsV1 interface object.
	-  Relevant request: [create_params](https://wayland.app/protocols/linux-dmabuf-v1#zwp_linux_dmabuf_v1:request:create_params)
- Use the ZwpLinuxBufferParamsV1 interface to build a dmabuf backed wl-buffer:
	- Use the frame parameters obtained earlier from screencopy's `linux_dmabuf` event to create a GBM buffer object (`create_buffer_object`)
	- Get a file descriptor to the buffer object and pass it to the ZwpLinuxBufferParamsV1 object's add method to connect the wayland object to it's backing buffer
	-  Use the [create_immed](https://wayland.app/protocols/linux-dmabuf-v1#zwp_linux_buffer_params_v1:request:create_immed)method to immediately initialize a wl_buffer object.

### Passing the wl-buffer to compositor for copy
Once the requisite wl-buffer has been created a handle to it needs to be passed to the compositor so that the compositor can fill it in. The steps for this are:
-  Create the dmabuf backed wl-buffer using the linux-dmabuf protocol as discussed above.
- Pass the dmabuf backed wl-buffer to the compositor to be filled in. libwayshot already does this with the copy request on the ZwlrScreencopyFrameV1 interface. The process is the same for dmabuf backed wl-buffers.
### Returning the resulting buffer to the library user
Once the copy is complete libwayshot has to pass the resultant screenshot to the caller in a usable form. This can be done by returning one or all of the following objects:
- GBM Buffer Objects
- wayland-rs wl-buffer
- An EGL Image, this will add additional dependencies to the library but will add more flexibility to what can be done with the buffers. 
> [!question] What is EGL?
> From the [EGL specification](http://www.khronos.org/registry/egl/specs/eglspec.1.5.pdf) "EGL [is] an interface between rendering APIs such as OpenCL, OpenGL, OpenGL ES or OpenVG (referred to collectively as client APIs) and one or more underlying platforms (typically window systems such as X11)" (or in our case - Wayland)
- A OpenGL texture. This can be obtained from the EGLImage, bound to the GL context and can then be rendered on screen.
