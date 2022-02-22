# IOS逆向研究
ios逆向工程可以分为四步
砸壳，dump，hook，重签名
以土豆视频为例，记录一下我逆向ipa的过程。越狱设备iphone5s，ios 8.3.1。

逆向第一步，越狱机安装openssh或者usbmuxd，连接越狱机与mac。有一台ios10.3.1的iphone6越狱机就是无法通过openssh和usbmuxd连接，无奈放弃，换了另一台越狱机。换了ios 8.3.1的越狱机后，mac终端输入ssh root@xx.xx.xx.xx，连接成功。

# 一、砸壳

在砸壳之前，我们需要找到要逆向的app的沙盒路径，才能进行砸壳，以土豆视频为例，ssh连接上越狱机后，把iPhone上的所有App都关掉，唯独保留土豆视频，然后输入命令ps -e|grep /var/mobile/Containers，找到土豆视频相关的/var/mobile/Containers/Bundle/Application/1E8DD727-C95E-4413-A3EA-5CF9A0A71322/TDMainClient.app/TDMainClient，得到土豆视频的app名。

然后越狱机安装cycript，调用cycript -p TDMainClient，在命令行下和应用交互，通过directory = NSHomeDirectory()命令获取沙河位置，@"/var/mobile/Containers/Data/Application/61372F44-72EA-44F9-A7BF-ABA1174F6C72"。

接下来我们开始砸壳，在我们打出dis包上传app store，苹果审核通过之后，苹果会在原ipa包的基础上进行加密，使得ipa包中的二进制文件无法被dump
现在解密主要有两种方法，暴力破解，或者使用砸壳工具砸壳。
暴力破解要求高，且实行困难，不考虑。

但虽然包是加密过的，但在iphone内存中运行的文件是解密的，我们可以在越狱设备中读取内存中的文件，将其复制出来，就是解密后的ipa包，这个步骤即是砸壳。
我找到有3种砸壳工具，frida，clutch，dumpdecrypted
## [1.dumpdecrypted](https://github.com/stefanesser/dumpdecrypted)
我首先采用dumpdecrypted，但make的过程报错了
dyld: Symbol not found: ___chkstk_darwin
  Referenced from: dumpdecrypted.dylib
  Expected in: /usr/lib/libSystem.B.dylib
 in dumpdecrypted.dylib
 查询到需要下载与越狱机的ios版本对应的xcode版本，再调用sudo xcode-select -s /Users/xxx/Desktop/Xcode10.app/Contents/Developer，考虑到下载10个g的旧版本xcode太麻烦了，放弃这个方法

## [2.frida](https://github.com/frida/frida)
通过cydia安装frida，但又遇到了问题，cydia添加frida源后，frida安装失败，查过发现最新版不支持ios8.3了，但cydia又没有历史版本功能，只能在github上找到对应的旧版本12.8.10frida的deb，通过ssh传到越狱机中，借助dpkg安装，查看cydia，显示安装旧版本成功，ma也同样需要安装旧版本，但用pip安装firda和相关需要的库时因为python2，3的环境问题，无奈放弃该方法

## [3.clutch](https://github.com/KJCracks/Clutch)
通过ssh把clutch拷贝到越狱机/usr/bin目录下，chmod +x Clutch授予权限，Clutch -i命令获取可砸壳应用的包名和编号，再调用Clutch -d <bundleid or number>，在终端会输出砸壳后的ipa包路径，将其下载下来，就能得到砸壳包

# 二、dump

dump阶段我们需要找到我们要hook的方法，粗略猜测ipa的运行过程，为此我们需要一些应用

## class-dump
1、下载class-dump安装文件
2、双击打开安装
3、选择复制文件：class-dump
4、粘贴到目录：/usr/local/bin
5、打开终端执行命令检查：class-dump

在终端执行命令：class-dump -H xxx -o 输出目录，即可获得应用逆向后的头文件，有些头文件类和方法的命名非常清晰，可以推测方法的作用

