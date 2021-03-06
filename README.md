# iOS-Monitor-Platform

这篇文章是我在开发 iOS 性能检测平台 SDK 搜集资料的总结和整理。主要会探讨下在 iOS 平台下如何采集性能指标，如 CPU 占用率、内存使用情况、FPS、冷启动、热启动时间等，剖析每一项指标的具体实现方式。

## CPU

获取当前应用的 CPU 占有率，注意方法最后要调用 vm_deallocate，防止出现内存泄漏，该方法采集的 CPU 数据和腾讯的 [GT](https://github.com/Tencent/GT)、Instruments 数据接近。

``` c
#import <mach/mach.h>
#import <assert.h>

float cpu_usage()
{
    kern_return_t kr;
    task_info_data_t tinfo;
    mach_msg_type_number_t task_info_count;

    task_info_count = TASK_INFO_MAX;
    kr = task_info(mach_task_self(), TASK_BASIC_INFO, (task_info_t)tinfo, &task_info_count);
    if (kr != KERN_SUCCESS) {
        return -1;
    }

    task_basic_info_t      basic_info;
    thread_array_t         thread_list;
    mach_msg_type_number_t thread_count;

    thread_info_data_t     thinfo;
    mach_msg_type_number_t thread_info_count;

    thread_basic_info_t basic_info_th;
    uint32_t stat_thread = 0; // Mach threads

    basic_info = (task_basic_info_t)tinfo;

    // get threads in the task
    kr = task_threads(mach_task_self(), &thread_list, &thread_count);
    if (kr != KERN_SUCCESS) {
        return -1;
    }
    if (thread_count > 0)
        stat_thread += thread_count;

    long tot_sec = 0;
    long tot_usec = 0;
    float tot_cpu = 0;
    int j;

    for (j = 0; j < (int)thread_count; j++)
    {
        thread_info_count = THREAD_INFO_MAX;
        kr = thread_info(thread_list[j], THREAD_BASIC_INFO,
                         (thread_info_t)thinfo, &thread_info_count);
        if (kr != KERN_SUCCESS) {
            return -1;
        }

        basic_info_th = (thread_basic_info_t)thinfo;

        if (!(basic_info_th->flags & TH_FLAGS_IDLE)) {
            tot_sec = tot_sec + basic_info_th->user_time.seconds + basic_info_th->system_time.seconds;
            tot_usec = tot_usec + basic_info_th->user_time.microseconds + basic_info_th->system_time.microseconds;
            tot_cpu = tot_cpu + basic_info_th->cpu_usage / (float)TH_USAGE_SCALE * 100.0;
        }

    } // for each thread

    kr = vm_deallocate(mach_task_self(), (vm_offset_t)thread_list, thread_count * sizeof(thread_t));
    assert(kr == KERN_SUCCESS);

    return tot_cpu;
}

```
[GT](https://github.com/Tencent/GT) 中获得 App 的 CPU 占有率的方法

``` objective-c
- (float)getCpuUsage
{
    kern_return_t           kr;
    thread_array_t          thread_list;
    mach_msg_type_number_t  thread_count;
    thread_info_data_t      thinfo;
    mach_msg_type_number_t  thread_info_count;
    thread_basic_info_t     basic_info_th;
    
    kr = task_threads(mach_task_self(), &thread_list, &thread_count);
    if (kr != KERN_SUCCESS) {
        return -1;
    }
    cpu_usage = 0;
    
    for (int i = 0; i < thread_count; i++)
    {
        thread_info_count = THREAD_INFO_MAX;
        kr = thread_info(thread_list[i], THREAD_BASIC_INFO,(thread_info_t)thinfo, &thread_info_count);
        if (kr != KERN_SUCCESS) {
            return -1;
        }
        
        basic_info_th = (thread_basic_info_t)thinfo;

        if (!(basic_info_th->flags & TH_FLAGS_IDLE))
        {
            cpu_usage += basic_info_th->cpu_usage;
        }
    }
    
    cpu_usage = cpu_usage / (float)TH_USAGE_SCALE * 100.0;
    
    vm_deallocate(mach_task_self(), (vm_offset_t)thread_list, thread_count * sizeof(thread_t));
    
    return cpu_usage;
}

```

## Memory

获取当前 App memory 使用情况

``` objective-c
- (NSUInteger)getResidentMemory
{
    struct task_basic_info t_info;
	mach_msg_type_number_t t_info_count = TASK_BASIC_INFO_COUNT;
	
	int r = task_info(mach_task_self(), TASK_BASIC_INFO, (task_info_t)&t_info, &t_info_count);
	if (r == KERN_SUCCESS)
	{
		return t_info.resident_size;
	}
	else
	{
		return -1;
	}
}

```

获取当前设备的 Memory 使用情况

``` c
int64_t getUsedMemory()
{
    size_t length = 0;
    int mib[6] = {0};
    
    int pagesize = 0;
    mib[0] = CTL_HW;
    mib[1] = HW_PAGESIZE;
    length = sizeof(pagesize);
    if (sysctl(mib, 2, &pagesize, &length, NULL, 0) < 0)
    {
        return 0;
    }
    
    mach_msg_type_number_t count = HOST_VM_INFO_COUNT;
    
    vm_statistics_data_t vmstat;
    
    if (host_statistics(mach_host_self(), HOST_VM_INFO, (host_info_t)&vmstat, &count) != KERN_SUCCESS)
    {
		return 0;
    }
    
    int wireMem = vmstat.wire_count * pagesize;
	int activeMem = vmstat.active_count * pagesize;
    return wireMem + activeMem;
}

```

## Startup Time

毫无疑问移动应用的启动时间是影响用户体验的一个重要方面，那么我们究竟该如何通过启动时间来衡量一个应用性能的好坏呢？启动时间可以从冷启动和热启动两个角度去测量

* 冷启动：指的是应用尚未运行，必须加载并构建整个应用，完成初始化的工作，冷启动往往比热启动耗时长，而且每个应用的冷启动耗时差别也很大，所以冷启动也存在了很大的优化空间，冷启动时间从`applicationDidFinishLaunching:withOptions:`方法开始计算，很多应用会在该方法对其使用的第三方库初始化。
* 热启动：应用已经在后台运行(常见的场景是用户按了 Home 按钮)，由于某个事件将应用唤醒到前台，应用会在`applicationWillEnterForeground:`方法接收应用进入前台的事件

先来研究下冷启动，因为在它里面存在很多资源密集型的操作，下面先看看苹果官方文档给的应用的启动时序图

<img src="Images/app_launch_fg.png" style="display: block; margin: 0 auto;" width="500">

t(App 总启动时间) = t1(main() 之前的加载时间) + t2(main() 之后的加载时间)。

t1 = 系统的 dylib (动态链接库)和 App 可执行文件的加载时间

t2 = main 函数执行之后到 AppDelegate 类中的`applicationDidFinishLaunching:withOptions:`方法执行结束前这段时间

先来看看如何通过打点的方式统计 main 函数之后的时间，下面代码是有些文章给出的一种实现

``` objective-c
CFAbsoluteTime StartTime;

int main(int argc, char * argv[]) {
    @autoreleasepool {
        StartTime = CFAbsoluteTimeGetCurrent();
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}

extern CFAbsoluteTime StartTime;
 ...
 
// 在 applicationDidFinishLaunching:withOptions: 方法的最后统计
dispatch_async(dispatch_get_main_queue(), ^{
    NSLog(@"Launched in %f sec", CFAbsoluteTimeGetCurrent() - StartTime);
});


```
> 上述代码使用`CFAbsoluteTimeGetCurrent()`方法来计算时间，`CFAbsoluteTimeGetCurrent()`的概念和`NSDate`非常相似，只不过参考点是：以GMT为标准的，2001年一月一日00：00：00这一刻的时间绝对值。`CFAbsoluteTimeGetCurrent()`也会跟着当前设备的系统时间一起变化，也可能会被用户修改。他的精确度可能是微秒（μs）

其实还可以通过`mach_absolute_time()`来计算时间，这个一般很少用，他表示 CPU 的时钟周期数（ticks），精确度可以达到纳秒（ns），`mach_absolute_time()`不受系统时间影响，只受设备重启和休眠行为影响。示例代码如下

``` objective-c
static uint64_t loadTime;
static uint64_t applicationRespondedTime = -1;
static mach_timebase_info_data_t timebaseInfo;

static inline NSTimeInterval MachTimeToSeconds(uint64_t machTime) {
    return ((machTime / 1e9) * timebaseInfo.numer) / timebaseInfo.denom;
}

@implementation XXStartupMeasurer

+ (void)load {
    loadTime = mach_absolute_time();
    mach_timebase_info(&timebaseInfo);
    
    @autoreleasepool {
        __block id<NSObject> obs;
        obs = [[NSNotificationCenter defaultCenter] addObserverForName:UIApplicationDidFinishLaunchingNotification
                                                                object:nil queue:nil
                                                            usingBlock:^(NSNotification *note) {
            dispatch_async(dispatch_get_main_queue(), ^{
                applicationRespondedTime = mach_absolute_time();
                NSLog(@"StartupMeasurer: it took %f seconds until the app could respond to user interaction.", MachTimeToSeconds(applicationRespondedTime - loadTime));
            });
            [[NSNotificationCenter defaultCenter] removeObserver:obs];
        }];
    }
}

```

> 因为类的`+ load`方法在 main 函数执行之前调用，所以我们可以在`+ load`方法记录开始时间，同时监听`UIApplicationDidFinishLaunchingNotification`通知，收到通知时将时间相减作为应用启动时间，这样做有一个好处，不需要侵入到业务方的`main`函数去记录开始时间点。

## FPS

首先来看 wikipedia 上是怎么定义 FPS(Frames Per Second)。
> Frame rate (expressed in frames per second or FPS) is the frequency (rate) at which consecutive images called frames are displayed in an animated display. The term applies equally to film and video cameras, computer graphics, and motion capture systems. Frame rate may also be called the frame frequency, and be expressed in hertz.

通过定义可以看出 FPS 是测量用于保存、显示动态视频的信息数量，每秒钟帧数愈多，所显示的动作就会愈流畅，一般应用只要保持 FPS 在 50-60，应用会给流畅的感觉。

接下来我们看下网络上流传的最多的关于测量 FPS 的方法：

``` objective-c

@implementation YYFPSLabel {
    CADisplayLink *_link;
    NSUInteger _count;
    NSTimeInterval _lastTime;    
}

- (id)init {
    self = [super init];
    if( self ){        
    _link = [CADisplayLink displayLinkWithTarget:[YYWeakProxy proxyWithTarget:self] selector:@selector(tick:)];
    [_link addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
        
    }
    return self;
}

- (void)dealloc {
    [_link invalidate];
}

- (void)tick:(CADisplayLink *)link {
    if (_lastTime == 0) {
        _lastTime = link.timestamp;
        return;
    }
    
    _count++;
    NSTimeInterval delta = link.timestamp - _lastTime;
    if (delta < 1) return;
    _lastTime = link.timestamp;
    float fps = _count / delta;
    _count = 0;    
}

```
> 上面是 YYText 中 Demo 的 YYFPSLabel，主要是基于`CADisplayLink`以屏幕刷新频率同步绘图的特性，尝试根据这点去实现一个可以观察屏幕当前帧数的指示器。`YYWeakProxy`的使用是为了避免循环引用。

值得注意的是基于`CADisplayLink`实现的 FPS 在生产场景中只有指导意义，不能代表真实的 FPS，因为基于`CADisplayLink`实现的 FPS 无法完全检测出当前 Core Animation 的性能情况，因为它只能检测出当前 RunLoop 的帧率。


## 参考资料

* [iOS-System-Services](https://github.com/Shmoopi/iOS-System-Services)
* [GT](https://github.com/Tencent/GT)
* [Optimizing Facebook for iOS start time](https://code.facebook.com/posts/1675399786008080/optimizing-facebook-for-ios-start-time/)
* [Apple 文档：Strategies for Handling App State Transitions](https://developer.apple.com/library/content/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/StrategiesforHandlingAppStateTransitions/StrategiesforHandlingAppStateTransitions.html)
* [iOS关于时间的处理](http://mrpeak.cn/blog/ios-time/)
* [StartupMeasurer](https://github.com/fealebenpae/StartupMeasurer)
* [Frame rate](https://en.wikipedia.org/wiki/Frame_rate)
* [YYText](https://github.com/ibireme/YYText)

