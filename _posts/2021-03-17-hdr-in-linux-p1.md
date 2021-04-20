---
layout: post
title:  "HDR in Linux"
date:   2021-04-09 14:32:38 -0400
categories: blog fedora graphics hdr
---

In recent months I have been investigating high dynamic range (HDR) support for
the Linux desktop, and what needs to be done so that a user could, for example,
watch a high dynamic range video.

The problem with HDR is not so much that there is no material out there
covering it, but that there's a huge amount of material can be rather
confusing. I thought it best to add more material that is probably also
confusing, but may help me think through HDR better. The following content is
likely wrong as I have no background in colorimetry, the human visual system,
or graphics generally.

## Background

Before attempting to understand dynamic range, I found it helpful to understand
the go through the basics of electromagnetic radiation (light) and how the
human visual system (eyes) interact with light so that we can produce graphics
for human consumption.  With an understanding of how to do that, we can
complete our ultimate goal of tricking the human visual system into seeing
extremely realistic cats even if there are no cats around.

### Light

Human eyes detect a very small range of electromagnetic radiation. Light can be
described as a wave with an amplitude and frequency/wavelength or a particle
(photon) with a frequency/wavelength. More particles is equivalent to a higher
amplitude wave. The range of wavelengths eyes typically see is approximately
380 nanometers to 750 nanometers.

![The visible light spectrum](/assets/visible_spectrum.png)

Humans detect wavelength of light as colors, with blue in the 450-490nm range,
green in 525-560nm range, and red in the 630-700nm range.

More photons of a given wavelength are perceived by humans as a brighter light
of the color associated with the wavelength.

### Luminance

