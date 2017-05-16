title: "基于同义词词林扩展版的词语相似度计算"
tags: [Java]
date: 2015-08-04 22:06:36
categories: Algorithms

---
#### 词语相似度计算
词义相似度计算在很多领域中都有广泛的应用，例如信息检索、信息抽取、文本分类、词义排歧、基于实例的机器翻译等等。国内目前主要是使用知网和同义词词林来进行词语的相似度计算。

本文主要是根据《基于同义词词林的词语相似度计算方法--田久乐》论文中所提出的分层算法实现相似度计算，程序采用Java语言编写。

#### 同义词词林扩展版
《同义词词林》是梅家驹等人于1983年编纂而成，这本词典中不仅包括了一个词语的同义词，也包含了一定数量的同类词，即广义的相关词。《同义词词林扩展版》是由哈尔滨工业大学信息检索实验室所重新修订所得。该版收录词语近7万条，全部按意义进行编排，是一部同义类词典。

同义词词林按照树状的层次结构把所有收录的词条组织到一起，把词汇分成大、中、小三类，大类有12个，中类有97个，小类有1400个。每个小类里都有很多的词，这些词又根据词义的远近和相关性分成了若干个词群（段落）。每个段落中的词语又进一步分成了若干个行，同一行的词语要么词义相同，要么词义有很强的相关性。

《同义词词林》提供了5层编码，第1级用大写英文字母表示；第2级用小写英文字母表示；第3级用二位十进制整数表示；第4级用大写英文字母表示；第5级用二位十进制整数表示。例如：
Cb30A01= 这里 这边 此地 此间 此处 此
Cb30A02# 该地 该镇 该乡 该站 该区 该市 该村
Cb30A03@ 这方

