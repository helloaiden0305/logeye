# Log 分析模式库

> **Version**: v6.0
> **Last Updated**: 2026-05-06
> **说明**: 合并原 patterns.md + feedback.md + base.txt 历史案例，一站式诊断知识库（已脱敏）

每次分析发现新的场景特征时，追加到此文件。分析前自动加载，用于匹配和加速排查。

---

## 模式索引

| 编号 | 问题类型 | 关键特征 | 案例数 | 置信度 |
|------|---------|---------|--------|--------|
| P001 | Widget 无响应 | LOW_MEMORY → crash → init failed | 1 | 90% |
| P002 | 通话刚接通就被挂断 | CODE_USER_TERMINATED_BY_REMOTE (510) | 1 | 95%+ |
| P003 | 通话挂断不同步 | SET_DISCONNECTED 已发送但 HANGUP 未到底层 | 1 | 待验证 |
| P004 | 来电无提醒 | 静音 + 无震动 + proximity 触发灭屏 | 1 | 100% |
| P005 | 通话字幕二次开启失效 | ASR 引擎未重新拉起 | 1 | 95% |
| P006 | 多方会议显示异常 | 合并时一方提前 DISCONNECTED (cause=45) | 1 | 90% |
| P007 | 短信搜索"删不干净" | 全局搜索索引延迟 Commit | 2 | 95% |
| P008 | 拨号盘白屏/启动慢 | 低内存杀进程 + 冷启动 | 1 | 90% |
| P009 | SIM 卡注册慢 | 开机风暴 + 主线程消息积压 (700+) | 1 | 85% |
| P010 | 通话回音 | 音频路由切 SPEAKER + AEC 未跟上 | 1 | 80% |
| P011 | 来电语音播报不停 | TTS 未随接听打断 | 1 | 90% |
| P012 | 悬浮窗切全屏遮挡异常 | 窗口模式切换重置 CutoutMode | 1 | 95% |
| P013 | 桌面卡片点击无响应 | Widget 空指针 (更新时进程已死) | 2 | 85% |
| P014 | 护眼模式定时变色 | DIVIDED_TIME_CONTROL_START | 1 | 100% |
| P015 | 默认上网卡弹窗 | isDataInService + isWifiConnected 检查 | 1 | 95% |
| P016 | 来电两种界面 | 亮屏时横幅 + 全屏同时触发 | 1 | 85% |
| P017 | 5G 消息与 5G 流量无关 | 关闭 5G 流量仍可发 5G 消息 | 2 | 95% |
| P018 | 通话无声 | 音频库频繁报错 | 1 | 80% |
| P019 | 桌面角标不更新 | 短信已通知但桌面未刷新 | 1 | 90% |
| P020 | 验证码短信显示异常 | 转换后不能自动至底 | 1 | 待验证 |
| P021 | 无法拍照添加头像 | 相机权限/启动异常 | 1 | 待验证 |
| P022 | 通话界面背景异常 | 毛玻璃降级为纯色 | 1 | 90% |
| P023 | 自动录音未触发 | AI 通话介入导致取消 | 1 | 90% |
| P024 | 5G 消息注册失败 | 网络抖动 + 无 DNS 下发 | 2 | 85% |
| P025 | 来电响铃但无提示框 | 未抓到有效现场 log | 1 | 待验证 |
| P026 | 通话自动挂断 | 对端快速挂断 | 1 | 95% |
| P027 | 短信聚合失效 | 克隆后未自动执行 | 1 | 待验证 |
| P028 | 通话设置页面闪烁 | 横竖屏切换沉浸式 | 1 | 待验证 |
| P029 | 通话录音状态栏消失 | 横屏沉浸式遮挡 | 1 | 待验证 |
| P030 | 短信列表抖动 | 输入法卡顿 | 1 | 85% |
| P031 | Monkey 重启测试窗口 crash | shutdownTest + 服务批量死亡 + bad array lengths | 1 | 98% |
| P032 | 简易模式权限弹窗导航栏遮挡 | mNavbarFlag 切换 + WindowToken type=2005 + NavigationBar vis:0→3 | 1 | 90% |

---

## 正模式

### P001 Widget 无响应
- **问题类型**: Widget 无响应
- **日志类型**: logcat
- **关键特征**:
  - `LOW_MEMORY` 或内存压力日志
  - Widget 进程随后出现异常或崩溃
  - 重启后 `CARD_WIDGET initial failed` 或类似的 init failed
  - `system_app_crash` / `dropbox` 中包含 Widget 相关 crash
