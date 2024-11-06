# DNG SDK for modern CMake

This is a fork of Adobe's DNG SDK, set up to be used with modern CMake.
Source files are unchanged other than fixes for compatibility with latest jpeg-xl APIs.

Build tested on Windows, macOS, iOS, Android,and Emscripten.

Currently synced to DNG SDK version 1.7.1

Requires the following libraries:
- Adobe XMP ([modern CMake fork here](https://github.com/developandfix/xmp))

For JPEG-XL support (WITH_JPEGXL=ON):
- JXL
- Brotli
- HWY
- LCMS2
