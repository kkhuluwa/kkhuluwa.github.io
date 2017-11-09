---
title: 2017-11-8 Android唯一设备id分析
date: 2017-11-08 23:57:59
tags: Android 
---
## Android DID 问题

**Android DID**,全称是**Android Device ID**，及Android设备的唯一标识id。Android中系统中并没有可靠获取设备唯一id的方法，这个是业内的一种无奈，最主要的原因，应该是各家厂商以及不同的Android版本。

<!-- more -->

### 一、设备动态获取信息
昨天接到付款码需求，需要唯一确认设备id来做数据校验等操作。由于Android目前的问题，我们没有办法按照某一条设备值获取这条信息，那Android中，我们可以取的数据有哪些呢？

#### 1. DEVICE_ID
先说一下最常用的**IMEI** (International Mobile Equipment Identity)，国际移动设备身份码的缩写，国际移动装备辨识码，是由15位数字组成的“电子串号”，它与每台移动电话机一一对应，而且该码是全世界唯一的。每一只移动电话机在组装完成后都将被赋予一个全球唯一的一组号码，这个号码从生产到交付使用都将被制造生产的厂商所记录。

**MEID** 移动设备识别码(Mobile Equipment Identifier)是CDMA手机的身份识别码，也是每台CDMA手机或通讯平板唯一的识别码。通过这个识别码，网络端可以对该手机进行跟踪和监管。用于CDMA制式的手机。MEID的数字范围是十六进制的，和IMEI的格式类似

**ESN**是电子序列号Electronic Serial Number的缩写，CDMA网络术语，它是一个32bits长度的参数，它可唯一标识一台CDMA移动台设备，类似于GSM网络中的IMEI

Android系统中通常用下面这段代码获取。
```
/**
 * 获取手机IMEI号
 * 
 * 需要动态权限: android.permission.READ_PHONE_STATE
 */
public static String getDeviceId(Context context) {
        TelephonyManager telephonyManager = (TelephonyManager) context.getSystemService(context.TELEPHONY_SERVICE);
        String imei = telephonyManager.getDeviceId();

        return imei;
    }
```

手机设备：手机制式为 GSM 时，返回手机的 IMEI 。手机制式为 CDMA 时，返回手机的 MEID 或 ESN。

非手机设备：如平板电脑、电子书、电视、音乐播放器等。这些设备没有通话的硬件功能，系统中也就没有TELEPHONY_SERVICE，自然也就无法通过上面的方法获得DEVICE_ID，因此为null

优点：
1. 全球设备唯一
2. Android手机设备绝对有该值，且大于等于1（双卡设备可能有两个）

缺点：
1. 获取需要权限，手机厂商会提醒用户是否赋予电话权限，360等第三方安全应用也会提醒拦截，Android系统在6.0版本之后加入了权限控制，默认进入APP之后都需要用户申请权限。
2. 厂商定制系统中存在bug,少数手机设备上，由于该实现有漏洞，会返回垃圾，如:zeros或者asterisks

#### 2. IMSI
**IMSI** 国际移动用户识别码（IMSI：International Mobile Subscriber Identification Number）是区别移动用户的标志，储存在SIM卡中，可用于区别移动用户的有效信息。

不同于手机设备的标识IMEI
：**IMEI是与手机定**绑的。**IMSI是与SIM**（Subscriber Identity Module，用户识别模块）**或者USIM**（Universal Subscriber Identity Module，全球用户身份模块）绑定的

```
/**
 * 获取手机IMSI号
 * 需要动态权限: android.permission.READ_PHONE_STATE
 */
 public static String getIMSI(Context context){
    TelephonyManager mTelephonyMgr = (TelephonyManager) context.getSystemService(Context.TELEPHONY_SERVICE);
    String imsi = mTelephonyMgr.getSubscriberId();

    return imsi ;
}

```
优点：
1. 与用户的SIM卡绑定，可确认是某个运营商的客户。

缺点：
1. 同样在获取方式上需要动态申请权限，与IMEI缺点相同。
2. 不能插入SIM卡的设备绝对获取不到，且需要用户插SIM卡是才能读取


#### MAC ADDRESS
**MAC（Media Access Control，介质访问控制）地址**，也叫硬件地址，长度是48比特（6字节），由16进制的数字组成，分为前24位和后24位：
前24位叫做组织唯一标志符（Organizationally Unique Identifier，即OUI），是由IEEE的注册管理机构给不同厂家分配的代码，区分了不同的厂家。
后24位是由厂家自己分配的，称为扩展标识符。同一个厂家生产的网卡中MAC地址后24位是不同的。

