# 通过objc源码了解内存管理与runtime

通过objc源码，在main中调用了一个自己写的，继承了NSObject的类，了解一个类的调用过程中经过了什么。

## cls->instanceSize，callAlloc，obj->initInstanceIsa
程序从main()开始，调用类获取实例的方法`[self alloc]`，然后依次调用`_objc_rootAlloc(self)` ——》 `callAlloc(cls, false/*checkNil*/, true/*allocWithZone*/)` ，然后判断类有没有重写构造函数，如果有，通过objc_msgSend调用类的alloc函数，如果没有，依次调用 `_objc_rootAllocWithZone(cls, nil)` ——》`_class_createInstanceFromZone(cls, 0, nil,
                                         OBJECT_CONSTRUCT_CALL_BADALLOC)`。
                                         `_class_createInstanceFromZone`的源码为：

```
static ALWAYS_INLINE id
_class_createInstanceFromZone(Class cls, size_t extraBytes, void *zone,
                              int construct_flags = OBJECT_CONSTRUCT_NONE,
                              bool cxxConstruct = true,
                              size_t *outAllocatedSize = nil)
{
    ASSERT(cls->isRealized());

    // Read class's info bits all at once for performance
    bool hasCxxCtor = cxxConstruct && cls->hasCxxCtor();
    bool hasCxxDtor = cls->hasCxxDtor();
    bool fast = cls->canAllocNonpointer();
    size_t size;

    size = cls->instanceSize(extraBytes);
    if (outAllocatedSize) *outAllocatedSize = size;

    id obj;
    if (zone) {
        obj = (id)malloc_zone_calloc((malloc_zone_t *)zone, 1, size);
    } else {
        obj = (id)calloc(1, size);
    }
    if (slowpath(!obj)) {
        if (construct_flags & OBJECT_CONSTRUCT_CALL_BADALLOC) {
            return _objc_callBadAllocHandler(cls);
        }
        return nil;
    }

    if (!zone && fast) {
        obj->initInstanceIsa(cls, hasCxxDtor);
    } else {
        // Use raw pointer isa on the assumption that they might be
        // doing something weird with the zone or RR.
        obj->initIsa(cls);
    }

    if (fastpath(!hasCxxCtor)) {
        return obj;
    }

    construct_flags |= OBJECT_CONSTRUCT_FREE_ONFAILURE;
    return object_cxxConstructFromClass(obj, cls, construct_flags);
}
```

alloc过程中，runtime通过`size = cls->instanceSize(extraBytes)`方法，求出`cache.fastInstanceSize(extraBytes)`对于extrabytes字节对齐，求出关于16字节倍数的空间大小，并抹去余数（align16），计算对象需要申请的空间大小。
& 在某个ios版本中通过zone去开辟内存空间的方法已被废除，现在基本是通过`calloc`来开辟内存空间的。
调用过calloc后，就可以在lldb上通过po obj打印出类的地址了，在这之前打印都是nil。
最后`obj->initInstanceIsa(cls, hasCxxDtor)`，调用isa.setClass(cls, this)， 将对象obj的isa与类cls关联起来。

## objc_msgSend
OC的方法调用用clang转为c++后就是调用objc_msgSend，在objc源码中的objc-msg-arm64.s文件中找到了汇编的实现。不太理解汇编，从网上找了一个带注释的版本