- **根因链条**: 内存压力 → Widget 进程被杀/崩溃 → 进程重启 → Widget 初始化失败 → 按钮无响应
- **排查关键词**: `LOW_MEMORY`, `CARD_WIDGET initial failed`, `system_app_crash`, `updateAppWidget`, `BroadcastReceiver`
- **置信度**: 90%+
- **案例记录**: 2026-04-27 联系人桌面卡片按钮无响应 ✅

---

### P002 通话刚接通就被挂断
- **问题类型**: 通话挂断类
- **日志类型**: logcat
- **关键特征**:
  - `ImsReasonInfo` 包含 `CODE_USER_TERMINATED_BY_REMOTE (510)`
  - 通话从 ACTIVE 到 DISCONNECTED 仅 1-2 秒
  - proximity 传感器日志在挂断后才出现
- **根因链条**: 对方用户主动挂断 → IMS 发送 510 原因码 → 本端收到 DISCONNECT → 通话终止
- **排查关键词**: `510`, `ImsReasonInfo`, `DisconnectCause`, `hangup`
- **快速过滤策略**: 先搜 `ImsReasonInfo` 或 `DisconnectCause`，看原因码
- **置信度**: 95%+
- **案例记录**: 2026-04-29 通话刚接通就被挂断 ✅

---

### P003 通话挂断不同步
- **问题类型**: 通话挂断不同步
- **日志类型**: logcat + RIL
- **关键特征**:
  - Telecom 层 `SET_DISCONNECTED` 已发送
  - 但 RIL 层未收到 `HANGUP` 命令
  - 底层呼叫流程继续
- **根因链条**: 用户快速挂断 → HANGUP 命令未发送 → 底层继续呼叫流程
- **排查关键词**: `SET_DISCONNECTED`, `HANGUP`, `Telecom`, `ImsPhoneCallTracker`
- **验证状态**: 待更多案例验证
- **案例记录**: 2026-04-28 通话挂断后底层呼叫仍继续 ✅

---

### P004 来电无提醒（用户以为屏幕不亮）
- **问题类型**: 来电通知类
- **日志类型**: logcat
- **关键特征**:
  - `SET_RINGING` 正常触发
  - `wakePowerGroupLocked` 正常唤醒
  - 用户反馈"屏幕没亮"但日志显示正常
  - 最终 `DISCONNECTED (510)` 对方挂断
- **根因链条**: 静音 + 无震动 → 用户未察觉来电 → 拿起手机触发 proximity → 屏幕灭 → 用户以为"来电时没亮"
- **排查关键词**: `SET_RINGING`, `wakePowerGroupLocked`, `proximity`, `静音`, `免打扰`
- **快速过滤策略**:
  1. 确认屏幕是否正常唤醒
  2. 检查用户是否开启静音/免打扰
  3. 检查 proximity 触发时机
- **置信度**: 100%
- **案例记录**: 2026-04-17 来电话屏幕不亮 ✅（用户反馈纠正）

---

### P005 通话字幕二次开启失效
- **问题类型**: AI 通话/语音识别
- **日志类型**: logcat + hilog
- **关键特征**:
  - 首次开启字幕正常，第一句识别成功
  - 用户关闭后再开启，UI 显示"已开启"但无字幕
  - 底层 `stop Recognize` 后未再 `startAsrService`
  - `mCurrentAiCallMode=2` 但 ASR 未拉起
- **根因链条**: 用户关闭字幕 → ASR 引擎停止 → 再次开启时 `updateVoiceState()` 未调用 `startAsrService()` → 引擎未重新初始化
- **排查关键词**: `startAsrService`, `updateVoiceState`, `AI_CALL_MODE_TYPE_REPLY`, `SpeechRecognizerImpl`
- **修复方案**: 在 `updateVoiceState()` 中，当 `mCurrentAiCallMode == AI_CALL_MODE_TYPE_REPLY` 时强制调用 `startAsrService()` 和 `setNeedRecord(true)`
- **置信度**: 95%
- **案例记录**: 通话字幕二次开启失效 ✅

---

### P006 多方会议显示异常
- **问题类型**: 多方通话/会议电话
- **日志类型**: logcat + RIL
- **关键特征**:
  - 合并通话时一方突然 `DISCONNECTED`
  - `cause=45` (IMS_MERGED_SUCCESSFULLY)
  - 会议对象 `childs(0)` (空会议)
  - 界面显示"正在通话"但无号码管理
  - 对方全挂断后本机仍显示 ACTIVE
  - 无通话记录生成
