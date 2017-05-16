title: CCF第六次CSP认证考试题解
tags: [CCF]
date: 2015-12-21 23:27:52
categories: Algorithms

---
#### CCF计算机职业资格认证
CCF**（China Computer Federation）**是计算机领域内一个权威的学术组织，具有高端定位、崇高的价值追求、先进的治理架构和制度规范，拥有众多资深的学者和企业家为骨干会员。CCF开展的该认证工作，具有客观公正及很强的专业性，将解决企业及高校界普遍关心的软件开发人才评价问题，便于有关单位了解求职或求学者的实际开发能力，可有助甄别及吸纳具有真才实学的技术人才，有效地减轻企业与高校在人才选择过程中组织大量上机考核的成本投入。为了更加贴近用人单位的需要，使得该认证有更强的针对性，CCF会同处于行业领先地位的高校与知名企业的专家，共同制定认证标准，审查考题内容。更多详情[点我](http://cspro.ccf.org.cn/lead/info.do?__action=info_view&catalog=notice&id=hrvnsypp-1gg&__forward=true)。

#### CSP认证第六次考试
本次考试时间四小时，共五道题。分别是：
- 数位之和（难度-易）
- 消除类游戏（难度-中）
- 画图（难度-中）
- 送货（难度-难）
- 矩阵（难度-难）

#### 数位之和
**题目说明**：输入任意一个整数，输出各位数位之和。输入的整数不会超过整型的最大表示范围。
**解答**：
```java
import java.util.Scanner;
public class Main {
    public static void main(String[] args) {
        Scanner in = new Scanner(System.in);
        while (in.hasNext()) {
            int num = in.nextInt();
            int sum = getSum(num);
            System.out.println(sum);
        }

    }
    public static int getSum(int num) {
        int sum = 0;
        while (num > 0) {
            int temp = num % 10;
            sum += temp;
            num = num / 10;
        }
        return sum;
    }
}
```

#### 消除类游戏
**题目说明**：
![](http://7xig3q.com1.z0.glb.clouddn.com/CCF_CSP_Elimination_Game.png)
**解答**：
```java
import java.util.Scanner;

public class Main {
    public static void main(String[] args) {
        Scanner in = new Scanner(System.in);
        int[][] arrs = null;//原始数组
        int[][] dest = null;//消除之后的数组
        while (in.hasNext()) {
            int row = in.nextInt();
            int col = in.nextInt();
            in.nextLine();
            arrs = new int[row][col];
            dest = new int[row][col];
            for (int i = 0; i < row; i++) {
                for (int j = 0; j < col; j++) {
                    arrs[i][j] = in.nextInt();
                    dest[i][j] = arrs[i][j];
                }
                in.nextLine();
            }
            //处理行
            for (int i = 0; i < row; i++) {
                int start = 0;
                int end = 0;
                for (int j = 0; j < col - 1; j++) {
                    if (arrs[i][j] == arrs[i][j + 1]) {
                        end = j + 1;
                    } else {
                        end = j + 1;
                        if (start == 0) {
                            if (end - start >= 3) {
                                for (int k = start; k < end; k++) {
                                    dest[i][k] = 0;
                                }
                            }
                        } else {
                            if (end - start + 1 >= 3) {
                                for (int k = start; k < end; k++) {
                                    dest[i][k] = 0;
                                }
                            }
                        }
                        start = end;
                    }

                }
                if (start == 0) {
                    if (end - start >= 3) {
                        for (int k = start; k <= end; k++) {
                            dest[i][k] = 0;
                        }
                    }
                } else {
                    if (end - start + 1 >= 3) {
                        for (int k = start; k <= end; k++) {
                            dest[i][k] = 0;
                        }
                    }
                }
            }
            //处理列
            for (int i = 0; i < col; i++) {
                int start = 0;
                int end = 0;
                for (int j = 0; j < row - 1; j++) {
                    if (arrs[j][i] == arrs[j + 1][i]) {
                        end = j + 1;
                    } else {
                        end = j + 1;
                        if (start == 0) {
                            if (end - start >= 3) {
                                for (int k = start; k < end; k++) {
                                    dest[k][i] = 0;
                                }
                            }
                        } else {
                            if (end - start + 1 >= 3) {
                                for (int k = start; k < end; k++) {
                                    dest[k][i] = 0;
                                }
                            }
                        }
                        start = end;
                    }

                }
                if (start == 0) {
                    if (end - start >= 3) {
                        for (int k = start; k <= end; k++) {
                            dest[k][i] = 0;
                        }
                    }
                } else {
                    if (end - start + 1 >= 3) {
                        for (int k = start; k <= end; k++) {
                            dest[k][i] = 0;
                        }
                    }
                }
            }
            for (int[] arr : dest) {
                for (int a : arr) {
                    System.out.print(a + " ");
                }
                System.out.println();
            }
        }

    }
}

```

#### 画图
**题目说明**：
![](http://7xig3q.com1.z0.glb.clouddn.com/CCF_CSP_Paint_1.png)
![](http://7xig3q.com1.z0.glb.clouddn.com/CCF_CSP_Paint_2.png)
**解答**：
```java
import java.util.Scanner;

public class Main {
    static int COL = 0;
    static int ROW = 0;

    public static void main(String[] args) {
        Scanner in = new Scanner(System.in);
        while (in.hasNext()) {
            COL = in.nextInt();
            ROW = in.nextInt();
            int num = in.nextInt();
            String[][] arrs = new String[ROW][COL];

            for (int i = 0; i < arrs.length; i++) {
                for (int j = 0; j < arrs[i].length; j++) {
                    arrs[i][j] = "·";
                }
            }
            in.nextLine();
            for (int i = 0; i < num; i++) {
                int first = in.nextInt();
                if (first == 0) {//表示画线段操作
                    int x1 = in.nextInt();
                    int y1 = in.nextInt();
                    int x2 = in.nextInt();
                    int y2 = in.nextInt();
                    if (x1 == x2) {
                        // |
                        int big = y1 > y2 ? y1 : y2;
                        int small = y1 < y2 ? y1 : y2;
                        for (int start = small; start <= big; start++) {
                            if (arrs[ROW - start - 1][x1] == "-") {
                                arrs[ROW - start - 1][x1] = "+";
                            } else {
                                arrs[ROW - start - 1][x1] = "|";
                            }
                        }
                    }
                    if (y1 == y2) {
                        // -
                        int big = x1 > x2 ? x1 : x2;
                        int small = x1 < x2 ? x1 : x2;
                        for (int start = small; start <= big; start++) {
                            if (arrs[ROW - y1 - 1][start] == "|") {
                                arrs[ROW - y1 - 1][star