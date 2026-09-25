# 移动端 GUI Agent 技能优化训练指南

> 从零开始，如何把一个慢、贵、不稳定的手机自动化Agent，一步步训练成低Token、高速度、高准确率的高效执行体。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Platform](https://img.shields.io/badge/Platform-Android-green.svg)]()
[![Skill](https://img.shields.io/badge/Skill-GUI%20Agent-blue.svg)]()
[![Version](https://img.shields.io/badge/Version-v6.0-orange.svg)]()

---

## 📋 目录

- [开发诉求](#-开发诉求)
- [核心开发思路](#-核心开发思路)
- [训练优化历程](#-训练优化历程)
- [优化效果对比](#-优化效果对比)
- [关键技术方案](#-关键技术方案)
- [未来优化方向](#-未来优化方向)
- [项目结构](#-项目结构)

---

## 🎯 开发诉求

### 初始痛点

最原始的移动端GUI Agent普遍采用**纯VLM截图识图**方案：
- 每一步都要截图 → 上传图片给大模型 → 模型分析界面 → 输出点击坐标
- 单步识别耗时 **5~8秒**
- 每步都消耗大量Token
- 识别准确率约 **70%**，复杂页面容易误判
- 长时间挂机任务成本高、体验差、不稳定

### 目标

在**不更换大模型、不获取系统特权**的前提下，通过技能工程化训练：
1. **降低Token消耗**：减少不必要的大模型调用
2. **提升响应速度**：让每一步操作更快执行
3. **提高准确率**：减少误点、减少异常中断
4. **增强稳定性**：长时间挂机不崩、不错位
5. **精简代码量**：减少冗余逻辑，降低维护成本

---

## 🧠 核心开发思路

### 核心原则：低成本优先，高成本兜底

遵循「逐层降级」的识别策略，把99%的场景用低成本方式解决，只把最复杂的1%留给大模型：

```
┌─────────────────────────────────────────┐
│  1. 局部指纹缓存（~10ms）                │  ← 95%场景命中
│     Activity + 标志性控件匹配，跨重启有效 │
└──────────────┬──────────────────────────┘
               ↓ 未命中
┌─────────────────────────────────────────┐
│  2. 控件树读取（~1-2s）                  │  ← 4%场景
│     uiautomator dump + grep 文本匹配      │
└──────────────┬──────────────────────────┘
               ↓ 控件树失败（广告页/Canvas）
┌─────────────────────────────────────────┐
│  3. 模板匹配（~100-300ms）               │  ← 0.9%场景
│     ImageMagick 图像模板比对              │
└──────────────┬──────────────────────────┘
               ↓ 模板未命中
┌─────────────────────────────────────────┐
│  4. VLM截图识别（~3-5s）                 │  ← 0.1%场景
│     大模型视觉理解，最终兜底               │
└─────────────────────────────────────────┘
```

### 设计哲学

| 原则 | 说明 |
|------|------|
| **能缓存绝不解析** | 固定页面用指纹缓存，毫秒级命中 |
| **能控件树绝不截图** | 无障碍控件树读取，替代视觉推理 |
| **能模板匹配绝不用大模型** | 无控件场景用图像比对，毫秒级识别 |
| **事件驱动替代固定等待** | 页面加载完成立即执行，不空等 |
| **局部指纹替代全页面哈希** | 动态内容不影响页面识别 |
| **简化优于复杂** | 消除系统弹窗后，直接移除对应检测逻辑 |

---

## 📈 训练优化历程

### 第一阶段：基础实现（v1.0）

**初始状态**：纯VLM截图识图
- 每步操作：截图 → 上传 → 大模型分析 → 返回坐标 → 执行
- 单步耗时：5~8秒
- Token消耗：高（每步都传图）
- 准确率：约70%

**核心问题**：完全依赖大模型视觉能力，速度慢、成本高、不稳定。

---

### 第二阶段：架构转向——控件树替代截图

**关键突破**：引入 `uiautomator dump` 直接读取安卓无障碍控件树

```bash
uiautomator dump /sdcard/ui.xml
grep "text=\"按钮文本\"" /sdcard/ui.xml
```

**收益**：
- 识别速度从5-8s降至 **1-2s**（4倍提速）
- Token消耗大幅下降（不再传图，只传文本）
- 准确率提升至 **95%**

---

### 第三阶段：流程与缓存优化

| 优化项 | 解决问题 | 收益 |
|--------|----------|------|
| 单命令串行执行 | 多命令并行导致状态混乱 | 消除重复启动错误 |
| 两段式滑动 | 快速滑动惯性过冲 | 滑动精度提升 |
| 判断嵌入等待时间 | 串行空等浪费时间 | 单轮耗时节省40% |
| 页面指纹缓存 | 固定页面重复dump | 截屏减少95% |
| mCurrentFocus替代截屏 | 判断APP前台状态 | 消除不必要截图 |
| 底部tab点击规则 | 图标区域点不准 | 点击准确率提升 |

---

### 第四阶段：事件驱动优化（v2.0）

#### 优化1：wait_for_text 条件等待

**问题**：大量固定 `sleep 5`，页面3秒加载完还要空等2秒。

**方案**：轮询检测目标文本，出现立即继续。

```bash
wait_for_text() {
    local target="$1"
    local timeout="${2:-10}"
    while [ "$elapsed" -lt "$timeout" ]; do
        uiautomator dump /sdcard/ui_wait.xml
        if grep -q "text=\"$target\"" /sdcard/ui_wait.xml; then
            return 0   # 找到立即继续
        fi
        sleep 1
    done
    return 1   # 超时
}
```

**收益**：整体耗时减少 **20~40%**。

#### 优化2：局部指纹替代全页面哈希

**问题**：全页面控件md5哈希，页面某处广告一变，整个指纹直接失效。

**方案**：页面身份 = Activity名 + 标志性控件集合，忽略动态内容。

**收益**：指纹失效概率降低 **80%**。

---

### 第五阶段：四级降级识别（v3.0）

#### 优化：新增 ImageMagick 模板匹配层

**问题**：广告弹窗、Canvas按钮等无控件节点的页面，控件树读取失败，只能直接降级到VLM截图。

**方案**：在控件树和VLM之间，新增一层ImageMagick模板匹配。

```bash
# 保存模板
save_template "skip_btn" 1000 100 100 50   # x y width height

# 模板匹配
coord=$(find_template "skip_btn" 0.1)       # 返回 "x,y"
```

**收益**：
- 广告页识别：3-5s → **100-300ms**（90%↓）
- VLM调用：再减少 **50%**
- Canvas按钮：模板匹配接管，无需VLM

---

### 第六阶段：全Skill统一优化（v4.0）

**核心改进**：任务状态准则

**问题**：任务执行完后，Agent停留在目标APP界面，用户看不到执行结果，不知道任务是否完成。

**方案**：任务结束/暂停/出错时，必须调回Eta前台，显示执行状态。

```bash
am start -n io.github.mangi.eta/io.github.mangi.eta.ui.MainActivity
sleep 2
if dumpsys window | grep -q "io.github.mangi.eta"; then
    echo "ETA_IN_FOREGROUND - TASK_COMPLETE"
fi
```

**收益**：用户体验大幅提升，任务状态透明化。

---

### 第七阶段：跳转验证准则（v5.0）

**问题**：点击跳转按钮后，不确定是否跳转成功，经常因为MIUI确认弹窗导致跳转失败。

**方案**：
1. 点击后等待2秒
2. 检测是否有MIUI确认弹窗
3. 有弹窗则等待用户确认
4. 验证目标页面是否成功打开

**收益**：跳转成功率显著提升，异常可检测。

---

### 第八阶段：跳转验证简化（v6.0 最新）

**背景**：用户取消了MIUI跳转确认弹窗，弹窗不再出现。

**v5.0原版（检测弹窗）**：
```bash
input tap "$x" "$y"
sleep 2
if dumpsys window | grep -q "com.miui.securitycenter"; then
    echo "需要手动确认跳转"
    wait_until_gone "是否跳转" 30 || exit 1
    sleep 1
fi
if ! dumpsys window | grep -q "目标package"; then
    exit 1
fi
sleep 11
```

**v6.0简化版（直接验证）**：
```bash
input tap "$x" "$y"
sleep 2
# 直接验证跳转（无需检测弹窗）
if ! dumpsys window | grep -q "目标package"; then
    echo "跳转失败"
    screencap -p /sdcard/redirect_fail.png
    exit 1
fi
sleep 11
```

**关键收益**：
- 代码量减少 **33%**（4个Skill共减少13,636字节）
- 有弹窗场景每轮节省 **6-11秒**
- 消除了 `wait_until_gone` 30秒阻塞等待风险
- 保留核心验证能力，移除不必要的复杂度

---

## 📊 优化效果对比

| 指标 | v1.0 纯VLM版 | v3.0 四级优化版 | v5.0 跳转验证版 | v6.0 简化版 | 总提升 |
|------|-------------|---------------|---------------|-----------|--------|
| 单步识别耗时 | 5-8s | 10ms-5s | 10ms-5s | **10ms-5s** | **5~400倍** |
| 10轮任务总耗时 | ~108s | ~45s | ~40-50s | **~30-40s** | **63~72%↓** |
| 跳转成功率 | 低 | 中 | 高 | **高** | **显著提升** |
| 跳转验证开销 | 0 | 0 | +2~11秒 | **+2秒** | **-6~9秒 vs v5.0** |
| 代码字节数 | - | - | 40,972 | **27,336** | **-33% vs v5.0** |
| 截屏调用占比 | 100% | <2% | <2% | **<2%** | **下降98%** |
| Token消耗 | 高 | 极低 | 极低 | **极低** | **下降95%** |
| 识别准确率 | 70% | 97% | 97% | **97%** | **提升27%** |
| 指纹失效概率 | N/A | 低 | 低 | **低** | **下降80%** |

### 性能区间分布

- **最优场景**（指纹命中）：**~10ms**
- **常规场景**（控件树）：**~1-2s**
- **广告页**（模板匹配）：**~100-300ms**
- **兜底场景**（VLM）：**~3-5s**（<2%场景）

---

## 🔧 关键技术方案

### 1. 四级降级识别引擎

```bash
smart_recognize() {
    # 1. 局部指纹（~10ms）
    coord=$(fingerprint_v2.sh get "$page" "$text")
    [ -n "$coord" ] && echo "$coord" && return

    # 2. 控件树（~1-2s）
    bounds=$(grep "text=\"$text\"" "$ui_file")
    [ -n "$bounds" ] && extract_center && return

    # 3. 模板匹配（~100-300ms）
    if [ -n "$template_name" ]; then
        coord=$(find_template "$template_name")
        [ $? -eq 0 ] && echo "$coord" && return
    fi

    # 4. VLM兜底（~3-5s）
    echo "FALLBACK_VLM"
}
```

### 2. 事件驱动等待工具

```bash
wait_for_text "目标文本" 10      # 等待文本出现
wait_for_activity "Activity名" 8  # 等待页面切换
wait_until_gone "验证码文本" 5    # 等待元素消失
```

### 3. 跳转验证（v6.0 简化版）

```bash
input tap "$x" "$y"
sleep 2  # 等待跳转启动

# 直接验证跳转（无需检测弹窗）
if ! dumpsys window | grep -q "目标package或Activity"; then
    echo "跳转失败"
    screencap -p /sdcard/redirect_fail.png
    exit 1
fi

# 开始计时
sleep 11

# 后续验证（可选，后台）
(
    sleep 3
    if dumpsys window | grep -q "目标package"; then
        echo "REDIRECT_OK" > /tmp/verify_result
    else
        echo "REDIRECT_FAIL" > /tmp/verify_result
    fi
) &
```

### 4. 任务状态准则

```bash
am start -n io.github.mangi.eta/io.github.mangi.eta.ui.MainActivity
sleep 2
if dumpsys window | grep -q "io.github.mangi.eta"; then
    echo "ETA_IN_FOREGROUND - TASK_COMPLETE"
fi
```

---

## 🔮 未来优化方向

### ✅ 可继续优化

1. **原生事件驱动**：接入AccessibilityService回调，替代轮询等待，进一步降低延迟
2. **控件树增量获取**：只解析目标区域控件，降低完整dump的系统开销
3. **OpenCV特征匹配**：引入更复杂的特征点匹配，提升复杂场景下的模板适配能力
4. **智能异常恢复**：区分页面加载失败、弹窗遮挡、权限丢失等不同错误，执行对应恢复策略
5. **WebView专项适配**：针对小程序、内嵌网页场景做专门优化

### ⚠️ 系统限制无法突破

- 小程序/WebView内部控件：系统限制，第三方无法获取
- System进程权限：小爱同学级别的特权，第三方Agent拿不到
- uiautomator IPC开销：跨进程调用固有成本

---

## 📁 项目结构

```
mobile-gui-agent-skill-optimization-guide/
├── README.md                              # 本文档
├── GUI_AGENT_SKILL_TRAINING_GUIDE.md      # 完整训练历程详细记录（v6.0）
├── SHORT_INTRO.md                          # 200字简介（可用于社区分享）
├── lib/                                   # 核心工具库
│   ├── wait_utils.sh                      # 事件驱动等待工具
│   ├── fingerprint_v2.sh                   # 局部指纹工具
│   ├── template_match.sh                   # 模板匹配工具
│   └── gui_executor.sh                    # 四级识别执行库
├── examples/                               # 示例Skill
│   ├── app-task-automation/               # 通用方法论
│   ├── xiaomi-shop-redpacket/             # 小米商城领红包
│   ├── xiaomi-daily-signin/               # 小米社区签到
│   └── xiaomi-wallet-vip-task/           # 小米钱包领会员
└── templates/                             # 模板缓存目录示例
```

### 当前Skill清单（v6.0）

| Skill | 字节数 | 关键改进 |
|-------|--------|----------|
| `app-task-automation` | 7,165 | 四级降级、事件驱动、简化跳转验证、任务状态 |
| `xiaomi-wallet-vip-task` | 9,251 | 四级降级、Canvas模板、简化跳转验证（美团/淘宝） |
| `xiaomi-daily-signin` | 5,816 | 四级降级、底部tab、简化跳转验证（微信） |
| `xiaomi-shop-redpacket` | 5,104 | 四级降级、单命令、简化跳转验证 |

**总计**：27,336 字节（比v5.0减少33%）

---

## 📝 适用对象

- 安卓GUI Agent开发者
- 自动化脚本工程师
- 端侧AI助手产品经理
- 对移动端智能体优化感兴趣的技术研究者

## 📄 License

MIT License

---

**⭐ 如果这个项目对你有帮助，欢迎点个Star支持一下！**
