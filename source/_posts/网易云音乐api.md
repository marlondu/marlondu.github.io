---
title: 网易云音乐api
date: 2017-05-06 21:58:04
comments: true
categories:
- 爬虫
tags:
- 爬虫
---

因为非常喜欢听网易云音乐，其社交化的功能真的非常的体验非常好，可以让有相同音乐爱好的人聚集在一起，一起互相交流。毕竟一边听歌一边看评论真的是非常的有意思。除此之外，其歌单功能，炫酷的界面等，可以称登上是良心之作。当然最有意思还是网友贡献的评论，有一段时间网易将得赞数最高的评论打印出来，印在地铁里，感动了无数人。

于是，就想写个爬虫，把他的数据爬下来，岂不美哉。于是动手研究一番。

## 关于加密

经过研究发现，为了防止爬虫，对于一些敏感的数据，它对传递的参数进行了加密。经过查找js, 打断点等，终于找到一些端倪。

其实过程比较简单，它每次的请求都是两个参数params 和 seckey。

- params: 是请求的加密之后的参数
- seckey: 用于解密参数的密钥

他的过程是这样的：

1. 随机生成一个16位长的key
2. 根据这个Key将参数用aes算法加密（还是两次）作为params
3. 然后键key使用rsa算法进行加密作为seckey

### js版本

进行加密的核心js代码如下:

```js
/**
* 生成密钥
* a: 要生成的密钥的长度, 一般是16
*/
function a(a) {
        var d, e, b = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789", c = "";
        for (d = 0; a > d; d += 1)
            e = Math.random() * b.length,
            e = Math.floor(e),
            c += b.charAt(e);
        return c
    }
/**
 * aes加密算法
 * a : 要加密的内容，就是明文的params
 * b : 使用的密钥key
 */
 function b(a, b) {
    var c = CryptoJS.enc.Utf8.parse(b)
    , d = CryptoJS.enc.Utf8.parse("0102030405060708")
    , e = CryptoJS.enc.Utf8.parse(a)
    , f = CryptoJS.AES.encrypt(e, c, {
      iv: d,
      mode: CryptoJS.mode.CBC
    });
    return f.toString()
 }
/**
 * rsa加密方法
 * a: 要加密的内容，就是a方法生成的16位key
 * b: pubkey
 * c: modulus
 */
    function c(a, b, c) {
        var d, e;
        return setMaxDigits(131),
        d = new RSAKeyPair(b,"",c),
        e = encryptedString(d, a)
    }
/**
 * 总的加密方法
 * d: 明文json串形式的参数
 * e: pubkey
 * f: modulus
 * g: nonce
 */
    function d(d, e, f, g) {
        var h = {}
          , i = a(16);
        return h.encText = b(d, g),
        h.encText = b(h.encText, i),
        h.encSecKey = c(i, e, f),
        h
    }
```

