# 交互验证（GUI 自动化测试方法）

此 build 的交互能力有限（仅底部 3 个 tab 可点击），但为后续自动化测试提供了可复用的方法。

## 工具链

| 工具 | 用途 |
|------|------|
| `xvfb-run` / `Xvfb` | 虚拟显示器，无头环境运行 GUI 应用 |
| `wmctrl` | 查询窗口列表、几何位置、激活窗口 |
| `xdotool` | 鼠标点击、键盘输入、窗口聚焦 |
| `import` (ImageMagick) | 截取窗口或全屏截图 |
| `tesseract` (chi_sim) | OCR 提取界面文字，定位 UI 元素 |
| `convert` (ImageMagick) | 像素颜色探测，辅助定位导航区域 |

## 操作流程

```
启动应用 → 截图 → OCR → 解析文字位置 → xdotool 点击 → 再截图 → OCR 对比变化
```

## 关键步骤

1. **启动**：`Xvfb :99` 创建虚拟桌面，`DISPLAY=:99 ./bin/studio/qtcloud_course_studio` 启动
2. **截图**：`import -window $WID /tmp/shot.png` 截取指定窗口
3. **OCR 定位**：`tesseract shot.png stdout -l chi_sim --psm 6 tsv` 输出 TSV 格式含 bounding box
4. **颜色探测**：`convert shot.png -crop 1x1+X+Y -format "%[hex:p{0,0}]" info:` 获取像素颜色，辅助定位导航栏等区域
5. **点击**：`xdotool mousemove --window $WID $X $Y click 1` 在窗口相对坐标点击
6. **验证**：再次截图 + OCR，对比前后文字判断页面是否切换

## 精确定位点击的方法

### 方法一：OCR bounding box（推荐）

```bash
# 1. 截图并 OCR 输出 TSV（含每个字的坐标）
import -window $WID /tmp/shot.png
tesseract /tmp/shot.png /tmp/shot -l chi_sim --psm 6 tsv

# 2. 解析目标文字的中心坐标
# TSV 列: level page block par line word left top width height conf text
# word-level (level=5) 的 left/top/width/height 是像素坐标
awk -F'\t' '$1==5 && $11!="-1" {printf "%s  center=(%d,%d)\n", $12, $7+$9/2, $8+$10/2}' /tmp/shot.tsv

# 示例输出:
# 课程  center=(384,669)
# 研发  center=(385,681)

# 3. 点击该坐标（窗口相对坐标）
xdotool mousemove --window $WID 384 675 click 1
```

### 方法二：像素颜色探测

先通过颜色变化找到 UI 区域边界，再在区域内点击：

```bash
# 垂直扫描底部，找到导航栏的起始 y
for y in $(seq 700 10 819); do
  c=$(convert /tmp/shot.png -crop 1x1+666+$y -format "%[hex:p{0,0}]" info:)
  echo "y=$y color=#$c"
done
# 找到颜色突变的位置，那就是导航栏边界

# 水平三等分，点击对应 tab
# 窗口宽度 W，tab 中心在 W/6, W/2, 5W/6
NAV_Y=<探测到的导航栏 Y 中心>
xdotool mousemove --window $WID $((W/2)) $NAV_Y click 1   # 课程研发
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

## 已知限制

- `xdotool` 使用**窗口相对坐标**（`--window` 参数），需精确计算 UI 元素位置
- 窗口管理器装饰（标题栏、边框）会影响坐标计算，需通过 `wmctrl -lG` 获取准确几何
- tesseract 中文字 OCR 在复杂背景或小字号时准确率下降
- Flutter 应用可能不响应模拟的鼠标/键盘事件（取决于 build 类型）
