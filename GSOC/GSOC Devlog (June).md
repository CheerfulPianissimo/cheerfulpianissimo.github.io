# Week 1
## Refactoring the MVP into not-a-MVP
Spent some time slowly turning the MVP into a draft API in libwayshot.
Some thoughts:
- We need to make libwayshot work in two modes, basically:
	- The shm mode in which libwayshot presently works
		- Some relevant data-structures:
			- FrameCopy:
				```
				struct FrameCopy {
					pub frame_format: FrameFormat,
				    pub frame_color_type: ColorType,
				    pub frame_mmap: MmapMut,
				    pub transform: wl_output::Transform,
				    /// Logical region with the transform already applied.
				    pub logical_region: LogicalRegion,
				    pub physical_size: Size,
				}
				```
				This struct stores the frame metadata along with the buffer that backs it
			- FrameGuard:
				```
				pub struct FrameGuard {
					pub buffer: WlBuffer,
					pub shm_pool: WlShmPool,
					}
				```
				This struct serves as a "Guard" for that wayland-rs proxy objects that calls `destroy()` on them when dropped
		- The DMA-BUF mode:
			- In this mode we'll have to store a `ZwpLinuxDmabufV1` to create the WlBuffers needed for copy(). One instance of this can be reused across multiple frames.
			- A GBMDevice to create GBM backing buffer objects on. Again, one instance of this can be reused across frames.
	
- Implemented a new constructor to initialize a GBM Device and store it in the main state of the program `WayshotConnection`
	- Also created a 
		```
		struct DMABUFState {
				linux_dmabuf: ZwpLinuxDmabufV1,
				gbmdev: GBMDevice<Card>,
			}
		```
		which stores reusable state for dmabuf mode in WayshotConnection
		-  Not sure if WayshotConnection is the right place for this, presently most of this state for shm screencopy is stored in the intermediate `capture_*` functions.

- In general there's stuff that needs to be done once like init, getting output information or creating a screencopy manager with the output and region selected and then there is the stuff that needs to be done every frame: getting the buffer parameters, properly initializing the buffers, and actually copying the frame.
	- Presently wayshot is not geared to really separate this: for instance, it creates a new screencopy manager for every frame initialized  and it's clearly designed to be a single use screenshot library rather than a continuous screen-capture system. 
	- I intend to build out the dmabuf functionality in the current framework, will see if it can be refactored out later to more clearly separate the frame capture init from the actual frame capture loop.
## Questions for Week 1
- How to distribute program state?
	- There are multiple places to store it: `WayshotConnection`, a mostly unused `WayshotState` struct and the `CaptureFrameState` struct
	- Some state changes from frame to frame:
		- DMABUF creation parameters like frame formats, frame size, etc
	- While other state remains the same between frames:
		- The GBMDevice, the `ZwpLinuxDmabufV1` proxy object
