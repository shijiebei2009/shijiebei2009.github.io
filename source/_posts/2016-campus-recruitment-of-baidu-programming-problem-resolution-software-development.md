title: 百度2016校园招聘之编程题解析-软件研发
mathjax: true
tags: [Java]
categories: Algorithms
date: 2015-09-29 19:20:41

---

### 比大小
![](http://7xig3q.com1.z0.glb.clouddn.com/compare_sizes_baidu_examination_1.png)

**解题思路**：解此题需要使用到**[康托展开](http://baike.baidu.com/link?url=Xx0gZYn3QJ1ykJdRvLp9HxOUBLs-J57DqebWUtjSNnoWj78xEhZeWKnCfXkhYgGMEdJ5Xz7QEwwUrIqLE7KX6a)**，康托展开的公式如下

$$X=a\_n\*(n-1)!+a\_{n-1}\*(n-2)!+\cdot\cdot\cdot+a\_i\*(i-1)!+\cdot\cdot\cdot+a\_2\*(2-1)!+a\_1\*(1-1)!$$

公式看不懂没关系，下面以一个例子来讲解公式的使用！

例如：有一个数组`S=["a","b","c","d"]`，它的其中之一个排列是`S1=["b","c","d","a"]`，现在欲把`S1`映射成`X`，需要怎么做呢？按如下步骤走起

$$X=a\_4\*3!+a\_3\*2!+a\_2\*1!+a\_1\*0!$$

1. 首先计算n，n等于数组S的长度，n=4
2. 再来计算a4="b"这个元素在数组`["b","c","d","a"]`中是第几大的元素。"a"是第0大的元素，"b"是第1大的元素，"c"是第2大的元素，"d"是第3大的元素，所以a4=1
3. 同样a3="c"这个元素在数组`["c","d","a"]`中是第几大的元素。"a"是第0大的元素，"c"是第1大的元素，"d"是第2大的元素，所以a3=1
4. a2="d"这个元素在数组`["d","a"]`中是第几大的元素。"a"是第0大的元素，"d"是第1大的元素，所以a2=1
5. a1="a"这个元素在数组`["a"]`中是第几大的元素。"a"是第0大的元素，所以a1=0
6. 所以`X(S1)=1\*3!+1\*2!+1\*1!+0\*0!=9`
7. 注意所有的计算都是按照从0开始的，如果["a","b","c","d"]算为第1个的话，那么将`X(S1)+1`即为最后的结果


**Java算法实现**：
```java
import java.util.Iterator;
import java.util.Scanner;
import java.util.Set;
import java.util.TreeSet;

/**
 * <p>
 * Created with IntelliJ IDEA. 2015/9/29 16:16
 * </p>
 * <p>
 * ClassName:Main
 * </p>
 * <p>
 * Description:本题需要用到康托展开，其公式为 X=an*(n-1)!+an-1*(n-2)!+...+ai*(i-1)!+...+a2*1!+a1*0!
 * </P>
 *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0
 */
public class Main {
    //    3
    //    abcdefghijkl
    //    hgebkflacdji
    //    gfkedhjblcia
    static int charLength = 12;//定义字符序列的长度

    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        while (scanner.hasNextInt()) {
            int n = scanner.nextInt();
            String lines[] = new String[n];
            int res[] = new int[n];//存储结果的数组
            for (int i = 0; i < n; i++) {
                lines[i] = scanner.next();
                res[i] = calculate(lines[i]);
            }
            for (int s : res) {
                System.out.println(s);
            }

        }
    }

    //计算某个字符序列的位次
    private static int calculate(String line) {
        Set<Character> s = new TreeSet<Character>();
        for (char c : line.toCharArray()) {
            s.add(c);
        }
        //存储每一个字符在该序列中是第几大的元素，然后将其值存储到counts数组中
        int counts[] = new int[s.size()];
        char[] chars = line.toCharArray();

        for (int i = 0; i < chars.length; i++) {
            Iterator<Character> iterator = s.iterator();
            int temp = 0;
            Character next;
            while (iterator.hasNext()) {
                next = iterator.next();
                if (next == chars[i]) {
                    counts[i] = temp;
                    s.remove(next);
                    break;
                } else {
                    temp++;
                }
            }
        }
        int sum = 1;
        for (int i = 0; i < counts.length; i++) {
            sum = sum + counts[i] * factorial(charLength - i - 1);
        }
        return sum;
    }

    //计算阶乘的函数
    private static int factorial(int n) {
        if (n > 1) {
            return n * factorial(n - 1);
        } else {
            return 1;
        }
    }
}
```

### 拓展一下-求其逆过程
![](http://7xig3q.com1.z0.glb.clouddn.com/compare_sizes_baidu_examination_1_expand.png)

**解题思路**：使用康托逆展开，辗转相除得到的值为这个字符是第几大，这样取出对应位置的字符，然后利用后面的字符覆盖该字符即可，防止取到重复的字符。取模得到余数之后，重复上述过程。
例如：已知`S=["a","b","c","d"]`，那么当输入10的时候，或者说X(S1)=9的时候能否推出`S1=["b","c","d","a"]`呢？

由

$$X(S1)=a\_4\*3!+a\_3\*2!+a\_2\*1!+a\_1\*0!=9$$

所以问题变成由9能否唯一地映射出一组a4、a3、a2、a1？首先如果不考虑ai的范围，那么有如下：

$$1\*3!+1\*2!+1\*1!+0\*0!=9$$

$$0\*3!+4\*2!+1\*1!+0\*0!=9$$

$$0\*3!+3\*2!+3\*1!+0\*0!=9$$

$$0\*3!+2\*2!+5\*1!+0\*0!=9$$

......，但是每一个ai其实是有取值范围的，首先要知道ai表示的含义，其代表在当前剩余的序列中ai是处于第几大的位置，那么满足`0<=ai<=i`，同时a1必然为0，因为最后始终剩余一个元素。所以上式中只有第一个满足条件，那么a4=1，a3=1，a2=1，a1=1。推导出`S1=["b","c","d","a"]`。

**Java算法实现**：
```java
import java.util.Scanner;

/**
 * <p>
 * Created with IntelliJ IDEA. 2015/9/29 16:16
 * </p>
 * <p>
 * ClassName:Main
 * </p>
 * <p>
 * Description:本题需要用到康托展开，其公式为 X=an*(n-1)!+an-1*(n-2)!+...+ai*(i-1)!+...+a2*1!+a1*0!
 * </P>
 *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0
 */
public class MainExpand {
    static int charLength = 12;//定义字符序列的长度

    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        while (scanner.hasNextInt()) {
            int n = scanner.nextInt();
            int lines[] = new int[n];
            String res[] = new String[n];//存储结果的数组
            for (int i = 0; i < n; i++) {
                lines[i] = scanner.nextInt();
                res[i] = calculate(lines[i] - 1);
            }
            for (String s : res) {
                System.out.println(s);
            }
        }

    }

    //计算某个字符序列的位次
    private static String calculate(int line) {
        char alpha[] = {'a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k', 'l'};
        StringBuilder sb = new StringBuilder();
        for (int i = charLength - 1; i >= 0; i--) {
            int temp = line / factorial(i);
            line = line % factorial(i);
            sb.append(String.valueOf(alpha[temp]));
            for (int j = temp; j < alpha.length - 1; j++) {
                alpha[j] = alpha[j + 1];
            }

        }
        return sb.toString();
