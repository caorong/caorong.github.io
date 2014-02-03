---
layout: post
title: "intellij IDEA 13's licence generater source"
description: ""
category: "java"
tags: [idea, java ,crack]
---
{% include JB/setup %}

###注意：以下仅作为学习目的，如有任何问题，请mail我删除。

由于网上下载的keygen要求jdk7+ ，而我的mac只有1.6，so，decompile了。以下是licence code generate 部分的源码。
在main函数里重写自己的name，选择自己licence 过期时间。 

{% highlight java %} 
import java.math.BigInteger;
import java.util.Date;
import java.util.zip.CRC32;

/**
 * Created by caorong on 14-1-4.
 */
public class Idea13 {

    public static short getCRC(String s, int i, byte[] bytes) {
        CRC32 crc32 = new CRC32();
        if (s != null) {
            for (int j = 0; j < s.length(); j++) {
                char c = s.charAt(j);
                crc32.update(c);
            }
        }
        crc32.update(i);
        crc32.update(i >> 8);
        crc32.update(i >> 16);
        crc32.update(i >> 24);
        for (int k = 0; k < bytes.length - 2; k++) {
            byte byte0 = bytes[k];
            crc32.update(byte0);
        }
        return (short) (int) crc32.getValue();
    }

    public static String encodeGroups(BigInteger biginteger) {
        BigInteger beginner1 = BigInteger.valueOf(60466176L);
        StringBuilder sb = new StringBuilder();
        for (int i = 0; biginteger.compareTo(BigInteger.ZERO) != 0; i++) {
            int j = biginteger.mod(beginner1).intValue();
            String s1 = encodeGroup(j);
            if (i > 0) {
                sb.append("-");
            }
            sb.append(s1);
            biginteger = biginteger.divide(beginner1);
        }
        return sb.toString();
    }

    public static String encodeGroup(int i) {
        StringBuilder sb = new StringBuilder();
        for (int j = 0; j < 5; j++) {
            int k = i % 36;
            char c;
            // char c;
            if (k < 10) {
                c = (char) (48 + k);
            } else {
                c = (char) (65 + k - 10);
            }
            sb.append(c);
            i /= 36;
        }
        return sb.toString();
    }

    public static String MakeKey(String name, int days, int id,
                                 ProductType prtype) {
        id %= 10000;
        byte[] bkey = new byte[12];
        bkey[0] = ((byte) prtype.id());
        bkey[1] = 13;

        Date d = new Date();
        long ld = d.getTime() >> 16;
        bkey[2] = ((byte) (int) (ld & 0xFF));
        bkey[3] = ((byte) (int) (ld >> 8 & 0xFF));
        bkey[4] = ((byte) (int) (ld >> 16 & 0xFF));
        bkey[5] = ((byte) (int) (ld >> 24 & 0xFF));

        days &= 0xFFFF;
        bkey[6] = ((byte) (days & 0xFF));
        bkey[7] = ((byte) (days >> 8 & 0xFF));

        bkey[8] = 105;
        bkey[9] = -59;
        bkey[10] = 0;
        bkey[11] = 0;

        int w = getCRC(name, id % 10000, bkey);
        bkey[10] = ((byte) (w & 0xFF));
        bkey[11] = ((byte) (w >> 8 & 0xFF));

        BigInteger pow = new BigInteger(
                "89126272330128007543578052027888001981", 10);
        BigInteger mod = new BigInteger("86f71688cdd2612ca117d1f54bdae029", 16);
        BigInteger k0 = new BigInteger(bkey);
        BigInteger k1 = k0.modPow(pow, mod);
        String s0 = Integer.toString(id);
        String sz = "0";
        while (s0.length() != 5) {
            s0 = sz.concat(s0);
        }
        s0 = s0.concat("-");

        String s1 = encodeGroups(k1);
        s0 = s0.concat(s1);
        return s0;
    }

    public enum ProductType {
        IDEA(1);

        private int id;

        private ProductType(int id) {
            this.id = id;
        }

        public int id() {
            return this.id;
        }

        public static ProductType getProductType(String str) {
            for (ProductType product : ProductType.values()) {
                if (product.toString().equalsIgnoreCase(str)) {
                    return product;
                }
            }
            return null;
        }
    }

    public static void main(String[] args) {
        // your name  and  licence time (day)
        // don't know what id means  type 1 and it works
        System.out.println(Idea13.MakeKey("your name here", 365 * 3, 1, ProductType.IDEA));
    }
}

{% endhighlight %}