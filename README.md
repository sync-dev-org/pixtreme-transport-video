# pixtreme-transport-video

`pixtreme.transport.video` reads and writes video files for pixtreme.transport.
Its `VideoEntrance` and `VideoExit` factories create pixtreme.transport sessions
with the module name `video`, using buffers owned by the caller. Input uses the
pixtreme-transport-ffmpeg demuxer with NVDEC or FFmpeg decoding; output uses
NVENC or FFmpeg encoding with the pixtreme-transport-ffmpeg muxer. The module
also provides `PacketWriter` for writing already encoded packets to a container
without a session, encoder or decoder.

## Installation and requirements

Install from PyPI:

```sh
python -m pip install pixtreme-transport-video
```

The release platforms are Linux x86_64 (glibc 2.28 or later) and native Windows
x86_64. Install a wheel for your platform from the selected release; a platform
is available only when that release provides its wheel. The supported Python
versions are standard CPython 3.12 and 3.13; wheels use the CPython 3.12 stable
ABI. Video sessions use an NVIDIA GPU and require the CUDA-compatible driver
specified by the installed [pixtreme](https://pypi.org/project/pixtreme/) release;
NVDEC and NVENC also need a GPU and driver that support the selected codec.

The direct runtime dependencies are `pixtreme>=2.31.1`,
`pixtreme-transport>=0.53.0`, `pixtreme-transport-ffmpeg>=0.7.1`,
`pixtreme-transport-nvdec>=0.15.0` and `pixtreme-transport-nvenc>=0.15.0`.
All three codec companions are installed whichever library a session selects.
Python installation resolves these packages and their own dependencies.

## Python sessions

Create input and output sessions through `video`:

```python
from fractions import Fraction

from pixtreme.transport import video
from pixtreme.transport.session import ExitSettings, FrameBuffer, Pacing

device = 0  # Ordinal after CUDA_VISIBLE_DEVICES filtering.
input_buffer = FrameBuffer(2, mode="blocking")
source = video.VideoEntrance(device, input_buffer, {"path": "input.mp4"})

output_buffer = FrameBuffer(2, mode="blocking")
settings = ExitSettings(Fraction(1001, 24000))  # Frame period, in seconds.
settings.pacing = Pacing.NONE
target = video.VideoExit(device, output_buffer, settings, {"path": "output.mp4"})
```

Creation returns unopened sessions and does not open files or codec sessions.
Call `source.open()`, read with `source.next()` until it returns `None`, and
close the source in a `finally` block. Each returned item's `frame` contains the
picture and timing. To write, put the processed transport frames into
`output_buffer`, open the target, close the buffer after the last frame, and
call `target.join()` to wait for all writes. Close the target in a `finally`
block. Run the producer and consumer concurrently when filling a blocking buffer
beyond its capacity.

Input publishes decoded pictures in display order as quickly as its blocking
buffer accepts them, drains delayed pictures at the end of the file and supplies
48 kHz stereo PCM when the input has audio. It does not pace pictures by the OS
clock or attach receipt timestamps. Input requires a blocking buffer and rejects
a buffer that drops frames. For a file pipeline, `Pacing.NONE` and FIFO
selection write every accepted picture once and wait while the output buffer is
empty and open. Closing and draining the buffer finishes output.
`Pacing.CORE_CLOCK` with FIFO selection is also accepted; it writes the pictures,
repeats and audio supplied by the core at its output cadence.

Output writes the container to a temporary file in the destination directory
and replaces `path` only after the container trailer and the file close succeed.
A failure or a requested stop removes the temporary file and leaves an existing
file at `path` unchanged.

### Video options

Options are strings, supplied as a mapping or an iterable of name/value pairs.

| Option | Input | Output |
|---|---|---|
| `path` | Required video filename | Required video filename; its extension must select one container by the `PacketWriter` extension rule below |
| `decode.library` | `nvdec` (default) or `ffmpeg` | Not accepted |
| `provide_mv_residual` | `"true"` or `"false"` (default) | Not accepted |
| `encode.library` | Not accepted | `nvenc` (default) or `ffmpeg` |
| `encode.video` | Not accepted | Video codec: `h264` (default), `hevc` or `av1` with NVENC; an FFmpeg encoder name, default `libopenh264`, with FFmpeg |
| `encode.audio` | Not accepted | Audio encoder; default AAC |
| `encode.pixel_format` | Not accepted | Encoded pixel format; H.264 and HEVC default to 8-bit YUV 4:2:0, other codecs require it |
| `encode.video_option.<name>`, `encode.audio_option.<name>` | Not accepted | FFmpeg AVOptions for the video or audio encoder; FFmpeg only |
| `encode.nvenc.<name>` | Not accepted | NVENC encode settings; NVENC only |

Paths are literal filenames, including `%` and `%%`. A `%0Nd` number field is
rejected; use pixtreme-transport-sequence for numbered images. The selected
decoding or encoding library never falls back to another library; a codec,
format or setting that it cannot handle fails before the first picture with the
reason. Software decoding passes FFmpeg pixel formats that pixtreme can
represent without conversion and refuses others with the decoder's reason.

When `provide_mv_residual` is `"true"` with NVDEC and HEVC (Main or Main 10,
4:2:0, 8 or 10 bit, progressive), every frame carries the decoder's `mv` and
`residual` arrays under the `hevc-mv-residual` version 4 contract. FFmpeg
decoding and other codecs reject the request before publishing any picture.

When `encode.pixel_format` is omitted, output converts each picture to the
default format, except that it refuses pictures with alpha and PQ or HLG
pictures rather than discarding alpha or converting HDR to SDR. Explicit
settings take precedence. Different libraries' quality settings are passed to
the selected library unchanged rather than translated into one scale. Unknown,
duplicate, missing or inapplicable settings fail with the setting name and
reason; diagnostics omit setting values and name files by the caller's `path`.

## Writing encoded packets

`PacketWriter` writes FFmpeg packets that are already encoded, without a core
session, frame buffer, encoder or decoder:

```python
from pixtreme.transport import ffmpeg, video

with ffmpeg.Demuxer("input.mp4") as source:
    with video.PacketWriter("output.mp4") as writer:
        streams = {stream.index: writer.add_stream(stream) for stream in source.streams}
        for packet in iter(source.read, None):
            writer.write_packet(streams[packet.stream_index], packet)
```

`PacketWriter(path, *, format="")` accepts `str` and `PathLike` filenames.
It registers `ffmpeg.StreamDescription` objects with `add_stream` and accepts
`ffmpeg.Packet` objects with `write_packet`. The returned output stream numbers
can differ from the input indices. Register all streams before the first packet;
the first packet or `finish()` starts the container header. The caller supplies
the interleave and DTS order required by the container. The writer passes
packets unchanged to the muxer, which rescales each packet's own time base.

An explicit `format` selects a name from `ffmpeg.muxers()`. Otherwise the writer
matches the filename extension against that registry's extension lists and
counts distinct names. If several names match, a unique name equal to the
extension wins, ignoring case. This lets `.mp4` select `mp4` despite `psp` also
listing that extension, without depending on registry order. Unknown and still
ambiguous extensions fail; specify `format` to resolve them. This selection is
the video writer's rule, not FFmpeg's filename-guessing algorithm.

The writer owns a temporary file in the destination directory and gives the
muxer write and seek callbacks. `finish()` writes the trailer, closes the file
and replaces the destination only after success. `abort()` and unfinished
destruction discard the temporary file. A context manager finishes on normal
exit and aborts on an exception. Failures identify the operation and reason;
file failures show the caller's destination path, with other setting values
omitted.

For NVENC output, `video.stream_description_from_nvenc(effective_settings,
configuration, *, time_base=(numerator, denominator))` creates an FFmpeg stream
description from `session.effective_settings()` and
`session.codec_configuration()`. Query both snapshots before flushing or closing
the encoder. `video.packet_from_nvenc(access_unit, *, time_base=(numerator,
denominator))` creates an owning FFmpeg packet from each `nvenc.AccessUnit`.
Use the same explicit time base that the caller used for the input timestamps;
both integers must be positive. These functions copy the bytes, PTS, signed DTS
and IDR flag without encoding or decoding, and leave duration unknown. They do
not infer a clock from the frame rate. H.264/HEVC retain Annex B framing; AV1
retains its native OBU framing. Each present colour declaration in the accepted
settings is copied: primaries, transfer and matrix use their shared H.273 codes,
and full range maps to FFmpeg's limited/full range. Absent colour values remain
unspecified. Profile, pixel format and video delay remain unspecified rather
than inferred from NVENC enum values. The returned FFmpeg values can be passed
directly to `PacketWriter`; use the output stream number returned by `add_stream`.

## C and C++ SDK

Download the SDK archive for your platform and version, together with
`SHA256SUMS`, from the
[GitHub Releases](https://github.com/sync-dev-org/pixtreme-transport-video/releases).
Verify the archive's SHA256 before extracting it. The SDK provides the same
module library as the wheel, public headers, CMake package files and licenses.
Obtain compatible shared SDKs for pixtreme, pixtreme-transport,
pixtreme-transport-ffmpeg, pixtreme-transport-nvdec and
pixtreme-transport-nvenc separately; their libraries are supplied by those
distributions.

Point CMake's `pixtreme_transport_video_DIR` to the extracted SDK's
`pixtreme/transport/video/cmake` directory, and the dependencies' package
variables to their SDKs. C and C++ consumers link the module through its
imported CMake target:

```cmake
find_package(pixtreme_transport_video CONFIG REQUIRED)
target_link_libraries(application PRIVATE pixtreme::transport_video)
```

C++ consumers use C++20. For creation by name, install the SDK trees under one
common prefix so the core can locate `pixtreme/transport/video/lib/` on Linux or
`pixtreme/transport/video/bin/` on Windows. Configure the platform's shared
library search paths for the dependencies and keep one copy of each shared
library per process.

Use `pixtreme_transport_module_create_entrance` and
`pixtreme_transport_module_create_exit` from `<pixtreme/transport/c/module.h>`
in C, or `pixtreme::transport::module::create_entrance` and `create_exit` from
`<pixtreme/transport/module.h>` in C++, passing the module name `video`.
Supply the device, caller-owned blocking buffer, output settings where needed,
and the same video options shown above. Open each returned session to begin
I/O, and close sessions before releasing their owned handles.

C users write packets with `pixtreme/transport/video/c/packet_writer.h` and
convert NVENC output with `pixtreme_transport_video_packet_from_nvenc` and
`pixtreme_transport_video_stream_description_from_nvenc` from
`pixtreme/transport/video/c/nvenc.h`. C++ users include
`pixtreme/transport/video/packet_writer.h` and `pixtreme/transport/video/nvenc.h`
and use `pixtreme::transport::video::PacketWriter` and the same two function
names as Python in `pixtreme::transport::video`, with explicit time-base
numerator and denominator arguments.

## License and third-party notices

The video module is licensed under the
[MIT License](https://github.com/sync-dev-org/pixtreme-transport-video/blob/main/LICENSE).
The Python bindings incorporate nanobind (BSD-3-Clause) and robin-map (MIT).
Their copyright notices and license texts are in `nanobind.txt` and
`robin_map.txt`; `third-party.json` records their versions, source checksums and
build descriptions. These files are included in the wheel's license directory;
the `License-File` fields in its metadata give the relative paths. The SDK
includes the module license and the same license files and provenance record.
It supplies the native module without the Python bindings or their embedded
third-party implementations. The separately
installed pixtreme, pixtreme-transport and codec companion dependencies carry
their own licenses and notices.
