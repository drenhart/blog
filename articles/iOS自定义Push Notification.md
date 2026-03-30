# iOS自定义Push Notification

### 1.远程推送的流程

![apns@2x.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/KM7qe92koVXPqpj8/img/a5e35ef3-52d1-4952-a549-642c6326f67e.png)

首先需要在程序中获取device token上传到Provider server(也就是我们自己的服务器)，Provider server在发送通知到APNs服务器的时候需要将device token一并发送，最后APNs服务器将通知发送到我们的手机上。

### 2.通知的内容扩展

![image](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/KM7qe92koVXPqpj8/img/76e04b1a-a2d4-4e50-a62f-1a9a015e645d.png)

新建一个Notification Content Extension的target，需要注意的是这个target和主target是平级关系。

在NotificationViewController中可以自定义扩展内容，在didReceive(\_ notification: UNNotification)函数可以接收通知

在Info.plist文件中一些key有特殊作用

| key | value |
| --- | --- |
| UNNotificationExtensionCategory | 唯一的标识 |
| UNNotificationExtensionDefaultContentHidden | 是否隐藏默认内容 |
| UNNotificationExtensionInitialContentSizeRatio | 高度相对于宽度的比例 |

样式如图：

![push.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/KM7qe92koVXPqpj8/img/de1b983e-29fa-4c94-9f7f-e9b329b95a5c.png)

### 3.通知的服务扩展

新建一个Notification Service Extension的target，在NotificationService这个类的didReceive方法中可以对通知的内容做一些修改，比方说将加密的数据进行解密和用url下载图片。

要想让服务扩展生效在发送通知的时候需要把mutable-content字段设置为1，并且有一个包含title的alert字段。

```json
{
    "aps": {
        "alert": {
            "title": "title",
            "subtitle": "subtitle",
            "body": "body"
        },
        "category": "myNotificationCategory",
        "mutable-content": 1,
        "content-available": 1,
        "badge":1
    },
    "ENCRYPTED_DATA" : "Salted__·öîQÊ$UDì_¶Ù∞èΩ^¬%gq∞NÿÒQùw"
}
```

### 4.静默推送

静默推送苹果官方建议在aps中只包含content-available字段，同时在一个小时内发送不要超过3条，当App接收到一条静默推送后会有30秒的时间来执行任务。

content-available字段用于使application(\_:didReceiveRemoteNotification:fetchCompletionHandler:)方法生效，这个方法不仅在App运行时调用，而且还会在App处于后台或挂起状态时被调用。

```json
{
   "aps" : {
      "content-available" : 1
   },
   "acme1" : "bar",
   "acme2" : 42
}
```

服务端在向客户端发送静默推送时需要将apns-push-type字段设为background，apns-priority字段设为5。

需要注意当App处于挂起状态收到静默推送时会调用application(\_:didFinishLaunchingWithOptions:)函数走冷启动的流程，同时launchOptions的key为remoteNotification

### 5.APNS中各字段具体作用

title：标题

subtitle：副标题

body：文字内容

category：ContentExtension的info.plist文件中UNNotificationExtensionCategory字段的值，默认为myNotificationCategory

badge：程序角标数量

sound：声音文件的名字。默认为系统声音

content-available：是否为静默推送

mutable-content：用于触发ServiceExtension用于修改通知内容或显示图片

Header字段:

apns-priority：推送优先级。默认为5最高10

apns-push-type：推送类型。有alert和background等选项

apns-collapse-id：推送ID。具有相同ID的推送会合并

### 6.测试远程推送通知

