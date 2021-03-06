---
layout: post
title:  "HDR in Linux: Part 2"
date:   2021-06-28 14:36:30 -0400
categories: blog fedora graphics hdr
---

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
  bits per component, etc) in order to know what outputs are possible and/or
  optimal.

- Client applications may want to know what content encoding the Wayland
  display server supports and maybe even what the optimal content encoding is
  for the display it is on (the native primaries, transfer function, bit depth,
  luminance capabilities, etc)

- Client applications need to express the transfer function, bit depth, color
  space, and any HDR metadata for its content (the "content encoding") to the
  compositor. The compositor, in turn, needs to ensure the content from all the
  client applications are blended properly and output in a format the display
  can handle.

You might be familiar with text encoding. Adding HDR support is very similar to
adding support for various text encoding formats when previously everything
assumed content was encoded with ASCII. Instead of ASCII, the graphics stack
largely assumes everything is sRGB.

## Totem

At the top of the stack we have [Totem](https://gitlab.gnome.org/GNOME/totem),
which provides the user interface and playback management features. It relies
on GStreamer to deal with the video itself. It [appears to
use](https://gitlab.gnome.org/GNOME/totem/-/blob/V_3_38_0/src/backend/bacon-video-widget.c#L6155)
the
[playbin](https://gstreamer.freedesktop.org/documentation/playback/playbin.html)
plugin, which GStreamer describes as an "everything-in-one" abstraction for a
audio and video player.

Since Totem is all about wiring up a user interface to the GStreamer playbin
plugin, GStreamer needs to be HDR-aware. Totem uses GTK and Clutter-GTK to
create the user interface, so those libraries are also something we should
examine.

## GStreamer

Based on the GStreamer documentation and [this talk by Edward
Hervey](https://gstconf.ubicast.tv/videos/hdr-seeing-the-world-as-it-is/) all
the necessary features appear to already be implemented for HDR video playback.

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
it gets from the application, Totem, to the Wayland display server. The
client-to-server interaction occurs in the
[EGL](https://en.wikipedia.org/wiki/EGL_(API)) or [Vulkan Window System
Integration
(WSI)](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html#wsi)
APIs. These APIs are where OpenGL and Vulkan APIs are integrated into the
particular operating system. This is where buffers are allocated for use within
the graphics libraries and they include ways to provide the content encoding.
As long as the metadata GStreamer extracts is passed to the relevant EGL or
Vulkan API, we are all set. Mesa includes implementations of EGL and Vulkan
WSI, so we will examine those after GTK.

## Clutter-GTK

In the section on Totem, we noted that it uses
[Clutter-GTK](https://gitlab.gnome.org/GNOME/clutter-gtk). The documentation is
sparse and appears to assume the reader has an understanding of Clutter and why
one would want to use it.
[Clutter](https://developer.gnome.org/clutter/stable/) appears to be a library
for arranging a collection of 2-dimensional surfaces ("actors" in Clutter's
terms) in 3-dimensional space.

If we look at [Clutter's repository](https://gitlab.gnome.org/GNOME/clutter)
the README notes it is in "deep maintenance mode" and is intended to be
replaced with GTK 4. Based on the commit log this is true, so we will look no
further and focus on GTK 4.0.

## GTK

[GTK](https://docs.gtk.org/) is a "widget toolkit". It provides, either through
a dependency or on its own, an application abstraction, event loop, rendering
interfaces, methods to declare various user interface elements (buttons, text
fields, etc), and other useful utilities for building a graphical user
interface.

As of GTK 4, rendering is usually performed by OpenGL. As we'll examine later,
OpenGL APIs offer the necessary interfaces to handle HDR content, but GTK's
rendering model introduces a few issues. To understand the difficulties we need
to understand how GTK builds the application window.

### A Window into GTK

GTK documents its [drawing model](https://docs.gtk.org/gtk4/drawing-model.html)
in some detail and it would best to read through it before proceeding.

One of the most relevant parts of the documentation is that

> Each GTK toplevel window or dialog is associated with a windowing system
> surface. Child widgets such as buttons or entries don’t have their own
> surface; they use the surface of their toplevel"

GTK has each widget render itself, and then combines them all into a single
surface which it shares with the Wayland display server to be combined with all
the other client surfaces. We'll look at the Wayland protocol in detail later,
but what's important to know right now is that the surface is where the content
encoding should be stored.

This single surface that GTK produces, therefore, needs to be composed of
widgets that all use the same content encoding. Unfortunately, GTK does not
offer a way specify what content encoding to use for widgets it renders, nor
does it provide a way for users to convey the content encoding *they* are using
for their custom widgets. All content GTK produces on the application's behalf,
like buttons, menus, and the window decorations are rendered in an undefined
color space with an undefined transfer function. In practice, this seems to be
sRGB.  All textures GTK creates are hard-coded to use 8 bits per component,
including when it combines all the widgets into a single image.

This means, unfortunately, that at the moment there is no way to use GTK for an
application interested in producing HDR content (video players, image viewers,
content creation tools, etc).

### Ways Forward

There are a few ways for GTK to work in a world where sRGB isn't the only
content. I can't really say which approach is best, or even if this is an
exhaustive list of options since my knowledge of GTK is only a few days old.
Regardless, a good bit of work is necessary to make GTK HDR-ready.

#### Widgets with Content Encoding

One solution, if GTK wishes to remain in the business of blending images
together, is to ensure each widget includes a way to express how the content it
produces is encoded. GTK can then use this to ensure they are combined
correctly. If, for example, sRGB content and PQ-encoded content were combined
into a destination image without converting the sRGB content, the end result
would be that the sRGB portions of the image would be extremely dark. This, of
course, introduces a reasonable amount of complexity to GTK as it needs to
convert between a potentially long list of formats in addition to introducing
interfaces for specifying the content encoding everywhere it matters.

#### Sub-surfaces

Wayland offers a way to create surfaces with a parent-child relationship called
[subsurfaces](https://wayland.freedesktop.org/docs/html/apa.html#protocol-spec-wl_subsurface).
Since the surface is where the content encoding is stored, GTK could use
sub-surfaces to defer the blending of different content encoding to the Wayland
display server. This would allow GTK to remain in its undefined, sRGB-ish world
while allowing applications to handle modern content with other content
encoding. However, I am told these are problematic as they cannot be clipped or
transformed by GTK.

#### Require a single HDR Format

Rather than handling converting all the widgets to the same content encoding,
GTK could specify *one* HDR-capable format it supports and require all widgets
to use that if HDR has been requested for the application. A format like scRGB
with half-precision 16-bit floating point numbers for each color component.
This, however, has the downside that all HDR content needs to be converted from
its native format to scRGB, blended with the other GTK widgets, and then
converted back to some on-the-wire HDR format in the Wayland display server.

Even with this approach, GTK cannot blend the SDR content into HDR content
without considering what luminance range to map SDR content to, which needs to
correspond to the display brightness level the user has set.

#### Use Sub-surfaces Outside GTK

Applications can work around GTK not dealing with content encoding by not using
GTK for anything other than the menus, buttons, and so on. In fact, this is how
Firefox handles things; GTK is used for the "chrome" and the content is
rendered to a Wayland sub-surface which Firefox manages itself and overlays on
the GTK surface. This sub-surface can be properly configured for HDR and the
compositor can handle blending it with GTK's surface.

This, of course, isn't ideal as the applications have to work around the
toolkit rather than having the toolkit help them. However, it is a path
forward worth mentioning.


## Mesa

Mesa provides the OpenGL implementation that clients like GTK use to render
their content. To allocate storage for the output of that rendering, clients
use the EGL or Vulkan WSI interfaces for which Mesa also provides
implementations.

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
"work" with the latest Wayland protocol proposal. It doesn't *actually* work
yet, but it is a good approximation of the work required, which is implementing
the standard's functions to call libwayland functions with the content encoding
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
  movie, HDR metadata will be available. This metadata may remain the same for
  many frames (perhaps the whole movie), or it may change frame-by-frame.

Without this information, the Wayland display server cannot properly tone-map
or blend the content, nor could it offload those tasks to dedicated GPU
hardware or the display since they both need the same information.

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
with tone-mapping and would not need to know about what displays it might be
targeting. However, something like a video that generates content would benefit
from targeting the display's capabilities.

## Mutter

[Mutter](https://gitlab.gnome.org/GNOME/mutter) is, among other things, a
Wayland display server. Once a protocol is in place, Mutter needs to implement
it. To support HDR, Mutter needs to be able to:

* Inform clients of display capabilities.

* Act upon the content encoding provided by the clients to correctly convert
  all content to the same encoding and blend them all together into the desktop
  we know and love.

To inform clients of the display capabilities, Mutter can use the Extended
Display Identification Data (EDID) from the kernel, which contains the display
color primaries, supported HDR formats, and so on. It will be covered in detail
in the kernel section below. The EDID is, in fact, already being used in Mutter
so this will be a small addition to that. There is some trickiness around
multi-monitor setups as clients don't generally know which monitor they are
being displayed on, but displays with differing pixel density have a similar
challenge and Wayland handles that by providing events to the application when
it enters a new display and the display content encoding could work the same
way.

To act upon the client-provided content encoding, however, requires a bit more
work.

### Consolidating Existing Color Management

GNOME already has some color management capabilities. At the moment, they are
split up in gnome-settings-daemon, colord, and Mutter.

[colord](https://github.com/hughsie/colord/) is a system service that maintains
a database of devices, color profiles, and the association between the two
(including things like the default profile). At the moment, however,
gnome-settings-daemon is responsible for querying colord and sets up the
profile by calling Mutter D-Bus APIs.

Mutter needs to be aware of each display's color profile in order transform the
client content if necessary. The existing API for setting color profiles is
also difficult to use because it acts on CRTC objects rather than displays.
Some displays are composed of multiple panels tiled together and are therefore
backed by several CRTCs. It makes more sense for Mutter to query colord itself
rather than having gnome-settings-daemon do it.  gnome-settings-daemon also
handles the Night Light feature, which adjusts the color profile to shift the
white point towards red to remove the blue light.  Mutter needs to either
handle the Night Light feature or, preferably, provide a "temperature" API and
allow Mutter users to control when and how much to adjust the color
temperature.

### Support Converting Buffer Formats

There are a number of ways to store an image, but they generally fall into two
categories. The first is to represent the red, green, and blue components, and
that's the way we've been discussing images in this blog post series.

Another way is to represent the luminance of a pixel, and then two color
components for that same pixel. The third color component can be calculated
using the lumanince and two known color components. This approach is commonly
referred to as [YUV](https://en.wikipedia.org/wiki/YUV) although there are
several flavors. This is a valuable approach because humans are more spatially
sensitive to luminance than they are color which means it's possible to produce
a reasonably good-looking image even if you keep track of, say, every other
pixel's color. This is called [chroma
subsampling](https://en.wikipedia.org/wiki/Chroma_subsampling).

A lot of content, like films, are encoded using this approach, but at the
moment Mutter does not support YUV formats. It would be convenient, since we're
adding a way for clients to communicate content encoding, to handle such
formats. There has been some work in this area by [Niels De
Graef](https://gitlab.gnome.org/GNOME/mutter/-/commits/wip/nielsdg/meta-multi-texture-wip).

### Compositing the Content

Once Mutter has knowledge of the content encoding of every surface, it can
blend them together and convert them to the target encoding of the display.
This involves:

1. Convert YUV to RGB if necessary.

2. Perform a color space transformation so that all surfaces are using the same
   color primaries and whitepoint.

3. Removing the transfer function on surfaces so that Mutter is dealing with
   the optical (linear) values rather than the non-linear on-the-wire encoding.
   Adding two non-linear values together without accounting for the
   non-linearity results in incorrect luminance levels. You can try this with
   the following Python snippet:

   ```
   def gamma(u):
       """Apply the sRGB gamma definition to linear light levels"""
       return 1.055 * u**(1/24) - 0.055
   ```

   Consider adding linear light level 50 to linear light level 50 and then
   applying the transfer function versus adding the non-linear encoding of 50
   to itself:

   ```
    >>> gamma(100)
    1.2231616798531606
    >>> gamma(50) + gamma(50)
    2.3735497958717904
   ```

4. Blend the surfaces together. This becomes tricky with HDR since SDR does not
   have a clearly defined luminance range, it's just how bright your display is
   set, but PQ-encoded content does have a well-defined luminance range. Mutter
   will need define the SDR luminance range and map that content into that
   well-defined range to ensure SDR content isn't too dim or too bright.

### Tone-mapping

With HDR, clients may provide surfaces that encode luminances beyond the
capabilities of the target display. Displays account for this by tone-mapping
out-of-range content into the display range. This is described in [Section
5.4.1 of
BT.2390](https://www.itu.int/dms_pub/itu-r/opb/rep/R-REP-BT.2390-8-2020-PDF-E.pdf).
The important thing to know is that providing the display with out-of-range
luminance levels will result in something that looks fine to most end users,
but there may be a desire to carefully control what happens when this occurs.

Mutter would be responsible for implementing alternate tone-mapping if the
display's default behavior is unacceptable for the user. This would most likely
be a concern for content creators.

## The Kernel

We have arrived at the kernel, the final stop in this journey down the stack.
The good news is that the basic requirements for HDR have already been met.
We'll touch on those and then look at additional features necessary for us to
use certain hardware features.

### The Bare Minimum

These are the bare minimum features for HDR and are already present in the
kernel.

#### Display Capabilities

As we noted in the Mutter section, the
[EDID](https://en.wikipedia.org/wiki/Extended_Display_Identification_Data)
structure provided by a display is already available to userspace via the [EDID
connector
property](https://www.kernel.org/doc/html/latest/gpu/drm-kms.html#standard-connector-properties).
Userspace can parse it for the chromaticity coordinates of the display
primaries, the luminance, supported color spaces, and supported transfer
functions. At the moment, many different projects implement parsing the EDID
structure, but there have been some [requests for a
library](https://lists.x.org/archives/xorg-devel/2021-April/058690.html) to do
so.

One thing worth noting is that EDID is quite old and has a replacement
standard, DisplayID. Both standards are available from
[VESA](https://vesa.org/vesa-standards/) for free, although you do need to
provide your name and email address to access them. DisplayID has a [Display
Parameters](https://en.wikipedia.org/wiki/DisplayID#0x21_Display_Parameters)
block that includes luminance and chromaticity values, and the [Display
Interface
Features](https://en.wikipedia.org/wiki/DisplayID#0x26_Display_interface_features)
block contains supported color space and transfer function combinations as well
as bit depths. EDID included much of this information, but only as an extension
documented in CTA-861 (currently at revision G). Recently CTA made this
[available for
free](https://shop.cta.tech/products/a-dtv-profile-for-uncompressed-high-speed-digital-interfaces-cta-861-g),
although you must provide an email and billing address to download it.

Displays can provide an EDID, DisplayID, or both. For the curious, the VESA
E-DCC standard documents how these structures are accessed over
[I²C](https://en.wikipedia.org/wiki/I%C2%B2C). How I²C is sent over, for
example, DisplayPort is documented within the DisplayPort specification, which
is only available to VESA members so the very curious will have to resort to
reading the [Linux kernel's
implementation](https://elixir.bootlin.com/linux/v5.12/source/drivers/gpu/drm/drm_dp_helper.c#L226)
if they aren't members.

DisplayID can be embedded in the EDID structure as an extension so it may be
available to userspace via the existing EDID connector property, but at this
time there is not a dedicated kernel interface to retrieve a standalone
DisplayID structure that I am aware of. The work to add one would be minimal,
however, since it's extremely similar to the EDID.

#### HDR Metadata

The kernel exposes a `HDR_OUTPUT_METADATA` [connector
property](https://www.kernel.org/doc/html/v5.12/gpu/drm-kms.html#standard-connector-properties)
that userspace can use to send HDR metadata to the kernel and thus to the display.

At this time there is only one [supported metadata
type](https://www.kernel.org/doc/html/v5.12/gpu/drm-uapi.html#c.hdr_output_metadata),
which corresponds to the HDR metadata defined in CTA-861 in section 6.9
"Dynamic Range and Mastering InfoFrame". This *should* be all that is required
for HDR10 and HDR10+ content.

#### Bits Per Component

The ``max bpc`` standard connector property is a [range
property](https://www.kernel.org/doc/html/latest/gpu/drm-kms.html#property-types-and-blob-property-support)
that allows the hardware driver to indicate valid bits per color component.
Userspace can set this property so long as it is within the driver-provided
range to control the output bit depth.

### Nice-to-have Features

The bare minimum feature set is present for HDR content and allows us to query
display capabilities, pass HDR metadata on to the display, and configure the
bits-per-component sent to the display.

This leaves userspace to handle color space transformations, applying the
transfer function's inverse to get linear light values, blending, and
re-encoding the results with a (perhaps different) transfer function. All that
is possible, of course, but there are a number of hardware features for all
those operations and it would be great to use them when they are there rather
than performing the operations using expensive CPU or general purpose graphics
shaders cycles.

#### Planes

Before we cover the encoding, decoding, color space transformations, and so on,
it's important to become familiar with
[planes](https://www.kernel.org/doc/html/v5.12/gpu/drm-kms.html#plane-abstraction).
Planes hold images and descriptions of how those images should be blended into
the final output image. These are analogous to a Wayland surface, which
contains a buffer with the image data and metadata for the image.

Planes describe how they should be cropped, scaled, rotated, and blended with
other planes to compose the final image sent to the display. These planes are
backed by dedicated hardware that can efficiently do those operations. For the
curious, the [display
engine](https://01.org/sites/default/files/documentation/intel-gfx-prm-osrc-dg1-vol12-displayengine.pdf)
section of Intel's documentation describes some of the hardware plane's
capabilities. Of particular interest is the diagram around page 258 which
illustrates the plane pipeline.

One or more planes are used to compose the image the
[CRTC](https://www.kernel.org/doc/html/v5.12/gpu/drm-kms.html#crtc-abstraction)
scans out to the display.

#### Decoding, Color Space Transformations, Blending, and Encoding

Graphics hardware typically provides hardware dedicated to efficiently decode,
transform, and re-encode images. Encoding and decoding are done with look-up
tables that approximate the smooth curve of the desired transfer function by
mapping encoded values to optical values and optical values to encoded values.
For historical reasons, these are referred to as the gamma and de-gamma LUTs
respectively.

The hardware-backed look-up tables for encoding and decoding content with an
arbitrary transfer function are exposed to userspace as properties attached to
CRTC objects and are also documented in the [color
management](https://www.kernel.org/doc/html/v5.12/gpu/drm-kms.html#color-management-properties)
section of the KMS interface. The `GAMMA_LUT` and `DEGAMMA_LUT` properties and
their respective lengths allow us to set arbitrary transfer function
approximations. The number of look-up table entries is hardware-dependent so it
may not be appropriate to use them in all cases.

Color space transformations are applied by using the `CTM` (colorspace
transformation matrix) property defined in the [color
management](https://www.kernel.org/doc/html/v5.12/gpu/drm-kms.html#color-management-properties)
interface. The `CTM` property lets userspace define a color transformation
matrix to describe how to map from the source color space to the destination
color space. When combined with a de-gamma LUT and gamma LUT, it can be used to
efficiently decode, transform, and re-encode content for a particular color
space.

Finally, the aforementioned planes include [composition
properties](https://www.kernel.org/doc/html/v5.12/gpu/drm-kms.html#plane-composition-properties)
that allow userspace to control how planes are blended together.

*However*, these APIs have a few problems. The color management properties are
attached to the CRTC, which is intended to abstract the display pipeline and is
made up of planes. If the planes that make up the CRTC do not all have the same
transfer function applied to them or use different color spaces, there is no
way to express that each plane needs its own look-up table or color
transformation matrix.

While it used to be reasonably safe to assume all the planes were the same
(e.g. sRGB), with HDR this is no longer the case. There are likely going to be
HDR planes and SDR planes that need to be blended together, so there needs to
be a way to express the color management properties - the content encoding - on
a plane-by-plane basis. In addition to the content encoding of each plane, we
need to be able to express *how* HDR and SDR content should be blended. In the
case of PQ-encoded HDR content, it is expressed in absolute luminance up to
10,000 nits. Userspace needs to express how the SDR luminance range maps into
the HDR luminance range so the SDR encoding can be converted into HDR encoding.

Harry Wentland from AMD recently started a [discussion on API changes for
planes](https://lore.kernel.org/dri-devel/20210426173852.484368-1-harry.wentland@amd.com/)
which looks to address these shortcomings.

## Summary

All told, a good portion of the work necessary for basic HDR support is done or
in progress. Some of the larger challenges are in the compositors as they need
to line up client capabilities with hardware capabilities and make up any
differences between the two.

Applications that use OpenGL or Vulkan directly don't need to concern
themselves with GTK's lack of HDR support. However, there are a non-trivial
handful of applications that would benefit from GTK supporting HDR content,
like Totem, Eye of GNOME (the image viewer), and content creation tools.

Finally, there's plenty of work on the kernel side to ensure userspace can make
use of all the hardware available to it.