- **根因链条**: 视频降级语音 + 立即拨打第二通 + 合并 → Modem 提前上报 DISCONNECTED (cause=45) → 上层会议对象无子通话 → 状态机脱节
- **排查关键词**: `CONF_WITH`, `MERGE_COMPLETE`, `childs(0)`, `cause=45`, `IMS_MERGED_SUCCESSFULLY`
- **置信度**: 90%
- **案例记录**: 三方会议显示异常 ✅

---

### P007 短信搜索"删不干净"
- **问题类型**: 短信搜索/索引
- **日志类型**: logcat + HnSearchService
- **关键特征**:
  - 用户删除短信后再次搜索，又出现"已删除"的短信
  - 首次搜索 `cursorCount` 与实际显示不一致
  - `SearchServiceClient: deleteByQuery start` 后恢复正常
  - `IndexWriter: close and commit` 触发索引刷新
  - 索引版本号跳变 (如 18085 → 18119)
- **根因链条**: 全局搜索冷启动高负载 → MMS 索引延迟 Commit → 用户删除操作强制刷新索引 → 搜索恢复正常
- **排查关键词**: `deleteByQuery`, `IndexWriter`, `close and commit`, `cache version`, `HnSearchService`
- **快速过滤策略**:
  1. 检查 `HnSearchService` 是否被其他索引任务阻塞
  2. 查看索引版本号是否跳变
  3. 确认 `deleteByQuery` 是否触发强制 Commit
- **置信度**: 95%
- **案例记录**: 短信搜索删不干净 ✅

---

### P008 拨号盘白屏/启动慢
- **问题类型**: 启动性能
- **日志类型**: logcat
- **关键特征**:
  - 启动时 `acore` 进程已死亡
  - 多次 `LOW_MEMORY` 查杀日志
  - 冷启动时同步请求多个 ContentProvider
  - 主线程被跨进程查询卡住
  - `kswapd` 疯狂回收内存
- **根因链条**: 低内存杀进程 → 用户点击冷启动 → 同步请求多个 Provider + 系统 CPU 被内存回收占用 → 主线程卡死 3 秒
- **排查关键词**: `LOW_MEMORY`, `acore`, `ContentProvider`, `kswapd`, `冷启动`
- **置信度**: 90%
- **案例记录**: 拨号盘白屏/启动慢 ✅

---

### P009 SIM 卡注册慢
- **问题类型**: SIM 卡注册
- **日志类型**: logcat + RIL
- **关键特征**:
  - 开机后 SIM 卡状态变化延迟处理 (7 秒+)
  - `BlockMonitor: Total messages: 700+`
  - 大量 `LOW_MEMORY` 查杀
  - `kswapd running duration` 报警
  - `BOOT_COMPLETED` 广播后系统高负载
- **根因链条**: 开机风暴 → 内存耗尽 → CPU/IO 阻塞 → 主线程消息积压 (700+) → SIM 卡状态延迟处理
- **排查关键词**: `UNSOL_RESPONSE_SIM_STATUS_CHANGED`, `BlockMonitor`, `kswapd`, `BOOT_COMPLETED`
- **置信度**: 85%
- **案例记录**: SIM 卡注册慢 ✅

---

### P010 通话回音
- **问题类型**: 音频回音
- **日志类型**: logcat
- **关键特征**:
  - 通话中 `AUDIO_ROUTE` 切换到 `TYPE_SPEAKER`
  - `CARC.pM_USER_SWITCH_SPEAKER`
  - 用户反馈听到自己回音
  - 通话流程正常无异常
- **根因链条**: 用户开启免提 → 麦克风拾取扬声器声音 → AEC 算法未跟上 → 回音
- **排查关键词**: `AUDIO_ROUTE`, `TYPE_SPEAKER`, `USER_SWITCH_SPEAKER`, `AEC`
- **排查方向**: 转音频团队分析 AEC 是否正常工作
- **置信度**: 80%
- **案例记录**: 通话回音 ✅

---

### P011 来电语音播报不停
- **问题类型**: 语音助手/TTS
- **日志类型**: logcat
- **关键特征**:
  - 来电时 `com.android.voiceengine` 开始播报
  - 用户接听后 TTS 未停止
  - 通话结束后才 `disconnect TTS engine`
  - 播报持续整个通话过程
- **根因链条**: 来电触发语音播报 → 接听时未打断 TTS → 通话结束才强制断开
- **排查关键词**: `voiceengine`, `TTS`, `来电播报`, `打断`
- **置信度**: 90%
- **案例记录**: 来电语音播报不停 ✅

---

### P012 悬浮窗切全屏遮挡异常
- **问题类型**: UI 显示/窗口管理
- **日志类型**: logcat
- **关键特征**:
  - 全屏进入页面正常 (toolbar 遮挡内容)
  - 悬浮窗切全屏后内容顶到状态栏
  - `LAYOUT_IN_DISPLAY_CUTOUT_MODE_ALWAYS` 未重新应用
  - `onConfigurationChanged` 未调用 `adaptCutoutModeAlways`
