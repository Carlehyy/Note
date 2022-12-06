# LeetCode

# 5、最长回文子串

1、暴力解法，先找到所有的子串，然后依次判断是否是回文串

```java
class Solution {
    public String longestPalindrome(String s) {
        int length = s.length();
        if(length == 0) {
            return "";
        }
        String maxStr = "";
        for(int i = 0; i < length; i++) {
            for(int j = i; j < length; j++) {
                String subStr = s.substring(i, j+1);
                // 判断当前子串是否是回文串
                int k1 = 0;
                int k2 = subStr.length()-1;
                while(k1 < k2) {
                    if(subStr.charAt(k1) == subStr.charAt(k2)) {
                        k1++;
                        k2--;
                    } else {
                        break;
                    }
                }
                // 说明是回文串
                if(k1 >= k2 && maxStr.length() < subStr.length()) {
                    maxStr = subStr;
                }
            }
        }
        return maxStr;
    }
}
```



2、中心扩散法，遍历每个字符，然后往旁边拓展，判断是否是回文串

```java
```

3、动态规划

```java
class Solution {
    public String longestPalindrome(String s) {
        int len = s.length();
        if(len < 2) {
            return s;
        }
        int a1 = 0;
        int a2 = 0;
        boolean[][] table = new boolean[len][len];
        for(int L = 0; L < len; L++) {
            for(int i = 0; i <= len-L-1; i++) {
                int j = i+L;
                if (L == 0) {
                    table[i][i] = true;
                } else if(L == 1) {
                    table[i][i+1] = (s.charAt(i) == s.charAt(i+1));
                } else {
                    table[i][j] = table[i+1][j-1] && (s.charAt(i) == s.charAt(j));
                }
                if(table[i][j] && (j-i > a2-a1)) {
                    a1 = i;
                    a2 = j;
                }
            }
        }
        return s.substring(a1, a2+1);
    }
}
```





