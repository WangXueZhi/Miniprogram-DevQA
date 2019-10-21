# 小程序采坑实践

小程序开发过程中遇到的问题和解决思路

### 1. 如何在小程序内保持登录状态

问题：
在网页中我们常用 cookie 保持登录状态，但是小程序中没有 cookie 这个东西

解决：
可以把登录状态存在 storage 中，相当于网页中的 localStorage, 每次初始化小程序时，把 storage 中的登录 token 取出并放在请求的 header 中提交，后端需要从 header 中取出 token 验证

### 2. 如何在小程序的 web-view 内同步并保持登录状态

问题：
由于小程序和网页是两个独立的系统，各自独立维护登录状态，在小程序中打开网页时需要将登录状态传递进去，并且在后续页面还要持续保持

解决：

1. 在打开网页时，将登录的 token 拼接在网页 url 的参数内，网页中需要从 url 中取出登录 token，并保存在 localStorage，以便后续页面使用，
2. 为了同时兼容网页本身的 cookie 方式和小程序中的 url 参数方式，需要在网页中增加一个统一获取登录状态的方法：检查 url 是否有 token -> 检查 localStorage 是否有 token -> 使用 cookie。

### 3. 小程序中使用 canvas 处理图片，但不显示 canvas 元素

问题：
在小程序中，图片拼接需要用到 canvas，有时候我们不需要显示该 canvas，而只是需要利用该 canvas 处理出来的图片。

解决（二选一即可）：

