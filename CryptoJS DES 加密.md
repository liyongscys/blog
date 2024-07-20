---
date: 2024-07-19
title: 为什么 CryptoJS DES 加密的结果和 Java DES 不一样
category: 
  - 前端
tag: 
  - 前端
  - 加密
---



用CryptoJS库, 对数据做DES算法的加密/解密  
首选查看官方示例, 将密文进行Base64编码, 掉进一个大坑

```javascript
<script src="http://crypto-js.googlecode.com/svn/tags/3.1.2/build/rollups/tripledes.js"></script>
<script>
    var encrypted = CryptoJS.DES.encrypt("Message", "Secret Passphrase");
    // ciphertext changed every time you run it
    // 加密的结果不应该每次都是一样的吗?
    console.log(encrypted.toString(), encrypted.ciphertext.toString(CryptoJS.enc.Base64));
    var decrypted = CryptoJS.DES.decrypt(encrypted, "Secret Passphrase");
    console.log(decrypted.toString(CryptoJS.enc.Utf8));
</script>
```

对这些加密算法不了解, 只能求助Google  
des encrypion: js encrypted value does not match the java encrypted value  
In cryptoJS you have to convert the key to hex and useit as word just like above (otherwise it will be considered as passphrase)
For the key, when you pass a string, it's treated as a passphrase and used to derive an actual key and IV. Or you can pass a WordArray that represents the actual key.  
原来是我指定key的方式不对, 直接将字符串做为参数, 想当然的以为这就是key, 其实不然, CryptoJS会根据这个字符串算出真正的key和IV(各种新鲜名词不解释, 问我也没用, 我也不懂 -_-")  
那么我们只需要将key和iv对应的字符串转成CryptoJS的WordArray类型, 在DES加密时做为参数传入即可, 这样对Message这个字符串加密, 每次得到的密文都是YOa3le0I+dI=   

```js
    var keyHex = CryptoJS.enc.Utf8.parse('abcd1234');
    var ivHex = CryptoJS.enc.Utf8.parse('inputvec');
    var encrypted = CryptoJS.DES.encrypt('Message', keyHex, { iv: ivHex });
```

这样是不是就万事OK了? 哪有, 谁知道这坑是一个接一个啊.  
我们再试试Java这边的DES加密是不是和这个结果一样, 具体实现请参考Simple Java Class to DES Encrypt Strings  
果真掉坑里了, Java通过DES加密Message这个字符串得到的结果是8dKft9vkZ4I=和CryptoJS算出来的不一样啊...亲  
继续求助Google  
C# and Java DES Encryption value are not identical  
SunJCE provider uses ECB as the default mode, and PKCS5Padding as the default padding scheme for DES.(JCA Doc)  
This means that in the case of the SunJCE provider,  
    Cipher c1 = Cipher.getInstance("DES/ECB/PKCS5Padding");  
and
    Cipher c1 = Cipher.getInstance("DES");  
are equivalent statements.  
原来是CryptoJS进行DES加密时, 默认的模式和padding方式和Java默认的不一样造成的, 必须使用ECB mode和PKCS5Padding, 但是CryptoJS中只有Pkcs7, 不管了, 试试看... 

```js
<script src="http://crypto-js.googlecode.com/svn/tags/3.1.2/build/rollups/tripledes.js"></script>
<script src="http://crypto-js.googlecode.com/svn/tags/3.1.2/build/components/mode-ecb.js"></script>

<script>
    var keyHex = CryptoJS.enc.Utf8.parse('abcd1234');
    var encrypted = CryptoJS.DES.encrypt('Message', keyHex, {
        mode: CryptoJS.mode.ECB,
        padding: CryptoJS.pad.Pkcs7
    });
    console.log(encrypted.toString(), encrypted.ciphertext.toString(CryptoJS.enc.Base64));
</script>
```

咦...使用Pkcs7能得到和Java DES一样的结果了, 哇塞...好神奇  
那我们试试统一Java也改成Cipher.getInstance("DES/ECB/PKCS7Padding")试试, 结果得到一个大大的错误  
Error:java.security.NoSuchAlgorithmException: Cannot find any provider supporting DES/ECB/PKCS7Padding  
没办法, 继续Google  
java.security.NoSuchAlgorithmException: Cannot find any provider supporting AES/ECB/PKCS7PADDING  
I will point out that PKCS#5 and PKCS#7 actually specify exactly the same type of padding (they are the same!), but it's called #5 when used in this context. :)  
这位大侠给出的解释是: PKCS#5和PKCS#7是一样的padding方式, 对加密算法一知半解, 我也只能暂且认可这个解释了.  
忙完了DES的加密, 接下来就是使用CryptoJS来解密了. 

我们需要直接解密DES加密后的base64密文字符串. CryptoJS好像没有提供直接解密DES密文字符串的方法啊, 他的整个加密/解密过程都是内部自己在玩, 解密时需要用到加密的结果对象, 这不是坑我吗?  
只好研究下CryptoJS DES加密后返回的对象, 发现有一个属性ciphertext, 就是密文的WordArray, 那么解密的时候, 我们是不是只要提供这个就行了呢?

```js
    var keyHex = CryptoJS.enc.Utf8.parse('abcd1234');
    // direct decrypt ciphertext
    var decrypted = CryptoJS.DES.decrypt({
        ciphertext: CryptoJS.enc.Base64.parse('8dKft9vkZ4I=')
    }, keyHex, {
        mode: CryptoJS.mode.ECB,
        padding: CryptoJS.pad.Pkcs7
    });
    console.log(decrypted.toString(CryptoJS.enc.Utf8));
```

果不其然, 到此为止, 问题全部解决, 豁然开朗...  
完整代码请参考CryptoJS-DES.html  
Use CryptoJS encrypt message by DES and direct decrypt ciphertext, compatible with Java Cipher.getInstance("DES")  



来源

https://www.douban.com/note/276592520/?_i=1282930_e1ZIwh