- **根因链条**: 窗口模式切换 → 系统重置 Window 属性 → `onConfigurationChanged` 未重新设置 CutoutMode → 内容延伸失效
- **排查关键词**: `LAYOUT_IN_DISPLAY_CUTOUT_MODE_ALWAYS`, `onMultiWindowModeChanged`, `CutoutMode`
- **修复方案**: 在 `onMultiWindowModeChanged` 或 `onConfigurationChanged` 中重新应用 CutoutMode
- **置信度**: 95%
- **案例记录**: AI 通话设置悬浮窗遮挡异常 ✅

---

### P013 桌面卡片点击无响应
- **问题类型**: Widget/空指针
- **日志类型**: logcat
- **关键特征**:
  - 点击桌面卡片无响应
  - `NullPointerException: Attempt to invoke virtual method 'void android.appwidget.AppWidgetManager.updateAppWidget' on a null object reference`
  - `SparseArray.size()` 空指针
  - 长时间待机后复现
- **根因链条**: 待机时进程被杀 → Widget Provider 启动时 `AppWidgetManager` 为 null → 空指针异常
- **排查关键词**: `AppWidgetManager`, `NullPointerException`, `CardWidgetManager`, `updateAppWidget`
- **置信度**: 85%
- **案例记录**: 桌面卡片点击无响应 ✅

---

### P014 护眼模式定时变色
- **问题类型**: 显示/护眼模式
- **日志类型**: logcat
- **关键特征**:
  - 固定时间 (如 22:00) 屏幕变色
  - `DIVIDED_TIME_CONTROL_START`
  - `DE_SCENE_ACTION_EYEPROTECTION_TIME_ON`
  - 色温调整至 5600K 左右 (偏暖)
- **根因链条**: 定时护眼任务触发 → 系统下发开启指令 → 调整色温
- **排查关键词**: `DIVIDED_TIME_CONTROL`, `EYEPROTECTION`, `色温`
- **置信度**: 100%
- **案例记录**: 护眼模式定时变色 ✅

---

### P015 默认上网卡弹窗
- **问题类型**: 数据网络/弹窗
- **日志类型**: logcat
- **关键特征**:
  - 数据卡无服务时弹窗提示切换
  - `isDataInService` 检查失败
  - `isWifiConnected` 检查通过
  - `isSwitchDialogShown` 防重复弹窗
- **根因链条**: 默认数据卡无服务 → 满足 7 项校验条件 → 触发弹窗提示切换副卡
- **排查关键词**: `isDataInService`, `isSwitchDialogShown`, `切换数据卡`
- **置信度**: 95%
- **案例记录**: 默认上网卡弹窗 ✅

---

### P016 来电两种界面
- **问题类型**: 来电通知类
- **日志类型**: logcat
- **关键特征**:
  - 亮屏状态下同时触发横幅通知和全屏界面
  - `mScreenOn = true` 但 `InCallActivity` 仍被拉起
  - `enqueueNotificationInternal` 发送高优先级通知
  - `START u0 {act=android.intent.action.MAIN ... cmp=com.android.incallui/.InCallActivity}`
  - 用户反馈"两种 UI 界面同时出现"
- **根因链条**: 亮屏来电 → 系统发横幅通知 → 同时错误拉起全屏 InCallActivity → UI 重叠
- **排查关键词**: `mScreenOn`, `HeadsUp`, `InCallActivity`, `FullScreenIntent`, `START`
- **置信度**: 85%
- **案例记录**: 来电两种界面 ✅

---

### P017 5G 消息与 5G 流量无关
- **问题类型**: 5G 消息/RCS
- **日志类型**: logcat + RIL
- **关键特征**:
  - 关闭 5G 流量开关后仍可发送 5G 消息
  - 状态栏显示 4G 但短信气泡为蓝色 (5G 消息)
  - `SETUP_DATA_CALL` 成功，IMS 通道在 4G 下正常工作
  - RCS 消息发送成功，运营商回执正常
- **根因链条**: 关闭 5G 流量 → 手机切换 4G 基站 → IMS 数据通道仍连通 → RCS 业务在 4G 下正常运行
- **排查关键词**: `RCS`, `IMS`, `SETUP_DATA_CALL`, `5G 消息`, `pcscf`
- **置信度**: 95%
- **案例记录**: 5G 消息与 5G 流量无关 ✅
- **设计说明**: 5G 消息走 IMS 通道，与 5G 流量开关独立，4G 网络下也可使用

