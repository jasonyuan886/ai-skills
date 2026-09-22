---
name: ai-pcb-generation
description: |
  基于Altium Designer二进制格式逆向工程的AI PCB自动生成技能。
  核心能力：
  1. Altium Designer .pcbdoc OLE2 Compound Document格式解析与写入
  2. 纯Python OLE2写入器（规避olefile mini-stream写入缺陷）
  3. AD 13.3兼容的PcbDoc v25格式生成（13个强制流全部符合规范）
  4. Components6流与WideStrings6数据流处理（AD优先从WideStrings6读取元件名）
  5. 自动布局布线算法（力导向布局+A*迷宫路由）
  6. 标准封装库管理（146+标准封装，覆盖SOP/DIP/QFP/SMD/接插件等）
  7. AD DelphiScript脚本生成（备用方案）
  技术突破：
  - 完整逆向AD 13.3 .pcbdoc OLE2容器格式
  - 开发纯Python OLE2写入器，规避olefile mini-stream写入bug
  - 成功输出669×2165mil LED驱动板.pcbdoc文件（BP2836驱动板迭代至v20）
  适用场景：给定原理图网表+元器件清单，自动生成AD兼容的.pcbdoc文件；
  封装库查询与管理；PCB布局布线优化；AD格式兼容性问题排查。
---

# AI PCB 自动生成技能

## 一、技能概述

本技能基于对Altium Designer 13.3二进制.pcbdoc格式的完整逆向工程经验，实现从原理图网表到可编辑PCB文件的全自动生成。

核心技术突破：
- **OLE2 Compound Document格式**：完全掌握.pcbdoc容器结构的读写
- **PcbDoc v25格式**：兼容AD 13.3的所有流（Stream）规范
- **WideStrings6数据流**：AD优先从此流读取元件名称，必须正确处理Unicode编码
- **Components6流**：元件定义的核心数据流，包含封装、焊盘、属性等完整信息
- **纯Python实现**：不依赖任何第三方OLE库的写入能力，规避olefile的mini-stream写入bug
- **成功验证**：输出669×2165mil LED驱动板文件，BP2836驱动板迭代至v20

## 二、参数输入

### 2.1 必需输入

| 参数 | 类型 | 说明 |
|------|------|------|
| `netlist` | dict | 网表数据：`{net_name: [(component_ref, pin_number), ...]}` |
| `components` | list | 元器件列表：`[{ref, value, footprint, package_type}, ...]` |
| `board_outline` | tuple | 板框尺寸 (width_mil, height_mil) |
| `layer_count` | int | 层数：2（双面板）或 4（四层板） |

### 2.2 可选输入

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `output_path` | str | `output.pcbdoc` | 输出文件路径 |
| `design_rules` | dict | 内置默认 | 设计规则（线宽、间距、过孔等） |
| `component_placement` | list | None | 预设元件坐标，None则自动布局 |
| `trace_width_default` | float | 10 mil | 默认走线宽度 |
| `via_drill_diameter` | float | 14 mil | 过孔钻孔直径 |

### 2.3 设计规则默认值

```python
DEFAULT_DESIGN_RULES = {
    "trace_width_min": 6,        # mil
    "trace_width_default": 10,   # mil
    "trace_width_power": 20,     # mil
    "clearance": 6,              # mil
    "via_drill": 14,             # mil
    "via_annular": 10,           # mil
    "track_to_track": 6,         # mil
    "component_clearance": 10,   # mil
}
```

## 三、封装库管理

### 3.1 封装库结构

```
footprint_library/
├── index.json              # 封装索引
├── footprints/
│   ├── SOIC-8.json
│   ├── DIP-8.json
│   ├── SMD-0805.json
│   └── ...
└── pads/
    └── pad_definitions.json
```

### 3.2 封装定义格式

