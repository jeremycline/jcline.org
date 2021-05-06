---
layout: post
title:  "HDR in Linux: Part 2"
date:   2021-05-14 15:05:08 -0400
categories: blog fedora graphics hdr
---

# HDR In the Linux Desktop

Now that we understand HDR - a larger luminance range that requires more bits
per component, new transfer functions to encode that luminance, and potentially
some metadata - we can examine the work required to use it in a "standard"
Linux desktop. As we'll see, it's already partially usable in "non-standard"
Linux environments.

To do this, we'll examine each portion of the stack, starting with the kernel
and working our way up.

Let's review the high-level requirements:

- We need to be able to determine what output formats the display or displays
  we're targeting are capable of. This includes the color primaries of the
  display, the luminance capabilities, what transfer functions the display can
  decode, and how many bits per component are permitted. This allows us to
  determine what output encoding to use.

- We need to be able to express the transfer function, bit depth, color space,
  and, if the PQ transfer function is in use, the HDR metadata for our content
  all the way from applications to the hardware transmitting to the display.
  This is required to correctly blend, tone-map, or otherwise transform the
  content. I refer to this as "content encoding".

- Somewhere we need to make the application's content encoding match the output
  encoding, even when there are multiple applications on the screen, even when
  they overlap, even when they are partially transparent and need to be blended
  together.

If you have ever worked with text content, you're probably familiar with text
encoding. Adding HDR support is very similar to adding support for Unicode when
previously everything assumed content was encoded with ASCII (or Latin-1).

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

## Wayland

Graphical applications like Firefox have a client-server relationship with a
Wayland compositor. The compositor services requests of client applications and
handles the interactions between the kernel and userspace. The interfaces
discussed in the section on the kernel are used by the Wayland compositor.

The compositor is in control of which applications receive inputs like keyboard
strokes and how the various applications are composed into a single image.
However, it is not creating the content that fills that image. The graphical
applications create the content, and are therefore in control of what color
space is used and how the content is encoded.

There needs to be a way for the client application to tell the compositor what
the content encoding is. We need the equivalent of the `Content-Type` header
used in email or HTTP. Unfortunately, the core Wayland protocols do not include
a way for clients to express how the content has been encoded. Instead, it is
assumed to be sRGB. This is a bit like assuming all text content in the world
is ASCII. Depending on where you are in the world it is often "right", but
there's a bunch of text out there that is not encoded with ASCII and it becomes
unreadable unless encoding is explicitly taken into account everywhere.

Fortunately, Wayland allows us to define extensions to the protocol. There has
been a [work-in-progress merge
request](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/merge_requests/14)
to do just that. Regardless of the final form this protocol takes, we can
expect that it will provide clients with a way to express:

- The color space of the data.
- The transfer function applied applied to the data.
- The luminance range of the data, if applicable (HDR metadata).

Additionally, it may include a way to explicitly express whether or not it
intends to tone-map the content. Some applications, like video games or media
creation applications like Blender, will want to handle tone-mapping content
for the display themselves. Other applications will just want the application
to match the brightness of desktop generally and leave the compositor to handle
that. Finally, some applications like media players may be able to provide
content that is not tone-mapped for the display, but includes the HDR metadata
necessary for the display to handle tone-mapping itself.

For a client to tone-map and gamut-map its content for a display, it needs to
know what the display capabilities are, so the Wayland protocol extension also
needs to provide a way for the compositor to inform clients of those
capabilities.

## Mesa

Clients render their content with Mesa, and this is where the Wayland protocol
changes are implemented on the client side.

[EGL](https://www.khronos.org/registry/EGL/sdk/docs/man/html/eglCreatePlatformWindowSurface.xhtml)
and
[Vulkan](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html#_wsi_swapchain)
both include the color space and transfer function associated with surfaces.
Additionally, both include extensions for expressing luminance details:
[EXT_surface_SMPTE2086_metadata](https://www.khronos.org/registry/EGL/extensions/EXT/EGL_EXT_surface_SMPTE2086_metadata.txt)
and
[vkSetHdrMetadataEXT](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html#_hdr_metadata).

These functions just need to map the EGL and Vulkan color space definitions to
Wayland definitions and send the data along to the Wayland compositor using the
protocol decided upon when called.

Ville Syrjälä created a proof-of-concept branch back in 2017 to do this, and I
[rebased it](https://gitlab.freedesktop.org/jcline/mesa/-/tree/hdr_poc)
recently to "work" with the latest Wayland protocol proposal. It needs more
polish and, of course, a compositor to do something with the information.

# Summary

In this post we covered what HDR is and how it maps into the lower levels of
the userspace graphical stack. In a follow-up post, I plan on covering the work
necessary in [Mutter](https://gitlab.gnome.org/GNOME/mutter) to support HDR, as
well as the work being done in Weston, Wayland's reference compositor.
