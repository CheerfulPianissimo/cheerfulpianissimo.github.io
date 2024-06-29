## Changelog for add-screencopy-dmabuf-backend branch
- Added new API function `capture_output_frame_dmabuf` in WayshotConnection which provides a full on-GPU pipeline to transfer screenshot data from the wayland compositor, returns a `WlBuffer` and gbm-rs `BufferObjects`.
	- See examples\waymirror.rs for a demo
## Changelog for freeze-feat-andreas
- In-built clipboard support using wl-clipboard-rs, use the --clipboard flag to copy to clipboard