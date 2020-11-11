> 文章以及资料（开源）：[github地址](https://github.com/mindcarver/blockchain_guide)  

[TOC]


## 密码

首先明白密码不是私钥，它是在创建账户时候的密码（**注意可以修改**）。密码在以下情况下会使用到：

1. 作为转账的支付密码
2. 用keystore导入钱包的时候需要输入的密码，用来解锁keystore的

## 私钥

私钥由64位长度的十六进制的字符组成，比如：`0xA4356E49C88C8B7AB370AF7D5C0C54F0261AAA006F6BDE09CD4745CF54E0115A`，**一个账户只有一个私钥且不能修改**，。通常一个钱包中私钥和公钥是成对出现的，有了私钥，我们就可以通过一定的算法生成公钥，再通过公钥经过一定的算法生成地址，这一过程都是不可逆的。私钥一定要妥善保管，若被泄漏别人可以通过私钥解锁账号转出你的该账号的数字货币。

## keystore

Keystore常见于以太坊钱包，它是将私钥以加密的方式保存为一份 JSON 文件，这份 JSON 文件就是 keystore，所以它就是加密后的私钥。Keystore必须配合钱包密码才能使用该账号。

Keystore 文件大致样子：

```
{
    "crypto" : {
        "cipher" : "aes-128-ctr",
        "cipherparams" : {
            "iv" : "83dbcc02d8ccb40e466191a123791e0e"
        },
        "ciphertext" : "d172bf743a674da9cdad04534d56926ef8358534d458fffccd4e6ad2fbde479c",
        "kdf" : "scrypt",
        "kdfparams" : {
            "dklen" : 32,
            "n" : 262144,
            "r" : 1,
            "p" : 8,
            "salt" : "ab0c7876052600dd703518d6fc3fe8984592145b591fc8fb5c6d43190334ba19"
        },
        "mac" : "2103ac29920d71da29f15d75b4a16dbe95cfd7ff8faea1056c33131d846e3097"
    },
    "id" : "3198bc9c-6672-5ab3-d995-4942343ae5b6",
    "version" : 3
}
```

名词解释：

- cipher：对称 AES 算法的名称;
- cipherparams：上述 cipher 算法需要的参数;
- ciphertext：你的以太坊私钥使用上述 cipher 算法进行加密;
- kdf：密钥生成函数，用于让你用密码加密 keystore 文件;
- kdfparams：上述 kdf 算法需要的参数;
- Mac：用于验证密码的代码。

## 助记词

私钥是64位长度的十六进制的字符，不利于记录且容易记错，所以用算法将一串随机数转化为了一串12 ~ 24个容易记住的单词，方便保存记录。注意：

1. 助记词是私钥的另一种表现形式
2. 助记词可以获取相关联的多个私钥，反过来私钥没法获取助记词。

要弄清楚助记词与私钥的关系，得清楚BIP协议，是`Bitcoin Improvement Proposals`的缩写，意思是Bitcoin 的改进建议，用于提出 Bitcoin 的新功能或改进措施。BIP协议衍生了很多的版本，主要有BIP32、BIP39、BIP44。

### 以太坊对BIP的支持

BIP是用于提出 Bitcoin 的新功能或改进措施，那么对于以太坊来说如何支持呢？

- 以太坊在[EIPs/issues/84](https://github.com/ethereum/EIPs/issues/84)中讨论，是否遵循 BIP32 和 BIP44，社区里提出来很多有意思的观点，比特币是基于 UTXO 的，所以可以使用 HD 钱包（BIP32）为每个交易分配一个新地址，以保护您的隐私。然而，以太坊是基于帐户，每个帐户都有一个地址，BIP 是比特币的提案，而且比特币的数据结构的设计是围绕改变地址的想法构建的，BIP 的一些提案可能并不适合以太坊。以太坊的模式和比特币UTXO 不同，以太坊转账不能改变地址，如果在以太坊上实现 UTXO ，用户还必须签名两个交易以将余额的一部分发送到一个地址，将余额的一部分发送到第二个地址 - 这将使成本增加一倍，而且第二个交易可能不会在同一个区块中，当然以太坊也可以通过智能合约的方式实现。另外，以太坊目前官方钱包采用 KDF 的形式，也就是我们常说的 Keystore 的形式。
- 以太坊在[EIPs/issues/85](https://github.com/ethereum/EIPs/issues/85)中讨论，以太坊社区似乎也采用了 BIP32 的做法，提议 HD 路径为 : `m/44'/60'/0'/0/n`，n 是第 n 次生成地址。目前以太坊客户端实现了BIP32的客户端有：`Jaxx, Metamask, Exodus, imToken, TREZOR (ETH) & Digital Bitbox`。

## 密码、私钥、keystore与助记词的关系

![image-20201012091537816](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjma8otsfxj314s0s0jvw.jpg)

## 如何解锁账户

解锁账户有如下几种方式：

- 私钥（Private Key）
- Keystore+密码（Keystore+Password）
- 助记词（Mnemonic code）

我们可以得到以下总结：

- 通过私钥+密码可以生成keystore，即加密私钥。
- 通过keystore+密码可以获取私钥，即解密keystore。
- 通过助记词根据不同的路径获取不同的私钥，即使用HD钱包将助记词转化成种子来生成主私钥，然后派生海量的子私钥和地址。