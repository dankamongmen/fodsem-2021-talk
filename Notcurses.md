# Notcurses, 21st century tech for 20th century UIs

for FOSDEM 2021

## Introductory material

### Who are you? What is this?

I'm Nick Black, the lead author of Notcurses. I'm primarily an HPC and
compilers guy, but I did a lot of demoscene code as a teenager, writing
x86 assembly on my trusty 8088. I'd written a few large NCURSES programs,
and I thought X/Open Curses could be improved on, so I did.

Notcurses is a library for blingful character graphics and text user
interfaces. It's a clear intellectual descendant of Curses, but it does
not attempt compatibility with that API. I hope to demonstrate why I
think it could be the de facto TUI/CG library for new applications.

The first commit was made November 2019; a little over 4000 commits
later, Notcurses is in Fedora, Debian, Arch, Alpine, Void, FreeBSD,
and DragonFly, and there's a PR outstanding to bring it to OS X.
Since both myself and the author of the C++ wrappers are employed
by Microsoft, we'll hopefully get it there soon, too.

### Character graphics

Text mode is primarily defined by two things:

* *Text mode*: Drawable unit is a glyph rather than a pixel
* *Stream I/O*: We read and write streams rather than e.g. a framebuffer

Can be a Linux or FreeBSD virtual console, a terminal emulator, a pseudotty
(SSH, a terminal multiplexor, or even more exotic options).

The glyphs we can draw are a function of our *encoding*, our *font*, and
our *terminal*.

"Smart terminals" support *control sequences* amidst the glyph stream.
libterminfo abstracts most of these control sequences. The Curses API
(usually implemented by NCURSES) provides TUI abstraction atop terminfo.

### Why not Curses (NCURSES)?

The Curses API originated in the 70s, over forty years ago -- it's one of
very few APIs older than I am, and though it's very widespread -- ubiquitous,
really -- it's not in my opinion a very good API (though NCURSES is a very
good implementation).

 * Limited Unicode support
 * PseudoColor and weird colorpair system baked into the API
 * Control via global variables
 * Limited threading support
 * Identifier naming all over the place
 * Basic functionality available only as extensions

### Why not some other library?

Several dozen TUI/character graphics libraries have been developed, some
of them claiming Curses compatibility, some not, and in any number of
languages.

[ show list ]

I deemed them all unacceptable for one reason or another.

### Design goals

* Written in C
** C is the base language of UNIX and its system calls
** We can wrap it with just about any other language
** Very real performance results

* Two modes:
** "Direct mode" works with standard I/Ogood for batch or line-driven programs
** "Rendered mode" renders fullscreen frames, then blits them

* Multithreading support. This means:
** Specifying which functions can be called in parallel
** Designing to maximize the area of safe parallel intersections

* General surface support
** Surfaces can be any size, and are free to be partially/totally offscreen
** Z-axis (total ordering) within each pile
** Binding (directed acyclic forest) within each pile with resize cascades
** Three independent channels -- glyph, foreground color, background color

* TrueColor, color blending, and default colors 
** Default colors allow transparency to the desktop
** More generally, default colors allow a degree of user configuration
** Palette-indexed PseudoColor to minimize bandwidth

* Multimedia support
** Sits atop FFmpeg or OpenImageIO
** State-of-the-art quad- and sex-blitters

* Widgets
** Progress bar, selector, multiselector, reels, input box, menus, plots
** libreadline in direct mode
** Several types of boxes, polyfill, rotations

* Perf domination
** O(1) translation, z-axis move, reparenting, destruction
** Optimal rendering and rasterization based off painter's algorithm / damage maps
** (Coming soon) automatic palette generation
** Extensive profiling, performance tracking as part of CI, `notcurses-demo` provides 25 benchmarks

## A Tour of Notcurses

### Direct mode

Direct mode is the best option for "command-line UI": scrolling line-based I/O.

"Style your printf()s": use regular stdio, plus `ncdirect_*()` functions to
emit control sequences.

The cursor can be moved, and screen coordinates can be determined. This is
useful for e.g. spinners and multiline widgets. Direct mode provides boxes,
progress bars, and image rendering.

[ show ncls ]

For more complex graphics, including forms, plots, and just about anything
where we need be rapidly updating the full screen, we use rendered mode.

### Rendered mode

*Piles* made up of *planes* are *rendered* into *frames*.
** Piles are independent -- multiple threads can mutate multiple piles
** The painter's algorithm is used for rendering.

A frame is then *rasterized* to the terminal, bringing the visual area
into synchronization with the rendered frame.
** Damage maps are used to optimize rasterization.

Mixing standard I/O with rendered mode will lead to madness, and must
be avoided.

Can use the *alternate screen*: goes away on exit, no scrollback.

### A Notcurses context

Comes with the *standard pile*, consisting of the *standard plane*.
The standard plane always has the geometry of the visual area, and
cannot be destroyed, reparented, or moved. Since the standard plane
cannot be destroyed, the standard pile cannot be destroyed.

At all times, the context has a single "last rendered frame". This
is used both to compute each frame's damage map during rasterization,
and to redraw the screen in the case of external damage.

