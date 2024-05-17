# Google Summer Of Code 2024

I'm pretty excited to be participating in [Google's 2024 Summer of Code ](https://en.wikipedia.org/wiki/Google_Summer_of_Code)program with [Waycrate](https://waycrate.github.io/index.html). I have been eyeing GSoC for a few years and I'm happy that after a few false starts I was able to find a project I was interested in and an organization I could fit into.

# What's your project about?
To put it in a nutshell: It's basically about adding hardware accelerated [Wayland ](https://en.wikipedia.org/wiki/Wayland_(protocol))screen capture to a Rust library called [libwayshot](https://github.com/waycrate/wayshot). 

> [!question]- Never heard of Wayland, what's it?
> Technically, It's a set of protocols that determine how programs that want to draw things on the screen communicate with each other on Linux (and BSD-based) systems. More colloquially, think of it as an ecosystem of building blocks for the Linux desktop. Chances are good you're already using Wayland if you're on a recent Linux distro. 

In a tad bit more detail: 

Wayshot is a screen capture *client* for Wayland compositors (which in this instance act as servers) that implement the [wlr-screencopy](https://wayland.app/protocols/wlr-screencopy-unstable-v1)  protocol (which at this point is mostly compositors based on the [wlroots](https://github.com/swaywm/wlroots) library including the amazing [sway](https://en.wikipedia.org/wiki/Sway_(window_manager)) compositor which is used by yours truly).
> [!question]- What is a Wayland Compositor? 
> It's basically a program that manages how all *other* programs on your system are displayed. All your programs draw stuff and pass it to your compositor which ultimately draws them onto your monitor after adding stuff like window decorations. It's kinda a big part of the Linux desktop user experience

Right now, wayshot uses the non-hardware accelerated screen capture provisions of this protocol which has its drawbacks in terms of performance, my project will be to modify libwayshot so that it uses the part of the protocol that *does* do on-GPU screen copy. This is done via  DMA-BUFs (Direct memory access buffers) which is a part of the Linux Kernel's [Direct Rendering Manager](https://en.wikipedia.org/wiki/Direct_Rendering_Manager) subsystem. 

It's all a bit steeped in domain specific knowledge which is distributed all over the internet. I have a unorganized and non-exhaustive pile of links I used to get a better idea of the problem which may be of interest to some:  [[GsoC Research Links]]

# My Proposal

I have redacted the first and last pages but this is otherwise identical to the proposal that got accepted:
![[Waycrate_GSOC_2024.pdf]]
