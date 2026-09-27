# Frame rate drops from 144 to 72 FPS

## Symptom

On my laptop (i7-12700H, GeForce RTX 3060 Laptop 6 GB, Iris Xe, 144 Hz VRR panel), the game starts at 144 FPS, then drops to 72 FPS and mostly stays there. The Windows power mode is set to "Best performance".

72 FPS is 13.9 ms, exactly two refreshes at 144 Hz: every frame waits one extra refresh.

I want to understand why.

## Measurements

### PresentMon

I discovered [PresentMon](https://github.com/GameTechDev/PresentMon/releases/tag/v2.6.0) thanks to this issue :D

Command (game already running, 30 s capture):

```powershell
.\PresentMon.exe --process_name MegaProject.exe --output_file c:\tmp\presentmon.csv --timed 30 --terminate_after_timed
```

Raw capture: [framerate drops/presentmon.csv](framerate%20drops/presentmon.csv)

Summary per second:

| Seconds | FPS | GPU time per frame (`MsGPUTime`) | Present mode |
|---|---|---|---|
| 0 to 1 | 144 | ~1.8 ms | Composed: Flip |
| 2 to 8 | 72 | ~5.75 ms | Composed: Flip |
| 9 to 19 | 144, with short dips | ~1.8 to 2.1 ms | Hardware Composed: Independent Flip |
| 20 to 24 | 72 | ~5.75 ms | Hardware Composed: Independent Flip |

- The present mode is not the cause: the drop happens in both modes.
- The GPU time per frame switches between ~1.8 ms and ~5.75 ms, and the 72 FPS phases match the slow level. This turned out to be a consequence of the drop, not its cause (see below).
- At that time, `MsGPUWait` was ~0.01 ms and `MsGPULatency` almost the whole frame: the CPU and the GPU worked one after the other.

### nvidia-smi

I also discovered [this tool](https://docs.nvidia.com/deploy/nvidia-smi/index.html)!

Command (run in another terminal while the game runs, stopped with Ctrl+C):

```powershell
nvidia-smi --query-gpu=timestamp,pstate,clocks.gr,clocks.mem,utilization.gpu,power.draw --format=csv -lms 250 -f c:\tmp\nvsmi.csv
```

Raw capture: [framerate drops/nvsmi.csv](framerate%20drops/nvsmi.csv)

| Phase | Power state | Graphics clock | Memory clock | GPU utilization |
|---|---|---|---|---|
| 144 FPS | P0 | 1282 MHz | 6001 MHz | ~20 % |
| 72 FPS | P5 (sometimes P3) | 975 down to 427 MHz | 810 MHz | ~33 to 40 % |

- The game renders on the RTX 3060.
- In P5 the memory clock is divided by 7, which explains the GPU time going from 1.8 ms to 5.75 ms.
- The driver lowers the clocks **because** the game produces fewer frames: this is an effect of the drop.

### Engine timing logs

Temporary timing logs (removed since) measured, once per second, the average and max of each step of the frame loop on `features/FramesInFlight` (CPU and GPU already working in parallel), at 1280x720.

Raw log: [framerate drops/framestats.txt](framerate%20drops/framestats.txt)

| | 144 FPS | 72 FPS |
|---|---|---|
| Wait for the swap chain (frame latency waitable object) | 5.1 ms | **11.0 ms** |
| Wait for the GPU (back buffer fence) | 0.001 ms | 0.001 ms |
| One `UpdateScene` | 0.77 ms | 0.77 ms |
| Updates per frame | 1.39 | 2.78 |
| Recording the frame | 0.30 ms | 0.33 ms |
| `Present` | 0.31 ms | 0.36 ms |
| CPU work per frame | ~1.7 ms | ~2.8 ms |

- The CPU work stays under 3 ms per frame and one update costs the same in both phases: the CPU is not the cause.
- The engine never waits for the GPU: the NVIDIA GPU is not the cause.
- The engine waits 11 ms per frame for the swap chain to accept a new frame. That wait is a symptom: it fills whatever time remains until the display side takes the next frame. The display side only takes one frame every 13.9 ms.

### Which GPU drives the screen

```powershell
nvidia-smi --query-gpu=name,display_active,display_mode --format=csv
Get-CimInstance -Namespace root\wmi -ClassName WmiMonitorConnectionParams | Select-Object InstanceName, VideoOutputTechnology
```

- `display_active` is `Disabled`: the RTX 3060 drives no screen.
- The panel (`SHP154D`) uses an internal connection (`VideoOutputTechnology` 0x80000000), driven by the Intel Iris Xe.

## Leads ruled out

| Test | Result |
|---|---|
| Windows power mode "Best performance" | still drops to 72 |
| NVIDIA Control Panel, "Power management mode: Prefer maximum performance" | still drops |
| Rendering at 1280x720 instead of 1920x1080 | still drops |
| Game limited to the P-cores (Task Manager affinity, CPUs 0 to 11) | still drops |
| Fixed 144 Hz instead of "Dynamic" refresh rate (Windows 11 DRR) | still drops |
| Continuous mouse and keyboard input while at 72 | still drops |
| Intel display power features disabled | still drops |
| Frames in flight (this branch) | still drops |
| **Display mode "NVIDIA GPU only"** | **holds 144 FPS** |

First hypothesis, disproved: the NVIDIA driver lowering its clocks (P0 to P5) while the engine serialized CPU and GPU work. The timing logs showed the engine never waits for the GPU, and the clock drop is a consequence of the lower frame rate.

## Conclusion

The laptop runs in hybrid mode ([NVIDIA Optimus](https://fr.wikipedia.org/wiki/Optimus_(NVIDIA))).

The RTX 3060 renders the frames, but the internal screen is driven by the Intel Iris Xe. Each frame is copied from the NVIDIA to the Intel chip, which puts it on screen. After a few seconds, this step only accepts one frame every two refreshes: 72 FPS instead of 144. Neither the CPU nor the NVIDIA GPU is the bottleneck, and the engine cannot measure or control that step.

**Fix:** set the display mode to **"NVIDIA GPU only"** (NVIDIA Control Panel > Manage Display Mode, or the laptop manufacturer's utility). The RTX 3060 then drives the screen directly, and the game holds 144 FPS.

The "frames in flight" changes of this branch do not fix this issue, but they are kept: the CPU and the GPU now work in parallel instead of waiting for each other twice per frame (see [Frames in flight changes](#frames-in-flight-changes-kept)).

## Detect the flaw in-game

Claude gave me a plan I should check in the future. The main idea is to check whether the screen showing my window is physically connected to the GPU we used to render with.

The steps:

1. Identify the rendering GPU (Get its LUID, using DXGI I guess?)
2. Same thing for the screen
3. List the outputs of the rendering GPU
4. Compare all the results

Pointers given by claude:

- ID3D12Device::GetAdapterLuid: the LUID of the device's adapter.
- IDXGIFactory4::EnumAdapterByLuid: to get the adapter back from its LUID.
- IDXGIAdapter::EnumOutputs: to list an adapter's outputs, and how the end of the list is signaled.
- IDXGIOutput::GetDesc and the DXGI_OUTPUT_DESC structure: look at the Monitor field.
- MonitorFromWindow, which you already use in MegaDisplay.cpp.
- To understand the context: Microsoft's pages on hybrid systems / cross-adapter presentation, and NVIDIA's documentation on Optimus and Advanced Optimus (which switches display mode on the fly).

I'll check it in the future, if required