### objc_msgSend
```
ENTRY _objc_msgSend     //_objc_msgSend入口
UNWIND _objc_msgSend, NoFrame   //调用了一个宏
	
	//p0与立即数0作比较
		cmp	p0, #0			//检查p0是否为空或者是TAGGED_POINTERS
#if SUPPORT_TAGGED_POINTERS
//b表示跳转，le表示带符号的小于或等于
	b.le	LNilOrTagged		//小于等于0,跳转到LNilOrTagged    
#else
	b.eq	LReturnZero       //等于0,跳转到LReturnZero
#endif
	ldr	p13, [x0]		// 读p13内存，p13 = isa   
	GetClassFromIsa_p16 p13		//GetClassFromIsa_p16是宏，作用是将获取到的 class 存在 p16 上面。
LGetIsaDone:
	// calls imp or objc_msgSend_uncached
	CacheLookup NORMAL, _objc_msgSend, __objc_msgSend_uncached
#if SUPPORT_TAGGED_POINTERS
LNilOrTagged:
	b.eq	LReturnZero		// nil check

	// tagged 得到有全局变量tagged地址的4KB页的基址,adrp就是将该页的基址存到寄存器X10中
	adrp	x10, _objc_debug_taggedpointer_classes@PAGE
    //x10是页的基址，x10 + tagged@PAGEOFF拿到tagged的地址，存到x10中
	add	x10, x10, _objc_debug_taggedpointer_classes@PAGEOFF
	ubfx	x11, x0, #60, #4  //从x0的第60位开始，取4位到x11中，剩余高位用0填充
	ldr	x16, [x10, x11, LSL #3] 
	adrp	x10, _OBJC_CLASS_$___NSUnrecognizedTaggedPointer@PAGE
	add	x10, x10, _OBJC_CLASS_$___NSUnrecognizedTaggedPointer@PAGEOFF
	cmp	x10, x16
	b.ne	LGetIsaDone

	// ext tagged
	adrp	x10, _objc_debug_taggedpointer_ext_classes@PAGE
	add	x10, x10, _objc_debug_taggedpointer_ext_classes@PAGEOFF
	ubfx	x11, x0, #52, #8
	ldr	x16, [x10, x11, LSL #3]
	b	LGetIsaDone
// SUPPORT_TAGGED_POINTERS
#endif

LReturnZero:
	// x0 is already zero
	mov	x1, #0
	movi	d0, #0
	movi	d1, #0
	movi	d2, #0
	movi	d3, #0
	ret

	END_ENTRY _objc_msgSend


```

Tagged Pointer指针的值不再是地址了，而是真正的值。所以，实际上它不再是一个对象了，它只是一个披着对象皮的普通变量而已，所有有特殊的处理逻辑。所以，它的内存并不存储在堆中，也不需要malloc和free。
在内存读取上有着3倍的效率，创建时比以前快106倍。

objc_msgSend先比较p0与0, 如果支持tagged pointer，p0 <= 0就跳转去LNilOrTagged，否则p0 = 0时就跳转去LReturnZero，LReturnZero对相关的寄存器清零；而在LnilOrTagged中当p0 = 0时也会调用LReturnZero，如果p0 < 0，就会进入GetTaggedClass获取Tagged Pointer的类，然后跳回LGetIsaDone；从x0中读出isa放到p13, 用p13 & mask拿到真正的类地址存入p16。
然后执行CacheLookup NORMAL, _objc_msgSend, __objc_msgSend_uncached
### CacheLookup
从代码注释中可以知道它的用法为：
CacheLookup NORMAL|GETIMP|LOOKUP 
CacheLookup 的参数分为三种，NORMAL（正常的去查找） 、 GETIMP（直接返回 IMP） 和 LOOKUP（主动的慢速去查找）
它的作用就是在类的方法缓存中通过sel去查找imp，x1中是sel, x16中是要查找的类，如果找到了就把imp放入x17中，把对应的类放入x16; 如果没找到，就进入JumpMiss。


