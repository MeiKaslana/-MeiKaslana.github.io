# IOS逆向防护
学习到了如何逆向ipa包，那我们才可以知己知彼，了解到如何防止被逆向。
网络上查到的防止逆向的方法有以下几个

## 白名单（难维护）
在objective-c中我们可以通过以下代码获取到ipa包用到的dylib

#include<mach-o/dyld.h>

int count = _dyld_image_count();

for (int i = 1; i < count; i++) {

    //打印所有image name,用,隔开
    printf("%s,",_dyld_get_image_name(i));
}

我试过在多个模拟器上运行这段代码，打印出来的dylib都有很大不同，不同ios版本的动态库路径就不一样，ios版本更新之后比较难维护。而且用到新的库之后就要修改白名单。同时，白名单不能直接放在本地，最后通过服务器传输或者对字符串加密，字符串是可以被strings方法解密二进制文件导出来的。

## 检查bundleid（不推荐）
原理上一篇文章说过了，[[NSBundle mainBundle] bundleIdentifier]也是可以被逆向的，并不准。

## Ptrace防护
//参数1：PT_DENY_ATTACH 表示当前进程不允许被附加
//参数2：进程id，0就是本进程
//参数3：地址，根据参数1而定，这里传0
//数据4：数据，根据参数1而定，这里传0

ptrace(PT_DENY_ATTACH, 0, 0, 0);

APP中调用ptrace()函数，那么lldb断点调试APP时就会奔溃，而正常点开APP却可以运行。

## sysctl防护
sysctl函数可以查询进程是否被附加

BOOL isAttached(void){

    int name[4];             //里面放字节码。查询的信息

    name[0] = CTL_KERN;      //内核查询

    name[1] = KERN_PROC;     //查询进程

    name[2] = KERN_PROC_PID; //传递的参数是进程的ID

    name[3] = getpid();      //获取当前进程ID

    struct kinfo_proc info;  //接受查询结果的结构体

    size_t info_size = sizeof(info);  //结构体大小

    if(sysctl(name,4, &info, &info_size, 0, 0)){

        NSLog(@"查询失败");

        return NO;

    }

    /*
     查询结果看info.kp_proc.p_flag 的第12位，如果为1，表示调试附加状态。
     info.kp_proc.p_flag & P_TRACED 即可获取第12位
    */
  
  return ((info.kp_proc.p_flag & P_TRACED) != 0);

}

## dlopen动态调用防护
但MonkeyAPP就默认使用了fishhook拦截了Ptrace，dlsym、syscall、sysctl函数，所以单纯调用上述防护可能并不会凑效。所以我们需要使用动态调用的方式隐藏sysctl符号

 //A异或B等到C,C再异或A得到B,隐藏sysctl

unsigned char str[] = 

{

             ('q' ^ 's'),

             ('q' ^ 'y'),

             ('q' ^ 's'),

             ('q' ^ 'c'),

             ('q' ^ 't'),

             ('q' ^ 'l'),

             ('q' ^ '\0')

      };

      unsigned char * p = str;

      while (((*p) ^= 'q') != '\0') p++;

      int (*m_sysctl)(int *, u_int, void *, size_t *, void *, size_t);

     void* handle = dlopen(0, RTLD_GLOBAL | RTLD_NOW);//获得具柄

       m_sysctl=dlsym(handle,(const char *)str);//动态查找sysctl符号

}

这样利用IDA等工具已经找不到sysctl外部符号了

## 汇编防护
查询到被逆向直接崩溃的话，会给闪退日志留下信息，然后定位到防护工具的位置，从而被hook或者直接修改二进制文件的汇编代码返回NOP，导致应用被逆向，所以我们可以通过汇编代码调用Ptrace等，然后使用svc软中断，同时清除堆栈信息。清除堆栈信息后sbt就看不到任何调用堆栈。


## 个人想法
上面都是非常专业而且常用的防护方式，然后我来说说根据之前的逆向经历，我想到的可能能进一步防止逆向的方法

### embedded.mobileprovision
对比了一下app store下载的包和一些被逆向过的包：

app store下载的包，砸壳包，monkeydev的模拟器调试包，没有embedded.mobileprovision
monkeydev的真机测试包，重签名包有embedded.mobileprovision
一般来说，禁止游戏包在模拟器运行，且确保找不到embedded.mobileprovision，可以尽可能避免被逆向。

