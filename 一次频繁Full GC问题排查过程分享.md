## 问题描述
* 应用收到频繁Full GC告警
## 问题排查
* 登录到对应机器上去，查看GC日志，发现YGC一分钟已经达到了15次，比Full GC还要频繁一些，其中Full GC平均10分钟超过了4次，如下图<br>
![](https://raw.githubusercontent.com/wycm/md-image/master/2018-11-05/1.png)
* 使用jstat -gcutil 5280 1000查看实时GC情况，年老代采用的是CMS收集器，发现触发Full GC的原因是年老代占用空间达到指定阈值70%（-XX:CMSInitiatingOccupancyFraction=70）。
* 这时候猜测是某个地方频繁创建对象导致，通过`jmap -dump:format=b,file=temp.dump 5280` dump文件，然后下载到本地通过jvisualvm分析对象的引用链的方式来定位具体频繁创建对象的地方，dump文件下载下来有5G多，整个导入过程都花了10多分钟。想查看所占空间较多对象的引用链，直接OOM了，dump对象太大了。这时候就换了种思路，查看占用空间比较大的一系列对象，看能不能找出什么端倪。占用空间最大的几类对象如下图<br>
![](https://raw.githubusercontent.com/wycm/md-image/master/2018-11-05/3.png)
发现排第一的`chart[]`对象里面，存在一些metrics监控的具体指标的相关内容，排第二的`io.prometheus.client.Collector$MetricFamilySample$Sample`和排第9和第13对象都是spring boot中metrics指标监控相关的对象，所以此时怀疑metrics监控的某个地方在频繁创建对象，首先考虑的是否因为metrics指标太多导致的，于是登录线上机器`curl localhost:8080/mertrics > metrics.log`，发现响应内容有50多M，参考其他相关的正常应用，指标总共内容也就10多M左右，打开指标内容发现了很多类似如下图的指标<br>
![](https://raw.githubusercontent.com/wycm/md-image/master/2018-11-05/4.png)
 <br>看到了这里已经可以确定代码中上报这个指标是存在问题的，并没有达到我们想要的效果，所以也怀疑也是这个地方导致的Full GC频繁。
## 问题初步解决
* 由于这个指标也无关紧要，初步解决方案就把上报该指标的代码给干掉。上线后看下Full GC问题是否会得到改善，果然，上线后Full GC告警问题已经解决。
## 初步解决后的思考，为什么会有这个问题？
* 外部监控系统，每25s会来调用metrics这个接口，这个接口会把所有的metrics指标转成字符串然后作为http响应内容响应。监控每来调用一次就会产生一个50多M的字符串，导致了频繁YGC，进而导致了晋升至年老代的对象也多了起来，最终年老代内存占用达到70%触发了Full GC。  
## 根源问题重现
* 此处采用metrics的作用：统计线程池执行各类任务的数量。为了简化代码，用一个map来统计，重现代码如下
```
    import java.util.Map;
    import java.util.concurrent.*;
    import java.util.concurrent.atomic.AtomicInteger;
    
    /**
     * 线程池通过submit方式提交任务，会把Runnable封装成FutureTask。
     * 直接导致了Runnable重写的toString方法在afterExecute统计的时候没有起到我们想要的作用，
     * 最终导致几乎每一个任务（除非hashCode相同）就按照一类任务进行统计。所以这个metricsMap会越来越大，调用metrics接口的时候，会把该map转成一个字符返回
     */
    public class GCTest {
        /**
         * 统计各类任务已经执行的数量
         * 此处为了简化代码，只用map来代替metrics统计
         */
        private static Map<String, AtomicInteger> metricsMap = new ConcurrentHashMap<>();
    
        public static void  main(String[] args) throws InterruptedException {
            ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(10, 10, 0, TimeUnit.SECONDS, new LinkedBlockingQueue<>()){
                /**
                 * 统计各类任务执行的数量
                 * @param r
                 * @param t
                 */
                @Override
                protected void afterExecute(Runnable r, Throwable t) {
                    super.afterExecute(r, t);
                    metricsMap.compute(r.toString(), (s, atomicInteger) ->
                            new AtomicInteger(atomicInteger == null ? 0 : atomicInteger.incrementAndGet()));
                }
            };
            /**
             * 源源不断的任务添加进线程池被执行
             */
            for (int i =0; i < 1000; i++) {
                threadPoolExecutor.submit(new SimpleRunnable());
            }
            Thread.sleep(1000 * 2);
            System.out.println(metricsMap);
            threadPoolExecutor.shutdownNow();
        }
        static class SimpleRunnable implements Runnable{
    
            @Override
            public void run() {
                System.out.println("SimpleRunnable execute success");
            }
            /**
             * 重写toString用于统计任务数
             * @return
             */
            @Override
            public String toString(){
                return this.getClass().getSimpleName();
            }
        }
    }
```

## 最终解决
* 可以把submit改成execute即可
## 总结
* 以上重显代码可以看出metricsMap中的元素是会越来越多的。如果就这样下去，最终的结果也会出现OOM。
* 根本原因还是对ThreadPoolExecutor不够熟悉，所以出现了这次问题。
* 个人感觉Full GC类问题是比较让人头疼的。这些问题并不会想代码语法问题一样，ide会提示我们具体错在哪里，我们只要修改对应地方基本都能解决。造成Full GC频繁的原因也有很多，比如可能是jvm参数设置不合理、Metaspace空间触发、频繁创建对象触发等等。
* 如果确定了是频繁创建对象导致，那么接下来的目的就是确定频繁创建对象的对应代码处，这时候可以选择通过dump线上堆栈，然后下载到本地。选择一些可视化分析工具进行分析。最终定位到出问题的代码处，然后解决问题。