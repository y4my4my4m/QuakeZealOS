# Quake for ZealOS

A Quake engine written natively in ZealC. No C source is translated from id's
release: everything here is written against the published file format specs
(PAK, BSP29, MDL, WAD2, progs.dat).


## Game data

`id1/PAK0.PAK` is the shareware archive (id's freely redistributable episode 1).
It is gitignored - fetch it separately:

    curl -O https://ftp.gwdg.de/pub/misc/ftp.idsoftware.com/idstuff/quake/quake106.zip
    unzip quake106.zip                 # yields resource.1, an LHA self-extractor
    7z x -oout resource.1              # yields out/ID1/PAK0.PAK
    cp out/ID1/PAK0.PAK src/Apps/Quake/id1/

Expected: 18254423 bytes, md5 `5906e5998fc3d896ddaf5e6a62e03abb`, 339 entries.

## Files

| File | Role |
|---|---|
| `QFmt.ZC` | Little-endian byte readers; exact binary32 -> F64 widening |
| `QPak.ZC` | PAK mount / lookup / load |
| `QPal.ZC` | `palette.lmp` + `colormap.lmp`, plus a truecolor expansion |
| `QBSP.ZC` | BSP29 loader, all 15 lumps, surface extents, miptex |
| `QTest.ZC` | In-guest self-checks |
| `Run.ZC` | Include chain and entry point |

## ZealC constraints this code is written around

These are silent-failure traps, not style preferences:

- **No struct overlay on file bytes.** ZealC arithmetic is F64-only and postfix
  casts (`x(F64)`) are no-ops. Every on-disk field is decoded explicitly and
  binary32 is widened by rebuilding the double's bit pattern.
- **Globals are not zeroed** by the JIT. Anything read before its first write
  needs an explicit initializer.
- **`<<` and `>>` bind tighter than `*` and `/`** (opposite of C). Shift
  expressions mixed with arithmetic are parenthesized.
- **No `continue`, no ternary, no function-like `#define`.**
- **Locals are function-scoped**, so declarations are hoisted to the top.
- **Brace initializers only at global scope.**
- **A raw `$` in source is a DolDoc command** to the lexer, even inside a
  comment. Use the byte value `0x24`.
- **Mutual recursion through a forward declaration miscompiles.** BSP traversal
  must use direct self-recursion.
- **Single-pass compile**, so `Run.ZC` include order is load-bearing.

## Verifying the data layer

    Cd("::/Apps/Quake");;ExeFile("Run");

`QTest` should print values matching this host-computed ground truth for
`maps/e1m1.bsp` (BSP version 29):

    planes 1810  vertices 7358  edges 13497  surfedges 26702
    nodes 2750   leafs 1531     texinfo 489  faces 5516
    marksurfaces 7073  models 58
    visdata 40843 bytes  lightdata 168590 bytes  entities 26284 bytes

    vert bounds  x  -592.0 .. 1504.0
                 y  -416.0 .. 3064.0
                 z  -592.0 ..  272.0

    solid leafs 1 of 1531
    textures: slipbotsd(16x64) +0slipbot(64x64) slipside(16x16)
              sliplite(16x16) sfloor4_2(64x64)
    model0 mins  -607.0 -431.0 -607.0
    model0 maxs  1519.0 3071.0  287.0

    pal[0] = 00000000   pal[15] = 00EBEBEB   pal[255] = 009F5B53

The vertex bounds are the sharpest check on the binary32 decoder: a mantissa or
exponent mistake shows up there immediately.

## Launching

From the YDE launcher menu: **Quake**. Or from a terminal:

    Cd("::/Apps/Quake");;ExeFile("Run");