```
.macro CacheLookup
    mov	x15, x16			// x16是类
LLookupStart\Function:
    	// p1 = SEL, p16 = isa  通过isa偏移16得到类的cache地址存入p11,p11为_maskAndBuckets
	//#CACHE 是一个宏定义 #define CACHE (2 * __SIZEOF_POINTER__)，代表16个字节
	#if CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16_BIG_ADDRS
	ldr	p10, [x16, #CACHE]	//ldr读内存命令，p10 = mask|buckets
	lsr	p11, p10, #48			// LSR 逻辑右移：低位移出，高位补零，去掉mask|buckets中的buckets，p11 = mask
	and	p10, p10, #0xffffffffffff	// 通过按位与操作，p10 = buckets
	and	w12, w1, w11			// x12 = _cmd & mask
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16
	ldr	p11, [x16, #CACHE]			// p11 = mask|buckets
#if CONFIG_USE_PREOPT_CACHES
#if __has_feature(ptrauth_calls) //判断编译器是否支持指针身份验证功能，指针身份验证，针对arm64e架构；使用Apple A12或更高版本A系列处理器的设备（如iPhone XS、iPhone XS Max和iPhone XR或更新的设备）支持arm64e架构
    //tbnz 测试位不为0则跳转。与tbz对应。 p11 第0位不为0则跳转 LLookupPreopt\Function。
	tbnz	p11, #0, LLookupPreopt\Function
	and	p10, p11, #0x0000ffffffffffff	//p10 = _bucketsAndMaybeMask & 0x0000ffffffffffff = buckets
#else
	and	p10, p11, #0x0000fffffffffffe	//p10 = _bucketsAndMaybeMask & 0x0000fffffffffffe = buckets
	tbnz	p11, #0, LLookupPreopt\Function
	//p11 第0位不为0则跳转 LLookupPreopt\Function。
#endif
    //eor 逻辑异或(^) 格式为：EOR{S}{cond} Rd, Rn, Operand2
    //p12 = selector ^ (selector >> 7) select 右移7位&自己给到p12
	eor	p12, p1, p1, LSR #7
	and	p12, p12, p11, LSR #48		
#else
	and	p10, p11, #0x0000ffffffffffff	//p10 = _bucketsAndMaybeMask & 0x0000ffffffffffff = buckets
	and	p12, p1, p11, LSR #48		
	//p12 = selector & (_bucketsAndMaybeMask >>48) = sel & mask = buckets中的下标
#endif // CONFIG_USE_PREOPT_CACHES
//arm64 32
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_LOW_4
    //后4位为mask前置0的个数的case
    ldr p11, [x16, #CACHE]              // p11 = mask|buckets
    and p10, p11, #~0xf         // p10 = buckets 相当于后4位置为0，取前32位
    and p11, p11, #0xf          // p11 = maskShift 取的是后4位，为mask前置位的0的个数
    mov p12, #0xffff
    lsr p11, p12, p11           // p11 = mask = 0xffff >> p11
    and p12, p1, p11            // x12 = _cmd & mask
#else
#error Unsupported cache mask storage for ARM64.
#endif
	//通过上面的计算 p10 = buckets，p11 = mask（arm64真机是_bucketsAndMaybeMask）， p12 = index
    // p13（bucket_t） = buckets + 下标 << 4   PTRSHIFT arm64 为3.  <<4 位为16字节 buckets + 下标 *16 = buckets + index *16 也就是直接平移到了第几个元素的地址。
    add p13, p10, p12, LSL #(1+PTRSHIFT)
    // p13 = buckets + ((_cmd & mask) << (1+PTRSHIFT))
    //这里就直接遍历查找了，因为arm64下cache_next相当于遍历（这里只扫描了前面）                 
    //p17 = imp, p9 = sel ，p1=cmd
1:  ldp p17, p9, [x13], #-BUCKET_SIZE，{imp, sel} = *bucket--   
    //sel - _cmd != 0 则跳转到3:，也就意味着没有找到就跳转到__objc_msgSend_uncached
    cmp p9, p1              
    b.ne    3f                                    
    //找到则调用或者返回imp，Mode为 NORMAL
2:  CacheHit \Mode              
//缓存中找不到方法就走__objc_msgSend_uncached逻辑了。
//cbz 为0跳转 sel == nil 跳转 \MissLabelDynamic
3:  cbz p9, \MissLabelDynamic        //有空位没有找到说明没有缓存
    //bucket_t - buckets 由于是递减操作
    cmp p13, p10             //⚠️ 这里一直是往前找，后面的元素在后面还有一次循环。
    //无符号大于等于 则跳转到1:f b 分别代表front与back
    b.hs    1b
//根据注释，上方汇编代码逻辑如下
// do {
//     {imp, sel} = *bucket--
//     if (sel != _cmd) {
//         scan more
//     } else {
//          hit:    call or return imp  命中
//     }
//     if (sel == 0) goto Miss;
// } while (bucket >= buckets)
//没有命中cache  查找 p13 = mask对应的元素，也就是倒数第二个
#if CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16_BIG_ADDRS
    //p13 = buckets + (mask << 4) 平移找到对应mask的bucket_t。UXTW 将w11扩展为64位后左移4
    add p13, p10, w11, UXTW #(1+PTRSHIFT)
                        // p13 = buckets + (mask << 1+PTRSHIFT)
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16
    //p13 = buckets + (mask >> 44) 这里右移44位，少移动4位就不用再左移了。因为maskZeroBits的存在 就找到了mask对应元素的地址
    add p13, p10, p11, LSR #(48 - (1+PTRSHIFT))
                        // p13 = buckets + (mask << 1+PTRSHIFT)
                        // see comment about maskZeroBits
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_LOW_4
    //p13 = buckets + (mask << 4) 找到对应mask的bucket_t。
    add p13, p10, p11, LSL #(1+PTRSHIFT)
                        // p13 = buckets + (mask << 1+PTRSHIFT)
#else
#error Unsupported cache mask storage for ARM64.
#endif
    //p12 = buckets + (p12<<4) index对应的bucket_t
    add p12, p10, p12, LSL #(1+PTRSHIFT) // p12 = first probed bucket
//之前已经往前查找过了，这里从后往index查找
                        
    //p17 = imp p9 = sel
4:  ldp p17, p9, [x13], #-BUCKET_SIZE       //sel - _cmd，ldp读取指令
    cmp p9, p1              
    //sel == _cmd跳转CacheHit
    b.eq    2b              
    //sel ！= nil
    cmp p9, #0              
    //
    ccmp    p13, p12, #0, ne    //ccmp:双重比较，判断x13 和 x12 同时存在
    //有值跳转4:
    b.hi    4b
//根据注释，上方汇编代码逻辑相当于下方代码
// do {
//     {imp, sel} = *bucket--
//     if (sel == _cmd)
//         goto hit
// } while (sel != 0 && bucket > first_probed)
LLookupEnd\Function:
LLookupRecover\Function:
//仍然没有找到缓存，缓存彻底不存在 __objc_msgSend_uncached()
    b   \MissLabelDynamic
```

