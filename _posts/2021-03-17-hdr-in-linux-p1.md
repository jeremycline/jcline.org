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

# Background

Before attempting to understand dynamic range, I found it helpful to understand
the go through the basics of electromagnetic radiation (light) and how the
human visual system (eyes) interact with light so that we can produce graphics
for human consumption.  With an understanding of how to do that, we can
complete our ultimate goal of tricking the human visual system into seeing
extremely realistic cats even if there are no cats around.

## Light

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

## Luminance

[Luminance](https://en.wikipedia.org/wiki/Luminance) is the measure of the
amount of visible light that passes through an area.  The SI unit is candela
per square meter (cd/m²). This is often also referred to as a "nit", presumably
because cd/m² is somewhat laborious to type out.

The human eye can detect a luminance range from about 0.000001 cd/m² to
around 100,000,000 cd/m². We can readily go out into the world and [experience
this entire range,
too](https://en.wikipedia.org/wiki/Orders_of_magnitude_(luminance)). For
example, the luminance of the sun's disk at noon is around 1,600,000,000 cd/m².
This is much higher than the human eye can safely experience; do not go stare
at the sun.

You might suppose that if you show a human two lights, one with a luminance of
10 nits and a second with a luminance of 20 nits, the 20 nit light would look
twice as bright. However, the relationship between luminance and the human
perception of brightness is not linear. Rather, it's is roughly logarithmic
(you're probably familiar with
[decibels](https://en.wikipedia.org/wiki/Decibel) with respect to sound).

The fact that human sensitivity to luminance is non-linear becomes very
important later when we need to compress data.

## Human Visual System

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

While the human visual system is capable of detecting a very large range of
luminance levels, it cannot do so all at once. If you're outside on a sunny day
and walk into a dark room, it takes some time for your eyes to adjust to the
new luminance levels.

## Displays

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
process is often referred to as "tone mapping".

## Dynamic Range

Dynamic range is the ratio between the smallest value and largest value. When
used in reference to displays, the value is luminance.

The lowest luminance level is determined by how much ambient light is
reflecting off the surface of the display (e.g. are you sitting in a dark room,
outside on a sunny day, etc) and how much of the backlight (if there is one)
leaks through the display panel.

The highest luminance level is determined by the light source of the display.
This depends on the type of display, power usage requirements, user
preferences, and on the ambient temperature (emitting visible light also emits
heat, and heat can damage things).

Both these values depend on environmental factors so the dynamic range of a
display can change from moment to moment, which is something we may wish to
account for when compressing the luminance range of the world to fit the
particular display.

So what is a high dynamic range display? In short, a display that is
simultaneously capable of lower luminance levels and higher luminance levels
than previous "standard dynamic range" displays. What these levels are exactly
can be hand-wavy, but [VESA provides some
specifications](https://displayhdr.org/performance-criteria/).

This higher dynamic range ultimately means the way in which we compress the
luminance range (tone map) needs to change. The reason for this requires us to
understand the current approach so that we can understand its shortcomings and
what needs to change. We will cover this in the next post, after which we will
discuss what needs to change in Wayland compositors to allow for high dynamic
range content.

# From Creation to Display

In this section, we'll cover the journey of an image from its creation to its
display. The most self-contained examples are modern video games or films
animated with CGI. This allows us to dodge the added complication of camera
sensors and so on, but the process for real-world image capture is similar.

One note on color here. Converting from one color space to another is a
reasonably straight-forward process in many cases, but it requires that you
know what color space your content is in. Otherwise when the image is encoded
to say "this pixel needs X red and Y green", there's no way to know what
"red" and "green" you were working with. Thus, we need to keep track of what
the color space is along with the image data itself.

## Scene-referred Lighting

Animated films and video games typically model a world complete with realistic
lighting.  When they do this, they need to work with "real world" luminance
levels. For example, the sun that lights the scene may be modeled to be seen as
1.6 billion nits.  Working with real world luminance that cannot be directly
displayed is often called "scene-referred" luminance. We have to transform the
scene-referred luminance levels to something we can display
("display-referred") by compressing or shifting the values to a displayable
range.

This is one of the last steps in the image creation process since compressing
the range necessitates discarding information; you might map 1,000,000 nits to
the same value as 999,999 nits and if you wanted to reverse the operation
there's no way to undo that.

You might think it's best to leave content in scene-referred lighting levels
until the last possible moment (when the display is setting the value for each
pixel), and that would indeed make some things simpler. However, it does make
some things more difficult. Consider, for example, blending the scene-referred
image with content that is in the display's luminance range. You would need to
tone map the display-referred content into the scene-referred content's
luminance range, only to have it eventually be tone mapped back down to the
display level.

There's a more difficult problem with keeping all content scene-referred,
though. Representing numbers accurately in the 0.000001-1,600,000,000 cd/m²
range requires a lot of bits, and we're about to have a serious bit budget
imposed on us. We have to transport the image data from the host graphics
processing unit to the display. While both HDMI and DisplayPort have enormous
bandwidth, the image resolution is also quite large. If we used a 32 bit
floating point number for each red, green, and blue (RGB) component of a pixel
on a display with 3840 x 2160 (4K) pixels, it would require 32 x 3 x 3840 x
2160 bits per frame, or 760 Mbits per frame. At a modest 60 frames per second,
that's 44.5 Gbits. DisplayPort 1.4 provides 25.9 Gbits and HDMI 2.0 give us
14.4 Gbits.

Fortunately for us, as long as we ensure the way we encode luminance levels
matches up with the way the human eye detects luminance levels, we can save a
*lot* of bits. In fact, for the "standard" dynamic range, we can manage with
only 8 bits (256 luminance levels) per color (24 bits total for RGB).

## Tone Mapping

In order to convert scene-referred, real world luminance to a range for our
target display, we need to:

1. Have an algorithm for mapping one value to another. Input values have a
   range [0,∞) and the output values need to cover the range of our target
   display.

2. Know the capability of our target display.

We'll assume for now that #2 is easy.

As for the algorithm to use, there are many different approaches and results
can be subjective.  This is not a post about tone mapping (there are many
multi-hundred page publications on the subject), we'll just go over a couple
examples to get the idea.

An extremely silly, but technically valid tone mapping function would be `f(x)
= 1`. This maps any input luminance level to 1.

A more useful approach might be the function `f(x) = x^n/(x^n + s^n)` where `x`
is the input luminance, `n` is a parameter changes the slope of the mapping
curve, and `s` shifts the curve on the horizontal axis. For example, with `n =
3` and `s = 10`:

![Example tone mapping curve that maps all input luminances between 0 and 1](/assets/example-tone-map.png)

This [sigmoid function](https://en.wikipedia.org/wiki/Sigmoid_function) gently
approaches the minimum and maximum values. We can then map the [0, 1] range to
the display luminance range.

There are many articles and papers out there dedicated to tone mapping. There's
a chapter on the topic in "The high dynamic range imaging pipeline" by Gabriel
Eilertsen, for example, which include a number of sample images and a much more
thorough examination of the process than this blog post.

## Encoding

Once we've tone-mapped our content to use the display's luminance range, we
need to send the content off to the display so that we can see it. As the
section on scene-referred light mentioned, we have a limited amount of
bandwidth, so we need to be clever about how we use it. This is where many
of the complications with HDR compared with SDR occur.

### SDR

The "standard dynamic range" is not particularly standard, but usually tops out
around 300 nits. Usually, 8 bits per RGB component is used. 8 bits allow us to
express 256 luminance levels.

One approach is to evenly spread each level across the luminance range. However,
because of the non-linear nature of the human eye, doing so makes the steps
between low luminance levels obvious as clear bands in the image. The
difference between 1 nit and 2 nits is much more obvious than between 249 and
250.

Rather than evenly spreading them across the range, we more values at low
luminance levels than at high levels, and we want each level to be just barely
indistinguishable to the human eye from the level on either side of it. This
way gradients appear smooth rather than a series of color bands. This
non-linear encoding is often just called "gamma".