MAC地址对应于OSI参考模型的第二层数据链路层，工作在数据链路层的交换机维护着计算机MAC地址和自身端口的数据库，交换机根据收到的数据帧中的“目的MAC地址”字段来转发数据帧。

Android中获取方法
```
WifiManager wifiManager=(WifiManager) getSystemService(Context.WIFI_SERVICE);
WifiInfo wifiInfo=wifiManager.getConnectionInfo();
String mac=wifiInfo.getMacAddress();
```
这种方法比较通用，但是最近在Android 6.0系统上，这个方法失效了，返回了”02:00:00:00:00:00”的常量。这并不是一个BUG，在google的博客中找到如下一段话：
```
Most notably, Local WiFi and Bluetooth MAC addresses are no longer available. 
The getMacAddress() method of a WifiInfo object and the BluetoothAdapter.getDefaultAdapter().getAddress() method 
will both return 02:00:00:00:00:00 from now on.
```
因此我们支付、闪付、付款码各业务在在6.0以上设备做了获取方式的变更，但根据我参考的资料来说，android 7.0以上获取方式可能仍有变化，[详情可点击链接](http://blog.csdn.net/chaozhung_no_l/article/details/78329371)，由于手上没有7.0以及7.1、8.0系统的设备，详细情形待验证。

优点：
1. 现有几乎每台移动设备都有WiFi和蓝牙功能，获取mac地址权限相对DeviceID简单
2. 大多数设备都开启过WiFi（国内应该没有不开WiFi的土豪，流量贵伤不起），因此可以获取该设备地址。

缺点：
1. 如果重启手机后，Wifi没有打开过，是无法获取其Mac地址的，而蓝牙是只有在打开的时候才能获取到其Mac地址。（这个还好，可以考虑授予CHANGE_WIFI_STATE权限，开关一次wifi刷一下。）
2. 有一些定制系统的目录并不一样。 例如三星的目录为"cat /sys/class/net/eth0/address"，所以是否对所以机型都有效有待验证。（需要适配）
3. 网上也有反映mac变更问题，是不是刷mac或者wifi故障导致，也不确定。
4. 并不是所有的设备都有Wifi硬件，硬件不存在自然也就得不到这一信息。（这个还好）
5. 需要 ACCESS_WIFI_STATE 权限。（这个还好，不需要动态申请）
6. 极少数设备升级系统会变更mac地址（李轶哥反馈武小周有一台这样可爱的手机）

### 设备序列号（Serial Number, SN）
Android 硬件 设备序列号
获取办法：
```
String serialNum = android.os.Build.SERIAL;
```

优点：
1. Android 2.3可以获取，非手机设备可以通过该接口获取。
2. 不需要动态申请权限
缺点
1. 在少数的一些设备上，会返回垃圾数据（比例有多少，网上没有大规模的调研数据）。

装有SIM卡的设备获取办法：
```
getSystemService(Context.TELEPHONY_SERVIEC).getSimSerialNumber();
```
有文章说，该id就是IMEI,因此对于CMDA电信设备，获取数值为null

### ANDROID_ID
在设备首次启动时，系统会随机生成一个64位的数字，并把这个数字以16进制字符串的形式保存下来，这个16进制的字符串就是ANDROID_ID，当设备被恢复出厂设置后该值会被重置。

可以通过下面的方法获取：
```
import android.provider.Settings;   
String ANDROID_ID = Settings.System.getString(getContentResolver(), Settings.System.ANDROID_ID); 
```
优点：
1.android系统版本在非2.2系统上，都是非常可靠的
2.不需要获取权限
缺点：
1. Android2.2系统上不可靠（还好，现在的设备基本都在4.0以上）
2. 厂商定制系统的Bug：不同的设备可能会产生相同的ANDROID_ID：9774d56d682e549c。(摩托罗拉好像出现过这个问题)
3. 厂商定制系统的Bug：有些设备返回的值为null
4. 设备差异：对于CDMA设备，ANDROID_ID和TelephonyManager.getDeviceId() 返回相同的值。
5. 如果某个Andorid手机被Root过的话，这个ID也可以被改变。
6. 刷机或者重置手机都会变化。


### Installtion ID
这种方式的原理是在程序安装后第一次运行时生成一个ID，该方式和设备唯一标识不一样，不同的应用程序会产生不同的ID，同一个程序重新安装也会不同。

所以这不是设备的唯一ID，但是可以保证每个用户的ID是不同的。可以说是用来标识每一份应用程序的唯一ID（即Installtion ID），可以用来跟踪应用的安装数量等（其实就是UUID）。
获取方法：
```
public class Installation {  
    private static String sID = null;  
    private static final String INSTALLATION = "INSTALLATION";  
  
    public synchronized static String id(Context context) {  
        if (sID == null) {    
            File installation = new File(context.getFilesDir(), INSTALLATION);  
            try {  
                if (!installation.exists())  
                    writeInstallationFile(installation);  
                sID = readInstallationFile(installation);  
            } catch (Exception e) {  
                throw new RuntimeException(e);  
            }  
        }  
        return sID;  
    }  
  
    private static String readInstallationFile(File installation) throws IOException {  
        RandomAccessFile f = new RandomAccessFile(installation, "r");  
        byte[] bytes = new byte[(int) f.length()];  
        f.readFully(bytes);  
        f.close();  
        return new String(bytes);  
    }  
  
    private static void writeInstallationFile(File installation) throws IOException {  
        FileOutputStream out = new FileOutputStream(installation);  
        String id = UUID.randomUUID().toString();  
        out.write(id.getBytes());  
        out.close();  
    }  
}  
```
优点：
1. 不需要权限
2. 每当一个应用安装后即可获得。
缺点
1. 删除应用重新安装后id变更。

画个图总结下吧，看图

方式 | 权限 | 变化度| 可靠性 | 可能的问题
---|--- |--- |--- |---
DEVICE_ID | 申请 | 很低低 | 高 | 6.0后动态申请用户感知度极高
MAC ADDRESS |不需要| 低 | 高 | 极少数设备取不到或者有变化
IMSI | 申请 | 低等（换卡）|高|动态申请用户感知度极高、且依赖用户插入卡
Serial Number| 不需要 | 待确认 | 待确认 | 少数设备脏数据
AndroidID| 不需要| 待确认 | 待确认 | 厂商bug及重置手机后变更
Installtion ID| 不需要|待确认 |待确认 |  并不是设备标识，是应用安装标识

### 二、设备静态信息

#### 1. android.os.Build类
android.os.Build类中,包括了这样的一些信息。我们可以直接调用 而不需要添加任何的权限和方法。
```
android.os.Build.BOARD：获取设备基板名称
android.os.Build.BOOTLOADER:获取设备引导程序版本号
android.os.Build.BRAND：获取设备品牌
android.os.Build.CPU_ABI：获取设备指令集名称（CPU的类型）
android.os.Build.CPU_ABI2：获取第二个指令集名称
android.os.Build.DEVICE：获取设备驱动名称
android.os.Build.DISPLAY：获取设备显示的版本包（在系统设置中显示为版本号）和ID一样
android.os.Build.FINGERPRINT：设备的唯一标识。由设备的多个信息拼接合成。
android.os.Build.HARDWARE：设备硬件名称,一般和基板名称一样（BOARD）
android.os.Build.HOST：设备主机地址
android.os.Build.ID:设备版本号。
android.os.Build.MODEL ：获取手机的型号 设备名称。
android.os.Build.MANUFACTURER:获取设备制造商
android:os.Build.PRODUCT：整个产品的名称
android:os.Build.RADIO：无线电固件版本号，通常是不可用的 显示unknown
android.os.Build.TAGS：设备标签。如release-keys 或测试的 test-keys 
android.os.Build.TIME：时间
android.os.Build.TYPE:设备版本类型  主要为"user" 或"eng".
android.os.Build.USER:设备用户名 基本上都为android-build
android.os.Build.VERSION.RELEASE：获取系统版本字符串。如4.1.2 或2.2 或2.3等
android.os.Build.VERSION.CODENAME：设备当前的系统开发代号，一般使用REL代替
android.os.Build.VERSION.INCREMENTAL：系统源代码控制值，一个数字或者git hash值
android.os.Build.VERSION.SDK：系统的API级别 一般使用下面大的SDK_INT 来查看
android.os.Build.VERSION.SDK_INT：系统的API级别 数字表示
```
#### 2.  build.prop中获取当前系统属性
```
D:\AndroidStudio\sdk\platform-tools>adb shell cat /system/build.prop
ro.product.dmtklog=true
ro.vivo.oem.support=yes
persist.vivo.radio.type.abbr=A
persist.vivo.radio.type.list=TD-SCDM
ro.vivo.lcm.xhd=QDHD
ro.sf.lcd_density=640
ro.vivo.lte.voice.type=CSFB
ro.vivo.net.entry=no
ro.vivo.hardware.version=PD1522AMA
ro.board.bbk=MA
ro.vivo.product.solution=QCOM
ro.vivo.product.platform=QCOM8976
ro.vivo.board.version=MA
ro.product.net.entry.bbk=no
persist.vivo.product.cust.list=N
persist.sys.strictmode.disable=true

...

```
命令查看手中vivo手机信息，发现很多厂商定制信息，暂无找到需要的信息

### 三、阿里UTDID方案分析

接到该需求后，我询问了一些之前实验室的小伙伴们，一位在滴滴的学长和我提到了阿里的utdid，扔给了我一个jar包，并表示阿里集团就是这么用的，你反编译下自己写一份~

二话不说，我去反编译，结果该jar包混淆做的比较好，几乎无法分析，于是求之于百度，[阿里系UTDID库生成唯一性ID分析](http://blog.csdn.net/justFWD/article/details/50549971)。

utdid方案核心逻辑为在外置SD卡中生成一段数字,其算法可分为三部分连接
1. 时间戳 System.currentTimeMillis()/1000
2. DEVICE_ID,若因权限等问题获取不到，则生成随机值Random
3. 随机值Random

该方案为友盟SDK主导设计，具体使用场景为各个APP嵌入友盟SDK，utdid可以在多个APP中进行维护，在使用的有友盟、虾米音乐、阿里小智、老版本的支付宝、淘宝等。

优点：
1. 多APP维护该设备id，并且在多应用中可以维护
缺点：
utdid开发手册中是强制需要下面3个权限的：
```
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" /> 
<uses-permission android:name="android.permission.WRITE_SETTINGS" /> 
<uses-permission android:name="android.permission.READ_PHONE_STATE" />
```
1. **WRITE_SETTINGS**
targetSDK使用23(6.0)以后，即使申请了WRITE_SETTINGS权限，再想写settings中的数据会抛出了如下异常。 
IllegalArgumentException: You cannot keep your settings in the secure settings. 
想写Setting.System中的数据也已经没有权限了，即使想修改Settings.Global也需要是系统应用。
2. **READ_PHONE_STATE**
targetSDK使用23以后，需要手动授权。imei作为utdid生成字段的其中一部分，在生成utdid时，如果无法获取就使用随机值代替。这个即使不授权问题也不大。
3. **WRITE_EXTERNAL_STORAGE**
targetSDK使用23以后，需要手动授权。在settings中没有写入utdid的情况下，如果没有WRITE_EXTERNAL_STORAGE权限，应用外部的utdid是无法获得的，内部也没有的情况下，那么utdid势必会重新生成了。
4. 如果settings不能保存，那么应用外就需要寄希望于sdcard的存储。除了权限问题会导致sdcard中的数据无法取得外，三方的手机管理工具也会对sdcard中的数据做清除。（utdid外存储目录用AAA，BBB表示）

### 四、总结

在Android中，基本上没有绝对可靠的数据可以获取，但昨天和释翔哥、轶哥、学贤哥的讨论，我们可以在下个版本上传以Wifi Mac地址、设备序列号、ANDROID_ID为主要基准，配合android.os.Build中手机基本信息为参考，DEVICE_ID（用READ_PHONE_STATE权限做修正）等。如果可以的话，还可以参考手机号码、pin等数据做修正，如果修正后的数据仍无法满足，那我们可以针对这部分用户做增验等异常流程设计。

这些是我个人之前的一些经验和总结，若大家有其他方案或者有我有哪些遗漏和错误，烦请大家指出，谢谢啦。





**Refer**
>https://baike.baidu.com/item/IMEI/1626447?fr=aladdin
>https://baike.baidu.com/item/MEID/408369
>http://blog.csdn.net/qq524752841/article/details/42421783
>http://blog.csdn.net/zhangcanyan/article/details/52817866
>http://blog.csdn.net/aa1733519509/article/details/50053553
