title: "Java处理带BOM文本的推荐方法"
date: 2015-05-12 20:49:39
tags: [Java]
categories: Programming Notes
---

###什么是BOM？
参见[维基百科](http://zh.wikipedia.org/zh-cn/%E4%BD%8D%E5%85%83%E7%B5%84%E9%A0%86%E5%BA%8F%E8%A8%98%E8%99%9F)，通过阅读维基百科，简单点说BOM（byte-order mark）是位于码点U+FEFF的统一码字符的名称，当以[UTF-16](http://zh.wikipedia.org/wiki/UTF-16)或[UTF-32](http://zh.wikipedia.org/wiki/UTF-32)来将[UCS](http://zh.wikipedia.org/wiki/%E9%80%9A%E7%94%A8%E5%AD%97%E7%AC%A6%E9%9B%86)/统一码字符所组成的字符串编码时，这个字符被用来标示其字节序。它常被用来当做标示文件是以UTF-8、UTF-16或UTF-32编码的记号。

从Unicode3.2开始，U+FEFF只能出现在字节流的开头，只能用于标识字节序，就如它的名称——字节序标记——所表示的一样；除此以外的用法已被舍弃。

在类Unix系统中，它们都聪明的舍弃了BOM标记，而在牛叉哄哄却最不适合程序员的Windows中，BOM被光荣的保留了，并且Windows上的记事本程序会自动在文本中添加字节顺序标记（BOM）到UTF-8文件中。这个保留的BOM标记会使得我们在处理文本过程中遇到诸多问题。在你不知情的情况下，处理起来比较麻烦，因为BOM是不可见的。只有使用带16进制功能的编辑器才可见。Java对文本的通用操作中是无法识别BOM的，所以需要借助其它办法解决。

###带BOM文本的解决办法——法一
```java
import java.io.File;
import java.io.IOException;

import org.apache.commons.io.FileUtils;

/**
 * 
 * <p>
 * ClassName TestBom
 * </p>
 * <p>
 * Description 先按照UTF-8编码读取文本，之后跳过前三个字符，重新构建一个新的字符串，需要Apache commons IO包
 * </p>
 * 
 * @author TKPad wangx89@126.com
 *         <p>
 *         Date 2015年5月12日 下午8:08:32
 *         </p>
 * @version V1.0.0
 *
 */
public class TestBom {
	public static void main(String[] args) {
		// test.txt是带BOM编码的UTF-8编码
		try {
			String content = FileUtils.readFileToString(new File("C:/Users/TKPad/Desktop/test.txt"));
			byte[] bytes = content.getBytes();
			System.out.println(content);
			System.out.println("有BOM字节大小->" + bytes.length);
			System.out.println("有BOM字符串大小->" + content.length());
			String contentNoBom = new String(bytes, 3, bytes.length - 3);
			System.out.println(contentNoBom);
			System.out.println("无BOM字节大小->" + contentNoBom.getBytes().length);
			System.out.println("无BOM字符串大小->" + contentNoBom.length());
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
}
```

输出结果：
>﻿事件语料的建立对事件及其关系识别和推理有重要意义，因此针对该语料库的建立进行研究有一定的理论意义和应用价值。
有BOM字节大小->165
有BOM字符串大小->55
事件语料的建立对事件及其关系识别和推理有重要意义，因此针对该语料库的建立进行研究有一定的理论意义和应用价值。
无BOM字节大小->162
无BOM字符串大小->54

###带BOM文本的解决办法——法二
使用牛人封装的**UnicodeReader**和**UnicodeInputStream**，地址在[这里](http://koti.mbnet.fi/akini/java/unicodereader/)，该类可以识别带BOM标记的文本，同样可以向空文件中写入BOM标记。参见以下Demo

Example code using UnicodeReader class
```java
 public static char[] loadFile(String file) throws IOException {
      // read text file, auto recognize bom marker or use 
      // system default if markers not found.
      BufferedReader reader = null;
      CharArrayWriter writer = null;
      UnicodeReader r = new UnicodeReader(new FileInputStream(file), null);
		
      char[] buffer = new char[16 * 1024];   // 16k buffer
      int read;
      try {
         reader = new BufferedReader(r);
         writer = new CharArrayWriter();
         while( (read = reader.read(buffer)) != -1) {
            writer.write(buffer, 0, read);
         }
         writer.flush();
         return writer.toCharArray();
      } catch (IOException ex) {
         throw ex;
      } finally {
         try {
            writer.close(); reader.close(); r.close();
         } catch (Exception ex) { }
      }
   }
```

Example code to write UTF-8 with bom marker
```java
 public static void saveFile(String file, String data, boolean append) throws IOException {
      BufferedWriter bw = null;
      OutputStreamWriter osw = null;
		
      File f = new File(file);
      FileOutputStream fos = new FileOutputStream(f, append);
      try {
         // write UTF8 BOM mark if file is empty
         if (f.length() < 1) {
            final byte[] bom = new byte[] { (byte)0xEF, (byte)0xBB, (byte)0xBF };
            fos.write(bom);
         }

         osw = new OutputStreamWriter(fos, "UTF-8");
         bw = new BufferedWriter(osw);
         if (data != null) bw.write(data);
      } catch (IOException ex) {
         throw ex;
      } finally {
         try { bw.close(); fos.close(); } catch (Exception ex) { }
      }
   }
```
###带BOM文本的解决办法——法三
使用Apache Commons IO包中的工具类，API参见[这里](http://commons.apache.org/proper/commons-io/apidocs/org/apache/commons/io/input/BOMInputStream.html)

```java
import java.io.File;
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.IOException;

import org.apache.commons.io.ByteOrderMark;
import org.apache.commons.io.input.BOMInputStream;

/**
 * 
 * <p>
 * ClassName TestBom
 * </p>
 * <p>
 * Description 先按照UTF-8编码读取文本，之后跳过前三个字符，重新构建一个新的字符串
 * </p>
 * 
 * @author TKPad wangx89@126.com
 *         <p>
 *         Date 2015年5月12日 下午8:08:32
 *         </p>
 * @version V1.0.0
 *
 */
public class TestBom {
	public static void main(String[] args) {
		// test.txt是带BOM编码的UTF-8编码
		BOMInputStream bomWithOut;
		try {
			// 默认排除掉BOM，但是使用hasBOM方法仍然可以检测到BOM的存在
			bomWithOut = new BOMInputStream(new FileInputStream(new File("C:/Users/TKPad/Desktop/test.txt")));
			if (bomWithOut.hasBOM()) {
				System.out.println("当前流中包含BOM！");
			}
			byte[] bytes = new byte[1024];
			int length = 0;
			String res = null;
			while ((length = bomWithOut.read(bytes)) != -1) {
				res += new String(bytes, 0, length);
			}
			System.out.println("无BOM读取：" + res.getBytes().length);
		} catch (IOException e) {
			e.printStackTrace();
		}
		// 当然你也可以包含BOM读取
		BOMInputStream bomWithIn;
		try {
			bomWithIn = new BOMInputStream(new FileInputStream(new File("C:/Users/TKPad/Desktop/test.txt")), true);
			if (bomWithIn.hasBOM()) {
				System.out.println("当前流中包含BOM！");
			}
			byte[] bytes = new byte[1024];
			int length = 0;
			String res = null;
			while ((length = bomWithIn.read(bytes)) != -1) {
				res += new String(bytes, 0, length);
			}
			System.out.println("有BOM读取：" + res.getBytes().length);

		} catch (IOException e) {
			e.printStackTrace();
		}

		// 注意，此处要注释掉哦！！
		try {
			// 也可以指定检测多种编码的bom，但目前仅支持UTF-8/UTF-16LE/UTF-16BE三种，对于UTF32之类不支持。
			BOMInputStream bomIn = new BOMInputStream(new FileInputStream(new File("")), ByteOrderMark.UTF_16LE,
					ByteOrderMark.UTF_16BE);
		} catch (FileNotFoundException e) {
			e.printStackTrace();
		}

	}
}
```
输出结果：
>当前流中包含BOM！
无BOM读取：166
当前流中包含BOM！
有BOM读取：169