---

### P018 通话无声
- **问题类型**: 音频/通话
- **日志类型**: logcat
- **关键特征**:
  - 通话中听不到对方声音
  - 音频库频繁报错
  - `AudioFlinger: RecordThread: buffer overflow`
  - 通话流程正常无异常
- **根因链条**: 音频驱动/库异常 → 音频数据无法正常传输 → 通话无声
- **排查关键词**: `AudioFlinger`, `buffer overflow`, `音频库`, `AudioService`
- **排查方向**: 转音频团队分析音频库/驱动问题
- **置信度**: 80%
- **案例记录**: 通话无声 ✅

---

### P019 桌面角标不更新
- **问题类型**: 桌面/通知
- **日志类型**: logcat
- **关键特征**:
  - 短信已读但桌面红点长时间不消失
  - `LauncherProvider: launcher call - method:change_badge` 已触发
  - `BadgeContentObserver: newBadgeNumber=X` 已更新
  - 桌面 UI 未刷新角标显示
- **根因链条**: 短信应用已通知桌面更新角标 → 桌面进程未及时刷新 UI → 角标显示延迟
- **排查关键词**: `LauncherProvider`, `change_badge`, `BadgeContentObserver`, `角标`
- **排查方向**: 转桌面团队分析角标刷新机制
- **置信度**: 90%
- **案例记录**: 桌面角标不更新 ✅

---

### P020 验证码短信显示异常
- **问题类型**: 短信/UI 显示
- **日志类型**: logcat
- **关键特征**:
  - 验证码短信转换后不能自动至底
  - 输入法键盘弹出时页面位置异常
  - `IME_SHOW_OR_HIDE` 触发时列表未滚动
- **根因链条**: 输入法弹出 → 页面高度变化 → 列表未自动滚动到底部
- **排查关键词**: `IME_SHOW_OR_HIDE`, `输入法`, `自动滚动`, `验证码`
- **置信度**: 待验证
- **案例记录**: 验证码短信显示异常

---

### P021 无法拍照添加头像
- **问题类型**: 联系人/相机
- **日志类型**: logcat
- **关键特征**:
  - 点击拍照添加联系人头像无响应
  - 相机权限检查通过但启动失败
  - 9.0 系统有共性 DTS
- **根因链条**: 相机启动异常/权限问题 → 无法调用相机拍照
- **排查关键词**: `Camera`, `权限`, `联系人头像`, `拍照`
- **置信度**: 待验证
- **案例记录**: 无法拍照添加头像 ✅

---

### P022 通话界面背景异常
- **问题类型**: UI 显示/通话
- **日志类型**: logcat
- **关键特征**:
  - 通话界面点击"更多"后背景显示纯色而非毛玻璃
  - `more_dialog_shape_bg` 兜底背景被使用
  - `isDeviceBlurAbilityOn()` 返回 false 或毛玻璃逻辑未执行
  - 对比机现象一致或界面效果不好
- **根因链条**: 设备不支持实时模糊/性能降级 → 毛玻璃效果降级为纯色背景
- **排查关键词**: `毛玻璃`, `more_dialog_shape_bg`, `ColorDrawable`, `TRANSPARENT`
- **置信度**: 90%
- **案例记录**: 通话界面背景异常 ✅

---

### P023 自动录音未触发
- **问题类型**: 通话录音/AI 通话
- **日志类型**: logcat
- **关键特征**:
  - 通话接通后自动录音未启动
  - `AudioService.RecordingActivityMonitor: event: 2` 录音活动被检测到
  - `VoiceManager::stopAsrServices` / `destroyAsr` 录音被停止
  - AI 通话介入后返回通话界面取消自动录音
- **根因链条**: 对端未接听前进入 AI 通话 → 对端接听后返回通话界面 → 自动录音被取消
- **排查关键词**: `自动录音`, `AI 通话`, `RecordingActivityMonitor`, `stopAsrServices`
- **置信度**: 90%
- **案例记录**: 自动录音未触发 ✅

---

### P024 5G 消息注册失败
- **问题类型**: 5G 消息/RCS 注册
- **日志类型**: logcat + RIL
- **关键特征**:
  - 5G 消息注册失败或频繁掉线
  - `SETUP_DATA_CALL` 成功但 `dnses=[]` (无 DNS 下发)
  - `pcscf=[/240e:67:...]` 有 SIP 服务器 IP 但无 DNS
  - `new_status = 3` (CONFIG_FAILED)
  - `needLoginWithNetworkAndSim needLogin = false` 登录被拦截
  - 特定运营商/地区网络环境问题
