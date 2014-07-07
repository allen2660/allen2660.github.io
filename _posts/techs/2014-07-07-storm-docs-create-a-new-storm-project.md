---
layout: default
title:  storm-rationale
---


本文为Storm官方文档[Creating-a-new-Storm-project](http://storm.incubator.apache.org/documentation/Creating-a-new-Storm-project.html)的读书笔记

本页面描述了如何创建一个Storm project以供开发。包括如下步骤：

+ 将Storm jar包加进classpath。
+ 如果使用多语言，将多语言目录加进classpath。

按照下列步骤，将[storm-starter](http://github.com/nathanmarz/storm-starter)集成进eclipse。

## 将Storm jar包加进classpath。

你需要将Storm jar包加进classpath，来开发Storm 拓扑。强烈推荐使用[Maven](http://storm.incubator.apache.org/documentation/Maven.html)。[这里](https://github.com/nathanmarz/storm-starter/blob/master/m2-pom.xml)是如何设置Storm的pom.xml例子。如果不使用Maven，那么就需要把Storm release中的jar包加进classpath。

storm-starter使用[Leiningen](http://github.com/technomancy/leiningen)来做编译和依赖解决。你可以通过运行[这个脚本](https://raw.github.com/technomancy/leiningen/stable/bin/lein)安装Leiningen，将其放到PATH中，设为可执行。想要获得Storm的依赖，只需要在项目根目录运行`lein deps`。

想要在eclipse中设置classpath，创建一个新的Java项目，将src/jvm/目录设为src path，并且确保lib/和lib/dev/下所有的jar包都在Reference Libraries中。


## 多语言目录加入classpath。

如果你使用非Java语言实现Spout或Bolt，这些实现需要放在multilang/resources/目录下。为了让Storm在local mode下找到这些文件，resources/ 目录需要在classpath中。你可以在eclipse中通过将resources/设为src path来实现这点。你可能也需要将multilang/resources/放进src dir。

关于使用其他语言编写拓扑，可见[Using non-JVM languages with Storm](http://storm.incubator.apache.org/documentation/Using-non-JVM-languages-with-Storm.html)。

为了测试一切在eclipse中都ok，你现在应该可以运行WordCountTopology.java 文件。你可以在console中看到信息被提交。