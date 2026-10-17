# Cameras

How Veyra talks to cameras and what has been verified.

## Backends

| Backend | Use it for | Needs | Verified against |
|---|---|---|---|
| **Simulated camera** | learning, demonstrations, tests | nothing | simulation; every inspection carries the simulated ground truth |
| **Image folder (replay)** | running a station on real images without a camera | a folder of images | automated tests |
| **USB camera or stream (OpenCV)** | UVC cameras and webcams; RTSP and HTTP streams | OpenCV (included) | a laptop's built-in UVC camera on Windows 11 (DirectShow): 640 × 480, 15 frames/s, controls read back |
| **Basler (pylon)** | Basler USB3 Vision and GigE Vision cameras | `pypylon` (an optional component), which brings Basler's pylon runtime and its licence | pylon's camera emulator, in the automated tests (discovery, frames, controls with read-back, software trigger); **not yet a physical Basler camera** |
| **GenICam / GenTL** | GigE Vision, USB3 Vision, CoaXPress and Camera Link cameras of most makers, through the vendor's GenTL producer (`.cti`) | `harvesters` (an optional component) and a GenTL producer | the simulated devices of the GenTL producer the `genicam` package ships, in the automated tests (discovery, frames, controls); Basler pylon 7's producers load on the reference PC; **not yet a physical camera** |

GenTL producers are found where camera vendors' installers register them (the folders in `GENICAM_GENTL64_PATH`), or
named in the camera settings. Pixel formats: Mono8 to Mono16 (kept as 8 bits, most significant first), the 8-bit Bayer
patterns (demosaiced), RGB8, BGR8 and their alpha variants. Simulated devices are marked as such wherever they appear.

Not implemented, and shown as such in the camera dialog:

- **Cognex** and **Keyence** through their own SDKs and protocols: proprietary and licensed separately, and their
  smart cameras are vision controllers rather than image sources. Their cameras that speak GigE Vision are reached
  through the GenICam backend with a GenTL producer.

## How a live camera behaves

- A reader thread keeps only the newest frame, as a free-running camera does. Frames the inspection could not take are
  counted as dropped; nothing piles up.
- A failed read is retried. After a silence (10 s by default; for a triggered camera, only after repeated errors) the
  camera is closed and reopened. Only if that fails does the camera's job fault: its tile says why, an event is
  recorded, and the other cameras keep inspecting. A failed acquisition never becomes an OK result.
- **Controls** are those the device reports. Values are set when the camera opens and read back: a value the device
  did not take is shown as *not applied*, with the value it chose. OpenCV's values are in the driver's own units
  (DirectShow exposure is a power of two in seconds, for example); pylon's are the camera's (exposure in µs).
- **Triggers** (pylon, GenICam with the standard trigger features): free running, software trigger, or hardware
  trigger on `Line1` (or another line).

## Setting up a camera

**Stations → Camera assignments → Add camera…** (or **Camera settings…** for an existing one):

1. **Find cameras** lists the USB cameras OpenCV can open (by index; OpenCV cannot read names), the Basler cameras
   pylon enumerates and the GenICam cameras of every GenTL producer found. Emulated and simulated cameras are marked.
2. Choose a source and set it up.
3. **Test** opens the camera in the runtime, takes frames, and shows one with the resolution, the measured frame rate,
   the time to open and to the first frame, and the controls. If it fails, it says at which step and why.
4. **Use this camera**, choose the pipeline it runs, and **Save assignments**.

**Check readiness…** on the Stations page tests every camera again (with the rest of the pre-start checklist) before
production starts.
