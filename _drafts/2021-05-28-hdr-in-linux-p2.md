---
layout: post
title:  "HDR in Linux: Part 2"
date:   2021-05-20 15:05:08 -0400
categories: blog fedora graphics hdr
---

# HDR In the Linux Desktop

In the [previous
post](https://www.jcline.org/blog/fedora/graphics/hdr/2021/05/07/hdr-in-linux-p1.html),
we learned what HDR is: a larger luminance range that requires more bits per
component, new transfer functions to encode that luminance, and potentially
some metadata. We can examine the work required to use it in a "standard" Linux
desktop. By "standard", I of course meant *my* desktop environment, which is
GNOME on Fedora.

To do this, we'll consider a single use-case and examine each portion of the
stack, starting at the top and working our way down. The use-case is watching
an HDR movie in GNOME's video application, Totem. In this scenario, the
application isn't likely to tone-map its content as it has been created with
HDR metadata the display itself can use to tone-map when necessary, but I will
note where this *could* happen.

Let's review the high-level requirements:

- The Wayland display server (compositor) must determine the display
  capabilities (color primaries and white point, luminance, transfer function,
  bits per component, etc), and client applications may also want this
  information. In our case, the video player doesn't need to know it as either
  the Wayland display server or the display itself will handle tone-mapping.

- Client applications need to express the transfer function, bit depth, color
  space, and any HDR metadata for its content (the "content encoding") to the
  compositor. The compositor, in turn, needs to ensure the content from all the
  client applications are blended properly and output in a format the display
  can handle.

You might be familiar with text encoding. Adding HDR support is very similar to
adding support for various Unicode encoding formats when previously everything
assumed content was encoded with ASCII.

## Totem

[Totem](https://gitlab.gnome.org/GNOME/totem) provides the user interface and
playback management features. It relies on GStreamer to deal with the video
itself. It [appears to
use](https://gitlab.gnome.org/GNOME/totem/-/blob/V_3_38_0/src/backend/bacon-video-widget.c#L6155)
the
[playbin](https://gstreamer.freedesktop.org/documentation/playback/playbin.html)
plugin, which GStreamer describes as an "everything-in-one" abstraction for a
audio and video player, which makes sense.

Since Totem is all about wiring up a user interface to the GStreamer playbin
plugin, GStreamer needs to be HDR-aware.

## GStreamer

Based on the GStreamer documentation all the necessary features appear to
already be implemented for HDR video playback.

The [GStreamer video
library](https://gstreamer.freedesktop.org/documentation/video/index.html)
contains an [HDR
section](https://gstreamer.freedesktop.org/documentation/video/video-hdr.html).
The APIs, available since 1.20, include interfaces for working with SMPTE ST
2086 (static metadata) and SMPTE 2094-40 (dynamic metadata). This metadata can
be attached to a
[GstBuffer](https://gstreamer.freedesktop.org/documentation/video/video-hdr.html#gst_buffer_add_video_hdr_meta).

The
[GstVideoInfo](https://gstreamer.freedesktop.org/documentation/video/video-info.html)
structure includes
[colorimetry](https://gstreamer.freedesktop.org/documentation/video/video-color.html#GstVideoColorimetry)
including the color primaries and transfer function. GstVideoInfo also includes
the
[GstVideoFormatInfo](https://gstreamer.freedesktop.org/documentation/video/video-format.html#GstVideoFormatInfo)
structure which describes the pixel format, including bit depth. These contain
all the information required to decode and convert the content. The
[GstVideoFrame](https://gstreamer.freedesktop.org/documentation/video/video-frame.html)
ties the ``GstBuffer`` and ``GstVideoInfo`` together.

So GStreamer has all the content metadata we need. Next, we need to make sure
it gets from the application, Totem, to the Wayland display server. This is
done with [EGL](https://en.wikipedia.org/wiki/EGL_(API)) or [Vulkan Window
System
Integration (WSI)](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html#wsi)
APIs. These APIs are where OpenGL and Vulkan APIs are integrated into the
operating system. This is where buffers are allocated for use within the
graphics libraries and they include ways to provide the content encoding. As
long as GStreamer passes the metadata to the relevant EGL or Vulkan API, it
GStreamer is all set. Mesa includes implementations of EGL and Vulkan WSI, so
we'll examine how those work next.

## Mesa

Clients render their content with Mesa. To allocate a place to perform that
rendering, clients use the EGL or Vulkan WSI interfaces.

### EGL

The full [EGL standard](https://www.khronos.org/egl) is available online. Of
particular interest to us are the interfaces for creating surfaces.
[eglCreatePlatformWindowSurface](https://www.khronos.org/registry/EGL/sdk/docs/man/html/eglCreatePlatformWindowSurface.xhtml)
includes a list of attributes, including the ``EGL_GL_COLORSPACE`` attribute.
As of EGL 1.5, only ``EGL_GL_COLORSPACE_SRGB`` (which defines both color
primaries and a transfer function) and ``EGL_GL_COLORSPACE_LINEAR`` (presumably
the sRGB primaries, linear light) are defined. However there are a number of
extensions to EGL for additional color space attributes including:

* [BT2020 with linear or PQ-encoded
  luminance](https://www.khronos.org/registry/EGL/extensions/EXT/EGL_EXT_gl_colorspace_bt2020_linear.txt)

* [Display-P3 with linear and "sRGB-like" encoded
  luminance](https://www.khronos.org/registry/EGL/extensions/EXT/EGL_EXT_gl_colorspace_display_p3.txt)

Additionally, there is an
[extension](https://www.khronos.org/registry/EGL/extensions/EXT/EGL_EXT_image_gl_colorspace.txt)
to allow the ``EGL_GL_COLORSPACE`` attribute to be applied to
``eglCreateImage``

These extensions make it possible to convey the color primaries and transfer
function to the Wayland display server.

However, this is not currently done because the Wayland protocol has not yet
accepted a way to send the content encoding. The
[proposed](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/merge_requests/14)
protocol has been under discussion (200+ comments) for a year. The details will
be covered in the Wayland section below, but for now what's important to know
is that the protocol needs to define a message for the color space and transfer
function that is sent from the client via Mesa when a surface is created in
[EGL](https://gitlab.freedesktop.org/mesa/mesa/-/blob/21.1/src/egl/drivers/dri2/platform_wayland.c#L303).

Finally, there is an EGL extension for specifying the HDR metadata for a
surface.  By calling
[eglSurfaceAttrib](https://www.khronos.org/registry/EGL/sdk/docs/man/html/eglSurfaceAttrib.xhtml)
and providing the attributes defined in
[EXT_surface_SMPTE2086_metadata](https://www.khronos.org/registry/EGL/extensions/EXT/EGL_EXT_surface_SMPTE2086_metadata.txt)
a client can set the HDR metadata for the content. This is actually already
[partially wired up in
Mesa](https://gitlab.freedesktop.org/mesa/mesa/-/commit/799b3d16d4bb0caa16dc35de66e11eca8517cd02).
All that remains is for the Wayland-specific EGL code to send the HDR metadata
to the Wayland display server. Again, there is no protocol for this at the
moment.

While it is a dizzying array of extensions, EGL already includes all the
metadata and APIs to send the content encoding from the client (Totem, by way
of GStreamer) to the Wayland compositor.

### Vulkan WSI

The Vulkan Window System Interface requires much less bouncing between
extensions to determine how it all works.

For WSI, the content encoding is set when creating a
[swapchain](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html#_wsi_swapchain)
by providing a ``VkSwapchainCreateInfoKHR`` structure. This includes the pixel
format as well as the [color
space](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html#VkColorSpaceKHR).
Like EGL, the color space enumeration defines both the color primaries and the
transfer function together. There is support for sRGB, Display-P3, BT2020, and
more, all with linear and one or more transfer function options.

The HDR metadata can be provided by the client using
[vkSetHdrMetadataEXT](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html#_hdr_metadata),
which applies to the swapchain. With those two calls, a client can describe the
content encoding.

Like EGL, the WSI implementation in Mesa does not forward this content encoding
to the Wayland compositor because there's no protocol.

Ville Syrjälä created a proof-of-concept branch of Mesa back in 2017 to have
Mesa use the Wayland protocol proposal (of the time), and I [rebased
it](https://gitlab.freedesktop.org/jcline/mesa/-/tree/hdr_poc) recently to
"work" with the latest Wayland protocol proposal. It probably doesn't work, but
it is a good approximation of the work required, which is just implementing the
standard's functions to call libwayland functions with the content encoding
details.

## The Wayland Protocol

As we touched on in the Mesa section, the Wayland protocol is still being
discussed. The discussion is extensive, and I cannot do it justice in this
post, but in this section I'll outline the basics of what the protocol *must*
include and approximately what that would look like.

Wayland is a client-server
[architecture](https://wayland.freedesktop.org/architecture.html). The clients
are applications like the video player Totem. A Wayland display server handles
client requests and notifies them of relevant events.

You can, at this point, probably guess what information the protocol needs to
include. The client needs to inform the server of:

* The color primaries (red, green, blue, and white point coordinates)

* The transfer function used to encode the luminance.

* HDR metadata when it is available - in the case of Totem playing an HDR
  movie, HDR metadata will be available.

Without these facts, the Wayland display server cannot properly tone-map or
blend the content, nor could it offload those tasks to dedicated GPU hardware
or the display since they both need the same information.

The protocol includes an object that represents a rectangular area that can be
displayed, the
[wl_surface](https://wayland.freedesktop.org/docs/html/apa.html#protocol-spec-wl_surface).
The surface is backed by a
[wl_buffer](https://wayland.freedesktop.org/docs/html/apa.html#protocol-spec-wl_buffer).
The ``wl_buffer`` holds the content and the ``wl_surface`` describes the role
of the content and how the server should interpret the contents of the
``wl_buffer``, such as how it should be scaled or rotated. Since the
``wl_surface`` includes metadata about the content, it is probably the right
place to include other details about how the server should interpret the
buffer, namely the color primaries, transfer function, and HDR metadata. While
the color primaries and transfer function aren't likely to change over the
lifetime of a surface, the HDR metadata can change - HDR10+ and Dolby Vision
allow for frame-by-frame HDR metadata - so the protocol should account for
that.

Additionally, the Wayland display server should be able to inform the clients
of:

* The native color primaries of the display. This will *not* be a standard
  color space such as sRGB.

* Standard color spaces the display will accept (and internally convert to its
  native color space).

* Transfer functions the display can decode.

* Minimum luminance, maximum sustainable luminance, and peak (burst) luminance.

* HDR metadata formats the display will accept.

In the case of Totem playing an HDR movie, it would not need to concern itself
with tone-mapping and would not need to know the details listed above. However,
something like a video that generates content would benefit from targeting the
display's capabilities.

## Mutter

[Mutter](https://gitlab.gnome.org/GNOME/mutter) is, among other things, a
Wayland display server.

TODO: outline mutter work.

## The Kernel

Since the kernel deals with talking to hardware, it's the first stop in making
all this work. Fortunately, a good deal of work has already been done in the
kernel.

### Display Capabilities

The first thing we need to be able to do is determine the capabilities of the
display we're preparing content for. We need to know:

- The exactly color of each primary on the display (chromaticity coordinates).

- The maximum luminance. Ideally this would include the peak luminance, what
  percentage of the screen can achieve peak luminance at once, the maximum
  luminance that can be sustained on the entire display, and so on.
  Unfortunately, at the moment much of this information might not be available.

- The minimum luminance.

- What types of content encoded are allowed.

- The types of HDR metadata it supports. These fall into two categories:
  static and dynamic. Static metadata applies to the entire piece of media,
  whereas dynamic metadata can change from scene to scene or even frame to
  frame. There are several metadata formats, so we need to know which formats
  we can use.

#### EDID

Some of the display capabilities are listed in the
[EDID](https://en.wikipedia.org/wiki/Extended_Display_Identification_Data). The
EDID structure is made available to userspace by the kernel in the [EDID
connector
property](https://www.kernel.org/doc/html/latest/gpu/drm-kms.html#standard-connector-properties).

The base EDID specification is [available
online](https://vesa.org/vesa-standards/). The format describes the display
chromaticity coordinates, but has no details on luminance. For that, we'll need
to look to an extension block, which the base EDID specification describes how
to parse.

The CEA-861 specification describes an EDID extension that includes the HDR
metadata the display supports, as well as the minimum, maximum, and average
luminance levels the display wishes (if any). This structure is parsed in [the
kernel](https://elixir.bootlin.com/linux/v5.11/source/drivers/gpu/drm/drm_edid.c).
This gives us an idea of the structure format even if we don't have access to
the specification itself.

One unfortunate thing about EDID is that very few fields are required, so we
can't be sure that we'll find all the details we need. This is true of most
things in life, though, so we'll just have to do our best if the EDID is all we
have.

#### DisplayID

[DisplayID](https://en.wikipedia.org/wiki/DisplayID) is meant to replace EDID.
The latest revision, 2.0, was released in 2017 and its structure is inspired by
the CEA-861 EDID extension. It has many more required fields, including a block
on [display
parameters](https://en.wikipedia.org/wiki/DisplayID#0x21_Display_Parameters)
that includes:

- Chromaticity coordinates

- Maximum luminance (full display coverage)

- Maximum luminance (10% display coverage)

- Minimum luminance

- Color depth (bits per component)

- Display technology (OLED, LCD, etc)

The [display
interface](https://en.wikipedia.org/wiki/DisplayID#0x26_Display_interface_features)
section, also required, covers supported color space and transfer functions.

At this time, the DisplayID of a display is not exposed to userspace. It will
be nice to have, but we will also have to live with the EDID for quite some
time when the display doesn't offer a DisplayID table.

### Content Encoding, Metadata, and Accelerated Operations

Graphics hardware typically provides hardware dedicated to efficiently decode,
transform, and encode images with look-up tables (LUTs). These look-up tables
approximate the smooth curve of the desired transfer function by mapping
encoded values to optical values (de-gamma LUT) and optical values to encoded
values (gamma LUT). Transformations could include color space conversions
(CSC), blending images together into a single image for the display, and so on.

The existence of these features and the number of look-up table entries is
hardware-dependent, but we would like to use them when they are there and
provide enough accuracy for the transfer function rather than calculating the
values using, say, expensive CPU or general purpose graphics shaders cycles.

#### Encoding and Decoding

The hardware-backed look-up tables for encoding and decoding content with an
arbitrary transfer function are exposed to userspace as properties attached to
CRTC objects and are documented in the [color
management](https://www.kernel.org/doc/html/v5.11/gpu/drm-kms.html#color-management-properties)
section of the KMS interface.

The `GAMMA_LUT` and `DEGAMMA_LUT` properties and their respective lengths allow
us to approximate the smooth curves we saw in the section on transfer functions.

#### Color Space Conversions

The documentation for [color
management](https://www.kernel.org/doc/html/v5.11/gpu/drm-kms.html#color-management-properties)
interfaces also mentions the `CTM` property. The `CTM` property lets userspace
define a color transformation matrix to describe how to map from the source
color space to the destination color space. When combined with a de-gamma LUT
and gamma LUT, it can be used to efficiently decode, transform, and re-encode
content for a particular color space.

Note that both `CTM` and the look-up tables apply to a CRTC, which is
problematic if we wish to use the hardware to blend planes together that have
different content encoding.

#### Blending with Planes

It is possible to have hardware blend ("composite") images together using
[planes](https://www.kernel.org/doc/html/v5.11/gpu/drm-kms.html#plane-composition-properties).

However, the documentation does not mention using the `GAMMA_LUT` or
`DEGAMMA_LUT` to blend the content using linear, optical values so it is not
clear what impact this feature has on color correctness, even when the planes
being blended are all the in same color space.

More work is required here to use this feature when correct colors are a
concern. Harry Wentland from AMD recently started a [discussion on API changes
for
planes](https://lore.kernel.org/dri-devel/20210426173852.484368-1-harry.wentland@amd.com/)
which looks to address these shortcomings.

#### HDR Metadata

The kernel exposes a `HDR_OUTPUT_METADATA` [connector
property](https://www.kernel.org/doc/html/v5.11/gpu/drm-kms.html#standard-connector-properties)
that userspace can use to send HDR metadata to the kernel and thus to the display.

At this time there is only one [supported metadata
type](https://www.kernel.org/doc/html/v5.11/gpu/drm-uapi.html#c.hdr_output_metadata),
which corresponds to the static HDR metadata required for the HDR10 standard.

Other standards that use dynamic metadata rather than static metadata include
HDR10+ and Dolby Vision. More work is required here to support these standards.
