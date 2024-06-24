# ExternalAPI测试策略分享



## 背景

ExternalAPI代码库升级。为了保证升级的质量我们采用了一些方法来对比升级前后的效果。在此分享一下。



## 相关内容介绍

这里是一些需要了解的ExternalAPI的相关内容。

##### 1.EventStore

提到ExternalAPI就不得不说一下EventStore，EventStore的主要作用就是存储Event。这里有几个基本概念：
EventStore可以简单理解为一个数据库，而Event就是一条条数据内容，
Stream类似于一张表，负责存放不同的Event。

##### 2. External API的作用：

ExternalAPI主要包含两部分内容。
第一部分是Feedgenerator Service，这是一个worker，负责将我们系统产生的Event（称为Domain Event），按一定逻辑转化为给其他系统使用的Event（Platform Event）。

第二部分是ExternalAPI Service，提供对外接口，根据需求将Feedgenerator产生的Platform Event以xml的形式提供给外部系统。

![b517a608-59c6-4e34-b20f-da389dd6b760](C:\Users\jiaxi\Desktop\external\b517a608-59c6-4e34-b20f-da389dd6b760.png)

## 测试策略



有了上面的了解后，可以开始考虑我们的测试策略了。
我们的目的是在升级后使ExternalAPI可以给外部系统提供和之前一样的内容。

为了达到这个目的，首先需要Feedgenerator转化Domain Event到Platform Event的过程正确。然后需要External API Service 以XML呈现Platform Event一致。

今天我们重点来看第一步

### Feedgenerator测试

总体流程如下：

1.收集尽可能多类型的Domain Event，每个类型取一个

2.分别在在Dotnet6版本和Framework版本下的Feedgenerator将这些Event转化为Platform Event

3.对比2和3中产生的platform Event的结果。



在第一步收集过程中为了尽量模拟生产环境的数据我们最初选择在Test获取Event Store，因为Test环境的数据从Production环境而来。在中途发现QA环境的Event类型比Test环境丰富一些所以最终是从QA环境获取Event。

在第二步中我们选择将第一个阶段中产生的Event存到一个新的Stream里，因为Feedgenerator中，有不同类型的Job来处理Event，我们只需再写一个Job从新生成的Stream中获取Event再进行处理即可。![Screenshot 2023-06-15 233009](C:\Users\jiaxi\Pictures\Screenshots\Screenshot 2023-06-15 233009.png)



##### 小坑1：

然而完成之后发现，Event上有一个叫StreamId的字段，这个字段表示Event处于哪个Stream。而且这个字段会被用于后续的业务逻辑。然而我们将stream存到了一个新的stream中，造成了这个值的改变。

只能转而采取新的方式：

![feedsolution](C:\Users\jiaxi\Desktop\external\feedsolution.png)

如图Original Solution和Current Solution所示。在获取到所有Event之后，直接将这些Event存到一个文件中，同时Feedgenerator将中读取Event的来源改成这个文件。这样就避免了StreamID的改变。

这里还有一个another solution要用到EventStore中另一个概念Projection，这里并未使用这种方式所以不作过多介绍。

##### 小坑2：

实际上我们收集到的Domain Event起初并不能被正确转化，因为在Feedgenerator中并不能处理所有的Domain Event，有些类型代码里就不支持，还有一些因为Event中缺失信息。这样的话我们就无法验证能被正常转化的Event。

Test环境的EventStore中Assignee和workRecord相关的Event数量在百万级，我们无法保证我们找到的那一些数据刚好能被处理。

所以在我们的工具代码中，也加入了一个新的逻辑。就是只有能处理的Event才会被我们收集。

```
private bool TryConvert(SimpleEntry simpleEntry)
{
    try
    {
        var platformEvents = eventHandlers
            .Where(h => h.CanHandle(simpleEntry.EventType))
            .Select(h => h.Handle(simpleEntry.Id))
            .Where(e => e.HasValue())
            .ToList();
        return platformEvents.Any();
    }
    catch (Exception e)
    {
        Console.WriteLine(e);
        return false;
    }
}
```

这样我们分别在Dotnet6和Framework下拿到了Feedgenerator产生的Platform Event，最后通过Json序列化后对比结果即可。



### 小结

我们这个测试的重点是Domain Event到Platform Event的转化，因为如果上线后这个逻辑发现了改变，我们很难去修复已经产生的错误的Event。通过工具可以按类型保证真实环境中存在的Event可以正常地被消费，与之前的行为保持一致。

