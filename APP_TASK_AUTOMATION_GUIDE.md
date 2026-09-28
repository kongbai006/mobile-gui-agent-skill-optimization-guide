# App 打卡/领奖励任务通用方法论

**版本**：v9.6 全经验汇总版
**适用对象**：所有安卓端 GUI Agent、自动化任务脚本开发者

---

## 性能优化架构

### 五级识别策略（按速度排序）
1. **局部指纹缓存**（~10ms）：Activity + 标志性控件匹配，命中直接用缓存坐标，跳过dump
2. **局部 dump**（~2s）：`uiautomator dump --bounds` 只dump指定区域
3. **全局 dump**（~7s）：完整控件树（局部dump失败时兜底）
4. **图像模板匹配**（~100-300ms）：控件树失效时（Canvas按钮、广告图标）
5. **VLM截图 + 人工**（最后兜底）：仅在前四级全部失败时使用

### 核心原则
- **事件驱动等待**：`wait_for_text` 轮询控件树，页面加载完成立即继续
- **任务状态准则（必须遵守）**：任务结束/暂停/出错时，必须调回Agent前台
- **跳转验证准则**：用 `mCurrentFocus` 直接验证（不截屏）
- **截屏原则**：仅在控件树 + 指纹都失败时使用（最后兜底）

---

## 统一入口

```bash
. /data/data/com.your.agent/files/skills/lib/task_common.sh
```

### 可用函数列表
| 函数 | 作用 |
|------|------|
| `wait_for_text "文本" [超时秒]` | 事件驱动等待（保留ui.xml） |
| `wait_for_activity "Activity" [超时秒]` | 等待页面切换 |
| `wait_until_gone "文本" [超时秒]` | 等待控件消失（验证码场景） |
| `return_to_agent "状态"` | 调回Agent前台（必须遵守） |
| `verify_redirect "目标" [秒]` | 跳转验证（不截屏） |
| `tap_coord "x,y"` | 解析坐标并点击 |
| `dump_partial "页面" "bounds" "marker"...` | 局部dump（~2s） |
| `dump_and_save "页面" "marker"...` | 全局dump（兜底，~7s） |
| `smart_recognize "页面" "文本" ...` | 五级降级识别 |
| `match_local_fp "页面" "ui.xml" "marker"...` | 指纹匹配（跳过dump） |
| `get_cached_coord "页面" "控件文本"` | 获取缓存坐标 |

---

## 关键经验（实测沉淀）

### 指纹匹配
- `wait_for_text` 保留ui.xml：找到文本后不删除文件，后续指纹匹配可用
- 指纹命中时跳过dump：省2-7秒
- 指纹未命中时自动降级到局部dump

### 底部Tab点击规则
- ✅ 正确：点 `text="Tab名"` bounds中心
- ❌ 错误：点图标ImageView区域（事件被吸收）
- **铁律：底部tab一律点文本bounds中心点**

### 启动策略
- 第三方App：用 `monkey -p <pkg> -c android.intent.category.LAUNCHER 1`
- Agent本身：用 `am start -n <pkg>/.ui.MainActivity`

### 返回策略
- 从第三方App返回：用 `monkey` 拉前台（推荐）或 `am start`
- ⚠️ `am start` 例外：部分App返回时会触发URL bug崩溃页 → 改用 `keyevent 4` BACK
- 第三方软件内禁止用 `keyevent 4`（BACK），会进入其内部页面

---

## 单任务标准循环

### 阶段一：启动 + 状态判断
```bash
. /data/data/com.your.agent/files/skills/lib/task_common.sh

# 启动目标App
monkey -p <package> -c android.intent.category.LAUNCHER 1 2>/dev/null

# 事件驱动等待主页加载
wait_for_text "首页" 8 || { echo "主页加载超时"; return_to_agent "FAILED"; exit 1; }

# 尝试指纹匹配（跳过dump，~10ms）
if match_local_fp "home_page" /sdcard/ui.xml "首页" "底部tab1"; then
    echo "✅ 指纹命中，跳过 dump"
else
    echo "❌ 指纹未命中，执行局部 dump"
    dump_partial "home_page" "[0,0][1200,500]" "首页" "底部tab1"
fi

# 广告检查
ui_file="/sdcard/ui_partial.xml"
[ -f "$ui_file" ] || ui_file="/sdcard/ui.xml"
if [ -f "$ui_file" ] && grep -q 'text="跳过"' "$ui_file"; then
    tap_coord "$(smart_recognize "ad" "跳过" "$ui_file")"
    wait_for_text "首页" 5
fi
```

