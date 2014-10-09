# 引用 android-support-v4.jar 时候冲突问题解决

------

在开发应用的时候，难以避免的会用到很多第三方的开源项目，这些项目中都会使用android-support-v4.jar包，而我的项目也使用它。

再加上这些开源项目之间还存在各种复杂的引用关系。

就这样引用来、引用去，就可能会出现android-support-v4.jar的冲突问题，类似于：

> Jar mismatch! Fix your dependencies

> Found 2 versions of android-support-v4.jar in the dependency list,but not all the versions are identical 

我的解决方法是：

1. 
新建一个空的工程，里面仅仅在libs/下放置一个版本正确的android-support-v4.jar包。

2. 
删除你的项目，以及你引用项目中的所有android-support-v4.jar。

3. 
为每一个需要用到android-support-v4.jar的工程，增加对步骤1中创建的空工程的依赖。
Properties > Android> Library box > Add

这样以来，可以保证整个工程中用到的v4.jar是单例的。




