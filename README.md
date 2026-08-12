cairo
=====

Cairo for sysl — 2D vector graphics: paths, filling, stroking, dashing, gradients, clipping,
transforms and text, rendered into memory or straight into a PDF, an SVG or a PostScript file.

```
brew install cairo                 # macOS
sudo apt install libcairo2-dev     # Debian / Ubuntu
```

```hocon
dependencies {
  cairo { git = "github.com/sysl-lang/cairo", version = "0.2.0" }
}
```

```sysl
import sh.sysl.cairo.*
import sysl.math.pi

main()
    var s = image_surface(Format.Argb32, 200, 200)
    var cr = context(s)

    cr.set_source_rgb(0.1, 0.2, 0.9)
    cr.arc(100.0, 100.0, 60.0, 0.0, 2.0 * pi)
    cr.fill()

    s.write_png("circle.png")
```

The library has to be found at link time, which on a Homebrew machine is one flag:

```
sysl run . --link-path /opt/homebrew/lib
```

There is **no `--include-path`**, and that is worth noticing: this package compiles no C of its own,
so no header is ever read. Only the linker needs to be told where cairo is.

The same drawing goes to a page
-------------------------------

Cairo's central idea is that the backend is chosen once and the drawing code does not know which one
it got. Only the first line changes:

```sysl
var s = pdf_surface("out.pdf", 612.0, 792.0)     -- points: US Letter
var s = svg_surface("out.svg", 200.0, 200.0)
var s = ps_surface("out.ps", 595.0, 842.0)       -- A4
```

A **recording** surface is the fourth kind: it keeps the operations rather than pixels, so one
drawing is replayed into a thumbnail and a poster without being described twice.

PDF output is not only a page of marks. `pdf_set_metadata`, `pdf_add_outline`, `pdf_set_page_label`
and `Context.tag_begin` build a *tagged* PDF — document structure for accessibility, a bookmark
sidebar, named destinations and live hyperlinks:

```sysl
cr.tag_begin(TAG_LINK, "uri='https://sysl.sh'")
cr.show_text("sysl.sh")
cr.tag_end(TAG_LINK)
```

Errors live on the object
-------------------------

This is cairo's most distinctive convention, and the binding keeps it rather than translating it
into something more familiar. Almost nothing answers a status: a call that fails **records** the
failure on the context or the surface, and every later call on that object does nothing. So a
program draws a whole page without checking anything and asks once at the end:

```sysl
if !cr.status().ok()
    print(cr.status())          -- "cairo_restore() without matching cairo_save()"
```

What makes that safe rather than sloppy is that the failure is **sticky**: it cannot be overwritten
by a later success, so the single check at the end sees the first thing that went wrong. There is a
test asserting exactly that.

**A constructor never answers null** for the same reason — cairo hands back a static "nil" object
whose `status()` says why — so there is no `Option` here. Ask `ok()` if you want to know.

Nothing is closed by hand
-------------------------

`Surface`, `Context`, `Pattern`, `FontOptions`, `FontFace` and `ScaledFont` each hold a C allocation,
each is reached through `&T`, and each has an `impl Drop` that releases it when the last reference
goes. **There is no `destroy` in this API**, and no `reference` either: cairo's objects are reference
counted and so is `&T`, so the two agree. A surface handed to a pattern outlives the variable it was
made in because the pattern took a count of its own.

A handle cairo *lends* rather than gives — `Context.target`, `.source`, `.font_face`, `.scaled_font`
— is referenced before it is wrapped, so what comes back owns a share like anything else. The
previous shape of this package could only warn about those in a comment, and getting it wrong was a
use-after-free.

Every enumeration is an enum
----------------------------

`set_operator` takes an `Operator`, `image_surface` takes a `Format`, `status()` answers a `Status`.
There is no way to pass a `Filter` where a `LineCap` belongs, or a bare `3`:

```sysl
cr.set_line_cap(LineCap.Round)
cr.set_operator(Operator.Multiply)

if cr.line_cap() == LineCap.Round
    print(cr.antialias())            -- "subpixel", not "3"
```

Each carries `code()` for the number C wants and `of(code)` for the number C gave, and each has an
`Other(code: int)` arm — so a cairo newer than this binding reporting something it has not heard of
keeps a program running and printing rather than stopping at a `match` with no arm for it.

The 138 values behind them were **checked against `cairo.h` by compiling them**, and they agree.
They are still transcribed rather than asked of the C compiler with a `c const` block: doing that
would make this package compile C for the first time, which turns "install libcairo" into "install
the development headers and pass an include path". That is a trade to make deliberately, and it is
not made here — which is why the `--include-path` note above still holds.

What is here
------------

| area | what it covers |
|---|---|
| **Surfaces** | image, PDF, SVG, PostScript, recording; similar surfaces, sub-surfaces, device offset and scale |
| **Pixels** | direct access to an image surface's bytes, and `image_surface_for_data` over a buffer the caller owns |
| **PNG** | to and from a file, and to and from **memory** — the stream forms, so nothing has to touch the filesystem |
| **Paths** | lines, relative moves, cubic Béziers, arcs, rectangles, current point, extents |
| **Painting** | fill, stroke, paint, mask, the whole Porter-Duff and blend-mode operator set, groups |
| **Clipping** | clip, clip extents, and hit testing with `in_fill`, `in_stroke`, `in_clip` |
| **Transforms** | translate, scale, rotate, an explicit `Matrix`, and conversion between user and device space |
| **Patterns** | solid, surface, linear and radial gradients; extend and filter modes; a matrix of their own |
| **Text** | the toy interface, text and font extents, text as a path, positioned glyphs, font options, scaled fonts |
| **Tagged output** | PDF metadata, outlines, page labels, structure tags, links and destinations |

**What is not**: font faces built from a *specific* backend. `cairo_ft_font_face_create_for_ft_face`
needs FreeType and the Quartz one needs Core Text — a second library on the link line, and therefore
a separate package, by the same rule that made SDL3 four of them. Also absent are `cairo_copy_path`
(cairo's path data is a C union, which sysl has no spelling for), the mesh and raster-source pattern
kinds, devices, regions and the X11 backends.

Against `plutovg`
-----------------

The org has [another 2D vector graphics package](https://github.com/sysl-lang/plutovg), and neither
replaces the other. Cairo has the vector backends — PDF, SVG, PostScript — and is already installed
on most machines. PlutoVG carries its own C, asks for no system library and needs no allocator it
cannot be given, so it runs on a microcontroller, which cairo does not. The division is about where
the program runs rather than about which has more features.

Tests
-----

Thirty, and every one of them **draws for real and reads the result back** — a graphics binding that
only checks return codes is checking that C was called rather than that the right thing happened.
The image surface makes that cheap: a fill is asserted by looking at a pixel, a gradient by looking
at both ends of one, a clip by finding paint on one side of it and none on the other.

```
sysl test . --link-path /opt/homebrew/lib
```

There is no shim
----------------

Not one public function in cairo takes or returns a struct by value — checked by walking every
`cairo_public` declaration in the header rather than by reading a few of them. The only non-scalar
parameters in the whole library are enums, which are `int`s, and function pointers. So the entire
library is reachable by declaring it, and the `UsefulBufC` problem that reshaped
[qcbor](https://github.com/sysl-lang/qcbor)'s whole API does not arise.

The structs cairo does have — a matrix, the two kinds of extents, a glyph — are always written
through a pointer into storage the *caller* owns, so a sysl struct with the same fields is handed
over with `ptr_cast` and cairo writes back through it. `sh.sysl.cairo.externs` is the verbatim C
surface and `sh.sysl.cairo` is what a program imports.

License
-------

[ISC](LICENSE)