- **根因链条**: 网络抖动/无 DNS 下发 → RCS 客户端无法连接服务器 → 注册失败/掉线
- **排查关键词**: `SETUP_DATA_CALL`, `dnses`, `pcscf`, `RCSAutoConfig`, `new_status = 3`
- **置信度**: 85%
- **案例记录**: 5G 消息注册失败 ✅

---

### P025 来电响铃但无提示框
- **问题类型**: 来电通知类
- **日志类型**: logcat
- **关键特征**:
  - `SET_RINGING` 正常触发
  - 用户反馈无来电提示框
  - 未抓到有效现场 log
  - 可能为偶现问题
- **根因链条**: 待更多案例验证
- **排查关键词**: `SET_RINGING`, `InCallUI`, `来电提示`
- **置信度**: 待验证
- **案例记录**: 来电响铃但无提示框

---

### P026 通话自动挂断
- **问题类型**: 通话挂断类
- **日志类型**: logcat + RIL
- **关键特征**:
  - 拨通电话后短时间内自动挂断
  - `ImsReasonInfo` 包含 `CODE_USER_TERMINATED_BY_REMOTE (510)`
  - 通话生命周期完整：DIALING → ALERTING → ACTIVE → END
  - 对端快速挂断
- **根因链条**: 对端用户主动挂断 → IMS 上报 510 原因码 → 通话终止
- **排查关键词**: `510`, `ImsReasonInfo`, `DisconnectCause`, `USER_TERMINATED`
- **置信度**: 95%
- **案例记录**: 通话自动挂断 ✅

---

### P027 短信聚合失效
- **问题类型**: 短信/数据克隆
- **日志类型**: logcat
- **关键特征**:
  - 从旧手机克隆后短信没有聚合
  - 手动关闭再打开聚合开关后恢复
  - `MmsProvider: inside checkSelection` 数据库查询触发
  - `ConversationList` UI 刷新
  - 克隆功能未自动执行聚合操作
- **根因链条**: 数据克隆后未触发聚合 → 用户手动切换开关强制刷新 → 聚合恢复
- **排查关键词**: `短信聚合`, `MmsProvider`, `checkSelection`, `克隆`
- **置信度**: 待验证
- **案例记录**: 短信聚合失效 ✅

---

### P028 通话设置页面闪烁
- **问题类型**: UI 显示/通话设置
- **日志类型**: logcat
- **关键特征**:
  - 通话设置页面开启/关闭开关时闪烁
  - `mCurrentFocus=Window{... com.android.phone/com.android.phone.CallAutoAnswerSettings}`
  - 横竖屏切换时沉浸式模式触发
  - 对比机现象一致
- **根因链条**: 横竖屏切换/沉浸式模式适配问题 → 页面布局闪烁
- **排查关键词**: `CallAutoAnswerSettings`, `横竖屏`, `沉浸式`, `闪烁`
- **置信度**: 待验证
- **案例记录**: 通话设置页面闪烁

---

### P029 通话录音状态栏消失
- **问题类型**: UI 显示/通话录音
- **日志类型**: logcat
- **关键特征**:
  - 横屏进入通话录音页面后状态栏不显示
  - 切回竖屏后状态栏仍不显示
  - 下滑状态栏后恢复
  - 横屏沉浸式模式导致状态栏隐藏
- **根因链条**: 横屏沉浸式 → 状态栏隐藏逻辑未随竖屏恢复 → 需手动触发刷新
- **排查关键词**: `横屏`, `沉浸式`, `状态栏`, `通话录音`
- **置信度**: 待验证
- **案例记录**: 通话录音状态栏消失

---

### P030 短信列表抖动
- **问题类型**: 短信/UI 性能
- **日志类型**: logcat
- **关键特征**:
  - 设置和短信页面切换时出现抖动
  - 切屏时输入法卡顿
  - `InsetsController: IME_SHOW_OR_HIDE, controlAnimationUncheckedInner ime not ready`
  - 输入法未准备好导致动画不同步
- **根因链条**: 输入法启动延迟 → 页面切换时动画不同步 → 视觉抖动
- **排查关键词**: `IME_SHOW_OR_HIDE`, `输入法`, `抖动`, `InsetsController`
- **置信度**: 85%
- **案例记录**: 短信列表抖动 ✅

---

### P031 Monkey 重启测试窗口 crash
- **问题类型**: 系统崩溃/Crash（测试场景）
- **日志类型**: logcat
- **关键特征**:
  - `Monkey` 进程正在执行自动化测试
  - `com.huawei.poweronoff_reboot_alarm` 或类似重启测试脚本
  - `shutdownTest` 关机原因
  - `ACTION_SHUTDOWN` 广播后服务批量死亡
  - `signal 15` 终止信号（来自 vold/init）
  - `hwservicemanager: Waiting on init to shut this process down`
  - `SurfaceFlinger` 死亡后出现 `DEAD_OBJECT`
  - `bad array lengths` Binder 序列化异常
