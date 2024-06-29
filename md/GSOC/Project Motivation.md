---
tags: []
---
# Project Motivation   
   
   
   
libwayshot is a Rust crate that implements screen capture functionality for wlroots based compositors. It’s also associated with wayshot - a cli app based on libwayshot that provides an executable for capturing wlroots outputs or parts of them.   
   
Presently libwayshot uses shared memory (in the form of wl_shm buffers) to transfer screen capture data from the compositor via the wlr-screencopy protocol.   
   
However, most wlroots compositors are hardware accelerated and will make use of the VRAM to perform all of their rendering. Transferring graphics data via memfd_create necessitates that the compositor move the required data from the VRAM via the CPU, adding a degree of overhead.   
   
Now, some applications may be just fine with this because they already have no option but to download data from the VRAM in order to encode and store it on the filesystem or send it over the network or other such CPU-centric processes. This is the case for one-off screencapture applications like wayshot.   
   
However, there is a class of applications which need to perform further GPU accelerated operations on this graphics data. Consider:   
   
-   Screen recorders that use hardware acceleration for video encoding.   
-   Desktop portals like xdg-portal-luminous that route screen content to other apps.   
-   Screen mirroring apps that apply various transformations on screen content before returning it to the compositor to display.   
   
In all these cases the screen content needs to be uploaded back into the GPU. In these cases using shared memory simply adds unnecessary downloads and uploads with respect to the GPU. wlroots provides the capability to avoid this inefficiency by providing methods to keep the captured content on VRAM without any copies.   
   
My motivation here is to propose a way for libwayshot to support these methods and widen its applicability to a larger class of projects that involve screen capture.   
