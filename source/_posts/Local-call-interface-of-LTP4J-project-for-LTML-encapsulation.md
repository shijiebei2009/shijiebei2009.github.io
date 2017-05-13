title: "封装LTP4J的本地LTML调用接口"
date: 2015-05-13 16:53:00
tags: [LTP]
categories: Programming Notes
---

###LTP4J简介
LTP4J是对LTP的Java接口封装，众所周知LTP底层均是C++实现，所以对于需要Java接口的开发人员来说要通过调用LTP4J的接口实现调用LTP的目的，但是该项目仅仅封装了几个独立的方法，分别是**NER/Parser/Postagger/Segmentor/SRL**与之对应的实现功能是**命名实体识别/依存句法分析/词性标注/分词/语义角色标注**，LTP4J中封装的都是native方法，无法查看源代码。

###Java之native关键字
Java native Interface简称JNI，即Java本地接口，JNI是一个编程框架，可以使得运行在[Java虚拟机](https://zh.wikipedia.org/wiki/Java%E8%99%9A%E6%8B%9F%E6%9C%BA)上的Java程序调用与平台相关的其它语言实现的程序，比如C/C++/汇编等。

换言之，JNI允许用本地代码来解决纯粹用Java编程不能解决的平台相关的特性。对于Windows平台上的C++程序而言，在Java中将某个方法声明为native的，然后用C++程序实现该方法的具体功能，之后将C++程序编译为[DLL](https://zh.wikipedia.org/wiki/%E5%8A%A8%E6%80%81%E9%93%BE%E6%8E%A5%E5%BA%93)文件即可。在Java虚拟机执行到该native方法的时候，会由操作系统去寻找对应的DLL文件进而调用该方法的C++实现。

所以使用LTP4J的前提是需要编译LTP源码及LTP4J的源码，在Windows X64平台上编译LTP源码及LTP4J源码请参见：
+ 
[编译哈工大语言技术平台云LTP（C++）源码及LTP4J（Java）源码][id]
[id]: http://codepub.cn/2015/05/07/Compile-the-Language-Technology-Platform(C++)-and-LTP4J(Java)source-code/

在Windows平台上使用LTP4J需要处理BOM问题，解决办法请参见：
+ [Java处理带BOM文本的推荐方法](http://codepub.cn/2015/05/12/Recommended-processing-method-of-text-with-bom-in-Java/)

###LTML简介
有关LTML的详细信息参见[LTML数据表示](https://github.com/HIT-SCIR/ltp/blob/master/doc/ltp-document-3.0.md)
LTML 标准要求如下：结点标签分别为 xml4nlp, note, doc, para, sent, word, arg 共七种结点标签：

1. xml4nlp 为根结点，无任何属性值；
2. note 为标记结点，具有的属性分别为：sent, word, pos, ne, parser, srl；分别代表分句，分词，词性标注，命名实体识别，依存句法分析，词义消歧，语义角色标注；值为”n”，表明未做，值为”y”则表示完成，如pos=”y”，表示已经完成了词性标注；
3. doc 为篇章结点，以段落为单位包含文本内容；无任何属性值；
4. para 为段落结点，需含id 属性，其值从0 开始；
5. sent 为句子结点，需含属性为id，cont；id 为段落中句子序号，其值从0 开始；cont 为句子内容；
6. word 为分词结点，需含属性为id, cont；id 为句子中的词的序号，其值从0 开始，cont为分词内容；可选属性为 pos, ne, parent, relate；pos 的内容为词性标注内容；ne 为命名实体内容；parent 与relate 成对出现，parent 为依存句法分析的父亲结点id 号，relate 为相对应的关系；
7. arg 为语义角色信息结点，任何一个谓词都会带有若干个该结点；其属性为id, type, beg，end；id 为序号，从0 开始；type 代表角色名称；beg 为开始的词序号，end 为结束的序号；

LTP的在线版提供LTML格式的返回数据(亦即XML格式)，地址在[这里](http://www.ltp-cloud.com/demo/)，但是本地版不提供，需要自己封装。

###封装本地版LTML的Java调用接口
对外提供了两个接口，其一是根据文件内容获取完整的LTML标注结果，文件编码默认UTF-8，代码如下：
```java
package edu.shu.ltp4j.test;

import org.junit.Test;

import edu.shu.ltp4j.util.LTPUtil;

public class TestGetFileLTML {
	@Test
	public void test() {
		String path = "test.txt";
		LTPUtil ltpUtil = new LTPUtil();
		ltpUtil.setNe(true);
		ltpUtil.setParser(true);
		ltpUtil.setPos(true);
		ltpUtil.setSrl(true);
		String ltml = ltpUtil.getLTML(path);
		System.out.println(ltml);
	}
}
```
本地分析输出结果：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<xml4nlp>
	<note ne="y" parser="y" pos="y" sent="y" srl="y" word="y" />
	<doc>
		<para id="0">
			<sent cont="我爱你中国！" id="0">
				<word cont="我" id="0" ne="O" parent="1" pos="r" relate="SBV" />
				<word cont="爱" id="1" ne="O" parent="-1" pos="v" relate="HED">
					<arg beg="0" end="0" id="0" type="A0" />
					<arg beg="2" end="3" id="1" type="A1" />
				</word>
				<word cont="你" id="2" ne="O" parent="3" pos="r" relate="ATT" />
				<word cont="中国" id="3" ne="S-Ns" parent="1" pos="ns" relate="VOB" />
				<word cont="！" id="4" ne="O" parent="1" pos="wp" relate="WP" />
			</sent>
		</para>
		<para id="1">
			<sent cont="事件语料的建立对事件及其关系识别和推理有重要意义，因此针对该语料库的建立进行研究有一定的理论意义和应用价值。" id="0">
				<word cont="事件" id="0" ne="O" parent="1" pos="n" relate="ATT" />
				<word cont="语料" id="1" ne="O" parent="3" pos="n" relate="ATT" />
				<word cont="的" id="2" ne="O" parent="1" pos="u" relate="RAD" />
				<word cont="建立" id="3" ne="O" parent="-1" pos="v" relate="HED">
					<arg beg="4" end="13" id="0" type="A1" />
				</word>
				<word cont="对" id="4" ne="O" parent="11" pos="p" relate="ADV" />
				<word cont="事件" id="5" ne="O" parent="8" pos="n" relate="SBV" />
				<word cont="及其" id="6" ne="O" parent="7" pos="c" relate="LAD" />
				<word cont="关系" id="7" ne="O" parent="5" pos="n" relate="COO" />
				<word cont="识别" id="8" ne="O" parent="4" pos="v" relate="POB">
					<arg beg="5" end="7" id="0" type="A0" />
				</word>
				<word cont="和" id="9" ne="O" parent="10" pos="c" relate="LAD" />
				<word cont="推理" id="10" ne="O" parent="8" pos="v" relate="COO" />
				<word cont="有" id="11" ne="O" parent="3" pos="v" relate="VOB">
					<arg beg="12" end="13" id="0" type="A1" />
				</word>
				<word cont="重要" id="12" ne="O" parent="13" pos="a" relate="ATT" />
				<word cont="意义" id="13" ne="O" parent="11" pos="n" relate="VOB" />
				<word cont="，" id="14" ne="O" parent="3" pos="wp" relate="WP" />
				<word cont="因此" id="15" ne="O" parent="20" pos="c" relate="ADV" />
				<word cont="针对" id="16" ne="O" parent="20" pos="p" relate="ADV" />
				<word cont="该" id="17" ne="O" parent="18" pos="r" relate="ATT" />
				<word cont="语料库" id="18" ne="O" parent="16" pos="n" relate="POB" />
				<word cont="的" id="19" ne="O" parent="16" pos="u" relate="RAD" />
				<word cont="建立" id="20" ne="O" parent="3" pos="v" relate="COO">
					<arg beg="15" end="15" id="0" type="DIS" />
					<arg beg="21" end="30" id="1" type="A1" />
				</word>
				<word cont="进行" id="21" ne="O" parent="30" pos="v" relate="ATT">
					<arg beg="22" end="29" id="0" type="A1" />
				</word>
				<word cont="研究" id="22" ne="O" parent="21" pos="v" relate="VOB" />
				<word cont="有" id="23" ne="O" parent="22" pos="v" relate="COO">
					<arg beg="24" end="27" id="0" type="A1" />
				</word>
				<word cont="一定" id="24" ne="O" parent="27" pos="b" relate="ATT" />
				<word cont="的" id="25" ne="O" parent="24" pos="u" relate="RAD" />
				<word cont="理论" id="26" ne="O" parent="27" pos="n" relate="ATT" />
				<word cont="意义" id="27" ne="O" parent="23" pos="n" relate="VOB" />
				<word cont="和" id="28" ne="O" parent="29" pos="c" relate="LAD" />
				<word cont="应用" id="29" ne="O" parent="23" pos="v" relate="COO" />
				<word cont="价值" id="30" ne="O" parent="20" pos="n" relate="VOB" />
				<word cont="。" id="31" ne="O" parent="3" pos="wp" relate="WP" />
			</sent>
		</para>
	</doc>
</xml4nlp>
```

其二根据一个完整的句子获取到完整的LTML标注结果，完整的句子意思是使用“。|？|！|；”进行切分的结果，代码如下：
```java
package edu.shu.ltp4j.test;

import org.junit.Test;

import edu.shu.ltp4j.util.LTPUtil;

public class TestGetSentectLTML {
	@Test
	public void testSentence() {
		LTPUtil ltpUtil = new LTPUtil();
		ltpUtil.setNe(true);
		ltpUtil.setParser(true);
		ltpUtil.setPos(true);
		ltpUtil.setSrl(true);
		String s = "事件语料的建立对事件及其关系识别和推理有重要意义，因此针对该语料库的建立进行研究有一定的理论意义和应用价值。";
		String ltmlBySentence = ltpUtil.getLTMLBySentence(s);
		System.out.println(ltmlBySentence);
	}
}
```
本地分析输出结果：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<xml4nlp>
	<note ne="y" parser="y" pos="y" sent="y" srl="y" word="y" />
	<doc>
		<para id="0">
			<sent cont="事件语料的建立对事件及其关系识别和推理有重要意义，因此针对该语料库的建立进行研究有一定的理论意义和应用价值。" id="0">
				<word cont="事件" id="0" ne="O" parent="1" pos="n" relate="ATT" />
				<word cont="语料" id="1" ne="O" parent="3" pos="n" relate="ATT" />
				<word cont="的" id="2" ne="O" parent="1" pos="u" relate="RAD" />
				<word cont="建立" id="3" ne="O" parent="-1" pos="v" relate="HED">
					<arg beg="4" end="13" id="0" type="A1" />
				</word>
				<word cont="对" id="4" ne="O" parent="11" pos="p" relate="ADV" />
				<word cont="事件" id="5" ne="O" parent="8" pos="n" relate="SBV" />
				<word cont="及其" id="6" ne="O" parent="7" pos="c" relate="LAD" />
				<word cont="关系" id="7" ne="O" parent="5" pos="n" relate="COO" />
				<word cont="识别" id="8" ne="O" parent="4" pos="v" relate="POB">
					<arg beg="5" end="7" id="0" type="A0" />
				</word>
				<word cont="和" id="9" ne="O" parent="10" pos="c" relate="LAD" />
				<word cont="推理" id="10" ne="O" parent="8" pos="v" relate="COO" />
				<word cont="有" id="11" ne="O" parent="3" pos="v" relate="VOB">
					<arg beg="12" end="13" id="0" type="A1" />
				</word>
				<word cont="重要" id="12" ne="O" parent="13" pos="a" relate="ATT" />
				<word cont="意义" id="13" ne="O" parent="11" pos="n" relate="VOB" />
				<word cont="，" id="14" ne="O" parent="3" pos="wp" relate="WP" />
				<word cont="因此" id="15" ne="O" parent="20" pos="c" relate="ADV" />
				<word cont="针对" id="16" ne="O" parent="20" pos="p" relate="ADV" />
				<word cont="该" id="17" ne="O" parent="18" pos="r" relate="ATT" />
				<word cont="语料库" id="18" ne="O" parent="16" pos="n" relate="POB" />
				<word cont="的" id="19" ne="O" parent="16" pos="u" relate="RAD" />
				<word cont="建立" id="20" ne="O" parent="3" pos="v" relate="COO">
					<arg beg="15" end="15" id="0" type="DIS" />
					<arg beg="21" end="30" id="1" type="A1" />
				</word>
				<word cont="进行" id="21" ne="O" parent="30" pos="v" relate="ATT">
					<arg beg="22" end="29" id="0" type="A1" />
				</word>
				<word cont="研究" id="22" ne="O" parent="21" pos="v" relate="VOB" />
				<word cont="有" id="23" ne="O" parent="22" pos="v" relate="COO">
					<arg beg="24" end="27" id="0" type="A1" />
				</word>
				<word cont="一定" id="24" ne="O" parent="27" pos="b" relate="ATT" />
				<word cont="的" id="25" ne="O" parent="24" pos="u" relate="RAD" />
				<word cont="理论" id="26" ne="O" parent="27" pos="n" relate="ATT" />
				<word cont="意义" id="27" ne="O" parent="23" pos="n" relate="VOB" />
				<word cont="和" id="28" ne="O" parent="29" pos="c" relate="LAD" />
				<word cont="应用" id="29" ne="O" parent="23" pos="v" relate="COO" />
				<word cont="价值" id="30" ne="O" parent="20" pos="n" relate="VOB" />
				<word cont="。" id="31" ne="O" parent="3" pos="wp" relate="WP" />
			</sent>
		</para>
	</doc>
</xml4nlp>
```
整个项目的源代码已经上传到Github仓库，欢迎[下载使用](https://github.com/shijiebei2009/BuildLTMLForLTP)！