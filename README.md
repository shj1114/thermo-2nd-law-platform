/**
 * ============================================================
 *  MSPM0G3507 + HMI串口屏 电流监测仪
 *  2026全国大学生电子设计竞赛 B题
 * ============================================================
 * 
 * 项目目录结构:
 *   1224/
 *   ├── README.md                    ← 本文件
 *   ├── HMI串口屏配置指南.md           ← HMI屏配置详细指南
 *   ├── hmi_screen_project.txt       ← HMI屏界面设计说明
 *   ├── download_tools.ps1           ← 软件下载脚本
 *   ├── setup_environment.bat        ← 环境配置批处理
 *   │
 *   ├── main.c                       ← 主程序 (电流采样+显示)
 *   ├── hmi_uart.c                   ← HMI串口驱动实现
 *   ├── hmi_uart.h                   ← HMI串口驱动头文件
 *   │
 *   ├── ccs_project/                 ← CCS项目目录
 *   │   ├── .projectspec
 *   │   ├── .cproject
 *   │   ├── target_config.ccxml      ← 调试器配置
 *   │   ├── mspm0g3507.lds           ← 链接脚本
 *   │   ├── startup_mspm0g3507.c     ← 启动代码
 *   │   ├── main.c                   ← 主程序副本
 *   │   ├── hmi_uart.c               ← 驱动代码副本
 *   │   └── hmi_uart.h               ← 驱动头文件副本
 *   │
 *   └── 其他说明文件...
 * 
 * ============================================================
 */

# MSPM0G3507 + HMI串口屏 电流监测仪

## 快速开始

### 步骤1：安装开发工具

双击运行 `setup_environment.bat`，会自动打开所需软件的下载页面。

**需要安装的软件：**
- **USART HMI** - HMI屏界面编辑软件
  - 下载: http://wiki.tjc1688.com/doku.php?id=start
  - 用于设计HMI屏幕的显示界面

- **TI Code Composer Studio (CCS)** - MSPM0开发环境
  - 下载: https://www.ti.com/tool/CCSTUDIO
  - 用于编译和下载MSPM0G3507的程序

### 步骤2：创建HMI屏幕界面

1. 打开 USART HMI 软件
2. 新建工程，选择 HMI01-0430 型号
3. 参考 `hmi_screen_project.txt` 设计页面
4. 参考 `HMI串口屏配置指南.md` 配置参数
5. 编译并烧录到HMI屏幕

### 步骤3：导入CCS项目

1. 打开 Code Composer Studio
2. 菜单: File -> Import -> CCS Projects
3. 选择 `ccs_project` 目录
4. 导入项目
5. 编译并下载到MSPM0G3507

### 步骤4：硬件接线

| MSPM0G3507 | HMI屏幕 |
|------------|---------|
| 3.3V       | VCC     |
| GND        | GND     |
| P1.3 (TX)  | RX      |
| P1.2 (RX)  | TX      |

### 步骤5：运行测试

1. 连接HMI屏幕到电脑 (USB转TTL)
2. 烧录HMI屏幕工程
3. 连接MSPM0G3507和HMI屏幕
4. 下载MSPM0G3507程序
5. 观察屏幕显示电流数据

---

## 软件架构

```
┌─────────────────────────────────────────────┐
│              MSPM0G3507 主控                 │
├─────────────────────────────────────────────┤
│                                             │
│  main.c                                     │
│  ├── 系统初始化                              │
│  ├── ADC采样 (100ms间隔)                    │
│  ├── 电流计算与滤波                         │
│  ├── 最大值/最小值记录                      │
│  ├── 曲线数据更新                           │
│  └── HMI显示更新 (500ms间隔)               │
│                                             │
│  hmi_uart.c / hmi_uart.h                   │
│  ├── UART初始化 (9600bps)                  │
│  ├── 指令发送API                            │
│  │   ├── HMI_SetInt()                      │
│  │   ├── HMI_SetFloat()                    │
│  │   ├── HMI_AddCurve()                    │
│  │   └── HMI_ClearCurve()                  │
│  ├── 数据接收解析                           │
│  └── 中断服务函数                           │
│                                             │
└──────────────┬──────────────────────────────┘
               │ UART (9600bps)
               │ 3.3V TTL
               ▼
┌─────────────────────────────────────────────┐
│           正点原子 HMI01 串口屏              │
├─────────────────────────────────────────────┤
│                                             │
│  页面0: 主界面                              │
│  ├── n0: 实时电流值 (0.01A精度)            │
│  ├── n1: 最大电流记录                      │
│  ├── n2: 最小电流记录                      │
│  ├── c0: 电流趋势曲线                      │
│  ├── b0: 校准按钮                          │
│  ├── b1: 清零按钮                          │
│  └── b2: 设置按钮                          │
│                                             │
│  页面1: 设置界面                            │
│  ├── n10: 设备地址                         │
│  └── b10: 返回按钮                         │
│                                             │
└─────────────────────────────────────────────┘
```

