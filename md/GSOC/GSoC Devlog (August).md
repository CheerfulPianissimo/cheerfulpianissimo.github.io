---
tags: []
---
   
# Weeks 1-2   
With the final weeks of GSoC approaching, it was time to consolidate all the existing work in the demos into the library as a public API.   
   
Most of the work in this time was around figuring out how to do the libwayshot API and refactoring the code for it from the waymirror-egl demo into the main libwayshot library.   
After some discussions with my mentor, I decided to use the following three level API structure for the EGLImage related APIs:   
   
- At the lowest level, `capture_output_frame_eglimage_on_display` : this obtains a GBM BufferObject from the `capture_output_frame_dmabuf` implemented in the last month and turns it into a EGLImage on a given egl_display using a given egl API instance.   
	- A note here: I created a new wrapper type `EGLImageGuard` in order to safely handle Image type structs from the `khronos_egl` library. The `Image` type is simply a thin wrapper around a C FFI pointer with no cleanup when it is dropped, leading to possible resource leaks. The `EGLImageGuard` struct has a `Drop` impl that ensures that eglDestroyImage is called whenever the type is dropped.   
- At the level above, I implemented `capture_output_frame_eglimage` which would do the same as the first function but without requiring an egl_display parameter. An egl_display is created using the wayland display configured in libwayshot.   
- At the highest level is `bind_output_frame_to_gl_texture`. With this function, the user doesn't have to care about EGL, they can instead use libwayshot to directly bind the screen contents into a OpenGL texture in a single function call. This is the culmination of all the parts of this project, using the full pipeline of WlBuffer/GBMBO->EGLImage->GLES Texture. This is the function that I've now refactored the waymirror-egl demo to use. There are a few caveats with it's usage I've tried to explain in it's doc comments namely that OpenGL having a state machine centric model doesn't play well with any kind of return oriented API. So this function is more of an helper that finds and calls the requisite function to bind the EGLImage as a texture on the GL context. It's up to the caller to set up everything till that point.   
   
I also spent some time documenting and testing the use of Regions to capture parts of the screen using these functions in over these two weeks.