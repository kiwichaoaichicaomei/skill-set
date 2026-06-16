# PDF 文档优化示例

本文档展示 `pdf-optimizer` Skill 在实际场景中的优化效果，涵盖印刷报告、屏幕分发、扫描文档数字化三种典型类型。

---

## 示例 1：年度财务报告（印刷+屏幕双用途）

### 📄 原始状态

-   **文件大小**：28.5 MB | **页数**：86 页
-   **问题现象**：
    -   浏览器打开需等待 8 秒才显示首页
    -   Windows 以外系统打开 P12/P34/P56 出现乱码
    -   前 10 页为扫描件，Ctrl+F 无法搜索
    -   印刷厂反馈颜色偏差严重

### 🔍 诊断结果

| 问题 | 严重程度 | 页码 |
|------|----------|------|
| 字体未嵌入（SimSun） | 🔴 高 | P12, P34, P56 |
| 缺少 OCR 文本层 | 🔴 高 | P1-P10 |
| 未启用 Fast Web View | 🔴 高 | 全文 |
| 图片 DPI=600（屏幕用途过剩） | 🟡 中 | P20-P45 |
| 颜色空间混用（RGB/CMYK） | 🟡 中 | P15, P28 |
| 书签层级错乱 | 🟡 中 | 全书 |
| Keywords 元数据缺失 | 🟢 低 | 文档属性 |
| 3 个失效超链接 | 🟢 低 | P42, P67, P81 |

### ✨ 优化方案与命令

**1. 嵌入所有字体**
```bash
gs -dPDFSETTINGS=/prepress \
   -dEmbedAllFonts=true \
   -dSubsetFonts=true \
   -sOutputFile=step1_fonts.pdf \
   input.pdf
```

**2. 添加 OCR 文本层（跳过已有文本的页面）**
```bash
ocrmypdf --skip-text \
         --deskew \
         --clean \
         -l chi_sim+eng \
         step1_fonts.pdf step2_ocr.pdf
```

**3. 降采样 + 统一色彩空间 + 线性化**
```bash
gs -dPDFSETTINGS=/ebook \
   -dDownsampleColorImages=true \
   -dColorImageResolution=150 \
   -dGrayImageResolution=150 \
   -dConvertType1ToType1C=true \
   -dUseCIEColor=true \
   -sOutputFile=final_optimized.pdf \
   step2_ocr.pdf

# 启用 Fast Web View
qpdf --linearize final_optimized.pdf final_web.pdf
```

**4. 修复书签层级（Acrobat 手动或 pdftk）**
```bash
# 导出书签
pdftk final_web.pdf dump_data output metadata.txt
# 编辑 metadata.txt 调整 BookmarkLevel
# 重新导入
pdftk final_web.pdf update_info metadata.txt output final_fixed.pdf
```

### 📊 优化后效果

| 指标 | 优化前 | 优化后 | 改善 |
|------|--------|--------|------|
| 文件大小 | 28.5 MB | 14.2 MB | **-50%** |
| 首屏加载 | 8.2s | 1.1s | **-87%** |
| 可搜索页面 | 76/86 | 86/86 | **+10 页** |
| 跨平台显示 | ❌ 3 页乱码 | ✅ 全部正常 | — |
| 印刷色差 | ❌ 明显 | ✅ ΔE<2 | — |

💡 *备注：Ghostscript `-dPDFSETTINGS=/ebook` 预设已包含 150dpi 降采样与 Flate 压缩；若印刷用途需保留 300dpi，改用 `/prepress` 并单独设置 `-dColorImageResolution=300`；ocrmypdf 的 `--skip-text` 避免对已有文本层重复 OCR。*

---

## 示例 2：政府公文归档（PDF/A-2b 合规）

### 📄 原始状态

-   **文件来源**：Word 导出 PDF
-   **合规检查**：VeraPDF 报告 12 项违规
-   **主要问题**：
    -   使用了透明效果（PDF/A 禁止）
    -   2 个字体未嵌入
    -   文档语言未声明
    -   包含 JavaScript 动作
    -   元数据缺少 CreationDate

### 🔍 诊断结果

| 问题 | 严重程度 | PDF/A 条款 |
|------|----------|-----------|
| 透明效果 | 🔴 高 | ISO 19005-2:2011 §6.4 |
| 字体未嵌入 | 🔴 高 | ISO 19005-2:2011 §6.2.3 |
| JavaScript 动作 | 🔴 高 | ISO 19005-2:2011 §6.9 |
| 文档语言缺失 | 🟡 中 | ISO 19005-2:2011 §6.7.2 |
| 元数据不完整 | 🟡 中 | ISO 19005-2:2011 §6.7.3 |
| RGB 颜色空间 | 🟢 低 | 建议转 DeviceCMYK/sRGB |

### ✨ 优化方案

**1. 移除 JavaScript 与透明效果（Acrobat Pro）**
-   工具 → 操作向导 → 创建新操作 → 添加"删除 JavaScript"+"展平透明度"
-   或使用 Ghostscript 展平：`-dNoTransparency`

