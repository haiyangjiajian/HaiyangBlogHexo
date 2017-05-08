---
layout: post
title: react native与android native代码的集成开发 
tags: [react_native, android]
category: 编程
---

这篇文章讲的不是[native module](https://facebook.github.io/react-native/docs/native-modules-android.html)相关内容，即如何在react native的js代码中调用androd的native代码。也不是[native components](https://facebook.github.io/react-native/docs/native-components-android.html)相关内容，即如何封装一些原生的组建供react native层使用。这里主要想讲如何在android的应用中，集成react native的页面和如何在react native中启动android activity。这样能达到原生android页面和react native页面互相切换的目的，从而使开发更加灵活。


这篇博客的示例代码在[TestRNIntegrationNative](https://github.com/haiyangjiajian/TestRNIntegrationNative)可供参考运行

---

## android应用中集成react native

这部分内容主要从[官方文档Integration With Existing Apps ](https://facebook.github.io/react-native/docs/integration-with-existing-apps.html)android版本翻译而来，根据自己的理解翻译，不是严格对照原文，加入了一些自己在实践过程中碰到的问题

### 关键内容
react native对于从头建立一些新的app非常好用。它同样可以用于在一个已有app的基础上加入一些页面。通过下面几个步骤，可以在已有的app上加入一些基本的react native的功能，页面，view组建等。

将react native集成到已有andriod app的关键步骤有如下几个：

1. 理解你想要集成react native的哪个组件
2. 在你的Android项目的根目录下安装react-native，创建node_modules目录
3. 用react-native写你需要的组件
4. 在你的build.gradle文件中添加com.facebook.react:react-native:+和指向node_module下react-native的二进制文件的maven引用
5. 在android中写一个activity，这个activity创建一个ReactRootView
6. 打开react-native server，运行app
7. 增加更多的组件
8. [调试](https://facebook.github.io/react-native/releases/next/docs/debugging.html)
9. [准备](https://facebook.github.io/react-native/releases/next/docs/signed-apk-android.html) [发布](https://facebook.github.io/react-native/docs/running-on-device-android.html)
10. 部署优化


下面将详细介绍具体步骤

### 准备
按照[getting-started](https://facebook.github.io/react-native/docs/getting-started.html)将需要的配置装好

### 在app中加入js

在app的根目录下执行

``` bash
npm init
npm install --save react react-native
curl -o .flowconfig https://raw.githubusercontent.com/facebook/react-native/master/.flowconfig
```

打开package.json，在scripts中加入

```
"start": "node node_modules/react-native/local-cli/cli.js start"
```
在index.android.js中加入如下代码，如index.android.js不存在，则在根目录创建一个



``` javascript
'use strict';
import React from 'react';
import {
  AppRegistry,
  StyleSheet,
  Text,
  View
} from 'react-native';

class HelloWorld extends React.Component {
  render() {
    return (
      <View style={styles.container}>
        <Text style={styles.hello}>Hello, World</Text>
      </View>
    )
  }
}
var styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
  },
  hello: {
    fontSize: 20,
    textAlign: 'center',
    margin: 10,
  },
});

AppRegistry.registerComponent('HelloWorld', () => HelloWorld);
```

### 配置当前app

在app的build.gradle文件中加入如下对react native的依赖

```
dependencies {
    ...
    compile "com.facebook.react:react-native:+" // From node_modules.
}
```

如果你想要指定react native的版本，可以将+换成具体的版本号

在工程的build.gradle中指定所依赖的本地react native的地址

```
allprojects {
    repositories {
        ...
        maven {
            // All of React Native (JS, Android binaries) is installed from npm
            url "$rootDir/node_modules/react-native/android"
        }
    }
    ...
}

```
在AndroidManifest.xml中增加网络访问权限

```
<uses-permission android:name="android.permission.INTERNET" />
```


### 增加native代码

需要有native的代码来启动React Native，并开始渲染。我们在android native代码中新建一个activity，ReactActivity，其中创建了ReactRootView。通过它来启动React Native相关程序。

tips1: 如果Android version <5,用com.android.support:appcompat下的AppCompatActivity来取代Activity。

tips2：如果app执行的时候报出Didn't find class "com.facebook.jni.IteratorHelper"的异常，取消掉setUseOldBridge这句的注释



``` java
public class MyReactActivity extends Activity implements DefaultHardwareBackBtnHandler {
    private ReactRootView mReactRootView;
    private ReactInstanceManager mReactInstanceManager;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        mReactRootView = new ReactRootView(this);
        mReactInstanceManager = ReactInstanceManager.builder()
                .setApplication(getApplication())
                .setBundleAssetName("index.android.bundle")
                .setJSMainModuleName("index.android")
                .addPackage(new MainReactPackage())
                .setUseDeveloperSupport(BuildConfig.DEBUG)
                .setInitialLifecycleState(LifecycleState.RESUMED)
                //.setUseOldBridge(true) // uncomment this line if your app crashes
                .build();
        mReactRootView.startReactApplication(mReactInstanceManager, "HelloWorld", null);

        setContentView(mReactRootView);
    }

    @Override
    public void invokeDefaultOnBackPressed() {
        super.onBackPressed();
    }
}
```



其中的HelloWorld，需要与index.android.js中AppRegistry.registerComponent()的第一个参数匹配

在AndroidManifest.xml中为ReactActivity指定主题。

``` xml
<activity
    android:name=".ReactActivity"
    android:label="@string/app_name"
    android:theme="@style/Theme.AppCompat.Light.NoActionBar">
</activity>
```




我们需要传递一些callback到ReactInstanceManager中

``` java
@Override
protected void onPause() {
    super.onPause();

    if (mReactInstanceManager != null) {
        mReactInstanceManager.onPause();
    }
}

@Override
protected void onResume() {
    super.onResume();

    if (mReactInstanceManager != null) {
        mReactInstanceManager.onResume(this, this);
    }
}

@Override
protected void onDestroy() {
    super.onDestroy();

    if (mReactInstanceManager != null) {
        mReactInstanceManager.onDestroy();
    }
}
```


传递对退出事件的处理到react native中

``` java
@Override
 public void onBackPressed() {
    if (mReactInstanceManager != null) {
        mReactInstanceManager.onBackPressed();
    } else {
        super.onBackPressed();
    }
}
```



这将允许JavaScript可以控制用户按下物理退出键时的处理。如果JS层没有特殊处理，会调用默认的invokeDefaultOnBackPressed，关闭当前默认的activity。

然后我们需要增加React Native开发的调试菜单。默认React Native是通过摇手机来调出调试菜单的，但是在模拟器上却不能实现。所有我们设置当按android的物理菜单键时，出现React Native 的开发调试菜单（也可以按Ctrl + M）


``` java
@Override
public boolean onKeyUp(int keyCode, KeyEvent event) {
    if (keyCode == KeyEvent.KEYCODE_MENU && mReactInstanceManager != null) {
        mReactInstanceManager.showDevOptionsDialog();
        return true;
    }
    return super.onKeyUp(keyCode, event);
}
```

### 运行app

在你已有的app中的某个activity里可以启动我们的ReactActivity了。像这样

``` java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Button bt = (Button)findViewById(R.id.gotoRN);
        bt.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Intent intent = new Intent();
                intent.setClass(MainActivity.this, ReactActivity.class);
                startActivity(intent);
                MainActivity.this.finish();
            }
        });
    }
}

```


打开React Native dev server。在根目录运行


``` bash
npm start
```

然后在android studio里像往常一样运行app就可以了

tips1: 如果你使用Android Studio来build这个项目，需要在运行npm start之前安装[watchman](https://facebook.github.io/watchman/),可以防止Android Studio和React Native包冲突引起的崩溃

运行如图

<img src="/assets/img/rn/native-view.png" width = "300" height = "200" alt="native view" />

<img src="/assets/img/rn/react-native-view.png" width = "300" height = "200" alt="native view" />


### 用Android Studio打包发布

相比较于普通的app打包，要多了一个步骤，在用Android Studio打包之前，执行如下命令生成React Native的bundle。把路径替换成你的真实路径

```bash
$ react-native bundle --platform android --dev false --entry-file index.android.js --bundle-output android/com/your-company-name/app-package-name/src/main/assets/index.android.bundle --assets-dest android/com/your-company-name/app-package-name/src/main/res/
```

## 在react native的JavaScript代码中启动activtiy

实现的核心是在native层封装一个Java的方法供JavaScript调用，封装[native module](https://facebook.github.io/react-native/docs/native-modules-android.html)。这个Java方法提供了启动其他activity的功能。

### 增加native代码

首先新建一个Module，这个module中封装了启动一个native activity的方法gotoMainActivity

```java
public class NativeActivityModule extends ReactContextBaseJavaModule {

    public NativeActivityModule(ReactApplicationContext reactContext) {
        super(reactContext);
    }


    @Override
    public String getName() {
        return "GotoActivity";
    }

    @ReactMethod
    public void gotoMainActivity() {
        Activity currentActivity = getCurrentActivity();
        Intent intent = new Intent(currentActivity, MainActivity.class);
        currentActivity.startActivity(intent);
        currentActivity.finish();
    }
}

```

新建一个package封装这个module

```java
public class NativeActivityPackage implements ReactPackage {
    @Override
    public List<NativeModule> createNativeModules(ReactApplicationContext reactContext) {
        List<NativeModule> modules = new ArrayList<>();

        modules.add(new NativeActivityModule(reactContext));

        return modules;
    }

    @Override
    public List<ViewManager> createViewManagers(ReactApplicationContext reactContext) {
        return Collections.emptyList();
    }

    @Override
    public List<Class<? extends JavaScriptModule>> createJSModules() {
        return Collections.emptyList();
    }
}

```

将这个NativeActivityPackage，增加到我们之前创建的ReactActivity中，主要是对mReactInstanceManager增加一个package，addPackage(new NativeActivityPackage())

```java
protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        mReactRootView = new ReactRootView(this);
        mReactInstanceManager = ReactInstanceManager.builder()
            .setApplication(getApplication())
            .setBundleAssetName("index.android.bundle")
            .setJSMainModuleName("index.android")
            .addPackage(new MainReactPackage())
            .setUseDeveloperSupport(BuildConfig.DEBUG)
            .setInitialLifecycleState(LifecycleState.RESUMED)
            .addPackage(new NativeActivityPackage())
            //.setUseOldBridge(true) // uncomment this line if your app crashes
            .build();
        mReactRootView.startReactApplication(mReactInstanceManager, "HelloWorld", null);

        setContentView(mReactRootView);
    }}

```

### 在JavaScript层中调用封装的方法

```javascript
const GotoActivity = NativeModules.GotoActivity;

<TouchableOpacity style={styles.button} onPress={() => {GotoActivity.gotoMainActivity()}}>
            <Text style={styles.gotoNative}>Goto Native</Text>
</TouchableOpacity>

```

至此我们完成了在android的应用中，集成react native的页面和如何在react native中启动android activity，这样能达到原生android页面和react native页面互相切换的目的。这篇博客的示例代码在[TestRNIntegrationNative](https://github.com/haiyangjiajian/TestRNIntegrationNative)可供参考运行

