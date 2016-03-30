转载：
http://my.oschina.net/DragonWang/blog/62082

Android系统源码目录 system/framework 下各个jar包的用途

 - am.jar：终端下执行am命令时所需的java库。源码目录：framework/base/cmds/am
 - android.policy.jar：锁屏界面需要用到的jar包，该包引用了android.test.runner.jar，源码目录：framework/base/policy
 - android.test.runner.jar：测试应用所需的jar包，该包引用了core.jar,core-junit.ajr以及framework.jar，源码目录：framework/base/test-runner
 - bmgr.jar：adb shell命令下对Android Device所有package备份和恢复的操作时所需的java库。官方文档：http://developer.android.com/guide/developing/tools/bmgr.html。不过这个android服务默认是Disabled，而且要backup的应用必须实现BackupAgent，在AndroidManifest.xml的application标签中加入android：backupAgent属性。源码目录：framework/base/cmds/bmgr
 - bouncycastle.jar： java三方的密匙库，网上资料说用来apk签名、https链接之类，官网 ：http://www.bouncycastle.org/java.html
 - com.android.future.usb.accessory.jar：用于管理USB的上层java库，在系统编译时hardware层会调用到。源码目录：frameworks/base/libs/usb
 - com.android.location.provider.jar：
 - com.android.nfc_extras.jar：NFC外部库。android/nfc/NfcAdapter.java会调用到包中的NfcAdapterExtras.java。源码目录：frameworks/base/nfc-extras
 - core-junit.jar ：junit核心库，在运行*Test.apk时被调用。
 - core-junitrunner.jar：未知，公司话机上有。
 - core-tests*.jar：framework下的一系列测试jar包，不做测试时可删除。
 - core.jar：核心库，启动桌面时首先加载这个。源码目录： 
 - ext.jar：android外部三方扩展包，源码主要是external/nist-sip（java下的sip三方库）、external/apache-http（apache的java三方库）、external/tagsoup（符合SAX标准的HTML解析器）。其实这个jar包可以添加外部扩展jar包，只需在framework/base/Android.mk中的ext-dir添加src目录即可。
 - framework-res.apk：android系统资源库。
 - framework.jar：android的sdk中核心代码。
 - ime.jar：ime命令所需jar包，用于查看当前话机输入法列表、设置输入法。源码目录：framework/base/cmds/ime
 - input.jar：input命令所需的jar包，用于模拟按键输入。源码目录：framework/baes/cmds/input
 - javax.obex.jar：java蓝牙API，用于对象交换协议。源码目录：framework/base/obex
 - monkey.jar：执行monkey命令所需jar包。源码目录：framework/base/cmds/monkey
 - pm.jar：执行pm命令所需的jar包，pm详情见adb shell pm，源码目录：framework/base/cmds/pm
 - services.jar：话机框架层服务端的编译后jar包，配合libandroid_servers.so在话机启动时通过SystemServer以循环闭合管理的方式将各个service添加到ServiceManager中。源码目录：framework/base/service
 - sqlite-jdbc.jar： sqlite的Java DataBase Connextivity jar包。
 - svc.jar：svc命令所需jar包，可硬用来管理wifi,power和data。源码目录：framework/base/cmds/svc，详情见：http://madgoat.cn/2011/02/android_svc/