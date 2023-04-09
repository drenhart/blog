# 修改Aab内容并重新打包

### 工具

- [bundleTool](https://github.com/google/bundletool/releases)
- [jarSigner](https://docs.oracle.com/javase/7/docs/technotes/tools/windows/jarsigner.html)
- [aapt2](https://developer.android.com/studio/command-line/aapt2?hl=zh-cn)
- [apktool](https://ibotpeaches.github.io/Apktool/)
- android.jar

### 运行环境

- Windows

### IDE

- Visual Studio

### 全流程

1.aab->apks

```shell
java -jar bunletoolPath build-apks --bundle=aabFileName.aab --output=newApks.apks --mode=universal --ks jksFilePath.jks --ks-pass pass:jksFilePassword --ks-key-alias aliasName
```

 2.apks->apk

```
newApks.apks后缀名改成zip, 解压得到universal.apk
```

3.解压成decompile_apk文件夹

```
apktool d .\universal.apk -s -o decompile_apk -f
```

4.修改内容

```
修改app名字 启动图等
```

5.编译res文件夹得到res.zip

```
aapt2 compile --dir decompile_apk\res -o res.zip
```

> 如果遇到error: resource 'drawable/$avd_hide_password__0' has invalid entry name'错误,删掉public.xml中$avd开头的行

6.得到base.zip

```
aapt2 link --proto-format -o base.zip -I android.jar --manifest decompile_apk\AndroidManifest.xml --min-sdk-version 21 --target-sdk-version 32 --version-code 1 --version-name 1.0 -R res.zip --auto-add-overlay
```

7.解压base.zip到base文件夹

```
目录结构应该是这样
base/
/AndroidManifest.xml
/res
/resources.pb
```

8.切换工作路径到base文件夹

9.移动文件到base文件夹

```
创建一个manifest文件夹,然后将base/AndroidManifest.xml移动到manifest文件夹中
拷贝decompile_apk/assets文件夹到base/assets
拷贝decompile_apk/lib文件夹到base/lib
拷贝decompile_apk/unknown文件夹中所有内容到base/root
拷贝decompile_apk/kotlin文件夹到base/root/kotlin
新建dex文件夹 把所有.dex文件复制到里面
```

```
在这之后目录结构应该是这样
base/
/assets
/dex
/lib
/manifest
/res
/root
/resources.pb
```

10.压缩base文件夹

```
jar cMf base.zip manifest dex res root lib assets resources.pb
```

11.用bundleTool将base.zip转换成base.aab

```
java -jar bundletoolFilePath build-bundle --modules=base.zip --output=base.aab
```

12.签名

```
jarsigner -verbose -sigalg SHA256withRSA -digestalg SHA-256 -keystore jksFilePath -storepass jksFilePassword -signedjar signed.aab .\base.aab aliasName
```

13.验证签名

```
jarsigner -verify -verbose .\base.aab
```

### 可能的错误

- 如果运行起来闪退并报Module with the Main dispatcher is missing错误, 请在Proguard 规则文件中添加如下代码.

```
// 不混淆coroutines
-keep class kotlinx.coroutines.** { *; }
```

### 代码

https://github.com/drenhart/AndroidBatchPackage

### 参考资料

- [命令行工具](https://developer.android.com/studio/command-line?hl=zh-cn)

- [Convert APK TO AAB File](https://community.niotron.com/t/convert-apk-to-aab-file/9684)