# 财新网 - 登录密码AES加密 + Charles抓包实战

## 项目信息

- **网站**：https://u.caixin.com/web/login
- **难度**：⭐⭐⭐⭐（中高级）
- **技术**：AES-ECB加密 + Charles抓包
- **完成时间**：2024

---

## 技术亮点

### 🔥 解决抓包难题

**问题**：Chrome DevTools无法抓到登录请求
- 原因：登录后立即跳转，Network面板被清空
- 即使开启`Preserve Log`，载荷仍会丢失

**解决方案**：使用Charles系统级代理
- Charles独立于浏览器，不受页面刷新影响
- 配置SSL证书解密HTTPS
- 成功捕获完整请求数据

---

## 逆向过程

### Step 1：Charles抓包

配置Charles代理 → 捕获登录请求 → 提取加密password：
```
ZNjpn6lHCMxsI9vuCJAczA%3D%3D
```

![Charles抓包](./image-20260816122038134.png)

---

### Step 2：定位加密入口

1. 全局搜索 `password:`
2. 下断点调试，找到参数组装代码：

```javascript
callback: function() {
    var o = {
        account: this.form.mobile,
        password: this.encode(this.encrypt(this.form.password)),
        deviceType: this.deviceType,
        unit: this.unit,
        areaCode: this.encode(this.form.areaCode),
        extend: this.encode(JSON.stringify({
            resource_article: null != s ? s : ""
        }))
    };
}
```

**加密链路**：
```
明文密码 → encrypt加密 → encode编码 → 提交
```

---

### Step 3：拆解加密算法

跟进`encrypt`函数：

```javascript
u = function(t) {
    // AES固定密钥
    var e = r.a.enc.Utf8.parse("G3JH98Y8MY9GWKWG")
    var n = r.a.enc.Utf8.parse(t)
    var a = r.a.AES.encrypt(n, e, {
        mode: r.a.mode.ECB,
        padding: r.a.pad.Pkcs7
    });
    
    // URL编码
    return encodeURIComponent(a.toString())
}
```

---

## 核心参数

- **加密算法**：AES-ECB
- **密钥**：`G3JH98Y8MY9GWKWG`
- **填充方式**：Pkcs7
- **后置处理**：`encodeURIComponent` URL编码

---

## 完整流程

```
明文密码
  ↓
UTF8编码
  ↓
AES-ECB加密（Pkcs7填充）
  ↓
Base64编码（AES库自动）
  ↓
encodeURIComponent URL编码
  ↓
ZNjpn6lHCMxsI9vuCJAczA%3D%3D
```

---

## 学到的经验

### 1. 抓包技巧升级
- Chrome DevTools有局限性
- Charles/Fiddler是必备工具
- 系统级代理可以抓到所有流量

### 2. 编码细节很重要
- `encodeURIComponent`是标准函数
- 空格编码为`%20`（不是`+`）
- 特殊符号要百分号编码
- 如果实现不标准会导致密文不匹配

### 3. AES-ECB vs AES-CBC
- ECB模式不需要IV（初始向量）
- CBC模式需要IV参数
- ECB安全性较低但实现简单

---

## 商业价值

这个案例展示了两个重要能力：

1. **抓包能力**：很多网站会有跳转/重定向，需要用Charles/Fiddler
2. **标准加密处理**：AES-ECB是最常见的加密方式

这两个技能在接单中**极其常用**，约50%的项目都会遇到类似问题。

---

## 相关项目

- [一品威客双重加密](../一品威客/)
- [漫画爬虫完整项目](../../comic_scraper/)

---

## ⚠️ 免责声明

**本项目仅供学习交流使用，请勿用于非法用途。**

- 本案例用于技术研究和学习目的
- 请遵守目标网站的robots.txt协议和服务条款
- 请勿用于商业爬虫、数据售卖等违法行为
- 请勿对目标网站进行高频请求，避免影响正常服务
- 使用本项目代码产生的一切后果由使用者自行承担

**技术无罪，使用需谨慎。请合法合规使用爬虫技术。**