其中CACHE_MASK_STORAGE在objc-config中有定义如下
```
#if defined(__arm64__) && __LP64__
#if TARGET_OS_OSX || TARGET_OS_SIMULATOR
#define CACHE_MASK_STORAGE CACHE_MASK_STORAGE_HIGH_16_BIG_ADDRS
#else
#define CACHE_MASK_STORAGE CACHE_MASK_STORAGE_HIGH_16
#endif
```
意味着在arm64架构的64位OS X devices设备中，CACHE_MASK_STORAGE为CACHE_MASK_STORAGE_HIGH_16_BIG_ADDRS。
CONFIG_USE_PREOPT_CACHES在objc-internal中定义如下，查了一下应该是跟是否有共享缓存，ios真机中CONFIG_USE_PREOPT_CACHES为1
```
#if defined(__arm64__) && TARGET_OS_IOS && !TARGET_OS_SIMULATOR && !TARGET_OS_MACCATALYST
#define CONFIG_USE_PREOPT_CACHES 1
#else
#define CONFIG_USE_PREOPT_CACHES 0
#endif
```
上面的代码主要流程是通过class-->catch-->buckets-->index（通过哈希操作得到）-->bucket -->imp
1.判断receiver是否存在。
2.通过receiver对象中的isa&mask获取当前的class信息。
3.class通过内存平移0x10获取对应的成员变量 cache。
4.通过cache中的_bucketsAndMaybeMask成员，获取到当前buckets的首地址。
5.通过哈希 sel & mask获取第一次查找的index。
6.buckets + index << 4（ index << 4 即index乘以16得到当前的index前面的内存的大小）然后buckets移动对应大小内存，到达对应的index 对应的buckets。
7.do-while循环找缓存，这里将从index向前查找 index-->0,然后再通过mask-->index查找流程，当命中内存返回对应的imp，转码进行调用。当找不到，或sel为空，即走__objc_msgSend_uncached进行慢速查找流程。

### __objc_msgSend_uncached
   ```
   STATIC_ENTRY __objc_msgSend_uncached
   UNWIND __objc_msgSend_uncached, FrameWithNoSaves
   
   MethodTableLookup
   TailCallFunctionPointer x17
   
   END_ENTRY __objc_msgSend_uncached
```
   
MethodTableLookup 后面是比较复杂的逻辑，下面会分析，TailCallFunctionPointer x17 若找到了 IMP 会放到 x17 寄存器中，然后把 x17 的值传递给 TailCallFunctionPointer 宏调用方法。

### MethodTableLookup
```
.macro MethodTableLookup
	
	SAVE_REGS MSGSEND //创建堆栈帧并保存所有参数寄存器，为函数调用做准备。
	mov	x2, x16
	mov	x3, #3
	bl	_lookUpImpOrForward
    // 跳转到c函数lookUpImpOrForward(obj, sel, cls, LOOKUP_INITIALIZE | LOOKUP_RESOLVER)，x0是_lookUpImpOrForward函数的返回值imp
	// IMP in x0
	mov	x17, x0

	RESTORE_REGS MSGSEND //恢复所有参数寄存器并弹出由SAVE_REGS创建的堆栈帧。

.endmacro
```

