# iOS实时活动

iOS 的**实时活动**（Live Activities）功能允许应用在锁屏和灵动岛（Dynamic Island）上展示实时更新的信息。这适用于显示诸如比赛比分、外卖订单状态、计时器等持续变化的内容。该功能从 iOS 16.1 开始支持。

开发者可以使用 **ActivityKit** 框架来创建和管理实时活动，并通过 **Push Notifications** 或 **Background Tasks** 实时更新内容，确保用户无需进入应用即可查看最新状态。

## 实时活动的几种显示状态

1.单独在灵动岛

灵动岛左右区域被同一个实时活动占满

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Mp7ldVm1vjdvqBQN/img/678819f8-1923-4833-a489-cb860863fae6.png)

2.与另一个实时活动同时存在于灵动岛

占据灵动岛的左半边或者右半边

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Mp7ldVm1vjdvqBQN/img/929fbe39-7521-4988-9644-aa2c5a0ac3dc.png)

3.长按之后的展开状态

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Mp7ldVm1vjdvqBQN/img/39296af2-5226-4d59-b9ca-cb7861a81cb2.png)

4.在通知中心

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Mp7ldVm1vjdvqBQN/img/3860abc8-0ca3-4057-a24a-172c3a47e483.png)

5.横幅提示

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Mp7ldVm1vjdvqBQN/img/4073a466-08e4-4e5b-a6f7-6e0934416eea.png)

## 展开状态视图的分割区域

默认情况下系统将展开状态视图分割为Center,Leading,Trailing,Bottom四个区域

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Mp7ldVm1vjdvqBQN/img/37c98c05-7d4e-4d4a-8f4b-14ef9b358d16.png)

## 集成实时活动功能

1.在项目中添加Widget Extension并勾选Include Live Activity

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/Mp7ldVm1vjdvqBQN/img/dc6cdbc5-3301-4659-88e8-bdebb1903a9d.png)

2.在Info.plist文件中添加Supports Live Activities字段并设置为YES

3.定义包含静态和动态数据的Attributes

```swift
struct LiveActivityExAttributes: ActivityAttributes {
    public struct ContentState: Codable, Hashable {
        // 动态数据
        var emoji: String
    }

    // 静态数据
    var name: String
}
```

4.用Attributes配置实时活动的视图

```swift
        ActivityConfiguration(for: LiveActivityExAttributes.self) { context in
            // Lock screen/banner UI goes here
            VStack {
                Text("Hello \(context.state.emoji)")
            }
            .activityBackgroundTint(Color.cyan)
            .activitySystemActionForegroundColor(Color.black)

        } dynamicIsland: { context in
            DynamicIsland {
                // Expanded UI goes here.  Compose the expanded UI through
                // various regions, like leading/trailing/center/bottom
                DynamicIslandExpandedRegion(.leading) {
                    Text("Leading")
                }
                DynamicIslandExpandedRegion(.trailing) {
                    Text("Trailing")
                }
                DynamicIslandExpandedRegion(.bottom) {
                    Text("Bottom \(context.state.emoji)")
                    // more content
                }
            } compactLeading: {
                Text("L")
            } compactTrailing: {
                Text("T \(context.state.emoji)")
            } minimal: {
                Text(context.state.emoji)
            }
            .widgetURL(URL(string: "http://www.apple.com"))
            .keylineTint(Color.red)
        }
```

5.在主Target编写启动更新和结束实时活动的代码

#### 开始实时活动

```swift
    @objc private func showActivity() {
        if ActivityAuthorizationInfo().areActivitiesEnabled {
            do {
                if currentActivity == nil {
                    let attributes = LiveActivityExAttributes(name: "Activity")
                    let initialState = LiveActivityExAttributes.ContentState(emoji: "😭")
                    let activity = try Activity.request(attributes: attributes, content: .init(state: initialState, staleDate: nil), pushType: nil)
                    currentActivity = activity
                }
            } catch {
                print(error.localizedDescription)
            }
        }
    }
```

#### 更新实时活动

```swift
    private func performUpdate() async throws {
        if let activity = currentActivity {
            try await Task.sleep(for: .seconds(2))
            let newState = LiveActivityExAttributes.ContentState(emoji: "😈")
            let alertConfig = AlertConfiguration(
                title: "😈 has been knocked down!",
                body: "Open the app and use a potion to heal 😈.",
                sound: .default
            )
            await activity.update(ActivityContent<Activity<LiveActivityExAttributes>.ContentState>(state: newState, staleDate: Date.now + 15, relevanceScore: 100), alertConfiguration: alertConfig)
        }
    }
```

#### 结束实时活动

```swift
    private func performEnd() async {
        if let activity = currentActivity {
            let finalState = LiveActivityExAttributes.ContentState(emoji: "😓")
            await activity.end(ActivityContent(state: finalState, staleDate: nil), dismissalPolicy: .immediate)
        }
    }
```

## SwiftUI 对标的 UIKit 视图