其中aes加密的部分依赖crypto-js加密库，rsa加密方法参见[这里](https://github.com/marlondu/my-notes/blob/master/scripts/js/netease_rsa_enc.js)。

### python版本

既然知道了是使用了aes 和rsa加密就好办了，而且网上也已经有人有过研究，并提供了python版本的加密方法：

```python
#!/usr/bin/enn python

import json
import os
import base64
import binascii

modulus = '00e0b509f6259df8642dbc35662901477df22677ec152b5ff68ace615bb7b725152b3ab17a876aea8a5aa76d2e417629ec4ee341f56135fccf695280104e0312ecbda92557c93870114af6c9d05c4f7f0c3685b7a46bee255932575cce10b424d813cfe4875d3e82047b97ddef52741d546b8e289dc6935b3ece0462db0a22b8e7'
nonce = '0CoJUm6Qyw8W8jud'
pubKey = '010001'

def createSecretKey(size):
	return (''.join(map(lambda xx: (hex(ord(xx))[2:]), os.urandom(size))))[0:16]

def aesEncrypt(text, secKey):
	pad = 16 - len(text) % 16
	text = text + pad * chr(pad)
	#print(text)
	encryptor = AES.new(secKey, 2, '0102030405060708')
	ciphertext = encryptor.encrypt(text)
	ciphertext = base64.b64encode(ciphertext)
	return ciphertext

def rsaEncrypt(text, pubKey, modulus):
	text = text[::-1]
	rs = int(binascii.hexlify(text),16) ** int(pubKey, 16) % int(modulus, 16)
	return format(rs, 'x').zfill(256)


def encrypted_request(text):
	text = json.dumps(text)
	secKey = createSecretKey(16)
	print(secKey)
	encText = aesEncrypt(aesEncrypt(text, nonce), secKey)
	encSecKey = rsaEncrypt(secKey, pubKey, modulus)
	print(encSecKey)
	data = {
		'params': encText,
		'encSecKey': encSecKey
	}
	return data

if __name__ == '__main__':
	params = "{'name':'sdsd'}"
    encParams = encrypted_request(params)
    print(encParams['encText'])
    print(encParams['encSeckey'])
```

代码来源: [网易云音乐新版WebAPI分析](https://github.com/darknessomi/musicbox/wiki/%E7%BD%91%E6%98%93%E4%BA%91%E9%9F%B3%E4%B9%90%E6%96%B0%E7%89%88WebAPI%E5%88%86%E6%9E%90%E3%80%82)

### Java版本

重点来了，由于很多人习惯用java来写爬虫，下面我提供一种java版本的：

```java
private static final String modulus = "00e0b509f6259df8642dbc35662901477df22677ec152b5ff68ace615bb7b725152b3ab17a876aea8a5aa76d2e417629ec4ee341f56135fccf695280104e0312ecbda92557c93870114af6c9d05c4f7f0c3685b7a46bee255932575cce10b424d813cfe4875d3e82047b97ddef52741d546b8e289dc6935b3ece0462db0a22b8e7",
            nonce = "0CoJUm6Qyw8W8jud",
            pubKey = "010001";
    private static final String AES_CBC_PADDING = "AES/CBC/PKCS5Padding";
    private static final Charset charset = Charset.forName("UTF-8");

    public static String createSecKey(int len) {
        final String b = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
        StringBuilder c = new StringBuilder();
        for (int d = 0; d < len; d++) {
            int e = (int) Math.floor(Math.random() * b.length());
            c.append(b.charAt(e));
        }
        return c.toString();
    }

    public static String aesEncrypt(String text, String secKey) {
        try {
            Cipher cipher = Cipher.getInstance(AES_CBC_PADDING);
            SecretKeySpec key = new SecretKeySpec(secKey.getBytes(charset), "AES");
            cipher.init(Cipher.ENCRYPT_MODE, key, new IvParameterSpec("0102030405060708".getBytes(charset)));
            try {
                byte[] data = cipher.doFinal(text.getBytes(charset));
                String result = Base64.getEncoder().encodeToString(data);
                return result;
            } catch (IllegalBlockSizeException | BadPaddingException e) {
                e.printStackTrace();
            }
        } catch (NoSuchAlgorithmException | NoSuchPaddingException | InvalidKeyException | InvalidAlgorithmParameterException e) {
            e.printStackTrace();
        }
        return null;
    }

   /**
     * 此方法计算量很大，极耗CPU
     * @param text
     * @return
     */
    public static String rsaEncrypt(String text){
        text = new StringBuilder(text).reverse().toString();
        BigInteger integer = new BigInteger(text.getBytes());
        BigInteger pubkeyInt = new BigInteger(pubKey, 16);
        BigInteger modulusInt = new BigInteger(modulus, 16);
        String result = integer.pow(pubkeyInt.intValue()).mod(modulusInt).toString(16);
        while(result.length() < 256)
            result = "0" + result;
        return result;
    }

    public static void main(String[] args) {
        String text = "{'name':'value'}";
        String secKey = createSecKey(16);
        String params = aesEncrypt(aesEncrypt(text, nonce), secKey);
        System.out.println("params:\n" + params);
        String encKey = rsaEncrypt(secKey);
        System.out.println("encKey:\n" + encKey);
    }
```

不过在使用过程中，我发现在进行rsa加密时，可能时计算量太大，使得cpu占用达到100%。不过还好，有一个非常好的解决办法，那就是使用**固定**的seckey和enckey就可以了，不需要每次都生成的新的seckey然后加密成enckey。【偷笑】

好了正文开始：

#### 1. 获取歌曲信息

- url : [http://music.163.com/song?id=[229383\]](http://music.163.com/song?id=[229383])
- params : 歌曲Id
- method : get
- return : Html / 包含歌曲名称，专辑，歌手

#### 2. 歌曲评论

- url : [http://music.163.com/weapi/v1/resource/comments/R_SO_4_[229383\]/?csrf_token=](http://music.163.com/weapi/v1/resource/comments/R_SO_4_[229383]/?csrf_token=)
- params: {"rid":"R_SO_4_229383","offset":"0","total":"true","limit":"20","csrf_token":""}
- 参数说明 : 需要加密
- method : post
- return : json / 
  {"code":200,"isMusician":false,"userId":-1,"topComments":[],"moreHot":false,"hotComments":[],"total":2323, "comments":[]}

#### 3. 歌手信息

- url : [http://music.163.com/artist?id=7652](http://music.163.com/artist?id=7652)
- method : get
- return : html / top50歌曲

#### 4. mv信息评论信息

- url : [http://music.163.com/weapi/v1/resource/comments/R_MV_5_[5331658\]/?csrf_token=](http://music.163.com/weapi/v1/resource/comments/R_MV_5_[5331658]/?csrf_token=)
- method : post
- 参数说明: 需要加密
- params : "{"rid":"R_MV_5_5331658","offset":"0","total":"true","limit":"20","csrf_token":""}"
- return : json /  
  {"isMusician":false,"userId":-1,"topComments":[],"moreHot":false,"hotComments":[],"code":200,"comments":[],"total":5,"more":false}

#### 5. 歌单信息
- url : [http://music.163.com/playlist?id=523664323](http://music.163.com/playlist?id=523664323)
- method : get
- return : html / 播放量，

#### 6. 歌单评论信息
- url : [http://music.163.com/weapi/v1/resource/comments/A_PL_0_523664323/?csrf_token=](http://music.163.com/weapi/v1/resource/comments/A_PL_0_523664323/?csrf_token=)
- params : {"rid":"A_PL_0_523664323","offset":"0","total":"true","limit":"20","csrf_token":""}
- 参数说明: 需要加密
- method : post
- return : json /