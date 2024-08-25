---
tags: []
---
# GSoC Changelog   
   
## Changelog for add-screencopy-dmabuf-backend branch   
   
- Added new API function `capture_output_frame_dmabuf` in WayshotConnection which provides a full on-GPU pipeline to transfer screenshot data from the wayland compositor, returns a `WlBuffer` and gbm-rs `BufferObjects`.   
	- See `examples/\waymirror.rs` for a demo   
- Added new API functions `capture_output_frame_eglimage_on_display`, `capture_output_frame_eglimage` and `bind_output_frame_to_gl_texture`   
	- See `examples/waymirror-egl` for a demo project using the GL API pipeline   
- New build dependency - libegl is required for statically linking in the EGL libraries. libwayland is also now a required dependency for the EGL/OpenGL pipeline.    
   
## Changelog for freeze-feat-andreas   
   
- In-built clipboard support using wl-clipboard-rs, use the --clipboard flag to copy to clipboard