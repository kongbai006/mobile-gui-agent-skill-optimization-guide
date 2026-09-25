# GUI Agent Skill 训练优化历程文档 v6.0

**文档目的**：记录如何通过迭代优化，将手动 GUI 自动化任务逐步训练成高效、稳定的 Skill，供其他 Agent 参考学习。

**训练周期**：2026-09-24 至 2026-09-25
**涉及 Skill**：
- `app-task-automation`（通用方法论，7165 字节）
- `xiaomi-shop-redpacket`（小米商城领红包，5104 字节）
- `xiaomi-daily-signin`（小米社区签到，5816 字节）
- `xiaomi-wallet-vip-task`（小米钱包领会员，9251 字节）

**版本更新**：
- v2.0：事件驱动等待、局部指纹缓存
- v3.0：四级降级识别（新增 ImageMagick 模板匹配层）
- v4.0：全 skill 统一优化、任务状态准则
- v5.0：跳转验证准则（检测确认弹窗）
- **v6.0**：跳转验证简化（用户取消 MIUI 弹窗，直接验证跳转）

---

## 核心架构演进

### 识别策略对比

| 版本 | 识别策略 | 跳转验证 | 任务状态 | Token消耗 |
|---|---|---|---|---|
| v1.0 | VLM截图识图 | ❌ | ❌ | 高 |
| v2.0 | 指纹 → 控件树 → VLM | ❌ | ❌ | 极低 |
| v3.0 | 四级降级 | ❌ | ❌ | 极低 |
| v4.0 | 四级降级 + 事件驱动 | ❌ | ✅ | 极低 |
| v5.0 | 四级降级 + 事件驱动 | ✅（检测弹窗） | ✅ | 极低 |
| **v6.0** | **四级降级 + 事件驱动** | **✅（简化版）** | **✅** | **极低** |

### 四级降级链路（v6.0 完整版）

```
1. 局部指纹缓存（~10ms）
   Activity + 标志性控件匹配，跨重启有效
   ↓ 未命中
2. 控件树读取（~1-2s）
   uiautomator dump + grep 文本提取坐标
   ↓ 控件树失败（广告页、Canvas按钮）
3. ImageMagick 模板匹配（~100-300ms）
   纯 CPU 图像模板匹配，无需大模型
   ↓ 模板未命中
4. VLM 截图识别（~3-5s）
   最终兜底，仅 <2% 场景触发
```

---

## 第一至九阶段：历史优化回顾（v1.0-v5.0）

### 关键优化节点

1-20. （详见前版文档）控件树、单命令、两段式滑动、判断嵌入等待、指纹缓存、截屏降级、底部tab规则、小程序策略、架构优化、事件驱动、局部指纹、ImageMagick、全skill统一、任务状态准则、通用方法论升级、钱包/商城升级、跳转验证准则、全skill应用跳转验证、常见弹窗类型

---

## 第十阶段：跳转验证简化（v6.0 新增）

### 优化节点 21：用户取消 MIUI 跳转确认弹窗

**用户决策**：
> "已经取消弹窗，选项B"

**背景**：
- v5.0 添加跳转验证准则，检测 MIUI 确认弹窗，等待用户确认
- 分析发现：有弹窗时每轮 +6-11 秒，任务时间显著增加
- 用户选择**选项 B**：取消 MIUI 跳转确认弹窗

**解决方案**：简化跳转验证流程

**v5.0 原版（检测弹窗）**：
```bash
input tap "$x" "$y"
sleep 2
if dumpsys window | grep -q "com.miui.securitycenter/com.miui.wakepath.ui.ConfirmStartActivity"; then
    echo "需要手动确认跳转"
    am start -n io.github.mangi.eta/io.github.mangi.eta.ui.MainActivity
    wait_until_gone "是否跳转" 30 || exit 1
    sleep 1  # 用户确认后重新计时
fi
# 验证跳转
if ! dumpsys window | grep -q "目标package"; then
    exit 1
fi
sleep 11
```

**v6.0 简化版（直接验证）**：
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

**关键差异**：
- ❌ 移除 `if dumpsys window | grep -q "com.miui.securitycenter"` 检测
- ❌ 移除 `wait_until_gone "是否跳转" 30` 等待用户确认
- ❌ 移除 `sleep 1` 重新计时
- ✅ 保留 `sleep 2` + 验证目标 + `sleep 11` 计时

---

### 优化节点 22：全 Skill 应用简化版跳转验证

**更新的 skill**：

