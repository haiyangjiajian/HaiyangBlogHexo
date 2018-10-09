---
layout: post
title: Android app重新签名和查看已有签名
tags: [android]
category: coding
---

有时候会需要将一个已经签名的Android apk重新签名。比如oppo软件商店的app认领，它会要求你下载一个空包，并将与认领应用一致的签名写入空包中。可以如下操作

<!-- more -->

---

### 删除原apk签名文件


```bash
mkdir test
mv TestSign.apk test
cd test
jar -xvf TestSign.apk   //解压apk
rm -rf META-INF		//删除META-INF
rm -rf TestSign.apk 	//删除原apk
jar -cvf ../TestSign.apk ./     //将当前文件夹中的内容打包成apk到外层文件夹
```


### 生成keystore
如果打算使用已有keystore，可以不生成，直接进行下一步

```bash
keytool -genkey -v -keystore test.keystore -alias test -keyalg RSA -validity 10000
```

### apk重新签名
最后的“test”与上面生成keystore制定的-alias要一致

```bash
jarsigner -verbose -keystore test.keystore -signedjar -TestSigned.apk TestSign.apk test
```

### 查看生成的apk的签名

```bash
mkdir test
mv TestSigned.apk test
cd test
jar -xvf TestSign.apk   //解压apk
keytool -printcert -file  META-INF/xxx.RSA
```








