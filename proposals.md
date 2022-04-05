# Proposed Extensions to SLIC
Author: [Peter Hoddie](mailto:peter@moddable.com)<br>
Copyright 2022 Moddable Tech, Inc. All rights reserved.

SLIC distinguishes itself from other fast lossless formats, most notably as QOI, in supporting RGB565 pixels directly. This set of changes enhances SLIC to better support rendering on device with RGB565 pixels.

## Implementation Status
The proposals described here have been implemented in a [fork](https://github.com/phoddie/SLIC) to the [reference SLIC](https://github.com/bitbank2/SLIC) implementation. The changes to the implementation as as light as practical, but are not fully backwards compatible, either at the API or bit-stream level

## Palette Optimizations
The 8-bit SLIC profile offers significant space savings for certain images. This smaller size is desirable on devices with limited storage space. The way in which the palette is stored can be adjusted to improve rendering runtime performance and reduce storage space further.

- Change the palette stored for the 8-bit SLIC profile. The palette entires are stored in RGB565 profile rather than as 8-bit per channel RGB values.
	- This allows the palette entries to be used directly on RGB565 targets without conversion
	- For backwards compatibility, the SLIC APIs that operate on a palette accept and return 24-bit palette entries.
	- This does reduce the color resolution stored from 24-bit to 16-bit
- Store only the palette entries used. As originally implemented the palette is always 256 entires, but many images require fewer colors.
	- The palette storage is changed to add a one byte header storing one less than the number of colors in the palette (allowing a palette lengths from 1 to 256).

## Enhanced Alpha Support
Image formats that store an alpha channel allow for storage of non-rectangular images. GIF images do this primitively using key color, effectively a 1-bit alpha. PNG supports a full alpha. QOI and SLIC also allow for an alpha channel to be stored when storing 32-bit pixels. However, rendering 32-bit pixels to an RGB565 destination on a resource constrained devices is inefficient.

These changes propose adding a SLIC profile to support an alpha channel with RGB565 pixels, `RGB565A`. Because there is no space for an alpha channel in a 565 pixel, the approach taken is to encode the alpha channel separately from the RGB pixels. The alpha channel in QOI and SLIC is 8-bits to match the resolution of the color channels. For RGB565A, this proposal stores the alpha channel in 5 bits, the same as the red and blue channels.

- `RGB565A` SLIC profile stores color information in (almost) exactly the same way as the original `RGB565` SLIC profile.
- One difference is that the range of the `SLIC_OP_BADRUN16` opcode is reduced by half, from 64 to 32, to make room for an opcode to store the alpha channel.
- The `SLIC_OP_ALPHA` opcode is added by using `SLIC_OP_BADRUN16` with the high bit of the count set to 1. This leaves 5 bits for the alpha channel.
- The initial value of alpha is initialized to 0, with the assumption that the top-left corner will more often be transparent than opaque.
- A `SLIC_OP_ALPHA` opcode is output whenever the alpha changes.
- The alpha value is constant for all pixels in RUN, BADRUN, or INDEX opcodes. The encoder enforces this requirement.
- When the alpha value is 0, the color value is irrelevant as the pixel should not be displayed. In this case, the encoder uses the last encoded color, instead of the actual pixel color. This has been shown to marginally reduce the encoded image size by preventing a fully transparent pixel from using a hash index entry.
- This approach effectively encodes the alpha as an independent, run length compressed plane. This simple approach works well because alpha is commonly used either in long runs of opaque (31) or transparent (0) with only a few alpha values in-between at the edges of shapes.
- The encoding API has been extended to accept the BPP value of 21 (16 + 5) to signal that the `RGB565A` profile should be generated. In this case, the input pixels are 32-bit BGRA, as when encoding the SLIC 32 bit variant. The encoder subsamples the color and alpha channels.
- Colors values are not premultiplied with the alpha. Doing so might improve rendering performance but also might reduce the effectiveness of the hash index.

In the interest of minimizing the number of SLIC profiles, it could make sense to replace the `RGB565` profile with the `RGB565A` profile. The `BADRUN` opcode will be slightly less efficient for runs longer than 32 pixels, but this is already the least efficient op-code. If the alpha value were instead initialized to opaque (31) there would be no space overhead for many of the fully opaque images encoded by the `RGB565` today. Decoders can decide to process or ignore the alpha values.

The technique used to add alpha support to the 16-bit SLIC profile should also work to add alpha support to the 8-bit SLIC profile. This would put SLIC in the unique and enviable position of supporting alpha at 8, 16, and 24 bit color depths.