---

## 通信协议

### MCU → HMI (发送指令)

```
格式: 指令内容 + 0xFF 0xFF 0xFF (结束标志)
示例: prints n0,150 + 0xFF 0xFF 0xFF
      ↑         ↑  ↑
      指令     变量 数值(1.50A)
```

### HMI → MCU (返回数据, bkcmd=3)

```
格式: 变量名.值\r\n
示例: n0.150\r\n → 电流为1.50A
      b0.1\r\n   → 按钮b0被按下
```

### 常用指令

| 指令 | 用途 | 示例 |
|------|------|------|
| `page N` | 跳转到页面N | `page 0` |
| `prints N,V` | 设置整型变量 | `prints n0,123` |
| `add C,V` | 添加曲线数据点 | `add c0,50` |
| `cle C` | 清除曲线 | `cle c0` |
| `vis N,F` | 显示/隐藏控件 | `vis t0,1` |
| `click B` | 触发按钮事件 | `click b0` |

---

## 文件说明

### 核心文件

| 文件 | 说明 |
|------|------|
| `main.c` | 主程序，包含ADC采样、电流计算、HMI显示 |
| `hmi_uart.c` | HMI串口屏驱动实现 |
| `hmi_uart.h` | HMI串口屏驱动头文件 |

### 配置文件

| 文件 | 说明 |
|------|------|
| `HMI串口屏配置指南.md` | HMI屏详细配置指南 |
| `hmi_screen_project.txt` | HMI屏界面设计说明 |
| `hmi_project.txt` | HMI屏工程配置参考 |

### 工具脚本

| 文件 | 说明 |
|------|------|
| `setup_environment.bat` | 环境配置批处理 |
| `download_tools.ps1` | 软件下载脚本 |

### CCS项目

| 文件 | 说明 |
|------|------|
| `ccs_project/` | 完整的CCS工程目录 |

---

## 开发资源

- MSPM0G3507 数据手册: https://www.ti.com/lit/ds/symlink/mspm0g3507.pdf
- MSPM0 SDK: https://www.ti.com/tool/MSPM0-SDK
- USART HMI 指令集: http://wiki.tjc1688.com/commands/index.html
- 正点原子HMI资料: http://www.openedv.com/ATK-Prod/ATK-HMI/docs/hmi_doc.html

---

## 注意事项

1. **波特率匹配**: MSPM0G3507 和 HMI屏幕 必须使用相同波特率 (默认9600)
2. **电压兼容**: 确认HMI屏幕支持3.3V TTL电平
3. **交叉接线**: MCU的TX接HMI的RX，MCU的RX接HMI的TX
4. **共地**: 确保两个设备有公共地线
5. **指令结束**: 所有指令必须以 0xFF 0xFF 0xFF 结尾
6. **按钮发送**: HMI屏幕按钮事件中需要发送数据到MCU

---

## 故障排除

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 屏幕无响应 | 波特率不匹配 | 检查两端设置 |
| 显示乱码 | 电压不匹配 | 确认3.3V TTL电平 |
| 指令无效 | 缺少结束标志 | 添加0xFF 0xFF 0xFF |
| 按钮无反应 | 未启用触摸 | 检查tsw设置 |
| CCS编译错误 | 缺少SDK | 安装MSPM0 SDK |
| HMI编译失败 | 软件版本不对 | 使用最新版USART HMI |

---

## 更新记录

- V1.0 (2026-07-30): 初始版本，支持电流监测仪基本功能
