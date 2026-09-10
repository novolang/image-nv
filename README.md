# image-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

The front over novo-lang's image codecs. A decoded image is dimensions,
a colour model and a flat buffer of samples; `decode` dispatches on the
bytes' own magic and `encode` on a format the caller named together
with that format's options; and beside those sit the operations a
notebook or a thumbnailer needs — resize with a named filter, fit
inside a box, crop, rotate by a right angle, flip, composite, and
convert between colour models.

PNG and QOI come from [png-nv](https://github.com/novolang/png-nv) and
[qoi-nv](https://github.com/novolang/qoi-nv). JPEG is here.

```
novo pkg add image-nv
novo pkg build
novo test
```

## The one example that will work

```novo
use imagecodec
use imageops

fn thumbnail(file: Bytes) -> Result<Bytes, ImageError>
    let img = imagecodec.decode(file)!
    let small = imageops.fit_within(img, 256, 256, RfCatmullRom)!
    imagecodec.encode(small, imagecodec.default_encoding(FmtPng))
```

Any of the three formats in, a 256-pixel PNG out, the aspect ratio
kept, and the resampling done in linear light so the result is not
visibly darker than what went in.

## The image-nv / png-nv decision, and the argument for it

**image-nv DEPENDS on png-nv. It does not contain a second PNG codec,
and png-nv is not folded into it.**

The must-have plan gives png-nv "the format on its own" and image-nv
"the multi-format front", and the honest reading of those two rows is
a dependency. The precedent for the other answer — heapless-nv
absorbing the ringbuf-nv row — applies where two rows turn out to be
one thing. These are two things, and the test is that a sensible
program wants exactly one of them:

- **Duplicating would leave one codec and a copy.** Whichever were
  fixed first, the other would carry the bug, and a registry with two
  PNG decoders in it is a registry where a reader has to ask which one
  is maintained.
- **Absorbing would make the smaller package unavailable.** A program
  that only reads PNG — a favicon service, a plot renderer, a
  screenshot differ, an icon pipeline — should not acquire a JPEG
  decoder, an Adam7 composer and a Lanczos resampler in order to do it.
  png-nv is `core` with two dependencies; image-nv is `core` with
  three, one of which is png-nv. The grid exists so that a caller can
  take the smaller one.
- **The layers permit it.** Both are `core`, so `core` depending on
  `core` is inside the rule, and the audit's `dep-layer` row checks it
  on every run.

The same reasoning gives qoi-nv the same treatment, and for the same
reason: QOI is a one-page format that a program embedding a fast frame
dump wants on its own.

**JPEG is the asymmetry, and it is deliberate rather than tidy.** The
plan's row for image-nv names JPEG as this package's own
responsibility, so `jpegcodec.nv` declares it here rather than in a
`jpeg-nv` of its own. If the implementation lane finds that JPEG wants
a package — and it is the largest of the three by a long way — the move
is a new row, a dependency, and `ImgJpegFailed` becoming a wrapper like
`ImgPngFailed`. That is a one-variant diff, and nothing else in the
shape below changes.

## The layer, and why

`core` — no effects at all. The front sniffs a magic and hands the rest
to a codec; the operations are resampling kernels over a buffer the
caller already holds. Nothing is opened and nothing is waited for.

The one function that meets a stream stays inside the budget by
**binding** its cost rather than spending one:

```novo
pub fn read_all<S: Read[e]>(src: S) -> Result<Image, ImageError> [e]
```

`S: Read[e]` binds the effect parameter of the standard library's
`Read` trait and the clause uses it, so the row means *whatever the
impl behind `S` supplies*.

**No device claim.** There is no `tests/embedded_probe.nv` and the
audit's `core-embedded` row passes by saying so. This package is a
front over three codecs and a resampler; a microcontroller that wants
an image wants qoi-nv, and one that wants a colour wants color-nv.

## The load-bearing interface

```novo
pub struct Image
    width:   Int
    height:  Int
    kind:    PixelKind
    samples: Bytes

pub fn decode(src: Bytes) -> Result<Image, ImageError>
pub fn encode(img: Image, e: ImageEncoding) -> Result<Bytes, ImageError>
```

One struct and two functions, and everything else in the package
produces an `Image`, consumes one, or describes the enums those two
signatures name.

**The pixels are bytes, not colours.** A 4K RGBA image is 33 million
bytes; as a list of colour values — one heap cell and one reference
count per pixel — it is somewhere north of half a gigabyte, and every
operation on it walks pointers. So `Image` holds one `Bytes` and a
`PixelKind` that says how to read it, and `image.pixel` is the
accessor for a caller who wants a colour at a coordinate rather than a
buffer. This is also what makes the stack agree with itself: png-nv's
`PngEvRow` drains raw samples and qoi-nv's decoder drains raw samples,
and this is where they land with no conversion in between.

**`ImageFormat` and `ImageEncoding` are two types on purpose.** The
first is what `sniff` answers and carries no options, because a decoder
needs none. The second is what `encode` takes and pairs each format
with its own options in the variant, which makes an impossible
combination unrepresentable: there is no way to hand a JPEG quality to
a PNG encoder, because the constructor does not admit one.

## Four decisions worth arguing with

**Sniffing reads bytes and never a name.** A file name is a claim its
owner made; a magic number is a claim the encoder made. `decode`
sniffs, always. `imagefmt.of_extension` exists for the caller who has a
name and no bytes — choosing an output format from a path, filling in a
`Content-Type` — and its documentation says so, because a decoder that
trusted an extension is the classic upload vulnerability.

**`encode` refuses rather than converting.** Writing an RGBA image as
JPEG has to lose the alpha; writing a 16-bit image as QOI has to lose
eight bits a channel. Both are `Err(ImgFormatCannotCarry)` naming what
would have been lost, and `imagecodec.kind_for` says what to convert
to. A caller who meant to calls `imageops.convert` first — and the
conversion is then a line in their code that a reader can see, instead
of a silence inside a call that looked like it was only choosing a
container.

**Every resampling filter but `RfNearest` works on light.** Averaging
encoded sRGB averages the wrong numbers: halfway between black and
white is 188 encoded and 128 in light, which is why a photograph shrunk
in encoded space comes out visibly too dark and a thin bright line
disappears. Alpha is premultiplied before a resize and unpremultiplied
after, which is what stops the dark fringe around a resized cut-out.
Both happen without being asked, and this paragraph is the disclosure.

**Rotation is by right angles only.** 90, 180 and 270 move every pixel
to another pixel exactly — no resampling, no loss, no `Result`. A
rotation by any other angle is a resampling operation with its own
filter, its own edge policy and its own output size, and it belongs
with a general affine transform rather than beside `flip`.

## The reference implementations, and what is specification

Rust's `image` crate and Python's Pillow are the reference
implementations for the front and the operations. **libjpeg-turbo is
the reference implementation for JPEG and its corpus is the oracle**,
which is the plan's own instruction for this row.

**Specification, and binding on this package**

- PNG's and QOI's formats, which are png-nv's and qoi-nv's problem and
  are documented there.
- JPEG: ITU T.81's baseline and progressive processes, the marker
  structure, the DCT and the quantisation and Huffman stages, and the
  YCbCr transform.
- ITU T.83's conformance bound on the inverse DCT.

**Choices this package makes, which a test may not treat as
correctness**

- **The quality number.** A "quality 85" JPEG is one whose
  quantisation tables were scaled by libjpeg's formula from libjpeg's
  own baseline tables. Photoshop's 85, MozJPEG's 85 and a phone's 85
  are all different files. This package follows libjpeg's scaling
  because that is what every tool that prints a number means.
- **4:2:0 as the default subsampling.** It is what almost every
  encoder defaults to, it is right for a photograph, and it is wrong
  for a screenshot with coloured text — which is why `Chroma444` is
  one field away.
- The four resampling filters and their coefficients: Catmull-Rom is
  Mitchell with B = 0 and C = 0.5, Lanczos is windowed at three lobes.
  Both are conventions with no standard behind them.
- The eight `PixelKind`s. There is no palette kind — an indexed image
  is a compression of an RGBA one, png-nv resolves the palette, and a
  kind that could be indexed would put a branch in every operation in
  `imageops` for a representation only one format has.

**JPEG's output is not exact, and the corpus test has to say so.** Two
conforming JPEG decoders may disagree by a level or two per channel,
because the inverse DCT is specified as a mathematical transform and
every implementation approximates it — libjpeg-turbo alone ships three.
So the measurement against the oracle is *within T.83's bound of
libjpeg-turbo*, never *the same bytes as libjpeg-turbo*, and a test
that asserted equality would fail on a correct implementation.

**Deliberately not ported:** GIF, WebP, TIFF, BMP and AVIF — each is a
package's worth of work, none is on the grid yet, and `ImageFormat` is
where they would arrive without changing the shape of `decode`. CMYK
JPEGs are refused rather than converted, because the conversion needs
an ICC profile this package cannot read and the naive one is visibly
wrong. Animation of any kind: `Image` is one frame, and a package that
returned several would be a different interface.

## Status

Every function is `todo()`. `novo test` runs the API suite, and every
assertion in it reaches `not implemented: image-nv.<module>.<fn>` —
which is the expected result until the bodies land, and is what makes
the suite a description of the interface rather than of nothing.
`novo test --isolate tests/<file>` is the readable form: one verdict
per test, naming the function it stopped at.

| module | public types | functions | implemented |
| --- | --- | --- | --- |
| `image` | 2 | 15 | no |
| `imagefmt` | 1 | 8 | no |
| `imagecodec` | 1 | 12 | no |
| `imageops` | 3 | 11 | no |
| `jpegcodec` | 4 | 9 | no |
| `imageerror` | 1 | 3 | no |
| **total** | **12** | **58** | **no** |
