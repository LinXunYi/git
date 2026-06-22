# WeGui-ARGB 设计参考

> 本文档从 README.md、CLAUDE.md、WEGUI_API_REFERENCE.md 及源码中提炼，供快速查阅框架设计思路与关键实现细节。

---

## 1. 项目定位

轻量级嵌入式 GUI 框架，面向 MCU/SoC 多平台，同时提供 SDL2 PC 模拟器。
版本 V0.2.1，Apache License 2.0。

核心目标：
- **极低资源占用**：零 malloc、零浮点路径，Cortex-M0 原生可用
- **高可移植性**：平台无关内核 + 移植层分离，新增 MCU 只需实现端口回调
- **完整 GUI 能力**：21 个内置控件、中央动画引擎、脏矩形刷新、输入手势系统

---

## 2. 架构总览

```
┌──────────────────────────────────────────────────┐
│              用户应用层 (main.c / demo_xxx.c)      │
├──────────────────────────────────────────────────┤
│            控件层 (Core/widgets/...)              │
│  label│btn│img│arc│group│toggle│slider│dropdown   │
├──────────────────────────────────────────────────┤
│         GUI 内核 (Core/we_gui_driver.h)           │
│  tick│timer│task│脏矩形│PFB 渲染│动画引擎│输入分发  │
├──────────────────────────────────────────────────┤
│          平台移植层 (各目标头文件 + 端口实现)       │
│  LCD 驱动│输入驱动│存储驱动│DMA│色彩格式适配        │
└──────────────────────────────────────────────────┘
```

---

## 3. 目录结构

| 路径 | 说明 |
|------|------|
| `Core/` | 平台无关内核：渲染、脏矩形引擎、字体、所有控件实现 |
| `Core/widgets/xxx/` | 各控件源码 + `widget.md` 详细设计文档 |
| `Demo/` | 21 个独立 demo，每个控件对应 `demo_xxx.c` |
| `STM32F103/` | F103 硬件入口、Keil 工程、LCD/输入/外挂 Flash 端口 |
| `STM32F030/` | F030 硬件入口、Keil 工程、LCD/输入端口 |
| `Simulator/` | SDL2 模拟器入口、SDL 端口、CMake 构建脚本 |
| `tool/` | 图片转换 (bin2c)、字库生成 (font2c)、外挂 Flash 下载工具 |
| `we_user_config.h` | **统一用户配置入口**（屏幕/色深/PFB/脏矩形/定时器/输入/存储）|
| `we_hw_port.h` | 平台路由头（根据宏选择具体硬件配置）|
| `WEGUI_API_REFERENCE.md` | 完整 API 参考 |
| `CLAUDE.md` | 架构说明、运行时模型、代码风格指南 |

---

## 4. 核心设计机制

### 4.1 PFB（Partial Frame Buffer）局部帧缓冲

- **不用全屏显存**，仅用少量行缓冲区（默认 8 行 = `SCREEN_WIDTH × 8` 像素）
- 240×240 屏幕下 PFB 仅 3.84 KB（整屏的 3.3%）
- 渲染逐行带（strip）进行：标记脏区 → 定位 PFB 行范围 → 设置 LCD 窗口 → 填充 PFB → DMA 推送到 LCD → 下一行带
- 增大行数可减少 flush 次数，以 RAM 换速度

### 4.2 脏矩形刷新策略

通过 `WE_CFG_DIRTY_STRATEGY` 配置：

| 值 | 策略 | 适用场景 |
|----|------|---------|
| 0 | 全屏重绘 | 最简单，适合画面频繁大面积变化 |
| 1 | 单一合并包围盒 | 变化区域集中时效率高 |
| 2 | 多矩形合并（最多 N 个） | **默认**，变化区域分散时最优 |

- `WE_CFG_DIRTY_MAX_NUM = 10`（当前默认值）
- 每个控件 set 操作自动标记对应区域为脏
- `WE_CFG_DEBUG_DIRTY_RECT = 1` 可用红色叠加显示脏区域（调试用）

### 4.3 中央动画引擎（we_anim_t）

```c
typedef struct we_anim_s {
    struct we_anim_s *next;
    void (*step_cb)(void *owner, uint16_t elapsed_ms);
    void *owner;
} we_anim_t;
```

设计要点：
- **不占用 GUI task 槽**（`WE_CFG_GUI_TASK_MAX_NUM` 的 4 个槽完全留给内核/用户扩展）
- 节点内嵌在控件结构体中 → **零堆分配**
- 数量无上限，`we_anim_start()` 不会失败
- 到达目标态后在 `step_cb` 内自行 `we_anim_stop()` 摘链 → **空闲零开销**
- `we_gui_task_handler()` 每周期遍历链表驱动所有动画

