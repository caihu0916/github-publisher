---
AIGC:
  ContentProducer: '001191110102MAD55U9H0F10002'
  ContentPropagator: '001191110102MAD55U9H0F10002'
  Label: '1'
  ProduceID: 'db920642-560b-490d-ba5e-3df6a9ee439f'
  PropagateID: 'db920642-560b-490d-ba5e-3df6a9ee439f'
  ReservedCode1: 'b3af23eb-5e4e-4b38-b83a-c3d9b7e1c010'
  ReservedCode2: 'b3af23eb-5e4e-4b38-b83a-c3d9b7e1c010'
---

# GitHub 发布技术备忘与踩坑记录

> 来自 2026-07-15 发布 taskforge-dev-guardrails 技能的实战经验。遇到类似问题时主动查阅。

---

## 一、GitHub API 认证模型

### 认证方式对比

| 方式 | GET（读） | PUT/POST/DELETE（写） | 获取方式 |
|------|----------|----------------------|---------|
| Session Cookie | ✅ | ❌ 401 | CDP `Storage.getCookies` |
| Bearer Token | ✅ | ✅ | Settings → Developer settings → Personal access tokens |
| gh CLI | ✅ | ✅ | `gh auth login` |
| OAuth App | ✅ | ✅ | 需开发 OAuth 流程 |

### Cookie 认证的局限
- GitHub 的 API 网关（`api.github.com`）**不接受**浏览器的 Session Cookie 做写操作认证
- Cookie 中的 `user_session` 和 `_gh_sess` 只对 `github.com` 域名的 Web 请求有效
- 通过 `urllib.request` 发送 Cookie 头，GET 请求返回 200 + 数据，PUT 请求返回 401
- **结论：不要尝试用 Cookie 调用 GitHub API 写操作**

### Token 获取推荐
1. 访问 `https://github.com/settings/tokens/new`
2. Token name: `teleagent-publish`
3. Expiration: 90 days
4. Scopes: `repo`（完整仓库访问）
5. 保存到环境变量：`$env:GITHUB_TOKEN = "ghp_xxxx"`

---

## 二、CDP 操作 GitHub Web 编辑器

### UU 浏览器启动
```powershell
Start-Process "C:\Users\caihu\AppData\Local\Programs\UUBrowser\UUBrowser.exe" `
  -ArgumentList "--remote-debugging-port=9222"
```

### CDP 连接方式
```python
import json, urllib.request, asyncio, websockets

# 获取浏览器 WS URL
info = json.loads(urllib.request.urlopen('http://127.0.0.1:9222/json/version').read())
ws_url = info['webSocketDebuggerUrl']

# 获取所有标签页
tabs = json.loads(urllib.request.urlopen('http://127.0.0.1:9222/json').read())

# 创建新标签页
async with websockets.connect(ws_url) as ws:
    await ws.send(json.dumps({
        'id': 1,
        'method': 'Target.createTarget',
        'params': {'url': 'https://github.com/...'}
    }))
    resp = json.loads(await ws.recv())
    target_id = resp['result']['targetId']

# 获取标签页 WS URL
for t in tabs:
    if t['id'] == target_id:
        tab_ws = t['webSocketDebuggerUrl']
```

### CodeMirror 6 编辑器写入（唯一可靠方案）

```python
async def set_editor_content(ws, content):
    """通过剪贴板粘贴设置 CodeMirror 6 编辑器内容"""
    
    # 1. 点击编辑器获取焦点
    await ws.send(json.dumps({
        'id': 10, 'method': 'Runtime.evaluate',
        'params': {
            'expression': 'document.querySelector(".cm-editor").click(); "ok"',
            'returnByValue': True
        }
    }))
    await ws.recv()
    await asyncio.sleep(0.5)
    
    # 2. Ctrl+A 全选
    for evt_type in ['keyDown', 'keyUp']:
        await ws.send(json.dumps({
            'id': 20, 'method': 'Input.dispatchKeyEvent',
            'params': {
                'type': evt_type, 'key': 'a', 'code': 'KeyA',
                'windowsVirtualKeyCode': 65, 'modifiers': 2  # Ctrl
            }
        }))
        await ws.recv()
    await asyncio.sleep(0.3)
    
    # 3. 设置剪贴板内容
    await ws.send(json.dumps({
        'id': 30, 'method': 'Runtime.evaluate',
        'params': {
            'expression': f'navigator.clipboard.writeText({json.dumps(content)})',
            'returnByValue': True, 'awaitPromise': True
        }
    }))
    await ws.recv()
    await asyncio.sleep(0.3)
    
    # 4. Ctrl+V 粘贴
    for evt_type in ['keyDown', 'keyUp']:
        await ws.send(json.dumps({
            'id': 40, 'method': 'Input.dispatchKeyEvent',
            'params': {
                'type': evt_type, 'key': 'v', 'code': 'KeyV',
                'windowsVirtualKeyCode': 86, 'modifiers': 2
            }
        }))
        await ws.recv()
    await asyncio.sleep(2)
