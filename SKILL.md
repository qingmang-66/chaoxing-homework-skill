# 浏览器自动化：网页作业/表单填写

## ⚠️ 全局最高优先规则（不可覆盖）

**默认情况下只保存填写的内容，在用户没有明确同意提交的情况下，严格不能提交。**

- ✅ 可以：填写答案、点击「暂时保存」
- ❌ 禁止：点击「提交」按钮（除非用户明确说"帮我提交"）
- 这是最高优先规则，任何情况下都不能违反

---

## 记忆与执行模式

### 课程列表记忆

批量遍历课程时，**只在第一次询问用户**，后续复用记忆：

```
第一次：
  提取课程列表 → 展示给用户询问 → 记住用户选择

后续（同一会话）：
  直接使用记忆中的课程列表，不再询问

后续（新会话）：
  重新提取课程列表 → 展示给用户询问 → 记住用户选择
```

**记忆格式**（写入 MEMORY.md）：
```
## chaoxing-homework 课程列表记忆
- 用户选择的课程序号：1,2,3,5,6,9,10
- 课程列表（含 URL）：
  1. 数字图像处理 | https://mooc1.chaoxing.com/...
  2. 物联网工程导论 | https://mooc1.chaoxing.com/...
  ...
- 更新时间：2026-05-07
- 有效期：120天（一个学期）
```

### 执行模式

| 用户意图 | 执行模式 |
|----------|----------|
| "帮我看看哪些课有作业" | 批量遍历 → 询问课程 → 检查作业 |
| "检查所有课程作业" | 批量遍历 → 询问课程 → 检查作业 |
| "帮我做应用密码学的作业" | 单课程 → 直接打开 → 填写 |
| "帮我做第15-16讲的作业" | 单作业 → 直接打开 → 填写 |
| 用户已打开作业页面 | 直接填写，不遍历 |

### 课程匹配策略（精确 → 模糊 → 全量）

当用户指定了课程名称时，按以下优先级匹配：

```
1. 精确匹配：课程名完全一致 → 直接使用
2. 模糊匹配：课程名包含关键词 → 返回匹配列表让用户确认
3. 无匹配：展示全部课程列表让用户选择
```

**示例流程**：

用户说："帮我做密码学的作业"

```
Step 1: 模糊匹配 → 找到「应用密码学」
Step 2: 返回「找到以下匹配课程：1. 应用密码学」
Step 3: 用户确认 → 直接打开
```

用户说："帮我做密码学原理的作业"

```
Step 1: 模糊匹配 → 无结果
Step 2: 展示全部课程列表：
        1. 数字图像处理
        2. 物联网工程导论
        ...
        15. 程序设计基础
        没有找到「密码学原理」，请选择：
Step 3: 用户选择 → 继续
```

---

## 适用场景

当用户需要在浏览器中填写在线作业、问卷、表单时触发。典型特征：
- 题目包含单选题、多选题、判断题、填空题、简答题等
- 页面有"保存"、"提交"按钮
- 文本输入区域可能是 `<input>`、`<textarea>`，也可能是 `<iframe>` 内嵌富文本编辑器
- 支持超星学习通、智慧树、雨课堂、问卷星等各类在线平台
- 支持"帮我看看哪些课有作业"、"检查所有课程作业"等批量遍历场景

**浏览器支持**：Edge、Chrome、Firefox 等所有支持 CDP 的浏览器。

> 💡 **名称说明**：虽然叫 chaoxing-homework，但技能本身不限于超星学习通，也不限于作业场景。核心是通过 CDP 控制浏览器完成任何网页表单的自动化操作。

---

## 前置：连接浏览器

### 浏览器选择（询问 → 记忆）

**第一次使用时**，如果用户没有指定浏览器，询问：

```
需要使用哪个浏览器？
1. Edge（推荐，Windows 自带）
2. Chrome
3. Firefox
请输入序号：
```

用户选择后，将偏好写入 MEMORY.md：

```
## chaoxing-homework 浏览器偏好
- 浏览器：Edge
- CDP 端口：9222
- 用户数据目录：C:\Users\钟健祺\AppData\Local\Microsoft\Edge\User Data
- 更新时间：2026-05-07
```

