leetcode 344 反转字符串

题目

```
//编写一个函数，其作用是将输入的字符串反转过来。输入字符串以字符数组 char[] 的形式给出。
//
// 不要给另外的数组分配额外的空间，你必须原地修改输入数组、使用 O(1) 的额外空间解决这一问题。
//
// 你可以假设数组中的所有字符都是 ASCII 码表中的可打印字符。
//
//
//
// 示例 1：
//
// 输入：["h","e","l","l","o"]
//输出：["o","l","l","e","h"]
//
//
// 示例 2：
//
// 输入：["H","a","n","n","a","h"]
//输出：["h","a","n","n","a","H"]
// Related Topics 双指针 字符串
// 👍 325 👎 0
```

**解题思路：**

双指针分别指向头尾

每次循环交换位置

![image-20201110171447462](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20201110171447462.png)

```java
/**
 * @author by cyf
 * @date 2020/11/10.
 */
public class ReverseString {

    /**
     * 解题思路 双指针 头尾两边各有一个指针 每次交换头尾
     *
     * @param s 目标
     */
  private static void reverseString(char[] s) {
        int len = s.length;
        int i, j = len;
        for (i = 0; i < len / 2; i++) {
            j--;
            char temp = s[i];
            s[i] = s[j];
            s[j] = temp;

        }
    }

    public static void main(String[] args) {
        //char[] chars = {'h', 'e', 'l', 'l', 'o'};
        char[] chars = {'H', 'a', 'n', 'n', 'a', 'h'};
        reverseString(chars);
        System.out.println(chars);
    }
}
```

