# GSI Fixes

## Android12+ SkiaGL Fix
Android 12 switched the default rendering backend from GLES to SkiaGL. This prevents the tablet from booting. One solution is to switch back to GLES (setting `debug.renderengine.backend` to `gles`).

Another solution is to update the GPU libs. The ones available [here](./Android12+%20SkiaGL%20Fix/) are the libs that need to be updated. Their version is `UM S 5.59` and were grabbed from [this repo](https://github.com/khadas/android_vendor_rockchip_common/tree/khadas-edge2-android12/gpu/libG6110/G6110_64/vendor/lib/hw).

## Android12+ Codecs Fix
On Android 12 and higher media playback is broken (videos don't play). This can be fixed by updating the VPU libs. The ones available [here](./Android12+%20Codecs%20Fix/) are the libs that need to be updated and were grabbed from [this repo](https://github.com/khadas/android_vendor_rockchip_common/tree/khadas-edge2-android13/vpu).