**后续使用**：直接使用记忆中的浏览器，不再询问。除非用户主动说"换用 Chrome"等。

### 启动浏览器

**Edge（推荐）**：
```bash
msedge.exe --remote-debugging-port=9222 --user-data-dir="C:\Users\<用户名>\AppData\Local\Microsoft\Edge\User Data"
```

**Chrome**：
```bash
chrome.exe --remote-debugging-port=9222
```

**Firefox**：
需要在 `about:config` 中开启 `devtools.debugger.remote-enabled`，然后：
```bash
firefox.exe --remote-debugging-port=9222
```

> ⚠️ 使用 `--user-data-dir` 加载你的配置文件，这样能保持登录状态。

### 连接

```
browser_use action=connect_cdp url=http://localhost:9222
```

连接后用 `tabs list` 确认当前标签页状态。

---

## 场景一：单页面作业/表单填写

适用于用户已经打开某个作业页面，需要你帮忙填写。

### 步骤 1：分析页面结构

先 snapshot 了解页面整体布局，然后用 JavaScript 探测输入控件：

```javascript
// 统计各类输入控件
var r = '';
r += 'iframes: ' + document.querySelectorAll('iframe').length + '\n';
r += 'inputs: ' + document.querySelectorAll('input').length + '\n';
r += 'textareas: ' + document.querySelectorAll('textarea').length + '\n';
r += 'contenteditable: ' + document.querySelectorAll('[contenteditable]').length + '\n';
r += 'radios: ' + document.querySelectorAll('input[type=radio]').length + '\n';
r += 'checkboxes: ' + document.querySelectorAll('input[type=checkbox]').length + '\n';
r += 'selects: ' + document.querySelectorAll('select').length;
return r;
```

### 步骤 2：确认控件-题目映射

**对 iframe 富文本类型**，执行 DOM 分析确认每个 iframe 对应哪道题：

```javascript
// 获取每个 iframe 对应的题目文本
var iframes = document.querySelectorAll('iframe');
return Array.from(iframes).map(function(f, i) {
  // 尝试多种选择器找到题目容器
  var container = f.closest('.stem_answer') || f.closest('.question') ||
                  f.closest('.form-item') || f.closest('[class*="answer"]') ||
                  f.closest('[class*="field"]') || f.parentElement;
  var text = container?.parentElement?.innerText?.substring(0, 100) || 'unknown';
  return i + ': ' + text;
}).join('\n');
```

> ⚠️ **不要用截图 OCR！** DOM 分析准确率 100%，OCR 无法判断 iframe 和题目的对应关系。

**对普通 input 类型**，snapshot 直接获取 ref 和标签文本即可。

### 步骤 3：填写答案

#### 3a. 选择题/判断题

snapshot 获取 radio/checkbox 的 ref，然后 click。

#### 3b. 下拉选择

```
browser_use action=select_option ref=<select_ref> values_json='["选项值"]'
```

#### 3c. 普通文本框

```
browser_use action=click ref=<input_ref>
browser_use action=type ref=<input_ref> text="答案"
```

#### 3d. iframe 富文本编辑器（最通用）

```javascript
// 填写第 N 个 iframe
var iframe = document.querySelectorAll('iframe')[N];
iframe.contentDocument.body.focus();
iframe.contentDocument.execCommand('selectAll');
iframe.contentDocument.execCommand('delete');
iframe.contentDocument.execCommand('insertText', false, '答案内容');
var p = iframe.contentDocument.querySelector('p');
if (p) { p.dispatchEvent(new Event('input', {bubbles: true})); }
```

```javascript
// 批量填写所有 iframe
var iframes = document.querySelectorAll('iframe');
var data = ['答案1', '答案2', '答案3'];
data.forEach(function(text, i) {
  var iframe = iframes[i];
  if (!iframe?.contentDocument) return;
  iframe.contentDocument.body.focus();
  iframe.contentCommand('selectAll');
  iframe.contentDocument.execCommand('delete');
  iframe.contentDocument.execCommand('insertText', false, text);
  var p = iframe.contentDocument.querySelector('p');
  if (p) { p.dispatchEvent(new Event('input', {bubbles: true})); }
});
```

