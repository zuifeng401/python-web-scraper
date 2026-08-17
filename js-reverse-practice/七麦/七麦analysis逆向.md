# 七麦数据 analysis 参数逆向笔记

目标网站：[https://www.qimai.cn/](https://link.wtturl.cn/?target=https%3A%2F%2Fwww.qimai.cn%2F&scene=im&aid=497858&lang=zh) 这次要逆向的是接口里的 analysis 加密参数。

打开七麦数据网页，刷新页面抓接口，我需要的榜单数据接口地址是： [https://api.qimai.cn/indexV2/getIndexRank?analysis=eyEzECU8OEd1EhgKCgVcTjNUSA8dDTNbVVIbNgBXXSVFVlpNT04FAAVWVFkBdkZV&setting=0&genre=5000](https://link.wtturl.cn/?target=https%3A%2F%2Fapi.qimai.cn%2FindexV2%2FgetIndexRank%3Fanalysis%3DeyEzECU8OEd1EhgKCgVcTjNUSA8dDTNbVVIbNgBXXSVFVlpNT04FAAVWVFkBdkZV%26setting%3D0%26genre%3D5000&scene=im&aid=497858&lang=zh) 多次重复发起请求测试，观察到 analysis 参数每次请求的值都不一样，确定这个动态参数就是本次逆向目标。

## 定位加密入口

一开始还是老办法，直接全局关键词搜 `analysis`，结果页面 JS 里完全匹配不到这个词，这条路走不通。 换思路，直接检索接口路径 `indexV2/getIndexRank`，搜出来两处相关代码，第一处就是页面发起请求的地方：

```
getData: function(t) {
    var a = this;
    this.loading = !0;
    var e = this.getReqestParams(t);
    this.$http.get("/indexV2/getIndexRank", {
    params: e
    }).then((function(t) {
    a.loading = !1;
    var e = t.data;
    1e4 === e.code ? a.rankReqSuccess(e) : a.rankReqFailed(e)
    }
    )).catch((function(t) {
    a.$Message.destroy(),
    a.$Message.error("网络错误，请稍后重试")
    }
    ))
}
```

我在这段代码打上断点，刷新页面触发接口，程序成功断住。顺着调用栈一步步往里跟，追到了一段经过 OB 混淆的代码块：

```
o()[Wt][Kt][Ut](function(t) {
                try {
                    var n;
                    f || $ != s || (n = (0,
                    i[Zt])(m),
                    s = c[x][k][Rt] = -(0,
                    i[Zt])(l) || +new R[K] - r2 * n);
                    var e, r = +new R[K] - (s || H) - 1661224081041, a = [];
                    return void 0 === t[Ot] && (t[Ot] = {}),
                    R[W][o7](t[Ot])[M](function(n) {
                        if (n == v)
                            return !B;
                        t[Ot][_2](n) && a[b](t[Ot][n])
                    }),
                    a = a[jt]()[I5](N),
                    a = (0,
                    i[Jt])(a),
                    a = (a += p + t[qt][T](t[Tt], N)) + (p + r) + (p + 3),
                    e = (0,
                    i[Jt])((0,
                    i[Qt])(a, d)),
                    -B == t[qt][O](v) && (t[qt] += (-B != t[qt][O](Bn) ? Nn : Bn) + v + B5 + R[V5](e)),
                    t
                } catch (t) {}
            }
```

这里能明显看出逻辑：变量`t`刚传入的时候，请求参数里还没有 analysis，走到这段代码执行完之后，t 内就生成好了 analysis，能确定加密逻辑就藏在这段 OB 混淆代码里。

我重新走一遍调用流程，锁定核心生成语句： `e = (0,i[Jt])((0,i[Qt])(a, d))` 这行就是生成加密字符串的核心入口，分两层函数，逐层跟进拆开：

1. 内层 `(0,i[Qt])(a, d)` 对应函数 h，是核心异或加密逻辑：

```
function h(n, t) {
                t = t || u();
                for (var e = (n = n[$5](N))[z], r = t[z], a = q5, i = H; i < e; i++)
                    n[i] = o(n[i][a](H) ^ t[(i + 10) % r][a](H));
                return n[I5](N)
}
```

1. 外层 `(0,i[Jt])()` 对应函数 p，负责转码处理：

```
function p(t) {
                t = R[V5](t)[T](/%([0-9A-F]{2})/g, function(n, t) {
                    return o(Y5 + t)
                });
                try {
                    return R[Q5](t)
                } catch (n) {
                    return R[W5][K5](t)[U5](Z5)
                }
}
```

## OB 混淆代码调试小技巧

整套 JS 是标准 OB 混淆，变量全是无意义单字母、随机命名，直接看完全读不懂。 浏览器控制台自带调试优化，把鼠标悬浮在混淆变量上，会自动显示变量对应的真实值与引用内容，不用手动挨个解混淆，大幅降低调试难度。

## 本地逻辑还原

把剥离出来的两段核心加密函数完整抠出，整理加密流程，在本地 Python 环境复现生成 analysis 参数的逻辑。 相关调试截图保存路径：

1. 浏览器混淆代码调试截图：![image-20260816154634134](C:\Users\Admin\Desktop\七麦\image-20260816154138119.png)
2. 本地代码实现截图 1：![image-20260816154653885](C:\Users\Admin\Desktop\七麦\image-20260816154653885.png)
3. 本地代码实现截图 2：![image-20260816154705199](C:\Users\Admin\Desktop\七麦\image-20260816154255826.png)

希望我的逆向思路对你有帮助
---

## ⚠️ 免责声明

**本项目仅供学习交流使用，请勿用于非法用途。**

- 本案例用于技术研究和学习目的
- 请遵守目标网站的robots.txt协议和服务条款
- 请勿用于商业爬虫、数据售卖等违法行为
- 请勿对目标网站进行高频请求，避免影响正常服务
- 使用本项目代码产生的一切后果由使用者自行承担

**技术无罪，使用需谨慎。请合法合规使用爬虫技术。**
