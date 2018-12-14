# ffmpeg-ffnvcodec-explanation
How to get ffmpeg to export the needed symbols on (K)ubuntu cosmic 18.10 so OBS and MPV can use NVENC and NVDEC on Fermi, Maxwell, Kepler, Pascal, Volta and Turing architectures and how to use hardware accelerated chromium
https://developer.nvidia.com/video-encode-decode-gpu-support-matrix

**0. Get Nvidia's proprietary driver:**
```
sudo apt-get install ppa-purge # for safety
sudo add-apt-repository ppa:graphics-drivers/ppa
# please check the support plan for your GPU – nvidia-driver-{390,396,410,415} available so your display server doesn't fail!
sudo apt-get install nvidia-driver-415
```

**1. Install the nv-codec-headers package:**

```
sudo apt-get install make git
mkdir ~/devel/ && cd ~/devel/
git clone https://git.videolan.org/git/ffmpeg/nv-codec-headers.git
cd nv-codec-headers
make && sudo make install
```
```
FFmpeg version of headers required to interface with Nvidias codec APIs.

Corresponds to Video Codec SDK version 8.2.15.

Minimum required driver versions:
Linux: 396.24 or newer
Windows: 397.93 or newer

Optional CUDA 10 features:
Linux: 410.48 or newer
Windows: 411.31 or newer
```

**2. Compile FFmpeg:**

ffmpeg will automatically detect the ffnvcodec-headers — extract from `./configure --help`:
```
  The following libraries provide various hardware acceleration features:
  --disable-amf            disable AMF video encoding code [autodetect]
  --disable-audiotoolbox   disable Apple AudioToolbox code [autodetect]
  --enable-cuda-sdk        enable CUDA features that require the CUDA SDK [no]
  --disable-cuvid          disable Nvidia CUVID support [autodetect]
  --disable-d3d11va        disable Microsoft Direct3D 11 video acceleration code [autodetect]
  --disable-dxva2          disable Microsoft DirectX 9 video acceleration code [autodetect]
  --disable-ffnvcodec      disable dynamically linked Nvidia code [autodetect]
  --enable-libdrm          enable DRM code (Linux) [no]
  --enable-libmfx          enable Intel MediaSDK (AKA Quick Sync Video) code via libmfx [no]
  --enable-libnpp          enable Nvidia Performance Primitives-based code [no]
  --enable-mmal            enable Broadcom Multi-Media Abstraction Layer (Raspberry Pi) via MMAL [no]
  --disable-nvdec          disable Nvidia video decoding acceleration (via hwaccel) [autodetect]
  --disable-nvenc          disable Nvidia video encoding code [autodetect]
  --enable-omx             enable OpenMAX IL code [no]
  --enable-omx-rpi         enable OpenMAX IL code for Raspberry Pi [no]
  --enable-rkmpp           enable Rockchip Media Process Platform code [no]
  --disable-v4l2-m2m       disable V4L2 mem2mem code [autodetect]
  --disable-vaapi          disable Video Acceleration API (mainly Unix/Intel) code [autodetect]
  --disable-vdpau          disable Nvidia Video Decode and Presentation API for Unix code [autodetect]
  --disable-videotoolbox   disable VideoToolbox code [autodetect]
```

Should the standard compilation not fit your needs (you have the need to link in specific libraries/dont need some libraries or you want to enable/disable specific features) then you can change the build-rules in `~/devel/ffmpeg/ffmpeg-4.0.2/debian/rules`
```
sudo apt-get build-dep ffmpeg
mkdir -p ~/devel/ffmpeg
cd ~/devel/ffmpeg
sudo apt-get source ffmpeg
cd ffmpeg-4.0.2 # cd ffmpeg-x.x.x [x.x.x represents the version number]
debuild -us -uc -b
cd ..
rm libavcodec58* # creates package conflicts - if ffmpeg complains about library configuration mismatches don't worry, it's not broken
# you can: alias ffmpeg='ffmpeg -hide_banner' if you think your terminal gets too cluttered with debug messages
# put it in your ~/.bashrc to make it permanent
sudo dpkg -i *.deb
sudo apt-mark hold ffmpeg libavcodec-dev libavcodec-extra58 libavfilter-dev libavformat58 libavresample-dev libavutil-dev libavutil56
```
Should `debuild` fail make sure the files are "yours" and dont belong to user `root` -> `sudo chown -hR $USER:$USER *` in the ffmpeg source-tree

