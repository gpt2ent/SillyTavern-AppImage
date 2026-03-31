# SillyTavern-AppImage

Vibecoded appimage for sillytavern v1.17.0 + agent instructions. Supports GLIBC 2.31 for ubuntu 22.04 compatibility.

The purpose of this repo is to share the binary and notes that my agent compiled while building it, to save tokens and minimize retracing dead ends when reproducing the build process.

Here's the gist of it:

- Download AppImage for ST v1.17.0 from [releases page](https://github.com/gpt2ent/SillyTavern-AppImage/releases)
- The AppImage will **NOT** be auto-updated following new ST releases. It will be updated only if AppImage has problems that I can reproduce.
- If you want to build an AppImage for newer ST release, you can look or direct your coding agent to the files in this repository to save some time and tokens on this task:
  - [PATCHING_FOR_APPIMAGE_FINAL](https://github.com/gpt2ent/SillyTavern-AppImage/blob/main/PATCHING_FOR_APPIMAGE_FINAL.md) is the compilation of all the steps that will reproduce the building process.
  - [PATCHING_FOR_APPIMAGE](https://github.com/gpt2ent/SillyTavern-AppImage/blob/main/PATCHING_FOR_APPIMAGE.md) is more detailed "scratchpad" that my agent maintained while patching sillytavern repo to support building an AppImage off of it. If some instructions are not clear in FINAL version, maybe there is more useful details in this "raw" scratchpad.
  - Note that new versions are likely to introduce some changes to the building process or patches, so you will still need to work through some obstacles. Additionally, you might choose to change some of the instructions, e.g. if you dont need backward compatibility for older distros.