//app store下载的包没有embedded.mobileprovision,可以用来判断包是否被人从app store下载后处理过
 
NSString *embeddedPath = [[NSBundle mainBundle] pathForResource:@"embedded" ofType:@"mobileprovision"];

if (! [[NSFileManager defaultManager] fileExistsAtPath:embeddedPath]) 

{
    
	NSLog(@"ipa safe");

	return true;
    
}

### 避免抓包
ios包防止被抓包，避免网络请求暴露方法名与变量名。土豆视频就是通过对比成功和失败的网络请求，发现是签名导致的问题。

//客户端判断当前是否设置了代理,如果设置了代理,不允许进行访问

+ (BOOL)getProxyStatus 

{
    
	NSDictionary *proxySettings = NSMakeCollectable([(NSDictionary *)CFNetworkCopySystemProxySettings() autorelease]);
    
	NSArray *proxies = NSMakeCollectable([(NSArray *)CFNetworkCopyProxiesForURL((CFURLRef)[NSURL URLWithString:@"http://www.google.com"], (CFDictionaryRef)proxySettings) autorelease]);
    
	NSDictionary *settings = [proxies objectAtIndex:0];

	NSLog(@"host=%@", [settings objectForKey:(NSString *)kCFProxyHostNameKey]);
    
	NSLog(@"port=%@", [settings objectForKey:(NSString *)kCFProxyPortNumberKey]);
    
	NSLog(@"type=%@", [settings objectForKey:(NSString *)kCFProxyTypeKey]);
    
    if ([[settings objectForKey:(NSString *)kCFProxyTypeKey] isEqualToString:@"kCFProxyTypeNone"])
    
	{
    
	    //没有设置代理
    
	    return NO;
    
	}
    
	else
    
	{
    
	    //设置代理了
    
	    return YES;
    
	}

}

2.客户端本地做证书校验,并且设置不仅仅校验公钥,设置完整的正式校验模式
+(AFSecurityPolicy*)customSecurityPolicy

{

    // /先导入证书
    NSString *cerPath = [[NSBundle mainBundle] pathForResource:@"test" ofType:@"cer"];//证书的路径
    NSData *certData = [NSData dataWithContentsOfFile:cerPath];
    // AFSSLPinningModeCertificate 使用证书验证模式 (AFSSLPinningModeCertificate是证书所有字段都一样才通过认证，AFSSLPinningModePublicKey只认证公钥那一段，AFSSLPinningModeCertificate更安全。但是单向认证不能防止“中间人攻击”)
    AFSecurityPolicy *securityPolicy = [AFSecurityPolicy policyWithPinningMode:AFSSLPinningModeCertificate];
    // allowInvalidCertificates 是否允许无效证书（也就是自建的证书），默认为NO
    // 如果是需要验证自建证书，需要设置为YES
    securityPolicy.allowInvalidCertificates = YES;

    //validatesDomainName 是否需要验证域名，默认为YES；
    //假如证书的域名与你请求的域名不一致，需把该项设置为NO；如设成NO的话，即服务器使用其他可信任机构颁发的证书，也可以建立连接，这个非常危险，建议打开。

    //置为NO，主要用于这种情况：客户端请求的是子域名，而证书上的是另外一个域名。因为SSL证书上的域名是独立的，假如证书上注册的域名是www.google.com，那么mail.google.com是无法验证通过的；当然，有钱可以注册通配符的域名*.google.com，但这个还是比较贵的。
    //如置为NO，建议自己添加对应域名的校验逻辑。
    securityPolicy.validatesDomainName = YES;
    NSSet<NSData*> * set = [[NSSet alloc]initWithObjects:certData  , nil];
    securityPolicy.pinnedCertificates = set;
    
     
    return securityPolicy;

}

### 混淆
混淆方法名和类名可以增加逆向者理解代码逻辑的难度，无法简单通过hopper就理清楚应用的运行逻辑。同时，混淆字符串可以避免日志和关键参数名暴露，可能提示逆向者该方法的用途。字符串混淆可以用char[]组合，或者配合stringByReplacingOccurrencesOfString替换字符串，把字符串拆解；又或者对所有字符串加密，用到时在对加密过的字符串解密。