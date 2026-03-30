# iOS Communication Notifications和time sensitive notification

## time sensitive notification(即时通知)

即时通知可以忽略专注模式等的通知拦截，直接可以展示同时在锁屏上持续保留1小时，且在通知栏上会有 “时效性” 字样。 

### apns payload

在aps下发字段中需要将interruption-level设置为time-sensitive

```java
                Aps = new Aps() {
                    MutableContent = true,
                    Sound = "default",
                    CustomData = new Dictionary<string, object>()
                    {
                        { "interruption-level", "time-sensitive" }
                    }
                }
```

### 证书及客户端所需修改

证书勾选Time Sensitive Notifications，同时客户端添加Time Sensitive Notifications的capability

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/2M9qP5jgYe874O01/img/3ad2a88e-f92a-4ca4-9760-7b55268ca2e9.png)

## Communication Notifications

绕过概览单独显示并且能绕过勿扰模式(Focus)，同时可将左侧App图标替换为自定义的图标。

### plist文件字段添加

在Notification Service Extension中添加如下字段

```java
<key>NSExtension</key>
	<dict>
		<key>NSExtensionAttributes</key>
		<dict>
			<key>IntentsSupported</key>
			<array>
				<string>INSendMessageIntent</string>
			</array>
		</dict>
</dict>
```

在主工程中也添加如下字段

```java
<key>NSUserActivityTypes</key>
	<array>
		<string>INSendMessageIntent</string>
	</array>
```

### 证书及客户端所需修改

证书勾选Communication Notifications，同时客户端添加Time Sensitive Notifications的capability

### Notification Service Extension中的操作

在Notification Service Extension中构建INSendMessageIntent

```java
    private func sendMessageIntent(title: String) async -> INSendMessageIntent {
        let handle = INPersonHandle(value: "unique-user-id", type: .unknown)
        
        let urlString = "https://flowercover.kydap.cn/cppartner/4x1/41x0/410x0/41000102143/41000102143.jpg?t=1706927032539&image_process=resize,w_200"
        
        let avatar = await loadAvatar(from: urlString) ?? INImage.systemImageNamed("person.circle.fill")
        
        let sender = INPerson(personHandle: handle,
                              nameComponents: nil,
                              displayName: title,
                              image: avatar,
                              contactIdentifier: nil,
                              customIdentifier: "unique-user-id")


        let intent = INSendMessageIntent(recipients: nil,
                                         outgoingMessageType: .outgoingMessageText,
                                         content: "Message content",
                                         speakableGroupName: nil,
                                         conversationIdentifier: "unique-user-id-conv",
                                         serviceName: nil,
                                         sender: sender,
                                         attachments: nil)
        return intent
    }
```

之后在didReceive方法中调用

```java
    override func didReceive(_ request: UNNotificationRequest, withContentHandler contentHandler: @escaping (UNNotificationContent) -> Void) {
            let intent = await sendMessageIntent(title: request.content.title)
            let interaction = INInteraction(intent: intent, response: nil)
            interaction.direction = .incoming
            try? await interaction.donate()
    }
```

### 与右侧小图并存问题

原有Firebase库下载右侧小图的逻辑会在下载完图片之后直接调用 contentHandler(bestAttemptContent) ，这会导致提前结束通知流程。

解决方案是自己实现下载右侧小图的逻辑，在Communication Notification逻辑处理完之后再统一调用contentHandler(bestAttemptContent)来结束整个流程。

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/2M9qP5jgYe874O01/img/731d6c6c-f86d-4662-b1cd-4ed92e6f765c.png)

### 即时通知与Communication Notification可否并存？

不冲突，可并存。效果就是既有即时通知的效果，同时左侧更换为自定义图标，右侧小图也可以存在。

### 资料：

[Stay Connected: Mastering iOS Communication Notifications](https://arturgruchala.com/stay-connected-mastering-ios-notifications-for-seamless-communication/)

[Send communication and Time Sensitive notifications](https://wwdcnotes.com/documentation/wwdcnotes/wwdc21-10091-send-communication-and-time-sensitive-notifications/)

[Implementing communication notifications](https://developer.apple.com/documentation/usernotifications/implementing-communication-notifications)

[iOS 时效性通知（即时通知）](https://docs.getui.com/getui/scene/iosNotify/)