- **Testing:**
```
$ ffmpeg -hide_banner -encoders  | grep nvenc
 V..... h264_nvenc           NVIDIA NVENC H.264 encoder (codec h264)
 V..... nvenc                NVIDIA NVENC H.264 encoder (codec h264)
 V..... nvenc_h264           NVIDIA NVENC H.264 encoder (codec h264)
 V..... nvenc_hevc           NVIDIA NVENC hevc encoder (codec hevc)
 V..... hevc_nvenc           NVIDIA NVENC hevc encoder (codec hevc)
$ ffmpeg -hide_banner -decoders | grep cuvid
 V..... h264_cuvid           Nvidia CUVID H264 decoder (codec h264)
 V..... hevc_cuvid           Nvidia CUVID HEVC decoder (codec hevc)
 V..... mjpeg_cuvid          Nvidia CUVID MJPEG decoder (codec mjpeg)
 V..... mpeg1_cuvid          Nvidia CUVID MPEG1VIDEO decoder (codec mpeg1video)
 V..... mpeg2_cuvid          Nvidia CUVID MPEG2VIDEO decoder (codec mpeg2video)
 V..... mpeg4_cuvid          Nvidia CUVID MPEG4 decoder (codec mpeg4)
 V..... vc1_cuvid            Nvidia CUVID VC1 decoder (codec vc1)
 V..... vp8_cuvid            Nvidia CUVID VP8 decoder (codec vp8)
 V..... vp9_cuvid            Nvidia CUVID VP9 decoder (codec vp9) 
 $ ffmpeg -hide_banner -hwaccels
Hardware acceleration methods:
vdpau
cuda
vaapi
drm
cuvid
```

**3. Install OBS (no need to compile)** 
```
sudo add-apt-repository ppa:obsproject/obs-studio
sudo apt-get update
sudo apt-get install obs-studio
```


**4. Build MPV to use NVDEC for video decoding**
```
mkdir -p ~/devel/mpv
cd ~/devel/mpv
sudo apt-get source mpv
cd mpv-0.29.0 # cd mpv-x.x.x [x.x.x represents version]
debuild -us -uc -b
cd ..
sudo dpkg -i mpv*.deb # we dont need libmpv{-dev}
sudo apt-mark hold mpv
...
mpv --hwdec=nvdec input
```
**5. Use hardware-acceleration enabled chromium**
Thanks to the author (Saikrishna Arcot) who patched chromium against vaapi (you'll need the vdpau-va-driver)
https://www.linuxuprising.com/2018/08/how-to-enable-hardware-accelerated.html
```
sudo add-apt-repository ppa:saiarcot895/chromium-dev
sudo apt-get update
sudo apt install vdpau-va-driver chromium-browser vdpauinfo vainfo
```
Then in chromium type in the adressbar `chrome://flags/#enable-accelerated-video` and enable it and maybe zero-copy too etc. 
Check out `chrome://gpu` or `about:gpu` to see what configuration works best for you (e.g. on my system out of process rasterization doesn't work well and GpuMemoryBuffers are only implemented in software)
Install the h264ify extension https://chrome.google.com/webstore/detail/h264ify/aleakchihdccplidncghkekgioiakgal
(even you now have a system that could offload VP9 decoding to the video card it is not implemented in any browser)

- **Checking if chromium actually uses the video card**

Go to `chrome://media-internals` or `about:media-internals` when h264ify is enabled, play a youtube file and click on the box `video_decoder GpuVideoDecoder` should be the status - if it is `FFmpegVideoDecoder` or `VpxVideoDecoder` you have an error somewhere.