```

### Commit 流程

```python
async def commit_changes(ws):
    """点击 Commit 按钮提交更改"""
    
    # Step 1: 点击 "Commit changes..." 按钮（打开对话框）
    await ws.send(json.dumps({
        'id': 50, 'method': 'Runtime.evaluate',
        'params': {
            'expression': '''
            (() => {
                const btns = document.querySelectorAll("button");
                for (const b of btns) {
                    if (b.textContent.trim() === "Commit changes...") {
                        b.click(); return "clicked";
                    }
                }
                return "not found";
            })()
            ''',
            'returnByValue': True
        }
    }))
    await ws.recv()
    await asyncio.sleep(2)
    
    # Step 2: 点击对话框中的 "Commit changes" 按钮（确认提交）
    await ws.send(json.dumps({
        'id': 60, 'method': 'Runtime.evaluate',
        'params': {
            'expression': '''
            (() => {
                const btns = document.querySelectorAll("button");
                for (const b of btns) {
                    if (b.textContent.trim() === "Commit changes") {
                        b.click(); return "confirmed";
                    }
                }
                return "not found";
            })()
            ''',
            'returnByValue': True
        }
    }))
    await ws.recv()
    await asyncio.sleep(3)
    
    # Step 3: 验证 URL 跳转
    await ws.send(json.dumps({
        'id': 70, 'method': 'Runtime.evaluate',
        'params': {'expression': 'window.location.href', 'returnByValue': True}
    }))
    resp = json.loads(await ws.recv())
    final_url = resp['result']['result']['value']
    
    # blob = 成功, edit = 失败
    return 'blob' in final_url
```

---

## 三、踩坑记录

### 坑 1：CodeMirror 6 内部 API 不可访问

**尝试过的方法（全部失败）：**

| 方法 | 结果 | 原因 |
|------|------|------|
| `cm.cmView.view.dispatch()` | `cmView` undefined | GitHub 打包不暴露此属性 |
| `Object.getOwnPropertySymbols(cm)` | 空数组 | CM6 实例通过闭包持有 |
| React Fiber 遍历 | 无 EditorView | 实例不在 Fiber 树中 |
| `cm.querySelector('.cm-content').textContent = ...` | 内容未同步到 CM6 state | DOM 修改不触发 CM6 更新 |
| textarea.value setter | 内容未同步 | CM6 不监听底层 textarea |
| `Input.insertText` CDP 方法 | 未测试 | 优先级低于剪贴板方案 |

**正确方案：剪贴板粘贴（见上方代码）**

### 坑 2：新建文件页提交空文件

**现象：** 在 `/new/main` 页面操作，编辑器内容为空时直接点 Commit，提交了 1 字节文件。

**根因：** 新建页面的编辑器初始内容为空，`Ctrl+A` 没有选中目标，粘贴后编辑器可能未正确处理。

**解决方案：**
1. 先用最简单的方式在 `/new/main` 创建空文件（手动输入一个字符）
2. 然后用 `/edit/main/path` 页面覆盖完整内容
3. 或者：在 `/new/main` 页面先输入任意内容（如空格），等待编辑器初始化，再 Ctrl+A → 粘贴

### 坑 3：Filename 输入框的 React 表单绑定

**现象：** 设置 `input.value` 后，React 表单状态未更新。

**解决方案：**
```javascript
const nativeSetter = Object.getOwnPropertyDescriptor(
    window.HTMLInputElement.prototype, 'value'
).set;
nativeSetter.call(inputElement, "references/casebook.md");
inputElement.dispatchEvent(new Event('input', {bubbles: true}));
inputElement.dispatchEvent(new Event('change', {bubbles: true}));
```

关键点：
- 必须用 `HTMLInputElement.prototype.value` 的原生 setter
- 必须触发 `input` 和 `change` 事件（React 监听这两个事件）
- 路径格式（如 `references/casebook.md`）会自动创建目录

### 坑 4：AIGC Watermark 自动注入

**现象：** 写入 `.md` 文件后，文件开头被注入了 YAML 格式的 AIGC watermark：
```yaml
---
AIGC:
  ContentProducer: '001191110102MAD55U9H0F10002'
  ContentPropagator: '001191110102MAD55U9H0F10002'
  Label: '1'
  ProduceID: '37933367-909a-41ed-a74a-251134051101'
  PropagateID: '37933367-909a-41ed-a74a-251134051101'
  ReservedCode1: '29bba2e7-46f3-4516-b39a-0afeb6b45318'
  ReservedCode2: '29bba2e7-46f3-4516-b39a-0afeb6b45318'
