---
name: 网易《我的世界》PE 资源上架（CDP 自动化）
description: >-
  【网易《我的世界》PE 资源上架（CDP 自动化）】>- 【网易《我的世界》PE 资源上架（CDP 自动化）】>- 在网易《我的世界》开发者平台（mcdev.webapp.163.com / mc-launcher.webapp.163.com） 上架 PE 资源（附加包 / 地图 / 材质光影 / 皮肤）的全流程自动化。 用 CDP 直驱浏览器 + 直接 HTTP 打平台接口完成：填表、传图、保存、提交审核。 当需要「上架我的世界组件」「PE 资源提审」「mcdev 表单自动化」「附加包发布」 「把 XX 模组提交审核」，或遇到「点保存没反应」「服务器异常」「生成PC模组数据失败」 「请按照提供的尺寸上传所有图片」「已有重名模组」这类报错时使用。
agent_created: true
---

# 网易《我的世界》PE 资源上架（CDP 自动化）

## 核心原则

1. **不要用鼠标硬点网页**。用 CDP（Chrome DevTools Protocol）执行 JS + 真实点击 + 塞文件，
   不受窗口前台/焦点/DPI 影响，还能读 Vue 组件内部状态。
2. **保存接口用 Cookie 直接 HTTP 打**，比在页面里点按钮可控得多，而且能拿到服务端结构化报错。
3. **判断成败只看后台列表的「提审时间 / 状态」**，不要相信页面提示。

## 一、准备：拿到可用的浏览器 + 登录态

| 浏览器 | 说明 |
|---|---|
| QQ浏览器 | 登录态通常在这里。但**带 `--remote-debugging-port` 启动不稳定**（会被自己关掉），无头模式更短命 |
| Edge / Chrome | 稳定，`--remote-debugging-port=9222` 可用，但往往**没有网易平台的登录态** |

→ **把 QQ浏览器 的 Cookie 搬到 Edge**：
1. 读 `<QQ浏览器 profile>/Local State` → `os_crypt.encrypted_key` → base64 解码后去掉前 5 字节 `DPAPI`
   → `CryptUnprotectData` → AES 主密钥。
2. 读 `Default/Network/Cookies`（SQLite，用 `file:...?immutable=1` 打开）。
3. `v10` 解密：`nonce=blob[3:15]`，`tag=密文末 16 字节`，AES-GCM。
   ⚠️ **明文前 32 字节是 `sha256(host)[:32]` 域名哈希，必须剥掉**，否则 CDP 报
   `Sanitizing cookie failed`。
4. 用 `Network.setCookie` 注入 Edge（`domain=host`、`path=/`、`url=https://mcdev.webapp.163.com/`）。

⚠️ **CDP 的 HTTP 探测要绕开系统代理**：
`urllib.request.build_opener(urllib.request.ProxyHandler({}))`，否则连 127.0.0.1 都 ConnectionRefused。

## 二、表单结构（Element-UI + Vue2）

- 7 个 `.el-form` 各有自己的 `model`，**都是副本**。
- 主控组件：`PEEditForm`（从 `#app` 的 `__vue__` 往下遍历，找 `$options.name==='PEEditForm'`）。
- `$refs` = `baseInfoForm / priceForm / detailForm / resourceForm / syncInfoForm / syncDetailForm / syncResourceForm`。
- **保存链路**：按钮 → 逐个 `validForm()` → **任一 false 则 `getFormData()` 返回 `{}`（静默失败，不发请求）**。
  → 「点保存没反应」先查谁 `validForm()` 返回 false。

下拉/级联用 JS 点不动时：**直接找 `.el-select` / `.el-cascader` 的 `__vue__` 改内部值**。
级联标签的值是**扁平叶子值数组**（如 `[101,202]`），不是路径数组 `[[1,101],[2,202]]`。

富文本（Quill）：`.ql-editor` 先 `focus()`，再用 CDP `Input.insertText` 输入。

## 三、⭐ 图片上传（最容易卡死的一步）

现象：`DOM.setFileInputFiles` 塞进去后 `el-upload.fileList[0].status === 'ready'`，永不 success。

原因链：
1. 这份代码 `autoUpload=false` → 只入列不上传；
2. `upload(file)` 会先进 `beforeUpload` → 打开**图片裁剪框**（`cropper:{enabled:true}`），
   返回一个不点确认就永不 resolve 的 Promise；裁剪框还挂在 `body` 上、不在组件树里，关不掉。

