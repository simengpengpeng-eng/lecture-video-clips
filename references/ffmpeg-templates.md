# ffmpeg 命令模板（讲座视频切片）

以下 `$FF` 指 ffmpeg 可执行文件路径，`SRC` 指源视频，`T0` 指片头结束时刻（秒，精确到 0.1），`A0/A1` 指话题起止时刻。编码参数统一内联：
`-c:v libx264 -crf 18 -preset medium -pix_fmt yuv420p -c:a aac -b:a 160k -movflags +faststart`

## 1. 横屏切片（保持源分辨率）

片头(0–T0) + 话题(A0–A1)，话题时长 = 90 − T0：

```bash
$FF -hide_banner -loglevel error -i "$SRC" -filter_complex "\
[0:v]split=2[vA][vB];[0:a]asplit=2[aA][aB];\
[vA]trim=0:$T0,setpts=PTS-STARTPTS[v0];[aA]atrim=0:$T0,asetpts=PTS-STARTPTS[a0];\
[vB]trim=$A0:$A1,setpts=PTS-STARTPTS[v1];[aB]atrim=$A0:$A1,asetpts=PTS-STARTPTS[a1];\
[v0][a0][v1][a1]concat=n=2:v=1:a=1[v][a]" \
-map "[v]" -map "[a]" \
-c:v libx264 -crf 18 -preset medium -pix_fmt yuv420p -c:a aac -b:a 160k -movflags +faststart \
"输出-切片N-横屏.mp4" -y
```

## 2. 横屏切片（低分辨率源放大到 1280×720）

在 concat 后加 `[vc][ac];[vc]scale=1280:720:flags=lanczos[v]`，并把 `-map "[ac]"`：

```bash
...concat=n=2:v=1:a=1[vc][ac];[vc]scale=1280:720:flags=lanczos[v]" \
-map "[v]" -map "[ac]" ...
```

## 3. 竖屏切片（1080×1920，完整画面居中，logo 完整）

```bash
$FF -hide_banner -loglevel error -i "$SRC" -filter_complex "\
[0:v]split=2[vA][vB];[0:a]asplit=2[aA][aB];\
[vA]trim=0:$T0,setpts=PTS-STARTPTS[v0];[aA]atrim=0:$T0,asetpts=PTS-STARTPTS[a0];\
[vB]trim=$A0:$A1,setpts=PTS-STARTPTS[v1];[aB]atrim=$A0:$A1,asetpts=PTS-STARTPTS[a1];\
[v0][a0][v1][a1]concat=n=2:v=1:a=1[cv][ac];\
[cv]split=2[bg][fg];\
[bg]scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920,boxblur=luma_radius=20:luma_power=2[bgm];\
[fg]scale=1080:-2:flags=lanczos[fgs];\
[bgm][fgs]overlay=(W-w)/2:(H-h)/2[v];[ac]anull[a]" \
-map "[v]" -map "[a]" \
-c:v libx264 -crf 18 -preset medium -pix_fmt yuv420p -c:a aac -b:a 160k -movflags +faststart \
"输出-切片N-竖屏.mp4" -y
```

要点：
- `concat` 必须输出两个标签 `[cv][ac]`，音频经 `[ac]anull[a]` 旁路，漏写会报 `Invalid stream specifier`。
- 前景 `scale=1080:-2` **不裁剪**，保证画面角落的台标/logo 完整；高度非偶数时 ffmpeg 自动取偶。
- 若源高度使前景超过 1080×1920 可容纳高度（1920），源高宽比 > 16:9 时需另行缩小前景。

## 4. 封面

```bash
# 横屏封面（标题页中点时刻 MID）
$FF -hide_banner -loglevel error -ss $MID -i "$SRC" -frames:v 1 -q:v 2 "封面-横屏.jpg" -y

# 竖屏封面（同帧模糊背景+居中）
$FF -hide_banner -loglevel error -ss $MID -i "$SRC" -filter_complex "\
[0:v]split=2[bg][fg];\
[bg]scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920,boxblur=luma_radius=20:luma_power=2[bgm];\
[fg]scale=1080:-2[fgs];\
[bgm][fgs]overlay=(W-w)/2:(H-h)/2" \
-frames:v 1 -q:v 2 "封面-竖屏.jpg" -y
```

## 5. 辅助命令

```bash
# 读时长/分辨率（无 ffprobe 时）
$FF -hide_banner -i "$SRC" 2>&1 | grep -E "Duration|Stream.*(Video|Audio)"

# 无损提取音轨
$FF -hide_banner -loglevel error -i "$SRC" -vn -c:a copy audio.m4a -y

# 抽帧（-q:v 2 高质量封面 / 3 普通检查帧）
$FF -hide_banner -loglevel error -ss $T -i "$SRC" -frames:v 1 -q:v 3 frame.jpg -y

# 剪切点校验小片段（无损音频）
$FF -hide_banner -loglevel error -ss $X -t 12 -i audio.m4a -c:a copy probe/p.m4a -y

# 静音检测（定位停顿点）
$FF -hide_banner -ss $X -t $LEN -i audio.m4a -af "silencedetect=noise=-32dB:d=0.45" -f null - 2>&1 | grep -oE "silence_(start|end): [0-9.]+"

# 校验成片时长/分辨率
$FF -hide_banner -i 成片.mp4 2>&1 | grep -E "Duration|Stream.*Video"
```

## 6. 归档

```bash
# 按切片建文件夹：{编号姓名}切片{N}/ 内含横屏、竖屏 mp4 + 两张封面 jpg
cd 成品目录
for f in *-切片*-横屏.mp4; do
  base="${f%-切片*-横屏.mp4}"
  num=$(echo "$f" | sed -E 's/.*-切片([0-9]+)-横屏\.mp4/\1/')
  dir="${base}切片${num}"
  mkdir -p "$dir"
  mv -f "${base}-切片${num}-横屏.mp4" "${base}-切片${num}-竖屏.mp4" "$dir"/
  for o in 横屏 竖屏; do
    [ -f "${base}-封面-${o}.jpg" ] && cp -f "${base}-封面-${o}.jpg" "$dir"/
  done
done
rm -f *-封面-*.jpg   # 根目录副本已复制进文件夹后清理
```
