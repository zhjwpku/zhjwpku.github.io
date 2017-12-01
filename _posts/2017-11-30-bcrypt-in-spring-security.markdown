---
layout: post
title: Spring Security 中的 Bcrypt
date: 2017-11-30 20:00:00 +0800
tags:
- bcrypt
- algorithm
---

最近在写用户管理相关的微服务，其中比较重要的问题是如何保存用户的密码，加盐哈希是一种常见的做法。知乎上有个问题大家可以先读一下: [加盐密码保存的最通用方法是？](https://www.zhihu.com/question/20299384)

对于每个用户的密码，都应该使用独一无二的盐值，每当新用户注册或者修改密码时，都应该使用新的盐值进行加密，并且这个盐值应该足够长，使得有足够的盐值以供加密。随着彩虹表的出现及不断增大，MD5算法不建议再使用了。

**存储密码的步骤**

1. 使用基于加密的伪随机数生成器（Cryptographically Secure Pseudo-Random Number Generator – CSPRNG）生成一个足够长的盐值，如Java中的[java.security.SecureRandom](https://docs.oracle.com/javase/8/docs/api/java/security/SecureRandom.html)
2. 将盐值混入密码(常用的做法是置于密码之前)，并使用标准的加密哈希函数进行加密，如SHA256
3. 把哈希值和盐值一起存入数据库中对应此用户的那条记录

**校验密码的步骤**

1. 从数据库取出用户的密码哈希值和对应盐值
2. 将盐值混入用户输入的密码，并使用相同的哈希函数进行加密
3. 比较上一步结果和哈希值是否相同，如果相同那么密码正确，反之密码错误

加盐使攻击者无法采用特定的查询表或彩虹表快速破解大量哈希值，但不能阻止字典攻击或暴力攻击。这里假设攻击者已经获取到用户数据库，意味着攻击者知道每个用户的盐值，根据[Kerckhoffs's
principle](https://en.wikipedia.org/wiki/Kerckhoffs%27s_principle)，应该假设攻击者知道用户系统使用密码加密算法，如果攻击者使用高端GPU或定制的[ASIC](https://en.wikipedia.org/wiki/Application-specific_integrated_circuit)，每秒可以进行数十亿次哈希计算，针对每个用户进行字典查询的效率依旧很高效。

为了降低这类攻击，可以用一种叫做**密钥扩展**的技术，让哈希函数变得很慢，即使GPU或ASIC字典攻击或暴力攻击也会慢得让攻击者无法接受。密钥扩展的实现依赖一种CPU密集型哈希函数，如[PBKDF2](https://en.wikipedia.org/wiki/PBKDF2)和本文将要介绍的[Bcrypt](https://en.wikipedia.org/wiki/Bcrypt)。这类函数使用一个安全因子或迭代次数作为参数，该值决定了哈希函数会有多慢。

**Bcrypt**

Bcrypt是由Niels Provos和DavidMazières基于Blowfish密码设计的一种密码散列函数，于1999年在USENIX上发布。

wikipedia上Bcrypt词条中有该算法的伪代码:

```
Function bcrypt
    Input:
        cost:     Number (4..31)                // 该值决定了密钥扩展的迭代次数 Iterations = 2^cost
        salt:     array of Bytes (16 bytes)     // 随机数
        password: array of Bytes (1..72 bytes)  // 用户密码
    Output:
        hash:     array of Bytes (24 bytes)     // 返回的哈希值

    // 使用Expensive key setup算法初始化Blowfish状态
    state <- EksBlowfishSetup(cost, salt, password)     // 这一步是整个算法中最耗时的步骤

    ctext <- "OrpheanBeholderScryDoubt"     // 24 bytes，初始向量
    repeat (64)
        ctext <- EncryptECB(state, ctext)   // 使用 blowfish 算法的ECB模式进行加密

    return Concatenate(cost, salt, ctext)

// Expensive key setup
Function EksBlowfishSetup
    Input:
        cost:     Number (4..31)
        salt:     array of Bytes (16 bytes)
        password: array of Bytes (1..72 bytes)
    Output:
        state:    opaque BlowFish state structure

    state <- InitialState()
    state <- ExpandKey(state, salt, password)
    repeat (2 ^ cost)           // 计算密集型
        state <- ExpandKey(state, 0, password)
        state <- ExpandKey(state, 0, salt)

    return state

Function ExpandKey(state, salt, password)
    Input:
        state:    Opaque BlowFish state structure  // 内部包含 P-array 和 S-box
        salt:     array of Bytes (16 bytes)
        password: array of Bytes (1..72 bytes)
    Output:
        state:    Opaque BlowFish state structure

    // ExpandKey 是对输入参数进行固定的移位异或等运算，这里不列出
```

通过伪代码可以看出，通过制定不同的cost值，可以使得EksBlowfishSetup的运算次数大幅提升，从而达到慢哈希的目的。

**Spring Security 中的 Bcrypt**

理解了Bcrypt算法的原理，再来看Spring Security中的实现就很简单了。

```java
package org.springframework.security.crypto.bcrypt;

...省略import...

public class BCryptPasswordEncoder implements PasswordEncoder {
    ...省略log...

    private final int strength;         // 相当于wiki伪代码中的cost，默认为10

    private final SecureRandom random;  // CSPRNG

    // 构造函数
    public BCryptPasswordEncoder() {
        this(-1);
    }

    // 相当于伪代码中的cost, 长度 4 ~ 31
    public BCryptPasswordEncoder(int strength) {
        this(strength, null);
    }

    // 构造函数
    public BCryptPasswordEncoder(int strength, SecureRandom random) {
        if (strength != -1 && (strength < BCrypt.MIN_LOG_ROUNDS || strength > BCrypt.MAX_LOG_ROUNDS)) {
            throw new IllegalArgumentException("Bad strength");
        }
        this.strength = strength;
        this.random = random;
    }

    // 加密函数
    public String encode(CharSequence rawPassword) {
        String salt;
        if (strength > 0) {
            if (random != null) {
                salt = BCrypt.gensalt(strength, random);
            }
            else {
                salt = BCrypt.gensalt(strength);
            }
        }
        else {
            salt = BCrypt.gensalt();
        }
        return BCrypt.hashpw(rawPassword.toString(), salt);
    }

    // 密码匹配函数
    public boolean matches(CharSequence rawPassword, String encodedPassword) {
        if (encodedPassword == null || encodedPassword.length() == 0) {
            logger.warn("Empty encoded password");
            return false;
        }

        if (!BCRYPT_PATTERN.matcher(encodedPassword).matches()) {
            logger.warn("Encoded password does not look like BCrypt");
            return false;
        }

        return BCrypt.checkpw(rawPassword.toString(), encodedPassword);
    }
}

```

```java
package org.springframework.security.crypto.bcrypt;

public class BCrypt {

    // 生成盐值的函数 "$2a$" + 2字节log_round + "$" + 22字节随机数Base64编码
    public static String gensalt(int log_rounds, SecureRandom random) {
        if (log_rounds < MIN_LOG_ROUNDS || log_rounds > MAX_LOG_ROUNDS) {
            throw new IllegalArgumentException("Bad number of rounds");
        }
        StringBuilder rs = new StringBuilder();
        byte rnd[] = new byte[BCRYPT_SALT_LEN];

        random.nextBytes(rnd);

        rs.append("$2a$");
        if (log_rounds < 10) {
            rs.append("0");
        }
        rs.append(log_rounds);
        rs.append("$");
        encode_base64(rnd, rnd.length, rs);
        return rs.toString();   // 总长度29字节
    }

    /**
     * Hash a password using the OpenBSD bcrypt scheme
     * @param password the password to hash
     * @param salt the salt to hash with (perhaps generated using BCrypt.gensalt)
     * @return the hashed password
     * @throws IllegalArgumentException if invalid salt is passed
     */
    public static String hashpw(String password, String salt) throws IllegalArgumentException {
        // 该函数在验证阶段也会用到，因为前29字节为盐值，所以可以将之前计算过的密码哈希值做为盐值
        BCrypt B;
        String real_salt;
        byte passwordb[], saltb[], hashed[];
        char minor = (char) 0;
        int rounds, off = 0;
        StringBuilder rs = new StringBuilder();

        if (salt == null) {
            throw new IllegalArgumentException("salt cannot be null");
        }

        int saltLength = salt.length();

        if (saltLength < 28) {
            throw new IllegalArgumentException("Invalid salt");
        }

        if (salt.charAt(0) != '$' || salt.charAt(1) != '2') {
            throw new IllegalArgumentException("Invalid salt version");
        }
        if (salt.charAt(2) == '$') {
            off = 3;
        }
        else {
            minor = salt.charAt(2);
            if (minor != 'a' || salt.charAt(3) != '$') {
                throw new IllegalArgumentException("Invalid salt revision");
            }
            off = 4;
        }

        if (saltLength - off < 25) {
            throw new IllegalArgumentException("Invalid salt");
        }

        // Extract number of rounds
        if (salt.charAt(off + 2) > '$') {
            throw new IllegalArgumentException("Missing salt rounds");
        }
        rounds = Integer.parseInt(salt.substring(off, off + 2));

        real_salt = salt.substring(off + 3, off + 25);
        try {
            passwordb = (password + (minor >= 'a' ? "\000" : "")).getBytes("UTF-8");
        }
        catch (UnsupportedEncodingException uee) {
            throw new AssertionError("UTF-8 is not supported");
        }

        // 解析成16字节的字节数组
        saltb = decode_base64(real_salt, BCRYPT_SALT_LEN);

        // 这里new了一个新的对象是因为会用到BCrypt中int P[]和 int S[]，扩展的密钥存放在这两个结构体
        B = new BCrypt();
        hashed = B.crypt_raw(passwordb, saltb, rounds);

        rs.append("$2");
        if (minor >= 'a') {
            rs.append(minor);
        }
        rs.append("$");
        if (rounds < 10) {
            rs.append("0");
        }
        rs.append(rounds);
        rs.append("$");
        encode_base64(saltb, saltb.length, rs);
        encode_base64(hashed, bf_crypt_ciphertext.length * 4 - 1, rs);
        return rs.toString();
    }

    /**
     * Perform the central password hashing step in the bcrypt scheme
     * @param password the password to hash
     * @param salt the binary salt to hash with the password
     * @param log_rounds the binary logarithm of the number of rounds of hashing to apply
     * @return an array containing the binary hashed password
     */
    private byte[] crypt_raw(byte password[], byte salt[], int log_rounds) {
        int cdata[] = (int[]) bf_crypt_ciphertext.clone();
        int clen = cdata.length;
        byte ret[];

        // rounds = 1 << log_round
        long rounds = roundsForLogRounds(log_rounds);

        init_key();
        ekskey(salt, password);
        for (long i = 0; i < rounds; i++) { // 最耗时的密钥扩展
            key(password);
            key(salt);
        }

        for (int i = 0; i < 64; i++) {
            for (int j = 0; j < (clen >> 1); j++) {
                encipher(cdata, j << 1);
            }
        }

        ret = new byte[clen * 4];
        for (int i = 0, j = 0; i < clen; i++) {
            ret[j++] = (byte) ((cdata[i] >> 24) & 0xff);
            ret[j++] = (byte) ((cdata[i] >> 16) & 0xff);
            ret[j++] = (byte) ((cdata[i] >> 8) & 0xff);
            ret[j++] = (byte) (cdata[i] & 0xff);
        }
        return ret;
    }

    /**
     * Check that a plaintext password matches a previously hashed one
     * @param plaintext the plaintext password to verify
     * @param hashed the previously-hashed password
     * @return true if the passwords match, false otherwise
     */
    public static boolean checkpw(String plaintext, String hashed) {
        return equalsNoEarlyReturn(hashed, hashpw(plaintext, hashed));
    }

    static boolean equalsNoEarlyReturn(String a, String b) {
        char[] caa = a.toCharArray();
        char[] cab = b.toCharArray();

        if (caa.length != cab.length) {
            return false;
        }

        byte ret = 0;
        for (int i = 0; i < caa.length; i++) {
            ret |= caa[i] ^ cab[i];
        }
        return ret == 0;
    }
}
```

**性能**

慢哈希既要防止攻击者无法使用暴力破击，又不能影响用户体验，由于机器性能的差异，获取强度参数最好的办法就是执行一个简短的性能基准测试，找到使哈希函数大约耗费0.5秒的值。

```java
public class BCryptBench {
    public static void main(String[] args) {
        long startTime, endTime, duration;

        // the default strength 10
        BCryptPasswordEncoder bCryptPasswordEncoder10 = new BCryptPasswordEncoder();
        duration = 0;

        for (int i = 0; i < 10; i++) {
            startTime = System.currentTimeMillis();
            System.out.println(bCryptPasswordEncoder10.encode("admin"));
            endTime = System.currentTimeMillis();
            duration += (endTime - startTime);
        }

        System.out.println(duration / 10.0);    // 88.4ms

        // strength 11
        // the default strength 10
        BCryptPasswordEncoder bCryptPasswordEncoder11 = new BCryptPasswordEncoder(11);
        duration = 0;

        for (int i = 0; i < 10; i++) {
            startTime = System.currentTimeMillis();
            System.out.println(bCryptPasswordEncoder11.encode("admin"));
            endTime = System.currentTimeMillis();
            duration += (endTime - startTime);
        }

        System.out.println(duration / 10.0);    // 175.3ms

        // strength 12
        BCryptPasswordEncoder bCryptPasswordEncoder12 = new BCryptPasswordEncoder(12);
        duration = 0;

        for (int i = 0; i < 10; i++) {
            startTime = System.currentTimeMillis();
            System.out.println(bCryptPasswordEncoder12.encode("admin"));
            endTime = System.currentTimeMillis();
            duration += (endTime - startTime);
        }

        System.out.println( duration / 10.0);   // 344.3ms

        // strength 13
        BCryptPasswordEncoder bCryptPasswordEncoder13 = new BCryptPasswordEncoder(13);
        duration = 0;

        for (int i = 0; i < 10; i++) {
            startTime = System.currentTimeMillis();
            System.out.println(bCryptPasswordEncoder13.encode("admin"));
            endTime = System.currentTimeMillis();
            duration += (endTime - startTime);
        }

        System.out.println(duration / 10.0);    // 703.8ms

        // strength 14
        BCryptPasswordEncoder bCryptPasswordEncoder14 = new BCryptPasswordEncoder(14);
        duration = 0;

        for (int i = 0; i < 10; i++) {
            startTime = System.currentTimeMillis();
            System.out.println(bCryptPasswordEncoder14.encode("admin"));
            endTime = System.currentTimeMillis();
            duration += (endTime - startTime);
        }

        System.out.println(duration / 10.0);    // 1525.0ms

        // strength 15
        BCryptPasswordEncoder bCryptPasswordEncoder15 = new BCryptPasswordEncoder(15);
        duration = 0;

        for (int i = 0; i < 10; i++) {
            startTime = System.currentTimeMillis();
            System.out.println(bCryptPasswordEncoder15.encode("admin"));
            endTime = System.currentTimeMillis();
            duration += (endTime - startTime);
        }

        System.out.println(duration / 10.0);    // 2921.9ms
    }
}
```

从测试的结果可以看出，如果想选定一个执行时间为0.5秒的慢哈希，需要将Bcrypt函数的强度设置为12或13。而在我们自己的微服务中，使用了默认的强度10。

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Salted Password Hashing - Doing it Right][hasing-security]<br>
2 [Salted Password Hashing - Doing it Right 译文][61872]
</span>

[61872]: http://blog.jobbole.com/61872/
[hasing-security]: https://crackstation.net/hashing-security.htm