- **根因链条**: Monkey 重启测试 → PowerManagerService.shutdown() → 系统发送关机广播 → 服务批量终止 → SurfaceFlinger 死亡 → 应用尝试添加窗口 → Binder 状态不一致 → crash
- **排查关键词**: `Monkey`, `reboot_alarm`, `shutdownTest`, `ACTION_SHUTDOWN`, `signal 15`, `hwservicemanager`, `DEAD_OBJECT`, `bad array lengths`
- **快速过滤策略**:
  1. 搜索 `Monkey` 确认是否有自动化测试
  2. 搜索 `shutdownTest` 或 `reboot` 确认是否重启测试
  3. 搜索 `signal 15` 确认服务终止信号来源
  4. 检查 crash 是否发生在关机广播之后
- **置信度**: 98%
- **案例记录**: 2026-04-30 Monkey 重启测试添加窗口 crash ✅
- **注意**: 这是测试场景的预期行为，非用户场景 bug

---

### P032 简易模式权限弹窗导航栏遮挡
- **问题类型**: UI 显示/窗口布局（三键导航）
- **日志类型**: logcat
- **关键特征**:
  - 权限弹窗/加密弹窗底部按钮被导航栏遮挡
  - `NavigationBar: vis:0` 短暂隐藏后恢复 `vis:3`
  - `mNavbarFlag:0` → `mNavbarFlag:3` 状态切换
  - `WindowToken type=2005` (DIALOG 类型) 在导航栏隐藏期间创建
  - 设备采用三键导航模式 (`mNavbarFlag:3`)
- **根因链条**: 用户点击返回 → IME 收起 → 导航栏短暂隐藏 (vis:0) → 权限弹窗在导航栏隐藏期间创建 → 应用测量布局时导航栏 inset=0 → 导航栏恢复显示 (vis:3, mNavbarFlag:3) → 应用未及时重新测量 → 底部按钮被遮挡
- **排查关键词**: `NavigationBar`, `mNavbarFlag`, `vis:0`, `WindowToken type=2005`, `三键导航`, `权限弹窗`, `加密弹窗`
- **快速过滤策略**:
  1. 搜索 `NavigationBar` 确认导航栏状态变化序列
  2. 搜索 `WindowToken type=2005` 确认弹窗创建时机
  3. 检查弹窗创建时 `mNavbarFlag` 是否处于切换过渡期
  4. 确认设备是否为三键导航模式 (`mNavbarFlag:3`)
- **修复方案**:
  ```java
  // 方案 1: 使用 WindowInsetsCompat 监听导航栏 inset
  ViewCompat.setOnApplyWindowInsetsListener(dialogView, (v, insets) -> {
      int bottomInset = insets.getInsets(WindowInsetsCompat.Type.navigationBars()).bottom;
      v.setPadding(0, 0, 0, bottomInset);
      return insets;
  });
  
  // 方案 2: 设置正确的 Window 属性
  dialog.getWindow().setAttributes(params -> {
      params.layoutInDisplayCutoutMode = LAYOUT_IN_DISPLAY_CUTOUT_MODE_ALWAYS;
      params.flags |= WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS;
  });
  ```
- **置信度**: 90%
- **案例记录**: 2026-05-06 简易模式编辑文档权限加密弹窗遮挡 ✅
- **影响范围**: 仅三键导航模式，手势导航无此问题

---

## 反模式（容易误判的场景）

### AP001 看到 `LOW_MEMORY` 就认为是内存问题
- **教训**: LOW_MEMORY 可能是其他问题导致内存泄露的结果，而不是原因
- **正确做法**: 配合"谁占内存"的日志一起看，如 `Bitmap`, `Allocation`, `heap`

### AP002 看到 ANR 就只查主线程
- **教训**: 可能是 Binder 对端服务卡住，导致本进程等待超时
- **正确做法**: 同时检查主线程是否等待锁、是否有 Binder 调用超时、对端服务是否存活

### AP003 看到 crash 就修 crash
- **教训**: crash 可能是其他问题的结果，不是根本原因
- **正确做法**: 追溯 crash 前的操作链路

### AP004 Widget 无响应就是 Widget 本身的问题
- **教训**: 可能是 SystemUI、Launcher 等系统服务崩溃导致
- **正确做法**: 同时检查 Widget 所属进程、SystemUI 进程、Launcher 进程