1. 使用 cover-类的组件覆盖在 canvas 上面，如[cover-view](https://developers.weixin.qq.com/miniprogram/dev/component/cover-view.html)
2. 将 canvas 元素以绝对定位的方式定位到 page 的上边外部

### 4. 小程序中打开网页

问题：
在小程序中，我们可能会打开各种不同的 web 页面，为每个页面创建一个 page 一定是最差的方式，所以我们需要用过一个 page 处理各个 web 页面。

解决：
封装打开页面的统一方法，处理参数的拼接，处理不同环境下的域名：

```javascript
/**
     * 使用webview打开url
     * @param {string} url url
     * @param {object} query 查询参数
     * @param {boolean} checkEnv 检查环境
     */
    oepnWebPage(url, query, checkEnv = true) {
      let oepnUrl = url;

      // 是否内部链接
      if (!!checkEnv && (oepnUrl.includes("//static1.wdai.com/heyjie/") || oepnUrl.includes("//m.heyjie.cn/"))) {
        // 根据环境替换跳转地址
        if (config.deployEnv == "release") {
          oepnUrl = oepnUrl.replace("//static1.wdai.com/heyjie/m/", "//m.heyjie.cn/");
        } else {
          oepnUrl = oepnUrl.replace("//m.heyjie.cn/", "//static1.wdai.com/heyjie/m/");
        }
      }

      // 解析hash
      const hasHash = url.includes("#/");
      const hash = !!hasHash ? "#/" + oepnUrl.split("#/")[1] : "";

      // 解析参数
      let queryUrl = oepnUrl.split("#/")[0];
      const hasSearch = queryUrl.includes("?");
      let search = !!hasSearch ? `?${queryUrl.split("?")[1]}` : "";
      let newQuery = {};
      if (search) {
        newQuery = util.parseQueryString(search);
      }

      // 匹配白名单确认是否带上token，白名单需要开发者自己配置
      for (let i = 0; i < config.host_white_list.length; i++) {
        if (oepnUrl.includes(config.host_white_list[i])) {
          newQuery["miniprogram_t"] = wx.getStorageSync(config.LoginTokenKey);
        }
      }

      // 添加自定义参数
      if (query) {
        newQuery = { ...newQuery, ...query };
      }

      // 合成字符串参数
      const queryString = util.joinQueryString(newQuery);

      // 拼接url
      const finalUrl = queryUrl.split("?")[0] + queryString + hash;
      console.log(finalUrl)

      // 跳转webView
      wx.navigateTo({
        url: `/pages/webpage/webpage?url=${encodeURIComponent(finalUrl)}`
      })
    },
```

### 5. web-view 内页面设置小程序的分享内容

问题：
由于在网页中需要兼容微信网页分享和小程序分享，这里涉及到 web 页面和小程序通信，还需要根据环境做不同的处理。

解决：

1. 判断环境

```javascript
util.checkWechatEnv = function(callback) {
  var ua = window.navigator.userAgent.toLowerCase();
  if (ua.match(/MicroMessenger/i) == "micromessenger") {
    //判断是否是微信环境
    //微信环境
    wx.miniProgram.getEnv(function(res) {
      if (res.miniprogram) {
        // 小程序环境下
        callback("miniprogram");
      } else {
        //非小程序环境
        callback("wechat");
      }
    });
  } else {
    //非微信环境
    callback("outOfWechat");
  }
};
```

2. 设置分享

```javascript
// 环境
let env = "";
// 是否初始化过分享
let hasInitShare = false;
```

```javascript
// 设置微信分享
const wxShare = (shareData = {}) => {
  // 合并分享信息
  const sData = Object.assign(config.wechat.shareData, shareData);

  const creatShare = function(data) {
    if (env == "miniprogram") {
      // 在小程序内，利用wx.miniProgram.postMessage和小程序通信
      wx.miniProgram.postMessage({ data: { shareData: data } });
    }

    // 微信内非小程序环境
    if (env == "wechat") {
      // 设置
      const set = function(data) {
        wx.ready(() => {
          wx.onMenuShareAppMessage(data); //分享给朋友
          wx.onMenuShareTimeline(data); //分享到朋友圈
          wx.onMenuShareQQ(data); //分享到QQ
          wx.onMenuShareWeibo(data); //分享到微博
          wx.onMenuShareQZone(data); //分享到QZone
        });
      };

      if (!hasInitShare) {
        // 初始化微信分享
        share({
          appId: config.wechat.appId,
          shareUrl: location.href
        }).then(res => {
          // 设置初始化信息
          wxConfigData.timestamp = res.data.timestamp;
          wxConfigData.nonceStr = res.data.nonce;
          wxConfigData.signature = res.data.signature;

          if (typeof wx !== "undefined") {
            hasInitShare = true; // 修改初始化状态
            wx.config(wxConfigData); // 初始化配置
            set(data); // 设置分享内容
          }
        });
      } else {
        if (typeof wx !== "undefined") {
          set(data); // 设置分享内容
        }
      }
    }
  };

  // 已知环境，设置分享，未知环境，先检查环境再设置分享
  if (!!env) {
    creatShare(sData);
  } else {
    util.checkWechatEnv(res => {
      env = res;
      creatShare(sData);
    });
  }
};
```

3. 小程序内获取分享数据

```html
// wxml
<web-view src="{{url}}" wx:if="{{!!url}}" bindmessage="postmessage"></web-view>
```

```javascript
// js
postmessage(e) {
    const detail = e.detail;
    this.shareData = detail.data[0].shareData;
}
```

ps: [小程序内 wx.miniProgram.postMessage 触发时机](https://developers.weixin.qq.com/miniprogram/dev/component/web-view.html)

### 6. 小程序上传图片

问题：
通常我们图片都是以 blob 或者 base64 的形式提交给后端，然而，小程序中两种获取图片文件的方式： wx.chooseImage 和 wx.canvasToTempFilePath，都只能拿到图片文件的临时路径，无法直接转化成 blob 或者 base64。

解决：
小程序提供了 wx.uploadFile 方法把临时路径的文件转成 blob 的形式提交给指定的后端接口。

```javascript
wx.uploadFile({
  url: url, // 后端的接口地址
  filePath: tempFilePath, // 小程序里的临时路径
  name: "file",
  success(res) {
    resolve(JSON.parse(res.data));
  },
  fail(res) {
    reject(res);
  }
});
```

### 7. 小程序 swiper 指定 display-multiple-items 的 bug

问题：
当指定 display-multiple-items 数量时，如果实际 swiper-item 数量小于指定的值，会不显示任何内容

解决：
获取实际元素数量，填充空元素补满 display-multiple-items 指定的数值

### 8. 小程序内嵌 H5 处理未登录或登录状态过期的问题

问题：
当小程序打开 H5 页面时，传入 token 来传递登录状态，但是这个 token 可能会过期，需要重新完成登录流程

解决：

1. 在 H5 内需要封装请求函数做未登录的统一处理
2. 当接口返回未登录状态，判断网页环境，如果是在小程序环境内，跳转到小程序登录页
3. 小程序的 web-view 组件提供了开放接口，使 web 页面有能力跳转小程序页面

```html
<script
  type="text/javascript"
  src="https://res.wx.qq.com/open/js/jweixin-1.3.2.js"
></script>
```

```javascript
wx.miniProgram.navigateTo({ url: "/path/login/login?from=web" });
```

4. 上面跳转的地址后面带上了参数“from=web'”，是为了方便在小程序的逻辑内去处理刷新 web 页面登录状态，当小程序登录页面收到“from=web'”后，可以在小程序的缓存中存储标记
5. 完成登录后返回到 web 页面，小程序查找缓存中的标记，如果有该标记，则去取缓存中的新 token，修改到 web 页面的链接参数中，完成登录状态的更新

### 9. 小程序内富文本数据的处理

问题：
有时候，我们的数据形式是在后台以富文本的形式编辑保存的，也就是 html 字符串，比如文章，商品详情，如何处理富文本以及富文本中的图片

解决：

1. 小程序提供了 rich-text 组件用来展示富文本字符串，代码如下

```html
<rich-text nodes="{{htmlText}}"></rich-text>
```

2. 但是后台编辑的内容中，图片大小并不确定，可能非常大，到了小程序里就无法直视，所以需要对 img 标签的样式做修改，可以对图片宽做统一处理成 100%宽，代码如下：

```javascript
let newHtmlText = oldHtmlText.replace(
  /\<img/g,
  '<img style="width:100%;height:100%;display:block" '
);
this.setData({
  htmlText: newHtmlText
});
```

3. 更多富文本的使用说明[rich-text](https://developers.weixin.qq.com/miniprogram/dev/component/rich-text.html)