```json
{
  "name": "SOIC-8",
  "type": "SMD",
  "description": "Small Outline IC, 8 pins, 1.27mm pitch",
  "dimensions": {
    "body_width": 154,
    "body_length": 193,
    "pitch": 50,
    "pad_width": 24,
    "pad_length": 60
  },
  "pins": [
    {"number": "1", "x": -75, "y": -76.2, "layer": "Top Layer"},
    {"number": "2", "x": -75, "y": -25.4, "layer": "Top Layer"}
  ],
  "silkscreen": {"lines": [], "arcs": []},
  "courtyard": {"outline": []},
  "3d_model": null
}
```

### 3.3 已入库标准封装（146个）

- **SOP/SOIC系列**：SOIC-8/14/16/18/20/24/28, TSSOP-8/14/16/20/24/28/32/44/48
- **DIP系列**：DIP-8/14/16/18/20/24/28/40
- **QFP系列**：LQFP-32/44/48/64/100, TQFP-32/44/48/64
- **电阻电容**：SMD-0201/0402/0603/0805/1206/1210/2010/2512
- **SOT系列**：SOT-23/SOT-23-5/SOT-223/SOT-89
- **二极管**：SMA/SMB/SMC/SOD-123/SOD-323
- **电感**：CD54/CD75/CD104/CD127
- **接插件**：USB-A/USB-C/XT30/XT60/JST-PH/JST-XH
- **LED**：LED-0603/0805/3528/5050/PLCC-2/PLCC-4

### 3.4 封装查询与匹配

```python
def query_footprint(component_type, pin_count, package_variant=None):
    """根据元器件类型、引脚数和可选封装变体查询最佳匹配封装"""
```

## 四、自动布局算法

### 4.1 布局策略

1. **功能分区**：按电路功能模块分组（电源区、信号区、接口区）
2. **关键元件优先**：先放置IC/连接器等关键元件
3. **层次化放置**：按信号流向从左到右、从上到下排列
4. **散热考虑**：大功率元件分散布置，预留散热铜皮区域

### 4.2 布局步骤

```
Step 1: 解析网表，构建元件连接图
Step 2: 识别功能模块（基于元件类型和连接拓扑）
Step 3: 确定模块在板上的大致区域
Step 4: 模块内元件按连接紧密度聚类
Step 5: 力导向算法优化元件间距
Step 6: 边界约束检查（不超出板框、满足间距要求）
Step 7: 输出最终坐标 [{ref, x, y, rotation, layer}]
```

### 4.3 布局约束

- 元件不得超出板框（含courtyard）
- 元件间距 ≥ component_clearance（默认10mil）
- 连接器统一靠板边放置
- 去耦电容紧邻对应IC的VCC/GND引脚
- 散热元件远离温度敏感器件

## 五、自动布线算法

### 5.1 布线优先级

```
1. 电源线（VCC/GND网络）  → 宽线、优先铺铜
2. 关键信号线（时钟/差分） → 等长匹配、最短路径
3. 普通信号线             → 默认规则
4. 测试点/跳线            → 最后处理
```

### 5.2 布线算法（基于A*迷宫路由器）

```
Step 1: 按网络优先级排序待布线网络
Step 2: 对每个网络：
  a. 获取该网络所有焊盘坐标
  b. 构建连接对（最小生成树确定布线顺序）
  c. 对每个连接对执行A*寻路：
     - 代价函数：距离 + 转弯惩罚 + 层切换惩罚
     - 避障：已布线track、焊盘、过孔、板框
  d. 布线结果写入track/via列表
Step 3: 铺铜处理（GND/VCC铜皮）
Step 4: DRC检查
```

### 5.3 过孔管理

- 通孔过孔：顶层↔底层切换时使用
- 过孔尺寸：drill=14mil, pad=34mil（默认）
- 尽量减少过孔数量
- 高速信号换层时就近添加回流地过孔

## 六、OLE2格式技术细节

### 6.1 OLE2 Compound Document结构

.pcbdoc文件是标准的OLE2（Compound File Binary Format, CFB）容器：