| **SwiftUI** | **UIKit** |
| --- | --- |
| Text 和 Label | UILabel |
| TextField | UITextField |
| TextEditor | UITextView |
| Button 和 Link | UIButton |
| Image | UIImageView |
| NavigationView | UINavigationController 和 UISplitViewController |
| ToolbarItem | UINavigationItem |
| ScrollView | UIScrollView |
| List | UITableView |
| LazyVGrid 和 LazyHGrid | UICollectionView |
| HStack 和 LazyHStack | UIStack |
| VStack 和 LazyVStack | UIStack |
| TabView | UITabBarController 和 UIPageViewController |
| Toggle | UISwitch |
| Slider | UISlider |
| ProgressView | UIProgressView 和 UIActivityIndicatorView |
| Picker | UISegmentedControl |
| DatePicker | UIDatePicker |
| Alert | UIAlertController |
| ActionSheet | UIAlertController |
| Map | MapKit |

## 图片问题

实时活动扩展本身不支持发起网络请求，需要更新profile以支持App Group然后在主Target下载图片到共享文件夹。在这个过程中涉及图片的解码问题，实测66kb大小图片解码后会占7.93M内存，过高的内存占用会导致实时活动无法推出或一直显示为加载状态，所以在下载图片后注意一定要进行压缩处理。

**远程开启实时活动**

 iOS>=17.2开始允许[通过APNs启动Live Activity](https://developer.apple.com/documentation/activitykit/starting-and-updating-live-activities-with-activitykit-push-notifications)，可以在APP处于非存活状态唤起实时活动。注意当前功能仅仅是启动LA，若要通过APNs更新LA还需要根据上述获取当前LA的pushToken来更新。

开发者需要在App启动后第一时间获取pushToStartToken，用于推送启动Live Activity。

```swift
    // MARK: 注册实时活动
    func listenPushToStartTokenUpdates() {
        Task {
            if #available(iOS 17.2, *) {
                let startTokenData = Activity<RemoteLiveActivityAttributes>.pushToStartToken
                
                if let token = startTokenData {
                    let startToken = String(data: token, encoding: .utf8)
                    //TODO: 开发者上传现有的pushToStartToken
                    uploadTokenToServer(token: startToken, activityID: "")
                    DLog.info("pushToStartToken Activity startToken=\(startToken) ")
                }
                
                //pushToStartToken为空。App创建LA时，会触发下面的回调。可以拿到pushToStartToken
                Task {
                    for await tokenData in Activity<RemoteLiveActivityAttributes>.pushToStartTokenUpdates {
                        //监听token更新 注意线程
                        let startToken = tokenData.map { String(format: "%02x", $0) }.joined()
                        //TODO: 开发者上传变更后的startToken
                        uploadTokenToServer(token: startToken, activityID: "")
                        DLog.info("pushToStartTokenUpdates Activity startToken=\(startToken) ")
                    }
                }
            } else {
                // Fallback on earlier versions
            }
        }
    }
```

经过测试，首次App启动时，pushToStartToken可能没有回调：

 ios18.0以前的系统版本在APP在手机重启、app首次安装的情况下会 回调一次返回 push-to-start-token(被动触发条件苛刻),后续只有在 拉起实时活动的时候才会回调；iOS18+的系统每次APP重启都会回调

苹果官方问答 [https://forums.developer.apple.com/forums/thread/741939](https://links.jianshu.com/go?to=https%3A%2F%2Fforums.developer.apple.com%2Fforums%2Fthread%2F741939)

pushToStartToken传不同的Attributes都是同一个token，pushToStartTokenUpdates传不同的Attributes也都是同一个token，这两个方法的token也是一样的。

pushToStartToken不走的原因是不能在冷启时调用，应用冷启动时，系统可能需要时间初始化推送服务，令牌不会立即可用。

[https://developer.apple.com/forums/thread/759827](https://developer.apple.com/forums/thread/759827)

## iOS18的限制

iOS18上限制5-15秒才允许刷新一次，官方称实时活动最初的设计目的并非用于创建“实时体验”，之前允许每秒刷新是一个“API 漏洞”。

### 资料

[Live Activities](https://developer.apple.com/design/human-interface-guidelines/live-activities/)

[实时活动设计指导](https://developer.apple.com/cn/design/human-interface-guidelines/live-activities)

[实时活动System Experience](https://developer.apple.com/design/human-interface-guidelines/live-activities/?utm_source=chatgpt.com)

[Displaying live data with Live Activities](https://developer.apple.com/documentation/ActivityKit/displaying-live-data-with-live-activities)

[Starting and updating Live Activities with ActivityKit push notifications](https://developer.apple.com/documentation/activitykit/starting-and-updating-live-activities-with-activitykit-push-notifications#Add-the-Push-Notifications-capability)

[SwiftUI essentials Creating and combining views](https://developer.apple.com/tutorials/swiftui/creating-and-combining-views)

[SwiftUI essentials Building lists and navigation](https://developer.apple.com/tutorials/swiftui/building-lists-and-navigation)