**2. 嵌入字体 + 补全元数据**
```bash
gs -dPDFA=2 \
   -dPDFSETTINGS=/prepress \
   -dEmbedAllFonts=true \
   -sOutputICCProfile=srgb.icc \
   -c ".setpdfmark << /Title (2026年度工作报告) /Author (XX局办公室) /CreationDate (D:20260616) /Lang (zh-CN) >> setpdfmark" \
   -f input.pdf \
   -sOutputFile=pdfa_compliant.pdf
```

**3. 验证合规性**
```bash
verapdf --format text pdfa_compliant.pdf
# 期望输出：PASS
```

### 📊 优化后效果

| 检查项 | 优化前 | 优化后 |
|--------|--------|--------|
| VeraPDF 结果 | ❌ FAIL (12 errors) | ✅ PASS |
| 字体嵌入 | 2 个缺失 | 全部嵌入 |
| JavaScript | 存在 | 已移除 |
| 文档语言 | 未声明 | zh-CN |
| 文件大小 | 3.2 MB | 3.8 MB (+19%) |

💡 *备注：PDF/A 合规通常会增加文件体积（因嵌入字体、移除压缩特性），这是正常代价；Ghostscript `-dPDFA=2` 自动处理大部分转换，但复杂排版仍需 Acrobat 预检；务必用 VeraPDF 做最终验证，不可仅依赖生成工具的报告。*

---

## 示例 3：扫描合同数字化（OCR + 无障碍）

### 📄 原始状态

-   **文件类型**：纯扫描图片 PDF（无文本层）
-   **页数**：15 页 | **大小**：18 MB
-   **问题**：
    -   无法搜索/复制文字
    -   页面倾斜 2-3°
    -   背景有扫描污渍
    -   屏幕阅读器完全无法读取

### 🔍 诊断结果

| 问题 | 严重程度 | 位置 |
|------|----------|------|
| 无 OCR 文本层 | 🔴 高 | 全文 |
| 页面倾斜 | 🟡 中 | P3, P7, P12 |
| 背景污渍 | 🟡 中 | P1-P15 |
| 无 Tagged PDF 结构 | 🔴 高 | 全文 |
| 图片无 Alt Text | 🔴 高 | 全文 |
| 文档语言未声明 | 🟡 中 | 文档属性 |

### ✨ 优化方案

**1. OCR + 纠偏 + 去污**
```bash
ocrmypdf --deskew \
         --clean \
         --remove-background \
         --skip-text \
         -l chi_sim \
         --output-type pdfa \
         scanned_contract.pdf ocr_clean.pdf
```

**2. 添加语义标签与 Alt Text（Acrobat Pro）**
-   工具 → 辅助工具 → 自动标记文档
-   手动校正表格/签名区域的标签
-   为印章/签名图片添加 Alt Text："甲方公章"、"乙方签字"

**3. 设置文档语言与阅读顺序**
```bash
# 使用 qpdf 补充元数据
qpdf --replace-input \
     --add-attachment lang.xml \
     ocr_clean.pdf tagged_contract.pdf
```
或在 Acrobat 中：文件 → 属性 → 高级 → 语言设为"中文（简体）"

### 📊 优化后效果

| 指标 | 优化前 | 优化后 |
|------|--------|--------|
| 可搜索文字 | ❌ 0% | ✅ 98%（手写体除外） |
| 页面倾斜 | 2-3° | <0.5° |
| 背景干净度 | 有明显污渍 | 纯净白底 |
| 屏幕阅读器 | ❌ 无法读取 | ✅ 逐段播报 |
| PDF/A 合规 | ❌ | ✅ PDF/A-2b |
| 文件大小 | 18 MB | 12.5 MB (-31%) |

💡 *备注：ocrmypdf `--output-type pdfa` 一步到位生成合规归档文件；手写体识别率有限，建议关键条款人工校对；Tagged PDF 的自动标记准确率约 70%，合同类文档必须人工校正表格与签名区域；Alt Text 应简洁描述内容而非外观（如"甲方公章"而非"红色圆形印章"）。*

---

## 使用技巧

### ✅ 获得最佳效果的建议

1.  **明确用途优先**：告诉 Copilot "这是印刷用/屏幕用/归档用"，优化策略完全不同
2.  **提供合规要求**：如"必须符合 PDF/A-2b""需要 WCAG AA"，Skill 会针对性检查
3.  **描述具体痛点**：如"打印颜色不对""手机打开很慢""盲人同事读不了"，便于精准定位
4.  **告知可用工具**：如"只有 Acrobat Reader""可以装命令行工具"，避免推荐无法执行的方案
5.  **大文件分步处理**：超过 100MB 的 PDF 建议先拆分章节，优化后再合并

### ❌ 常见误区

-   **不要只说"压缩 PDF"**：未说明用途可能导致印刷质量损失或归档不合规
-   **不要跳过验证**：PDF/A 合规必须用 VeraPDF 验证，生成工具的自报告不可靠
-   **不要忽略备份**：PDF 优化是不可逆操作，务必保留原始文件
-   **不要期望 OCR 100% 准确**：扫描件识别总有误差，关键内容必须人工校对
-   **不要用 Word 另存为 PDF/A**：Word 导出的 PDF/A 常隐藏违规项，务必用专业工具验证

---

**更多示例**：欢迎在社区分享你的优化案例，优秀案例将被收录至官方示例库。
