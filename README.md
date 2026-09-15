# image-nv

A raster image is a rectangle of pixels. An image file stores that
rectangle compressed, in one of several formats. This package is the
front over those formats for novo-lang: it recognises a file from its
own first bytes, decodes it into one in-memory shape, offers the
operations a thumbnailer or a notebook performs on it, and writes it
back out. Its references are the Rust crate
[image](https://docs.rs/image) and Python's
[Pillow](https://pillow.readthedocs.io/) for the interface, and
[libjpeg-turbo](https://libjpeg-turbo.org/) for JPEG.

PNG comes from [png-nv](https://novo-lang.org/packages/png-nv) and QOI
from [qoi-nv](https://novo-lang.org/packages/qoi-nv). JPEG is here.
Colours are [color-nv](https://novo-lang.org/packages/color-nv)'s.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What an image is here

An `Image` is four things: a width, a height, a **pixel kind**, and a
flat buffer of bytes. Rows run one after another with no padding, so the
**stride**, the number of bytes in one row, is exactly the width times
the bytes in one pixel.

A **pixel kind** says how to read the buffer: how many **channels** a
pixel has and how many bits each one holds. A channel is one number per
pixel, such as the red one. **Alpha** is a channel saying how opaque the
pixel is.

| Kind | Channels | Bits per channel |
| --- | --- | --- |
| `PixLuma8` | grey | 8 |
| `PixLumaAlpha8` | grey, alpha | 8 |
| `PixRgb8` | red, green, blue | 8 |
| `PixRgba8` | red, green, blue, alpha | 8 |
| `PixLuma16` | grey | 16, big-endian |
| `PixLumaAlpha16` | grey, alpha | 16, big-endian |
| `PixRgb16` | red, green, blue | 16, big-endian |
| `PixRgba16` | red, green, blue, alpha | 16, big-endian |

A **format** is a file format. Three are supported.

| Format | Lossless | Alpha | Codec |
| --- | --- | --- | --- |
| `FmtPng` | yes | yes | png-nv |
| `FmtJpeg` | no | no | this package's `jpegcodec` |
| `FmtQoi` | yes | yes | qoi-nv |

**Sniffing** is recognising a format from the first bytes of a file,
which every one of the three begins with a distinctive pattern called a
**magic number**.

**Resampling** is computing the pixels of a resized image from the
pixels of the original. A **filter** decides how: which source pixels
contribute to a destination pixel, and with what weights.

An 8-bit channel does not hold an amount of light. It holds an **sRGB**
value, which is light after a curve that gives dark tones more of the
range. Averaging those encoded numbers averages the wrong quantity:
half-way between black and white is 188 encoded, not 128. Averaging
in **linear light** means undoing the curve first and reapplying it
afterwards.

**Premultiplied alpha** means each colour channel has already been
multiplied by the alpha. Resampling a cut-out without premultiplying
blends the colour of fully transparent pixels into the edge, which shows
as a dark fringe.

## Install

```
novo pkg add image-nv
```

## Example

```novo
use std.bytes
use image
use imageops
use imagecodec

fn main() [io]
    // A four-by-four image, four 8-bit channels per pixel, all zero.
    let img = image.new(4, 4, PixRgba8)
    println("${image.byte_size(img)} bytes, ${image.stride(img)} per row")

    // Scale it to fit inside a two-by-two box, keeping the aspect ratio.
    // Catmull-Rom averages, so the resampling happens in linear light.
    match imageops.fit_within(img, 2, 2, RfCatmullRom)
        Err(e) => println(e.message())
        Ok(small) =>
            // QOI has no encoder options at all, and it carries alpha,
            // so this encode cannot lose a channel.
            match imagecodec.encode(small, EncQoi)
                Err(e)   => println(e.message())
                Ok(file) => println("${bytes.len(file)} bytes of QOI")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: image-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `image` | The `Image` struct and the eight pixel kinds, the arithmetic over a kind, the constructors, and reading and writing one pixel or one row. |
| `imagefmt` | The three formats: their names, their media types, their file extensions, the sniff, and what each one can carry. |
| `imagecodec` | Decoding, with and without a named format or a wanted pixel kind and with a size cap, reading the header alone, encoding with per-format options, and reading a whole image from a source. |
| `imageops` | Resize with a named filter, fit inside a box, crop, rotate by a right angle, flip, convert between pixel kinds, and composite one image onto another. |
| `jpegcodec` | JPEG: the two processes, the chroma subsampling choices, the encoder options, the header information, and decode and encode. |
| `imageerror` | Every reason a decode, an encode or an operation refuses, as one enum with ten variants, and three questions to ask of one. |

## How to choose an entry point

**`imagecodec.decode` is the usual way in.** It reads the magic number
and dispatches. `decode_as` is for a caller who already knows the
format, and `decode_as_kind` asks the codec for a particular pixel kind
so that a conversion pass is not needed afterwards.

**`imagecodec.read_size` reads only the header.** Take it when you want
the dimensions of a thousand files and the pixels of none.

**`imagecodec.decode_bounded` caps the pixel count before allocating.**
Take it for anything that arrives from outside the program. A small
compressed file can declare an enormous image.

**`imagecodec.read_all` reads from a source rather than a buffer.** It
is the only function here that declares an effect, and the effect is
whatever the source brings: reading a file costs what a file costs, and
reading from memory costs nothing.

**Take png-nv or qoi-nv directly when you need one format.** A program
that only reads PNG does not need a JPEG decoder and a Lanczos
resampler to do it.

## The rules a user needs

1. **A format is decided by bytes, never by a file name.** A name is a
   claim its owner made and a magic number is a claim the encoder made.
   `imagefmt.of_extension` exists for a caller choosing an output format
   from a path or filling in a media type, and for nothing else.
2. **`encode` refuses rather than dropping a channel.** Writing an image
   with alpha as JPEG would lose the alpha, and writing a 16-bit image
   as QOI would lose eight bits per channel. Both answer
   `ImgFormatCannotCarry`, naming what would be lost.
   `imagecodec.can_carry` asks in advance and `imagecodec.kind_for` says
   what to convert to. Call `imageops.convert` first, so that the loss
   is a line a reader can see.
3. **Every filter but `RfNearest` averages in linear light**, and
   premultiplies alpha before the resize and undoes it after. Neither
   has to be asked for.

   | Filter | Use it for |
   | --- | --- |
   | `RfNearest` | pixel art and a nearest-neighbour zoom; it invents no colours |
   | `RfBox` | a downscale by an exact integer ratio, where it is also the best |
   | `RfCatmullRom` | a photograph; the general answer |
   | `RfLanczos3` | the sharpest result, at the cost of ringing on a hard edge |

4. **Rotation is by right angles only.** Ninety, 180 and 270 degrees
   move every pixel to another pixel exactly, so `imageops.rotate`
   cannot fail and loses nothing. A rotation by any other angle is a
   resampling operation with its own filter, its own edge policy and its
   own output size.
5. **A JPEG quality number follows libjpeg's scaling.** Quality 85 means
   libjpeg's baseline quantisation tables scaled by libjpeg's formula.
   Photoshop's 85 and a phone camera's 85 are different files.
   `jpegcodec.at_quality` refuses a number outside the range.
6. **JPEG defaults to 4:2:0 chroma subsampling**, which stores one
   colour sample for every four pixels. It is right for a photograph and
   wrong for a screenshot with coloured text, where `Chroma444` is one
   field away.
7. **Two conforming JPEG decoders may disagree by a level or two per
   channel.** ITU T.81 specifies the inverse discrete cosine transform
   as a mathematical transform and every implementation approximates it.
   ITU T.83 sets the bound they must stay inside. A comparison against
   another decoder is a comparison within that bound, never byte for
   byte.
8. **The buffer holds bytes, not colour values.** A 4K image with four
   channels is 33 million bytes. As a list of colour values, with a heap
   cell each, it would be several hundred megabytes and every operation
   would walk pointers. `image.pixel` is the accessor for a caller who
   wants one colour at one coordinate.
9. **There is no padding between rows.** The stride is the width times
   the bytes per pixel, so a caller who does not care about rows may
   treat the whole buffer as one run.
10. **An image has at least one pixel in each direction.** `image.new`
    refuses a zero width or height.
11. **There is no indexed pixel kind.** An indexed image is a compressed
    form of one with full colour, and png-nv resolves the palette while
    decoding.

## What is not included

- **GIF, WebP, TIFF, BMP and AVIF.** Each is a package's worth of work.
  `ImageFormat` is where one would arrive without changing the shape of
  `decode`.
- **CMYK JPEGs.** Converting them needs a colour profile this package
  cannot read, and the conversion that ignores the profile is visibly
  wrong. They are refused.
- **Animation.** An `Image` is one frame.
- **Rotation by an arbitrary angle, and a general affine transform.**
  See rule 4.
- **A microcontroller build.** This package is a front over three codecs
  and a resampler. A device that wants an image wants qoi-nv, and one
  that wants a colour wants color-nv. This package makes no device claim
  and ships no device probe.

## Related packages

- [png-nv](https://novo-lang.org/packages/png-nv) is the PNG format on
  its own, and this package depends on it rather than carrying a second
  copy. Take it directly when PNG is the only format you read.
- [qoi-nv](https://novo-lang.org/packages/qoi-nv) is the QOI format on
  its own. QOI is one page of specification and it builds for a
  microcontroller, which this package does not.
- [color-nv](https://novo-lang.org/packages/color-nv) owns the colour
  value `image.pixel` answers and the conversions between colour spaces.
- [flate-nv](https://novo-lang.org/packages/flate-nv) is the compression
  underneath PNG, and arrives through png-nv.
- [plot-nv](https://novo-lang.org/packages/plot-nv) fills one of these
  images when a chart is drawn to a raster rather than to a document.
- `std.net` and the standard library's file reading are where the bytes
  come from. `imagecodec.read_all` binds whatever effect the source
  brings.

## Tests

```bash
novo test tests/image_tests.nv         # 14 tests: the struct, the kinds and the arithmetic
novo test tests/imagecodec_tests.nv    # 28 tests: sniffing, decoding, encoding and the refusals
novo test tests/imageops_tests.nv      # 17 tests: the operations and the filters
```

The Rust `image` crate and Pillow are the references for the front and
the operations. libjpeg-turbo's corpus is the oracle for JPEG, measured
within the bound of rule 7 rather than byte for byte. PNG's and QOI's
own conformance belongs to png-nv and qoi-nv and is tested there.

The suite asserts that a sniff reads bytes and not a name, that an
encode that would lose a channel refuses and names what it would lose,
that the stride has no padding in it, that a rotation by a right angle
is exact, that a zero dimension is refused, and that a bounded decode
refuses an image larger than its cap before allocating.

The tests compile today and fail at run, each on the
`not implemented: image-nv.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `image.Image`, `.PixelKind`, `imagefmt.ImageFormat`, `imagecodec.ImageEncoding` | declared |
| `imageops.ResizeFilter`, `.Rotation`, `.FlipAxis` | declared |
| `jpegcodec.JpegMode`, `.ChromaSubsampling`, `.JpegOptions`, `.JpegInfo` | declared |
| `imageerror.ImageError` | declared |
| `image.channel_count`, `.bytes_per_sample`, `.bytes_per_pixel`, `.has_alpha`, `.is_grey`, `.with_alpha_channel` | no |
| `image.stride`, `.byte_size`, `.new`, `.filled`, `.from_samples` | no |
| `image.pixel`, `.with_pixel`, `.row`, `.with_row` | no |
| `imagefmt.format_name`, `.mime_type`, `.extensions`, `.magic_length` | no |
| `imagefmt.sniff`, `.of_extension`, `.supports_alpha`, `.is_lossless` | no |
| `imagecodec.encoding_format`, `.encoding_name`, `.default_encoding` | no |
| `imagecodec.decode`, `.decode_as`, `.decode_as_kind`, `.decode_bounded`, `.read_size`, `.read_all` | no |
| `imagecodec.encode`, `.can_carry`, `.kind_for` | no |
| `imageops.filter_name`, `.filter_averages`, `.rotation_degrees`, `.axis_name` | no |
| `imageops.resize`, `.fit_within`, `.crop`, `.rotate`, `.flip`, `.convert`, `.overlay` | no |
| `jpegcodec.mode_name`, `.chroma_name`, `.chroma_ratio`, `.default_options`, `.at_quality` | no |
| `jpegcodec.is_jpeg`, `.read_info`, `.decode`, `.encode` | no |
| `imageerror`'s ten variants, `.offset_of`, `.is_format_failure`, `.needs_more_bytes` and its `Error` implementation | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
