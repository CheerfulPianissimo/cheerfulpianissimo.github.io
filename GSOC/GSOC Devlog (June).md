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
	- Not sure if WayshotConnection is the right place for this, presently most of this state for shm screencopy is stored in the intermediate `capture_*` functions.
- In general there's stuff that needs to be done once like init, getting output information or creating a screencopy manager with the output and region selected and then there is the stuff that needs to be done every frame: getting the buffer parameters, properly initializing the buffers, and actually copying the frame.
	- Presently wayshot is not geared to really separate this: for instance, it creates a new screencopy manager for every frame initialized  and it's clearly designed to be a single use screenshot library rather than a continuous screen-capture system. 
	- I intend to build out the dmabuf functionality in the current framework, will see if it can be refactored out later to more clearly separate the frame capture init from the actual frame capture loop.
## Questions
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
	