- How to let intermediate functions know which mode we are in?
	- The protocol logic is presently spread across several functions, we want to reuse as many of these as possible while also allowing them to work with the new buffer type. 
	- As such, they need to figure out what mode the program is in and act accordingly while setting up the state.
  - How to modify the core structs so that they work for both modes?
	  - Should the choice between the two modes be made at time of `WayshotConnection` creation?
		  - Or should it be made for each individual frame - this is the natural state of the protocol.
	  - How to store DMABUFs instead of SHMs in structs like `FrameCopy` or `FrameGuard`?
	  - How to reconcile two different types of frame metadata: 
		  - `Buffer` events give the parameters for shm mode and is stored in `FrameFormat` struct in `CaptureFrameState` presently
		  - Should the equivalent parameters for `dmabuf` event be stored in `FrameFormat` (even though the `dmabuf` event doesn't give the stride parameter that `Buffer` gives and `FrameFormat` requires) or should a new struct be created specifically for storing `dmabuf` parameters?
# Week 4
I spent Weeks 2 and 3 busy with my semester exams. In that time, I outlined a strategy to refactor libwayshot and implemented it in Week 4 after discussing the above questions with my mentors.

I would modify the core data structures in libwayshot and then make code changes wherever the compiler demands it.  Then I would modify/add whatever functions are required for the API.

Reading the code, I discovered that there are three levels of abstraction in libwayshot:
- The `screenshot_*` series of functions: These are the public API and mostly wrap around one of the `capture_*` series of functions.
- The `capture_frame_copy*` and `capture_output_frame_shm_fd` series of functions: `capture_output_frame_shm_fd` is a part of the public libwayshot API  that allows for low-level access to the screencapture data in the form of file descriptors and a `FrameGuard` struct that contains a `WlBuffer`. The `capture_frame_copy*` functions handles the creation of a `FrameCopy` struct which is a sort of intermediate data structure between raw shared memory and a `DynamicImage`, these functions are used by the `screenshot*` function layer above and are not public.
- The `capture_output_frame_get_state` and `capture_output_frame_inner` functions: These two functions deal with the nitty-gritty of interacting with the wlr-screencopy protocol. The primary data structure here is `CaptureFrameState`.
	- `capture_output_frame_get_state` sets up a [zwlr_screencopy_manager_v1](https://wayland.app/protocols/wlr-screencopy-unstable-v1#zwlr_screencopy_manager_v1 "zwlr_screencopy_manager_v1 interface") with the correct sub-region and the output to be captured to get a  [zwlr_screencopy_frame_v1](https://wayland.app/protocols/wlr-screencopy-unstable-v1#zwlr_screencopy_frame_v1 "zwlr_screencopy_frame_v1 interface") 
	- `capture_output_frame_inner` manages the copy of frame data from the created zwlr_screencopy_frame_v1 into the shared memory buffers.
	- `capture_output_frame_shm_fd` and it's sister function `capture_output_frame_shm_from_file` call the above two functions one after the other to handle everything to do with wlr-screencopy.
## First Draft API Implementation
For the first phase I decided to create dmabuf version of the lower two levels of abstraction. My goal was to create a dmabuf equivalent for `capture_output_frame_shm_fd` which would provide low level access to the dma buffer objects and a wl-buffer. 

To this goal, I created the new function variants `capture_output_frame_get_state_dmabuf` and `capture_output_frame_inner_dmabuf` to replicate the lowest layer and `capture_output_frame_dmabuf` to replicate the public API on the level above. `capture_output_frame_dmabuf` would return a GBM BufferObject, a FrameGuard containing a WlBuffer and the frame format descriptor.

Two new structs `DMAFrameFormat` and `DMAFrameGuard` were created as counterparts to `FrameFormat` and `FrameGuard`. 

For the most part adapting the code from the MVP into these functions was straightforward with the shm variants as a reference.

One part I have some trouble with is error handling. Using unwrap() in library code is generally frowned upon but I haven't figured out a good way to pass up errors to the caller without running into compiler errors like "? couldn't convert the error to `error::Error`
the question mark operation (`?`) implicitly performs a conversion on the error value using the `From` trait". This requires some research.

The next phase will be testing. I will need some way to test the new API, ideally without having any VRAM->CPU copying. The simplest plan is to start with a basic wayland window demo and attach the `WlBuffer` obtained from the API to a `WlSurface` and see what the result looks like. 
## Testing the Draft API
I started with the simple window client [example](https://github.com/Smithay/wayland-rs/blob/master/wayland-client/examples/simple_window.rs) from the wayland-rs project. This sample program simply sets up a window and attaches a shared memory backed `WlBuffer` containing a simple image to a `WlSurface`.

All I had to do was to swap out the shm `WlBuffer` with the `WlBuffer` obtained from the new API at `capture_output_frame_dmabuf`. I was surprised to find it working out of the box:
![[Pasted image 20240619212710.png]]
The window on the right mirrors whatever is on the screen via the new API.

