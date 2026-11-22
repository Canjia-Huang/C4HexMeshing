![new-new-teaser](https://github.com/user-attachments/assets/110726b8-4d61-4e38-8d63-75f78a90bec7)
# *Flexible Singularities* now integrated into C4HexMeshing!
An implementation of our new paper [Volume Quantization with Flexible Singularities for Hexahedral Meshing](https://graphics.cs.uni-paderborn.de/papers/FlexibleSingularities_Bruckler_EG2026.pdf) (coming to Eurographics 2026) has been integrated recently.
It allows adaptive simplification of the initial singularity graph at the quantization level via [QGP3D](https://github.com/HendrikBrueckler/QGP3D) and can extract the simplified structure or a hex mesh.

Additionally we have recently integrated direct volume mesh visualization via [polyhydra](https://github.com/CG-UPB/polyhydra) into the library, enabling fast and interactive visualization of the algorithm's stages and their results.
***

![teaser](https://github.com/HendrikBrueckler/C4HexMeshing/assets/38473042/f70ef0d1-e1f4-4e35-a024-ad9440baf83c)
# Collapsing Cubical Cell Complexes for Hex Meshing

`C4HexMeshing` is an implementation of [Collapsing Embedded Cell Collapses for Safer Hexahedral Meshing \[Brückler et al. 2023\]](http://graphics.cs.uos.de/papers/T-Collapsing_Bru%CC%88ckler_SA2023_Preprint.pdf) (SIGGRAPH Asia 2023) distributed under GPLv3.

If you make use of `C4HexMeshing` in your scientific work, please cite our paper. For your convenience,
you can use the following bibtex snippet:

```bibtex
    @article{C4HexMeshing,
        author     = {Hendrik Br{\"{u}}ckler and
                     Marcel Campen},
        title      = {Collapsing Embedded Cell Collapses for Safer Hexahedral Meshing},
        journal    = {ACM Trans. Graph.},
        volume     = {42},
        number     = {6},
        year       = {2023},
    }
```

***

## How does it work?

`C4HexMeshing` makes use of the [3D Motorcycle Complex](https://github.com/HendrikBrueckler/MC3D) to partition a tetrahedral mesh, equipped with a suitable seamless map, into blocks. A quantization of the cubical cell complex is then computed using [QGP3D](https://github.com/HendrikBrueckler/QGP3D), assigning non-negative integer lengths to the complex' edges.

To extract from this complex a valid integer-grid map and thus a hex mesh, all edges, patches and blocks of the complex which were quantized to zero extent are collapsed and their content redistributed among the remaining elements. On the remaining cell complex, whose cells are now blocks of strictly positive integer extents, an integer-grid map can be computed blockwise via (relatively simple) cube maps.
From this a hex mesh can be extracted.

Note, that (currently) no geometric optimization other than tentative untangling of the integer-grid map is performed, but some might be added in the future.

![Collapses](https://github.com/HendrikBrueckler/C4HexMeshing/assets/38473042/4fbb3c16-1baf-47e7-9d6b-58b7fc8555b3)

***

### Dependencies
- GMP (NOT included, must be installed on your system)
- Clp (NOT included, must be installed on your system)
- NLOPT (optional, for map optimization, NOT included, must be installed on your system)
- [MC3D](https://github.com/HendrikBrueckler/MC3D) (included as submodule, together with all subdependencies)
- [QGP3D](https://github.com/HendrikBrueckler/QGP3D) (icluded as submodule, together with all subdependencies)
- [polyhydra](https://github.com/CG-UPB/polyhydra) (volume mesh viewer, included as submodule, enabled via CMake flag ```-DMC3D_WITH_VIEWER=On```)

### Building
In root directory

    mkdir build
    cd build
    cmake -DGUROBI_BASE=<path/to/gurobi/> ..
    make

### Usage
An example command-line application is included that reads a tetrahedral mesh including a seamless parametrization from a file in .hexex-format, as used and documented in [libHexEx](https://www.graphics.rwth-aachen.de/software/libHexEx/).
It can generate the output of several stages of the algorithm, including the original MC, collapsed MC, integer-grid map (unoptimized) and hex mesh (unoptimized).

After building, the CLI app can be found in ```build/Build/bin/``` .
For full information on its usage, execute

    c4hex_cli --help

### Input data
Example input can be found in folder ```extern/MC3D/tests/resources```.
A dataset of 200 precomputed motorcycle complexes on seamless parametrizations from various sources is available under [MC3D-samples](https://github.com/HendrikBrueckler/MC3D-samples).

### Viewer Support
Additionally, the CLI (and library) can be built with support for volume mesh visualization via [polyhydra](https://github.com/CG-UPB/polyhydra), by calling CMake with the additional flag ```-DMC3D_WITH_VIEWER=On```.
In this case a dedicated viewer app for inspecting the extracted meshes is built as ```build/Build/bin/c4hex_viewer_demo```.

### API
For details on the API of the library, check the headers in ```include```, they are thoroughly documented. Apart from that, ```cli/main.cpp``` demonstrates usage of the entire pipeline for both simple and advanced usage.

***

## Fork-specific changes

This repository is a fork of [HendrikBrueckler/C4HexMeshing](https://github.com/HendrikBrueckler/C4HexMeshing), based on
its commit [`c3971f3`](https://github.com/HendrikBrueckler/C4HexMeshing/commit/c3971f3) (*Submodule bugfix*), which is the
tip of the original repository's `main` branch.
All changes are confined to the example command-line application ([`cli/main.cpp`](cli/main.cpp)); the library
(`src`, `include`) and the submodules are untouched (see `git diff c3971f3..main`).
In short:

1. [New option `--output-igm-pre-requant`](#1-output-the-igm-between-collapsing-and-re-quantization): write the IGM as
   it is right after collapsing, i.e. before it is re-quantized.
2. [`.hexex` suffix](#2-hexex-suffix-for-igm-outputs): IGM outputs are now written with a `.hexex` extension.
3. [`--output-igm-pre-requant` requires `--collapse`](#3-output-igm-pre-requant-requires-collapse) and stops the
   pipeline after writing.
4. [MC walls in every IGM output](#4-mc-walls-in-every-igm-output): the double precision IGM output now contains the
   facets separating different cells, so all IGM outputs can be read back with `--input-has-walls`.

### 1. Output the IGM between collapsing and re-quantization

In block structured mode (`--collapse --block-structured`) the quantization is redone after the zero-elements of the
motorcycle complex have been collapsed, i.e. the IGM that the pipeline finally outputs no longer corresponds to the
quantization that the collapse was based on.
The new option

    --output-igm-pre-requant <prefix>

generates the IGM from the collapsed (still fully quantized) motorcycle complex, i.e. exactly the state the
re-quantization starts from, writes it to

    <prefix>_pre_requant_{clean,dirty}.hexex        (double precision)
    <prefix>_pre_requant_{clean,dirty}_exact.hexex  (exact rational numbers)

and then stops the run (`clean`/`dirty` indicates whether inversions remain in the IGM).
Because generating an IGM also refines and remeshes the base tet mesh, continuing after this step would change the
subsequent re-quantization and the final output; the run therefore stops here and all other requested outputs
(`--output-igm-file`, `--output-hex-file`, `--output-constraint-file`, re-quantization) are skipped, with a
warning being logged in that case.
Since there is no post-collapse state without collapsing, the option requires `--collapse` (rejected by the parser
otherwise).
Note that without `--block-structured` no re-quantization happens at all, so there the post-collapse IGM is the
state the final IGM is generated from anyway.

To obtain both the pre- and the post-re-quantization IGM, run the pipeline twice:

    # IGM of the collapsed quantization, before re-quantization (stops after writing)
    c4hex_cli --input hand.hexex --collapse --num-hexes 500 \
              --output-igm-pre-requant out/hand_pre

    # IGM after re-quantization, plus hex mesh (stops as usual, after the final output)
    c4hex_cli --input hand.hexex --collapse --block-structured --num-hexes 500 \
              --output-igm-file out/hand_post --output-hex-file out/hand_post

### 2. `.hexex` suffix for IGM outputs

IGM outputs were written with a `.igm` extension, but their content is .hexex format (as documented by
[libHexEx](https://www.graphics.rwth-aachen.de/software/libHexEx/)), which is also what the input (`--input`) expects.
They are therefore written with a `.hexex` extension now, i.e.

    <output-igm-file>_<times-minimal>_{clean,dirty}.hexex
    <output-igm-file>_<times-minimal>_{clean,dirty}_exact.hexex

for the final IGM and `.hexex` instead of `.igm` for `--output-igm-pre-requant` as well.

### 3. `--output-igm-pre-requant` requires `--collapse`

Requesting the post-collapse IGM without collapsing used to run the whole pipeline and silently write nothing (the
collapse branch is never entered). `--collapse` is now a hard dependency of the option, so that the parser rejects
this combination with a clear error instead.

### 4. MC walls in every IGM output

The double precision IGM output was written via `Writer::writeIGM()`, which omits the MC wall section, while the
exact output already used `Writer::writeIGMAndWalls()`. As a consequence, the double precision file could not be
read back with `--input-has-walls`.
Both outputs are now written with the MC walls, i.e. with the facets that separate the individual blocks (cells) of the
motorcycle complex, taken from the `IS_WALL` property (which is maintained consistently with the
`BLOCK_MESH_TETS`/`PATCH_MESH_HALFFACES` block and patch mappings throughout collapsing, remeshing and IGM
generation). The walls also include the boundary faces, so every block is a closed surface in the file and
`--input-has-walls` reconstructs the same block structure, e.g.

    c4hex_cli --input out/hand_post_100_clean.hexex --input-has-walls \
              --output-igm-file out/hand_re

For a collapsed motorcycle complex with *N* blocks, removing the wall facets from the written tet mesh yields exactly
*N* connected components, i.e. the walls isolate the cells exactly.