**关键顺序**：focus → selectAll → delete → insertText → dispatchEvent(input)

#### 3e. contenteditable 元素（非 iframe）

```javascript
var editor = document.querySelector('[contenteditable="true"]');
editor.focus();
document.execCommand('selectAll');
document.execCommand('delete');
document.execCommand('insertText', false, '答案内容');
editor.dispatchEvent(new Event('input', {bubbles: true}));
```

### 步骤 4：验证

```javascript
// 读取所有 iframe 内容确认
var iframes = document.querySelectorAll('iframe');
return Array.from(iframes).map(function(f, i) {
  return i + ': ' + (f.contentDocument?.body?.innerText?.substring(0, 80) || 'empty');
}).join('\n');
```

同时用 snapshot 验证选择题/判断题的选中状态。

### 步骤 5：保存/提交

snapshot 找到"保存"或"提交"按钮的 ref，然后 click。

---

## 场景二：批量遍历课程/页面检查作业

适用于用户说"帮我看看哪些课有作业"、"检查所有课程作业"等场景。

### 通用遍历流程

```
┌──────────────────────────────────────────────────────────────┐
│  1. 打开入口页面（如个人空间、课程列表页）                      │
│  2. 提取所有课程链接                                          │
│  3. 展示课程列表（带序号），询问用户哪些课程需要检查           │
│     ⚠️ 学习通会显示所有学期的课程，已结课的课程无需遍历        │
│  4. 对用户选中的课程：                                        │
│     a. 打开课程页面                                           │
│     b. 找到并点击「作业」或类似入口                            │
│     c. 读取作业列表，记录未完成项                              │
│  5. 汇总结果并汇报                                            │
└──────────────────────────────────────────────────────────────┘
```

### 步骤 1：打开入口页面

根据用户当前浏览器状态，可能需要：
- 直接导航到课程列表页
- 或在当前页面点击菜单进入课程列表

### 步骤 2：提取课程列表

用 snapshot 提取所有课程链接（含课程名和 URL）。

对于 iframe 内的链接（如超星个人空间），用 frame_selector：

```
browser_use action=snapshot frame_selector=iframe[name='frame_content']
```

### 步骤 3：课程匹配与询问

**⚠️ 不要直接遍历所有课程！** 学习通会显示所有学期的课程，很多已结课的课程根本没有作业，遍历会浪费大量资源。

**执行逻辑（按优先级）**：

```
1. 用户指定了课程名称？
   ├─ 是 → 模糊匹配课程列表
   │       ├─ 精确/模糊匹配到 → 返回匹配结果让用户确认 → 直接打开
   │       └─ 未匹配到 → 展示全部课程列表让用户选择
   └─ 否 → 检查 MEMORY.md 记忆
           ├─ 有记忆且未过期（120天内）→ 直接使用，不再询问
           └─ 无记忆或已过期 → 展示全部课程列表让用户选择
```

**模糊匹配示例**：

用户说："帮我做密码学的作业"
```
找到以下匹配课程：
1. 应用密码学（2025-2026第二学期）
确认是这门课吗？
```

用户说："帮我做密码学原理的作业"
```
没有找到「密码学原理」，当前课程列表：
1. 数字图像处理（2025-2026第二学期）
2. 物联网工程导论（2025-2026第二学期）
3. 计算机网络（2025-2026第二学期）
4. 编译原理
5. 应用密码学（2025-2026第二学期）
...
15. 程序设计基础

请选择要检查的课程（输入序号，如：1,2,5），或输入「全部」。
```

用户说："帮我看看哪些课有作业"（无记忆）
```
找到以下课程：
1. 数字图像处理（2025-2026第二学期）
2. 物联网工程导论（2025-2026第二学期）
...
15. 程序设计基础

哪些课程需要检查作业？
请输入序号（如：1,2,3,5），或输入「全部」。
```

**用户回复后**：
1. 只遍历用户选中的课程
2. 将用户选择写入 MEMORY.md（格式见「记忆与执行模式」），有效期 120 天

### 步骤 4：逐课程检查

对每个用户选中的课程：

1. **打开页面**：`browser_use action=open url=<course_url>`
2. **找到作业入口**：snapshot 找"作业"按钮，或用 eval 点击
3. **读取作业列表**：snapshot 作业列表区域
4. **记录结果**：有未完成作业的记下来

