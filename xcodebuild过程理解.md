# xcodebuild过程理解

我们先根据xcodebuild命令的日志输出看看xcodebuild做了什么

build过程没有设置的话默认使用new build system
# prepare build过程
## 创建xcodebuild任务中需要的文件夹
先是在DerivedData下以target名创建的文件夹下创建build文件夹，里面包含Intermediates.noindex/XCBuildData，Intermediates.noindex/{target}.build,以及Product/Debug-iphoneos(具体名字与正式环境测试环境，真机还是模拟器有关)/{target}.app

## 创建embedded.mobileprovision和Entitlements.plist
将xcode用到的证书文件复制到{target}.app文件夹下，改名为embedded.mobileprovision。同时，根据Xcode的capability生成Entitlements.plist，授权赋予应用一定的特定的功能或安全权限。

## 创建hmap文件
hmap是header map的实体，类似于一个Key-value的形式，Key值是头文件的名称，Value是头文件的实际物理路径。xcode在第一次编译的时候会帮我们生成这些hmap文件，再次编译的时候可以通过这些文件可以快速找到对应的头文件，所以编译速度会快许多。也可以通过这个文件提升编译速度，详情参考这个文章[这个文章](https://tech.meituan.com/2021/02/25/cocoapods-hmap-prebuilt.html)

## 调用run script脚本
调用xcode工程build phases中的添加的run script脚本，以及一些pod添加的[CP]Check Pods Manifest.lock，但发现有些脚本在prepare build时调用，有些脚本比如cocoa的[CP] Embed Pods Frameworks在complie时调用，光看xcode工程配置没看出原因，之后再研究。

# Build target过程
## CompileC
xcode会complie所有的.m文件生成.o，我们以一个SceneDelegate.m举例。
```> /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang -x objective-c -target arm64-apple-ios15.0 -fmessage-length\=0 -fdiagnostics-show-note-include-stack -fmacro-backtrace-limit\=0 -std\=gnu11 -fobjc-arc -fobjc-weak -fmodules -gmodules -fmodules-cache-path\=/Users/chenyue/Library/Developer/Xcode/DerivedData/ModuleCache.noindex -fmodules-prune-interval\=86400 -fmodules-prune-after\=345600 -fbuild-session-file\=/Users/chenyue/Library/Developer/Xcode/DerivedData/ModuleCache.noindex/Session.modulevalidation -fmodules-validate-once-per-build-session -Wnon-modular-include-in-framework-module -Werror\=non-modular-include-in-framework-module -fmodule-name\=xiaoyima -Wno-trigraphs -fpascal-strings -O0 -fno-common -Wno-missing-field-initializers -Wno-missing-prototypes -Werror\=return-type -Wdocumentation -Wunreachable-code -Wquoted-include-in-framework-header -Wno-implicit-atomic-properties -Werror\=deprecated-objc-isa-usage -Wno-objc-interface-ivars -Werror\=objc-root-class -Wno-arc-repeated-use-of-weak -Wimplicit-retain-self -Wduplicate-method-match -Wno-missing-braces -Wparentheses -Wswitch -Wunused-function -Wno-unused-label -Wno-unused-parameter -Wunused-variable -Wunused-value -Wempty-body -Wuninitialized -Wconditional-uninitialized -Wno-unknown-pragmas -Wno-shadow -Wno-four-char-constants -Wno-conversion -Wconstant-conversion -Wint-conversion -Wbool-conversion -Wenum-conversion -Wno-float-conversion -Wnon-literal-null-conversion -Wobjc-literal-conversion -Wshorten-64-to-32 -Wpointer-sign -Wno-newline-eof -Wno-selector -Wno-strict-selector-match -Wundeclared-selector -Wdeprecated-implementations -DDEBUG\=1 -DOBJC_OLD_DISPATCH_PROTOTYPES\=0 -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS15.2.sdk -fstrict-aliasing -Wprotocol -Wdeprecated-declarations -g -Wno-sign-conversion -Winfinite-recursion -Wcomma -Wblock-capture-autoreleasing -Wstrict-prototypes -Wno-semicolon-before-method-body -Wunguarded-availability -fembed-bitcode-marker -index-store-path /Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Index/DataStore -iquote /Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Intermediates.noindex/xiaoyima.build/Debug-iphoneos/xiaoyima.build/xiaoyima-generated-files.hmap -I/Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Intermediates.noindex/xiaoyima.build/Debug-iphoneos/xiaoyima.build/xiaoyima-own-target-headers.hmap -I/Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Intermediates.noindex/xiaoyima.build/Debug-iphoneos/xiaoyima.build/xiaoyima-all-non-framework-target-headers.hmap -ivfsoverlay /Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Intermediates.noindex/xiaoyima.build/Debug-iphoneos/xiaoyima.build/all-product-headers.yaml -iquote /Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Intermediates.noindex/xiaoyima.build/Debug-iphoneos/xiaoyima.build/xiaoyima-project-headers.hmap -I/Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Products/Debug-iphoneos/include -I/Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Intermediates.noindex/xiaoyima.build/Debug-iphoneos/xiaoyima.build/DerivedSources-normal/arm64 -I/Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Intermediates.noindex/xiaoyima.build/Debug-iphoneos/xiaoyima.build/DerivedSources/arm64 -I/Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Intermediates.noindex/xiaoyima.build/Debug-iphoneos/xiaoyima.build/DerivedSources -F/Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Products/Debug-iphoneos -F/Users/chenyue/Documents/GitHub/xiaoyima/xiaoyima -MMD -MT dependencies -MF /Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Intermediates.noindex/xiaoyima.build/Debug-iphoneos/xiaoyima.build/Objects-normal/arm64/SceneDelegate.d --serialize-diagnostics /Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Intermediates.noindex/xiaoyima.build/Debug-iphoneos/xiaoyima.build/Objects-normal/arm64/SceneDelegate.dia -c /Users/chenyue/Documents/GitHub/xiaoyima/xiaoyima/SceneDelegate.m -o /Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Intermediates.noindex/xiaoyima.build/Debug-iphoneos/xiaoyima.build/Objects-normal/arm64/SceneDelegate.o
```

这个命令看起来很长，本质就是代码预处理、编译、汇编生成.o文件。objective-c的编译依赖于Clang+LLVM，采用clang作为前端，llvm作为后端。clang将代码处理后生成LLVM IR，llvm根据LLVM IR生成对应架构的机械码，我们可以用clang命令来分解一下。另外由于单独运行clang会有环境问题，我们用封装好了的xcrun方法调用clang
### 预处理（Prepressing）
处理源代码文件中的以"#"开头的预编译指令。规则如下：
"#define"删除并展开对应宏定义。
处理所有的条件预编译指令。如#if/#ifdef/#else/#endif。
"#include/#import"包含的文件递归插入到此处。
删除所有的注释"//或/**/"。
添加行号和文件名标识。如“# 1 "main.m"”,编译调试会用到。
xcrun -sdk iphoneos clang -E SceneDelegate.m -o SceneDelegate.i

### 编译（Compilation）
编译就是把上面得到的.i文件进行：词法分析、语法分析、静态分析、中间语言生成、生成汇编代码，得到.s文件。

词法分析：
这一步把源文件中的代码转化为特殊的标记流. 词法分析器读入源文件的字符流, 将他们组织称有意义的词素(lexeme)序列，对于每个词素，此法分析器产生词法单元（token）作为输出.
源码被分割成一个一个的字符和单词, 在行尾Loc中都标记出了源码所在的对应源文件和具体行数, 方便在报错时定位问题. 类似于下面:
```int 'int'     [StartOfLine]    Loc=<main.m:14:1>
identifier 'main'     [LeadingSpace]    Loc=<main.m:14:5>
l_paren '('        Loc=<main.m:14:9>
int 'int'        Loc=<main.m:14:10>
identifier 'argc'     [LeadingSpace]    Loc=<main.m:14:14>
comma ','        Loc=<main.m:14:18>
char 'char'     [LeadingSpace]    Loc=<main.m:14:20>
star '*'     [LeadingSpace]    Loc=<main.m:14:25>
```

语法分析：
clang -fmodules -E -Xclang -dump-tokens （源文件）
词法分析的Token流会被解析成一颗抽象语法树(abstract syntax tree - AST). 在这里面每一节点也都标记了其在源码中的位置.
有了抽象语法树，clang就可以对这个树进行分析，找出代码中的错误。比如类型不匹配，亦或Objective C中向target发送了一个未实现的消息.
AST是开发者编写clang插件主要交互的数据结构，clang也提供很多API去读取AST.

静态分析：
clang -fmodules -fsyntax-only -Xclang -ast-dump （源文件）
把源码转化为抽象语法树之后，编译器就可以对这个树进行分析处理。静态分析会对代码进行错误检查，如出现方法被调用但是未定义、定义但是未使用的变量等，以此提高代码质量. 也可以使用 Xcode 自带的静态分析工具（Product -> Analyze).

