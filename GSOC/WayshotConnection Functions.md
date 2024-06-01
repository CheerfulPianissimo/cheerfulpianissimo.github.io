Below is a dependency graph that outlines which functions in the `WayshotConnection` struct call which other functions. 

```scss
WayshotConnection
    ├── new() -> Result<Self> [Public API]
    │   └── from_connection(conn: Connection) -> Result<Self>
    │       └── refresh_outputs(&mut self) -> Result<()>
    ├── from_connection(conn: Connection) -> Result<Self> [Public API]
    │   └── refresh_outputs(&mut self) -> Result<()>
    ├── get_all_outputs(&self) -> &[OutputInfo] [Public API]
    ├── refresh_outputs(&mut self) -> Result<()>
    ├── capture_output_frame_shm_fd<T: AsFd>(...) -> Result<(FrameFormat, FrameGuard)>
    │   ├── capture_output_frame_get_state(...)
    │   └── capture_output_frame_inner<T: AsFd>(...)
    ├── capture_output_frame_get_state(...) -> Result<(CaptureFrameState, EventQueue<CaptureFrameState>, ZwlrScreencopyFrameV1, FrameFormat)>
    ├── capture_output_frame_inner<T: AsFd>(...)
    ├── capture_output_frame_shm_from_file(...) -> Result<(FrameFormat, FrameGuard)>
    │   ├── capture_output_frame_get_state(...)
    │   └── capture_output_frame_inner<T: AsFd>(...)
    ├── capture_frame_copy(...) -> Result<(FrameCopy, FrameGuard)>
    │   └── capture_output_frame_shm_from_file(...)
    ├── capture_frame_copies(...) -> Result<Vec<(FrameCopy, FrameGuard, OutputInfo)>>
    │   └── capture_frame_copy(...)
    ├── overlay_frames_and_select_region(...) -> Result<LogicalRegion>
    ├── screenshot_region_capturer(...) -> Result<DynamicImage>
    │   ├── capture_frame_copies(...)
    │   │   └── capture_frame_copy(...)
    │   └── overlay_frames_and_select_region(...)
    ├── screenshot(...) -> Result<DynamicImage> [Public API]
    │   └── screenshot_region_capturer(...)
    │       ├── capture_frame_copies(...)
    │       │   └── capture_frame_copy(...)
    │       └── overlay_frames_and_select_region(...)
    ├── screenshot_freeze(...) -> Result<DynamicImage> [Public API]
    │   └── screenshot_region_capturer(...)
    │       ├── capture_frame_copies(...)
    │       │   └── capture_frame_copy(...)
    │       └── overlay_frames_and_select_region(...)
    ├── screenshot_single_output(...) -> Result<DynamicImage> [Public API]
    │   └── capture_frame_copy(...)
    ├── screenshot_outputs(...) -> Result<DynamicImage> [Public API]
    │   └── screenshot_region_capturer(...)
    │       ├── capture_frame_copies(...)
    │       │   └── capture_frame_copy(...)
    │       └── overlay_frames_and_select_region(...)
    ├── screenshot_all(...) -> Result<DynamicImage> [Public API]
        └── screenshot_outputs(...)
            └── screenshot_region_capturer(...)
                ├── capture_frame_copies(...)
                │   └── capture_frame_copy(...)
                └── overlay_frames_and_select_region(...)
```

### Explanation

- **`WayshotConnection::new()`**
  - Calls `WayshotConnection::from_connection()`, which in turn calls `WayshotConnection::refresh_outputs()`.

- **`WayshotConnection::capture_output_frame_shm_fd()`**
  - Calls `WayshotConnection::capture_output_frame_get_state()` to prepare the state for capturing the frame.
  - Calls `WayshotConnection::capture_output_frame_inner()` to complete the frame capture process.

- **`WayshotConnection::capture_output_frame_shm_from_file()`**
  - Calls `WayshotConnection::capture_output_frame_get_state()` to prepare the state for capturing the frame.
  - Calls `WayshotConnection::capture_output_frame_inner()` to complete the frame capture process.

- **`WayshotConnection::capture_frame_copy()`**
  - Calls `WayshotConnection::capture_output_frame_shm_from_file()` to capture the frame and get the image data.

