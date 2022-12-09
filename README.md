# nvidia-vaapi-driver

This is an VA-API implementation that uses NVDEC as a backend. This implementation is specifically designed to be used by Firefox for accelerated decode of web content, and may not operate correctly in other applications.

This library requires that the `nvidia_drm` kernel module is [configured with the parameter](https://wiki.archlinux.org/title/Kernel_parameters) `nvidia-drm.modeset=1`

# Table of contents

- [nvidia-vaapi-driver](#nvidia-vaapi-driver)
- [Table of contents](#table-of-contents)
- [Codec Support](#codec-support)
- [Installation](#installation)
  - [Package manager](#package-manager)
  - [Building](#building)
  - [Removal](#removal)
- [Configuration](#configuration)
  - [Environment Variables](#environment-variables)
  - [Firefox](#firefox)
  - [MPV](#mpv)
  - [Direct Backend](#direct-backend)
- [Testing](#testing)

# Codec Support

Only decoding video is currently supported.

| Codec | Supported | Comments |
|---|---|---|
|AV1|:heavy_check_mark:|Firefox 98+ is required.|
|H.264|:heavy_check_mark:||
|HEVC|:heavy_check_mark:||
|VP8|:heavy_check_mark:||
|VP9|:heavy_check_mark:|Requires being compiled with `gstreamer-codecparsers-1.0`|
|MPEG-2|:heavy_check_mark:||
|VC-1|:heavy_check_mark:||
|MPEG-4|:x:|VA-API does not supply enough of the original bitstream to allow NVDEC to decode it.|
|JPEG|:x:|This is unlikely to ever work, the two APIs are too different.|

YUV444 video support currently unavailable. It is a work in progress [here](/../../pull/134).

To view which codecs your card is capable of decoding you can use the `vainfo` command with this driver installed, or visit the NVIDIA website [here](https://developer.nvidia.com/video-encode-and-decode-gpu-support-matrix-new#geforce).

# Installation

To install and use `nvidia-vaapi-driver`, follow the steps in installation and configuration. It is recommended to follow testing as well to verify hardware acceleration is working as intended.

**Requirements**

* NVIDIA driver series 470 or 500+

## Package manager

| Package manager | Package name                                                                          | Notes          |
|-----------------|---------------------------------------------------------------------------------------|----------------|
| pacman (AUR)    | [libva-nvidia-driver](https://aur.archlinux.org/packages/libva-nvidia-driver)         | Latest release |
| pacman (AUR)    | [nvidia-vaapi-driver-git](https://aur.archlinux.org/packages/nvidia-vaapi-driver-git) | @master        |

Feel free to add your distributions package in an issue/PR.

## Building

You'll need `meson`, the `gstreamer-plugins-bad` library, and [`nv-codec-headers`](https://git.videolan.org/?p=ffmpeg/nv-codec-headers.git) installed.

| Package manager | Packages                                        |
|-----------------|-------------------------------------------------|
| pacman          | meson gst-plugins-bad ffnvcodec-headers         |
| apt             | meson gstreamer1.0-plugins-bad nv-codec-headers |

Then run the following commands:

```sh
meson setup build
meson install -C build
```

## Removal

By default the driver installs itself as `/usr/lib64/dri/nvidia_drv_video.so` (this might be `/usr/lib/x86_64-linux-gnu/dri/nvidia_drv_video.so` on some distros). To uninstall the driver, simply remove this file. In addition, this file is usually symlinked to `/usr/lib64/dri/vdpau_drv_video.so` (or `/usr/lib/x86_64-linux-gnu/dri/vdpau_drv_video.so`) if the VDPAU to VA-API driver is installed, so this symlink will need to be restored for that driver to work normally again.

# Configuration

**IMPORTANT**: The [direct backend](#direct-backend) is currently required on NVIDIA driver series 525 due to a regression (see [issue #126](/../../issues/126)).

## Environment Variables

Environment variables used to control the behavior of this library.

| Variable | Purpose |
|---|---|
| `NVD_LOG` | Used to control logging. `1` to log to stdout, anything else to append to the given file. |
| `NVD_MAX_INSTANCES` | Controls the maximum concurrent instances of the driver will be allowed per-process. This option is only really useful for older GPUs with not much VRAM, especially with Firefox on video heavy websites. |
| `NVD_BACKEND` | Controls which backend this library uses. Either `egl` (default), or `direct`. See [direct backend](#direct-backend) for more details. |

## Firefox

To use the driver with firefox you will need at least Firefox 96, the following config options need to be set in the `about:config` page:

| Option | Value | Reason |
|---|---|---|
| media.ffmpeg.vaapi.enabled | true | Required, enables the use of VA-API. |
| media.rdd-ffmpeg.enabled | true | Required, default on FF97. Forces ffmpeg usage into the RDD process, rather than the content process. |
| media.av1.enabled | false | Optional, disables AV1. If your GPU doesn't support AV1, this will prevent sites using it and falling back to software decoding. |
| gfx.x11-egl.force-enabled | true | Required, this driver requires that Firefox use the EGL backend. It may be enabled by default. It is recommended to test it with the `MOZ_X11_EGL=1` environment variable before enabling it in the Firefox configuration. |
| widget.dmabuf.force-enabled | true | Required on NVIDIA 470 series drivers, not required on 495+. Note that Firefox isn't coded to allow DMA-BUF support without GBM support, so it may not function completely correctly when it's forced on. |

In addition the following environment variables need to be set. For permanent configuration `/etc/environment` may suffice.

| Variable | Value | Reason |
|---|---|---|
| LIBVA_DRIVER_NAME | nvidia | Forces libva to load the `nvidia` backend, as the current version doesn't know which driver to load for the nvidia-drm driver. |
| MOZ_DISABLE_RDD_SANDBOX | 1 | Disables the sandbox for the RDD process that the decoder runs in. |
| EGL_PLATFORM | wayland | Required on FF98+ when running on Wayland, due to a regression that has been introduced. |
| __EGL_VENDOR_LIBRARY_FILENAMES | /usr/share/glvnd/egl_vendor.d/10_nvidia.json | Required for the 470 driver series only. It overrides the list of drivers the glvnd library can use to prevent Firefox from using the MESA driver by mistake. |

## MPV

Currently this only works with a build of MPV from git master.

There's no real reason to run it with mpv except for testing, as mpv already supports using nvdec directly. The `test.sh` script will run mpv with the file provided and various environment variables set to use the newly built driver

## Direct Backend

The direct backend is a experimental backend that accesses the NVIDIA kernel driver directly, rather than using EGL to share the buffers. This allows us
a greater degree of control over buffer allocation and freeing.

Given this backend accesses the NVIDIA driver directly, via NVIDIA's unstable API, this module is likely to break often with new versions of the kernel driver. If you encounter issues using this backend raise an issue and including logs generated by `NVD_LOG=1`.

**Currently tested and verified generations**

* Pascal (10XX)
* Turing mobile (20XX)

This backend uses headers files from the NVIDIA [open-gpu-kernel-modules](https://github.com/NVIDIA/open-gpu-kernel-modules)
project. The `extract_headers.sh` script, along with the `headers.in` file list which files we need, and will copy them from a checked out version of the NVIDIA project
to the `nvidia-include` directory. This is done to prevent everyone needing to checkout that project.

# Testing

To verify that the driver is being used to decode video, you can use nvidia-settings or nvidia-smi.

- nvidia-settings

  By selecting the relevant GPU on the left of the nvidia-settings window, it will show `Video Engine Utilization` on the right. While playing a video this value should be non-zero.

- nvidia-smi

  Running `nvidia-smi` while decoding a video should show a Firefox process with `C` in the `Type` column. In addition `nvidia-smi pmon` will show the usage of the decode engine per-process, and `nvidia-smi dmon` will show the usage per-GPU.
