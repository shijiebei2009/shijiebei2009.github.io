title: Faceted Project Problem (Java Version Mismatch) Error Message
date: 2015-06-02 16:38:58
tags: [Java, Eclipse]
categories: Programming Notes
toc: false

---

在eclipse的 "problems" 选项卡中显示如下错误信息
```java
Description:Type Project facet Java 1.8 is not supported by target runtime Apache Tomcat v7.0
Resource:groupping
...
```
由StackOverflow上的回答可知，Java facet的版本总是需要和Java编译器的版本一致，所以最好的方式是通过Project Facets Properties面板进行修改。
1. 查看Problems面板信息
![][1]
2. 打开Project Facets Properties面板
![][2]
3. 修改configuration信息
![][3]
4. 调整到和Java编译器版本相匹配
![][4]


  [1]: http://7xig3q.com1.z0.glb.clouddn.com/eclipse-problems-panel.png
  [2]: http://7xig3q.com1.z0.glb.clouddn.com/eclipse-project-facets-configuration.png
  [3]: http://7xig3q.com1.z0.glb.clouddn.com/project-facets-properties.png
  [4]: http://7xig3q.com1.z0.glb.clouddn.com/matching-java-compiler-compliance-level.png


参考资料
【1】http://javahonk.com/project-facet-java-is-not-supported-by-target-runtime/
【2】http://stackoverflow.com/questions/2239959/faceted-project-prblem-java-version-mismatch-error-message

