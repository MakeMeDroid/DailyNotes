动态settings区别于位于Server配置文件中的配置项，可以通过如下方式设置：

1. 在`users.xml`配置文件中的`<profiles>`标签中配置；
2. 在ClickHouse客户端交互式模式下通过语句`SET setting=value`设置，或者在使用HTTP协议时在session中设置（需要指定`session_id`）；
3. 查询时设置。如果使用ClickHouse客户端的非交互模式，则使用命令行参数`--setting=value`设置；如果通过HTTP接口，则通过`URL?setting_1=value&setting_2=value`设置。



按以上设置顺序，如果有重复设置，则后面的会覆盖前面的。



## 自定义配置项（Custom Settings）

除以系统预定义的配置项之外，用户可以自定义配置项。（其实这个功能应该叫自定义变量）

自定义配置项都会有个固定的前缀，这个前缀在系统配置文件中定义：

```xml
<custom_settings_prefixes>custom_</custom_settings_prefixes>
```

然后按如下方式自定义配置项：

```
SET custom_a = 123;
```

可以通过`getSetting()`函数获取自定义配置项的值：