### 阶段二：导航到目标页面
```bash
ui_file="/sdcard/ui_partial.xml"
[ -f "$ui_file" ] || ui_file="/sdcard/ui.xml"
coord=$(smart_recognize "home_page" "目标按钮" "$ui_file" "标志性控件")
tap_coord "$coord"

# 跳转验证
verify_redirect "目标package" 2 || { echo "跳转失败"; return_to_agent "FAILED"; exit 1; }
```

### 阶段三：任务循环
```bash
# 点击任务按钮
coord=$(smart_recognize "task_page" "任务按钮" /sdcard/ui.xml "任务按钮")
tap_coord "$coord"

# 事件驱动等待完成信号
if wait_for_text "领取奖励" 15; then
    # 返回目标App
    am start -n <package>/<activity> 2>/dev/null
    wait_for_activity "<activity>" 5 || exit 1
    
    # 局部dump更新任务列表
    dump_partial "task_list" "[0,0][1200,800]" "任务1" "任务2"
    
    remaining=$(grep -c 'text="任务按钮"' /sdcard/ui_partial.xml 2>/dev/null)
    [ "$remaining" -gt 0 ] && echo "NEXT_ROUND" || echo "ALL_DONE"
else
    echo "等待超时"
    return_to_agent "TIMEOUT"
fi
```

### 阶段四：终验 + 调回Agent
```bash
dump_partial "final_page" "[0,0][1200,800]" "完成信号"

ui_file="/sdcard/ui_partial.xml"
[ -f "$ui_file" ] || ui_file="/sdcard/ui.xml"
remaining=$(grep -c 'text="任务按钮"' "$ui_file" 2>/dev/null)

if [ "$remaining" -eq 0 ]; then
    echo "✅ 任务完成"
    return_to_agent "TASK_COMPLETE"
else
    echo "❌ 任务未完成"
    return_to_agent "FAILED"
fi
```

---

## 铁律

### 任务状态（必须遵守）
- 任务结束/暂停/出错必须调回Agent前台
- 禁止：任务失败后停留在其他App

### 事件驱动
- 禁止固定sleep：用 `wait_for_text` / `wait_for_activity` 替代

### 白名单识别
- 第三方任务用白名单（标题同行左侧 + 指定文案前缀）
- 不用固定第三方黑名单（会漏新情况）

### 完成态判定
- 不要用「无按钮=完成」，要匹配明确的完成文本
- 浮层弹窗dump抓不到 → 用底层可控文本判定

### 脚本与文档同步
- 改脚本后必须同步SKILL.md，防版本脱节
- SKILL.md只写「1条命令 + 流程概括 + 状态码表 + 铁律」

---

## 失败处理

- 任何失败：`return_to_agent "状态"` 调回Agent
- 指纹未命中：自动降级到局部dump
- dump超时：重试一次，仍失败调回Agent
- 跳转失败：`verify_redirect` 返回1，调回Agent
- 验证码出现：`wait_until_gone` 轮询，超时调回Agent

---

## Shell常见踩坑（Ash）

| 问题 | 原因 | 解决 |
|------|------|------|
| `echo 1>file` 写入空 | Ash把`1>`当fd重定向 | `echo 1 >file` 加空格 |
| `grep {2,40}` 中文超限 | 按字节计数 | 用 `[^"]+` |
| `timeout sh -c` 丢函数 | 子shell不继承函数 | 前台手动计时 |
| `{px,py}.txt` rm失败 | Android sh不支持花括号展开 | 显式列出文件名 |

---

**文档版本**：v9.6
**更新日期**：2026-09-28
**核心价值**：一套可直接复用的App任务自动化标准模板，包含五级识别、事件驱动、白名单识别、完成态判定等全部最佳实践
