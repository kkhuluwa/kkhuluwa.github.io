---
title: Gradle动态常量设置
date: 2017-09-06 14:44:48
tags: Android
---
# Gradle动态配置 （1）
## 什么是BuildTypes、Flavors、BuildVariants
**BuildTypes**：构建类型，AndroidStudio的Gradle组件默认提供给了“debug”“release”两个默认配置，此处用于配置是否需要混淆、是否可调试等 

**Flavors**：产品渠道，默认不提供任何默认配置，在实际发布中，根据不同渠道，我们可能需要用不同的包名，服务器地址等

**BuildVariants**：每个buildtype和flavor组成一个buildvariant
>
<!--more-->
```
	productFlavors {
		shichang360 {
		//设置manifest变量
			manifestPlaceholders = [INSTALL_CHANNEL_VALUE: "360"]
		}
		wandoujia {
			manifestPlaceholders = [INSTALL_CHANNEL_VALUE: "wandoujia"]
		}
		yingyongbao {
			manifestPlaceholders = [INSTALL_CHANNEL_VALUE: "yingyongbao"]
		}
	}
```
也就是说，可以根据三个市场版本，与三种不同的buildType 形成九个组合，既9个BuildVariants
```
	buildTypes {

		//内网测试混淆版本
		releaseTest {
			buildConfigField("String", "APP_DOMAIN_URL", "\"http://172.25.50.50:8105\"")
			buildConfigField("String", "APP_DOMAIN_NO_SE_URL", "\"http://172.25.50.51:8008\"")
			buildConfigField("String", "SAFEGUARD_RELEASE_KEY", "\"DixGK928Zqyge0BR2SC23ZzYgkm4SAcU\"")
			buildConfigField("String", "APP_FEEDBACK_URL", "\"http://172.24.9.15:8101/app/help\"")
			// 移除无用的resource文件 动态资源绑定会被删除，暂时关闭
			shrinkResources false
			//zipAlign 优化 http://www.jizhuomi.com/android/environment/203.html
			zipAlignEnabled true
			minifyEnabled true
			proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro', 'proguard-robile-rules.pro'
			signingConfig signingConfigs.config
		}
		//debug运行版本
		debug {

			buildConfigField("String", "APP_DOMAIN_URL", "\"http://172.25.50.50:8105\"")
			buildConfigField("String", "APP_DOMAIN_NO_SE_URL", "\"http://172.25.50.51:8008\"")
			buildConfigField("String", "SAFEGUARD_RELEASE_KEY", "\"gQhUN4JRQAinOuM4+c/qLDFGE2cg1MIsdyjao0j8y60=\"")
			buildConfigField("String", "APP_FEEDBACK_URL", "\"http://172.24.9.15:8101/app/help\"")
			zipAlignEnabled false
			minifyEnabled false
			signingConfig signingConfigs.config
			manifestPlaceholders = [INSTALL_CHANNEL_VALUE: "test"]
		}
	}
```
默认debug运行时设置渠道为“test”
## AndroidManifest配置
```
	<application
		android:name="com.jd.wallet.bankapp.App"
		android:allowBackup="false"
		android:icon="@mipmap/ic_launcher"
		android:label="@string/app_name"
		android:supportsRtl="true"
		android:theme="@style/AppTheme">
		<meta-data
			android:name="INSTALL_CHANNEL_VALUE"
			android:value="${INSTALL_CHANNEL_VALUE}"/>
```
## Application读取渠道号码
```
		//获取渠道信息
		ApplicationInfo applicationInfo = null;
		try {
			applicationInfo = getPackageManager().getApplicationInfo(getPackageName(), PackageManager.GET_META_DATA);
		} catch (PackageManager.NameNotFoundException e) {
			e.printStackTrace();
		}
		if (applicationInfo != null) {
			String value = applicationInfo.metaData.getString("INSTALL_CHANNEL_VALUE");
			RunningContext.setInstallValue(value);
//			System.out.println("INSTALL_CHANNEL_VALUE" + value);
//			JDRToast.makeText(this, "INSTALL_CHANNEL_VALUE	" + value).show();
		}
```
参照钱包之前做法，渠道号用于接口请求公共参数上，至此，渠道号设置完毕。

[Gradle 语法](https://mp.weixin.qq.com/s/dNLqPUYsmzG7qyhU1O9hoA)


> [http://blog.csdn.net/angusing/article/details/47721765](http://blog.csdn.net/angusing/article/details/47721765)
> [http://blog.csdn.net/tiankong1206/article/details/50433558](http://blog.csdn.net/tiankong1206/article/details/50433558)
> [http://blog.csdn.net/xx326664162/article/details/48467343](http://blog.csdn.net/xx326664162/article/details/48467343)