**正解：跳过 beforeUpload，直接 `post(rawFile)`**
```js
const u = [...document.querySelectorAll('.el-upload')][槽位序号];
const w = u.__vue__;                        // 包装组件：handleChange/upload/post/uploadFiles
const f = w.fileList[0].raw;                // 真 File
if (!f.uid) f.uid = Date.now();
w.data.Authorization || (w.data.Authorization = window.__tok);
w.post(f);                                  // → status 变 success
```
**上传需要 `Authorization: Policy <token>`**，图片槽默认是空的（视频槽有值，所以视频一次就过）。
token 现取：往上找 `singleupload` → `await su.getToken()` → 会写进 `w.data.Authorization`，取一次可复用整批。

槽位固定尺寸见 `references/slot-sizes.md`，出图脚本见本 skill 的 `scripts/gen_pe_imgs.py`。

## 四、⭐ 保存接口与 payload

```
POST https://mc-launcher.webapp.163.com/items/categories/pe/upload
headers: Content-Type: application/json;charset=UTF-8
         Origin: https://mcdev.webapp.163.com
         Referer: https://mcdev.webapp.163.com/
body = dealFormData(await getFormData()) + {is_check_apply: false}   （Cookie 要带上）
```

### 报错 → 真实原因
| 返回 | 原因 | 修法 |
|---|---|---|
| `服务器异常` | payload 字段**结构**不对，服务端抛异常 | 逐字段移除二分法定位 |
| `params error` + `errors{...}` | 正常字段校验错 | 按字段名改 |
| `生成PC模组数据失败：{'_schema': ['次类别错误']}` | `sync_item_info` 里用了手机版类别枚举 | 换成电脑版枚举 |
| `已有重名模组` | 同名资源已存在（说明之前某次其实保存成功了） | 去列表看，别重复提交 |
| `未登录` | 请求没带 Cookie | 带上凭据 |

### 关键字段正确值
```
item_name / is_original=true / main_city=false / sync_pc_flag=true
price_type='diamond'  price=300  price_rank=0
detailForm.info / syncDetailForm.info = '<p>文案</p>'
syncInfoForm: brief(≤100字) include_map tag=[88] requirement=[]
syncResourceForm: available_scope='client/server'  label_type_list=[101,202]  ← 扁平叶子值
mc_version=[]  online_platform=[]  multi_tags='0'（字符串）
res=[{ res_url:<保持原始对象形态 {body,file_type,sign}>,
       res_id:1, res_name:<zip名>,
       res_info:{res_md5,res_size,res_time}, cdn_info:{...}, cdn_url:<url>,
       mc_version:[], add_version:true }]
tags  ← ⚠️ 不要传（传字符串数组会让服务端崩）
```

### ⭐ 手机版 / 电脑版是两套枚举
- 手机版：`add_ons=2` / `生物改造=8` / `更多生物=203`
- 电脑版：`功能组件=3 视觉组件=4 地图组件=5 形象组件=10 礼包=12`；
  子类 `工具类=15 冒险类=16 科技类=17 前置类=34 农业类=35 装饰类=36 魔法类=37 其他=38`
- payload **顶层**放手机版值；**`sync_item_info`（电脑版那包）里必须是电脑版值**，否则报「次类别错误」。

## 五、提交审核

```
上架与资源管理 →【必须切到「手机版」页签！】→ 行内【提交审核】→ 弹窗 →【直接提审】
→ 状态变「系统处理中（双端同步）」+ 出现提审时间
```
- **电脑版页签的行没有「提交审核」按钮**（只有 查看 / 前往PE同步编辑）。
- 点击前先 `Page.bringToFront`；注意 `el-loading-mask` 会盖住按钮吞掉点击，必要时先移除；
  首击常被吃，**连点两次**。

## 六、脚本（本 skill 的 scripts/）
| 脚本 | 用途 |
|---|---|
| `cdp.py` | CDP 客户端：attach / eval / click_xy / insert_text / set_upload_file |
| `cookie_transfer.py` | QQ浏览器 Cookie 解密 → 注入 Edge |
| `gen_pe_imgs.py` | 按各槽固定尺寸批量出图 |
| `pe_upload_all.py` | 批量上传全部图片槽（取 token + post） |
| `pe_bisect.py` | payload 逐字段二分，定位「服务器异常」 |
| `pe_try_payload.py` | payload 变体快速试错 |

## 七、坑速查
1. CDP 探测要 `ProxyHandler({})` 绕开系统代理。
2. v10 Cookie 明文前缀 32 字节要剥掉。
3. 塞文件 ≠ 上传，必须 `post(raw)`。
4. 上传要 `Authorization: Policy <token>`（`getToken()` 取）。
5. 裁剪框是不 resolve 的 Promise，别指望点掉它。
6. `tags` 会崩服务端；`res.res_url` 要保留原始对象形态。
7. 手机版/电脑版两套枚举，别混。
8. 提交审核在**手机版**页签；要点两次、躲开 loading 遮罩。