```
OLE2 Header (512 bytes)
├── FAT (File Allocation Table)
├── MiniFAT
├── DIFAT
└── Directory Entries
    ├── PcbDesign_Stream
    ├── Components6          ← 核心：元件定义
    ├── WideStrings6         ← 核心：Unicode字符串
    ├── Record
    ├── BinaryData
    ├── IsDesignatorVisible
    ├── NetNames6
    ├── DesignClass6
    ├── PcbRouting
    ├── FromTo
    ├── Masks
    ├── Plane6
    └── Classes
```

### 6.2 Header结构（512字节）

```python
OLE2_HEADER = {
    "magic": b"\xD0\xCF\x11\xE0\xA1\xB1\x1A\xE1",
    "clsid": b"\x00" * 16,
    "minor_version": 0x003E,
    "major_version": 0x0003,
    "byte_order": 0xFFFE,
    "sector_size": 0x0009,           # 2^9 = 512 bytes
    "mini_sector_size": 0x0006,      # 2^6 = 64 bytes
    "mini_stream_cutoff": 0x00001000, # 4096 bytes
}
```

### 6.3 Components6流格式

```
|RECORD=1            # Component记录头
|OWNERINDEX=0
|UNIQUEID=xxxxxxxxx
|AREACOLOR=16777215
...
|RECORD=41           # 子记录
|OWNERINDEX=0
|TEXT=U1             # 元件名（但AD优先读WideStrings6！）
|RECORD=4            # 焊盘
...
|RECORD=34           # Designator
|OWNERINDEX=0
|TEXT=U1
|RECORD=0            # 记录结束
```

### 6.4 WideStrings6流格式（关键！）

**核心发现：AD优先从WideStrings6流读取元件名称。**

```python
def encode_widestrings6(components):
    """
    编码WideStrings6流。
    - 以 |RECORD=41 开头的每个子记录对应一个字符串
    - INDEX字段指向Components6中的对应元件
    - TEXT字段包含UTF-16LE编码的字符串
    - AD读取元件名时，WideStrings6优先级高于Components6中的TEXT字段
    - 如果WideStrings6为空或缺失，元件名显示为空白
    """
```

### 6.5 纯Python OLE2写入器

**为什么不用olefile**：
- mini-stream（<4096字节流）写入时FAT链计算错误
- 目录项起始扇区分配不正确
- .pcbdoc中所有流都是mini-stream，olefile写入后AD完全无法识别

```python
class PcbDocWriter:
    """纯Python OLE2/PcbDoc v25写入器"""
    
    def __init__(self, sector_size=512, mini_sector_size=64):
        self.fat = []
        self.minifat = []
        self.directory = []
        self.streams = {}
        self.mini_stream = bytearray()
        
    def write_header(self): ...
    def build_fat(self): ...
    def build_minifat(self): ...
    def build_directory(self): ...
    def write_stream(self, name, data): ...
    def write_pcbdoc(self, filepath, pcb_data): ...
```

**Mini-Stream写入关键点**：
1. 手动计算mini-sector分配链
2. 所有mini-stream数据连续写入，每64字节一个mini-sector
3. MiniFAT终止标记必须是0xFFFFFFFD（ENDOFCHAIN）
4. Mini-stream容器大小必须是sector_size的整数倍
5. Header中mini_stream_start_sector必须指向正确sector

### 6.6 13个强制流清单

| # | 流名称 | 用途 |
|---|--------|------|
| 1 | PcbDesign_Stream | PCB设计全局设置 |
| 2 | Components6 | 元件定义（封装/焊盘） |
| 3 | WideStrings6 | Unicode字符串（元件名等） |
| 4 | Record | 图形记录（线段/弧/填充） |
| 5 | BinaryData | 二进制数据（3D模型等） |
| 6 | IsDesignatorVisible | 标号可见性 |
| 7 | NetNames6 | 网络名定义 |
| 8 | DesignClass6 | 设计类定义 |
| 9 | PcbRouting | PCB布线规则 |
| 10 | FromTo | 起终点对定义 |
| 11 | Masks | 阻焊/助焊层掩膜 |
| 12 | Plane6 | 内电层定义 |
| 13 | Classes | 类定义 |

## 七、AD 13.3格式规范

### 7.1 坐标系统

