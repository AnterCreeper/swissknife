# 照片胶片日期水印(ffmpeg)

给照片右下角叠加胶片机风格的橙色日期戳，字体为 7 段数码管（本仓库附 `MTC-7-Segment.ttf`)。

```bash
ffmpeg -i IMG_2665-6.jpg -filter_complex \
  "color=c=black@0.0:s=5776x3852,format=rgba,\
drawtext=fontsize=170:fontfile=MTC-7-Segment.ttf:text='24 04 28':fontcolor=#ff7515@0.9:x=4700:y=3400,boxblur=2[vf];\
[0:v][vf]overlay=alpha=1" \
  -frames:v 1 result.jpg
```

要点：
- `s` 设为照片分辨率，`x:y` 控制日期位置（右下），`fontsize` 按比例调整
- `fontcolor=#ff7515@0.9` 是胶片机典型的橙色半透明
- `boxblur=2` 让字体边缘轻微柔化，更像胶片效果

(merged from dateoverlay)
