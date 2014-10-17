# Android Ant 批量多渠道打包实例

------

关于批量打包，无需多言，这是每个国内Android开发者必须面对的一个问题。

下面，我就以开源项目「[知乎小报][1]」为例，详细说明如何使用ANT实现批量打渠道包。

## 1 Ant 安装

* 下载ANT

请前往 [http://ant.apache.org][2] 下载。

* 配置环境变量

设置环境变量后，在命令行下测试ant命令，如果出现以下内容，则说明配置成功：

``` shell
cundongdeMacBook-Pro:~ cundong$ ant
Buildfile: build.xml does not exist!
Build failed
```
* ant-contrib-1.0b3.jar下载

由于ant本身不支持迭代，因此我们需要用到一个第三方的库 ant-contrib来实现迭代功能。

下载ant-contrib，并将ant-contrib-1.0b3.jar文件拷贝至ANT安装目录。

下载地址：[http://ant-contrib.sourceforge.net/][3]

## 2 生成local.properties、build.xml文件

先介绍一下iZhihuPaper的工程依赖情况。

* iZhihuPaper 依赖 actionbarpulltorefresh.extras.actionbarsherlock、Crouton、PhotoView

* actionbarpulltorefresh.extras.actionbarsherlock 依赖 ActionBarPullToRefresh

* ActionBarPullToRefresh 依赖 actionbarsherlock、SmoothProgressBar

* Crouton、actionbarsherlock 依赖 SupportLib

* PhotoView、SmoothProgressBar、SupportLib 无任何依赖

###生成方式

我们需要为iZhihuPaper工程和他直接或者间接引用的所有工程（一共7个）都生成local.properties、build.xml文件。

命令格式：

android update project --target {target版本} --name {工程名字} --path {工程目录}

依次执行以下命令：

1.
``` shell
android update project --target android-20 --name SupportLib --path /Users/cundong/Documents/github/SupportLib
``` 

2.
``` shell
android update project --target android-20 --name PhotoView --path /Users/cundong/Documents/github/PhotoView
```

3.
``` shell
android update project --target android-20 --name SmoothProgressBar --path /Users/cundong/Documents/github/SmoothProgressBar
```

4.
``` shell
android update project --target android-20 --name Crouton --path /Users/cundong/Documents/github/Crouton
``` 

5.
``` shell
android update project --target android-20 --name actionbarsherlock --path /Users/cundong/Documents/github/actionbarsherlock
```

6.
``` shell
android update project --target android-20 --name ActionBarPullToRefresh --path /Users/cundong/Documents/github/ActionBarPullToRefresh
```

7.
``` shell
android update project --target android-20 --name actionbarpulltorefresh.extras.actionbarsherlock --path /Users/cundong/Documents/github/actionbarpulltorefresh.extras.actionbarsherlock
``` 

8.
``` shell
android update project --target android-20 --name iZhihuPaper --path /Users/cundong/Documents/github/iZhihuPaper
``` 

###注意事项

* BUILD SCUUCESS
如果执行命令后，出现如下所示：
``` shell
cundongdeMacBook-Pro:~ cundong$ android update project --target android-20 --name SupportLib --path /Users/cundong/Documents/github/SupportLib
Updated project.properties
Updated project.properties
Added file /Users/cundong/Documents/github/SupportLib/build.xml
Updated file /Users/cundong/Documents/github/SupportLib/proguard-project.txt
It seems that there are sub-projects. If you want to update them
please use the --subprojects parameter.
```

则说明执行成功。

* 常见BUILD FAILED问题

如果执行后，出现如下提示：
``` shell
BUILD FAILED
/Users/cundong/Documents/github/iZhihuPaper/build.xml:44: The following error occurred while executing this line:
/Users/cundong/Documents/github/iZhihuPaper/build.xml:59: The following error occurred while executing this line:
/Applications/adt-bundle-mac-x86_64/sdk/tools/ant/build.xml:470: Invalid file: /Users/cundong/Documents/github/SmoothProgressBar/build.xml
```

则说明它所依赖的工程缺少project.properties、project.properties文件，请先参照步骤1，为其依赖的工程生成project.properties、project.properties文件。

如果遇到以下问题：

``` shell
BUILD FAILED
/Applications/adt-bundle-mac-x86_64-20140624/sdk/tools/ant/build.xml:601: The following error occurred while executing this line:
/Applications/adt-bundle-mac-x86_64-20140624/sdk/tools/ant/build.xml:653: The following error occurred while executing this line:
/Applications/adt-bundle-mac-x86_64-20140624/sdk/tools/ant/build.xml:698: null returned: 1
```

则需要手动删除该工程的gen、bin目录。

## 配置 local.properties

配置local.properties文件，增加ant.dir、target.dir:

``` shell
sdk.dir=/Applications/adt-bundle-mac-x86_64/sdk
ant.dir=/Applications/apache-ant-1.9.4
target.dir=/Users/cundong/Documents/ZhihuPaperRelease
```
ant.dir为ant安装目录，target.dir为批量打包的apk存储目录。

详细例子可参考：[ZhihuPaper/local.properties][4]

## 3 添加 ant.properties文件

1.将签名文件（*.keystore）拷贝到工程的目录。

2.在根目录下新建ant.properties文件。

``` shell
key.store=android.keystore
key.alias=android
key.store.password=Cundong123456!@#
key.alias.password=Cundong123456!@#
market_channels=Wandoujia,360
app_name=ZhihuPaper
app_version=2.1
```

说明：
key.store为签名文件；
key.alias为签名文件别名；
key.store.password、key.alias.password为密码；
market_channels为我们需要生成的所有渠道列表，使用“，”分开；app_name为生成apk的文件名；
app_version为生成apk的版本号；

详细例子可参考：[ZhihuPaper/ant.properties][5]

##配置build.xml

为了实现批量打出多个渠道包，我们必须手动对刚刚生成的build.xml文件进行修改。

* 引入ant.properties文件。

 ```xml
 <property file="ant.properties" />
 ```

* 支持循环执行

```xml
    <!-- 支持循环执行 -->  
    <taskdef resource="net/sf/antcontrib/antcontrib.properties" >  
        <classpath>  
           <pathelement location="${ant.dir}/lib/ant-contrib-1.0b3.jar" />  
        </classpath>  
    </taskdef>  
    
    <echo>Run ant-contrib-1.0b3.jar ok</echo>  
```
* 配置循环打包代码

```xml
    <target name="deploy">   
        <foreach target="edit_and_build" list="${market_channels}" param="channel" delimiter=",">   
        </foreach>   
    </target>  
    
    <target name="edit_and_build">   
        <echo>Run '${channel}' apk</echo>  
        
		<replaceregexp
		    encoding="utf-8"
		    file="AndroidManifest.xml"
		    flags="s"
		    match='android:name="UMENG_CHANNEL".+android:value="([^"]+)"'
		    replace='android:name="UMENG_CHANNEL" android:value="${channel}"'/>
			
      	<property name="out.final.file"  location="${target.dir}/${app_version}/${app_name} V${app_version}(${channel}).apk" /> 
	    <antcall target="clean" />  
	    <antcall target="release" />  
     
    </target>   
```
配置后，会读取ant.properties中market_channels中配置项，得到一个渠道号数组，对这个数据进行迭代，替换AndroidMainfext.xml文件中的android:name="UMENG_CHANNEL"。
	
每替换好一个，将输出到"out.final.file"。

${target.dir}，即为local.properties文件中配置的target.dir=/Users/cundong/Documents/ZhihuPaperRelease
${app_name}，即为ant.properties文件中配置的app_name=ZhihuPaper
${app_version}，即为ant.properties文件中配置的app_version=2.1
${channel}，即为当前循环的渠道号

请务必保证${target.dir}/${app_version}目录真是存在并且有写权限。

当前例子中为：/Users/cundong/Documents/ZhihuPaperRelease/2.1，如果这么目录不存在，则会提示报错信息。

详细例子可参考：[ZhihuPaper/build.xml][6]

## 4 配置proguard-project.txt文件

proguard-project.txt，即混淆时的配置文件。

* 引用的第三方jar包，不要混淆；
* 自己写的控件，即需要配置在layout文件中的Widget，不要混淆；
* Android的基础组件，不要混淆。

* 需要在project.properties中配置：proguard.config=${sdk.dir}/tools/proguard/proguard-android.txt:proguard-project.txt

详细例子可参考： [ZhihuPaper/proguard-project.txt][7]

## 5 打包
在iZhihuPaper中创建一个批处理文件，Mac为*.sh文件，Window为*.bat文件：

```shell

cd /Users/cundong/Documents/github/iZhihuPaper
ant deploy
pause

```

调用这个批处理文件，即可进行批量打混淆后的渠道包。

  [1]: https://github.com/cundong/ZhihuPaper
  [2]: http://ant.apache.org
  [3]: http://ant-contrib.sourceforge.net/
  [4]: https://github.com/cundong/ZhihuPaper/blob/master/local.properties
  [5]: https://github.com/cundong/ZhihuPaper/blob/master/ant.properties
  [6]: https://github.com/cundong/ZhihuPaper/blob/master/build.xml
  [7]: https://github.com/cundong/ZhihuPaper/blob/master/proguard-project.txt
