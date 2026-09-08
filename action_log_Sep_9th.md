# Action Log — Sep 9th

## Session Summary

Exploration session: got the `open-dibr` demo running end-to-end on the current
dev machine. Required three separate bugfixes — a CUDA 13.3 API breaking change,
a silenced error-check macro, and a multi-GPU OpenGL/CUDA interop mismatch. After
fixes the Fan dataset demo runs with full 6DoF interaction.

---

## 1. CUDA 13.3 API breaking change — `cuCtxCreate`

`cuCtxCreate` was renamed/versioned to `cuCtxCreate_v4` in CUDA 13.3, which changed
the signature from 3 arguments to 4:

```
// old (CUDA ≤ 12.x)
cuCtxCreate(CUcontext*, unsigned int flags, CUdevice dev)

// new (CUDA 13.3)
cuCtxCreate(CUcontext*, CUctxCreateParams*, unsigned int flags, CUdevice dev)
```

The second parameter (`CUctxCreateParams*`) can be `NULL` to get default behaviour.

Two call sites fixed:

- `src/AppDecUtils.h` line 61:
  ```cpp
  ck(cuCtxCreate(cuContext, NULL, flags, cuDevice));
  ```
- `src/Application.h` line 584:
  ```cpp
  ck(cuCtxCreate(cuContext, NULL, 0, cuDevice));
  ```

Build verified with CUDA 13.3 at
`C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v13.3\`.

---

## 2. Silenced CUresult error-check macro

`src/NvCodecUtils.h` had the `CUresult` overload of `check()` commented out
(lines 36–46). Because all CUDA driver API calls go through the `ck()` macro
which resolves to `check()`, every `cuGraphicsGLRegisterImage`, `cuvidMapVideoFrame`,
etc. was silently returning `false` on failure with no output — making crashes look
like random segfaults rather than meaningful CUDA errors.

Re-enabled the block:

```cpp
#ifdef __cuda_cuda_h__
inline bool check(CUresult e, int iLine, const char *szFile) {
    if (e != CUDA_SUCCESS) {
        const char *szErrName = NULL;
        cuGetErrorName(e, &szErrName);
        std::cout << "CUDA driver API error " << szErrName
                  << " at line " << iLine << " in file " << szFile << std::endl;
        return false;
    }
    return true;
}
#endif
```

After this, `CUDA_ERROR_OPERATING_SYSTEM` on `cuGraphicsGLRegisterImage` became
visible, exposing the root cause of the crash.

---

## 3. Multi-GPU CUDA-GL interop fix — `NvOptimusEnablement`

**Root cause:** The machine has two GPUs — Intel UHD 770 (iGPU, handles display on
the primary monitor via integrated DP) and NVIDIA RTX 5070. Without an explicit
hint, Windows was starting the OpenGL context on the Intel iGPU while CUDA
initialised on the RTX 5070. CUDA-GL interop (`cuGraphicsGLRegisterImage`) requires
both to be on the same GPU, hence `CUDA_ERROR_OPERATING_SYSTEM`.

**Fix:** Added the standard NVIDIA and AMD Optimus export symbols to `src/main.cpp`:

```cpp
extern "C" { __declspec(dllexport) unsigned long NvOptimusEnablement = 1; }
extern "C" { __declspec(dllexport) int AmdPowerXpressRequestHighPerformance = 1; }
```

These are read by the NVIDIA driver at process start-up and force the OpenGL
context onto the discrete GPU, putting both GL and CUDA on the RTX 5070.

After rebuild the GL renderer string confirmed `NVIDIA GeForce RTX 5070/PCIe/SSE2`,
`cuGLGetDevices` returned count=1, and `cuGraphicsGLRegisterImage` succeeded. The
demo ran to exit code 0.

---

## Demo tested

**Dataset:** `examples/Fan/` — InterDigital content from the MPEG-I Visual test
sequence library. 4 colour + 4 depth H.264 MP4 streams, 1920×1080, perspective
projection, cameras separated by ~20 cm baseline.

**JSON config used:** `examples/Fan/example_omaf.json`

**Run command:**
```
bin\Release\RealtimeDIBR.exe examples\Fan\example_omaf.json
```

6DoF keyboard+mouse navigation confirmed working.

---

## OpenDIBR input format — notes for future custom datasets

The renderer requires per-frame **depth videos** alongside colour videos. A plain
RGB video without depth cannot be used directly. Realistic paths for custom content:

- RGB-D cameras (Intel RealSense, Azure Kinect) — depth out of the box
- Monocular depth estimation (e.g. Depth Anything V2) → encode depth frames as
  16-bit grayscale MP4
- COLMAP for camera calibration; use `example_colmap.json` as JSON template

Each camera entry in the JSON needs accurate intrinsics (`Focal`, `Principle_point`)
and extrinsics (`Position`, `Rotation`) in addition to file paths — incorrect
calibration produces warping/ghosting artefacts.

---

## Files changed

**`open-dibr` repo:**
- `src/main.cpp` — added `NvOptimusEnablement` / `AmdPowerXpressRequestHighPerformance` exports
- `src/NvCodecUtils.h` — re-enabled `CUresult` check overload
- `src/AppDecUtils.h` — fixed `cuCtxCreate` 4-arg form (CUDA 13.3)
- `src/Application.h` — fixed `cuCtxCreate` 4-arg form (CUDA 13.3)

---

## Next steps

1. Rebuild the `open-dibr` project on a clean machine to confirm the four fixes
   are sufficient with no other local environment dependencies.
2. Evaluate monocular depth estimation pipeline (Depth Anything V2 or similar) for
   generating depth videos from regular camera footage, enabling custom dataset use.
3. Consider upstreaming the CUDA 13.3 and `NvOptimusEnablement` fixes to the
   IDLabMedia/open-dibr repository as a PR.
