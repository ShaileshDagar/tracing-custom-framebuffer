# Custom Framebuffer Trace
We started with a simple program as detailed in this [article](https://embear.ch/blog/drm-framebuffer).
Get the program from its [github](https://github.com/embear-engineering/drm-framebuffer).

The above program from here onwards will be referred to as the ‘custom framebuffer’ program.

It’s a pretty straight-forward program where it just creates a frame buffer and fills it with random values from /dev/urandom.

A little tweak that I did to this program is that instead of piping the data from /dev/urandom file into our program’s stdin from the command line, I directly opened the /dev/urandom file in the program (opened it where the program was reading the data from stdin, and instead of supplying the stdin file descriptor I supplied the file descriptor of /dev/urandom file in the read function, also don’t forget to close this file). This is necessary if we want to run a utility like trace-cmd (command-line utility for ftrace) or strace on our custom framebuffer program.

Note: Run this program on a tty, because if something like an X server is running, it will be the master of the drm subsystem, and our custom framebuffer program will not be able to become the master (the program will fail).

## Static analysis of the custom framebuffer program code:

After preparing the framebuffer, and acquiring the details of resources like connectors, encoders and crtc, I **believe** the program sends the configuration and the pixel data through drmModeSetCrtc function. I consider this as our entry point to tracing the function calls.

Note: Check the kernel version on which we are doing the tracing. Preferably get it from Vicharak github repo.

The following tracing is done on the **Vaaman Linux Kernel version 4.4.190**. The successive function entries in this table are function calls that are nested within the previous function call entries.


+------------+---------------------+---------+------------------------+
| **called   | **function**        | **      | **remarks**            |
| from**     |                     | defined |                        |
|            |                     | in**    |                        |
+============+=====================+=========+========================+
| user       | drmModeSetCrtc()    | libdrm  |                        |
| program    |                     |         |                        |
+------------+---------------------+---------+------------------------+
| libdrm     | DRM_IOCTL           | libdrm  |                        |
+------------+---------------------+---------+------------------------+
| libdrm     | drmIoctl            | libdrm  |                        |
+------------+---------------------+---------+------------------------+
| libdrm     | ioctl               | kernel  | This is a system call  |
|            |                     |         | and is mapped in a     |
|            |                     |         | very verbose way by a  |
|            |                     |         | macro in ioctl.c file. |
|            |                     |         | Search for             |
|            |                     |         | "                      |
|            |                     |         | SYSCALL_DEFINE3(ioctl" |
|            |                     |         | within the ioctl.c     |
|            |                     |         | file.                  |
+------------+---------------------+---------+------------------------+
| kernel     | do_vfs_ioctl        | kernel  |                        |
| (from here |                     | (from   |                        |
| onwards)   |                     | here    |                        |
|            |                     | o       |                        |
|            |                     | nwards) |                        |
+------------+---------------------+---------+------------------------+
|            | vfs_ioctl           |         |                        |
+------------+---------------------+---------+------------------------+
|            | drm_ioctl           |         | registered as a        |
|            |                     |         | callback under         |
|            |                     |         | **unlocked_ioctl** in  |
|            |                     |         | the rockchip_drm_drv.c |
|            |                     |         | file.                  |
+------------+---------------------+---------+------------------------+
|            | drm_mode_setcrtc    |         | this function is       |
|            |                     |         | registered for the     |
|            |                     |         | ioctl command:         |
|            |                     |         | DRM_IOCTL_MODE_SETCRTC |
+------------+---------------------+---------+------------------------+
|            | drm_mode_           |         |                        |
|            | set_config_internal |         |                        |
+------------+---------------------+---------+------------------------+
|            | drm_atomi           |         | registered as a        |
|            | c_helper_set_config |         | callback under         |
|            |                     |         | **set_config** in      |
|            |                     |         | rockchip_drm_vop.c     |
+------------+---------------------+---------+------------------------+
|            | drm_atomic commit   |         |                        |
+------------+---------------------+---------+------------------------+
|            | rockchi             |         |                        |
|            | p_drm_atomic_commit |         |                        |
+------------+---------------------+---------+------------------------+
|            | rockchip_ato        |         |                        |
|            | mic_commit_complete |         |                        |
+------------+---------------------+---------+------------------------+
|            | drm_atomic_h        |         | read the comment above |
|            | elper_commit_planes |         | this function's code.  |
+------------+---------------------+---------+------------------------+
|            | vo                  |         | registered as a        |
|            | p_crtc_atomic_flush |         | callback under         |
|            |                     |         | **atomic_flush** in    |
|            |                     |         | the file               |
|            |                     |         | rockchip_drm_vop.c     |
+------------+---------------------+---------+------------------------+
|            | rockchip_dr         |         | a guess that this      |
|            | m_dma_attach_device |         | might be signaling dma |
|            |                     |         | controller to transfer |
|            |                     |         | the new state over to  |
|            |                     |         | the eDP.               |
|            |                     |         |                        |
|            |                     |         | **We might be able to  |
|            |                     |         | use this as our entry  |
|            |                     |         | point to sending data  |
|            |                     |         | directly to eDP.**     |
+------------+---------------------+---------+------------------------+


### ftrace
Navigate to the directory of the custom framebuffer program and use the following commands to obtain the ftrace of the program.
```trace-cmd record -p function_graph -F ./drm-framebuffer -d /dev/dri/card0 -c eDP-1```
```trace-cmd report -O tailprint > hello.ftrace.report```

### Note: 
-- You should cross-reference this with the output of ftrace.
-- ftrace outputs only function calls that are within the kernel.
-- Also, inline functions might not show up in the output of ftrace.
-- Sometimes, a function name is missing in the ftrace output, but the function calls inside it are present in the output, so it’s a clue that execution flow might be through that function.

While tracing the framebuffer, it is very obscure and embedded in abstractions, passed probably through references like framebuffer handle or id, which means that they must be resolved only when someone tries to access that memory region(or so I think).

We should probably use some tool like eBPF to track the access to the memory region holding the pixel data and other configurations.
[This video](https://www.youtube.com/watch?v=lrSExTfS-iQ) a good place to start and get a feel of the tool. 

### Tips: 
-- When functions are registered as callbacks, you can use the output of ftrace to find the function name which is registered as the callback.
-- Useful tool for kernel code navigation: Bootlin Linux Kernel Code Navigation(Mind the kernel version)