[Luminance](https://en.wikipedia.org/wiki/Luminance) is the measure of the
amount of light in the *visibile spectrum* that passes through an area.  The SI
unit is candela per square meter (cd/m²). This is often also referred to as a
"nit", presumably because cd/m² is somewhat laborious to type out.

The human eye can detect a luminance range from about 0.000001 cd/m² to
around 100,000,000 cd/m². We can readily go out into the world and [experience
this entire range,
too](https://en.wikipedia.org/wiki/Orders_of_magnitude_(luminance)). For
example, the luminance of the sun's disk at noon is around 1,600,000,000 cd/m².
This is much higher than the human eye can safely experience; do not go stare
at the sun.

One important fact is that the human perception of luminance is [roughly
logarithmic](https://en.wikipedia.org/wiki/Weber%E2%80%93Fechner_law). You
might already be familiar with this as
[decibels](https://en.wikipedia.org/wiki/Decibel) are used to describe sound
levels. The fact that human sensitivity to luminance is non-linear becomes very
important later when we need to compress data.

### Human Visual System

The eye is composed of two general types of cells. The rod cells which are very
sensitive and can detect very low amplitude (brightness) light waves, but don't
differentiate between wavelengths (color) well.

The cone cells, by contrast, come in several flavors where each flavor is
sensitive to a different wavelength. Many humans have three flavors,
imaginatively referred to as S, M, and L. S detects "short" wavelengths (blue),
M detects "medium" wavelengths (green), and L detects "long" wavelengths (red).

![A diagram of the sensitivity of typical cone cells](https://upload.wikimedia.org/wikipedia/commons/0/04/Cone-fundamentals-with-srgb-spectrum.svg)

Different colors are produced by stimulating these cone cells in with different
ratios of wavelengths.

One thing to keep in mind, as it adds a minor complication to calculating
luminance, is that not all the cone cells are equally sensitive. If you were to
take a blue light, a green light, and a red light that all emit the same number
of photons, the green light would appear the brightest by far. The luminance,
therefore, depends on the wavelength of the light.

While the human visual system is capable of detecting a very large range of
luminance levels, it cannot do so all at once. If you're outside on a sunny day
and walk into a dark room, it takes some time for your eyes to adjust to the
new luminance levels.

### Displays

There are several ways to display images for human consumption. One way is to
reflect light off a surface that absorbs some frequencies of light and reflect
the rest into the eye (paintings, books, e-ink, etc). The other way is to
produce light to shine into the eye, such as the liquid crystal or organic LED
screen you're probably looking at right now.

There's a few challenges with producing the light directly. The first is that
we need to produce a range of light frequencies to cover the visible spectrum
of human vision. As humans usually have three types of frequency-sensing cells,
we don't necessarily need to produce every single frequency in the visible
spectrum. Instead, we can pick three frequencies that align with where the cone
cells are sensitive and by adjusting the ratio of these three produce light
that contains all information the human eye uses and none of the information it
cannot detect.

This is why a display pixel is typically made up of blueish, greenish, and
reddish components. Since this isn't a perfect world, displays don't tend to be
capable of producing a single frequency. The frequency range of these
components impact how the three overlapping cone sensitivities can be
stimulated and therefore determine the range of colors the display is capable
of producing. This range is referred to as the display's "color gamut". The
color gamut can vary quite a bit from display to display, so we need to account
for this when we encode images. We also need to handle cases where the display
cannot produce the color we wish to show. We could not, for example, accurately
display an image of a laser emitting pure 525nm light unless the green
sub-pixel happened to also emit pure 525nm light. This is called "gamut
mapping" as we map one color gamut to another color gamut.

The second issue is that if we want to perfectly replicate the real world, we
would need to be able to emit light in the range of 0.000001-1,600,000,000
cd/m². However, as we've already established, the sun is not safe to look at so
it would be best to not faithfully reproduce it on a display. Even if we
restrict ourselves to the 0.000001-100,000,000 cd/m² range, the power
requirements and heat produced would be astronomical. Additionally, the human
eye cannot simultaneously detect the full range so our efforts at realism would
be wasted. We have to attempt to represent the world using a smaller luminance
range than we actually experience. In other words, we have to compress the
luminance range of our world to fit the luminance range of the display. This
process is often referred to as "tone-mapping".

### Dynamic Range

Dynamic range is the ratio between the smallest value and largest value. When
used in reference to displays, the value is luminance.

The lowest luminance level is determined by how much ambient light is
reflecting off the surface of the display (e.g. are you sitting in a dark room,
outside on a sunny day, etc), how much of the backlight (if there is one)
leaks through the display panel, and so on.

The highest luminance level is determined by the light source of the display.
This depends on the type of display, power usage requirements, user
preferences, on the ambient temperature (emitting visible light also emits
heat, and heat can damage things), and perhaps other factors I am not aware of.

As both these values depend on environmental factors the dynamic range of a
display can change from moment to moment, which is something we may wish to
account for when compressing the luminance range of the world to fit the
particular display.

So what is a high dynamic range display? In short, a display that is
simultaneously capable of lower luminance levels and higher luminance levels
than previous "standard dynamic range" displays. What these levels are exactly
can be hand-wavy, but [VESA provides some
specifications](https://displayhdr.org/performance-criteria/).

This higher dynamic range ultimately means the way in which we compress the
luminance range (tone-map) needs to change.

## Light from Scene to Display

In this section, we'll cover the journey of an image from its creation to its
display. Perhaps the most self-contained example is a modern video games. This
allows us to dodge the added complication of camera sensors, but the process
for real-world image capture is similar.

### Scene-referred Lighting

Video games typically model a world complete with realistic lighting.  When
they do this, they need to work with "real world" luminance levels. For
example, the sun that lights the scene may be modeled to be seen as 1.6 billion
nits. Working with real world luminance that cannot be directly displayed is
often called "scene-referred" luminance. We have to transform the
scene-referred luminance levels to something we can display ("display-referred"
luminance) by compressing or shifting the values to a displayable range.

You might think it's best to leave content in scene-referred lighting levels
until the last possible moment (when the display is setting the value for each
pixel), and that would indeed make some things simpler. However, it makes some
things more difficult, like attempting to blend the scene-referred image with
an image that is already display-referred. When doing operations like that, we
need every part to be in the same frame of reference, either scene-referred or
display-referred.

There's a more difficult problem with keeping all content scene-referred,
though. Representing numbers accurately in the 0.000001-1,600,000,000 cd/m²
range requires a lot of bits, and we're about to have a serious bit budget
imposed on us. We have to transport the image data from the host graphics
processing unit to the display. While both HDMI and DisplayPort have rather
enormous bandwidth, the image resolution is also quite large. If we used a 32
bit floating point number for each red, green, and blue (RGB) component of a
pixel on a display with 3840 x 2160 (4K) pixels, it would require 32 x 3 x 3840
x 2160 bits per frame, or 760 Mbits per frame. At a modest 60 frames per
second, that's 44.5 Gbits. DisplayPort 1.4 provides 25.9 Gbits and HDMI 2.0
give us 14.4 Gbits.

Fortunately for us, as long as we ensure the way we encode luminance levels
matches up with the way the human eye detects luminance levels, we can save a
*lot* of bits. In fact, for the "standard" dynamic range, we can manage with
only 8 bits (256 luminance levels) per color (24 bits total for RGB).

### Tone-mapping

In order to convert scene-referred, real world luminance to a range for our
target display, we need to:

1. Have an algorithm for mapping one luminance value to another. Input values
   have a range [0,∞) and the output values need to cover the range of our
   target display.

2. Know the capability of our target display.

For now we'll focus on #1, although #2 is important and, unfortunately,
somewhat tricky.

There are many different approaches to tone-mapping and results can be
subjective. This is not a post about tone-mapping (there are many multi-hundred
page publications on the subject), we'll just go over a couple examples to get
the idea.

An extremely silly, but technically valid tone-mapping algorithm would be `f(x)
= 1`. This maps any input luminance level to 1.

A more useful approach might be the function `f(x) = x^n/(x^n + s^n)` where `x`
is the input luminance, `n` is a parameter changes the slope of the mapping
curve, and `s` shifts the curve on the horizontal axis. As a somewhat random
example, with `n = 3` and `s = 10`:

![Example tone mapping curve that maps all input luminances between 0 and 1](/assets/example-tone-map.png)

This [sigmoid function](https://en.wikipedia.org/wiki/Sigmoid_function) gently
approaches the minimum and maximum values of 0 and 1. We can then map the [0,
1] range to the display luminance range.

There are many articles and papers out there dedicated to tone mapping. There's
a chapter on the topic in "The high dynamic range imaging pipeline" by Gabriel
Eilertsen, for example, which include a number of sample images and a much more
thorough examination of the process than this blog post.

This process is occasionally referred to as an "opti-optical transfer function"
or "OOTF" as it maps optical (luminance) values to different optical values.

### Encoding

Once we've tone-mapped our content to use the display's luminance range, we
need to send the content off to the display so that we can see it. As the
section on scene-referred light mentioned, we have a limited amount of
bandwidth, and any increase or decrease in bits used per pixel impacts how many
bits are required for a frame, and thus how many frames we can send to the
display every second. The key is to use as few bits as we can get away with.

The method used to encode luminance levels is called the "[transfer
function](https://en.wikipedia.org/wiki/Transfer_functions_in_imaging)",
"opto-electronic transfer function", or "OETF". These are functions that
approximate the human's sensitivity to luminance levels.

Since the display needs to decode the encoded data we send it back to linear
luminance values, we need functions that can be easily "undone" with an inverse
function. The property we're looking for in our encoding and decoding functions
is `g(f(x)) = x`. This is called the "inverse transfer function",
"electro-optical transfer function", or "EOTF" since it converts the
"on-the-wire" values back to optical values.

#### Encoding SDR

The "standard dynamic range" is not particularly standard, but usually tops out
around 300 nits. Typically, 8 bits per RGB component is used. 8 bits allow us
to express 256 luminance levels per component.

##### The Naïve Approach

One method would be to evenly spread each level across the luminance range.
What would this look like? In our thought experiment, we'll assume the
monitor's minimum luminance is 0.5 nits, and its maximum luminance is 200 nits.
This gives us a range of 199.5 nits. Equally distributed, each step increases
the luminance by about 0.78 nits.

Sending [0, 0, 0] for red, green, and blue results in the minimum luminance of
0.5 nits. Adding a step [1, 1, 1] would result in 1.28 nits.

Sending [254, 254, 254] results in 198.44 nits, and sending [255, 255, 255]
results in 200 nits.

![Graph of linear display-referred luminance mapped to code points](/assets/linear-encoded-luminance.png)

The problem with this approach is that with lower luminance values, each step
is above what the human eye differentiate, its [just-noticeable
difference](https://en.wikipedia.org/wiki/Just-noticeable_difference). At
higher luminance levels, each step is so far below the just-noticeable
difference that it would require many steps for a human to notice any
difference at all. It results in clearly visible bands in the lower luminance
areas of the image we display and completely undetectable detail in the higher
luminance areas of the image. We could resolve this by adding more bits (and
therefore more levels), but we can't afford to do that without giving up
resolution and framerate.

Instead, we want to take these wasted levels at high luminances and use them in
lower luminances.

##### Gamma

The "gamma" transfer function has a long and interesting history and is part of
the [sRGB standard](https://en.wikipedia.org/wiki/SRGB). The sRGB version is
actually a piecewise function which uses the gamma function for all but the
very lowest luminance levels, but we can ignore that complication here.

The basic function is in the form `f(x) = Axᵞ`. In sRGB, `A = 1.055` and `𝛾 =
1/2.4`. The inverse function used to decode the luminance is `A = 1/1.055` and
`𝛾 = 2.4`

The encoding curve looks like this:

![Graph of the gamma function with gamma = 1/2.4](/assets/gamma-encoded-luminance.png)

In this graph, we map the linear, display-referred luminance values into the
range [0, 1]. These input luminances are mapped to an encoded value also in the
range [0, 1]. We can see from the graph above that the lower third of the
linear luminance levels get mapped on the lower two thirds of our encoding
range.

We can convert the output to integer values between 0 and 255 by multiplying
the encoded luminance value by 255 and rounding:

![Graph of the gamma function with gamma = 1/2.4 mapped to 0-255](/assets/gamma-encoded-mapped-255.png)

Now, instead of wasting many of our precious bits on luminance steps on the
high end that no one can see, we're spending most of them on low luminance
levels.

This is a pretty good approach when we have 256 steps (8 bits) per component
and luminance range of a few hundred nits, but the further we stretch the range
of nits, the bigger our steps have to be, and eventually we'll cross a point
where some steps are about the just-noticeable difference line.

We can fix this by increasing the number of bits used. What happens if we move
to 10 or even 12 bits? At 12 bits per component, we have 4096 steps.

![Graph of the gamma function with gamma = 1/2.4 mapped to 0-4096](/assets/gamma-encoded-mapped-4096.png)

At first glance this seems fine, but what if we adjust the graph to display
actual nit values for the linear light? Here's the same graph with linear light
using a range of 0-1000 nits.

![Graph of the gamma function with gamma = 1/2.4 mapped to 0-4096](/assets/gamma-encoded-mapped-4096-1knits.png)

We can see that fewer than half our steps, or approximately 2000, land in the
luminance ranges we use today for standard dynamic range displays. Once again,
we're allocating too many steps to the very high luminance levels where the
human eye becomes less and less sensitive. This is even worse as the luminance
range increases:

![Graph of the gamma function with gamma = 1/2.4 mapped to 0-4096](/assets/gamma-encoded-mapped-4096-10knits.png)

Thus, for displays with a higher dynamic range, we may want to adjust the
transfer function to get the most value out of each bit we spend.

#### Encoding Higher Dynamic Ranges

As we saw when examining the gamma transfer function, it works reasonably
well for low luminance ranges, but as the range increases it starts to be
sub-optimal.

There are two major approaches for higher dynamic range encoding.

##### The Perceptual Quantizer

The [Perceptual Quantizer](https://en.wikipedia.org/wiki/Perceptual_Quantizer)
(PQ) transfer function, sometimes referred to by the much less cool name "SMPTE
ST 2084", defines a new curve to use for encoding. It is unlike the gamma
transfer function in that it is defined to cover a well-defined luminance
range, rather than just being fit to whatever the display happens to be capable
of. The luminance range the curve is defined to cover is 0.0001-10,000 cd/m².

This is a useful concept when encoding the content as we can now express
physical quantities in addition to "more/less luminance". It gives the decoding
side a frame of reference if it needs to further tone-map the content.

The curve looks like:

![Part of the Perceptual Quantizer curve](/assets/pq-encoded-luminance.png)

This graph is only part of the curve, and you can see the slope is so extreme
it doesn't render very well. It packs a vast majority of the code points into
the lower luminance range where the eye is most discerning. Currently, it's
common to use this curve with 10 bits per component.

The function that describes this curve is more complicated than the gamma curve
so I won't attempt to render it poorly here. The curious can consult the
Wikipedia page linked above.

The perceptual quantizer transfer function is paired with metadata describing
the image or images it is used to encode. This metadata is usually the primary
colors of the display used to create the content (which red, green, and blue)
as well as luminance statistics like the image's minimum, maximum, and average
luminance. This metadata is particularly useful when the display used to create
the content is not the same as the display used to view the content (movies, TV
shows, etc).

The PQ curve supports encoding luminance up to 10,000 nits, but there are no
consumers displays capable of that. Even professional displays built for
movie-making are well short of that, currently around 4,000 nits. This is still
much higher than current consumer displays, so the metadata can be used to
further tone-map the content from the levels the film studio used to the
capabilities of the individual display. In the future, when consumer monitors
become more capable, the same films will look better as they require less
tone-mapping and the original intent can be more accurately rendered.

##### The Hybrid Log-Gamma

As the name suggests, the second approach for higher dynamic range encoding is
to use a hybrid of the gamma function and a log function. The [hybrid
log-gamma](https://en.wikipedia.org/wiki/Hybrid_Log-Gamma) (HLG) curve is
designed with backwards-compatibility in mind.

HLG uses a piecewise function defined as follows when the input luminance has
been normalized to a range of 0-1:

- When the input luminance `L` is between 0 and 1/12: `sqrt(3) * L^0.5` (this
  is our old friend the gamma function `f(x) = Axᵞ`, with `A = 1.732` and `𝛾 =
  1/2`)

- When input luminance `L` is between 1/12 and 1: `a * ln(12L - b) + c` where `a =
  0.17883277`, `b = 1 - 4a`, and `c = 0.55991073`.

![The Gamma and Log curves in the HLG transfer function](/assets/hlg-encoded-luminance.png)

In the above graph, the red curve is the gamma curve, the green curve is the
log curve, and the vertical line is the point where the gamma curve stops being
used in favor of the log curve when encoding.

As this encoding scheme is defined in terms of relative luminance like the
gamma transfer function, there is no metadata for tone-mapping.

### Transmission and Display

Once we've encoded the image with a transfer function the display supports, the
bits are sent to the display via HDMI or DisplayPort. At this point, what
happens next is up to whatever is on the other end of the cable. I have no
pulled apart a display and learned what secrets it holds, but we can make some
reasonable guesses without destroying anything:

- If metadata is provided (PQ-encoded images only), the display examines it and
  determines of any tone-mapping is required due to the content exceeding its
  capabilities.

- If the display includes light sensors to detect ambient light levels, it
  might decide to tone-map the content, even if it's capable of displaying all
  luminance levels encoded.

- Displays tend to include configuration to alter the color and brightness. It
  will take these into account when deciding if/how to tone-map or gamut-map
  the content we gave it.

- Some gamut-mapping and tone-mapping will occur in the display depending on all
  the above variables, at which point it will emit some light.

Now, that's a very high-level overview of the process, but we can map these
high-level steps to portions of the Linux desktop and discuss what needs to
change in order for us to support HDR.

## HDR In the Linux Desktop

Now that we understand HDR - a larger luminance range that requires more bits
per component, a new transfer function to encode that luminance, and
potentially some metadata - we can examine the work required to use it in a
"standard" Linux desktop. As we'll see, it's already partially usable in
"non-standard" Linux environments.

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
  all the way from applications to the hardware transmitting to the display. This
  tells us how content was encoded.

- Somewhere we need to make the application's content encoding match the output
  encoding, even when there are multiple applications on the screen, even when
  they overlap, even when they are partially transparent and need to be blended
  together.

If you have ever worked with text content, you're probably familiar with text
encoding. Adding HDR support is very similar to adding support for Unicode when
previously everything assumed content was encoded with ASCII.

### The Kernel

Since the kernel deals with talking to hardware, it's the first stop in making
all this work. Fortunately, a good deal of work has already been done in the
kernel.

#### Display Capabilities

The first thing we need to be able to do is determine the capabilities of the
display we're preparing content for. We need to know:

- The exactly color of each primary on the display (chromaticity coordinates).

- The maximum luminance. Ideally this would include the peak luminance, what
  percentage of the screen can achieve peak luminance at once, the maximum
  luminance that can be sustained on the entire display, and so on.
  Unfortunately, at the moment much of this information might not be available.

- The minimum luminance.

- The types of HDR metadata it supports. These fall into two categories:
  static and dynamic. Static metadata applies to the entire piece of media,
  whereas dynamic metadata can change from scene to scene or even frame to
  frame. There are several metadata formats, so we need to know which formats
  we can use.

##### EDID

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

##### DisplayID

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

#### Content Encoding, Metadata, and Accelerated Operations

Graphics hardware typically provides hardware dedicated to efficiently decode,
transform, and encode images with look-up tables (LUTs). These look-up tables
approximate the smooth curve of the desired transfer function by mapping
encoded values to optical values (de-gamma LUT) and optical values to encoded
values (gamma LUT). Transformations could include color space conversions
(CSC), blending images together into a single image for the display, and so on.

The existence of these features and the number of look-up table entries is
hardware-dependent, but we would like to use them when they are there and
provide enough accuracy for the transfer function rather than calculating the
values using, say, expensive and precious CPU or general purpose graphics
shaders cycles.

##### Color Space Conversions

The kernel exposes a few standard connector properties for [color
management](https://www.kernel.org/doc/html/v5.11/gpu/drm-kms.html#color-management-properties)

These include the aforementioned look-up tables for transfer functions
(`GAMMA_LUT`, `DEGAMMA_LUT`) and the number of entries they support. We can use
these when they have a sufficient number of entries to handle the 10 or 12 bits
per component we need for the PQ or HLG transfer functions.

The `CTM` property lets userspace define a color transformation matrix to
describe how to map from the source color space to the destination color space.

##### Blending with Planes

It is possible to have hardware blend ("composite") images together using
[planes](https://www.kernel.org/doc/html/v5.11/gpu/drm-kms.html#plane-composition-properties).

However, the documentation does not mention using the GAMMA_LUT or DEGAMMA_LUT
to blend the content using linear, optical values so it is not clear what
impact this feature has on color correctness.

More work *may* be required here to use this feature when correct colors are a
concern.

##### Encoding


##### HDR Metadata

The kernel exposes a `HDR_OUTPUT_METADATA` [connector
property](https://www.kernel.org/doc/html/v5.11/gpu/drm-kms.html#standard-connector-properties)
that userspace can use to send HDR metadata to the kernel and thus to the display.

At this time there is only one [supported metadata
type](https://www.kernel.org/doc/html/v5.11/gpu/drm-uapi.html#c.hdr_output_metadata),
which corresponds to the static HDR metadata required for the HDR10 standard.

Other standards that use dynamic metadata rather than static metadata include
HDR10+ and Dolby Vision. More work is required here to support these standards.


### Wayland

Cover Wayland protocol extensions to include content encoding and metadata

### Mesa

Discuss the EGL/WSI layer that communicates from the graphics client to the
server (wayland) details about buffers (encoding, metadata)

### Client Frameworks

Tools on top of Mesa to create the content.

### Media Libraries

ffmpeg/gstreamer/etc that deals with encoded media.
