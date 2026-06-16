# Excel 表格优化示例

本文档展示 `excel-optimizer` Skill 在实际场景中的优化效果，涵盖月度报表、数据台账、分析仪表盘三种典型类型。

---

## 示例 1：月度销售报表优化

### 📄 原文片段（Sheet1!D2:D500）

**原公式**：
```excel
=VLOOKUP(A2,Data!$A:$D,4,0)
```

**问题现象**：
-   D 列 500 行中有 37 个单元格显示 `#N/A`
-   下游汇总公式 `=SUM(D:D)` 返回错误值
-   每次修改 A 列数据，全表重算耗时 8 秒

### 🔍 诊断结果

| 问题 | 严重程度 | 位置 |
|------|----------|------|
| VLOOKUP 未处理匹配失败 | 🔴 高 | Sheet1!D2:D500 |
| 整列引用导致性能瓶颈 | 🔴 高 | Sheet1!D2:D500 |
| 未使用结构化引用 | 🟡 中 | Sheet1!D2:D500 |
| 无数据验证防止无效输入 | 🟢 低 | Sheet1!A2:A500 |

### ✨ 优化后

**新公式**（Excel 365/2021/WPS 2023+）：
```excel
=XLOOKUP(A2, Data!A:A, Data!D:D, "未匹配", 0)
```

**降级方案**（Excel 2019 及以下 / WPS 2022）：
```excel
=IFERROR(VLOOKUP(A2, Data!$A$2:$D$1000, 4, FALSE), "未匹配")
```

**配套操作**：
1.  将 `Data!$A$2:$D$1000` 转换为 Table（Ctrl+T），命名为 `SalesData`
2.  公式改为结构化引用：`=XLOOKUP([@ProductID], SalesData[ProductID], SalesData[Amount], "未匹配")`
3.  为 A 列添加数据验证：允许"列表"，来源 `=SalesData[ProductID]`

**优化收益**：
-   错误值从 37 个降为 0，汇总公式正常返回 ¥1,285,000
-   计算耗时从 8s 降至 0.6s（提升 92%）
-   新增产品 ID 时自动下拉选择，杜绝拼写错误

💡 *备注：XLOOKUP 第 4 参数为默认返回值，无需嵌套 IFERROR；Table 结构化引用在插入新行时自动扩展，无需手动调整范围；WPS 个人版 2022 不支持 XLOOKUP，需使用降级方案。*

---

## 示例 2：客户信息台账优化

### 📄 原文片段（Sheet3!A1:E200）

**原始结构**：
| A | B | C | D | E |
|---|---|---|---|---|
| 客户信息（合并 A1:E1） | | | | |
| 姓名 | 电话 | 地址 | 备注 | |
| 张三 | 138**** | 杭州市... | VIP | |
| （空行） | | | | |
| 李四 | 139**** | 上海市... | | |

### 🔍 诊断结果

| 问题 | 严重程度 | 位置 |
|------|----------|------|
| 表头合并单元格 | 🔴 高 | Sheet3!A1:E1 |
| 存在空行破坏连续性 | 🔴 高 | Sheet3!A4:E4 |
| 电话号码为文本型数字 | 🟡 中 | Sheet3!B2:B200 |
| 无数据验证 | 🟡 中 | Sheet3!B2:B200 |
| 隐藏列 F-H 未清理 | 🟢 低 | Sheet3!F:H |

### ✨ 优化后

**新结构**：
| CustomerName | Phone | Address | Tag | CreatedAt |
|--------------|-------|---------|-----|-----------|
| 张三 | 13800138000 | 杭州市西湖区... | VIP | 2026-01-15 |
| 李四 | 13900139000 | 上海市浦东新区... | 普通 | 2026-02-20 |

**配套操作**：
1.  取消 A1:E1 合并，删除空行，填充标题至每列
2.  选中 B 列 → 数据 → 分列 → 固定宽度 → 完成，将文本型数字转为数值型
3.  为 Phone 列添加数据验证：自定义公式 `=AND(LEN(B2)=11, ISNUMBER(B2))`，错误提示"请输入 11 位手机号"
4.  删除隐藏列 F:H，全选 → Ctrl+G → 定位条件 → 空值 → 删除整行
5.  转换为 Table（Ctrl+T），命名 `CustomerDB`

**优化收益**：
-   数据透视表可正常识别所有字段，筛选/排序功能恢复
-   电话号码可参与 COUNTIFS/VLOOKUP 等数值匹配
-   文件大小从 2.3MB 降至 890KB（减少 61%）
-   新增记录时自动继承格式与验证规则

💡 *备注：合并单元格是 Excel 数据分析的头号反模式，务必优先修复；文本型数字常见于系统导出文件，分列转换是最安全的修复方式；Table 命名建议使用英文+驼峰，避免中文空格导致的引用错误。*

---

## 示例 3：销售分析仪表盘优化

### 📄 原文片段（Dashboard 工作表）

**原始状态**：
-   打开文件耗时 45 秒
-   包含 8 条条件格式规则，颜色显示混乱
-   使用 INDIRECT("Sheet"&A1&"!B1") 动态引用
-   图表为饼图展示 12 个月趋势
-   无切片器，手动筛选下拉框

### 🔍 诊断结果

| 问题 | 严重程度 | 位置 |
|------|----------|------|
| 易失性函数导致全表重算 | 🔴 高 | Dashboard!A1 |
| 条件格式规则过多且冲突 | 🔴 高 | Dashboard!A5:F20 |
| 图表类型误用 | 🟡 中 | Chart1 |
| 缺少交互控件 | 🟡 中 | Dashboard |
| 未设置打印区域 | 🟢 低 | Dashboard |

