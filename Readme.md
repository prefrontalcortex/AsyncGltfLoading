# Async KTX2 Texture Loading for Vulkan

This repository brings together the parts needed to load KTX2 files inside glTF asynchronously on Vulkan (e.g. Quest 2).  

**Current status (20221219):**
- Texture loading works on Windows with Vulkan backend
  - there are native crashes when loading many textures or things happening simultaneously
  - there are issues with colorspaces (see ["Known Issues"](#known-issues) below)
- Android not tested yet

## Prerequisites

- make sure all submodules are pulled
- scripting define KTX_UNITY_GPU_UPLOAD needs to be set
- render backend in the Editor needs to be set to Vulkan

## Building ktx_unity.dll

- open packages/KTX-Software-Unity
- run 
  ```
  cmake . -G "Visual Studio 16 2019" -A x64 -Bbuild_win_64 -DKTX_UNITY_FEATURE_VULKAN=ON -DKTX_UNITY_FEATURE_OPENGL=ON
  cmake --build build_win_64 --config MinSizeRel --target ktx_unity
  ```
- per-platform commands can be found in `.github/workflows/main.yaml` (make sure to add KTX_UNITY_FEATURE_VULAN=ON as above)
- close Unity, replace the ktx_unity.dll file or other platform-specific artifacts, open Unity again

You can also run Unity directly after building:
- in build_win_64, open ALL_BUILD.vxproj with Visual Studio
- set ktx_unity as Startup Project
- make sure to set MinSizeRel or Debug as targets
- rightclick > Build on ktx_unity
- in ktx_unity properties, set General Properties > Output Directory to `..\..\KtxUnity\Runtime\Plugins\x86_64`
- in Debugging settings, set Unity with the right project to launch, e.g. 
```
command: D:\Program Files\Unity\Hub\Editor\2020.3.42f1\Editor\Unity.exe
arguments: -projectPath "H:\git\AsyncGltfLoading\projects\Async-2020.3" -logFile "H:\git\AsyncGltfLoading\projects\Async-2020.3/Logs/UnityEditor.log"
```

This setup will allow pressing Build in Visual Studio, which replaces the DLL and launches Unity. This also allows setting breakpoints and such.

## Developing

- native code changes done so far are in packages/KTX-Software-Unity/src/KtxUnityPlugin.cpp:~140, see the submodule git history
- this uses existing libktx functionality to upload asynchronously
- the relevant C# code is on KtxNativeInstance.cs:~118 and ~133

## Testing

- open the glTFLoad scene
- press Play
- duplicate the disabled "glTF Asset" object and enable the copy
- this will now load via KTXUnity
- you can put more stress on the system (to check for race conditions) by duplicating many copies of the disabled "glTF Asset" and enabling them all at the same time.
- e.g. when loading 10 or so at the same time, the exception shown below occurs sometimes.

To reproduce a crash:
- press Play
- duplicate 10 glTF Asset, turn them on
- will most likely work
- duplicate 10 more from the disabled one, turn them on
- will most likely crash with an Access Violation

## Known Issues

- in KtxNativeInstance.cs:108 a creation mode can be set (creating the texture inside Unity, then updating from native, or creating it directly in native) - CreateThenUpdate runs into Unity bug 
   > IN-19024 [Windows][Vulkan] Scrambled texture objects are rendered when setting one with Texture2D.UpdateExternalTexture
   > Internal ID: UUM-20405
   > Issue Tracker: https://issuetracker.unity3d.com/issues/windows-vulkan-scrambled-texture-objects-are-rendered-when-setting-one-with-texture2d-dot-updateexternaltexture


- color space doesn't seem to be respected
   >  IN-19022 - [Graphics API] CreateExternalTexture ignores the linear/sRGB parameter, always interpreted as linear

- unclear occasional crashes, most likely due to lacking native understanding on my end
   > 
    ```cpp
    ========== OUTPUTTING STACK TRACE ==================
    0x00007FF96D740CB9 (ktx_unity) [H:\git\thirdparty\KTX-Software-Unity\KTX-Software\lib\texture.c:529] ktxTexture_GetElementSize 
    0x00007FF96D74D06E (ktx_unity) [H:\git\thirdparty\KTX-Software-Unity\KTX-Software\lib\vkloader.c:816] ktxTexture_VkUploadEx 
    0x00007FF96D7376C5 (ktx_unity) [H:\git\thirdparty\KTX-Software-Unity\src\KtxUnityPlugin.cpp:139] OnRenderEvent 
    0x00007FF63B7BE803 (Unity) GfxDeviceVK::InsertCustomMarkerCallbackAndData
    0x00007FF63B7BE5AC (Unity) GfxDeviceVK::InsertCustomMarkerCallback
    0x00007FF63D71AD67 (Unity) GfxDeviceWorker::RunCommand
    0x00007FF63D7211AD (Unity) GfxDeviceWorker::RunExt
    0x00007FF63D7212C8 (Unity) GfxDeviceWorker::RunGfxDeviceWorker
    0x00007FF63BD59196 (Unity) Thread::RunThreadWrapper
    0x00007FFA237F54E0 (KERNEL32) BaseThreadInitThunk
    0x00007FFA2434485B (ntdll) RtlUserThreadStart
    ========== END OF STACKTRACE ===========
    ```