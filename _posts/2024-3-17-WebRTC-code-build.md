# WebRTC code build for Windows 10

# Download

First of all, set http proxy and git proxy correctly.
Download code branch 6264
```
$ mkdir webrtc-checkout
$ cd webrtc-checkout
$ fetch --nohooks webrtc
$ gclient sync
```
# Build
For Windows platform, WebRTC does not support MSVC compiler by default, please use clang with new build command.
If you build with Visual Studio 2022(MSVC), many strange compile errors occur.

```
$ gn gen out/x64 "--args=target_os=\"win\" target_cpu=\"x64\" is_debug=true treat_warnings_as_errors=false"
$ autoninja -C out/x64
```

# Reference doc
[1] https://webrtc.googlesource.com/src/+/main/docs/native-code/development/