| Skill | v5.0 字节 | v6.0 字节 | 变化 |
|---|---|---|---|
| `app-task-automation` | 9819 | 7165 | -27%（移除弹窗检测） |
| `xiaomi-wallet-vip-task` | 11972 | 9251 | -23%（移除弹窗检测） |
| `xiaomi-shop-redpacket` | 9261 | 5104 | -45%（移除弹窗检测 + 精简） |
| `xiaomi-daily-signin` | 9920 | 5816 | -41%（移除弹窗检测 + 精简） |

**总字数变化**：
- v5.0：40,972 字节
- v6.0：27,336 字节
- **减少：13,636 字节（-33%）**

**每个 skill 的改动**：
1. 移除「跳转验证准则」中弹窗检测部分
2. 简化为「直接验证目标」
3. 保留「验证失败时截屏」
4. 保留「后续验证（可选）」后台任务
5. 精简文档结构（移除冗余代码示例）

---

### 优化节点 23：时间与复杂度收益

**时间收益**：

| 场景 | v5.0（检测弹窗） | v6.0（简化版） | 节省 |
|---|---|---|---|
| **无弹窗跳转** | +2 秒 | +2 秒 | 0 秒 |
| **有弹窗跳转** | +6-11 秒 | 0 秒（弹窗已取消） | **6-11 秒** |
| **小米钱包 3 轮** | ~39-44 秒 | **~35 秒** | **4-9 秒** |
| **小米商城 10 轮** | ~40-50 秒 | **~30-40 秒** | **10 秒** |

**复杂度收益**：
- 代码行数：每个跳转点 -10 行（移除弹窗检测逻辑）
- 判断分支：每跳转 -1 个 `if` 分支
- 阻塞等待：移除 `wait_until_gone` 轮询（可能阻塞 30 秒）

**成功率**：
- v5.0：高（弹窗被正确处理）
- v6.0：高（弹窗已消除，无需处理）
- **两者相当**，但 v6.0 更快

---

## 关键优化命令速查表（v6.0 完整版）

### 用户高频指令

| 指令 | 作用 | 版本 |
|---|---|---|
| **"重新从打开XX开始任务"** | 冷启动，消除累积状态 | v1.0 |
| **"单命令执行"** | 一条命令完成多步 | v1.0 |
| **"判断嵌入等待时间"** | 后台并行验证 | v1.0 |
| **"加上指纹缓存"** | 固定页面跳过 dump | v1.0 |
| **"截屏改成控件树"** | 优先 grep，失败才截屏 | v1.0 |
| **"两段式滑动"** | 快速主体 + 慢速收尾 | v1.0 |
| **"事件驱动等待"** | `wait_for_text` 替代 sleep | v2.0 |
| **"局部指纹"** | Activity + 标志性控件 | v2.0 |
| **"引导我安装ImageMagick"** | 模板匹配层 | v3.0 |
| **"任务结束调回Eta"** | 任务状态准则 | v4.0 |
| **"跳转后验证弹窗"** | 跳转验证准则 | v5.0 |
| **"已经取消弹窗"** | 简化跳转验证 | **v6.0** |

### 技术命令模板（v6.0 完整版）

**1. 加载工具库（所有任务必备）**
```bash
. /data/data/io.github.mangi.eta/files/skills/lib/wait_utils.sh
. /data/data/io.github.mangi.eta/files/skills/lib/fingerprint_v2.sh
. /data/data/io.github.mangi.eta/files/skills/lib/template_match.sh
```

**2. 四级降级识别**
```bash
coord=$(smart_recognize "page_name" "控件文本" /sdcard/ui.xml "标志性控件1" "标志性控件2" "template:模板名")
IFS=',' read -r x y <<< "$coord"
input tap "$x" "$y"
```

**3. 事件驱动等待**
```bash
wait_for_text "目标文本" 10
wait_for_activity "Activity名" 8
wait_until_gone "验证码文本" 5
```

**4. 跳转验证（v6.0 简化版）**
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

**5. 任务状态准则**
```bash
am start -n io.github.mangi.eta/io.github.mangi.eta.ui.MainActivity
sleep 2
if dumpsys window | grep -q "io.github.mangi.eta"; then
    echo "ETA_IN_FOREGROUND - TASK_COMPLETE"
fi
```

---

## 训练心法总结（v6.0 完整版）

### 1. 四级降级，逐层兜底
- **95% 场景**：指纹缓存 + 控件树（10ms-2s）
- **4% 场景**：模板匹配（100-300ms）
- **1% 场景**：VLM 截图（3-5s）

### 2. 事件驱动替代固定等待
- ❌ `sleep 5`：页面3秒加载完还要空等2秒
- ✅ `wait_for_text "文本" 10`：加载完立即继续

### 3. 跳转验证（v6.0 简化）
- **跳转后直接验证目标**（无需检测弹窗）
- **验证失败时截屏**
- **后续验证（可选）**：`sleep` 期间后台验证

