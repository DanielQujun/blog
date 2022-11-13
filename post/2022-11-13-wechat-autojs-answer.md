---
layout:     post
title:      "微信视频自动接听AutoJS"
subtitle:   "微信视频自动接听AutoJS"
date:       2022-11-13
author:     "QuJun"
URL: "/2022/11/13/wechat-autojs-answer/"
image:      "https://qiniu.bobozhu.cn/xihuLake.jpeg"
tags:
    - autojs
    - wechat
    - 微信
    - 自动接听

categories: [ life ]
---

## 背景

某天突然很想念90岁奶奶，因为工作的原因不能经常回去看她。于是想着买个平板给她，一开始在京东搜自动接听电话的平板，发现价格不便宜，也不太方便还需要装APP之类的。

然后转念一想不如买个安卓平板，写一个微信自动接听的脚本，然后就可以随时跟她视频了，这样比打电话更亲切，而且因为是微信，其他长辈用起来也更方便。

然而在网上一搜，发现只有大多数是一些灰产的软件或者按键精灵之类的东西，不太想用。有点意外这么广泛的需求居然没人做。

经过一番搜索找到了AutoJS，可以提供JS运行环境和接口可以模拟点击滑动等操作，非常强还可以用vscode跟手机联调，很符合我的场景。可惜的是因为灰产用的太多，作者目前已经停止更新开源版本，并且商业版也不再支持操作主流软件，好吧，本来嫌麻烦想买商业版的。。。

[https://github.com/hyb1996/Auto.js](https://github.com/hyb1996/Auto.js)

网上的教学视频很多，这里直接分享下我的代码吧。

## 自动接听

```javascript
auto;
toast("Starting wechat monitor!!!!!") 
var weixin_i = 0;

// 定时检测微信视频电话
setInterval(function(){
    log(weixin_i+"th detect weixin call!!!!!!!!!!")
    weixin_i++;
    toast(weixin_i * 6 + "秒");
    // findOne()是阻塞方法
    var weixin = className("android.widget.Button").desc("接听").findOne()
    if (weixin) {
    // 继续响铃3秒，避免秒接
    sleep(3000);
    click(weixin.bounds().centerX(), weixin.bounds().centerY())
    log("diji!!!!!!!!!!!!!!!!!!!!!")
    }
}, 6000);
```

## 唤醒屏幕

```javascript
toast("Starting wake-up device program!!!!!") 
var num = 0;

// 经测试30分钟平板进入彻底休眠，微信视频不会弹窗，需要不定期唤醒屏幕
// 省电模式无法关闭，脚本执行间隔不稳定
setInterval(function(){
    num++;
    log("This is "+ num + "th unlocking!!!!!!!!!!!!!!!!!!!!!")
    unlock();
}, 300000);

// 解锁屏幕
function unlock()
{
    if(!device.isScreenOn())
    {
        log("This is "+ num + "th waking up!!!!!!!!!!!!!!!!!!!!!");
        device.wakeUp();
        log("Device wakened!!!!!!!!!!!!!!!!!!!!!");
        sleep(500);
        swipe(500, 1000, 500, 500, 1000);
        log("swipe success!!!!!!!!!!!!!!!!!!!!!");
        password_input();
        //back_to_screen();
        var appName = "微信";
        launchApp(appName);
        sleep(5000);
        // 按下home键
        //home()
    } else {
        log("Screen is On!!!!!!!!!!!!!!!!!!!!!");
    }
}

function back_to_screen()
{
    var p = text("返回").findOne().bounds();
    click(p.centerX(), p.centerY());
    sleep(100);
}

function password_input()
{
    var password = "1234567"
    for(var i = 0; i < password.length; i++)
    {
        var keyboard = text(password[i].toString()).findOne(500)
        if (keyboard) { 
        p = keyboard.bounds();
        log(password[i] + "x Position:"+p.centerX()+password[i] + "y Position:"+p.centerY());
        if (p.centerX() >0 && p.centerY() >0) {
            click(p.centerX(), p.centerY());
            sleep(100);
        }
       

        }
    }
}
```

## 测试用例

大概测试了一周，主要以下几个场景：

1. 重启后自动运行
2. 低电量
3. 锁屏一个小时后

发现华为青春版有时候锁屏电量优化，过一段时间微信电话打过来都唤不醒屏幕，然后发现系统又没选项可以设置，经过一番折腾发现插着电源就不会优化了，也算是一种解决方案吧。。