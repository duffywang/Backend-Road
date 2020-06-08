## 什么是jar包冲突

jar包冲突是指项目中依赖相同的groupId和artifactId不同的version的jar包。

具体来说可分为两种情况：1）应用程序依赖的同一个Jar包出现了多个不同版本，并选择了错误的版本而导致JVM加载不到需要的类或加载了错误版本的类，为了叙述的方便，称之为第一类Jar包冲突问题；2）同样的类（类的全限定名完全一样）出现在多个不同的依赖Jar包中，即该类有多个版本，并由于Jar包加载的先后顺序导致JVM加载了错误版本的类，称之为第二类Jar包问题。这两种情况所导致的结果其实是一样的，都会使应用程序加载不到正确的类，那其行为自然会跟预期不一致了。

## jar包冲突产生原因

举个例子，图一种依赖关系简化为 A-B-C-D1(guava16.0.1)   ，图二中依赖关系简化为E-F-D2(guava 19.0)

当pom.xml文件中引入A、E两个依赖后，根据Maven传递依赖的原则，D1、D2都会被引入，而D1、D2是同一个依赖D的不同版本

当调用D2中某个方法method1()，如果D1中由于版本较低没有这个方法，这样在JVM加载A中的D1依赖时，是找不到method1方法的，则会报NoSuchMethodError

## 常见异常报错

java.lang.NoSuchMethodException：即找不到特定方法，第一类冲突和第二类冲突都可能导致该问题——加载的类不正确。若是第一类冲突，则是由于错误版本的Jar包与所需要版本的Jar包中的类接口不一致导致，例如antlr-2.7.2.jar升级到antlr-2.7.6.Jar时，接口antlr.collections.AST.getLine()发生变动，当maven仲裁选择了错误版本而加载了错误版本的类AST，则会导致该异常；若是第二类冲突，则是由于不同Jar包含有的同名类接口不一致导致，典型的案例：Apache的commons-lang包，2.x升级到3.x时，包名直接从commons-lang改为commons-lang3，部分接口也有所改动，由于包名不同和传递性依赖，经常会出现两种Jar包同时在classpath下，org.apache.commons.lang.StringUtils.isBlank就是其中有差异的接口之一，由于Jar包的加载顺序，导致加载了错误版本的StringUtils类，就可能出现NoSuchMethodError异常

java.lang.ClassNotFoundException：即java类找不到。这类典型异常通常是由于，没有在依赖管理中声明版本，maven的仲裁的时候选取了错误的版本，而这个版本缺少我们需要的某个class而导致该错误。例如httpclient-4.4.jar升级到httpclient-4.36.jar时，类org.apache.http.conn.ssl.NoopHostnameVerifier被去掉了，如果此时我们本来需要的是4.4版本，且用到了NoopHostnameVerifier这个类，而maven仲裁时选择了4.6，则会导致ClassNotFoundException异常

java.lang.NoSuchFieldError

java.lang.NoClassDefFoundError

java.lang.LinkageError

## 查找冲突

1、Maven默认仲裁机制

最短路径优先，即在工程的依赖树上，深度越浅，越被优先选择：

Maven 面对 D1 和 D2 时，会默认选择最短路径的那个 jar 包，即 D2。E->F->D2 比 A->B->C->D1 路径短 1。

最先声明优先：

如果路径一样的话，如： A->B->C1, E->F->C2 ，两个依赖路径长度都是 2，那么就选择pom文件中最先声明。

2、使用第三方插件

借助IDEA工具分析冲突包 ，如Fusion Analyzer  ，点击Conflicts查看冲突的jar包

3、mvn dependency:tree

以上命令可以看到依赖具体的jar包名，mvn dependency:tree --> tree.txt命令导出依赖树到txt文件，便于查看；之后用cat tree.txt | grep log4j查找tree.txt里面的log4j；如下所示：+-：表示下方还有兄弟节点，\-表示是该层次的最后一个节点；

 mvn dependency:tree -Dverbose延伸阅读：http://ian.wang/106.htm

4、在在项目启动时加上VM参数 -verbose:class，项目启动时会把所有加载的jar都打印出来



## 解决方案

1、通过用<excludes>排除不需要的版本，但这种做法带来的问题是每次引入带有传递性依赖的Jar包时，都需要一一进行排除，非常麻烦。

2、明确应用所需的jar包版本，显式去引入jar包，而不是交给maven推断（推荐）

比如需要依赖guava 19.0版本中的一个犯法，guava18.0中没有，则需要显示的声明guava 版本为19.0

maven提供了集中管理依赖信息的机制，即依赖管理元素<dependencyManagement>，对依赖Jar包进行统一版本管理，一劳永逸。通常的做法是，在parent模块的pom文件中尽可能地声明所有相关依赖Jar包的版本，并在子pom中简单引用该构件即可

参考:
https://blog.csdn.net/noaman_wgs/article/details/81137893

https://www.jianshu.com/p/100439269148