#### Piles

One or more piles at a time.

* Visual area (geometry can change at any time)
* Composed of one or more *piles*
** User don't work with piles explicitly
* Each is composes of one or more *planes*
** If a pile loses all its planes, it is destroyed
* Each pile has a z-axis, a total ordering of its plane
* Each pile has a binding forest
** Subtree-wide moves, reparentings, destroys
* The pile cannot be mutated while it is being rendered
* Only one thread may reorder planes within a pile at a time
* Multiple threads may mutate distinct piles

#### Planes

One or more planes per pile. Planes have (beyond pile bookkeeping):

* A geometry and an origin (relative to the visual area)
* An active background color, foreground color, and style
* A framebuffer of *cells* and a backing *egcpool*
* A user-managed opaque pointer and name
* A resize callback function
* A virtual cursor location
* A base cell, used where cells are undefined

#### Cells (and EGCPools)

A cell is a 16-byte structure, with possible spillover into the egcpool.

```
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |       gcluster (up to 4 UTF-8 bytes, OR a 24-bit index)       |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |    backstop   |     width     |           stylemask           |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |    foreground                                                 |
  +                            channels                           +
  |                                                 background    |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

* EGCs of 4 bytes or fewer of UTF-8 are encoded directly into gcluster
** All currently-defined Unicode characters are encoded in 4 bytes
** Larger EGC are stored in the EGCPool, with the offset stashed into
    the gcluster's LSBs. This is indicated with a first byte of 1.
* The backstop is always 0, so that gcluster can be used as a C string.
* The width is the number of columns occupied by the EGC.
* Stylemask is a bitfield corresponding to italics, reverse video, blink, etc.
* Secondary cells of a multicolumn EGC have gcluster = 0, width != 0

#### Channel

Encodes either

* 24-bit RGB, or
* up to 24-bit (usually 8-bit) palette index, or
* default color

```
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |0|n| ɑ |p|  0  |               rgb/palette index               |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### Generating output

Output is performed exclusively on planes. Planes can be created, destroyed,
and moved around. A plane bound to itself is one of the *root planes* of its
pile. A new pile is created by reparenting a plane to `NULL`.

#### Formatted output

Several families of character/string/formatted output:

`ncplane_putc()`: Write the provided `nccell` (including style/channels)
`ncplane_putchar()`: Write a 7-bit ASCII `char`
`ncplane_putwc()`: Write a single wide character as an EGC `wchar_t`
`ncplane_putegc()`: Write a single UTF-8 EGC `const char*`
`ncplane_putwegc()`: Write a single wide character EGC `const wchar_t*`
`ncplane_putstr()`: Write a UTF-8 string `const char*`
`ncplane_putwstr()`: Write a wide string `const wchar_t*`
`ncplane_printf()`: Write formatted output `const char*, ...`
`ncplane_vprintf()`: Write formatted output `const char*, va_list`

Each has `_yx()`, `_aligned()`, and `_stained()` variants.

#### Boxes / fills

At their most generic (`ncplane_box()`), boxes are drawn:

* at a starting location, with some geometry
* with 6 specified `nccell`s (including style and channels)
* with the ability to leave out edges/corners
* with the ability to interpolate between corner channels

`simple`, `double`, and `rounded` prepared variants.
`ncplane_perimeter` prepared geometry.

`ncplane_gradient()` fills a rectangular area with a gradient and EGC,
interpolating from four sets of corner channels.

`ncplane_polyfill()` replaces a region of the same glyph with an EGC.

#### Multimedia

Currently support FFmpeg and OpenImageIO, with GStreamer backend in progress.

Multimedia can be broken into a distinct package:

* most of Notcurses is in libnotcurses-core
* the multimedia backend (chosen at compile time) is in libnotcurses
* programs linking against only libnotcurses-core don't even need a multimedia
   backend installed
* even programs which link against libnotcurses ought check
   `notcurses_canopen_images()` and `notcurses_canopen_videos()`
* `ncvisual` objects created with `ncvisual_from_file()`, `ncvisual_from_rgba()`,
    and `ncvisual_from_plane()`

#### Blitters

| Name  | Geometry | ARatio | Fidelity | Glyphs |
| ----- |----------|--------|----------|--------|
|Space  | 1x1      | 2:1    | 100%     | 1      |
|Half   | 2x1      | 1:1    | 100%     | 3      |
|Quad   | 2x2      | 2:1    | 50%      | 15     |
|Sex    | 3x2      | 1.5:1  | 33%      | 63     |
|Braille| 4x2      | 1:1    | 12.5%    | 255    |

#### Widgets

 [ show PoCs for each ]

* Menus
* Selector, multiselector
* Progress bars
* Reels
* Plots

#### Plot blitters

| Name  | Geometry | Glyphs |
| ----- |----------|--------|
|Space  | 1x1      | 1      |
|Half   | 2x1      | 2      |
|Fourths| 4x1      | 4      |
|Quad   | 2x2      | 8      |
|Eighths| 8x1      | 8      |
|Sex    | 3x2      | 11     |
|Braille| 4x2      | 14     |