方法一：[Apple Push Console](https://icloud.developer.apple.com/dashboard/notifications)

填写相应的信息可以在控制台发送测试推送消息

![devicepush@2x.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/KM7qe92koVXPqpj8/img/5a4a9527-4b39-45e0-b4a5-8e9054ab49e9.png)

方法二：[Firebase Console](https://console.firebase.google.com/u/3/project/dramabox-3b7df/messaging)

填写fcm registration token可以发送测试通知，相较于Apple Push Console可以发送图片并且有预览

![Firebase Console.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/KM7qe92koVXPqpj8/img/3cb9efe3-fbf2-4c8a-939d-da44875e8910.png)

方法三：编写程序发送通知

下面是以C#程序为例编写的发送通知程序，填写相应fcm registration token就可以发送通知。好处是不用和服务端联调，调试一些字段自己就可以搞定。

```json
    static async Task Main(string[] args) {
        // 初始化 Firebase Admin SDK
        var serviceAccountKeyFilePath = "/Users/storymatrix06/Downloads/dramabox-3b7df-a135aea17d07.json";
        FirebaseApp.Create(new AppOptions() {
            Credential = GoogleCredential.FromFile(serviceAccountKeyFilePath)
        });

        await sendNoti();
    }

    static async Task sendNoti() {
        var registrationToken = "xxx";
        var message = new Message() {
            Data = new Dictionary<string, string>() {
                { "payload", "{\"actionType\":\"RECHARGE_LIST\"}" }
            },
            Notification = new Notification() {
                Title = "Price drop",
                Body = "5% off all electronics",
                ImageUrl = "xxx",
            },
            Apns = new ApnsConfig() {
                Headers = new Dictionary<string, string>() {
                    { "apns-priority", "10" },  // 设置 iOS 推送优先级为 10
                    { "apns-push-type", "alert" } // alert background location
                },
                Aps = new Aps() {
                    MutableContent = true,
                    Badge = 1,
                }
            },
            Token = registrationToken,
        };

        // Send a message to the device corresponding to the provided
        // registration token.
        string response = await FirebaseMessaging.DefaultInstance.SendAsync(message);
        // Response is a message ID string.
        Console.WriteLine("Successfully sent message: " + response);
    }
```

### 7.可能遇到的问题

##### ①依赖循环的问题

```json
Cycle inside DramaBox; building could produce unreliable results.Cycle details:→ Target 'DramaBox' has copy command from '/Users/storymatrix06/Library/Developer/Xcode/DerivedData/DramaBox-bazoyaqjeigjelawciltcfnazkuv/Build/Products/Debug-iphoneos/PushContentExtension.appex' to '/Users/storymatrix06/Library/Developer/Xcode/DerivedData/DramaBox-bazoyaqjeigjelawciltcfnazkuv/Build/Products/Debug-iphoneos/DramaBox.app/PlugIns/PushContentExtension.appex'○ That command depends on command in Target 'DramaBox': script phase “AppLovinQualityService”○ Target 'DramaBox' has process command with output '/Users/storymatrix06/Library/Developer/Xcode/DerivedData/DramaBox-bazoyaqjeigjelawciltcfnazkuv/Build/Products/Debug-iphoneos/DramaBox.app/Info.plist'○ Target 'DramaBox' has copy command from '/Users/storymatrix06/Library/Developer/Xcode/DerivedData/DramaBox-bazoyaqjeigjelawciltcfnazkuv/Build/Products/Debug-iphoneos/PushContentExtension.appex' to '/Users/storymatrix06/Library/Developer/Xcode/DerivedData/DramaBox-bazoyaqjeigjelawciltcfnazkuv/Build/Products/Debug-iphoneos/DramaBox.app/PlugIns/PushContentExtension.appex'
```

报错信息具体原因如下：

*   `DramaBox` 目标需要复制 `PushContentExtension.appex` 到其应用包中。
    
*   `DramaBox` 目标中包含的 "AppLovinQualityService" 脚本依赖于某些构建步骤，这些步骤又依赖于复制操作。
    
*   这种依赖关系最终形成了一个循环，使得构建过程无法继续。
    

解决这个问题需要将Build Phase中的Embed Foundation Extensions下面的Copy only when installing选项勾选。

![copy only when installing.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/KM7qe92koVXPqpj8/img/ebc20b48-62d7-406a-b691-f5a5573f6a81.png)

这个选项的作用是只在安装的时候才对扩展进行复制操作，于是打破了循环链条构建过程得以继续，问题也就解决了。

或者把AppLovinQualityService中的For install builds only选项勾选，也是一样的。

![for install builds only.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/KM7qe92koVXPqpj8/img/43508b9f-200f-409c-83b0-06dd41143598.png)

##### ②在真机上不生效问题

新建target默认的Minimum Deployments是最高版本，如果这个版本高于真机系统版本那么Notification Content Extension不会生效。

改成和主target一致的Minimum Deployments版本号就可以解决。

##### ③长按通知大图不显示问题

通过FirebaseMessaing发送通知时category字段已经设置为默认的myNotificationCategory，服务端编写程序时就不要显式指定category字段的值，否则会造成长按通知时大图显示不出来的问题。

### 8.通知相关回调的执行时机

##### ①UNUserNotificationCenterDelegate

收到通知后点击会走这个回调👇🏻

**func** userNotificationCenter(\_ center: UNUserNotificationCenter, didReceive response: UNNotificationResponse, withCompletionHandler completionHandler: **@escaping** () -> Void)

App在前台收到通知即将弹出时会走这个回调👇🏻

**func** userNotificationCenter(\_ center: UNUserNotificationCenter, willPresent notification: UNNotification, withCompletionHandler completionHandler: **@escaping** (UNNotificationPresentationOptions) -> Void)

##### ②UIApplicationDelegate

远程通知注册成功的回调，在这里可以拿到deviceToken👇🏻

**func** application(\_ application: UIApplication, didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data)

远程通知注册失败的回调

**func** application(\_ application: UIApplication, didFailToRegisterForRemoteNotificationsWithError error: **any** Error)

content-available字段为1时，这个方法不仅在App处于前台时调用，而且还会在App处于后台或挂起状态时被调用👇🏻

**func** application(\_ application: UIApplication, didReceiveRemoteNotification userInfo: \[AnyHashable: **Any**\], fetchCompletionHandler completionHandler: **@escaping** (UIBackgroundFetchResult) -> Void)

##### ③UNNotificationServiceExtension

在通知弹出之前执行，可以对通知内容做一些处理，时限为30秒👇🏻

**func** didReceive(\_ request: UNNotificationRequest, withContentHandler contentHandler: **@escaping** (UNNotificationContent) -> Void)

30秒时限即将到期时执行，处理通知的最后机会👇🏻

**func** serviceExtensionTimeWillExpire()

### 9.通知的几种处理形式

在willPresent函数回调中我们可以选择几种处理办法

1.  alert：无论程序在前台后台都显示一个警告框或横幅通知并添加到通知中心（iOS14方法过期）
    
2.  badge：增加角标
    
3.  sound：播放与通知相关联的声音文件
    
4.  list：程序在前台不显示横幅只添加到通知中心，程序在后台显示横幅并添加到通知中心（需要iOS14版本以上）
    
5.  banner：程序在前台显示横幅不添加到通知中心，程序在后台显示横幅并添加到通知中心（需要iOS14版本以上）
    

### 10.主target和App Extension之间的数据沟通

##### ①AppGroups

AppGroups是一个在主应用和App Extension之间共享的空间，主应用写入数据之后在App Extension中即可读取。

##### ②UserDefaults

需要修改证书添加App Groups的Capability，同时写入和读取也需要用UserDefaults(suiteName: "app group name")的形式。需要注意原有UserDefaults.standard形式写入的数据suitName形式是读取不了的

### 11.App Extension神策打点

初始化神策SDK和打点代码均需在主线程，不用担心在主线程执行会造成UI卡顿，因为本质上来讲App Extension和主Target是两个不同的App。

### 资料：

[UNNotificationContentExtension](https://developer.apple.com/documentation/usernotificationsui/unnotificationcontentextension)

[Customizing the Appearance of Notifications](https://developer.apple.com/documentation/usernotificationsui/customizing-the-appearance-of-notifications)

[Declaring your actionable notification types](https://developer.apple.com/documentation/UserNotifications/declaring-your-actionable-notification-types)

[UNNotificationServiceExtension](https://developer.apple.com/documentation/usernotifications/unnotificationserviceextension#mentions)

[Modifying content in newly delivered notifications](https://developer.apple.com/documentation/usernotifications/modifying-content-in-newly-delivered-notifications)

[Setting up a remote notification server](https://developer.apple.com/documentation/usernotifications/setting-up-a-remote-notification-server)

[Sending notification requests to APNs](https://developer.apple.com/documentation/usernotifications/sending-notification-requests-to-apns)

[Testing notifications using the Push Notification Console](https://developer.apple.com/documentation/usernotifications/testing-notifications-using-the-push-notification-console#Generate-an-authentication-token)

[Generating a remote notification](https://developer.apple.com/documentation/usernotifications/generating-a-remote-notification)

[Registering your app with APNs](https://developer.apple.com/documentation/usernotifications/registering-your-app-with-apns)

[Send an image in the notification payload](https://firebase.google.com/docs/cloud-messaging/ios/send-image#node.js)

[依赖循环问题解决](https://stackoverflow.com/questions/50709330/cycle-inside-building-could-produce-unreliable-results-xcode-error)

[Pushing background updates to your App](https://developer.apple.com/documentation/usernotifications/pushing-background-updates-to-your-app#Create-a-background-notification)