# DramaBox编译时间分析及优化

## 编译时间分析

首先在DramaBox主工程Build settings的Other Swift Flags选项中添加-Xfrontend -debug-time-function-bodies，这行的含义是在编译时显示出每个函数的编译时间。

之后在Report Navigator中就可以看到具体的编译时间。

![report navigator.jpeg](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZARBYY538qLX/img/22ef1efe-c382-471a-a9d0-249670dff877.jpeg)

我们的目的是找到编译时间过长的函数，在Report Navigator中一个一个找还是太麻烦。所以接下来通过命令行来编译并导出函数编译时间列表

```swift
xcodebuild -workspace DramaBox.xcworkspace -scheme DramaBox -destination 'platform=iOS,name=StoryMatrix 06 iPhone' clean build -verbose > culprits.txt
```
```swift
grep '^[0-9]\+\.[0-9]\+ms' culprits.txt | sort -nr > sorted_filtered_culprits.txt
```

最终得到这样一个按编译时间排序的列表

![函数编译时间.jpeg](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZARBYY538qLX/img/0fd4681c-240d-43c7-8fa8-02bb5eadcc9d.jpeg)

根据结果来看大概有6个文件的编译时间明显过长，其中3个达到4.5秒以上需要重点关注。

## 优化方向

1.  if let替代??运算符
    
2.  数组append替代+拼接
    
3.  if else替代三元运算符
    
4.  去除没有必要的类型强制转换
    
5.  过长的字面量数字计算拆分
    

## 项目中的实际优化操作

**情况1：**

下面这个函数的编译耗时在5秒左右，主要是cellWidth计算这一行导致的

*   `fitScale` 返回 `CGFloat`
    
*   `ceil` 接受 `Double` 参数，触发隐式转换（CGFloat → Double）
    
*   乘法操作涉及 `Double` 和 `Int` 的混合运算，触发二次类型推断
    

```swift
    // 角标外显版本分类的每个item的最大高度
    var corverVersionCategoryBookItemMaxHeight: CGFloat {
        let cellWidth = (screenWidth - ceil(fitScale(8)) * 2 - ceil(fitScale(16)) * 2) / 3
        let isOneLine = UILabel.isOneLineLabel(width: cellWidth, font: .regularFont(13), text: bookName)
        let showMark = commonCanShowMark(markMaxWidth: cellWidth)
        if isOneLine {
            return fitScale(145) + fitScale(4) + fitScale(16) + fitScale(showMark ? 21 : 0) + fitScale(16)
        } else {
            return fitScale(145) + fitScale(4) + fitScale(32) + fitScale(showMark ? 21 : 0) + fitScale(16)
        }
    }
```

解决方法是提前将值转换为CGFloat类型，避免隐式转换和二次类型推断叠加情况的发生。

优化后编译时间为18毫秒，减少99.6%。

```swift
    // 角标外显版本分类的每个item的最大高度
    var corverVersionCategoryBookItemMaxHeight: CGFloat {
        // 提前转换所有值为 CGFloat
        let scaled8 = CGFloat(ceil(fitScale(8)))
        let scaled16 = CGFloat(ceil(fitScale(16)))
        let totalSpacing = (scaled8 + scaled16) * 2
        let cellWidth = (screenWidth - totalSpacing) / 3
    }
```

**情况2：**

下面这个函数的编译耗时在800毫秒左右，主要是balance计算这一行导致的

*   4 层可选链解包（`userInfoModel?.coins`, `DRBAppInfo.coins` 等）
    
*   2 次 `CGFloat` 转换需要类型推断
    

```swift
    var userInfoModel: DRBUserModel? {
        didSet {
            walletTitleLabel.text = Loc.shared().str_my_wallet()
            topupBtn.setTitle(Loc.shared().str_top_up(), for: .normal)
            let balance = CGFloat(userInfoModel?.coins ?? (DRBAppInfo.coins ?? 0)) + CGFloat(userInfoModel?.bonus ?? (DRBAppInfo.bonus ?? 0))
            coinsNumLabel.text = String(Int(balance))
            updateTopupViewWidth()
        }
    }
```

解决方法是将可选链解包分开并统一进行类型转换。

优化后编译时间为5毫秒，减少99.3%。

```swift
    var userInfoModel: DRBUserModel? {
        didSet {
            walletTitleLabel.text = Loc.shared().str_my_wallet()
            topupBtn.setTitle(Loc.shared().str_top_up(), for: .normal)
            let appCoins = DRBAppInfo.coins ?? 0
            let appBonus = DRBAppInfo.bonus ?? 0
            let userCoins = userInfoModel?.coins ?? appCoins
            let userBonus = userInfoModel?.bonus ?? appBonus
            let balance = CGFloat(userCoins + userBonus)
            coinsNumLabel.text = String(Int(balance))
            updateTopupViewWidth()
        }
    }
```

**情况3：**

下面这个函数的编译耗时在700毫秒左右，主要是计算视频地址时效这一行导致的

Double(24 \* 3600 - 10 \* 60)计算之后进行类型强制转换

```swift
    func checkAndRequestData() {
        // 接口下发的视频地址有时效（24小时），当前时间减去最后一次请求的返回时间大于23时50分就刷新数据
        if let ts = lastRequestTime, Date().timeIntervalSince1970 - ts > Double(24 * 3600 - 10 * 60) {
            pageNumber = 1
            requestData(page: requestPage + 1)
            return
        }
        // 显示foryou页面时，内存数据videoList为空，请求数据
        if videoList.isEmpty, pushBookId == nil {
            pageNumber = 1
            requestData(page: requestPage + 1)
        }
    }
```

解决方法是提前计算并指定类型。

优化后编译时间为100毫秒，减少85.7%。

```swift
    func checkAndRequestData() {
        // 接口下发的视频地址有时效（24小时），当前时间减去最后一次请求的返回时间大于23时50分就刷新数据
        let threshold: Double = 24 * 3600 - 10 * 60
        if let ts = lastRequestTime, Date().timeIntervalSince1970 - ts > threshold {
            pageNumber = 1
            requestData(page: requestPage + 1)
            return
        }
    }
```

## 最终结果

在设备是2023年M2 Max芯片的Mac Studio情况下，优化前总编译时间在82秒左右，优化后总编译时间在65秒左右，大概减少总编译时间的20%。

**资料：**

[Regarding Swift build time optimizations](https://medium.com/p/fc92cdd91e31)

[Swift build time optimizations — Part 2](https://medium.com/swift-programming/swift-build-time-optimizations-part-2-37b0a7514cbe)

[Profiling your Swift compilation times](https://irace.me/swift-profiling)