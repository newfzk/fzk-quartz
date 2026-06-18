---
title: 数据加密 — AES-GCM 模式
date: 2026-05-26
updated: 2026-06-04
aliases:
  - AES-GCM
  - 对称加密
  - SM4
related:
  - "[[密钥管理策略]]"
  - "[[数据加密与密钥管理]]"
tags:
  - language/java
  - topic/系统设计
  - topic/安全
status: to-review
---

# 数据加密 — AES-GCM 模式

## AES-GCM 模式

### 推荐用于用户敏感数据加密（如手机号、收货地址）

```java
import javax.crypto.Cipher;
import javax.crypto.spec.GCMParameterSpec;
import javax.crypto.spec.SecretKeySpec;
import java.security.SecureRandom;

public class AesGcmEncryption {
    private static final int GCM_IV_LENGTH = 12;  // 推荐12字节
    private static final int GCM_TAG_LENGTH = 128;  // 认证标签长度
    
    // 加密
    public byte[] encrypt(byte[] data, byte[] key) throws Exception {
        byte[] iv = new byte[GCM_IV_LENGTH];
        SecureRandom random = new SecureRandom();
        random.nextBytes(iv);
        
        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        GCMParameterSpec spec = new GCMParameterSpec(GCM_TAG_LENGTH, iv);
        cipher.init(Cipher.ENCRYPT_MODE, new SecretKeySpec(key, "AES"), spec);
        
        byte[] encrypted = cipher.doFinal(data);
        
        // IV + 密文
        byte[] result = new byte[iv.length + encrypted.length];
        System.arraycopy(iv, 0, result, 0, iv.length);
        System.arraycopy(encrypted, 0, result, iv.length, encrypted.length);
        return result;
    }
    
    // 解密
    public byte[] decrypt(byte[] encryptedData, byte[] key) throws Exception {
        byte[] iv = new byte[GCM_IV_LENGTH];
        byte[] ciphertext = new byte[encryptedData.length - GCM_IV_LENGTH];
        System.arraycopy(encryptedData, 0, iv, 0, GCM_IV_LENGTH);
        System.arraycopy(encryptedData, GCM_IV_LENGTH, ciphertext, 0, ciphertext.length);
        
        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        GCMParameterSpec spec = new GCMParameterSpec(GCM_TAG_LENGTH, iv);
        cipher.init(Cipher.DECRYPT_MODE, new SecretKeySpec(key, "AES"), spec);
        
        return cipher.doFinal(ciphertext);
    }
}
```

### AES-GCM 优势

| 特性 | 说明 |
|------|------|
| **加密+认证** | 自带消息认证码（MAC），防止篡改 |
| **安全性高** | 防止填充Oracle攻击，比CBC模式更安全 |
| **效率高** | 并行加密，性能优于其他模式 |

### IV（初始化向量）要点

- **唯一性**：每个加密操作必须使用唯一IV
- **随机性**：使用`SecureRandom`生成
- **存储**：IV与密文一起存储（不需要保密）
- **长度**：推荐12字节（96位），兼顾安全性和性能

---

## 国密算法支持（SM4）

**国内合规要求时使用**

```java
import org.bouncycastle.jce.provider.BouncyCastleProvider;
import javax.crypto.Cipher;
import javax.crypto.spec.GCMParameterSpec;
import javax.crypto.spec.SecretKeySpec;
import java.security.Security;

public class SM4Encryption {
    static {
        Security.addProvider(new BouncyCastleProvider());
    }
    
    public byte[] encryptSM4Gcm(byte[] data, byte[] key, byte[] iv) throws Exception {
        Cipher cipher = Cipher.getInstance("SM4/GCM/NoPadding", "BC");
        GCMParameterSpec spec = new GCMParameterSpec(128, iv);
        cipher.init(Cipher.ENCRYPT_MODE, new SecretKeySpec(key, "SM4"), spec);
        return cipher.doFinal(data);
    }
}
```

---

## 对称加密 vs 非对称加密

| 概念 | 说明 |
|------|------|
| **对称加密** | 加密解密使用相同密钥（AES、SM4） |
| **非对称加密** | 加密解密使用不同密钥（RSA） |

---

## 面试要点

### 常见面试问题

**Q：为什么推荐AES-GCM而不是CBC？**

A：GCM模式自带认证功能，能检测数据是否被篡改，且避免了CBC模式的填充Oracle攻击风险。

## 参考链接

- [[密钥管理策略]] — 密钥分层、轮换与 KMS 管理
- [[数据加密与密钥管理]] — 加密与密钥管理知识地图（MOC）
- [瑞幸后端-Q4-用户数据加密存储与密钥管理](../20-Questions/瑞幸后端-Q4-用户数据加密存储与密钥管理.md)