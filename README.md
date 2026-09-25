# 移动端 GUI Agent 技能优化训练指南

> 从零开始，如何把一个慢、贵、不稳定的手机自动化Agent，一步步训练成低Token、高速度、高准确率的高效执行体。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Platform](https://img.shields.io/badge/Platform-Android-green.svg)]()
[![Skill](https://img.shields.io/badge/Skill-GUI%20Agent-blue.svg)]()
[![Version](https://img.shields.io/badge/Version-v7.6-orange.svg)]()

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
│  2. 局部 dump（~2s）                    │
│     uiautomator dump --bounds 区域截取    │
└──────────────┬──────────────────────────┘
               ↓ 局部dump失败
┌─────────────────────────────────────────┐
│  3. 全局 dump（~7s）                    │
│     完整控件树读取，grep文本匹配          │
└──────────────┬──────────────────────────┘
               ↓ 控件树失败（广告页/Canvas）
┌─────────────────────────────────────────┐
│  4. 模板匹配（~100-300ms）               │  ← 0.9%场景
│     ImageMagick 图像模板比对              │
└──────────────┬──────────────────────────┘
               ↓ 模板未命中
┌─────────────────────────────────────────┐
│  5. VLM截图识别（~3-5s）                 │  ← 0.1%场景
│     大模型视觉理解，最终兜底               │
└─────────────────────────────────────────┘
```

### 设计哲学

| 原则 | 说明 |
|------|------|
| **能缓存绝不解析** | 固定页面用指纹缓存，毫秒级命中 |
| **能局部绝不全局** | 局部dump只截取目标区域，比全局dump快3倍 |
| **能控件树绝不截图** | 无障碍控件树读取，替代视觉推理 |
| **能模板匹配绝不用大模型** | 无控件场景用图像比对，毫秒级识别 |
| **事件驱动替代固定等待** | 页面加载完成立即执行，不空等 |
| **渐进式识别** | 先text后content-desc，保持性能又兼容更多控件 |
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

### 第六阶段：全Skill统一优化（v4.0-v6.0）

**v4.0 任务状态准则**：
- 任务结束/暂停/出错时，必须调回Eta前台，显示执行状态
- 用户体验大幅提升，任务状态透明化

**v5.0 跳转验证准则**：
- 检测MIUI确认弹窗，验证目标页面是否成功打开
- 跳转成功率显著提升

**v6.0 跳转验证简化**：
- 用户取消MIUI弹窗后，直接移除检测逻辑
- 代码量减少33%，每轮任务省6-11秒

---

### 第七阶段：工程化架构升级（v7.0-v7.5）

#### v7.0 统一入口 task_common.sh
- 把所有工具函数整合到一个入口文件
- 每个Skill只需要一行加载：`. lib/task_common.sh`
- 减少重复代码，统一维护

#### v7.1 任务排除准则
- 自动识别并跳过需要跳转到其他APP的任务
- 避免不必要的跳转和等待

#### v7.2 全面事件驱动
- 所有固定`sleep`全部替换为`wait_for_text`
- 页面加载完成立即执行，消除无效等待

#### v7.3 指纹优化（命中时跳过dump）
- 指纹命中后直接复用坐标，跳过整个dump过程
- 每次命中节省 **2-7秒**

#### v7.4 局部dump（--bounds参数）
- 不再全页面dump，只截取目标区域
- 局部dump ~2s vs 全局dump ~7s，**速度提升3倍**

#### v7.5 指纹修复
- `wait_for_text`执行后保留ui.xml，后续指纹匹配可直接复用
- 指纹命中率显著提升

---

### 第八阶段：content-desc 支持（v7.6 最新）

**问题**：小米钱包「领视频会员」是`content-desc`属性，`text`为空，导致识别失败。

```xml
<!-- 钱包「领视频会员」控件 -->
<node content-desc="领视频会员" bounds="[0,1356][1200,1763]" clickable="true" ... />
<!-- text=""  ← text 属性为空！ -->
```

**解决方案**：渐进式识别
```bash
# 第 1 步：先用 text 搜索（快，大多数控件）
bounds=$(grep ... text="目标文本" ...)

# 第 2 步：如果 text 未找到，用 content-desc 兜底
if [ -z "$bounds" ]; then
    bounds=$(grep ... content-desc="目标文本" ...)
fi
```

**关键优势**：
- ✅ 保持性能：大多数控件用`text`（快）
- ✅ 支持content-desc：钱包等App可识别
- ✅ 不影响其他skill：渐进式，不改变原有逻辑

**测试结果**：
- ✅ `smart_recognize "领视频会员"` → 返回坐标 (600, 980)
- ✅ `wait_for_text "领视频会员"` → 成功识别（6.5秒）
- ✅ 指纹命中（之前失败）

---

## 📊 优化效果对比

| 指标 | v1.0 纯VLM版 | v6.0 四级降级版 | v7.6 五级降级版 | 总提升 |
|------|-------------|---------------|---------------|--------|
| 单步识别耗时 | 5-8s | 10ms-5s | **10ms-7s** | **5~400倍** |
| 10轮任务总耗时 | ~108s | ~30-40s | **~16s** | **85%↓** |
| 识别链路层级 | 1层（纯VLM） | 4层 | **5层** | 更精细 |
| 局部dump支持 | ❌ | ❌ | **✅** | 速度提升3倍 |
| content-desc支持 | ❌ | ❌ | **✅** | 兼容更多APP |
| 指纹命中跳过dump | ❌ | ❌ | **✅** | 省2-7s/次 |
| 统一入口 | ❌ | ❌ | **✅** | 维护成本大降 |
| 截屏调用占比 | 100% | <2% | **<2%** | **下降98%** |
| Token消耗 | 高 | 极低 | **极低** | **下降95%** |
| 识别准确率 | 70% | 97% | **97%+** | **提升27%+** |
| Skill代码总量 | - | 27,336字节 | **33,432字节** | 功能增强但逻辑精简 |

### 性能区间分布

- **最优场景**（指纹命中）：**~10ms**
- **局部dump**：**~2s**（比全局快3倍）
- **全局dump**：**~7s**（兜底）
- **广告页**（模板匹配）：**~100-300ms**
- **兜底场景**（VLM）：**~3-5s**（<2%场景）

### 实测数据（2026-09-25）

**小米商城领红包**：
- 阶段一：4.3s（指纹命中，跳过dump）
- 阶段二：9.2s（直接下滑 + 局部dump）
- **总耗时：~16s**
- 任务完成：44次，「去浏览」= 0

---

## 🔧 关键技术方案

### 1. 五级降级识别引擎（v7.6）

```bash
smart_recognize() {
    # 1. 局部指纹（~10ms）
    coord=$(fingerprint_v2.sh get "$page" "$text")
    [ -n "$coord" ] && echo "$coord" && return

    # 2. 局部 dump（~2s）
    uiautomator dump --bounds "[0,0][1200,800]" /sdcard/ui_partial.xml
    bounds=$(grep "text=\"$text\"" /sdcard/ui_partial.xml)

    # 2b. content-desc 兜底（v7.6新增）
    if [ -z "$bounds" ]; then
        bounds=$(grep "content-desc=\"$text\"" /sdcard/ui_partial.xml)
    fi
    [ -n "$bounds" ] && extract_center && return

    # 3. 全局 dump 兜底（~7s）
    uiautomator dump /sdcard/ui_full.xml
    bounds=$(grep "text=\"$text\"" /sdcard/ui_full.xml)
    [ -n "$bounds" ] && extract_center && return

    # 4. 模板匹配（~100-300ms）
    coord=$(find_template "$template_name")
    [ $? -eq 0 ] && echo "$coord" && return

    # 5. VLM兜底（~3-5s）
    echo "FALLBACK_VLM"
}
```

### 2. 统一入口加载（一行搞定）

```bash
# 所有任务只需要这一行，加载全部工具函数
. /data/data/io.github.mangi.eta/files/skills/lib/task_common.sh
```

### 3. 事件驱动等待（支持content-desc）

```bash
wait_for_text "目标文本" 10      # 等待文本出现（先text后content-desc）
wait_for_activity "Activity名" 8  # 等待页面切换
wait_until_gone "验证码文本" 5    # 等待元素消失
```

### 4. 指纹优化（命中跳过dump）

```bash
if match_local_fp "page_name" /sdcard/ui.xml "marker1" "marker2"; then
    echo "指纹命中，跳过 dump"
else
    dump_partial "page_name" "[0,0][1200,800]" "marker1" "marker2"
fi
```

### 5. 任务状态准则

```bash
return_to_eta "TASK_COMPLETE"
# 必须遵守：任务结束/暂停/出错调回 Eta 前台
```

---

## 🔮 未来优化方向

### ✅ 已达成（v7.6）
1. 五级降级识别：指纹 → 局部dump → 全局dump → 模板 → VLM
2. content-desc支持：先text后content-desc渐进式识别
3. 事件驱动等待：wait_for_text替代固定sleep
4. 局部指纹：Activity + 标志性控件匹配
5. 局部dump：--bounds参数，速度提升3倍
6. 指纹优化：命中时跳过dump，省2-7秒
7. 统一入口：task_common.sh，一行加载
8. 模板匹配：ImageMagick接管无控件场景
9. 任务状态准则：调回Eta前台
10. 跳转验证简化：直接验证目标

### ⚠️ 可进一步优化
11. **AccessibilityService回调**：需开发独立APK，真正事件驱动
12. **OpenCV特征匹配**：更复杂的模板匹配，提升复杂场景适配
13. **WebView专项优化**：针对小程序、内嵌网页场景做专门处理

### ❌ 系统限制无法突破
- 小程序/WebView内部控件：系统限制，第三方无法获取
- System进程权限：小爱同学级别的特权，第三方Agent拿不到
- uiautomator IPC开销：跨进程调用固有成本

---

## 📁 项目结构

```
mobile-gui-agent-skill-optimization-guide/
├── README.md                              # 本文档
├── GUI_AGENT_SKILL_TRAINING_GUIDE.md      # 完整训练历程详细记录（v7.6）
├── SHORT_INTRO.md                          # 200字简介（可用于社区分享）
├── lib/                                   # 核心工具库
│   ├── task_common.sh                      # 统一入口（6025字节）
│   ├── wait_utils.sh                       # 事件驱动等待（3295字节）
│   ├── fingerprint_v2.sh                   # 局部指纹工具（4100字节）
│   ├── template_match.sh                   # 模板匹配工具（8779字节）
│   └── gui_executor.sh                     # 执行库（4013字节）
├── examples/                               # 示例Skill
│   ├── app-task-automation/               # 通用方法论（7571字节）
│   ├── xiaomi-shop-redpacket/             # 小米商城领红包（8977字节）
│   ├── xiaomi-daily-signin/               # 小米社区签到（8499字节）
│   └── xiaomi-wallet-vip-task/           # 小米钱包领会员（8385字节）
└── templates/                             # 模板缓存目录示例
```

### 当前Skill清单（v7.6）

| Skill | 字节数 | 关键改进 |
|-------|--------|----------|
| `app-task-automation` | 7,571 | 通用方法论，五级降级，统一入口 |
| `xiaomi-wallet-vip-task` | 8,385 | content-desc支持，Canvas模板，指纹优化 |
| `xiaomi-daily-signin` | 8,499 | 指纹优化，底部tab，事件驱动 |
| `xiaomi-shop-redpacket` | 8,977 | 四级降级，任务排除，局部dump |

**总计**：33,432 字节（4个skill）

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
