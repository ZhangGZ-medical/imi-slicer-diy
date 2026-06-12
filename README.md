# imi-slicer-diy

IMI 立体定向导针规划数据 → 3D Slicer 可视化代码全流程自动化工具。

将一对 CSV 文件（轨迹坐标 + deposit 参数）一键转化为 3D Slicer 可导入的 Python 代码和 Excel 中间数据。

## 流水线概览

```
{case}_trajs.csv ───┬──→ [阶段2] Slicer 轨迹 Line 代码 (.py + .md)
                    │
                    ├──→ [阶段1] CSV 拆分 → {case}_trajN.xlsx (×3)
                    │         │
                    │         └──→ [阶段3] compute_deposits.py 坐标回填
                    │                   │
                    │                   └──→ [阶段4] Slicer Deposit Point List (.py + .md × 3)
                    │
{case}_imi.csv  ────┘
```

## 四个阶段

| 阶段 | 功能 | 产出 |
|------|------|------|
| **1. CSV 拆分** | 按人工指定区间将 45 个 deposit 拆为 3 组 | `{case}_traj1.xlsx` / `traj2.xlsx` / `traj3.xlsx` |
| **2. 轨迹 Line** | 从 CSV 生成 3 条导针轨迹的 Slicer Markups Line 代码 | `slicer_import_{case}_trajs.py` / `.md` |
| **3. 坐标回填** | 计算每个 deposit 的 RAS 三维坐标，回填 xlsx | 更新后的 xlsx（R/A/S 行已填充） |
| **4. Deposit 点** | 为每条轨迹生成独立的 Fiducial Point List 代码 | 每 traj 一对 `.py` + `.md`，共 6 个文件 |

## 最终交付文件

```
slicer_import_{case}_trajs.py          ← 轨迹 Line 代码
slicer_import_{case}_trajs.md          ← 轨迹 Line 文档（含坐标表）
slicer_import_{case}_traj1_deposits.py ← traj1 Deposit Point List
slicer_import_{case}_traj1_deposits.md
slicer_import_{case}_traj2_deposits.py ← traj2 Deposit Point List
slicer_import_{case}_traj2_deposits.md
slicer_import_{case}_traj3_deposits.py ← traj3 Deposit Point List
slicer_import_{case}_traj3_deposits.md
{case}_traj1.xlsx                      ← 已回填坐标
{case}_traj2.xlsx
{case}_traj3.xlsx
```

## Slicer 可视化规范

### 轨迹线 (Markups Line)
- 类型：`vtkMRMLMarkupsLineNode`
- 每轨迹 1 个节点，2 个控制点（entry → target）
- Glyph：`Vertex2D`
- 颜色：traj1 红 `[1,0,0]` / traj2 绿 `[0,1,0]` / traj3 蓝 `[0,0,1]`
- TextScale：`0.0`
- 控制点和节点均锁定

### Deposit 点 (Fiducial Point List)
- 类型：`vtkMRMLMarkupsFiducialNode`
- 每 traj 1 个节点，所有 deposit 点集中管理
- Glyph：`StarBurst`
- 颜色：与对应轨迹线一致（红/绿/蓝）
- TextScale：`0.0`
- 节点锁定

## 坐标系

- **系统**：RAS (Right / Anterior / Superior)
- **单位**：mm

## 触发关键词

`IMI全流程` `IMI转Slicer` `slicer全流程` `完整拆分导入` `CSV到Slicer` `一键生成Slicer代码` `imi_slicer`

## 依赖

- Python 3.x
- openpyxl
- 子技能：`imi-csv-diy`（CSV 拆分脚本）
- 计算脚本：`compute_deposits.py`（坐标变换与回填）