## hopper
Hopper Disassembler是Mac上的一款二进制反汇编器，基本上满足了工作上的反汇编的需要，包括伪代码以及控制流图(Control Flow Graph)，支持ARM指令集并针对Objective-C的做了优化。
在[官网](https://www.hopperapp.com)下载hopper并安装，免费版可以免费十几分钟，时间到了再打开就行了，还能继续白嫖。
安装好后把脱壳后的ipa包的二进制文件拖到hopper中，点击右上角的if(b) f(x)按钮，即可根据汇编代码生成对应的伪代码，方便理解。

## [MonkeyDev](https://github.com/AloneMonkey/MonkeyDev)
按照wiki安装配件和theos，配置好环境变量export THEOS=/opt/theos，终端输入 $THEOS/bin/nic.pl能找到命令不报错后，在xcode中创建monkeydev工程，把脱壳的ipa包放在TargetApp中，配置好证书，即可真机调试，在xcode中看到实时日志输出。同时可以通过xcode的DebugViewHierarchy获取当前用到了哪些uiviewcontroller。
同时monkeydev工程的dylib Target有个签名相关的小问题，需要在dylib的SETTING上添加CODE_SIGNING_ALLOWED的属性，并将其置为NO。

我先通过monkeydev查看DebugViewHierarchy界面，看看土豆视频显示用了什么类。
![avatar](/image/TDHomeViewController.jpg)


综合hopper，class-dump和MonkeyDev的日志与DebugViewHierarchy确定要hook的类和方法。在这里我决定hook土豆视频的TDHomeViewController类的init方法。
# 三、hook
monkeydev支持多种hook方式，我这里选择使用了captainhook，使用如下代码，生成dylib，将这个dylib注入二进制文件中。
真机测试后发现，虽然UIAlertView成功出现，说明hook成功了，但土豆视频的网络却一直连接不上，只能查看xcode的实时日志，尝试查找网络连接失败的原因。

![avatar](/image/fail_sys_protoparam_missed.jpg)
![avatar](/image/x-sign.jpg)

对比越狱机的实时日志，猜测是土豆视频的防逆向逻辑触发，不给网络请求的参数签名了，而签名的具体方法hopper的伪代码并不能转换成功，汇编代码阅读也非常费力，那我就猜测，土豆视频的防逆向逻辑应该与包名有关，因为越狱机的包和真机测试的包最大的不同就是包名，hook找不到签名的方法，那我就hook获取包名的[[NSBundle mainBundle] bundleIdentifier]方法，hook成功后，土豆视频果然能正常运行。
![avatar](/image/hooksuccess.jpg)

hook代码如下

//

//  HookAlert.m

//  monkeynixiangDylib

//

//  Created by qdazzle on 2022/2/10.

//



#import <Foundation/Foundation.h>

#import <CaptainHook/CaptainHook.h>

#import <UIKit/UIKit.h>

CHDeclareClass(TDHomeViewController)   //声明hook的类

//声明hook的方法,数字代码参数个数,第一个参数一般是self,第二个参数代表返回值,第三个参数是类名,第四个是方法名,后面接方法参数以及类
型

CHOptimizedMethod0(self, void, TDHomeViewController, init){

    //获取类变量view的值

    UIAlertView *alert = [[UIAlertView alloc]initWithTitle:@"提示" message:@"拦截成功"

         delegate:nil cancelButtonTitle:@"好的" otherButtonTitles: nil];

    [alert show];

   // 其他逻辑正常执行元源代码

    CHSuper0(TDHomeViewController, init);

}


CHDeclareClass(NSBundle)   //声明hook的类

CHOptimizedMethod0(self, NSString *, NSBundle, bundleIdentifier){

    NSLog(@"hook bundleid success");

    return @"com.tudou.tudouiphone";

}



// 构造方法,

CHConstructor{

    CHLoadLateClass(TDHomeViewController);  //再次声明hook的类

    CHClassHook0(TDHomeViewController, init);  //再次声明hook的方法

    CHLoadLateClass(NSBundle); 

    CHClassHook0(NSBundle, bundleIdentifier);  

}

# 四、重签名
重签名的步骤有：

1、用要重签名的证书打一个包，在包中获取embedded.mobileprovision

2、替换 embedded.mobileprovision文件

3、删除PlugIns，Watch文件夹

4、对所有framework，dylib和所有其他文件用codesign命令签名，codesign -f -s 证书名字 目标文件，证书名可以在钥匙串中找到

5、修改info.plist的bundleid

6、zip -r XX.ipa Payload

但在monkeydev生成app包的时候会生成一个shell脚本，调用即可生成重签名包。原理是因为monkeydev直接打出了app包，把app包放到Payload文件夹中压缩即可得到ipa包。