# 代码思路
1. 使用了BigInteger和BigDecimal，这样可以省去对字符串是否为数字的判断，也方便了后续直接使用字符串进行处理
2. 先找到小数点，将数字分为两个部分，整数部分还可以确定符号性，确定后，将整数部分改为正数，方便后续处理，这也是类的三个字段的由来
3.  小数部分很好处理，直接转换就行
4.  整数部分每4位分为一组
5.  每组的4位数在转换时，依次转换，遇到0时，不需要跟上单位
6.  我预留了处理一十几的读数，这样就能忽略最高位的一
7.  每组的4位数内，高位的“零”需要保留一个，因为分割数组时不会出现最高位是0的情况，多亏了BigInteger和BigDecimal的toString方法
8.  中间的两个0可以连读为一个0，末尾的连续0就不读了
9.  如果整组4个数都是0，需要按照最高位是0的情况保留一个0
10.  已经分好组的单位，可以发现规律，先是“万”，然后是“亿”，剩下的就靠循环补足“亿”即可

# 代码
java5 以上都支持，欢迎测试，甚至提出优化建议，一起交流，这个功能可以服务于文字转语音时，对数字读取的准确性，使得语音不是那么生硬
```java
import java.math.BigDecimal;
import java.math.BigInteger;

public class Main {
    public static void main(String[] args) {
        for (int i = 0; i <= 31; i++) {
            toStr(i);
        }
        for (int i = 50998; i <= 51010; i++) {
            toStr(i);
        }
        for (int i = 99998; i <= 100002; i++) {
            toStr(i);
        }

        toStr(-1200001234);

        String s = "-0.1234567890123";
        System.out.println(s + "=>" + new CN_NumString(new BigDecimal(s)).parse());

        s = "0.0001";
        System.out.println(s + "=>" + new CN_NumString(new BigDecimal(s)).parse());

        s = "-1002340005600.1000";
        System.out.println(s + "=>" + new CN_NumString(new BigDecimal(s)).parse());
    }

    public static void toStr(int i) {
        System.out.println(i + "=>" + new CN_NumString(new BigDecimal(Integer.toString(i))).parse());
    }
}

class CN_NumString {
    private static String[] CN_NUMBER = {"零", "一", "二", "三", "四", "五", "六", "七", "八", "九"};
    private static String[] CN_CONST = {"负", "点"};
    private static String[] CN_UNIT_1 = {"", "十", "百", "千"};
    private static String[] CN_UNIT_2 = {"", "万", "亿"};

    private boolean isNeg;
    private String iStr;
    private String fStr;

    public CN_NumString(BigDecimal b) {
        String s = b.toString();
        CN_NumString n = new CN_NumString(s);
        this.isNeg = n.isNeg;
        this.iStr = n.iStr;
        this.fStr = n.fStr;
    }

    public CN_NumString(BigInteger b) {
        String s = b.toString();
        CN_NumString n = new CN_NumString(s);
        this.isNeg = n.isNeg;
        this.iStr = n.iStr;
        this.fStr = "";
    }

    private CN_NumString(String string) {
        String[] s = string.split("\\.");
        this.iStr = s[0];
        this.isNeg = this.iStr.charAt(0) == '-';
        if (this.isNeg) this.iStr = this.iStr.substring(1);
        this.fStr = s.length == 2 ? s[1] : "";
        if (this.fStr.isEmpty() && "0".equals(this.iStr)) this.isNeg = false;
    }

    public String parse() {
        StringBuilder sb = new StringBuilder();
        if (isNeg) sb.append(CN_CONST[0]);
        String s = _0(parseInteger());
        sb.append(s.isEmpty() ? CN_NUMBER[0] : s);
        if (this.fStr != null && !this.fStr.isEmpty()) sb.append(CN_CONST[1]).append(parseFloat());
        return sb.toString();
    }

    private String parseInteger() {
        StringBuilder sb = new StringBuilder();
        String[] ss = _4(this.iStr);
        for (int i = 0; i < ss.length; i++) {
            int g = ss.length - 1 - i;
            if ("0000".equals(ss[g])) {
                sb.append(CN_NUMBER[0]);
            } else {
                sb.append(_0(parse_4(ss[g])))
                        .append(parseIntegerUnit(g));
            }
        }
        return sb.toString();
    }

    private String parseIntegerUnit(int g) {
        StringBuilder sb = new StringBuilder();
        if (g == 0) return CN_UNIT_2[0];
        if (g % 2 == 1) sb.append(CN_UNIT_2[1]);
        for (int i = 0; i < g - 1; i = i + 2) sb.append(CN_UNIT_2[2]);
        return sb.toString();
    }

    private String parse_4(String n) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < n.length(); i++) {
            String s = CN_NUMBER[n.charAt(i) - '0'];
            sb.append(s);
            if (!CN_NUMBER[0].equals(s)) {
                String u = CN_UNIT_1[n.length() - 1 - i];
                sb.append(u);
            }
        }
        return _10(sb.toString());
        //return sb.toString();
    }

    private String _10(String s) {
        return (s.length() == 2 || s.length() == 3)
                && s.charAt(0) == CN_NUMBER[1].charAt(0)
                && s.charAt(1) == CN_UNIT_1[1].charAt(0)
                ? s.substring(1) : s;
    }

    private String _0(String s) {
        return s.replace(CN_NUMBER[0] + CN_NUMBER[0] + CN_NUMBER[0], CN_NUMBER[0])
                .replace(CN_NUMBER[0] + CN_NUMBER[0], CN_NUMBER[0])
                .replaceAll(CN_NUMBER[0] + "+$", "");
    }

    private String[] _4(String n) {
        int l = n.length();
        int g = (l + 3) / 4;
        String[] r = new String[g];
        for (int i = 0; i < g; i++) {
            int e = l - i * 4;
            int s = Math.max(0, e - 4);
            r[i] = n.substring(s, e);
        }
        return r;
    }

    private String parseFloat() {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < fStr.length(); i++) {
            sb.append(CN_NUMBER[fStr.charAt(i) - '0']);
        }
        return sb.toString();
    }
}
```

# 部分测试结果

```
0=>零
1=>一
2=>二
3=>三
4=>四
5=>五
6=>六
7=>七
8=>八
9=>九
10=>十
11=>十一
12=>十二
13=>十三
14=>十四
15=>十五
16=>十六
17=>十七
18=>十八
19=>十九
20=>二十
21=>二十一
22=>二十二
23=>二十三
24=>二十四
25=>二十五
26=>二十六
27=>二十七
28=>二十八
29=>二十九
30=>三十
31=>三十一
50998=>五万零九百九十八
50999=>五万零九百九十九
51000=>五万一千
51001=>五万一千零一
51002=>五万一千零二
51003=>五万一千零三
51004=>五万一千零四
51005=>五万一千零五
51006=>五万一千零六
51007=>五万一千零七
51008=>五万一千零八
51009=>五万一千零九
51010=>五万一千零一十
99998=>九万九千九百九十八
99999=>九万九千九百九十九
100000=>十万
100001=>十万零一
100002=>十万零二
-1200001234=>负十二亿零一千二百三十四
-0.1234567890123=>负零点一二三四五六七八九零一二三
0.0001=>零点零零零一
-1002340005600.1000=>负一万亿零二十三亿四千万五千六百点一零零零
```