分层及编码表如下所示
![](http://7xig3q.com1.z0.glb.clouddn.com/tongyici-cilin-structure.jpg)
![](http://7xig3q.com1.z0.glb.clouddn.com/tongyici-cilin-encode.jpg)

由于第5级有的行是同义词，有的行是相关词，有的行只有一个词，分类结果需要特别说明，可以分出具体的3种情况。使用特殊符号对这3种情况进行区别对待，所以第8位的标记有3种，分别是“=”代表“相等”、“同义”；“#”代表“不等”、“同类”，属于相关词语；“@”代表“自我封闭”、“独立”，它在词典中既没有同义词，也没有相关词。

在对文本内容进行相似度计算中，采用该论文中给出的计算公式，两个义项的相似度用`Sim`表示
若两个义项不在同一棵树上，`Sim(A,B)=f`
若两个义项在同一棵树上：
若在第2层分支，系数为a `Sim(A,B)=1*a*cos*(n*π/180)((n-k+1)/n)`
若在第3层分支，系数为b `Sim(A,B)=1*1*b*cos*(n*π/180)((n-k+1)/n)`
若在第4层分支，系数为c `Sim(A,B)=1*1*1*c×cos*(n*π/180)((n-k+1)/n)`
若在第5层分支，系数为d `Sim(A,B)=1*1*1*1*d*cos*(n*π/180)((n-k+1)/n)`

当编码相同，而只有末尾是“#”时，那么认为其相似度为e。
例如`Ad02B04# 非洲人 亚洲人`  则其相似度为e。

其中n是分支层的节点分支总数，k是两个分支间的距离。
如：人 `Aa01A01=` 和 少儿 `Ab04B01=`
以A开头的子分支只有14个，分别是`Aa...，Ab...，Ac... ——— An...`，而不是以A开头的所有结点的个数，所以`n=14`；在第2层分支上，人的编码是`a`，少儿的编码是`b`，a和b之间差1，所以`k=1`。

该文献中给出的参数值为`a=0.65`，`b=0.8`，`c=0.9`，`d=0.96`，`e=0.5`，`f=0.1`。

#### Java代码实现
```java
package cn.codepub.algorithms.similarity.cilin;

import com.google.common.base.Preconditions;
import lombok.extern.log4j.Log4j2;
import org.apache.commons.io.IOUtils;
import org.apache.commons.lang3.StringUtils;
import org.junit.Test;

import java.io.IOException;
import java.io.InputStream;
import java.util.*;

import static java.lang.Math.PI;
import static java.lang.Math.cos;

/**
 * <p>
 * Created with IntelliJ IDEA. 2015/8/2 21:54
 * </p>
 * <p>
 * ClassName:WordSimilarity 同义词词林扩展版计算词语相似度
 * </p>
 * <p>
 * Description:<br/>
 * "=" 代表 相等 同义 <br/>
 * "#" 代表 不等 同类 属于相关词语 <br/>
 * "@" 代表 自我封闭 独立 它在词典中既没有同义词, 也没有相关词 <br/>
 * </P>
 *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0
 */
@Log4j2
//注意使用Log4j2的注解，那么在pom中必须引入2.x版本的log4j，如果使用Log4j注解，pom中引入1.x版本的log4j
//相应的配置文件也要一致，2.x版本配置文件为log4j2.xml，1.x版本配置文件为log4j.xml
public class WordSimilarity {
    /**
     * when we use Lombok's Annotation, such as @Log4j
     *
     * @Log4j <br/>
     * public class LogExample {
     * }
     * <p>
     * will generate:
     * public class LogExample {
     * private static final org.apache.logging.log4j.Logger log = org.apache.logging.log4j.Logger.getLogger(LogExample.class);
     * }
     * </p>
     */
    //定义一些常数先
    private static final double a = 0.65;
    private static final double b = 0.8;
    private static final double c = 0.9;
    private static final double d = 0.96;
    private static final double e = 0.5;
    private static final double f = 0.1;

    private static final double degrees = 180;


    //存放的是以词为key，以该词的编码为values的List集合，其中一个词可能会有多个编码
    private static Map<String, ArrayList<String>> wordsEncode = new HashMap<String, ArrayList<String>>();
    //存放的是以编码为key，以该编码多对应的词为values的List集合，其中一个编码可能会有多个词
    private static Map<String, ArrayList<String>> encodeWords = new HashMap<String, ArrayList<String>>();

    /**
     * 读取同义词词林并将其注入wordsEncode和encodeWords
     */
    private static void readCiLin() {

        InputStream input = WordSimilarity.class.getClass().getResourceAsStream("/cilin.txt");
        List<String> contents = null;
        try {
            contents = IOUtils.readLines(input);

            for (String content : contents) {
                content = Preconditions.checkNotNull(content);
                String[] strsArr = content.split(" ");
                String[] strs = Preconditions.checkNotNull(strsArr);
                String encode = null;
                int length = strs.length;
                if (length > 1) {
                    encode = strs[0];//获取编码
                }
                ArrayList<String> encodeWords_values = new ArrayList<String>();
                for (int i = 1; i < length; i++) {
                    encodeWords_values.add(strs[i]);
                }
                encodeWords.put(encode, encodeWords_values);//以编码为key，其后所有值为value
                for (int i = 1; i < length; i++) {
                    String key = strs[i];
                    if (wordsEncode.containsKey(strs[i])) {
                        ArrayList<String> values = wordsEncode.get(key);
                        values.add(encode);
                        //重新放置回去
                        wordsEncode.put(key, values);//以某个value为key，其可能的所有编码为value
                    } else {
                        ArrayList<String> temp = new ArrayList<String>();
                        temp.add(encode);
                        wordsEncode.put(key, temp);
                    }
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
            System.err.println("load dictionary failed！");
            log.error(e.getMessage());
        }
    }

    /**
     * 对外暴露的接口，返回两个词的相似度的计算结果
     *
     * @param word1
     * @param word2
     * @return 相似度值
     */
    public static double getSimilarity(String word1, String word2) {
        //在计算时候再加载，实现懒加载
   