常见的操作有:

1）当在代码中使用 ARC 时，编译器在编译期间，会做许多的类型检查. 最常见的是检查程序是否发送正确的消息给正确的对象，是否在正确的值上调用了正常函数。如果你给一个单纯的 NSObject* 对象发送了一个 hello 消息，那么 clang 就会报错，同样，给属性设置一个与其自身类型不相符的对象，编译器会给出一个可能使用不正确的警告.
> 一般会把类型分为两类：动态的和静态的。动态的在运行时做检查，静态的在编译时做检查。以往，编写代码时可以向任意对象发送任何消息，在运行时，才会检查对象是否能够响应这些消息。由于只是在运行时做此类检查，所以叫做动态类型。
至于静态类型，是在编译时做检查。当在代码中使用 ARC 时，编译器在编译期间，会做许多的类型检查：因为编译器需要知道哪个对象该如何使用。
2）检查是否有定义了，但是从未使用过的变量.
3）检查在 你的初始化方法中中调用 self 之前, 是否已经调用 [self initWith…] 或 [super init] 了.

中间语言生成：
clang -S -emit-llvm （源文件）
此处遍历语法树，最终生成LLVM IR代码。LLVM IR是前端的输出，后端的输入. Objective C代码在这一步会进行runtime的桥接：property合成，ARC处理等，并且在编译期就可以确定的表达式进行优化，比如代码里t1=2+6，可以优化t1=8。（假如开启了bitcode，）

