---
layout: post
title: react native related problems and solutions
tags: [react_native]
category: 编程
---

将用博客来记录自己在react native开发过程中遇到的问题，这篇主要侧重于配置过程中用到的问题。这种问题可能只需要一个合适的答案，找寻这个答案可能需要花费半天的工作时间，甚至更久。


1. error: could not install smartsocket listener: Address already in use
	
	解决方法：genymotion的adb设置Android sdk
   
2. cant find local.properties
	
	解决方法：手动添加文件，然后sdk.dir=xx
   
3. Undefined symbols for architecture x86_64: “std::terminate()”
	解决方法：I ran in to this issue as well, and the solution @charpeni proposed solved the issue. To be clear for others, if you are upgrading to 0.26+ then you need to make the following changes.
	In ios/YourProject.xcodeproj/project.pbxproj, look for the two lines like OTHER_LDFLAGS = "-ObjC";. Replace them with the following:

	``` bash
 	OTHER_LDFLAGS = (
        "-ObjC",
        "-lc++",
	);
	```

4. react-native run-android时出现Could not download imagepipeline.aar

	解决方法：修改build.gradle的版本，com.android.tools.build:gradle:2.1.0，改为更高的，然后更改gradle/wrapper/gradle-wrapper.properties中相应的gradle-2.10-all.zip。
   
5. Application ReactExample has not been registered. This is either due to a require() error during initialization or failure to call AppRegistry.registerComponent.

	解决方法： 关掉其他react-native server和node进程
   

6. realm对于schema的更新需要重新启动应用，如果应用已经在本地数据库中写入了数据，则需要将应用删除，重装，因为schema和已有数据会存在冲突。

7. 自己react-native init test.然后测试realm。在ios端总是提示‘Cannot read property 'debugHosts' of undefined’ 和‘Module AppRegistry is not a registered callable module’
	
	解决方法：是rnpm的问题，对于ios项目来说，project name中不能有excample和test,需要在packaje.json中加入。但是自己测试并没有成功。测试将project name命名为其他可以。Realm团队称在下一个版本中会修复这个问题

	``` javascript
	"rnpm": {
		"ios": {
		"project": "ios/<project-name>.xcodeproj"
		}
	}
	```
	对于andorid新建的包含realm的rn项目中还会有'Missing Realm constructor-please ensure RealmReact framework is included'的问题
	
	解决方法是在MainApplication.java中添加如下两行
 
	``` java
	import io.realm.react.RealmReactPackage; // ADD THIS
	  @Override
	    protected List<ReactPackage> getPackages() {
	      return Arrays.<ReactPackage>asList(
	          new MainReactPackage(),
	          new RealmReactPackage() // AND THIS
	      );
	    }
	```
	
  如在android中出现“Project with path ':realm' could not be found in project ':app'. realm”的问题，则需要在setting.gradle中添加
  
	``` java
 	include ':realm'
 	project(':realm').projectDir = new File(settingsDir, '../node_modules/realm/android')
	```
 

8. react native realm的可视化
官方文档中只介绍了Objective-C，Java，Swift中有realm的可视化工具Realm Browser（暂时只支持mac），在react native中也可以使用。

	``` bash
	console.log(db.path);//输出realm在模拟器上存储的目录
	adb pull /data/data/<packagename>/files/ . //从模拟器上或者真机（需要root）拉取realm文件
	```
	用Realm Browser打开即可，默认名字是default.realm

9. 在./gradlew assembleRelease的时候报错：
Execution failed for task ':app:bundleReleaseJsAndAssets'. A problem occurred starting process 'command 'node''

执行

	``` bash
	./gradlew --stop
	```

	停掉后台deamon进程，重新打包即可。

10. react native unable to resolve module react/lib/ReactUpdates

	<img src="/public/img/rn/unable_to_resolve.jpeg" width = "300" height = "200" alt="ReactUpdates" align=center />

	在[facebook issue 4968](https://github.com/facebook/react-native/issues/4968)中对这个问题有一些讨论，但是我试了较为普遍的回答：
	
	``` bash
	watchman watch-del-all
	npm cache clean && npm install
	```

	对于我的情况并不生效。后来发现是由于react-native和react版本不匹配造成的。执行
	
	``` bash
	npm install react@~15.3.1 --save
	```
	问题fix，15.3.1是对于我比较合适的版本

11. 运行时出现错误Got JS Exception: ReferenceError: Can't find variable: process，并在npm install时提示react-native@0.35.0 requires a peer of react@~15.3.1 but none was installed.

	这个问题同样是因为react的版本不对执行
	
	``` bash
	npm install react@~15.3.1 --save
	```
	问题fix