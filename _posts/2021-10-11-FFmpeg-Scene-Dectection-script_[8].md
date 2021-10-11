# [FFmpeg Scene Dectection script](https://github.com/yorkane/yorkane.github.io/issues/8)

# Mount NFS folder
```
mount -t nfs 192.168.1.204:/volume2/ariadata/ /mnt/cdata/
```


# yum install ffmpeg
```
ffmpeg -i SNIS-980.mp4 -vf select='gt(scene\,0.6)' -q:v 2 -vsync 0 -an %03d.jpg

ffmpeg -i input.flv  -filter:v "select='gt(scene,0.4)',showinfo" -q:v 2 -an %03d.jpg

ffmpeg -i miae-102.mp4 -vf fps=1,select='gt(scene\,0.6)'  -q:v 2 -vsync 0 -frame_pts 1 miae-102/z%d.jpg

ffmpeg -i test/3.mp4 -vf fps=1 -q:v 2 %5d.jpg #caputure one shot
```


# Compile & Install shotdetect
```
git clone https://github.com/yorkane/shotdetect.git
cd shotdetect
./complie
cp build/shotdetect-cmd /usr/sbin/shotdetect

shotdetect -i miae-102.mp4 -o ./ -s 120 -a miae-102 -y 2018 -r
```


```
ffmpeg -i e:\vrgf.mp4 -vcodec hevc -b:v 5000k -keyint_min 60 -g 60 -sc_threshold 0 e:\vrgf_compress1.mp4

ffmpeg -i onez-129.mp4 -vcodec hevc -s 1280x720  onez-129.hevc.mp4 

```

### Single TS file with range index
muxer will store all segments in a single MPEG-TS file, and will use byte ranges in the playlist.
```
ffmpeg -i input.ts -hls_flags single_file  -vcodec copy -acodec copy out.m3u8
```


```
#pip install cv2
pip install opencv-python
pip install imutils
python3 b.py --images ../../HEVC/video_name/scene/ --threshold 22
```