### lookUpImpOrForward
```
NEVER_INLINE
IMP lookUpImpOrForward(id inst, SEL sel, Class cls, int behavior)
{
    const IMP forward_imp = (IMP)_objc_msgForward_impcache;
    IMP imp = nil;
    Class curClass;

    runtimeLock.assertUnlocked();

    if (slowpath(!cls->isInitialized())) {
        behavior |= LOOKUP_NOCACHE;
    }

     //    加锁，目的是保证读取的线程安全
     runtimeLock.lock();

    //避免加载一个看起来像类但实际上不是类的二进制blob，导致被CFI攻击。（按注释）
    //是否是已知类：判断当前类是否是已经被认可的类，即已经加载的类           
    checkIsKnownClass(cls);
    // 判断类是否实现，如果没有，需要先实现
    cls = realizeAndInitializeIfNeeded_locked(inst, cls, behavior & LOOKUP_INITIALIZE);
    // runtimeLock可能已被删除，但现在已再次锁定
    runtimeLock.assertLocked();
    curClass = cls;

    //    递归，之后一个参数的for循环是死循环，主要靠goto和break脱离循环
    for (unsigned attempts = unreasonableClassCount();;) {
        if (curClass->cache.isConstantOptimizedCache(/* strict */true)) {
#if CONFIG_USE_PREOPT_CACHES
            imp = cache_getImp(curClass, sel);
            if (imp) goto done_unlock;
            //如果不存在，
            curClass = curClass->cache.preoptFallbackClass();
#endif
        } else {
            //method_t中可能是存储名称、类型和实现的三个相对偏移量的小imp，也有可能是存储选择器、类型和实现的三个指针的大imp
            method_t *meth = getMethodNoSuper_nolock(curClass, sel);
            if (meth) {
            //如果imp存在，退出循环，跳转到done
                imp = meth->imp(false);
                goto done;
            }
            //一层一层遍历curClass的父类
            if (slowpath((curClass = curClass->getSuperclass()) == nil)) {
                //父类遍历也找不到实现，调用转发
                imp = forward_imp;
                break;
            }
        }

        //unreasonableClassCount()有提供一个有上限的数，减到零后报错退出
        if (slowpath(--attempts == 0)) {
            _objc_fatal("Memory corruption in class list.");
        }

        //调用汇编获取缓存中父类的imp，也是走的CacheLookup
        imp = cache_getImp(curClass, sel);
        if (slowpath(imp == forward_imp)) {
            //在超类中找到forward。
            //停止搜索，但不要缓存；首先为此类调用方法解析器。
            break;
        }
        if (fastpath(imp)) {
            //在父类中找到了imp，跳出循环
            goto done;
        }
    }

    //未找到任何实现。尝试一下method resolver。
    if (slowpath(behavior & LOOKUP_RESOLVER)) {
        //C语言中的 ^= 意思为：按位异或后赋值
        behavior ^= LOOKUP_RESOLVER;
        //动态方法决议流程，可以理解为再给一次机会进行补救
        return resolveMethod_locked(inst, sel, cls, behavior);
    }

 done:
    if (fastpath((behavior & LOOKUP_NOCACHE) == 0)) {
#if CONFIG_USE_PREOPT_CACHES
        while (cls->cache.isConstantOptimizedCache(/* strict */true)) {
            cls = cls->cache.preoptFallbackClass();
        }
#endif
        //写入缓存中，提高下次查找速度
        log_and_fill_cache(cls, imp, sel, inst, curClass);
    }
 done_unlock:
    runtimeLock.unlock();
    if (slowpath((behavior & LOOKUP_NIL) && imp == forward_imp)) {
        return nil;
    }
    return imp;
}
```

### resolveMethod_locked
在本类和父类等的方法列表中都没有找到该方法，则会进入到动态方法决议
```
resolveMethod_locked
static NEVER_INLINE IMP
resolveMethod_locked(id inst, SEL sel, Class cls, int behavior)
{
    //加锁，线程安全
    runtimeLock.assertLocked();
    ASSERT(cls->isRealized());

    runtimeLock.unlock();
    // 判断当前是否是元类
    if (! cls->isMetaClass()) {
        // try [cls resolveInstanceMethod:sel]
        //进入实例方法决议
        resolveInstanceMethod(inst, sel, cls);
    } 
    else {
        // try [nonMetaClass resolveClassMethod:sel]
        // and [cls resolveInstanceMethod:sel]
        //进入类方法决议
        resolveClassMethod(inst, sel, cls);
        //如果没有在类方法中找到解决的方法决议
        if (!lookUpImpOrNilTryCache(inst, sel, cls)) {
            //再次去调用实例方法决议
            resolveInstanceMethod(inst, sel, cls);
        }
    }

    // chances are that calling the resolver have populated the cache
    // so attempt using it
    return lookUpImpOrForwardTryCache(inst, sel, cls, behavior);
}
```
在自己写的程序中，程序进入了resolveInstanceMethod方法

