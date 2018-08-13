#### 初步观察
目标网站：[https://free-ss.site/](https://free-ss.site/ "https://free-ss.site/")，一个分享免费ss账号的网站，ss账号是什么，大家都清楚，初看网站，很普通的一个表格展示，行列分的很清楚，想用爬虫爬取，必然非常简单。
![](https://i.imgur.com/ER7R1AI.png)
#### 源码研究
右键，查看网页源代码发现，貌似没找着表格的数据啊。只看见一堆好似乱码的东西，这时候我们知道，该网站是通过ajax异步加载数据，还进行了加密处理。
![](https://i.imgur.com/ziWsTqy.png)

先看看接口请求数据：右键，检查，查看接口请求数据
发现一次页面刷新过程，请求了两个接口：

1.[https://free-ss.site/ss.json?_=1534145860216](https://free-ss.site/ss.json?_=1534145860216 "https://free-ss.site/ss.json?_=1534145860216")

- 请求方式：GET
- 返回结果：![](https://i.imgur.com/b1eTtZX.png)

2.[https://free-ss.site/data.php](https://free-ss.site/data.php)

- 请求方式：POST
- 请求body：a=15f34145c8ebd57a&b=c75fd209a3861e4b&c=f8bf66da42417576
- 返回结果：一堆乱码，就不上了，截个图
![](https://i.imgur.com/1gdmKky.png)

看上去第一个就是我们要的数据了（一串json），仔细查看一个结果试试。

	["10/10/10/10", "170.51.203.24", "19174", "rc4-md5", "65947054", "15:35:01", "SG"]

从中我们发现了，IP地址（170.51.203.24），ss端口（19174），ss加密方式（rc4-md5），但是没有密码啊？

仔细找了一圈也没发现，65947054像是密码，但是亲测结果并不是。而且这时候我发现了更严重的问题，这串json数组的数据，在网页上根本找不到，匹配不上！

无奈试一下第二个，看起来像是乱码的string。。。

#### 【正题】 js解密
中间研究的部分舍去不说了，直接说结果吧

源码中含有两段eval包裹的function，这是非常明显的js加密代码段，可以使用在线js解密工具https://tool.lu/js/进行解密

![](https://i.imgur.com/55SzDvr.png)

解密后：

	var detect = false;
	if (navigator.webdriver || window.webdriver || self != top) {
		detect = true
	}
	if (window.outerWidth === 0 || window.outerHeight === 0) {
		try {
			var canvas = document.createElement('canvas');
			var ctx = canvas.getContext('webgl') || canvas.getContext('experimental-webgl');
			var exts = ctx.getSupportedExtensions()
		} catch (e) {
			detect = true
		}
	}
	function decodeImage(data, callback) {
		qrcode.callback = callback;
		qrcode.decode(data)
	}

![](https://i.imgur.com/NQp0wbi.png)

	var mode = 'CBC' + 'CFB' + 'CTR' + 'ECB' + 'OFB';
	var pad = 'Pkcs7' + 'ZeroPadding';
	var dec = CryptoJS.AES.decrypt(d, x, {
		iv: y,
		mode: CryptoJS.mode.ECB,
		padding: CryptoJS.pad.Pkcs7
	});

原来网页的一串乱码就变成了下面的：

	$(document).ready(function() {
		var detect = false;
		if (navigator.webdriver || window.webdriver || self != top) {
			detect = true
		}
		if (window.outerWidth === 0 || window.outerHeight === 0) {
			try {
				var canvas = document.createElement('canvas');
				var ctx = canvas.getContext('webgl') || canvas.getContext('experimental-webgl');
				var exts = ctx.getSupportedExtensions()
			} catch (e) {
				detect = true
			}
		}
		function decodeImage(data, callback) {
			qrcode.callback = callback;
			qrcode.decode(data)
		}
		if (detect) {
			var b = '142bc0e537d8f6a9';
			var c = 'c75fd209a3861e4b';
			var a = 'f8bf66da42417576';
		} else {
			var c = 'f8bf66da42417576';
			var a = '142bc0e537d8f6a9';
			var b = 'c75fd209a3861e4b';
		}
		decodeImage('data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAFcAAABXAQMAAABLBksvAAAABlBMVEX///8AAABVwtN+AAAACXBIWXMAAA7EAAAOxAGVKw4bAAAAr0lEQVQ4jc3RwQ3DIAwFUEccuJEFkLoGN1bKBCksQFbKjTWQWMKHKi60UdoDmGt8ekiIbxuAu9VM5HdBlBgrEH7PDljbHIzYcGCPYzszcsmNfz20XPoPJl+zNF2PMf1W0fKMdFhyhjXpRerlzO0YM+HDI2cFr1Xq9ZPbM4DYovjO2DcoOXDdg80Ha1XfhGlnbfNG8MSBnUxTBN4B9GoS55qrz/sdl/7LHq6/bvpe9QZUXf/HJHMWAAAAAABJRU5ErkJggg==',
		function(a) {
			$.post("data.php", {
				a: a,
				b: b,
				c: c
			}, function(d) {
				//关键代码：使用的是CryptoJS加密
				var mode = 'CBC' + 'CFB' + 'CTR' + 'ECB' + 'OFB';
				var pad = 'Pkcs7' + 'ZeroPadding';
				var dec = CryptoJS.AES.decrypt(d, x, {
					iv: y,
					mode: CryptoJS.mode.ECB,
					padding: CryptoJS.pad.Pkcs7
				});
			}}

java 代码解密：

	public class CryptoUtils {
	    /**
	     * 解密AES字符串
	     * @param key 加密key，必须为16位
	     * @param iv 偏移量，必须为16位
	     * @param format java-javascript 只支持部填充摸索
	     * @param data 加密字符串
	     * @return 解密字符串
	     */
	    public static String encryp(String key,String iv,String format,String data) {
	        try
	        {
	            byte[] encrypted1 = new BASE64Decoder().decodeBuffer(data);
	
	            Cipher cipher = Cipher.getInstance(format);
	            SecretKeySpec keyspec = new SecretKeySpec(key.getBytes(), "AES");
	            IvParameterSpec ivspec = new IvParameterSpec(iv.getBytes());
	
	            cipher.init(Cipher.DECRYPT_MODE, keyspec);
	
	            byte[] original = cipher.doFinal(encrypted1);
	            String originalString = new String(original, StandardCharsets.UTF_8);
	            System.out.println(originalString);
	            return originalString;
	        }
	        catch (Exception e) {
	            e.printStackTrace();
	        }
	        return null;
	    }
	
	    public static void main(String[] args) {
	        //				var dec = CryptoJS.AES.decrypt(d, x, {
	        //					iv: y,
	        //					mode: CryptoJS.mode.ECB,
	        //					padding: CryptoJS.pad.Pkcs7
	        //				});
	        String key = "15f34145c8ebd57a"; //对应网页的a
	        String iv = "c75fd209a3861e4b"; //对应网页的b
	        String format = "AES/ECB/PKCS5Padding"; //对应CryptoJS.mode.ECB
	        String data = "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";
	        System.out.println(encryp(key,iv,format,data));
	    }
	}

使用的加解密jar包：

	compile group: 'org.webjars', name: 'cryptojs', version: '3.1.2'

加密算法详解：https://my.oschina.net/u/1161660/blog/834429

不多赘述了，网上都有资料，使用的加密算法是cryptojs，有多种模式以及补码方式

运行结果：

![](https://i.imgur.com/wUOvnZ5.png)

![](https://i.imgur.com/Zc5PTHx.png)

非常工整的json数组，正式我们想要的，ip，端口，密码，ss加密方式都有，跟网页上的table一模一样！

#### 请求获取数据

之前我们提到，请求的第二个接口发了三个参数，是：

	a: 15f34145c8ebd57a
	b: c75fd209a3861e4b
	c: f8bf66da42417576

对应上文中第二个解密后的js代码中的
![](https://i.imgur.com/yC4s9ax.png)

仔细研究这串代码，是发送请求获取加密数据的关键代码：

	decodeImage('data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAFcAAABXAQMAAABLBksvAAAABlBMVEX///8AAABVwtN+AAAACXBIWXMAAA7EAAAOxAGVKw4bAAAAr0lEQVQ4jc3RwQ3DIAwFUEccuJEFkLoGN1bKBCksQFbKjTWQWMKHKi60UdoDmGt8ekiIbxuAu9VM5HdBlBgrEH7PDljbHIzYcGCPYzszcsmNfz20XPoPJl+zNF2PMf1W0fKMdFhyhjXpRerlzO0YM+HDI2cFr1Xq9ZPbM4DYovjO2DcoOXDdg80Ha1XfhGlnbfNG8MSBnUxTBN4B9GoS55qrz/sdl/7LHq6/bvpe9QZUXf/HJHMWAAAAAABJRU5ErkJggg==',
	function(a) {
		$.post("data.php", {
			a: a,
			b: b,
			c: c
		}

简要描述一下代码的运行流程：

1.解析一张base64 png的图片验证码，获得function(a) 中的a 参数

2.ajax请求获取data.php的数据

data:image/png;base64,*** 可以使用[https://codebeautify.org/base64-to-image-converter](https://codebeautify.org/base64-to-image-converter)在线解析出来，果不其然，是一张二维码，扫出来就是我们要发送的a,b,c当中的参数a

![](https://i.imgur.com/P2hBdGx.png)

然后拼接参数发送post请求即可获取加密后的结果。

#### 总结

该网站采用的加密技术主要是以下几种

- js eval加密（最简单也是最有效的加密方式）
- cryptojs 加密（AES加密方式，需要对应的key,iv,mode方可解密）
- base64 图片方式传输字符串,不仔细观察不容易发现

以上源代码可联系qq1398371419获取


