## YNU统一身份认证登录

### 1 登录信息

- 用户名, 即学号
- 密码
- 验证码

### 2 登录信息处理

1. 密码经过 AES 加密

2. 验证码获取

   ```
   https://ids.ynu.edu.cn/authserver/captcha.html
   ```

**详细介绍下用户密码加密过程**

加密js

```
https://ids.ynu.edu.cn/authserver/custom/js/encrypt.js
```

主要涉及到下面几个函数:

```js
function _gas(data, key0, iv0) { 
    key0 = key0.replace(/(^\s+)|(\s+$)/g, ""); 
    var key = CryptoJS.enc.Utf8.parse(key0); 
    var iv = CryptoJS.enc.Utf8.parse(iv0); 
    var encrypted = CryptoJS.AES.encrypt(data, key, { 
        				iv: iv, mode: CryptoJS.mode.CBC, padding: CryptoJS.pad.Pkcs7 
    				}); 
    return encrypted.toString(); 
} 

function encryptAES(data, _p1) { 
    if (!_p1) { 
        return data; 
    } 
    var encrypted = _gas(_rds(64) + data, _p1, _rds(16)); 
    return encrypted; 
}

var $_chars = 'ABCDEFGHJKMNPQRSTWXYZabcdefhijkmnprstwxyz2345678'; 
var _chars_len = $_chars.length; 

function _rds(len) { 
    var retStr = ''; 
    for (i = 0; i < len; i++) {
         retStr += $_chars.charAt(Math.floor(Math.random() * _chars_len)); 
    } 
    return retStr;
}
```

```js
// 返回加密信息
function _etd2(_p0, _p1) {
    try { 
         var _p2 = encryptAES(_p0, _p1); 
         $("#casLoginForm").find("#passwordEncrypt").val(_p2); 
    } catch (e) {
         $("#casLoginForm").find("#passwordEncrypt").val(_p0); 
    } 
}

// 调用过程
// https://ids.ynu.edu.cn/authserver/custom/js/login-wisedu_v1.0.js?v=1.0
_etd2(password.val(), casLoginForm.find("#pwdDefaultEncryptSalt").val());

// 源代码可见
// value 会变化, 目前尚未发现如何更新
<input type="hidden" id="pwdDefaultEncryptSalt" value="CjleTDtyuJHUiBCT"/>
```

这样逆向分析就可以发现:

1. 先获取登录密码和默认salt

2. 调用 `_etd2`函数. `_p0`为密码, `_p1` 为 salt

3. 调用 `encryptAES`函数加密

4. `encryptAES`调用 `_gas`函数

   - `_rds(64) + data`: 随机获取 64 位字符, 将用户密码拼接在最后

   - `_p1`:  salt
   - `_rds(16)`: 随机获取16位字符

   生成`key`(密码), `iv`(偏移量). 加密模式为 `CBC`, 填充方式为 `Pkcs7`, 数据块 128 位.

   返回加密信息

5. 将加密信息设为 `id="passwordEncrypt"` value

以上就是加密的完整过程.

**模拟加密过程**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
    <!-- 引入需要的js库 -->
    <script src="https://ids.ynu.edu.cn/authserver/custom/js/encrypt.js"></script>
</head>

<script>
    'use strict';
	// 覆盖引用js库的 _gas 方法
    function _gas(data, key0, iv0) { 
        key0 = key0.replace(/(^\s+)|(\s+$)/g, ""); 
        var key = CryptoJS.enc.Utf8.parse(key0); 
        alert(key);
        var iv = CryptoJS.enc.Utf8.parse(iv0); 
        alert(iv);
        var encrypted = CryptoJS.AES.encrypt(data, key, { iv: iv, mode: CryptoJS.mode.CBC, padding: CryptoJS.pad.Pkcs7 }); 
        return encrypted.toString(); 
	}
	
    function _etd2(_p0, _p1) {
        try { 
           var _p2 = encryptAES(_p0, _p1); 
           return _p2;
       } catch (e) { } 
    }
   
   console.log(_etd2('123456789', 'Gremejeoge5kaLwO'));
    </script>

<body>
    
</body>
</html>
```

输出结果:

```
key: 4772656d656a656f6765356b614c774f  ===> 16进制字符转ASCII  Gremejeoge5kaLwO  (salt值)
iv: 544d4d54517834355463647762743854   ===>  TMMTQx45Tcdwbt8T
加密结果:
Ue2odFBoIwaD/mB7UoAtoS2FtjpZEpVRYaFe20+dycK7+m1LmHqF1mGlJU650GneWKJxdMctb/KdWHI1lXeu8kHtCOyfGetzSPtwSw7ouIU=
```

解密函数

```js
function decryptAES (data, key0, iv0) {
    var key = CryptoJS.enc.Utf8.parse(key0);
    var iv = CryptoJS.enc.Utf8.parse(iv0);
    var decrypt = CryptoJS.AES.decrypt(data, key, {
        iv: iv,
        mode: CryptoJS.mode.CBC,
        padding: CryptoJS.pad.Pkcs7
    });
    var decryptedStr = decrypt.toString(CryptoJS.enc.Utf8);
    return decryptedStr.toString();
}
// 在线解密: http://tool.chacuo.net/cryptaes
// 16进制转ASCII: http://tool.haooyou.com/code?group=convert&type=hexToStr&charset=UTF-8

console.log(decryptAES('Ue2odFBoIwaD/mB7UoAtoS2FtjpZEpVRYaFe20+dycK7+m1LmHqF1mGlJU650GneWKJxdMctb/KdWHI1lXeu8kHtCOyfGetzSPtwSw7ouIU=', 'Gremejeoge5kaLwO', 'TMMTQx45Tcdwbt8T'))

// 解密结果
8RiXTY4scbhCDnW3apFwjxQN7hGY7GtKrBc7Tdi26NKRkfQnZYTpQYc8SsWH6ZnW123456789
// 可以看到从第65位开始为用户密码
```

到这里就完成了用户输入密码的加解密过程.



### 3 开始登录

接口地址

```
https://ids.ynu.edu.cn/authserver/login
POST方法
```

还需要其他参数, 这里遇到问题:

当提交数据的时候, 返回 302 , 如果允许转发则使用 GET 方法请求, 并返回200, 不会返回cookie.

通过抓包发现, 每次都返回 302, 但经过3次转发登录到目的网址. 不知道和模拟请求有什么区别

......

后面看能不能解决



### 声明

本项目仅用于交流学习使用, 请勿用作其他用途

如有侵犯你的权益, 请联系我删除

