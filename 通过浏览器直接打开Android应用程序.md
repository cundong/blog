# 通过浏览器直接打开Android应用程序
------

之前写过一篇blog，介绍如何通过点击浏览器中的链接，直接打开本地Android App。

实现的不太完美，最近看了微博、京东的手机版网页，感觉他们的实现方式很不错，就研究了一下，最终，实现以下效果：

如果本地已经安装了指定Android应用，就直接打开它；如果没有安装，则直接下载该应用的安装文件（也可以跳转到下载页面）。

## 实现原理

1.为Android应用的启动Activity设置一个Schema；

2.用户点击浏览器中的链接时，在网页中创建一个不可见的iframe，并且让这个iframe尝试去加载步骤1中的Schema；

3，如果在指定的时间内跳转未成功，则当前页跳转到apk的下载地址（或者下载页）；

## JavaScript代码
```javascript
		<script>
			(function(){
				var ua = navigator.userAgent.toLowerCase();
				var t;
			    var config = {
			        /*scheme:必须*/
			        scheme_IOS: 'cundong://',
			        scheme_Adr: 'cundong://splash',
			        download_url: document.getElementById('J-download-app').value,
			        timeout: 600
			    };

				function openclient() {
				    var startTime = Date.now();

				    var ifr = document.createElement('iframe');
                    
                
                    ifr.src = ua.indexOf('os') > 0 ? config.scheme_IOS : config.scheme_Adr;
                    ifr.style.display = 'none';
                    document.body.appendChild(ifr);
                    
                    var t = setTimeout(function() {
                        var endTime = Date.now();
                        
                        if (!startTime || endTime - startTime < config.timeout + 200) { //如果装了app并跳到客户端后，endTime - startTime 一定> timeout + 200
                            window.location = config.download_url;
                        } else {
                        	
                        }
                    }, config.timeout);

				    window.onblur = function() {
				        clearTimeout(t);
				    }
				}
				window.addEventListener("DOMContentLoaded", function(){
					document.getElementById("J-call-app").addEventListener('click',openclient,false);

				}, false);
			})()
		</script>
```
		
## Android代码

``` xml
        <activity
            android:name=".activity.LauncherActivity"
            android:configChanges="orientation|keyboardHidden|navigation|screenSize"
            android:label="@string/app_name"
            android:screenOrientation="portrait" >
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
            <intent-filter>
                <action android:name="android.intent.action.VIEW" />

                <category android:name="android.intent.category.DEFAULT" />
                <category android:name="android.intent.category.BROWSABLE" />

                <data
                    android:host="splash"
                    android:scheme="cundong" />
            </intent-filter>
        </activity>
```