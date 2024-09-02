---
title: 'Linux From Scratch'
date: 2024-08-28T18:02:21+02:00
draft: false
image: /images/lfs.jpg
---


<cite> "Linux From Scratch (LFS) is a project that provides you with step-by-step instructions for building your own customized Linux system entirely from source." [^1]</cite>

[^1]: [https://www.linuxfromscratch.org/lfs/](https://www.linuxfromscratch.org/lfs/)

## Understanding a linux distribution

When you install a common Linux distribution on your laptop, the process is relatively straightforward: you download the ISO, boot your laptop from it, and most likely run an installation script that sets everything up for you. The ISO contains a fully bootable file system image that provides a live OS, allowing you to run the installer and set up the distribution on your laptop's hard drive.

A Linux distribution includes the Linux kernel, along with all the tools needed for the user space: GNU tools, libraries, user utilities, a package manager, and so on. Each distribution differs in its choice of these components and how they are configured to work together. Typically, these pieces come precompiled and ready to use.

## What makes LFS different

Linux From Scratch, on the other hand, doesn’t provide a ready-to-use system. Instead, it gives you detailed instructions for building everything from scratch, using only the source code of each component. The project guides you through the process of assembling and configuring these pieces to create a fully functional system.

Since the beginning of my journey with GNU/Linux, I've always wanted to deeply understand how Linux systems work. I see implementing LFS as a learning opportunity to satisfy my curiosity and gain a deeper insight into how all the components of a Linux system fit together. I enjoy breaking down complex problems into their fundamental parts, and LFS offers a perfect chance to apply this _divide and conquer_ approach.

## The LFS process

### Setting up the partition

The process begins by creating a partition where you will compile and install your system. Then, you download the source code for all the necessary packages, along with any required patches.

At this point, you have only a basic directory structure in your new partition that will eventually become your target system. But before you start compiling, there’s an important consideration: you can’t simply use your host system to build packages for the target system, because this could result in dependencies on the host that shouldn’t be there. To avoid this, LFS uses cross-compilation to build the target system.

### Building the pieces of the puzzle

For clarity, I'll refer to your original machine as the host and the system you’re building as the target. Right now, the target is simply the empty partition you just created. It’s helpful to think of the host and target as two separate machines.

From now on, the process of building unfold in three main steps:

##### 1. Building a cross-compiler

The first step involves using your host machine’s compiler to create a cross-compiler. Since a cross-compiler is designed to produce software for a different system (the target), it can be used to build a minimal set of utilities specifically for the target environment. During this step, you configure the build with `--target=$(uname -m)-lfs-linux-gnu` to simulate cross-compilation by modifying the vendor triplet, and `--with-sysroot=<path-to-lfs-partition>` to direct the compiler to the necessary host files, ensuring that the packages built later don’t link to libraries on the host.

##### 2. Cross-compiling the initial system

With the cross-compiler in hand, the next step is to use it to build and install a minimal set of utilities on the target system. This is where you begin populating the target environment. As libraries are added, you need to recompile the cross-compiler. This step is crucial because the initial cross-compiler build had some features disabled due to missing dependencies. Recompiling the cross-compiler solves these circular dependency issues.

##### 3. Final build in chroot

The final step involves chrooting into the target system. In this environment, you rebuild the system components using the initial system that you’ve just built. This ensures greater stability of the packages and allows you to run their test suites.

Before entering chroot, it's necessary to set up a little bit more the target as you need to create the Virtual Kernel File Systems needed by some userspace applications.

### System configuration

The final chapter of LFS focuses on setting up the boot process and completing the system configuration. This involves configuring essential system files, such as network settings, system locale, and the init system. You'll also explore how devices and modules are managed. A critical step is building and installing the Linux kernel, which requires configuring it based on your hardware. Finally, you'll set up the bootloader, ensuring it can load your newly created operating system, bringing the entire system to life.

A particularly interesting aspect of this chapter is the opportunity to see how the init system and GRUB bootloader can be configured to suit your needs.

Assuming that the bootloader is correctly set up, you should now be able to restart your machine and boot into the newly installed operating system. This system will be very basic, serving as a foundation for further customization according to your needs. However, it lacks a package manager, so any additional software will need to be compiled from source as well. This is where the Beyond Linux From Scratch (BLFS) book comes in, offering comprehensive guidance on how to extend your LFS system by adding essential tools, desktop environments, networking utilities, and more. BLFS enables you to transform your minimal LFS installation into a fully functional Linux system tailored to your specific requirements.

## Was the hassle worth it?

Completing the Linux From Scratch process required a significant amount of time compared to using a typical installation script. The repetitive nature of building all the packages became boring fairly quickly. LFS does offer ALFS, a tool to automate the entire process. However, using it can take away from the main goal of understanding the system.

If you have the patience to follow each step, taking the time to read and understand why each one is necessary rather than just copying and pasting commands (which, if that’s your approach, ALFS might be a better fit), you gain a better comprehension of the system. You start to see how each component fits into the larger puzzle and how they work together.

It’s similar to building a house brick by brick. By the end, your perception of the house has changed: it's no longer just a single structure but a collection of parts, each with its own purpose. Similarly, after going through LFS, Linux is no longer just a system that "somehow works." Now, you can break it down and understand the function and significance of each piece that makes up the system.

Linux From Scratch is not just about building an operating system: it’s about gaining the skills and knowledge that can make you more confident and capable in navigating and customizing Linux environments.