### AP005 只看 ERROR 级别日志
- **教训**: 关键信息有时在 WARN 甚至 INFO 级别
- **正确做法**: 优先看 ERROR，但不局限于 ERROR

### AP006 用户主观描述=事实
- **教训**: 用户描述"屏幕没亮"可能是"没注意到屏幕亮了"
- **正确做法**: 以日志为准，验证用户描述与日志是否一致 (案例:P004)

### AP007 删除操作=立即生效
- **教训**: 删除后数据"又出现"可能是索引延迟，不是真没删
- **正确做法**: 检查底层索引 Commit 状态，确认是否延迟刷新 (案例:P007)

---

## 校准记录（误判案例）

### ❌ 2026-04-17 来电话屏幕不亮

| 项目 | 内容 |
|------|------|
| 问题描述 | 来电话屏幕都不长亮，然后对方挂断了屏幕才亮一下 |
| 我的判断 | full_screen_intent 延迟导致屏幕唤醒慢 |
| 实际根因 | 用户手机开静音 + 无震动，来电时屏幕正常亮起但用户没注意到；拿起手机时触发 proximity 灭屏 |
| 偏差原因 | 被用户主观描述带偏，未充分分析 proximity 和静音状态 |
| 教训 | 用户主观描述可能与事实有偏差，需要以日志为准 |

---

## 反馈统计

| 指标 | 数值 |
|------|------|
| 总案例数 | 32 |
| 正模式 | 32 |
| 反模式 | 7 |
| 误判案例 | 1 |
| 准确率 | 约 90%+ |

---

## 版本历史

| 版本 | 日期 | 更新内容 |
|------|------|---------|
| v1.0 | 2026-04-27 | 初始版本，仅 P001 |
| v1.1 | 2026-04-29 | 新增反模式区、待验证模式区 |
| v2.0 | 2026-04-29 | 正模式达到 2 个 |
| v3.0 | 2026-04-29 | 合并 patterns.md + feedback.md |
| v4.0 | 2026-04-29 | 整合 base.txt 历史案例，新增 P004-P015 |
| v5.0 | 2026-04-29 | 新增 P016-P030，模式库扩充至 30 个；全面脱敏（移除品牌名/单号） |
| v6.0 | 2026-05-06 | 新增 P031（Monkey 重启测试窗口 crash） |

---

## 快速检索索引

### 按关键词
- `LOW_MEMORY` → [P001, P008, AP001]
- `510` / `ImsReasonInfo` → [P002, P004, P026]
- `SET_DISCONNECTED` → [P003, P006]
- `proximity` → [P002, P004, P026, AP006]
- `ASR` / `startAsrService` → [P005, P023]
- `merge` / `conference` → [P006]
- `HnSearchService` / `Commit` → [P007]
- `ContentProvider` / `冷启动` → [P008]
- `BlockMonitor` / `SIM` → [P009]
- `AUDIO_ROUTE` / `SPEAKER` → [P010, P018]
- `TTS` / `voiceengine` → [P011, P023]
- `CutoutMode` / `悬浮窗` → [P012]
- `AppWidgetManager` / `NullPointerException` → [P013]
- `EYEPROTECTION` / `色温` → [P014]
- `isDataInService` → [P015]
- `HeadsUp` / `InCallActivity` → [P016]
- `RCS` / `5G 消息` / `pcscf` → [P017, P024]
- `buffer overflow` / `音频库` → [P018]
- `LauncherProvider` / `角标` → [P019]
- `IME_SHOW_OR_HIDE` / `输入法` → [P020, P030]
- `毛玻璃` / `more_dialog_shape_bg` → [P022]
- `自动录音` / `RecordingActivityMonitor` → [P023]
- `dnses` / `SETUP_DATA_CALL` → [P024]
- `横屏` / `沉浸式` → [P028, P029]
- `短信聚合` / `MmsProvider` → [P027]
- `Monkey` / `shutdownTest` / `signal 15` → [P031]
- `bad array lengths` / `DEAD_OBJECT` → [P031]

### 按问题类型
- **无响应类**: P001, P013
- **通话类**: P002, P003, P004, P006, P010, P011, P016, P018, P022, P023, P026, P028, P029
- **UI 显示类**: P008, P012, P014, P020, P022, P028, P029, P030
- **性能类**: P008, P009, P030
- **搜索/索引类**: P007, P027
- **网络/数据类**: P015, P017
- **AI 功能类**: P005, P023
- **5G 消息/RCS 类**: P017, P024
- **桌面/通知类**: P019
- **联系人/相机类**: P021
- **来电通知类**: P004, P016, P025
- **测试场景类**: P031
