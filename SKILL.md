---
agent_created: true
summary: "将扫描版 PDF 的译文烧录为可见中文文字层，同时保留原书图照与排版。"
read_when:
  - 用户要求把 PDF 译文变成「完全可见」的中文
  - 在扫描版 PDF 上叠加中文，且不破坏原书布局
---

# visible-zh-pdf-overlay

把已有中文译文覆盖到扫描版 PDF 上，俄文（或其他原文）被纸色遮罩隐藏，中文真实可选/可复制，图照、线稿、页眉页脚保留。

## 前置条件

- PyMuPDF (`fitz`) 已安装。
- 源文件为「可搜索 PDF」：每页是扫描图 + 隐形原文文字层（可用 Tesseract hOCR/fitz 生成）。
- 中文译文按页组织，例如 `【第0页】...【第1页】...`。
- 准备一款中文字体 TTF（推荐宋/黑；简体译本用 `HYBaoSongJ.ttf`，繁体才用 `HYBaoSongF.ttf`）。

## 核心流程

1. 读取 `<book>_searchable.pdf` 和每页中文译文。
2. 对每页：
   - 用 `page.get_text("words")` 取得原文单词坐标。
   - 过滤照片噪点：只保留含西里尔字母或数字的词（跳过拉丁噪点）。
   - 按 y 坐标聚合成行，再按行间距聚合成段落块。
   - 用纸色（取页面四边中位数）填充这些段落块 → 隐藏原文。
   - 按块顺序将中文折行填入，字号自动适应块高。
3. 保存 `<book>_zh.pdf`。

## 关键脚本

见 `D:\ocr_work\translate\build_zh_visible.py`（随用例演化）。主要参数：

```python
FONT = r"D:\Documents\ChemistryEqProject\字体\HYBaoSongJ.ttf"  # 简体；繁体改用 HYBaoSongF.ttf
page.insert_font(fontname="zh", fontfile=FONT)
page.draw_rect(rect, color=bg, fill=bg, width=0)       # 遮罩原文
page.insert_text((x, y), line, fontname="zh", fontsize=size,
                 render_mode=0, color=(0.08, 0.08, 0.08))  # 可见中文
```

## 注意事项

- `render_mode=3` 是隐形文字层；要可见必须用 `render_mode=0`。
- 不要用 `page.get_text("blocks")` 直接遮罩：OCR 噪点块会把整张照片盖住。应基于 `words` 重新算行/段落块。
- 纸色必须从页面边缘采样，不要用全页中位数（大照片会拉偏颜色）。
- 纯图页没有文字块时，回退到页面底部 18% 的横条放中文图注。
- **字体坑（必看）**：简体译本务必用「简体字体」。某些字体（如汉仪 HYBaoSongF＝报宋繁）只有简体码点，却把 `为/页/国` 等画成繁体字形，视觉上变繁体。判定方法：用 fontTools 取 NotoSerifCJK **SC** 子字体（idx 2）作简体参照、同文件 **TC**(idx 3) 作繁体参照，对目标字体逐字渲染成 64×64 掩膜比对——哪个参照更接近即该字体偏向哪种。`D:\ocr_work\translate\check_font.py` 可复用。已确认 `HYBaoSongJ.ttf`(报宋简) 为简体，用其即可。
- 改字体后同名覆盖 PDF，阅读器会缓存旧版；交付时提醒用户关闭重开。
