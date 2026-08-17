# 一品威客 signature 参数逆向记录

目标页面：[https://www.epwk.com/login.html](https://link.wtturl.cn/?target=https%3A%2F%2Fwww.epwk.com%2Flogin.html&scene=im&aid=497858&lang=zh)

打开一品威客登录页，填好账号密码抓登录接口，看请求载荷发现明文没加密，但请求头里多了个 signature。我重复提交了好几次登录，对比每一次的请求头，这个参数每次数值都不一样，确定这就是要逆向的签名参数。

## 找加密入口

一开始直接在网站 JS 文件里搜 signature，搜出来两处结果，其中一段代码看着关联性很强；另外也可以直接搜接口路径 user/login 来找，两种方法都行，最后都得一步步跟调用栈。

找到可疑代码后在这行打上断点：

```
j.Signature = Object(f.a)(j, L, h.i ? h.f : h.c);
```

重新触发登录提交，代码正好断在这里，我把相关变量复制到浏览器控制台跑了一遍，输出的字符串和抓包拿到的 signature 格式完全对得上，确认这里就是生成签名的入口。

顺着调用栈一步一步往里跟，追到了真正组装加密内容的函数 h：

```
h = function(t) {
    var data = arguments.length > 1 && void 0 !== arguments[1] ? arguments[1] : {}
      , e = arguments.length > 2 && void 0 !== arguments[2] ? arguments[2] : "a75846eb4ac490420ac63db46d2a03bf"
      , n = e + f(data) + f(t) + e;
    return n = d(n),
    n = v(n)
}
```

逻辑很清晰：先拿固定盐值 e、序列化后的请求参数 data、接口路径 t 拼出字符串 n，先过一层 d 加密，加密完的结果再丢进 v 函数二次加密，最后返回的结果就是接口要的 signature。

## 拆解两层加密算法

### 第一层 d 函数

```
d = function(data) {
    return o.a.MD5(data).toString()
}
```

很直观，就是标准 MD5 哈希，把拼接好的长字符串做 MD5 处理。

### 第二层 v 函数

```
, v = function(data) {
    return function(data) {
        return o.a.AES.encrypt(data, l.key, {
            iv: l.iv,
            mode: o.a.mode.CBC,
            padding: o.a.pad.Pkcs7
        }).toString()
    }(data)
}
```

第二层是 AES 加密，模式 CBC，Pkcs7 填充，用代码里定义好的 key 和 iv 对 MD5 结果再加密一次，最终密文放到请求头。

## 本地复现

把完整流程捋顺：固定盐值拼接参数与接口路径 → MD5 加密 → AES-CBC 二次加密，按照这个逻辑在本地写代码还原签名生成逻辑

![image-20260816145334733](C:\Users\Admin\Desktop\一品威客\image-20260816145001810.png)

固定盐值 + 序列化请求参数 + 序列化接口路径 + 固定盐值 → MD5 哈希 → AES-CBC 加密 → 输出密文作为请求头 signature