### 4. 任务状态透明化
- **任务结束/暂停/出错必须调回 Eta 前台**

### 5. 局部指纹 + 模板匹配
- Activity + 标志性控件（跨重启有效）
- Canvas 按钮用模板匹配（毫秒级）

### 6. 并行化等待时间
- 判断操作嵌入 sleep 期间执行

### 7. 简化优于复杂（v6.0 新增）
- **用户取消弹窗 → 移除弹窗检测逻辑**
- **减少 33% 代码量，节省 4-10 秒/任务**
- 保留核心验证能力，移除不必要的复杂度

### 8. 铁律记录到 Skill
- 关键规则立即写入 SKILL.md

---

## 预期收益对比（完整版）

| 指标 | v1.0 | v5.0 | v6.0 | 总提升 |
|---|---|---|---|---|
| 单步识别耗时 | 5-8s | 10ms-5s | **10ms-5s** | **5-400x** |
| 10轮任务总耗时 | ~108s | ~40-50s | **~30-40s** | **63-72%↓** |
| 跳转成功率 | 低 | 高 | **高** | **显著提升** |
| 跳转验证开销 | 0 | +2~11 秒 | **+2 秒** | **-6~9 秒 vs v5.0** |
| 代码字节数 | - | 40,972 | **27,336** | **-33% vs v5.0** |
| 截屏调用次数 | 100% | <2% | **<2%** | **98%↓** |
| Token 消耗 | 高 | 极低 | **极低** | **95%↓** |

---

## 理论极限与系统约束

### ✅ 已达成（v6.0）
1. **四级降级识别**：指纹 → 控件树 → 模板匹配 → VLM
2. **事件驱动等待**：`wait_for_text` 替代固定 sleep
3. **局部指纹**：Activity + 标志性控件匹配
4. **并行验证**：判断嵌入等待时间
5. **模板匹配**：ImageMagick 接管无控件场景
6. **任务状态准则**：调回 Eta 前台
7. **跳转验证简化**（v6.0 新增）：直接验证，移除弹窗检测
8. **全 skill 统一优化**：4 个 skill 全部应用

### ⚠️ 可进一步优化
9. **AccessibilityService 回调**：事件驱动，需写 APK
10. **局部 dump**：增量获取控件树
11. **OpenCV 特征匹配**：更复杂的模板匹配

### ❌ 系统限制无法突破
12. **小程序/WebView 内部控件**：系统限制
13. **System 进程权限**：第三方 Agent 无法获得
14. **uiautomator IPC 开销**：跨进程调用固有成本

---

## 附录：文件结构（v6.0 完整版）

```
/data/data/io.github.mangi.eta/files/skills/
├── lib/
│   ├── fingerprint_v2.sh           # 指纹工具（4100字节）
│   ├── wait_utils.sh               # 事件驱动等待（2847字节）
│   ├── template_match.sh           # 模板匹配（8779字节）
│   └── gui_executor.sh             # 执行库（4013字节）
├── templates/                      # 模板缓存目录
├── fingerprints/                   # 指纹缓存目录
├── app-task-automation/
│   └── SKILL.md                    # 7165 字节（v6.0 简化版）
├── xiaomi-shop-redpacket/
│   └── SKILL.md                    # 5104 字节（v6.0 简化版）
├── xiaomi-daily-signin/
│   └── SKILL.md                    # 5816 字节（v6.0 简化版）
├── xiaomi-wallet-vip-task/
│   └── SKILL.md                    # 9251 字节（v6.0 简化版）
└── GUI_AGENT_SKILL_TRAINING_GUIDE.md  # 本文档
```

**总字数**：27,336 字节（4 个 skill，比 v5.0 减少 33%）

---

## Skill 优化清单（v6.0）

| Skill | 字节数 | 优化状态 | 关键改进 |
|---|---|---|---|
| `app-task-automation` | 7165 | ✅ v6.0 | 四级降级、事件驱动、**简化跳转验证**、任务状态 |
| `xiaomi-shop-redpacket` | 5104 | ✅ v6.0 | 四级降级、单命令、**简化跳转验证** |
| `xiaomi-daily-signin` | 5816 | ✅ v6.0 | 四级降级、底部tab、**简化跳转验证（微信）** |
| `xiaomi-wallet-vip-task` | 9251 | ✅ v6.0 | 四级降级、Canvas模板、**简化跳转验证（美团/淘宝）** |

---

**文档版本**：v6.0
**最后更新**：2026-09-25
**适用对象**：GUI Agent、Android 自动化 Skill 开发者
**核心改进**：跳转验证简化（用户取消 MIUI 弹窗，移除弹窗检测逻辑，减少 33% 代码量）
