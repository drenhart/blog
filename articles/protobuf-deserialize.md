## resource.pb和dependencies.pb文件的解析

### pb文件是什么

> Protocol Buffers（简称 Protobuf）是一种轻量级、高效的数据序列化格式和接口定义语言（IDL）。它由 Google 开发，并于2008年开源发布。Protobuf 可以用于在不同平台和语言之间进行数据交换和通信，特别适用于高性能和可扩展性要求较高的系统。

简单来说就是和JSON作用差不多，不过在没有解析的情况下是不可读的。

### 对应proto文件的获取

pb文件需要对应的proto文件才可以解析。

resource.pb的对应proto文件在aapt2中。aapt2的代码开源在GoogleSource。

[Resources.proto和Configuration.proto](https://android.googlesource.com/platform/frameworks/base.git/+/refs/heads/master/tools/aapt2)

dependencies.pb的对应proto文件在bundletool中。bundletool的代码开源在GitHub。

[dependencies.proto](https://github.com/google/bundletool/tree/master/src/test)

### 对应类生成

protobuf的文档地址在这里 [地址](https://protobuf.dev/)。

点击右下角Download and install安装命令行工具。

这里有常用语言的命令行工具使用教程 [地址](https://protobuf.dev/getting-started/csharptutorial/)。

按照教程生成对应类，C#语言的具体例子如下：

```c#
// 生成Resources.cs
protoc -I=C:\Users\soumi\Desktop\tempfolder --csharp_out=C:\Users\soumi\Desktop\tempfolder C:\Users\soumi\Desktop\tempfolder\Resources.proto
// 生成Configuration.cs
protoc -I=C:\Users\soumi\Desktop\tempfolder --csharp_out=C:\Users\soumi\Desktop\tempfolder C:\Users\soumi\Desktop\tempfolder\Configuration.proto
// 生成Appdependencies.cs
protoc -I=C:\Users\soumi\Desktop\tempfolder --csharp_out=C:\Users\soumi\Desktop\tempfolder C:\Users\soumi\Desktop\tempfolder\app_dependencies.proto
```

### 转化成JSON和修改

>  将resources.pb转化成json并输出

```c#
	static void pbToJson(string pbPath, string outputPath) {
        ResourceTable resource;
        using (var file = File.OpenRead(pbPath)) {
            resource = ResourceTable.Parser.ParseFrom(file);
        }
        string str = JsonConvert.SerializeObject(resource, Formatting.Indented);
        File.WriteAllText(outputPath, str);
    }
```

> 修改resources.pb中的app_name属性

```c#
    static void ChangeAppName(string newName) {
        var resourcePbFile = Path.Combine(Directory.GetCurrentDirectory(), "pb", "resources.pb");
        ResourceTable resource;
        using (var file = File.OpenRead(resourcePbFile)) {
            resource = ResourceTable.Parser.ParseFrom(file);
        }

        foreach (var p in resource.Package) {
            var appNameEntry = p.Type.First(t => t.Name == "string")
                                .Entry.First(e => e.Name == "app_name");
            foreach (var e in appNameEntry.ConfigValue) {
                if (e.Value.Item.Str?.Value != null) {
                    e.Value.Item.Str.Value = newName;
                }
            }
        }

        File.Delete(resourcePbFile);

        using (var outfile = File.Create(resourcePbFile)) {
            resource.WriteTo(outfile);
        }
    }
```

> 将dependencies.pb转化成json并输出

```c#
    static void pbToJsonDependencies(string pbPath, string outputPath) {
        AppDependencies dependencies;
        using (var file = File.OpenRead(pbPath)) {
            dependencies = AppDependencies.Parser.ParseFrom(file);
        }
        string str = JsonConvert.SerializeObject(dependencies, Formatting.Indented);
        File.WriteAllText(outputPath, str);
    }
```

### 代码

https://github.com/drenhart/ProtoBufDeserialize