### resolveInstanceMethod

```
static void resolveInstanceMethod(id inst, SEL sel, Class cls)
{
    runtimeLock.assertUnlocked();
    ASSERT(cls->isRealized());
    SEL resolve_sel = @selector(resolveInstanceMethod:);//定义解决方法

    if (!lookUpImpOrNilTryCache(cls, resolve_sel, cls->ISA(/*authenticated*/true))) {
        // Resolver not implemented.去查找一下有没有解决问题这个方法，没有的的话，就返回，但是这个方法基本上不会进来的，系统在NSObject中有默认实现这个方法
        return;
    }

    BOOL (*msg)(Class, SEL, SEL) = (typeof(msg))objc_msgSend;
    //对着这个方法发送消息看下有没有解决问题
   /* NSObject中resolveInstanceMethod的方法实现如下
   + (BOOL)resolveInstanceMethod:(SEL)sel {
    return NO;
    }
    */
    bool resolved = msg(cls, resolve_sel, sel);// 如果你重写了，返回值return Yes，系统默认是NO，你没处理 

    // Cache the result (good or bad) so the resolver doesn't fire next time.
    // +resolveInstanceMethod adds to self a.k.a. cls
    IMP imp = lookUpImpOrNilTryCache(inst, sel, cls);//再次慢速查找这个解决方法的imp，重新指向的方法。

    if (resolved  &&  PrintResolving) {
        if (imp) {//操作的信息写入
            _objc_inform("RESOLVE: method %c[%s %s] "
                         "dynamically resolved to %p", 
                         cls->isMetaClass() ? '+' : '-', 
                         cls->nameForLogging(), sel_getName(sel), imp);
                         //有imp
        }
        else {
            // Method resolver didn't add anything?
            _objc_inform("RESOLVE: +[%s resolveInstanceMethod:%s] returned YES"
                         ", but no new implementation of %c[%s %s] was found",
                         cls->nameForLogging(), sel_getName(sel), 
                         cls->isMetaClass() ? '+' : '-', 
                         cls->nameForLogging(), sel_getName(sel));//没有imp
        }
    }
}
```
在调用实例方法的时候没有实现，进入实例方法的动态决议。我们是否实现了resolveInstanceMethod，但是通常NSObject会实现这个方法返回NO，没有处理。
对resolveInstanceMethod发送消息，再次慢速查找这个解决方法的imp但是不会进行消息转发，重新指向的方法。imp 查找过程`lookUpImpOrNilTryCache-> _lookUpImpTryCache->lookUpImpOrForward`
添加了解决方法的话 `lookUpImpOrForwardTryCache->_lookUpImpTryCache`再次慢速查找Imp找到进行消息发送。

## forwardingTargetForSelector
但如果resolveInstanceMethod和resolveClassMethod还是没有获取到imp，objc_sendMsg会返回空对象，这时候源代码就没有提到之后的调用了，按我们对消息发送和转发机制的了解，之后应该进入消息转发中有两次补救机会：快速转发+慢速转发。之后我们只能看到堆栈中，CoreFoundation的___forwarding___又调用了快速转发`-[NSObject(NSObject)forwardingTargetForSelector:]`方法，如果还是返回nil，CoreFoundation的_CF_forwarding_prep_0调用了慢速转发`-[NSObject(NSObject) methodSignatureForSelector:]`方法创建NSInvocation需要的NSMethodSignature，这两个方法我们都没有继承，那都返回nil,就会依次调用`-[NSObject(NSObject) forwardInvocation:]`，`-[NSObject(NSObject) doesNotRecognizeSelector:]`，`_objc_fatal("+[%s %s]: unrecognized selector sent to instance %p", 
                class_getName(self), sel_getName(sel), self)`方法弹出了我们熟悉的unrecognized selector sent to instance报错。

## objc_storeStrong
堆栈过程中我们还看到了objc_storeStrong
objc_storeStrong，其源码为：

