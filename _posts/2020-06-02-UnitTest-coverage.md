---
layout: post
title:  "单元测试覆盖率"
author: mew151
image: assets/images/202006/nature-venice-italy-rush-hour-1600x1200-wallpaper.jpg
---

##### 什么是单元测试覆盖率？
关于其定义，先来看一下维基百科上的一段描述：
>[代码覆盖（Code coverage）](https://zh.wikipedia.org/wiki/%E4%BB%A3%E7%A2%BC%E8%A6%86%E8%93%8B%E7%8E%87)是软件测试中的一种度量，描述程序中源代码被测试的比例和程度，所得比例称为代码覆盖率。

简单来理解，就是单元测试中代码执行量与代码总量之间的比率。

以一个最简单的例子来直观感受一下：

Service服务类：
```java
public class NumToStringServiceImpl implements NumToStringService {

    @Override
    public String num2Str(Integer i) {
        String str = "";
        switch (i) {
            case 1:
                str = "one";
                break;
            case 2:
                str = "two";
                break;
            default:
                str = "none";
        }
        return str;
    }
}
```

单元测试类：
```java
public class NumToStringServiceTest {

    @Autowired
    NumToStringService numToStringService;

    @Test
    void testNum2Str() {
        String str1 = numToStringService.num2Str(1);
        assertThat(str1, is("one"));
        String str2 = numToStringService.num2Str(2);
        assertThat(str2, is("two"));
    }
}
```

从上面的代码中能看出，单元测试方法`testNum2Str`能够覆盖到服务类`num2Str`方法的`case 1`和`case 2`两个分支，覆盖不到`default`分支。那么覆盖率就是`num2Str`方法`case 1`和`case 2`分支的代码量除以方法的总代码量。

##### 单元测试覆盖率框架

单元测试覆盖率常用的框架有**[JaCoCo](https://www.eclemma.org/jacoco/)**、**[EMMA](http://emma.sourceforge.net/)**和**[Cobertura](http://cobertura.sourceforge.net/)**。我们目前（在Jenkins中）使用的是JaCoCo。

JaCoCo可以统计的[指标](https://www.jacoco.org/jacoco/trunk/doc/counters.html)有：

1. **指令（C0 Coverage）**：JaCoCo计数的最小单元是单一的Java字节码指令。指令覆盖率提供了关于字节码执行数量、未执行数量的信息。
2. **分支（C1 Coverage）**：对所有的`if`和`switch`语句计算分支覆盖率。统计在方法中分支执行数量、未执行数量的信息。但要注意，异常处理不在此计算范围内。
3. **圈复杂度（Cyclomatic Complexity）**：对非抽象方法计算圈复杂度，并汇总类、包和组的（圈）复杂度。这个值可以做为单元测试用例是否完全覆盖的参考。
4. **行（Lines）**：一行可能包含一条或多条指令，如果至少有一条指令被执行了，那么该行就算作是被执行了。
5. **方法（Methods）**：每个非抽象方法至少包含一条指令。如果至少有一条指令被执行了，那么该方法就算作是被执行了。
6. **类（Classes）**：如果类中至少有一个方法被执行了，那么该类就算作是被执行了。

>注：个人认为，最需要关注的指标是**行**（Lines）和**分支**（C1 Coverage），其次是**方法**（Methods）和**类**（Classes），**指令**（C0 Coverage）和**圈复杂度**（Cyclomatic Complexity）可以不用关注，因为跟**行**（Lines）和**分支**（C1 Coverage）其实是差不多的，只不过多了一种参考维度。

##### 查看单元测试覆盖率
在IntelliJ IDEA中已经内置了JaCoCo插件，因此研发可以在本机运行单元测试来查看覆盖率：

1、点击IDE右上侧的"Edit Configurations..."：
![](/assets/images/202006/ideaJ-1.png)

2、在"Choose coverage runner"中选择JaCoCo：
![](/assets/images/202006/ideaJ-2.png)

3、点击"Run ... with Coverage"运行：
![](/assets/images/202006/ideaJ-3.png)

4、运行完成后会展示**分支**（C1 Coverage）、**行**（Lines）、**方法**（Methods）、**类**（Classes）这四个指标：
![](/assets/images/202006/ideaJ-4.png)

5、点击"Generate Coverage Report"可以生成一份html版的所有指标的报告：
![](/assets/images/202006/ideaJ-5.png)

##### JaCoCo与持续集成
1、需要在项目的`<plugins>`中加入JaCoCo插件：
```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.5</version>
    <executions>
        <execution>
            <id>default-prepare-agent</id>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>default-report</id>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```
目前发现如果项目中不加以上配置，而是在Jenkinsfile中
![](/assets/images/202006/Jenkins-1.png)
以命令的方式去应用JaCoCo，会导致不能生成`jacoco.exec`，进而无法运行覆盖率测试。

2、在Jenkinsfile中加入
![](/assets/images/202006/Jenkins-2.png)
```text
exclusionPattern: '**/controller/*.class', sourceExclusionPattern: '**/controller/*.java'
```
可以过滤掉controller层的检测。因为目前我们的单元测试主要是针对service层的，如果把controller层的类引入进来，会使单元测试覆盖率的值变低。

3、可以在Jenkins（http://${ip}:${port}/job/${your_project}/lastBuild/jacoco/）中查看生成的单元测试覆盖率报告：
![](/assets/images/202006/Jenkins-3.png)
该报告与IntelliJ IDEA中的报告都是JaCoCo原生的，是准确的。
>目前发现SonarQube中的报告一是不准，二是指标不全，建议不要查看SonarQube的报告。

---
其他相关资料：
- [Whats my Coverage? (C0 C1 C2 C3 + Path)](https://grosser.it/2008/04/04/whats-my-coverage-c0-c1-c2-c3-path-coverage/)
- [C0/C1/C2](http://atlas-softquality.com/faqanswer3.html)
- [Cyclomatic Complexity](https://docs.codeclimate.com/docs/cyclomatic-complexity)
- [Cognitive Complexity](https://docs.codeclimate.com/docs/cognitive-complexity)
- [Is Your Code Readable By Humans? Cognitive Complexity Tells You](https://www.tomasvotruba.com/blog/2018/05/21/is-your-code-readable-by-humans-cognitive-complexity-tells-you/)
- [Does Java have 'Debug' and 'Release' build mode like C#?](https://stackoverflow.com/questions/8613535/does-java-have-debug-and-release-build-mode-like-c)
- [Code Coverage - How to On-the-fly instrumentation for Jacoco](http://janejieblog.blogspot.com/2018/10/jacoco-how-to-on-fly-instrumentation.html)
- [浅谈代码覆盖率](https://tech.youzan.com/code-coverage/)
- [Skipping JaCoCo Execution Due to Missing Execution Data File](http://www.ffbit.com/blog/2014/05/21/skipping-jacoco-execution-due-to-missing-execution-data-file/)