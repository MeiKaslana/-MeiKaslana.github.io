# IOS逆向研究
ios逆向工程可以分为四步
砸壳，dump，hook，重签名
# 一、砸壳
在我们打出dis包上传app store，苹果审核通过之后，苹果会在原ipa包的基础上进行加密，使得ipa包中的二进制文件无法被dump
现在解密主要有两种方法，暴力破解，或者使用砸壳工具砸壳
暴力破解要求高，且实行困难，不考虑
但虽然包是加密过的，但在iphone内存中运行的文件是解密的，我们可以在越狱设备中读取内存中的文件，将其复制出来，就是解密后的ipa包，这个步骤即是砸壳
我找到有3种砸壳工具，frida，clutch，dumpdecrypted
## [1.dumpdecrypted](https://github.com/stefanesser/dumpdecrypted)
我首先采用dumpdecrypted，但make的过程报错了
dyld: Symbol not found: ___chkstk_darwin
  Referenced from: dumpdecrypted.dylib
  Expected in: /usr/lib/libSystem.B.dylib
 in dumpdecrypted.dylib
 查询到需要下载与越狱机的ios版本对应的xcode版本，再调用sudo xcode-select -s /Users/xxx/Desktop/Xcode10.app/Contents/Developer，考虑到下载10个g的旧版本xcode太麻烦了，放弃这个方法

## [2.frida](https://github.com/frida/frida)
通过cydia安装frida，但又遇到了问题，cydia添加frida源后，frida安装失败，查过发现最新版不支持ios8.3了，但cydia又没有历史版本功能，只能在github上找到对应的旧版本12.8.10frida的deb，通过ssh传到越狱机中，借助dpkg安装，查看cydia，显示安装旧版本成功，ma也同样需要安装旧版本，但用pip安装firda和相关需要的库时因为python2，3的环境问题已经有问题，无奈放弃该方法

## [3.clutch](https://github.com/KJCracks/Clutch)
通过ssh把clutch拷贝到越狱机/usr/bin目录下，chmod +x Clutch授予权限，调用Clutch -d <bundleid or number>，在终端会输出砸壳后的ipa包路径，将其下载下来，就能得到砸壳包

# 二、dump

# 三、hook
hook方面我们使用[MonkeyDev](https://github.com/AloneMonkey/MonkeyDev) 按照wiki安装配件，在xcode中创建monkeydev工程

# 四、重签名