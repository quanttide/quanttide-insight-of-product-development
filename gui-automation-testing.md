# 交互验证（GUI 自动化测试方法）

此 build 的交互能力有限（仅底部 3 个 tab 可点击），但为后续自动化测试提供了可复用的方法。

## 工具链

| 工具 | 用途 |
|------|------|
| `xvfb-run` / `Xvfb` | 虚拟显示器，无头环境运行 GUI 应用 |
| `wmctrl` | 查询窗口列表、几何位置、激活窗口 |
| `xdotool` | 鼠标点击、键盘输入、窗口聚焦（X11） |
| `ydotool` / `wtype` | Wayland 下替代 xdotool |
| `import` (ImageMagick) | 截取窗口或全屏截图 |
| `tesseract` (chi_sim) | OCR 提取界面文字，定位 UI 元素 |
| `convert` (ImageMagick) | 像素颜色探测，辅助定位导航区域 |
| `opencv-python-headless` | OpenCV 模板匹配 |

## 操作流程

### 循环

```
首次运行 → OCR 定位文字 → 截图裁剪保存为模板
日常运行 → 模板匹配（命中则用，未命中则 OCR 重定位并更新缓存）
         → xdotool 点击
         → 截图 + OCR 对比验证
```

## 关键步骤

1. **启动**：`Xvfb :99` 创建虚拟桌面，`DISPLAY=:99 ./bin/studio/qtcloud_course_studio` 启动
2. **截图**：`import -window $WID /tmp/shot.png` 截取指定窗口
3. **OS 交互**：`xdotool`（X11）或 `ydotool` / `wtype`（Wayland）
4. **验证**：再次截图 + OCR，对比前后文字判断页面是否切换

## 精确定位点击的方法

### 方法一：首屏自动定位 + 模板缓存（推荐）

首次运行时用 OCR 找到 UI 元素位置，截图保存为模板。后续运行直接用模板匹配：

```bash
# 自定位 + 点击函数
smart_click() {
  local label=$1       # 目标文字（如"课程研发"）
  local window=$2
  local cache_dir="${HOME}/.cache/widget-snaps"
  local template="${cache_dir}/${label}.png"

  mkdir -p "$cache_dir"
  import -window "$window" /tmp/shot.png

  # 模板匹配
  if [ -f "$template" ]; then
    local match=$(python3 -c "
import cv2, numpy as np, sys
shot = cv2.imread('/tmp/shot.png')
tmpl = cv2.imread('$template')
h, w = tmpl.shape[:2]
res = cv2.matchTemplate(shot, tmpl, cv2.TM_CCOEFF_NORMED)
_, max_val, _, max_loc = cv2.minMaxLoc(res)
if max_val > 0.7:
    cx, cy = max_loc[0] + w//2, max_loc[1] + h//2
    print(f'{cx} {cy}')
" 2>/dev/null)

    if [ -n "$match" ]; then
      xdotool mousemove --window "$window" $match click 1
      return 0
    fi
  fi

  # 回退：OCR 定位并更新缓存
  tesseract /tmp/shot.png /tmp/shot -l chi_sim --psm 6 tsv 2>/dev/null
  local pos=$(awk -F'\t' -v label="$label" '
    $1==5 && $11!="-1" && index($12, label) {
      cx=int($7+$9/2); cy=int($8+$10/2)
      print cx, cy
      # 同时保存区域坐标用于生成模板
      print $7, $8, $9, $10 > "/tmp/bbox"
    }' /tmp/shot.tsv | head -1)

  if [ -z "$pos" ]; then
    echo "❌ 找不到: $label"
    return 1
  fi

  # 裁剪保存为模板
  read left top width height < /tmp/bbox
  convert /tmp/shot.png -crop "${width}x${height}+${left}+${top}" "$template"
  echo "✅ 模板已缓存: $template"

  xdotool mousemove --window "$window" $pos click 1
}
```

使用：

```bash
wmctrl -l  # 找窗口 ID
WID=$(wmctrl -l | grep "课程" | awk '{print $1}')

smart_click "课程研发" $WID
sleep 1
import -window $WID /tmp/after.png
tesseract /tmp/after.png stdout -l chi_sim 2>/dev/null | grep "研发" \
  && echo "✅ 导航成功" || echo "❌ 导航失败"
```

### 方法二：OCR bounding box

手动定位，适合首次探索或临时使用：

