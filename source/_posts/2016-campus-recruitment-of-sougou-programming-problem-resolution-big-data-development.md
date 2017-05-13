title: 搜狗2016校园招聘之编程题解析-大数据开发
mathjax: true
tags: [Java]
categories: Algorithms
date: 2015-10-24 23:26:47

---

###最近邻居
![](http://7xig3q.com1.z0.glb.clouddn.com/nearest_neighbor_sougou_examination_1.png)

**解题思路**：
1. 使用JDK中的Point2D类，该类定义了坐标系空间中的一个点
2. Point2D是一个抽象类，但是在该类内部定义了静态的Double类，并且Double继承自Point2D
3. 可以通过Double的构造方法来实例化空间中的某个点
4. 将所有的输入数据全部实例化并存放在一个Point2D.Double的数组中
5. 对该数组进行暴力破解，计算其中任意两个点之间的距离，时间复杂度为$O(n^2)$，并保留下最小的两个点的编号，且编号小的在前

**Java算法实现**：
```java
import java.awt.geom.Point2D;
import java.util.Scanner;
/**
 * <p>
 * Created with IntelliJ IDEA. 2015/9/26 15:26
 * </p>
 * <p>
 * ClassName:Main
 * </p>
 * <p>
 * Description:
 * 测试数据
 * 3
 * 1.0 1.0002
 * 3.03 3.023
 * 0.0 -0.001
 * Closest points: 0, 2
 * Process finished with exit code 0
 * </P>
 *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0
 */
public class Main {

    static int[] getClosest(Point2D.Double[] points) {
        int[] result = new int[2];
        double distance = Double.MAX_VALUE;
        for (int i = 0; i < points.length; i++) {
            for (int j = i + 1; j < points.length; j++) {
                if (i != j) {
                    double distance1 = points[i].distance(points[j]);
                    if (distance1 < distance) {
                        distance = distance1;
                        if (i < j) {
                            result[0] = i;
                            result[1] = j;
                        } else {
                            result[0] = j;
                            result[i] = i;
                        }
                    }
                }
            }
        }
        return result;
    }

    public static void main(String[] args) {
        Point2D.Double[] points;
        Scanner input = new Scanner(System.in);
        {
            int n = input.nextInt();
            input.nextLine();
            points = new Point2D.Double[n];
            for (int i = 0; i < n; ++i) {
                double x = input.nextDouble();
                double y = input.nextDouble();
                input.nextLine();
                points[i] = new Point2D.Double(x, y);
            }
        }
        int[] result = getClosest(points);
        System.out.printf("Closest points: %d, %d\n", result[0], result[1]);
    }
}
```

###混乱还原

![](http://7xig3q.com1.z0.glb.clouddn.com/chaos_reduction_sougou_examination_2.png)

**解题思路**：
1. 利用伪随机特性，只要时间种子一样且上限一样，其实随机数每次都会产生相同的数
2. 既然要求还原，那么我们从后往前执行对应的操作即可
3. 使用一个额外的栈来存储所产生的随机数
4. 在乱序操作中，是将随机数对应的元素与最后一个元素进行交换，那么还原的时候，就要从第一个元素开始与最后产生的那个随机数对应的元素进行交换，依次类推，直到栈空即可

**Java算法实现**：
```java
import java.util.Arrays;
import java.util.Random;
import java.util.Scanner;
import java.util.Stack;

/**
 * <p>
 * Created with IntelliJ IDEA. 2015/9/26 15:26
 * </p>
 * <p>
 * ClassName:Main
 * </p>
 * <p>
 * Description:输入数据
 * 12312 3 1 2 3
 * Success!
 * </P>
 *
 * @author Wang Xu
 * @version V1.0.0
 * @since V1.0.0
 */


public class Main {
    static Stack<Integer> s = new Stack<Integer>();

    static void shuffle(int a[], int seed) {
        int n = a.length;
        Random random = new Random(seed);
        for (; n > 1; n--) {
            int r = random.nextInt(n);
            int tmp = a[n - 1];
            a[n - 1] = a[r];
            a[r] = tmp;
        }
    }

    static void restore(int a[], int seed) {
        int n = a.length;
        Random random = new Random(seed);
        int temp = n;
        for (; temp > 1; temp--) {
            int r = random.nextInt(temp);
            s.add(r);
        }
        for (int i = 0; i < n; i++) {
            if (!s.isEmpty()) {
                int r = s.pop();
                int tmp = a[i + 1];
                a[i + 1] = a[r];
                a[r] = tmp;
            }
        }
    }

    public static void main(String[] args) {
        int seed, n, i;
        int[] a, b;
        Scanner input = new Scanner(System.in);
        {
            seed = input.nextInt();
            n = input.nextInt();
            a = new int[n];
            for (i = 0; i < n; ++i)
                a[i] = input.nextInt();
        }
        b = Arrays.copyOf(a, a.length);
        shuffle(a, seed);
        restore(a, seed);
        for (i = 0; i < n; i++) {
            if (a[i] != b[i])
                break;
        }
        if (i == n)
            System.out.printf("Success!\n");
        else
            System.out.printf("Failed!\n");
    }
}
```
***算法如有疏漏或不妥之处，还望不吝赐教！***