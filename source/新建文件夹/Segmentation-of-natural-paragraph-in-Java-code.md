title: "Java代码实现按照文本的自然段落进行切分"
date: 2015-04-25 14:58:43
tags: [Java]
categories: Programming Notes
toc: false

---
本文给出了五种自然段落的组合方式，具体形式参见底部给出的链接，无积分下载，不管各个段落形式如何，只要段落之间存在换行分隔，该程序即可正确运行。在此提供两种切分段落的方法，分别见下面的法一和法二。
法一：
```java
import java.io.BufferedReader;
import java.io.FileInputStream;
import java.io.IOException;
import java.io.InputStreamReader;
import java.util.ArrayList;
import java.util.List;

/**
 * 
 * <p>
 * ClassName SplitParagraph
 * </p>
 * <p>
 * Description 对文本按照自然段落进行切分
 * </p>
 * 
 * @author TKPad wangx89@126.com
 *         <p>
 *         Date 2015年4月25日 下午3:37:42
 *         </p>
 * @version V1.0.0
 *
 */
public class SplitParagraph {
	/**
	 * 
	 * <p>
	 * Title: getParagraph
	 * </p>
	 * <p>
	 * Description: 根据给出的文本路径，读取文本内容，并按照段落切分，以段落为单位，存储在List集合中
	 * </p>
	 * 
	 * @param filePath
	 * @return List&lt;String&gt; 段落集合
	 * @throws IOException
	 *
	 */
	private List<String> getParagraph(String filePath, String charset) throws IOException {
		List<String> res = new ArrayList<String>();
		StringBuilder sb = new StringBuilder();// 拼接读取的内容
		InputStreamReader isr = new InputStreamReader(new FileInputStream(filePath), charset);
		BufferedReader br = new BufferedReader(isr);
		String temp;
		while ((temp = br.readLine()) != null) {
			sb.append(temp.trim() + "\n");
		}
		// \s A whitespace character: [ \t\n\x0B\f\r]
		String p[] = sb.toString().split("\\s*\n");
		for (String string : p) {
			res.add(string.replaceAll("\\s*", ""));
		}
		if (br != null)
			br.close();
		if (isr != null)
			isr.close();
		return res;
	}
}

```
法二：
```java
import java.io.BufferedReader;
import java.io.File;
import java.io.FileReader;
import java.io.IOException;
import java.util.ArrayList;
import java.util.Iterator;

/**
 * 
 * <p>
 * ClassName GetParagraph
 * </p>
 * <p>
 * Description 使用Java完成对一篇文本的自然段落的切分，在此给出了五种文本格式作为示例，对任一种格式，该程序均可以正确切分。
 * </p>
 * 
 * @author TKPad wangx89@126.com
 *         <p>
 *         Date 2015年2月11日 下午1:33:03
 *         </p>
 * @version V1.0.0
 *
 */
public class GetParagraph {
	public static void main(String[] args) throws IOException {
		ArrayList<String> res = new ArrayList<String>();// 段落切分结果
		StringBuilder sb = new StringBuilder();// 拼接读取的内容
		String temp = null;// 临时变量，存储sb去除空格的内容
		String filePath = "C:\\Users\\TKPad\\Desktop\\test/a.txt";
		BufferedReader reader = new BufferedReader(new FileReader(new File(filePath)));
		int ch = 0;
		while ((ch = reader.read()) != -1) {
			temp = sb.toString().trim().replaceAll("\\s*", "");// 取出前后空格，之后去除中间空格
			if ((char) ch == '\r') {
				// 判断是否是空行
				if (!"".equals(temp)) {
					// 说明到了段落结尾，将其加入链表，并清空sb
					res.add(temp);
				}
				sb.delete(0, sb.length());
			} else {
				// 说明没到段落结尾，将结果暂存
				sb.append((char) ch);
			}
		}
		if (reader.read() == -1) {
			System.out.println("哈哈，你读到了末尾嘞！");
		}
		// 最后一段如果非空， 将最后一段加入，否则不处理
		if (!"".equals(temp)) {
			res.add(temp);
		}

		Iterator<String> iterator = res.iterator();
		while (iterator.hasNext()) {
			String next = iterator.next();
			System.out.println("段落开始：");
			System.out.println(next);
		}
		System.out.println("段落的个数是：" + res.size());
	}
}

```
测试用例点我：http://download.csdn.net/download/shijiebei2009/8440133