- 单位：mil（1mm = 39.3701mil）
- 原点：板框左下角
- X轴向右，Y轴向上
- 精度：0.001mil

### 7.2 层定义

```python
LAYER_MAP = {
    "Top Layer": 1,
    "Mid Layer 1": 2,
    "Mid Layer 2": 3,
    "Bottom Layer": 32,
    "Top Overlay": 33,
    "Bottom Overlay": 34,
    "Top Paste": 35,
    "Bottom Paste": 36,
    "Top Solder": 37,
    "Bottom Solder": 38,
    "Keep Out Layer": 39,
    "Mechanical 1": 40,
    "Multi-Layer": 42,
    "Drill Guide": 44,
    "Drill Drawing": 45,
}
```

### 7.3 焊盘/走线/过孔记录格式

**焊盘（RECORD=18）**：
```python
PAD_RECORD = {
    "RECORD": 18,
    "OWNERINDEX": "...",    # 所属元件索引
    "X1": "...", "Y1": "...",  # 中心坐标
    "XSIZE": "...", "YSIZE": "...",  # 尺寸
    "SHAPE": "...",         # Round/Rect/Oval
    "LAYER": "...",
    "NET": "...",           # 网络名
    "DESIGNATOR": "...",    # 焊盘编号
    "HOLETYPE": "...",      # 0=无孔, 1=通孔
    "HOLEWIDTH": "...",
    "ROTATION": "...",
    "PLATED": "T/F",
}
```

**走线（RECORD=11）**：
```python
TRACK_RECORD = {
    "RECORD": 11,
    "X1": "...", "Y1": "...",
    "X2": "...", "Y2": "...",
    "WIDTH": "...",
    "LAYER": "...",
    "NET": "...",
}
```

**过孔（RECORD=12）**：
```python
VIA_RECORD = {
    "RECORD": 12,
    "X": "...", "Y": "...",
    "SIZE": "...",          # 焊盘直径
    "HOLESIZE": "...",      # 钻孔直径
    "LAYER": "Multi-Layer",
    "NET": "...",
}
```

## 八、完整执行流程

### Step 1：参数解析与验证
- 解析netlist和components
- 验证网表连通性（无悬空引脚）
- 检查封装库匹配度
- 初始化设计规则

### Step 2：封装选择与准备
- 为每个元器件查询/匹配封装
- 加载封装定义
- 验证封装引脚数与网表一致
- 缺失封装时从标准库推荐

### Step 3：板框与层设置
- 根据board_outline生成Keep-Out层板框
- 设置层叠结构
- 初始化PcbDesign_Stream
- 生成NetNames6流

### Step 4：自动布局
- 构建连接图 → 功能分区 → 力导向布局 → 约束检查 → 输出坐标

### Step 5：自动布线
- 电源线 → 关键信号线 → 普通信号线 → 铺铜 → DRC

### Step 6：OLE2格式输出
- 生成Components6流（所有元件+焊盘+属性）
- 生成WideStrings6流（所有元件名称，UTF-16LE）
- 生成Record流（走线+过孔+丝印+铜皮）
- 生成其余9个强制流
- 调用PcbDocWriter写入OLE2容器
- 输出.pcbdoc文件

### Step 7：输出验证
- 文件大小检查
- OLE2头部魔数验证（D0CF11E0A1B11AE1）
- 目录项完整性检查（13个强制流全部存在）
- FAT链/MiniFAT链完整性验证
- WideStrings6元件名与Components6索引匹配
- 用AD打开测试（如有条件）

## 九、常见技术难点与解决方案

### 9.1 WideStrings6编码问题
**问题**：AD打开后元件名显示乱码或空白
**原因**：WideStrings6流编码格式不正确
**解决**：确保UTF-16LE编码；INDEX字段与Components6记录索引严格对应

### 9.2 Mini-Stream写入失败
**问题**：AD报文件损坏
**原因**：MiniFAT链计算错误
**解决**：手动管理mini-sector分配；MiniFAT终止标记0xFFFFFFFD；容器大小对齐512字节

