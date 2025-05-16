---
title: 一次非常偶发的Dubbo线程全部BLOCKED问题排查
tags: []
categories: []
abbrlink: d60dc299
date: 2025-05-16 09:31:47
---
# 背景

2025-04-29迭代需求发布，某个应用按照正常发布流程发布后5分钟，出现大量的dubbo超时，快速排查发现该应用的其中一个POD，Dubbo线程被打满，其他POD正常

之前其他应用也出现过同样的问题，因为是线上环境且无剔除dubbo-provider手段，为快速恢复应用，紧急重启该POD，应用恢复正常。因没有保存快照且不太好复现，这个问题就先搁置了。

2025-05-07 预发环境出现了同样的问题，终于可以保存各种信息了解详细原因了

快速保存以下信息，用于排查问题

1. 保存线程信息
2. 保存应用堆栈信息

# 分析

从堆栈信息看，所有的Dubbo线程都在Blocked，同时指向

```log
    at com.fasterxml.jackson.databind.deser.DeserializerCache._createAndCacheValueDeserializer(DeserializerCache.java:230)
       local variable: com.fasterxml.jackson.databind.deser.DeserializerCache#2
       local variable: com.fasterxml.jackson.databind.deser.DefaultDeserializationContext$Impl#14
       local variable: com.fasterxml.jackson.databind.deser.BeanDeserializerFactory#3
       local variable: com.fasterxml.jackson.databind.type.SimpleType#36
       local variable: java.util.HashMap#8184
```

第一反应jackson的问题？按说这种三方包不应该出现这种问题，又考虑到我们的应用引入的版本可能比较老，先去github搜一下对应的信息