### 步骤 5：汇报

```
有未完成作业的课程：
1. 课程名 — 未完成 X/Y
   - 作业名1（状态：未交/待批阅）
   - 作业名2（状态）
2. 课程名 — 未完成 X/Y
   ...

无作业的课程：课程1、课程2、...
```

---

## 适配不同平台

### 超星学习通（chaoxing.com）

**页面层级**：
```
i.chaoxing.com/base (个人空间)
  └─ iframe[name=frame_content] → mooc1.chaoxing.com/visit/interaction (课程列表)
       └─ 点击课程 → mooc2-ans.chaoxing.com/mycourse/stu (课程主页)
            └─ 点击「作业」→ iframe[name=frame_content-zy] → mooc1.chaoxing.com/mooc2/work/list (作业列表)
                 └─ 点击作业 → mooc1.chaoxing.com/mooc2/work/dowork (答题页面)
```

**关键特征**：
- 课程列表在 `frame_content` iframe 内
- 作业列表在 `frame_content-zy` iframe 内（跨域，必须用 frame_selector）
- 答题页面的填空题/简答题用 UEditor iframe（`edui-editor-iframeholder`）
- 题目-iframe 映射：`iframe.closest('.stem_answer').parentElement.innerText`
- 保存按钮：通常是页面顶部第一个 ref（e1）的 link 元素
- 验证码：答题页面可能有 `inputCode` 输入框

### 智慧树（zhihuishu.com）

- 题目容器：`.answer-region`
- 编辑器 class：`rich-text-editor`
- 保存按钮：文本为"保存"或"提交"的 button

### 雨课堂（yuketang.cn）

- 主要用 `<textarea>`，极少 iframe
- 直接用 snapshot ref 填写即可

### 问卷星（wjx.cn）

- 普通 `<input>` 和 `<textarea>` 为主
- 极少使用 iframe

### 通用探测策略

如果不确定平台，先用 JavaScript 探测：

```javascript
var r = '';
if (document.querySelector('[class*="edui"]')) r += 'UEditor\n';
if (document.querySelector('[class*="tinymce"]')) r += 'TinyMCE\n';
if (document.querySelector('[class*="ck-editor"]')) r += 'CKEditor\n';
if (document.querySelector('[class*="quill"]')) r += 'Quill\n';
if (document.querySelector('[class*="stem_answer"]')) r += '超星学习通\n';
if (document.querySelector('[class*="answer-region"]')) r += '智慧树\n';
if (!r) r = '未知，使用通用方式';
return r;
```

---

## 常见错误及解决

| 错误 | 原因 | 解决 |
|------|------|------|
| iframe 内容全部错位 | 没有确认 iframe-题目映射关系 | 用 DOM 分析：`closest('.stem_answer').parentElement.innerText` |
| 内容写入但不被识别 | 没有触发 input 事件 | 写入后 `dispatchEvent(new Event('input'))` |
| 内容重复写入 | 没有先清除旧内容 | 先 `selectAll()` + `delete()` |
| 点击 ref 失败 | 页面重新加载后 ref 变化 | 重新获取 snapshot |
| iframe contentDocument 为 null | 跨域或 iframe 未加载 | 等待页面加载完成；用 frame_selector 代替直接访问 |
| 填空题每题多空 | 一个题目有多个 iframe | 按顺序一一对应 |
| 保存后内容丢失 | 页面重新加载 | 重新连接并验证 |
| eval 语法错误 | 不支持 var/const/分号 | 用 IIFE：`(function(){ ... })()` |
| 遍历课程时链接提取失败 | 链接在 iframe 内 | 用 `frame_selector=iframe[name='frame_content']` 访问 |
| 课程页面无权限 | 课程已结束 | 跳过该课程 |

## 技术细节

### frame_selector 跨域访问

Playwright 的 `frame_selector` 可以访问跨域 iframe（浏览器原生支持），但 `eval` 在主 frame 执行时无法跨域访问 iframe 内部。

