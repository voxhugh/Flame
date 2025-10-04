<h1 align='center'><span style="color: orange">装机配置 </span></h1>

| 硬&nbsp;&nbsp;件 |                    型号                    |   品牌   | 价格 |
| :--: | :----------------------------------------: | :------: | :--: |
| 显卡 |            RTX 5080 Ultra W OC             |  七彩虹  | 8900 |
| CPU  |             AMD Ryzen 7 9700X              |   AMD    | 1700 |
| 电源 |            振华 LEADEX VP 1000W            |   振华   | 688  |
| 内存 | 金士顿 FURY Beast DDR5 6000MHz 32GB(16G×2) |  金士顿  | 948  |
| 固态 |            致钛 TiPlus7100 2TB             |   致钛   | 908  |
| 主板 |        微星 B850M GAMING PLUS WIFI         |   微星   | 1400 |
| 机箱 |          联力 包豪斯 O11D Mini V2          |   联力   | 580  |
| 散热 |       利民 Phantom Spirit 120 White        |   利民   | 340  |
| 外显 |             泰坦军团 P275MV-A              | 泰坦军团 | 1060 |

---

## 技术栈

### **1. 游戏开发**

<span style="color: green">**C#**</span>

- **Unity 核心脚本语言**：直接驱动引擎生命周期（`Start()`/`Update()`），3 行代码实现物体移动，5 行代码完成碰撞交互，是连接场景、组件与玩法逻辑的唯一官方桥梁
- **深度绑定引擎功能**：原生操控`Transform`、`Rigidbody`等组件，直接调用物理 / 动画系统，通过`ScriptableObject`管理配置，无需适配层即可发挥 Unity 全部特性
- **覆盖全开发流程**：从角色 AI、UI 交互到编辑器工具、跨平台适配（如`#if UNITY_IOS`），贯穿游戏开发始终，是制作《帕斯卡契约》类 3D 动作游戏的基础

------


### **2. 日常脚本**

<span style="color: pink">**Python**</span>

- **极简语法哲学**：用`requests`库3行代码完成HTTP请求，`pandas`5行代码处理Excel复杂逻辑

- **胶水语言特性**：
  - 系统管理：用`os/subprocess`调用Shell命令，`psutil`监控硬件状态

  - 自动化：`selenium`控制浏览器，`pyautogui`模拟键鼠操作

- **AI增强脚本**：结合`OpenAI API`或本地运行的`Llama.cpp` ，实现自然语言生成自动化脚本

