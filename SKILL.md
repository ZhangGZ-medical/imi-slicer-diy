---
name: imi-slicer-diy
description: >
  IMI 导针 → 3D Slicer 全流程自动化技能。串联三个子技能：
  ① 将 {case}_trajs.csv + {case}_imi.csv 按人工指定区间拆分为 per-traj xlsx（imi-csv-diy）、
  ② 从 traj CSV 生成 Slicer Markups Line 代码（imi-trajs-diy）、
  ③ 计算 deposit RAS 坐标回填 xlsx 并生成 Slicer Fiducial Point List 代码（IMI_transform_diy）。
  触发词：IMI全流程、IMI转Slicer、slicer全流程、imi_slicer、完整拆分导入、CSV到Slicer。
agent_created: true
---

# imi-slicer-diy — IMI CSV → Slicer 全流程自动化

将一对 CSV 文件（`{case}_trajs.csv` + `{case}_imi.csv`）一次性转化为：
- 3 个 per-traj xlsx（坐标已回填）
- Slicer 轨迹 Line 代码（1 个 .py + 1 个 .md，3 条轨迹）
- Slicer deposit Point List 代码（每 traj 1 个 .py + 1 个 .md，共 6 个文件）

---

## 完整流水线

```
{case}_trajs.csv ───┬──→ [阶段2] Slicer 轨迹 Line 代码 (.py + .md)
                    │
                    ├──→ [阶段1] 拆分 → {case}_trajN.xlsx (×3)
                    │         │
                    │         └──→ [阶段3] compute_deposits.py 回填 R/A/S
                    │                   │
                    │                   └──→ [阶段4] Slicer deposit Point List (.py + .md × 3)
                    │
{case}_imi.csv  ────┘
```

---

## 阶段 1：CSV → xlsx 拆分（imi-csv-diy）

### 确认拆分区间

向用户确认 3 个 IMI deposit ID 区间。格式示例：`1-15→traj1, 16-30→traj2, 31-45→traj3`。

### 运行拆分

使用 `imi-csv-diy` 的拆分脚本（或直接用 openpyxl 等效实现）：

```bash
python ~/.workbuddy/skills/imi-csv-diy/scripts/split_csv_to_xlsx.py <case> <r1_s>,<r1_e> <r2_s>,<r2_e> <r3_s>,<r3_e>
```

产出：`{case}_traj1.xlsx`, `{case}_traj2.xlsx`, `{case}_traj3.xlsx`

---

## 阶段 2：轨迹 Line 代码（imi-trajs-diy）

从 CSV 读取 3 条轨迹的 entry/tip RAS 坐标，生成 Slicer Python Console 代码。

### 输出文件
- `slicer_import_{case}_trajs.py`
- `slicer_import_{case}_trajs.md`

### 代码规范

- 每个轨迹创建 1 个 `vtkMRMLMarkupsLineNode`
- 2 个控制点：`AddControlPoint(entry)` → `AddControlPoint(target)`
- **Glyph**：`Vertex2D`
- **颜色**：traj1=红 `[1,0,0]`、traj2=绿 `[0,1,0]`、traj3=蓝 `[0,0,1]`
- **TextScale**：`0.0`
- 锁定控制点和节点
- 每条轨迹外包 `try-except`

---

## 阶段 3：坐标计算与回填（IMI_transform_diy · 步骤 1）

对 3 个 xlsx 分别运行 `compute_deposits.py`：

```bash
python compute_deposits.py {case}_traj1.xlsx
python compute_deposits.py {case}_traj2.xlsx
python compute_deposits.py {case}_traj3.xlsx
```

每个 xlsx 的 deposits sheet R/A/S 行被回填为计算值。

---

## 阶段 4：Deposit Point List 代码（IMI_transform_diy · 步骤 2）

读取回填后的 xlsx，为每个 traj 生成独立的 Slicer 代码。

### 输出文件（共 6 个）

| 文件 | 内容 |
|------|------|
| `slicer_import_{case}_traj1_deposits.py` / `.md` | traj1 的 deposit 点 |
| `slicer_import_{case}_traj2_deposits.py` / `.md` | traj2 的 deposit 点 |
| `slicer_import_{case}_traj3_deposits.py` / `.md` | traj3 的 deposit 点 |

### 代码规范

- **每个 traj 只创建 1 个 `vtkMRMLMarkupsFiducialNode`**（单个 Point List）
- 所有 deposit 点通过 `AddFiducial(r, a, s)` + `SetNthFiducialLabel(i, label)` 添加
- **Glyph**：`StarBurst`
- **颜色**：同轨迹 Line（红/绿/蓝）
- **TextScale**：`0.0`
- 锁定节点 `SetLocked(True)`

### 生成模板

```python
import slicer

deposits = [
    ("d1 (dep=6,ang=0)", 19.21, -8.86, 0.12),
    # ... 全部 deposit
]

fid = slicer.mrmlScene.AddNewNodeByClass("vtkMRMLMarkupsFiducialNode")
fid.SetName("{case}_traj{N}_deposits")

for i, (label, r, a, s) in enumerate(deposits):
    try:
        fid.AddFiducial(r, a, s)
        fid.SetNthFiducialLabel(i, label)
    except Exception as ex:
        print(f"FAIL d{i+1}: {ex}")

disp = fid.GetDisplayNode()
disp.SetColor([R, G, B])
disp.SetSelectedColor([min(x + 0.3, 1.0) for x in color])
disp.SetGlyphType(slicer.vtkMRMLMarkupsDisplayNode.StarBurst)
disp.SetTextScale(0.0)
fid.SetLocked(True)
```

---

## 最终交付清单

执行全流程后交付以下文件：

```
slicer_import_{case}_trajs.py          ← 轨迹 Line 代码
slicer_import_{case}_trajs.md          ← 轨迹 Line 文档
slicer_import_{case}_traj1_deposits.py ← dep1 Point List
slicer_import_{case}_traj1_deposits.md
slicer_import_{case}_traj2_deposits.py ← dep2 Point List
slicer_import_{case}_traj2_deposits.md
slicer_import_{case}_traj3_deposits.py ← dep3 Point List
slicer_import_{case}_traj3_deposits.md
{case}_traj1.xlsx                      ← 已回填
{case}_traj2.xlsx
{case}_traj3.xlsx
```

---

## 触发关键词

- IMI全流程
- IMI转Slicer
- slicer全流程
- 完整拆分导入
- CSV到Slicer
- 一键生成Slicer代码
- imi_slicer

## 依赖

- Python 3.x + openpyxl
- 项目脚本：`compute_deposits.py`
- 子技能脚本：`~/.workbuddy/skills/imi-csv-diy/scripts/split_csv_to_xlsx.py`
