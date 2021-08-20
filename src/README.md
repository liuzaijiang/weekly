## 开发必懂的文件加解密？？

> **本文作者：[coolyuantao](https://juejin.cn/user/1047150053304157)**

![encrypt](./encrypt.png)

### 背景

最近团队遇到一个小需求，存在两个系统 A、B，系统 A 支持用户在线制作皮肤包，制作后的皮肤包用户可以下载后，导入到另外的系统 B 上。皮肤包本身的其实就是一个 zip 压缩包，系统 B 接收到压缩包后，**解压**并做一些常规的校验，比如版本、内容合法性校验等，整体功能也比较简单。

但没想到啊，一帮测试人员对我们开发人员一顿输出，首先绕过系统 A 搞了几个视频文件，把后缀改成 `zip` 就直接想上传，系统 B 每次都是等到上传完后才发现文件不合法，系统 B 在文件没上传完前又无法解压，也不知道文件内容是不是合法的，就这么消耗了大量带宽、大量时间后才提示用户皮肤包有问题。

这里涉及了两个问题，我们来捋一捋：
1. 文件如何做加密，这样用户便无法去逆向，压缩包内部的敏感信息不会泄露出去。
2. 服务端在接收到信息流时，在未传输完时如何去判断压缩包的合法性，提前告知用户。

## AES VS RSA

说到加密，自己很多人会想到[对称算法 AES](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard) 以及[非对称算法 RSA](https://en.wikipedia.org/wiki/RSA_(cryptosystem))。这两种算法按字面意思也较好理解，对称加密技术说白一点就是加密跟解密使用的是同一个密钥，这种加密算法速度极快，安全级别高，加密前后的大小一致；非对称加密技术则有`公钥PK`、`私钥SK`，算法的原理在于寻找两个素数，让他们的乘积刚好等于一个约定的数字，非对称算法的安全性是依赖于大数的分解，这个目前没有理论支持可以快速破解，它的安全性完全依赖于这个密钥的长度，一般用 1024 位已经足够使用。但是它的速度相比对称算法慢得多，一般仅用于少量数据的加密，**待加密的数据长度不能超过密钥的长度**。

## 使用 AES 对文件加密

结合这两种加密方式的优缺点，我们采用 AES 对文件本身做加解密，使用 AES 的原因主要考虑如下：
1. 加解密性能问题，AES 的速度极快，相比 RSA 有 1000 倍以上提升。
2. RSA 对源文有长度的要求，最大长度仅有密钥长度。

AES 的加密算法 Node.js 的[crypto](http://nodejs.cn/api/crypto.html)模块中已经有内置，具体的使用可以参考官方文档。

### AES 加密逻辑

```javascript
const crypto = require('crypto');
const algorithm = 'aes-256-gcm';

/**
 * 对一个buffer进行AES加密
 * @param {Buffer} buffer   待加密的内容
 * @param {String} key      密钥
 * @param {String} iv       初始向量
 * @return {{key: string, iv: string, tag: Buffer, context: Buffer}}
 */
function aesEncrypt (buffer, key, iv) {
    // 初始化加密算法
    const cipher = crypto.createCipheriv(algorithm, key, iv);
    let encrypted = cipher.update(buffer);
    let end = cipher.final();
    // 生成身份验证标签，用于验证密文的来源
    const tag = cipher.getAuthTag();
    return {
        key,
        iv,
        tag,
        buffer: buffer.concat([encrypted, end]);
    };
}
```

### AES 解密逻辑
解密整体跟加密一样，只是接口换个名字即可：

```javascript
const crypto = require('crypto');
const algorithm = 'aes-256-gcm';

/**
 * 对一个buffer进行AES解密
 * @param {{key: string, iv: string, tag: Buffer, buffer: Buffer}} ret   待解密的内容
 * @param {String} key      密钥
 * @param {String} iv       初始向量
 * @return {Buffer}
 */
function aesDecrypt ({key, iv, tag, buffer}) {
    // 初始化解密算法
    const decipher = crypto.createDecipheriv(algorithm, key, iv);
    // 生成身份验证标签，用于验证密文的来源
    decipher.setAuthTag(tag);
    let decrypted = decipher.update(buffer);
    let end = decipher.final();
    return Buffer.concat([decrypted, end]);
}
```

### AES 具体使用

有了上述两个接口后，我们便可实现一个简单的对称加密了：


```javascript
const key = 'abcdefghijklmnopqrstuvwxyz123456'; // 32 共享密钥，长度跟算法需要匹配上
const iv = 'abcdefghijklmnop';  // 16 初始向量，长度跟算法需要匹配上
let fileBuffer = Buffer.from('abc');

// 加密
let encrypted = aesEncrypt(fileBuffer, key, iv);

// 解密
let context = aesDecrypt(encrypted);
console.log(context.toString());
```
一般情况下，这个密钥较为重要，如果发生泄露则加密失去意义，所以`key`、`iv`会使用随机数动态生成，比如：

```javascript
const key = crypto.randomBytes(32);
const iv = crypto.randomBytes(16);
```

通过上述的调整后，加解密文件是比较容易的，回到我们的业务系统上面，系统 A 生成的压缩包，最终是需要给系统 B 使用，两个系统是隔离的，那这样 `key`、`iv` 如何传输到系统 B 上面呢，况且还是动态生成的，生成出来 `key` 系统 B 是不知道的。

读到这聪明的你可能会想到，在把**压缩包给到 B 的时候，顺便把 `key`、`iv` 一同提交过去不就可以了**，但细想了下，这个肯定不能明文把这个密钥发送过去，要不这个加密意义何在。


这时便需要用上 **RSA 非对称加密技术**了。

## 使用 RSA 算法对密钥再次进行非对称加密

RSA 的加密算法 Node.js 的 [crypto 模块](http://nodejs.cn/api/crypto.html) 中已经有内置，具体的使用可以参考官方文档。

### 生成 RSA 的公钥与私钥

使用 [openssl](https://www.openssl.org/) 组件可以直接生成 `RSA` 的公钥私钥对，具体的命令可以参考：[https://www.scottbrady91.com/OpenSSL/Creating-RSA-Keys-using-OpenSSL](https://www.scottbrady91.com/OpenSSL/Creating-RSA-Keys-using-OpenSSL)。


```bash
# 生成私钥
openssl genrsa -out private.pem 1024

# 提取公钥
openssl rsa -in private.pem -pubout -out public.pem
```

这样生成出来的两个文件 `private.pem`、`public.pem` 就可以使用了，下面我们使用 Node.js 实现具体的加解密逻辑。


### RSA 加密逻辑
```javascript
const fs = require('fs');
const crypto = require('crypto');
const PK = fs.readFileSync('./public.pem', 'utf-8');

/**
 * 对一个buffer进行RSA加密
 * @param {Buffer} 待加密的内容
 * @return {Buffer}
 */
function rsaEncrypt (buffer) {
    return crypto.publicEncrypt(PK, buffer);
}
```

### RSA 解密逻辑
```javascript
const fs = require('fs');
const crypto = require('crypto');
const SK = fs.readFileSync('./private.pem', 'utf-8');

/**
 * 对一个buffer进行RSA解密
 * @param {Buffer} 待解密的内容
 * @return {Buffer}
 */
function rsaDecrypt (buffer) {
    return crypto.privateDecrypt(SK, buffer);
}
```

### RSA 具体使用

有了上述接口后，便可对 AES 的密钥进行加密后再传输，服务器 B 保存好 `RSA 私钥` ，服务器 A 则可以直接用 `RSA 公钥` 对数据加密后再发送，结合刚 `AES` 的逻辑后，如下：


```javascript
/**
 * 加密文件
 * @param {Buffer} fileBuffer
 * @return {{file: Buffer, key: Buffer}}
 */
function encrypt (fileBuffer) {
    const key = crypto.randomBytes(32);
    const iv = crypto.randomBytes(16);
    const { tag, file } = aesEncrypt(fileBuffer, key, iv);
    return {
        file,
        key: rsaEncrypt(Buffer.concat([key, iv, tag]));     // 由于长度是固定的，直接连在一起即可
    };
}

/**
 * 解密文件
 * @param {{file: Buffer, key: Buffer}}
 * @return {Buffer}
 */
function decrypt ({file, key}) {
    const source = rsaDecrypt(key).toString();
    const k = source.slice(0, 32);
    const iv = source.slice(32, 48);
    const tag = source.slice(48);
    return aesDecrypt({
        key: k,
        iv,
        tag,
        buffer: file
    })
}
```

这样结合在一起后，服务器 A 生成的压缩包，只要包含好 `{file, key}` 这两块内容，服务器 B 便可把文件解密出来了，这样基本上实现了我们第一点的目标：**1. 文件如何做加密，这样用户便无法去逆向，压缩包内部的敏感信息不会泄露出去**


但还遗留了另外一个问题需要解决：**2. 服务端在接收到信息流时，在未传输完时如何去判断压缩包的合法性，提前告知用户**

## 优化加密文件

按上面的加密方式，输出的结果是一个 `buffer文件` 内容，以及一个 `加密过的key`，除了这些信息外，一般这个 `buffer文件` 压缩包还会有一些额外的信息，比如：版本号、压缩包生成时间，描述信息等。这些信息按常规的方式，可能是分成几个文件，然后再打一个压缩包把文件放在一起，比如：

```text
// zip file
- pkg
    manifest.json       // 额外的信息
    key.json            // 保存了加密过的密钥
    file.json           // 加密过的文件
```


但如果用这种方式保存，一般情况下还要对这个 `zip文件` 做下加密，然后改下后缀名，但是服务器 B 在读取这个文件后仍然是需要全部接收，再解压到临时目录，读取内容后才可以做校验，这样问题仍然解决不了。


除此之外，还有另外一个常见的需求，产品一般希望在浏览器侧在文件上传时就先做初步的解析，把明显不合法的文件提示到用户，这样用户体验更好。

这个问题的解决方案也不难，这些所有额外的信息都是可以把它当成二进制插入到文件的头部上的，比如：
```text
包字段描述：|----插入的额外信息----|----后面才是真正的文件内容----|  
二进制文件：010101010101010101010xxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### 文件头字段设计
我们把这些所有信息，按一定的格式，使用二进制的方式全部串连在一起，最终交付的只有一个组合过的文件，比如：

```text
// theme pkg.

0                8                16                 
|------flag------|--extra length--|
|----------extra data...----------|
|-------------data...-------------|
```

- `flag`：
固定标识 `THEME`，长度：`8 byte`，说明该压缩包为一个皮肤包，这样可以快速对压缩包进行识别

- `extra length`：
`extra data` 的真实长度，这是一个 16 进制的数据，长度：`8 byte`，说明插入的数据长度。比如：长度 `35` 的数据，转化为 16 进制后为 `0x23`，那这字段为 `00000023`

- `extra data`：
使用 `RSA` 加密过的数据，我们可以把上述需要用 `RSA` 加密的信息全部放在这里，比如 `key` 字段、版本号、描述信息等

- `data`：
使用 `AES` 加密过的数据，可以通过 `extra data` 里面保存的 `key` 把真实的数据全部解密出来

### 生成的新的加密文件
有了上面的理论基础后，马上可以实践起来，代码如下：
```javascript
/**
 * 加密文件
 * @param {Buffer} fileBuffer
 * @return {Buffer}
 */
function encrypt (fileBuffer) {
    const key = crypto.randomBytes(32);
    const iv = crypto.randomBytes(16);
    const version = 'v1.1';

    // 记录上所有额外的压缩外信息，比如版本号、原始的密钥
    const extraJSON = {
        version,
        key,
        iv
    }
    // 完成文件的AES加密，并输出身份验证标签
    const { tag, file } = aesEncrypt(fileBuffer, key, iv);
    extraJSON.tag = tag;

    // 对 extraJSON 整个进行RSA加密
    const extraData = rsaEncrypt(Buffer.from(JSON.stringify(extraJSON)));
    const extraLength = extraData.length;

    // 最终把所有数据合并在一起
    return Buffer.concat([
        Buffer.from('THEME'),
        Buffer.from(Buffer.from(extraLength.toString(16).padStart(8, '0'))),
        extraData,
        file
    ]);
}
```

通过这种加密方式后，相关的信息都放在文件的头部上，我们可以不用对整个文件进行操作的时候，便可以轻松读取出来，对于解密其实就是一个反向的操作。

### 对新生成的文件进行解密

```javascript
/**
 * 解密文件
 * @param {Buffer} fileBuffer
 * @return {Buffer}
 */
function decrypt (fileBuffer) {
    const type = fileBuffer.slice(0, 8);    // THEME
    const extraLength = +('0x' + fileBuffer.slice(8, 16).toString());
    const extraDataEndIndex = 16 + extraLength;

    // 对已经被RSA加密过的数据进行解密操作
    const extraData = rsaDecrypt(fileBuffer.slice(16, extraDataEndIndex));
    const extraJSON = JSON.parse(extraData);
    // 最终使用AES再对剩下文件进行解密操作，即为最终的文件
    return aesDecrypt({
        key: extraJSON.key,
        iv: extraJSON.iv,
        tag: extraJSON.tag,
        buffer: Buffer.slice(extraDataEndIndex)
    });
}
```

使用这种方式处理后，在 `RSA` 解密出 `extraData` 的时候，就可以对整个文件进行各种校验，整个过程只要有异常说明文件已经被篡改，用这种方式比用压缩包会好很多，特别是文件体积庞大的时候，可以流式处理，发现不合理时即可马上阻止。

### 浏览器端如何解析该文件
由于现在整个文件格式都是二进制流，现代的浏览器都是有相应的能力去读取并做处理的，这样也可以在用户上传文件时先做一定的初步处理，体验会有比较大的提升

可以使用 `DataView` 的方式把二进制数据读取出来，详情可以参考：[DataView](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/DataView)，初步的实现如下：

```javascript
/**
 * 把二进制流转成对应ascii字符
 * @param {DataView} dv         二进制数据库
 * @param {Number}   start      起始位置
 * @param {Number}   end        结束位置
 * @return {String}
 */
function buffer2Char (dv, start, end) {
    let ret = [];
    for (let i = start; i < end; i++) {
        let charCode = dv.getUint8(i);
        let code = String.fromCharCode(charCode);
        ret.push(code);
    }
    return ret.join('');
}

function test () {
    let fileDom = document.getElementById('file');
    let file = fileDom.files[0];
    let reader = new FileReader();
    reader.readAsArrayBuffer(file);
    reader.addEventListener("load", function(e) {
        let dv = new DataView(buffer);
        let flag = buffer2Char(dv, 0, 8);   // THEME
        var extraLength = +('0x' + buffer2Char(dv, 8, 16));
        var extraData = buffer2Char(dv, 16, extraLength);

        console.log(flag, extraLength, extraData);
    });
}
```
当然用这种方式有一个前提是需要把一部分非敏感的信息放出来，不要加密，这样便可以实现在浏览器端也对文件进行读取。只需要前后端的格式约定做好，都可以采用这种方式对压缩包进行一定的初步校验，当然后端的校验仍然是需要做好的。

至此，我们完成了对文件的加密、解密以及浏览器解析等操作，希望对你们有帮助

## 结语


文件的加密、解密在后端其实是一个很常规的操作，除了上面聊到的 `AES`、`RSA`，其实还有其它很多加密方案，具体可以看看 [Node.js crypto 模块](http://nodejs.cn/api/crypto.html)，已经有内置比较多的方案可以直接使用。

当然文件的加解密，也可以直接用 `zip`、`7z` 等这些压缩工具，再配合密码的方案，一般情况也是够用的，但是免不了有定制化的需求，一般也都是结合使用，比如上面的 `fileBuffer` 实际内部就是先用这些工具对文件进行了压缩并加密。还是以场景为重，多种方案结合效果更好。

文件加解密的就讲到这里吧，还有什么其它问题的可以在评论区讨论，谢谢。

## 声明
本网站上使用图片链接出自：[https://images.ctfassets.net/9ijwdiuuvngh/5K4AedH960A824yg6CQyCi/8b3ee8ad5e63e6d50f78b9d7aa749669/Header_Encryption.png](https://images.ctfassets.net/9ijwdiuuvngh/5K4AedH960A824yg6CQyCi/8b3ee8ad5e63e6d50f78b9d7aa749669/Header_Encryption.png)，使用搜索引擎查找，未能联系到图片相应作者，本着纯粹技术分享的原则与心态，如果有侵犯到你的权利，请联系我们，立马删除，谢谢。


