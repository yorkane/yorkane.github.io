# [Win CMD auto FFMPEG hevc converting](https://github.com/yorkane/yorkane.github.io/issues/9)

# Win CMD auto FFMPEG hevc converting
```
::指定起始文件夹
@echo off
set DIR="%cd%\115"
set num = 0
echo DIR=%DIR%

:: 参数 /R 表示需要遍历子文件夹,去掉表示不遍历子文件夹
:: %%f 是一个变量,类似于迭代器,但是这个变量只能由一个字母组成,前面带上%%
:: 括号中是通配符,可以指定后缀名,*.*表示所有文件
for /R %DIR% %%f in (*.mp4) do (
    echo/
    echo 开始处理 %%f
    if not exist %cd%HEVC\%%~nf\ (
        mkdir %cd%HEVC\%%~nf\
    )
        ffprobe  %cd%HEVC\%%~nf\%%~nf.hevc.mp4 -hide_banner > %cd%HEVC\%%~nf\%%~nf.hevc.mp4.txt 2>&1
    echo %cd%HEVC\%%~nf\%%~nf.hevc.mp4.txt 生成完毕
    if exist %cd%HEVC\%%~nf\%%~nf.source.txt (
        echo %cd%HEVC\%%~nf\%%~nf.source.txt 文件已存在 跳过处理
    ) else (
    echo ffmpeg -i %%f -vcodec hevc -r 20 -vf "scale=1280:720:force_original_aspect_ratio=decrease" %cd%HEVC\%%~nf\%%~nf.hevc.mp4  -hide_banner -y
    ffmpeg -i %%f -vcodec hevc -r 20 -vf "scale=1280:720:force_original_aspect_ratio=decrease" %cd%HEVC\%%~nf\%%~nf.hevc.mp4  -hide_banner -y
    echo compress over
    ffprobe %%f -hide_banner > %cd%HEVC\%%~nf\%%~nf.source.txt 2>&1
    echo %cd%HEVC\%%~nf\%%~nf.source.txt 生成完毕
    ffprobe %%f -hide_banner > %cd%HEVC\%%~nf\%%~nf.hevc.mp4.txt 2>&1
    echo %cd%HEVC\%%~nf\%%~nf.hevc.mp4.txt 生成完毕
    )
    if not exist %cd%HEVC\%%~nf\scene (
        mkdir  %cd%HEVC\%%~nf\scene\
        rem ffmpeg -i %%f -vf "select=gt(scene,0.3)" -q:v 2 -vsync 0 -frame_pts 1 %cd%HEVC\%%~nf\scene\f%d.jpg -hide_banner -y
    )
    if not exist %cd%HEVC\%%~nf\%%~nf.img.txt (
        echo ffmpeg -i %cd%HEVC\%%~nf\%%~nf.hevc.mp4 -vf fps=1,select="gt(scene\,0.3)" -q:v 2 -vsync 0 -frame_pts 1 %cd%HEVC\%%~nf\scene\^%%d.jpg -hide_banner -y
        ffmpeg -i %cd%HEVC\%%~nf\%%~nf.hevc.mp4 -vf fps=1,select="gt(scene\,0.3)" -q:v 2 -vsync 0 -frame_pts 1 %cd%HEVC\%%~nf\scene\^%%d.jpg -hide_banner -y
        echo 1>%cd%HEVC\%%~nf\%%~nf.img.txt
        echo %cd%HEVC\%%~nf\scene 图片已处理
        rem goto endp
    ) else (
        echo %cd%HEVC\%%~nf\%%~nf.img.txt 文件已存在 跳过场景截图处理
    )
)

:endp
pause

echo off
REM setlocal enabledelayedexpansion  
REM set "EXCEL_DIR=%cd%\excel"
REM for /R %EXCEL_DIR% %%f in (*.xls) do (
REM     set "FILE_PATH=%%f"
REM     echo 完整的路径: !FILE_PATH!
REM     set "FILE_DIR=%%~dpf"
REM     echo 所在的目录: !FILE_DIR!
REM     set "FILE_NAME=%%~nf"
REM     echo 简略文件名: !FILE_NAME!
REM     set "FILE_EXT=%%~xf"
REM     echo 文件后缀名: !FILE_EXT!
REM     set "FILE_FULLNAME=%%~nxf"
REM     echo 完整文件名: !FILE_FULLNAME!
REM     set "FILE_PATH_NO_EXT=%%~dpnf"
REM     echo 无后缀路径: !FILE_PATH_NO_EXT!
REM )
REM pause
```