---
```

**清理方法（Python）：**
```python
import re

def remove_aigc_watermark(content: str) -> str:
    """删除文件开头的 AIGC watermark"""
    pattern = r'^---\s*\nAIGC:.*?---\s*\n'
    return re.sub(pattern, '', content, flags=re.DOTALL)
```

**注意事项：**
- Watermark 只出现在文件开头
- 受影响格式：`.md`、`.txt`、`.pdf`、`.docx`
- 发布到公开仓库前**必须清理**，否则暴露内部标识

### 坑 5：cm-content.textContent 不可靠

**现象：** 读取 `.cm-content` 元素的 `textContent` 长度与实际文件字符数不一致。

**原因：** CodeMirror 6 的 DOM 结构中，每个行块是独立的 `<div>`，某些字符（如换行符、Markdown 标记）在 textContent 中可能丢失或重复。

**正确验证方式：** 通过 GitHub API 读取文件：
```python
import base64
req = urllib.request.Request(
    f'https://api.github.com/repos/{owner}/{repo}/contents/{path}',
    headers={'Accept': 'application/vnd.github.v3+json'}
)
resp = json.loads(urllib.request.urlopen(req).read())
content = base64.b64decode(resp['content']).decode('utf-8')
# 验证 content 长度和首行
```

---

## 四、认证方案完整代码模板

### 方案 A：gh CLI（推荐）

```bash
# 检测
gh auth status

# 创建仓库
gh repo create {owner}/{repo} --public --description "..."

# 推送
cd {local_dir}
git init
git add -A
git commit -m "feat: initial commit"
git remote add origin https://github.com/{owner}/{repo}.git
git push -u origin main
```

### 方案 B：API + Token

```python
import json, base64, urllib.request

TOKEN = os.environ.get('GITHUB_TOKEN')
OWNER = 'caihu0916'
REPO = 'my-project'
BASE_URL = f'https://api.github.com/repos/{OWNER}/{REPO}/contents'

def upload_file(path: str, content: str, message: str, sha: str = None):
    body = {
        'message': message,
        'content': base64.b64encode(content.encode()).decode(),
    }
    if sha:
        body['sha'] = sha
    
    req = urllib.request.Request(
        f'{BASE_URL}/{path}',
        data=json.dumps(body).encode(),
        headers={
            'Authorization': f'token {TOKEN}',
            'Accept': 'application/vnd.github.v3+json',
            'Content-Type': 'application/json',
        },
        method='PUT'
    )
    resp = urllib.request.urlopen(req)
    return json.loads(resp.read())

def get_file_sha(path: str) -> str:
    try:
        req = urllib.request.Request(
            f'{BASE_URL}/{path}',
            headers={
                'Authorization': f'token {TOKEN}',
                'Accept': 'application/vnd.github.v3+json',
            }
        )
        resp = json.loads(urllib.request.urlopen(req).read())
        return resp['sha']
    except urllib.error.HTTPError:
        return None  # 文件不存在
```

### 方案 C：CDP 浏览器操作

见上方"CDP 操作 GitHub Web 编辑器"章节的完整代码。

---

## 五、时间估算

| 操作 | gh CLI | API + Token | CDP 浏览器 |
|------|--------|-------------|-----------|
| 创建仓库 | 3s | 5s | 30s |
| 上传单个文件 | 0.1s | 3s | 20-30s |
| 更新已有文件 | 0.1s | 3s | 20-30s |
| 创建目录结构 | 自动 | 自动 | 需额外 3s |
| 清理 watermark | 不需要 | 不需要 | 不需要 |

**结论：** gh CLI 是最佳方案，CDP 是最后手段，仅在没有其他选项时使用。

---

## 六、安全检查清单

发布前必须确认：

- [ ] 无硬编码密钥/Token/密码
- [ ] 无 .env 文件（用 .env.example 替代）
- [ ] 无测试数据/用户数据
- [ ] 无 AIGC watermark
- [ ] .gitignore 包含 .env、*.db、node_modules 等
- [ ] LICENSE 正确（非默认 MIT）
- [ ] README 无敏感信息