```bash
# 1. 截图并 OCR 输出 TSV（含每个字的坐标）
import -window $WID /tmp/shot.png
tesseract /tmp/shot.png /tmp/shot -l chi_sim --psm 6 tsv

# 2. 解析目标文字的中心坐标
# TSV 列: level page block par line word left top width height conf text
# word-level (level=5) 的 left/top/width/height 是像素坐标
awk -F'\t' '$1==5 && $11!="-1" {printf "%s  center=(%d,%d)\n", $12, $7+$9/2, $8+$10/2}' /tmp/shot.tsv

# 3. 点击该坐标（窗口相对坐标）
xdotool mousemove --window $WID 384 675 click 1
```

### 方法三：像素颜色探测

应急调试用，通过颜色变化找到 UI 区域边界：

```bash
# 垂直扫描底部，找到导航栏的起始 y
for y in $(seq 700 10 819); do
  c=$(convert /tmp/shot.png -crop 1x1+666+$y -format "%[hex:p{0,0}]" info:)
  echo "y=$y color=#$c"
done
# 找到颜色突变的位置，那就是导航栏边界

# 水平三等分，点击对应 tab
NAV_Y=<探测到的导航栏 Y 中心>
xdotool mousemove --window $WID $((W/2)) $NAV_Y click 1
```

### 验证点击是否生效

```bash
# 点击前截图 + OCR 作为基线
import -window $WID /tmp/before.png
tesseract /tmp/before.png /tmp/before -l chi_sim --psm 6

# 执行点击
xdotool mousemove --window $WID $X $Y click 1
sleep 1

# 点击后截图 + OCR
import -window $WID /tmp/after.png
tesseract /tmp/after.png /tmp/after -l chi_sim --psm 6

# 对比文字变化
diff /tmp/before.txt /tmp/after.txt && echo "页面未变化" || echo "页面已切换"
```

## 三种定位方式对比

| 维度 | OCR bounding box | 像素颜色探测 | 模板匹配 + 自定位缓存 |
|------|-----------------|-------------|---------------------|
| **速度** | 慢（tesseract 全图 OCR） | 快 | 快（首次中，后续快） |
| **抗 UI 变动** | 文字不变即可 | ❌ 颜色一变就崩 | ✅ 相似度容忍细微变化 |
| **跨 session** | ✅ 文字通用 | ❌ 颜色可能变 | ✅ 缓存自动更新 |
| **维护成本** | 中（需解析 TSV） | 高（每区域手动调参） | **低**（首次自动缓存） |
| **依赖** | `tesseract` + 语言包 | ImageMagick | `opencv-python-headless` |
| **定位粒度** | 单字级别 | 像素级别 | **区域级别**（整块 UI） |
| **适合场景** | 首次探索、无模板 | 应急探测 UI 边界 | **日常自动化** |

## 置信度阈值参考

模板匹配的相似度阈值因 UI 元素而异：

| UI 类型 | 推荐阈值 | 说明 |
|---------|---------|------|
| 大块按钮（>80×40） | 0.90 | 特征丰富，匹配准 |
| 导航 tab | 0.80 | 有文字有背景色 |
| 小图标（<32×32） | 0.60 | 特征少，需降低阈值 |
| 透明/圆角元素 | 0.65 | 背景混入会降低评分 |

可在 `smart_click` 中加 `--threshold` 参数覆盖默认值。

## Wayland 支持

xdotool 基于 X11，Wayland 下不工作：

| 环境 | 截图 | 鼠标 | 键盘 |
|------|------|------|------|
| X11 | `import` | `xdotool` | `xdotool` |
| Wayland | `grim` / `slurp` | `ydotool` | `wtype` |
| 跨平台 | `gnome-screenshot` | `ydotool`（支持 X11+Wayland） | `ydotool` |

`ydotool` 是更好的基础选择（同时支持 X11 和 Wayland），只是需要系统服务 `ydotoold`。

## 已知限制

- `xdotool` 使用**窗口相对坐标**（`--window` 参数），需精确计算 UI 元素位置
- 窗口管理器装饰（标题栏、边框）会影响坐标计算，需通过 `wmctrl -lG` 获取准确几何
- tesseract 中文字 OCR 在复杂背景或小字号时准确率下降
- Flutter 应用可能不响应模拟的鼠标/键盘事件（取决于 build 类型）
- xdotool 基于 X11，Wayland 下需改用 ydotool/wtype