```
void
objc_storeStrong(id *location, id obj)
{
    id prev = *location;
    if (obj == prev) {
        return;
    }
    objc_retain(obj);
    *location = obj;
    objc_release(prev);
}
```
oc中用strong修饰一个对象，实际上是调用了objc_storeStrong。location是引用对象的指针的地址，obj是对象本身。首先和之前的引用相比判断是不是同一个引用，是的话就return；否则的话就对obj对象进行retain，并且释放*location之前的引用（也就是说*location指针不再指向之前的对象，要把之前对象引用计数减1）。objc_release的详细放到之后说。

```
__attribute__((aligned(16), flatten, noinline))
id 
objc_retain(id obj)
{
    if (_objc_isTaggedPointerOrNil(obj)) return obj;
    return obj->retain();
}
```
代码通过_objc_isTaggedPointerOrNil将obj转化为可以与nil对比的格式，判断obj是否为空，如果不为空，则结构体obj调用retain方法，而retain又调用roottetain。objc_retain函数主要是对对象引用计数加1，下面来看roottetain函数的实现。

```
ALWAYS_INLINE id
objc_object::rootRetain(bool tryRetain, objc_object::RRVariant variant)
{
    if (slowpath(isTaggedPointer())) return (id)this;

    bool sideTableLocked = false;
    bool transcribeToSideTable = false;

    isa_t oldisa;
    isa_t newisa;

    oldisa = LoadExclusive(&isa.bits);

    if (variant == RRVariant::FastOrMsgSend) {
        //提高效率，避免重复reload isa
        if (slowpath(oldisa.getDecodedClass(false)->hasCustomRR())) {
            ClearExclusive(&isa.bits);
            if (oldisa.getDecodedClass(false)->canCallSwiftRR()) {
                return swiftRetain.load(memory_order_relaxed)((id)this);
            }
            return ((id(*)(objc_object *, SEL))objc_msgSend)(this, @selector(retain));
        }
    }

    if (slowpath(!oldisa.nonpointer)) {
        // a Class is a Class forever, so we can perform this check once
        // outside of the CAS loop
        if (oldisa.getDecodedClass(false)->isMetaClass()) {
            ClearExclusive(&isa.bits);
            return (id)this;
        }
    }

    do {
        transcribeToSideTable = false;
        newisa = oldisa;
        //如果newisa不是nonpointer，直接操作散列表+1
        if (slowpath(!newisa.nonpointer)) {
            ClearExclusive(&isa.bits);
            if (tryRetain) 
                return sidetable_tryRetain() ? (id)this : nil;
            else 
                //在这里存储引用计数
                return sidetable_retain(sideTableLocked);
        }
        // don't check newisa.fast_rr; we already called any RR overrides
        if (slowpath(newisa.isDeallocating())) {
            ClearExclusive(&isa.bits);
            if (sideTableLocked) {
                ASSERT(variant == RRVariant::Full);
                sidetable_unlock();
            }
            if (slowpath(tryRetain)) {
                return nil;
            } else {
                return (id)this;
            }
        }
        uintptr_t carry;
        newisa.bits = addc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc++

        if (slowpath(carry)) {
            // newisa.extra_rc++ overflowed
            if (variant != RRVariant::Full) {
                ClearExclusive(&isa.bits);
                return rootRetain_overflow(tryRetain);
            }
            // Leave half of the retain counts inline and 
            // prepare to copy the other half to the side table.
            if (!tryRetain && !sideTableLocked) sidetable_lock();
            sideTableLocked = true;
            transcribeToSideTable = true;
            newisa.extra_rc = RC_HALF;
            newisa.has_sidetable_rc = true;
        }
    } while (slowpath(!StoreExclusive(&isa.bits, &oldisa.bits, newisa.bits)));

    if (variant == RRVariant::Full) {
        if (slowpath(transcribeToSideTable)) {
            // Copy the other half of the retain counts to the side table.
            sidetable_addExtraRC_nolock(RC_HALF);
        }

        if (slowpath(!tryRetain && sideTableLocked)) sidetable_unlock();
    } else {
        ASSERT(!transcribeToSideTable);
        ASSERT(!sideTableLocked);
    }

    return (id)this;
}
```