### 9.3 Components6记录索引错乱
**问题**：焊盘/丝印与元件不对应
**原因**：OWNERINDEX字段指向错误
**解决**：维护全局元件计数器；子记录OWNERINDEX指向父元件；`|RECORD=0`标记结束

### 9.4 层号映射错误
**问题**：焊盘出现在错误层
**解决**：严格使用LAYER_MAP；通孔焊盘必须设Multi-Layer（42）

### 9.5 坐标单位错误
**问题**：位置偏差1000倍
**解决**：AD内部统一mil；mm转mil公式 `mil = mm * 1000 / 25.4`

### 9.6 网表连接丢失
**问题**：网络名未关联到焊盘
**解决**：NetNames6中定义所有网络名；焊盘NET字段必须完全匹配

### 9.7 olefile库写入缺陷
**问题**：olefile写入后AD无法识别
**解决**：完全自研OLE2写入器；仅用olefile做读取验证

## 十、输出文件验证方法

### 10.1 基础验证（纯Python）

```python
def verify_pcbdoc(filepath):
    """验证生成的.pcbdoc文件完整性"""
    with open(filepath, 'rb') as f:
        magic = f.read(8)
        assert magic == b'\xD0\xCF\x11\xE0\xA1\xB1\x1A\xE1'
    
    import olefile
    ole = olefile.OleFileIO(filepath)
    
    REQUIRED_STREAMS = [
        "PcbDesign_Stream", "Components6", "WideStrings6",
        "Record", "BinaryData", "IsDesignatorVisible",
        "NetNames6", "DesignClass6", "PcbRouting",
        "FromTo", "Masks", "Plane6", "Classes"
    ]
    for stream in REQUIRED_STREAMS:
        assert ole.exists(stream), f"缺少强制流: {stream}"
    
    assert len(ole.openstream("WideStrings6").read()) > 0
    assert len(ole.openstream("Components6").read()) > 0
    
    ole.close()
    return True
```

### 10.2 内容验证

```python
def verify_content(filepath, expected_components):
    """验证文件内容与预期一致"""
    import olefile
    ole = olefile.OleFileIO(filepath)
    comp_data = ole.openstream("Components6").read()
    comp_count = comp_data.count(b'|RECORD=1')
    assert comp_count >= expected_components
    ole.close()
    return True
```

### 10.3 AD兼容性验证
1. 将.pcbdoc拷贝到Windows环境
2. 用AD 13.3+打开
3. 检查：文件正常打开、元件可见且位置正确、名称正确显示、焊盘位置正确、走线连接正确、丝印可见、板框完整

## 十一、AD DelphiScript脚本生成（备用方案）

```python
def generate_delphi_script(components, netlist, board_outline):
    """生成DelphiScript，在AD中运行以创建PCB"""
    # 生成Var声明、PCBServer调用、元件放置代码
    # AD中通过 Tools → Run Script 执行
```

## 十二、版本兼容性

| AD版本 | 格式版本 | 状态 |
|--------|----------|------|
| AD 13.x | PcbDoc v25 | ✅ 完全兼容（已验证） |
| AD 14-17.x | PcbDoc v25-v27 | ⚠️ 基本兼容 |
| AD 18+ | PcbDoc v28+ | ⚠️ 基础结构兼容 |
| AD 10及以下 | PcbDoc v24- | ❌ 不兼容 |

## 十三、关键经验总结

1. **AD优先读WideStrings6**：这是最关键的发现。如果WideStrings6为空或格式错误，AD不会显示元件名。
2. **olefile写入不可靠**：mini-stream场景下FAT链计算有bug，必须自研写入器。
3. **13个强制流缺一不可**：缺少任何一个流AD都会报错。
4. **层号必须精确**：Bottom Layer=32而非0，必须严格对照LAYER_MAP。
5. **坐标原点在左下角**：Y轴向上为正。
6. **封装库是基础**：146个标准封装覆盖常见LED驱动、电源、MCU应用。
7. **DelphiScript是后备方案**：直接生成.pcbdoc遇到兼容性问题时可生成AD脚本替代。