- **`WayshotConnection::capture_frame_copies()`**
  - Calls `WayshotConnection::capture_frame_copy()` in a thread scope to capture frames from multiple outputs simultaneously.

- **`WayshotConnection::screenshot_region_capturer()`**
  - Calls `WayshotConnection::capture_frame_copies()` to capture frames for the specified region or outputs.
  - Calls `WayshotConnection::overlay_frames_and_select_region()` if the region capture mode is `Freeze`.

- **`WayshotConnection::screenshot()`**
  - Calls `WayshotConnection::screenshot_region_capturer()` to take a screenshot of a specified logical region.

- **`WayshotConnection::screenshot_freeze()`**
  - Calls `WayshotConnection::screenshot_region_capturer()` to freeze the screen, allow region selection, and capture the selected region.

- **`WayshotConnection::screenshot_single_output()`**
  - Calls `WayshotConnection::capture_frame_copy()` to capture a frame from a single specified output.

- **`WayshotConnection::screenshot_outputs()`**
  - Calls `WayshotConnection::screenshot_region_capturer()` to take screenshots of specified outputs.

- **`WayshotConnection::screenshot_all()`**
  - Calls `WayshotConnection::screenshot_outputs()` to take screenshots of all accessible outputs.
## Function Descriptions

A concise explanation of each function in the `WayshotConnection` struct:
1 . **new() -> Result<Self>**: 
   - Establishes a new connection to the Wayland compositor and initializes the `WayshotConnection` object.
2. **from_connection(conn: Connection) -> Result<Self>**: 
   - Initializes the `WayshotConnection` object from an existing Wayland connection.

3. **get_all_outputs(&self) -> &[OutputInfo]**: 
   - Returns a slice of all detected Wayland outputs.

4. **refresh_outputs(&mut self) -> Result<()>**: 
   - Refreshes and updates the list of available Wayland outputs.

5. **capture_output_frame_shm_fd<T: AsFd>(...) -> Result<(FrameFormat, FrameGuard)>**: 
   - Captures a frame from a specified output and writes the data to a file descriptor.

6. **capture_output_frame_get_state(...) -> Result<(CaptureFrameState, EventQueue<CaptureFrameState>, ZwlrScreencopyFrameV1, FrameFormat)>**: 
   - Prepares the state necessary for capturing an output frame, including setting up the Wayland screencopy protocol.

7. **capture_output_frame_inner<T: AsFd>(...) -> Result<FrameGuard>**: 
   - Completes the frame capture process by copying pixel data into the provided file descriptor.

8. **capture_output_frame_shm_from_file(...) -> Result<(FrameFormat, FrameGuard)>**: 
   - Captures a frame from a specified output and writes the data to a given file.

9. **capture_frame_copy(...) -> Result<(FrameCopy, FrameGuard)>**: 
   - Captures a frame from a specified output and returns the pixel data and its format.

10. **capture_frame_copies(...) -> Result<Vec<(FrameCopy, FrameGuard, OutputInfo)>>**: 
    - Captures frames from multiple outputs simultaneously and returns the results.

11. **overlay_frames_and_select_region(...) -> Result<LogicalRegion>**: 
    - Overlays captured frames on the screen, allows the user to select a region, and returns the selected region.

12. **screenshot_region_capturer(...) -> Result<DynamicImage>**: 
    - Takes a screenshot of a specified region, possibly overlaying cursor data, and returns the captured image.

13. **screenshot(...) -> Result<DynamicImage>**: 
    - Takes a screenshot of a specified logical region and returns the image.

14. **screenshot_freeze(...) -> Result<DynamicImage>**: 
    - Freezes the screen, overlays the screenshot, runs a callback for region selection, and returns the captured image of the selected region.

15. **screenshot_single_output(...) -> Result<DynamicImage>**: 
    - Takes a screenshot of a single specified output and returns the image.

16. **screenshot_outputs(...) -> Result<DynamicImage>**: 
    - Takes screenshots of the specified outputs and returns the combined image.

17. **screenshot_all(...) -> Result<DynamicImage>**: 
    - Takes screenshots of all accessible outputs and returns the combined image.