生成汇编指令：
clang -S （源文件）
LLVM对IR进行优化后，会对代码进行编译优化例如针对全局变量优化、循环优化、尾递归优化等, 然后会针对不同架构，比如arm64,armv7,x86_64,生成不同的目标代码，最后以汇编代码的格式输出.

xcrun -sdk iphoneos clang -S SceneDelegate.m -o SceneDelegate.s 

### 汇编（Assembly）
汇编就是把上面得到的.s文件里的汇编指令一一翻译成机器指令。得到.o文件。其中.o文件可以通过nm命令来查看。
xcrun -sdk iphoneos clang -c SceneDelegate.s -o SceneDelegate.o

## 链接（link）
```> `Ld /Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Products/Debug-iphoneos/xiaoyima.app/xiaoyima normal (in target 'xiaoyima' from project 'xiaoyima')
>     cd /Users/chenyue/Documents/GitHub/xiaoyima
>     /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang -target arm64-apple-ios13.0 -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS15.2.sdk -L/Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Products/Debug-iphoneos -F/Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Products/Debug-iphoneos -F/Users/chenyue/Documents/GitHub/xiaoyima/xiaoyima -filelist /Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Intermediates.noindex/xiaoyima.build/Debug-iphoneos/xiaoyima.build/Objects-normal/arm64/xiaoyima.LinkFileList -Xlinker -rpath -Xlinker @executable_path/Frameworks -dead_strip -Xlinker -object_path_lto -Xlinker /Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Intermediates.noindex/xiaoyima.build/Debug-iphoneos/xiaoyima.build/Objects-normal/arm64/xiaoyima_lto.o -Xlinker -export_dynamic -Xlinker -no_deduplicate -fembed-bitcode-marker -fobjc-arc -fobjc-link-runtime -framework GameKit -Xlinker -no_adhoc_codesign -Xlinker -dependency_info -Xlinker /Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Intermediates.noindex/xiaoyima.build/Debug-iphoneos/xiaoyima.build/Objects-normal/arm64/xiaoyima_dependency_info.dat -o /Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Products/Debug-iphoneos/xiaoyima.app/xiaoyima`
```
    
