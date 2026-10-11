# FORK CHANGES

This adds support to make bitmap VobSub subtitles semi-transparent.

You'd need to build this into a DLL, register it, and configure your video player to use it.
Below are instructions based on how I did it for PotPlayer. Only the 64-bit builds have been
tested.

## Building

You need Visual Studio 2019 or 2022 with the "Desktop development with C++" workload and these
individual components:

- C++ ATL for latest (v142 or v143) build tools (x86 & x64)
- C++ MFC for latest (v142 or v143) build tools (x86 & x64)

The build also needs yasm. Download the win64 exe from the
[latest yasm release](https://github.com/yasm/yasm/releases/latest), rename it to `yasm.exe`, and
put it in a folder on PATH.

Open 'Developer Command Prompt for VS 2019' (or 2022) in the repository root and build the 64-bit
VSFilter:

```
MSBuild VSFilter.sln /m /t:vsfilter /p:Configuration=Release /p:Platform=x64
```

The DLL is written to `bin\lib_16.0\x64\Release\VSFilter.dll` (VS 2019) or
`bin\lib_17.0\x64\Release\VSFilter.dll` (VS 2022).

## Installing

Register the DLL from an administrator command prompt:

```
regsvr32 VSFilter.dll
```

Use the 64-bit DLL for 64-bit players. The filter keeps the DirectVobSub CLSIDs, so registering
it replaces any other registered DirectVobSub, for example the one from K-Lite Codec Pack. A
codec pack update can register its own copy again. Run regsvr32 on this DLL again after such an
update.

## Setting the opacity

The opacity is a DWORD registry value in percent, from 0 (invisible) to 100 (opaque). The default
is 100, which draws VobSub subtitles the same way as the original filter. The filter clamps
values outside 0-100.

```
HKEY_CURRENT_USER\Software\Gabest\VSFilter\VobSub\Opacity
```

For example, this sets 65% opacity:

```
reg add "HKCU\Software\Gabest\VSFilter\VobSub" /v Opacity /t REG_DWORD /d 65 /f
```

The filter reads the value when it loads, so a new value takes effect the next time you open a
file. The property pages have no field for it.

## Setting up PotPlayer

PotPlayer renders subtitles itself by default, and its built-in Matroska splitter doesn't pass
subtitle tracks to DirectVobSub. These steps make PotPlayer use LAV Splitter and the auto-loading
DirectVobSub. LAV Filters must be installed, for example from K-Lite Codec Pack.

1. Register VSFilter.dll as described in "Installing".
2. In Preferences > Filter Control > Filter Priority (Overall), click "Add registered filter...",
   select "DirectVobSub (auto-loading version)", and set its priority to "Prefer".
3. In the "Source/Splitter" section for "Matroska" dropdown choose "*LAV Splitter Source". If it
   doesn't exist click "Filter Management" button and if the "DirectShow Filter List" on the left
   is empty click "Scan" and OK, then you should have it in the dropdown.
4. Open an MKV file with VobSub tracks. There should be a green arrow icon in the system tray
   when the file is playing. Right click on that icon brings up its menu, which should also show
   a list of subtitles.

The system tray menu lists more than one group of tracks. The first group is the splitter's own
track list. Keep PotPlayer's subtitles on (Alt+H toggles them), because PotPlayer renders the
subtitle formats other than VobSub. Internally there's a concept of inputs, outputs and
connection pins. The auto-loading DirectVobSub connects the free VobSub output pins of a known
splitter, such as LAV Splitter, to its own subtitle inputs. Players with a built-in subtitle
renderer leave these pins unconnected, and then the embedded VobSub tracks don't reach the
filter. The filter doesn't touch pins that the player already uses. It doesn't connect pins of
other subtitle formats, such as SRT, so the player renders them with its own style.

What works in PotPlayer with this setup:

- VobSub tracks inside MKV files work, they're semi-transparent.
- SRT tracks inside MKV files work, PotPlayer renders them with its own style, which you can
  configure to be nicely semi-transparent with opaque outline
- External .idx/.sub files next to the video file - you get double subtitles rendered over
  each other, just use Alt-H to hide PotPlayer's.

## What about XySubFilter.dll?

This repo can build two different things, DirecVobSub (VSFilter.dll) and XySubFilter
(XySubFilter.dll). They're build from shared code but the way they integrate with video players
is completely different as far as I understand it. I tried getting both of them to work for
completeness sake, but XySubFilter turned out worse than DirectVobSub (VSFilter.dll). In
PotPlayer, from my quick testing, XySubFilter is less usable for MKV files:

* installed madVR (in madVR's folder ran install.bat as administrator)
* built XySubFilter like VSFilter, with the `xy_sub_filter` target instead of `vsfilter`
* copied bin\lib_16.0\x64\Release\XySubFilter.dll to _install
* registered XySubFilter `regsvr32 ...\xy-VSFilter\_install\XySubFilter.dll`
* set its opacity registry value (full path below)
* set up PotPlayer to use this path instead of DirectVobSub:
   - set "Video Renderer" to madVR ("Madshi Video Renderer"), note that this
     option is present even if madVR isn't installed and it won't do anything,
     to confirm it works look for a madVR icon in the system tray while a video
     plays
   - in "Filter Priority (Overall)" added XySubFilterAutoLoader with "Prefer",
     blocked "DirectVobSub (auto-loading version)"
   - opened a MKV with extracted .idx/.sub next to it. I extracted them with
     MKVToolNix: `mkvmerge -i movie.mkv` lists the tracks, and
     `mkvextract movie.mkv tracks "2:movie.idx"` writes VobSub track 2 to
     movie.idx and movie.sub
   - two tray icons appear, green arrow with 'XySubFilter' tooltip, and madVR

What I noticed:

* VobSub tracks inside MKV files don't work. XySubFilter doesn't load at all, and PotPlayer
  renders the tracks itself, opaque.
* External .idx/.sub files next to the video work, XySubFilter draws them semi-transparent. As
  with VSFilter, PotPlayer draws its own opaque copy over it.

I'm leaving this in since I already tried it, but I'll be using the DirectVobSub method.

Using XySubFilter.dll needs setting its own registry key:

```
HKEY_CURRENT_USER\Software\Gabest\XySubFilter\VobSub\opacity
```

Build it the same way, with `/t:xy_sub_filter` instead of `/t:vsfilter`.


Original README below:

---

# xy-VSFilter (pfmod) v3.2.0.810 (20240123)
https://github.com/pinterf/xy-VSFilter/

Active branch: xy_sub_filter_rc5

A modification by pinterf. 

Mod base: 'official' xy_sub_filter_rc5
https://github.com/Cyberbeing/xy-VSFilter/tree/xy_sub_filter_rc5

Plus:
- VSFilter: Avisynth filters

# Useful links:
[Doom9 forum](https://forum.doom9.org/showthread.php?t=168282)

[Avisynth wiki](http://avisynth.nl/index.php/Xy-VSFilter)

# Info:
- Package contains VSFilter.dll and XySubFilter.dll (x86 and x64 versions)

- VSFilter.DLL: Included Avisynth filters (name and parameter signature)
  
      VobSub "cs"
      TextSub "c[file]s[charset]i[fps]f[vfr]s"
      TextSubSwapUV "b"
      MaskSub "[file]s[width]i[height]i[fps]f[length]i[charset]i[vfr]s[pixel_type]s"

- Warning by the XySubFilter-3.1.0.746 beta authors (on forum's 1st post, from 2015):
  
  After downloading XySubFilter BETA3, you must ensure to run the 'Install' bat (not only replace) or else XySubFilter's autoload helper required for entering the DirectShow graph with external subtitles will not be installed

# Change log:

## v3.2.0.810 (20240123)

* Fix VsFilter crash on load/release (as external DirectVobSub in players). Regression since v804.

## v3.2.0.809 (20240102)

Updates, fixes and some backports from Cyberbeing/xy-VSFilter by TheOneric, Masaiki and Cyberbeing

* Gracefully handle negative Encoding values
  more on this: https://github.com/pinterf/xy-VSFilter/commit/102897189902658ec68ef691eb3dd8d89f099985
* Update CSRI name
* Use distinct CSRI name and update CSRI version:
  Adjust the name so users can distinguish Cyberbeing/xy-VSFilter
  from pinterf/xy-VSFilter if builds from both are placed into Aegisub’s csri directory
* fix: time shift when playing external sup files
* fix: close sup file and read stop time of sample
* Introduce LayoutRes{X,Y} script headers. See details here:
  https://github.com/pinterf/xy-VSFilter/commit/8a46df19666191f6fcaa44811123c34f165f358e

## v3.2.0.808 (20221116)
* Fix Possible unhandled exception in Masksub (https://github.com/pinterf/xy-VSFilter/issues/36)
* Fix: no more vertically flipped MaskSub result for RGB32
* remove latest scaling workaround
  More on that: https://github.com/pinterf/xy-VSFilter/pull/34

## v3.2.0.807 (20210522) - WIP
* fix: scale between subtitle and video on CWord (Masaiki) (Issue: frx and fry tags wrong scaling)

## v3.2.0.806 (20210412)

* proper WebVTT support fixes by adipose/MPC-HC

## v3.2.0.805 (20210412)

* WebVTT support, embedded subtitles as well.
  99% of it extracted from MPC-HC https://github.com/clsid2/mpc-hc, thanks to MPC-HC developers.
* Changed the Subtitles library timestamp to REFERENCE_TIME (64 bit) internally.
  For very-very long clips (use case: streaming) (slydiman)
* Fix: ASS subtitles: styles with space after their names would fail to work (MPC HC/BE + external XySubFilter)
  (report: MatiasMovie, solution by astiob)
* Various code syntax fixes (clang compiler)

## v3.2.0.804 (20210306 - pinterf)
* Add native YV16, YV24, P210 (YUV422P10) and P216 (YUV422P16) support.
  From TextSub Avisynth filter they even work.

## v3.2.0.803 (20210305 - pinterf)
* Source: new code base in new branch which finally works with Visual Studio 2019:\
  https://github.com/Cyberbeing/xy-VSFilter/tree/xy_sub_filter_rc5
* upstream:
  * (cyberbeing) Revert MPC-HC Tag Cache, restore xy Tag Cache
  
   Fixes a significant performance regression on certain SSA/ASS samples.
  * (jesec) subtitles, subpic: enable SSE2 unconditionally
     Target OS (10.0) required users to have SSE2
     It is meaningless to maintain backward compatibility
  * (jesec) Unicode conversion for file src/dsutil/DSUtil.cpp
     Remove support for DX7
  * (jesec) Rasterizer::Rasterize: use optimized memsetSSE2.
    Source: https://www.agner.org/optimize/
  * (cyberbeing) Add VS2019 environment
* Active branch changed to xy_sub_filter_rc5 on my (pinterf) side as well
* Avisynth plugin
  * re-add my changes from 2018
  * Add support for YV16
  * Throw error on unsupported formats
  * Avisynth plugin: keep frame properties when Avisynth+ interface is v8
* version set manually in src/filters/transform/vsfilter/version_in.h

## v3.2.0.802 (20181031)
* Change version numbering.
  
  Cyberbeing has released a patch for their original xy-VSFilter, their version stepped to v3.1.0.752
  
  Since I thought the project was full-dead, my version numbering kept - after a range skip - the original numbering.
  
  Now I'm changing my versioning scheme to 3.2.x.x to avoid possible future collision.
  
* Fix TextSub crash that could occur in rare circumstances (shssoichiro)

* Fix for SSA/ASS repositioning becoming permanently disabled after typesetting was displayed\
  (madshi -> Cyberbeing v3.1.0.752 7th October 2018)
  
* Add BT.2020 Support\
  (madshi -> Cyberbeing v3.1.0.751 27th September 2018)

## v3.1.0.801 (20180904)
* fix random crash in x64 build (address truncated to 32 bits, Rasterizer::Draw and Rasterizer::FillSolidRect were affected)
* XySubFilter.DLL now compiled with v141_xp toolset instead of v141 (like VSFilter.DLL)
* From upstream (madshi, Cyberbeing)
  * Fix a possible infinite loop in the Real Text subtitle parser on 64-bit
  * Fix external SRT subtitles with italic tag not being flagged as repositionable
  * Disable repositioning for SSA/ASS subtitles with default position level
  * Add setting to allow repositioning of SSA/ASS dialog

- v3.1.0.800 (20180411)
* Bring source to XySubFilter-3.1.0.746 level:\
  Apply file differences between Cyberbeing XySubFilter-3.1.0.746 (source from the release pack) \
  and xy-VSFilter 3.0.0.306 mod.
* make the source VS2017 compatible again
* make VSFilter work again as an Avisynth plugin
* Versioning: 3.1.0 prefix is kept, build number goes from 800 and up for my mod manually in src/filters/transform/vsfilter/version_in.h

## v3.0.0.400 (20180405) (work in February 2018)
* Get the last known xy-vsfilter version (3.0.0.306) which was still working as the Avisynth plugin.\
  https://github.com/Cyberbeing/xy-VSFilter/releases/tag/3.0.0.306\
  Plan: have a native 10/16 bit compatible Avisynth TextSub filter.\
* xy-vsfilter 3.0.0.306 rebuilt with Visual Studio 2017
* Avisynth+ headers
* Native support for YUV420P10 and YUV420P16 formats in Avisynth filter TextSub and probably VobSub.
  Internal conversion to and from P010 and P016 with SSE2 support
* Still doesn't work in any MT mode (I think): automatically registers itself as MT_SERIALIZED to prevent troubles
* Avisynth filter MaskSub: new pixel_type (default: "RGB32") parameter. Can be set to "YUV420P10" or "YUV420P16"
* Versioning: 3.0.0 prefix kept, build number goes from 400 and up for my mod manually in src/filters/transform/vsfilter/version_in.h