使用者：toggle、progress、indicator、msgbox、slideshow、scroll_panel、dropdown（滚动条淡出）

### 4.4 对象链表与渲染顺序

- 所有控件注册到 `lcd->obj_list_head` 单链表
- 绘制顺序 = 插入顺序（先插入先绘制，后绘制的覆盖在上面 → 画家算法）
- 碰撞检测从链表尾部开始遍历 → **最上层控件优先**接收触摸事件

### 4.5 输入系统与手势检测

触摸状态机：
```
NONE → PRESSED → STAY（持续按住）→ RELEASED → CLICKED
```

手势：
- 释放时若位移 > `WE_CFG_SWIPE_THRESHOLD`（默认 30px），分发滑动事件而非 CLICKED
- 主方向判定：|dx| vs |dy|，决定水平/垂直滑动
- 容器控件自动处理滑动事件做分页吸附

事件类型：
```
WE_EVENT_PRESSED / RELEASED / CLICKED / STAY
WE_EVENT_SWIPE_LEFT / RIGHT / UP / DOWN
WE_EVENT_VALUE_CHG / SCROLLED
```

### 4.6 容器透明度级联

- group / slideshow / scroll_panel 的透明度向子控件级联传播
- 机制：容器在子控件 pass 前后乘入 lcd 级 `opa_scale`
- 每个绘图基元入口处调用 `we_opa_apply` 消费一次
- **无级联时零逐像素开销**，嵌套自动组合

### 4.7 定点数学体系

| 概念 | 数值范围 | 说明 |
|------|---------|------|
| 角度（512 步） | 0~511 = 整圆 | 90° = 128, 用 `WE_DEG(deg)` 转换 |
| 缩放（Q8 定点） | 256 = 1.0x | 128 = 0.5x, 512 = 2.0x |
| 缓动（t ∈ [0,256]） | 输出 [0,256] | `we_ease_*` 系列函数 |
| 三角函数 | 返回 Q15 | `we_sin` / `we_cos`，范围 -32768~32767 |
| 线性插值 | `we_lerp(from, to, t)` | `from + (to - from) * t / 256` |

---

## 5. 运行时模型

### 5.1 核心对象：we_lcd_t

每个 GUI 实例以一个 `we_lcd_t` 为中心，拥有：
- PFB/GRAM 缓冲区 + LCD flush 回调
- 脏矩形管理器
- GUI 对象根链表
- GUI 内部 task 槽 + 用户 timer 槽
- 中央动画链表头（`anim_head`）
- 输入状态（`we_indev_data_t`）
- 可选存储回调
- 渲染统计计数器

### 5.2 主循环模式

```c
we_gui_init(&lcd, bg, gram, USER_GRAM_NUM,
            lcd_set_addr, LCD_FLUSH_PORT, input_cb, storage_cb);
we_xxx_simple_demo_init(&lcd);
we_gui_timer_create(&lcd, we_xxx_simple_demo_tick, 16U, 1U);

while (1) {
    we_gui_tick_inc(&lcd, elapsed_ms);     // 注入经过的时间
    we_gui_task_handler(&lcd);             // 输入 + 定时器 + 渲染 + flush
}
```

`we_gui_task_handler()` 内部流程：
1. 处理输入设备 → 分发事件到控件
2. 驱动所有活跃的 timer 和动画 step 回调
3. 遍历脏矩形列表
4. 对每个脏区：设置 LCD 窗口 → 遍历对象链表 → 调用各控件 draw_cb → DMA flush PFB

### 5.3 配置链

```
we_user_config.h（用户编辑入口）
  └→ we_hw_port.h（平台路由）
       ├→ WE_SIMULATOR         → Simulator/we_sim_port_config.h
       ├→ WE_PLATFORM_STM32F030 → STM32F030/Lcd_Port/stm32f030_hw_config.h
       └→ default               → STM32F103 配置
            └→ 选择 LCD_IC + LCD_PORT → 绑定 lcd_set_addr / flush 回调
                 └→ Core/we_gui_config.h（#error 验证所有必需宏）
```

---

## 6. 控件设计概览

### 6.1 统一模式

所有控件遵循：
1. 声明 **static** 对象变量（零堆分配）
2. `we_xxx_obj_init(...)` → 自动注册到渲染链表
3. `we_xxx_set_yyy(...)` → 修改属性并自动标记脏区重绘
4. `we_xxx_obj_delete(...)` → 从链表移除（若带动画需先 `we_anim_stop`）

### 6.2 控件基类

```c
typedef struct we_obj_t {
    struct we_lcd_t *lcd;
    int16_t x, y;
    uint16_t w, h;
    const we_class_t *class_p;   // draw_cb + event_cb
    struct we_obj_t *next;       // 链表指针
    struct we_obj_t *parent;     // 父控件（容器子控件用）
} we_obj_t;
```