> 经常出现的clang: error: linker command failed with exit code 1 (use -v to see invocation)就出现在这个阶段。排查思路主要有，1.未倒入框架，2.检查header search path、library search path、framework search path，是否未包括报错文件，3、Other Linker Flags 改为 -lz或－ObjC，某些情况要改为-all_load 或者-force_load指定某个文件，因为OC语言并不是对每一个函数或者方法建立符号表,而只是对每一个类创建了符号表.如果一个类有了分类,那么链接器就不会将核心类与分类之间的代码完成进行合并,这就阻止了在最终的应用程序中的可执行文件缺失了分类中的代码,这样函数调用接失败了。这种情况需要添加-all_load 或者-force_load。4、库的架构是否与工程不一致。


这个命令根据-filelist xiaoyima.LinkFileList中的需要链接的文件，生成了app文件夹中的同名mach—O文件。
这里扩展一下，这个mach-O文件可以用machoview打开，查看里面的详细信息
![avatar](/image/MachOView.png)
刚刚命令中的@executable_path/Frameworks决定了LC_RPATH,也就是二进制文件链接自己引入的动态库的路径。而GameKit,Foundation等系统库则是读取设备的共享空间，/System/Library/Frameworks目录中，这里存放了系统的动态库供程序调用，当然如果调用了旧版本不存在的库，会导致闪退，比如AppTrackingTransparency.framework,这个库在旧版本ios中是没有的，需要在link binary with Libraries中将framework的status设为optional。


## CompileStoryboard
编译.m文件的同时，storyboard和xib也会被编译

/Applications/Xcode.app/Contents/Developer/usr/bin/ibtool --errors --warnings --notices --module xiaoyima --output-partial-info-plist /Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Intermediates.noindex/xiaoyima.build/Debug-iphoneos/xiaoyima.build/Base.lproj/Main-SBPartialInfo.plist --auto-activate-custom-fonts --target-device iphone --target-device ipad --minimum-deployment-target 13.0 --output-format human-readable-text --compilation-directory /Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Intermediates.noindex/xiaoyima.build/Debug-iphoneos/xiaoyima.build/Base.lproj /Users/chenyue/Documents/GitHub/xiaoyima/xiaoyima/Base.lproj/Main.storyboard

这个命令在/Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Intermediates.noindex/xiaoyima.build/Debug-iphoneos/xiaoyima.build/Base.lproj文件夹下生成对应的SBPartialInfo.plist和storyboardc。这里生成的SBPartialInfo.plist没看出有什么用，但会在之后的ProcessInfoPlistFile过程中对所有plist文件做处理。生成的storyboardc是一个目录，里面存放storyboard中各个uiviewcontroller对应的nib。

## 拷贝资源文件（CompileAssetCatalog）
这个步骤中我们复制所有需要的图片资源，并将其打包成Assets.car。这个压缩包可以通过cartool来解压获取里面的图片资源。

## 处理info.plist文件（ProcessInfoPlistFile）

## 执行pod配置的脚本
即cocoa的[CP] Embed Pods Frameworks

## 签名（codesign）
/usr/bin/codesign --force --sign 4CF31B99206CF710625EA728FA20E3558292DC63 --entitlements /Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Intermediates.noindex/xiaoyima.build/Debug-iphoneos/xiaoyima.build/xiaoyima.app.xcent --timestamp\=none --generate-entitlement-der /Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Products/Debug-iphoneos/xiaoyima.app

## 验证(Validate)
builtin-validationUtility /Users/chenyue/Library/Developer/Xcode/DerivedData/xiaoyima-eleojfqolneelbdshhqikinukoil/Build/Products/Debug-iphoneos/xiaoyima.app


参考：
[1.iOS编译指令](https://juejin.cn/post/6844903574166568973)
[2.Clang-LLVM下，一个源文件的编译过程](https://juejin.cn/post/7036170157357531144)