### ✨ 优化后

**1. 替换易失性函数**
```excel
// 原公式
=INDIRECT("Sheet"&A1&"!B1")

// 新公式（INDEX + MATCH）
=INDEX(SheetList, MATCH(A1, SheetNames, 0), 2)
```
其中 `SheetList` 为预定义的二维数组常量或辅助表，避免运行时动态解析。

**2. 精简条件格式**
-   删除 5 条冗余规则，保留 3 条核心规则：
    -   销售额 > 目标值 → 绿色填充
    -   销售额 < 目标值 80% → 红色填充
    -   同比增长率 < 0 → 黄色字体
-   调整优先级：红 > 绿 > 黄，避免覆盖

**3. 更换图表类型**
-   饼图 → 带标记的折线图
-   添加趋势线（线性）
-   X 轴标签旋转 45°，Y 轴添加千分位分隔符
-   删除图例（单一系列无需图例），标题改为"2026 年月度销售趋势"

**4. 添加交互控件**
-   插入切片器：Region、ProductCategory
-   插入时间轴：OrderDate
-   关联所有数据透视表与图表

**5. 设置打印区域**
-   打印区域：A1:H30
-   页眉：左"销售分析仪表盘"，右"&D &T"
-   缩放：1 页宽 × 1 页高

**优化收益**：
-   打开耗时从 45s 降至 6s（提升 87%）
-   条件格式渲染流畅，无颜色闪烁
-   趋势变化一目了然，替代饼图的静态占比误导
-   用户可通过切片器自助筛选，减少重复制表需求
-   打印输出完整，无空白页

💡 *备注：INDIRECT 是性能杀手，凡是用它做动态引用的场景，90% 可用 INDEX-MATCH 或 CHOOSE 替代；条件格式规则数建议 ≤5，超过后考虑用辅助列+基础格式替代；切片器需基于 Table 或数据透视表，普通区域无法使用。*

---

## 示例 4：VBA 宏性能优化

### 📄 原代码（Module1）

```vba
Sub ProcessData()
    For i = 1 To 10000
        Cells(i, 1).Value = Cells(i, 1).Value * 1.1
        If Cells(i, 2).Value > 100 Then
            Cells(i, 3).Value = "High"
        Else
            Cells(i, 3).Value = "Low"
        End If
    Next i
End Sub
```

### 🔍 诊断结果

| 问题 | 严重程度 | 位置 |
|------|----------|------|
| 未关闭 ScreenUpdating | 🔴 高 | Module1 |
| 逐单元格读写（10000 次 COM 调用） | 🔴 高 | Module1 |
| 未声明变量类型 | 🟡 中 | Module1 |
| 未禁用事件触发 | 🟡 中 | Module1 |

### ✨ 优化后

```vba
Sub ProcessDataOptimized()
    ' 性能开关
    Application.ScreenUpdating = False
    Application.Calculation = xlCalculationManual
    Application.EnableEvents = False
    
    ' 批量读取到数组
    Dim dataRange As Range
    Set dataRange = ThisWorkbook.Sheets("Sheet1").Range("A1:C10000")
    Dim dataArray As Variant
    dataArray = dataRange.Value
    
    ' 内存中处理
    Dim i As Long
    For i = 1 To UBound(dataArray, 1)
        dataArray(i, 1) = dataArray(i, 1) * 1.1
        If dataArray(i, 2) > 100 Then
            dataArray(i, 3) = "High"
        Else
            dataArray(i, 3) = "Low"
        End If
    Next i
    
    ' 一次性写回
    dataRange.Value = dataArray
    
    ' 恢复环境
    Application.EnableEvents = True
    Application.Calculation = xlCalculationAutomatic
    Application.ScreenUpdating = True
    
    Set dataRange = Nothing
End Sub
```

**优化收益**：
-   执行时间从 28s 降至 0.4s（提升 98%）
-   屏幕无闪烁，用户体验流畅
-   避免触发其他工作表的 Change 事件导致连锁重算
-   显式声明 Long 类型，避免 Variant 隐式转换开销

💡 *备注：VBA 性能优化的黄金法则是"减少 COM 调用次数"，数组批量读写是最有效的手段；ScreenUpdating/Calculation/EnableEvents 三件套必须在退出前恢复，否则会导致 Excel 异常；Set obj = Nothing 虽非强制，但属于良好实践，避免内存泄漏。*

---

## 使用技巧

### ✅ 获得最佳效果的建议

1.  **明确表格用途**：告诉 Copilot "这是月度报表/数据台账/财务模型"，Skill 会自动调整优化策略
2.  **描述性能痛点**：如"打开要 1 分钟""公式经常报错""打印总是缺内容"，便于精准定位
3.  **提供使用者水平**：如"给不懂 Excel 的业务员用""给财务专家用"，公式复杂度和注释详细度会相应调整
4.  **标注 Excel 版本**：如"公司用的是 WPS 2022""只有 Excel 2016"，避免推荐不兼容的新函数
5.  **分表提交**：超过 10 个工作表的文件建议按业务模块分批提交，避免上下文丢失

### ❌ 常见误区

-   **不要只说"帮我改改 Excel"**：缺乏场景和痛点描述会导致优化方向偏差
-   **不要期望全自动重构**：Skill 提供建议与代码片段，结构性调整仍需人工确认
-   **不要忽略备份**：任何优化前务必备份，尤其是涉及公式重写和 VBA 修改
-   **不要跳过版本验证**：新函数在旧版本/WPS 中可能不可用，务必测试后再部署

---

**更多示例**：欢迎在社区分享你的优化案例，优秀案例将被收录至官方示例库。