### 6.3 各控件关键设计点

| 控件 | 关键设计 |
|------|---------|
| **label** | UTF-8 文本，支持 `\n` 多行，二分查找字形索引，4bpp 抗锯齿 |
| **btn** | 4 种状态（normal/selected/pressed/disabled），可自定义样式，支持自定义事件回调 |
| **img** | 支持 RGB565/888/555/444/332, ARGB8888/6666/4444, RLE, QOI, 索引 QOI |
| **img_ex** | 旋转/缩放，仅支持未压缩 RGB565 原图，支持偏心枢轴 |
| **arc** | 进度环，value 0~255 映射进度，512 步角度系统 |
| **group** | 轻量子控件容器，本地坐标系，透明度级联，触摸事件转发 |
| **slideshow** | 基于 group 的分页滑动容器，支持滑动吸附 |
| **checkbox** | 圆角矩形绘制复用公用解析式圆角填充函数 |
| **label_ex** | 旋转/缩放文字，同样使用 512 步角度 + Q8 缩放 |
| **chart** | 环形缓冲区 + 像素空间数据值，波形柔边参考 Arm-2D 但已重写 |
| **toggle** | 轨道/滑块复用公用解析式圆角矩形填充，动画走中央引擎 |
| **progress** | 0~255 目标值，平滑动画过渡 |
| **msgbox** | 模态弹窗 `we_popup_obj_t`，透明度淡入/淡出 + 圆角面板 |
| **slider** | 支持值变化回调（仅用户交互触发） |
| **scroll_panel** | 滚动容器，透明度级联，子控件触摸转发 |
| **dropdown** | 数据驱动（调用者拥有选项数组），像素级拖拽滚动，滚动条空闲自动淡出，通过 LCD 级 `popup_layer` 绘制（全屏幕仅一个弹窗） |
| **stepper** | 定点 int32 存储（real = value / 10^decimals），按住连续步进不占 timer 槽 |
| **indicator** | 圆形状态灯，亮灭过渡走中央动画引擎，可选外发光晕，默认只读 |
| **img_flash** | 外挂 SPI Flash 存储图片资源 |
| **font_flash** | 外挂 SPI Flash 存储字体资源 |

---

## 7. 资源占用实测

### 7.1 整体（dropdown 最小可交互应用）

| 目标 | ROM | RAM | 条件 |
|------|-----|-----|------|
| STM32F103 (Cortex-M3, 72MHz) | 23.02 KB | 6.13 KB | Keil .map 实测 |
| STM32F030 (Cortex-M0, 48MHz) | 22.26 KB | 6.09 KB | Keil .map 实测 |

- 280×240 RGB565，PFB 8 行（4.48 KB）
- ROM 含 5.9 KB 字体资产 + LCD/输入/存储端口 + 启动代码
- **GUI 库本体约 10 KB**

### 7.2 各 demo 单独占用（STM32F030 实测）

| ID | demo | ROM | RAM | 备注 |
|----|------|-----|-----|------|
| 1 | label | 17.0 KB | 5.96 KB | 基准底座（含 5.9 KB 字体） |
| 2 | btn | 18.7 KB | 6.04 KB | |
| 3 | img | 37.0 KB | 6.02 KB | 含内嵌图片 ~18 KB |
| 4 | img_ex | 28.3 KB | 5.98 KB | 含内嵌图片 ~10 KB |
| 5 | arc | 19.8 KB | 6.02 KB | |
| 6 | group | 20.2 KB | 6.43 KB | 含子控件 |
| 7 | slideshow | 25.6 KB | 6.95 KB | RAM 最高（多页状态）|
| 8 | concentric arc | 19.7 KB | 6.06 KB | |
| 9 | checkbox | 20.0 KB | 6.33 KB | |
| 10 | label_ex | 18.7 KB | 6.01 KB | |
| 11 | chart | 19.1 KB | 6.53 KB | |
| 12 | toggle | 19.4 KB | 6.29 KB | |
| 13 | progress | 20.8 KB | 6.13 KB | |
| 14 | msgbox | 22.2 KB | 6.13 KB | |
| 15 | flash img | 19.8 KB | 6.06 KB | 需外挂 SPI Flash |
| 16 | flash font | 19.6 KB | 6.18 KB | 需外挂 SPI Flash |
| 17 | slider | 20.1 KB | 6.18 KB | |
| 18 | scroll_panel | 22.2 KB | 6.69 KB | 含子控件 |
| 19 | dropdown | 22.3 KB | 6.09 KB | |
| 20 | stepper | 20.0 KB | 6.23 KB | |
| 21 | indicator | 19.4 KB | 6.36 KB | |

---

## 8. 关键配置宏速查