**正确姿势**：
- 读取 iframe 内容 → 用 `browser_use action=snapshot frame_selector=iframe[name='xxx']`
- 操作 iframe 内部元素 → 用 frame_selector 获取 ref，然后 click/type
- 操作 iframe 内部 DOM → 用 `eval` 在**主 frame** 执行 `document.querySelector('iframe').contentDocument...`（仅限同域）

### 为什么不推荐截图 OCR？

截图 OCR 无法准确判断 iframe 和题目的对应关系，容易搞混。DOM 结构分析可以直接获取每个 iframe 对应的题目文本，准确率 100%。仅在 DOM 分析失败时才考虑截图作为辅助。

### CDP 连接注意事项

- **Edge/Chrome**：`--remote-debugging-port=9222`
- **Firefox**：`about:config` 开启 `devtools.debugger.remote-enabled`
- **安全**：CDP 连接后可以完全控制浏览器，注意隐私
- **多标签页**：用 `browser_use action=tabs` 查看，确认目标 page_id

---

## 超星学习通（chaoxing.com）— 已验证流程

### 页面层级

```
i.chaoxing.com/base (个人空间)
  └─ iframe[name=frame_content] → mooc1.chaoxing.com/visit/interaction (课程列表)
       └─ 点击课程 → mooc2-ans.chaoxing.com/mycourse/stu (课程主页)
            └─ 点击「作业」→ iframe[name=frame_content-zy] → mooc1.chaoxing.com/mooc2/work/list (作业列表)
                 └─ 点击作业 → 新标签页打开 → mooc1.chaoxing.com/mooc2/work/dowork (答题页面)
```

### 关键特征

- **课程列表**在 `frame_content` iframe 内
- **作业列表**在 `frame_content-zy` iframe 内（跨域，必须用 frame_selector）
- **点击作业链接后，导航在新标签页中打开**（这是关键！）
- 答题页面的填空题/简答题用 UEditor iframe（`edui-editor-iframeholder`）
- 题目-iframe 映射：`iframe.closest('.stem_answer').parentElement.innerText`
- 「暂时保存」按钮：页面顶部第一个 ref（e1）的 link 元素

### 已验证的操作

| 操作 | 方式 | 结果 |
|------|------|------|
| 读取课程列表 | `frame_selector=iframe[name='frame_content']` + snapshot | ✅ |
| 读取作业列表 | `frame_selector=iframe[name='frame_content-zy']` + snapshot | ✅ |
| 点击作业链接 | `frame_selector + ref + click` | ✅ 新标签页打开 |
| 验证导航成功 | `tabs list` 检查所有标签页 | ✅ 找 URL 含 /work/dowork |
| 直接修改 iframe src | 修改 location.href | ❌ 404/无权限 |

### ⚠️ 关键经验

1. **`frame_selector + ref + click` 可以成功触发导航**，但导航在**新标签页**中打开
2. **原标签页的 iframe src 不会变**，不要用 eval 检查 iframe src 判断是否成功
3. **验证方式**：`tabs list` 检查所有标签页，找 URL 含 `/work/dowork` 的新标签页
4. **snapshot vs eval 差异**：snapshot 能跨域读取完整渲染结果，eval 只能访问外层 iframe 的同域 DOM

### 完整已验证流程

```
1. 连接 CDP → tabs list 检查当前标签页
2. 检查当前 URL，判断在哪个阶段：
   - 含 /base → 个人空间，点「课程」
   - 含 /mycourse/stu → 课程页面，点「作业」
   - 含 /work/list → 作业列表页，读取未完成作业
   - 含 /work/dowork → 作业详情页，直接填写
3. 点未完成作业链接（frame_selector + ref + click）
4. 等待 5-10 秒
5. tabs list 检查所有标签页，找到 URL 含 /work/dowork 的新标签页
6. 切换到新标签页，snapshot 确认作业详情页已加载
7. 填写作业内容
8. ⚠️ 默认点「暂时保存」，用户明确同意后才点「提交」
```

### 优化细节

- **用 wait_for 替代固定等待**：等待「暂时保存」按钮出现
- **操作前检查 URL**：避免重复操作
- **标签页管理**：关掉多余的标签页，只保留需要的
- **Cookie 过期**：URL 跳转到 passport2.chaoxing.com/login 时，提示用户手动登录
