# WeChat 公众号来源发现技术

## 场景
用户希望扩展 daily-news 新闻来源，但不确定具体有哪些公众号可用。

## 前置条件
- macOS 上 WeChat 桌面版已登录
- 用户WeChat的「订阅号消息」文件夹中有已关注的公众号

## 操作步骤

### 1. 确保WeChat在前台
```bash
osascript -e '
tell application "System Events"
    tell process "WeChat"
        set frontmost to true
    end tell
end tell
'
```

### 2. 截取屏幕
```bash
screencapture -x /tmp/wechat_screen.png
```

### 3. 用vision_analyze读取微信窗口
调用 vision_analyze(image_url="/tmp/wechat_screen.png")，提问：
"仔细看微信窗口。左侧聊天列表中，除了'文件传输助手'和'订阅号消息'，还有哪些对话？请列出全部可见的公众号名称。"

### 4. 如果列表被截断，滚动再截图
```bash
osascript -e '
tell application "System Events"
    tell process "WeChat"
        set frontmost to true
    end tell
end tell
delay 0.3
tell application "System Events"
    key code 121 -- Page Down
end tell
'
```
然后重复步骤2-3。

### 5. 整理清单
将识别到的公众号名称整理为列表，与用户确认。

## 注意事项
- WeChat 本地数据库（SQLite）是加密的，不可直接读取，必须通过屏幕截图+视觉识别
- 注意排查语音识别错误：屏幕识别可能的文字误读（如"易引床"→"易胤超"）
- 已经存在的联系人名片（刘杰灵 vs 刘杰玲）可交叉验证
- 搜狗微信搜索（https://weixin.sogou.com/）是获取公众号文章聚合的官方途径
- 但 Cron 定时任务运行时不保证 WeChat 在运行，适用于一次性发现阶段
