# Installation of FFmpeg

## With CentOS 7, you can install the GPG Key and Repository with the following: 
```
rpm --import http://li.nux.ro/download/nux/RPM-GPG-KEY-nux.ro rpm -Uvh http://li.nux.ro/download/nux/dextop/el7/x86_64/nux-dextop-release-0-5.el7.nux.noarch.rpm
```

## With CentOS 6, the same can be accomplished with the following:
```
rpm --import http://li.nux.ro/download/nux/RPM-GPG-KEY-nux.ro rpm -Uvh http://li.nux.ro/download/nux/dextop/el6/x86_64/nux-dextop-release-0-2.el6.nux.noarch.rpm
```

### yum install ffmpeg ffmpeg-devel -y


# Static Prebuilt Install of FFMpeg

```
wget https://raw.githubusercontent.com/q3aql/ffmpeg-install/master/ffmpeg-install
chmod a+x ffmpeg-install
./ffmpeg-install --install release
ffmpeg -version

```
# Manual Compile FFmpeg from source 
https://trac.ffmpeg.org/wiki/CompilationGuide/Centos

## Reverting Changes
```
rm -rf ~/ffmpeg_build ~/ffmpeg_sources ~/bin/{ffmpeg,ffprobe,ffserver,lame,nasm,vsyasm,x264,yasm,ytasm}
# yum erase autoconf automake bzip2 cmake freetype-devel gcc gcc-c++ git libtool mercurial zlib-devel
hash -r
```

EOF