### Tagged Pointer
isTaggedPointer判断一个对象类型是否为Tagged Pointer类型实际上是讲对象的地址与_OBJC_TAG_MASK进行按位与操作,结果在跟_OBJC_TAG_MASK进行对比。
为了节省内存和提高运行效率，对于64位程序，苹果提出了 Tagged Pointer 的概念。对于 NSNumber 和 NSDate 等小对象，它们的值可以直接存储在指针中。所以 这类对象只是普通的变量，只不过是苹果框架对它们进行了特殊的处理。所以这里判断是 Tagged Pointer 的话，直接返回。

### 编译器优化
这里的slowpath与fastpath对应
#define fastpath(x) (__builtin_expect(bool(x), 1))
#define slowpath(x) (__builtin_expect(bool(x), 0))
这个指令是gcc引入的，作用是允许程序员将最有可能执行的分支告诉编译器。这个指令的写法为：__builtin_expect(EXP, N)。
意思是：EXP==N的概率很大。通过这种方式，编译器在编译过程中，会将可能性更大的代码紧跟着前面的代码，从而减少指令跳转带来的性能上的下降。
target ->BuildSettings: 搜索:optimization，默认情况下，Debug环境为None，Release环境下为Fastest，Smallest[-OS]，在开启了optimization的情况下，fastpath和slowpath调用的代码在汇编页面展示的代码会精简很多，完成编译器优化。

## sidetable_retain
引用计数存储在散列表SideTables()[this].refcnts[this]中
```
id objc_object::sidetable_retain(bool locked)
{
#if SUPPORT_NONPOINTER_ISA
    ASSERT(!isa.nonpointer);
#endif
    SideTable& table = SideTables()[this];
    
    if (!locked) table.lock();
    size_t& refcntStorage = table.refcnts[this];
    if (! (refcntStorage & SIDE_TABLE_RC_PINNED)) {
        refcntStorage += SIDE_TABLE_RC_ONE;
    }
    table.unlock();

    return (id)this;
}
```
函数取出当前对象的引用计数信息refcntStorage，并且做了非负判断，判断通过后refcntStorage添加SIDE_TABLE_RC_ONE
# objc_release
objc_release函数对对象的引用计数进行减一，里面调用了`objc_object::rootRelease `——》 `uintptr_t 
objc_object::sidetable_release(bool performDealloc)`函数。rootRelease实现与rootRetain相似，再来看sidetable_release函数的实现
```
uintptr_t
objc_object::sidetable_release(bool locked, bool performDealloc)
{
#if SUPPORT_NONPOINTER_ISA
    ASSERT(!isa.nonpointer);
#endif
    SideTable& table = SideTables()[this];
// 根据对象地址获取到引用计数器所在的sideTable
    bool do_dealloc = false;
// 是否需要执行dealloc方法 默认是false
    if (!locked) table.lock();
    auto it = table.refcnts.try_emplace(this, SIDE_TABLE_DEALLOCATING);
    auto &refcnt = it.first->second;
    if (it.second) {
        do_dealloc = true;
    } else if (refcnt < SIDE_TABLE_DEALLOCATING) {
        // 如果引用计数的值小于 SIDE_TABLE_DEALLOCATING = 2(0010)
        // refcnt 低两位分别是SIDE_TABLE_WEAKLY_REFERENCED 0  SIDE_TABLE_DEALLOCATING 1
        // 这个对象需要被销毁
        do_dealloc = true;
        refcnt |= SIDE_TABLE_DEALLOCATING;
    } else if (! (refcnt & SIDE_TABLE_RC_PINNED)) {
    // 如果引用计数有值且未溢出那么-1
        refcnt -= SIDE_TABLE_RC_ONE;
    }
    table.unlock();
    // 如果需要执行dealloc 那么就调用这个对象的dealloc
    if (do_dealloc  &&  performDealloc) {
        ((void(*)(objc_object *, SEL))objc_msgSend)(this, @selector(dealloc));
    }
    return do_dealloc;
}
```

首先判断该SideTable是否包含该对象（迭代器it==refcnts.end()就证明map里面没有这个对象），如果没有，那么把这个对象标记为正在被dealloc；否则的话判断该对象是否没有引用，如果是的话，那么也把这个对象标记为正在被dealloc；如果两者都不是，那么就取出引用计数信息减1（通过减SIDE_TABLE_RC_ONE来实现第三位开始减1）。接下来如果发现这个对象需要被dealloc，那么直接执行对象的dealloc函数。 

# objc_autoreleasePoolPush
# objc_autoreleasePoolPop