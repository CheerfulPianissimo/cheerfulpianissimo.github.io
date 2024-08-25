---
tags:
  - gsoc
---

# Google Summer Of Code 2024 With Waycrate

I'm pretty excited to be participating in [Google's 2024 Summer of Code ](https://en.wikipedia.org/wiki/Google_Summer_of_Code)program with [Waycrate](https://waycrate.github.io/index.html). I have been eyeing GSoC for a few years and I'm happy that after a few false starts I was able to find a project I was interested in and an organization I could fit into.

# What's your project about?
To put it in a nutshell: It's basically about adding hardware accelerated [Wayland ](https://en.wikipedia.org/wiki/Wayland_(protocol))screen capture to a Rust library called [libwayshot](https://github.com/waycrate/wayshot). 

> [!question]- Never heard of Wayland, what's it?
> Technically, It's a set of protocols that determine how programs that want to draw things on the screen communicate with each other on Linux (and BSD-based) systems. More colloquially, think of it as an ecosystem of building blocks for the Linux desktop. Chances are good you're already using Wayland if you're on a recent Linux distro. 

In a bit more detail: 

Wayshot is a screen capture *client* for Wayland compositors (which in this instance act as servers) that implement the [wlr-screencopy](https://wayland.app/protocols/wlr-screencopy-unstable-v1)  protocol (which at this point is mostly compositors based on the [wlroots](https://github.com/swaywm/wlroots) library like [sway](https://en.wikipedia.org/wiki/Sway_(window_manager)) , river, etc).
> [!question]- What is a Wayland Compositor? 
> It's basically a program that manages how all *other* programs on your system are displayed. All your programs draw stuff and pass it to your compositor which ultimately draws them onto your monitor after adding stuff like window decorations. It's kinda a big part of the Linux desktop user experience

Right now, wayshot uses the non-hardware accelerated screen capture provisions of this protocol which has its drawbacks in terms of performance, my project will be to modify libwayshot so that it uses the part of the protocol that *does* do on-GPU screen copy. This is done via  DMA-BUFs (Direct memory access buffers) which is a part of the Linux Kernel's [Direct Rendering Manager](https://en.wikipedia.org/wiki/Direct_Rendering_Manager) subsystem. 

It's all a bit steeped in domain specific knowledge which is distributed all over the internet. I have an unorganized and non-exhaustive pile of links I used to get a better idea of the problem which may be of interest to some:  [[GsoC Research Links]]

# Devlog
I kept a devlog to document my GSoC 2024 experience. I go into some details  on my experience implementing this project in these entries:
- [[GSoC Devlog (May)]]
- [[GSOC Devlog (June)]]
- [[GSoC Devlog (July)]]
- [[GSoC Devlog (August)]]
## What I Did
The code may be found in this pull request: https://github.com/waycrate/wayshot/pull/122
The tracking issue is at https://github.com/waycrate/wayshot/issues/119
A changelog for this work may be found at [[GSoC Changelog]]
# Post-Project Report
The project timeline can be roughly organized into these parts:
- Creating an initial Minimal Viable Prototype: I created an MVP that used the wlr-screencopy DMA-BUF pipeline fitted into the existing API in order to figure out the necessary crates and types. Detailed in [[GSoC Devlog (May)]].
- Creating the lower level DMA-BUF public API: I built out the pipeline to interact with the compositor using the wlr-screencopy protocol with DMA-BUFs and then created two public facing API functions that returned the results to the user. In order to test this work, I also built a example for this API in the form of `waymirror`. More details in [[GSOC Devlog (June)]].
- Creating an MVP for the higher level EGL and OpenGL pipeline for libwayshot: I studied EGL and OpenGL and used the knowledge along with my mentors assistance to get a working demo that directly rendered screen contents as an texture with zero GPU downloads. I think this was the hardest part of the project for me. A particularly persistent bug with the EGL took me a week to solve. Detailed in [[GSoC Devlog (July)]].
- Creating the higher level EGL/OpenGL public API: With the  MVP working, the remaining work was in refactoring the code from the MVP into the library. Most of the work was in delineating code between the library/the application and figuring out the shape of the API. I added a new wrapper type to ensure proper cleanup of the EGLImage type. This type was simply a raw C pointer and I wanted the type system to do the work of handling it's validity. This is the part of the project where I fought the compiler the most- it dealt with lifetimes, traits and generics all at once. I feel like my understanding of the language has improved a lot with this. More details in [[GSoC Devlog (August)]].
# What's left to do
- **Proper DRM modifier handling**  The present code only works for single planar formats, which works on the hardware I have access to. I'm presently going along with whatever format GBM chooses as it's default but to be universally functional we may need to query the available formats via [linux_dmabuf_feedback](https://wayland.app/protocols/linux-dmabuf-v1#zwp_linux_dmabuf_feedback_v1) and pass that list to GBM. 
	- Also associated with this functionality is automatic GPU selection. Presently, the constructor demands a path to the DRI device from the user for maximum configurability. It should be possible to figure out the best device to choose for GBM buffer object creation even without this. 
- **Streaming support in libwayshot** This wasn't entirely in scope for this project but it's something that can now be pursued with DMA-BUF support available. libwayshot needs some refactoring to handle the streaming use case efficiently. I did modify the final MVP to continuously mirror screen content but the performance was sub-par due to the fact that the overall API is structured to take frame-to-frame screenshots rather than continuous streams. A lot of reduntant protocol setup and buffer creation happens between the frames.   
	- As a waypoint for this feature, some kind of benchmarking may also be advisable in order to track performance improvements and regressions.

# Latest Project Proposal
The updated project proposal doc in markdown format will be made available here:
- [[Project Motivation]]
- [[Project Design]]
I feel like it's more ergonomic and easier to handle to have it as an easily editable markdown file than as richtext.
# The Original Proposal

I have redacted the first and last pages but this is otherwise identical to the proposal that got accepted:
![[Waycrate_GSOC_2024.pdf]]