找到了这个问题[Use ReentrantLock instead of synchronized in DeserializerCache to avoid deadlock on pinning #443](https://github.com/FasterXML/jackson-databind/pull/4430)

这个问题是关于虚拟线程下的synchronized死锁，我们用的JDK11，所以不存在这种问题，问题暂时搁置了。

因为应用刚启动用visualVM分析内存也没有发现明显问题，原因越来越棘手了，还是要重新分析线程信息

既然全部是被_createAndCacheValueDeserializer这个方法阻塞的，那就找这个方法目前哪些线程在调用，在诸多线程中终于找到一个WAITING线程在调用这个方法

```log
"DubboServerHandler-xx.xx.xx.xx:xxx-thread-9" daemon prio=5 tid=505 WAITING
    at java.lang.Object.wait(Native Method)
    at java.util.concurrent.ForkJoinTask.externalAwaitDone(ForkJoinTask.java:330)
       local variable: java.util.stream.MatchOps$MatchTask#1
       local variable: java.util.stream.MatchOps$MatchTask#1
    at java.util.concurrent.ForkJoinTask.doInvoke(ForkJoinTask.java:412)
       local variable: java.util.stream.MatchOps$MatchTask#1
       local variable: org.apache.dubbo.common.threadlocal.InternalThread#6
    at java.util.concurrent.ForkJoinTask.invoke(ForkJoinTask.java:736)
       local variable: java.util.stream.MatchOps$MatchTask#1
    at java.util.stream.MatchOps$MatchOp.evaluateParallel(MatchOps.java:242)
       local variable: java.util.stream.MatchOps$MatchOp#1
       local variable: java.util.stream.ReferencePipeline$Head#2
       local variable: java.util.Spliterators$IteratorSpliterator#1
    at java.util.stream.MatchOps$MatchOp.evaluateParallel(MatchOps.java:196)
       local variable: java.util.stream.MatchOps$MatchOp#1
       local variable: java.util.stream.ReferencePipeline$Head#2
       local variable: java.util.Spliterators$IteratorSpliterator#1
    at java.util.stream.AbstractPipeline.evaluate(AbstractPipeline.java:233)
    at java.util.stream.ReferencePipeline.anyMatch(ReferencePipeline.java:528)
       local variable: java.util.stream.ReferencePipeline$Head#2
       local variable: com.xxx.kit.json.deserializer.PostDeserializeDelegatingDeserializer$1$$Lambda$1803#1
    at com.xxx.kit.json.deserializer.PostDeserializeDelegatingDeserializer$1.modifyDeserializer(PostDeserializeDelegatingDeserializer.java:39)
       local variable: com.xxx.kit.json.deserializer.PostDeserializeDelegatingDeserializer$1#1
       local variable: com.fasterxml.jackson.databind.DeserializationConfig#2
       local variable: com.fasterxml.jackson.databind.introspect.BasicBeanDescription#6
       local variable: com.fasterxml.jackson.databind.deser.BeanDeserializer#3
       local variable: com.fasterxml.jackson.databind.introspect.AnnotatedClass#8
    at com.fasterxml.jackson.databind.deser.BeanDeserializerFactory.buildBeanDeserializer(BeanDeserializerFactory.java:301)
       local variable: com.fasterxml.jackson.databind.deser.BeanDeserializerFactory#3
       local variable: com.fasterxml.jackson.databind.deser.DefaultDeserializationContext$Impl#25
       local variable: com.fasterxml.jackson.databind.type.SimpleType#36
       local variable: com.fasterxml.jackson.databind.introspect.BasicBeanDescription#6
       local variable: com.fasterxml.jackson.databind.deser.std.StdValueInstantiator#3
       local variable: com.fasterxml.jackson.databind.deser.BeanDeserializerBuilder#1
       local variable: com.fasterxml.jackson.databind.DeserializationConfig#2
       local variable: com.fasterxml.jackson.databind.deser.BeanDeserializer#3
       local variable: com.fasterxml.jackson.databind.util.ArrayIterator#1
       local variable: com.xxx.kit.json.deserializer.PostDeserializeDelegatingDeserializer$1#1
    at com.fasterxml.jackson.databind.deser.BeanDeserializerFactory.createBeanDeserializer(BeanDeserializerFactory.java:151)
       local variable: com.fasterxml.jackson.databind.deser.BeanDeserializerFactory#3
       local variable: com.fasterxml.jackson.databind.deser.DefaultDeserializationContext$Impl#25
       local variable: com.fasterxml.jackson.databind.type.SimpleType#36
       local variable: com.fasterxml.jackson.databind.introspect.BasicBeanDescription#6
       local variable: com.fasterxml.jackson.databind.DeserializationConfig#2
    at com.fasterxml.jackson.databind.deser.DeserializerCache._createDeserializer2(DeserializerCache.java:415)
       local variable: com.fasterxml.jackson.databind.deser.DeserializerCache#2
       local variable: com.fasterxml.jackson.databind.deser.DefaultDeserializationContext$Impl#25
       local variable: com.fasterxml.jackson.databind.deser.BeanDeserializerFactory#3
       local variable: com.fasterxml.jackson.databind.type.SimpleType#36
       local variable: com.fasterxml.jackson.databind.introspect.BasicBeanDescription#6
       local variable: com.fasterxml.jackson.databind.DeserializationConfig#2
    at com.fasterxml.jackson.databind.deser.DeserializerCache._createDeserializer(DeserializerCache.java:350)
       local variable: com.fasterxml.jackson.databind.deser.DeserializerCache#2
       local variable: com.fasterxml.jackson.databind.deser.DefaultDeserializationContext$Impl#25
       local variable: com.fasterxml.jackson.databind.deser.BeanDeserializerFactory#3
       local variable: com.fasterxml.jackson.databind.type.SimpleType#36
       local variable: com.fasterxml.jackson.databind.DeserializationConfig#2
       local variable: com.fasterxml.jackson.databind.introspect.BasicBeanDescription#6
       local variable: com.fasterxml.jackson.databind.type.SimpleType#36
    at com.fasterxml.jackson.databind.deser.DeserializerCache._createAndCache2(DeserializerCache.java:264)
       local variable: com.fasterxml.jackson.databind.deser.DeserializerCache#2
       local variable: com.fasterxml.jackson.databind.deser.DefaultDeserializationContext$Impl#25
       local variable: com.fasterxml.jackson.databind.deser.BeanDeserializerFactory#3
       local variable: com.fasterxml.jackson.databind.type.SimpleType#36
    at com.fasterxml.jackson.databind.deser.DeserializerCache._createAndCacheValueDeserializer(DeserializerCache.java:244)
       local variable: com.fasterxml.jackson.databind.deser.DeserializerCache#2
       local variable: com.fasterxml.jackson.databind.deser.DefaultDeserializationContext$Impl#25
       local variable: com.fasterxml.jackson.databind.deser.BeanDeserializerFactory#3
       local variable: com.fasterxml.jackson.databind.type.SimpleType#36
       local variable: java.util.HashMap#8184
```

可以看到这个方法在等待ForkJoinTask完成，里面可疑的就是公司kit包的代码，原始代码如下

```java
 public static Module getSimpleModule() {
        log.info("Registering JacksonPostHookDeserializer");
        return new SimpleModule().setDeserializerModifier(new BeanDeserializerModifier() {
            @Override
            public JsonDeserializer<?> modifyDeserializer(DeserializationConfig config,
                                                          BeanDescription beanDescription,
                                                          JsonDeserializer<?> originalDeserializer) {
                AnnotatedClass classInfo = beanDescription.getClassInfo();

                if (StreamSupport.stream(classInfo.memberMethods().spliterator(), true).
                        anyMatch(m -> m.hasAnnotation(PostDeserialize.class))) {
                    log.debug("BeanDescription {} ", classInfo);

                    return new PostDeserializeDelegatingDeserializer(originalDeserializer, beanDescription);
                }
                else {
                    return originalDeserializer;
                }
            }
        });
    }
```

可以看到是用并行流，查找一个方法注解

那问题就很明显了，就是这个并行流的问题，在并行流中，会一直等待所有任务结束，那就是有任务未结束，从代码看都是jdk自身的代码，不太可能出现问题，问题又陷入僵局...

一次发呆过程中，突然想到，有没有可能是并行流的线程池也满了，在等待可用线程？既然有新思路就开始排查，从线程信息看，有3个线程

![image](https://static.lianglianglee.com/2025/05/uWFL4sK.png)

跟踪源码`java.util.concurrent.ForkJoinPool#ForkJoinPool(byte)`发现，默认下`ForkJoinPool`线程是CPU-1，预发环境是4C，那就是ForkJoin全部线程都卡住了，没线程给并行流使用了，并行流就陷入到死循环等待中，同时Dubbo接口也用到jackson的序列化或者反序列化，所有的Dubbo线程也卡住了。

为什么ForkJoin线程卡住呢？

```java
"ForkJoinPool.commonPool-worker-3" daemon prio=5 tid=495 BLOCKED
    at com.fasterxml.jackson.databind.deser.DeserializerCache._createAndCacheValueDeserializer(DeserializerCache.java:230)
       local variable: com.fasterxml.jackson.databind.deser.DeserializerCache#2
       local variable: com.fasterxml.jackson.databind.deser.DefaultDeserializationContext$Impl#16
       local variable: com.fasterxml.jackson.databind.deser.BeanDeserializerFactory#3
       local variable: com.fasterxml.jackson.databind.type.SimpleType#36
       local variable: java.util.HashMap#8184
    at com.fasterxml.jackson.databind.deser.DeserializerCache.findValueDeserializer(DeserializerCache.java:142)
       local variable: com.fasterxml.jackson.databind.deser.DeserializerCache#2
       local variable: com.fasterxml.jackson.databind.deser.DefaultDeserializationContext$Impl#16
       local variable: com.fasterxml.jackson.databind.deser.BeanDeserializerFactory#3
       local variable: com.fasterxml.jackson.databind.type.SimpleType#36
    at com.fasterxml.jackson.databind.DeserializationContext.findRootValueDeserializer(DeserializationContext.java:654)
       local variable: com.fasterxml.jackson.databind.deser.DefaultDeserializationContext$Impl#16
       local variable: com.fasterxml.jackson.databind.type.SimpleType#36
    at com.fasterxml.jackson.databind.ObjectMapper._findRootDeserializer(ObjectMapper.java:4956)
       local variable: com.fasterxml.jackson.databind.ObjectMapper#2
       local variable: com.fasterxml.jackson.databind.deser.DefaultDeserializationContext$Impl#16
       local variable: com.fasterxml.jackson.databind.type.SimpleType#36
    at com.fasterxml.jackson.databind.ObjectMapper._convert(ObjectMapper.java:4537)
       local variable: com.fasterxml.jackson.databind.util.TokenBuffer$Parser#2
       local variable: com.fasterxml.jackson.databind.deser.DefaultDeserializationContext$Impl#16
    at com.fasterxml.jackson.databind.ObjectMapper.convertValue(ObjectMapper.java:4475)
    at com.xxx.kit.bean.impl.BeanServiceImpl.convert(BeanServiceImpl.java:60)
    at com.xxx.kit.bean.impl.JacksonImpl.convert(JacksonImpl.java:17)
    at com.xxx.common.util.BeanConvertUtil.convert(BeanConvertUtil.java:40)
       local variable: com.xxx.fulfill.basedata.biz.shipping.model.BaseShippingBO#2
       local variable: class com.xxx.fulfill.basedata.api.model.BaseShippingData
    at com.xxx.common.util.BeanConvertUtil.lambda$convert$0(BeanConvertUtil.java:80)
    at com.xxx.common.util.BeanConvertUtil$$Lambda$1906.apply(<unresolved string 0x0>)
    at java.util.stream.ReferencePipeline$3$1.accept(ReferencePipeline.java:195)
       local variable: java.util.stream.ReduceOps$3ReducingSink#2
    at java.util.ArrayList$ArrayListSpliterator.forEachRemaining(ArrayList.java:1655)
       local variable: java.util.ArrayList$ArrayListSpliterator#10
       local variable: java.util.stream.ReferencePipeline$3$1#2
       local variable: java.lang.Object[]#190278
    at java.util.stream.AbstractPipeline.copyInto(AbstractPipeline.java:484)
       local variable: java.util.stream.ReferencePipeline$3$1#2
    at java.util.stream.AbstractPipeline.wrapAndCopyInto(AbstractPipeline.java:474)
       local variable: java.util.stream.ReduceOps$3ReducingSink#2
    at java.util.stream.ReduceOps$ReduceTask.doLeaf(ReduceOps.java:952)
       local variable: java.util.stream.ReduceOps$ReduceTask#15
    at java.util.stream.ReduceOps$ReduceTask.doLeaf(ReduceOps.java:926)
       local variable: java.util.stream.ReduceOps$ReduceTask#15
    at java.util.stream.AbstractTask.compute(AbstractTask.java:327)
       local variable: java.util.stream.ReduceOps$ReduceTask#4
       local variable: java.util.ArrayList$ArrayListSpliterator#10
       local variable: java.util.stream.ReduceOps$ReduceTask#15
       local variable: java.util.stream.ReduceOps$ReduceTask#15
    at java.util.concurrent.CountedCompleter.exec(CountedCompleter.java:746)
       local variable: java.util.stream.ReduceOps$ReduceTask#4
    at java.util.concurrent.ForkJoinTask.doExec(ForkJoinTask.java:290)
       local variable: java.util.stream.ReduceOps$ReduceTask#4
       local variable: com.navercorp.pinpoint.bootstrap.interceptor.dynamic.SwitchInterceptor#420
    at java.util.concurrent.ForkJoinPool$WorkQueue.topLevelExec(ForkJoinPool.java:1020)
       local variable: java.util.concurrent.ForkJoinPool$WorkQueue#4
       local variable: java.util.stream.ReduceOps$ReduceTask#4
       local variable: java.util.concurrent.ForkJoinPool$WorkQueue#3
    at java.util.concurrent.ForkJoinPool.scan(ForkJoinPool.java:1656)
       local variable: java.util.concurrent.ForkJoinPool#1
       local variable: java.util.concurrent.ForkJoinPool$WorkQueue#4
       local variable: java.util.concurrent.ForkJoinPool$WorkQueue[]#1
       local variable: java.util.concurrent.ForkJoinPool$WorkQueue#3
       local variable: java.util.concurrent.ForkJoinTask[]#2
       local variable: java.util.stream.ReduceOps$ReduceTask#37
    at java.util.concurrent.ForkJoinPool.runWorker(ForkJoinPool.java:1594)
       local variable: java.util.concurrent.ForkJoinPool#1
       local variable: java.util.concurrent.ForkJoinPool$WorkQueue#4
    at java.util.concurrent.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:183)
       local variable: java.util.concurrent.ForkJoinWorkerThread#2
```

# 原因

问题原因浮出水面，虽然没有两个锁导致的锁冲突，但是出现了一个伪锁（并行流等待可用线程），导致类似死锁的场面。

* 所有Forkjoin线程在等待锁
* 拿到锁的任务在等待Forkjoin可用线程

# 解决方法

将自定义的`PostDeserializeDelegatingDeserializer`中的并行流改成当前线程执行，去掉这个伪锁