| 宏 | 说明 | 默认值 |
|----|------|--------|
| `LCD_DEEP` | 色深 | `DEEP_RGB565` 或 `DEEP_RGB888` |
| `SCREEN_WIDTH` / `SCREEN_HEIGHT` | 屏幕尺寸 | 240 / 240 |
| `GRAM_DMA_BUFF_EN` | DMA 双缓冲 | 0 或 1 |
| `USER_GRAM_NUM` | PFB 缓冲区大小（像素数）| `SCREEN_WIDTH × 8` |
| `WE_CFG_DIRTY_STRATEGY` | 脏矩形策略 | 2（多矩形合并）|
| `WE_CFG_DIRTY_MAX_NUM` | 最大脏矩形数 | 10 |
| `WE_CFG_DEBUG_DIRTY_RECT` | 调试显示脏区域 | 0 |
| `WE_CFG_ENABLE_INDEXED_QOI` | 索引 QOI 解码 | 0 或 1 |
| `WE_CFG_GUI_TASK_MAX_NUM` | 内部 task 槽数 | 4 |
| `WE_CFG_GUI_TIMER_MAX_NUM` | 用户 timer 槽数 | 8 |
| `WE_CFG_ENABLE_INPUT_PORT_BIND` | 输入端口绑定 | 0 或 1 |
| `WE_CFG_ENABLE_STORAGE_PORT_BIND` | 存储端口绑定 | 0 或 1 |
| `WE_CFG_SWIPE_THRESHOLD` | 滑动手势阈值（px）| 30 |

---

## 9. 构建方式

### Simulator（CMake + MinGW）

```powershell
# 清理并重建
powershell -NoProfile -ExecutionPolicy Bypass -File "Simulator/build_sim.ps1" -Clean

# 运行
powershell -NoProfile -ExecutionPolicy Bypass -File "Simulator/run_latest_sim.ps1"
```

### STM32（Keil MDK-ARM AC5）

```powershell
# F103
UV4.exe -r "STM32F103\MDK-ARM\Project.uvprojx" -t "WeGui_ARGB"

# F030
UV4.exe -r "STM32F030\MDK-ARM\Project.uvprojx" -t "STM32F030"
```

### Demo 选择

修改入口 `main` 顶部 `#define DEMO_ID`（0~21），三个目标编号统一。

---

## 10. 关键文件索引

| 文件 | 重要性 | 说明 |
|------|--------|------|
| `we_user_config.h` | ★★★★★ | 统一配置入口，首先阅读 |
| `we_hw_port.h` | ★★★★ | 平台路由，理解多目标架构 |
| `Core/we_gui_driver.h` | ★★★★★ | 核心运行时对象 + 公共 API |
| `Core/we_gui_config.h` | ★★★★ | 配置宏验证（#error 检查）|
| `Core/we_gui_driver.c` | ★★★★★ | 内核实现：tick/task/render/flush |
| `Core/dirty_driver.c` | ★★★★ | 脏矩形引擎实现 |
| `Core/we_render.h` | ★★★★ | 渲染基元声明 |
| `Core/we_motion.c/h` | ★★★ | 缓动函数库 |
| `Core/we_font_text.c/h` | ★★★ | 字体渲染 |
| `Core/image_res.c/h` | ★★★ | 图片解码（RLE/QOI）|
| `WEGUI_API_REFERENCE.md` | ★★★★★ | 完整 API 参考 + 用法示例 |
| `CLAUDE.md` | ★★★★★ | 架构说明 + 语义约定 |
| `Core/widgets/xxx/widget.md` | ★★★★ | 各控件详细 API 文档 |

---

## 11. 设计亮点总结

1. **零堆分配**：所有控件 static 声明，动画节点内嵌，无任何 malloc/free
2. **零浮点**：全部使用定点运算（512 步角度 + Q8 缩放 + Q15 三角），Cortex-M0 可用
3. **中央动画引擎**：统一驱动所有控件动画，不占 task 槽，无数量限制，空闲自摘链
4. **PFB 局部渲染**：仅 3.3% 整屏显存即可流畅运行
5. **脏矩形三策略**：灵活适配不同场景，最小化无效绘制
6. **透明度级联**：容器 → 子控件传播，零逐像素开销
7. **数据驱动控件**：dropdown/stepper 等控件不拷贝数据，仅持有指针
8. **统一 API 范式**：init → set → delete，自动脏标记，极低学习成本
9. **跨平台一致**：同一套内核代码跑在 STM32F103 / F030 / SDL2 模拟器上，demo 编号统一
10. **模块化工具链**：图片/字体转换工具齐全，支持外挂 Flash 资源

---

*文档生成时间：2026-06-23*
*来源：WeGui-ARGB V0.2.1 源码及文档*
