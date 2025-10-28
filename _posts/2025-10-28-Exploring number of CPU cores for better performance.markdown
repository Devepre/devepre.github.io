---
layout: post
title:  "Exploring number of CPU cores for better performance"
date:   2025-10-28 05:20:00 +0200
tags: Swift, concurrency, CPU
---
It's a well known fact modern applications may schedule multiple tasks to run simultaniously. You may write your own algorithm leveraging multi CPU environment or use API which may be configured to do so. But how to get know what is the best number of cores or threads to use? The very straightforward approach would be to get number of physical (or even logical) CPU cores.
There is an API for that [processorCount](https://developer.apple.com/documentation/foundation/processinfo/processorcount) and example of usage would be:
{% highlight swift %}
let cpuCount = ProcessInfo.processInfo.processorCount
{% endhighlight %}

`cpuCount` will be the number of processing cores available on the device.

Starting from A10 Fusion processor (iPhone 7) and all Apple Silicon processors, CPUs have assymetric cores. Those are performance cores and efficiency cores. As those names states, performance cores are designed to handle high-performance signle-threaded heavy tasks that require a lot of power. While efficiency cores designed to run low demand, typically background tasks, to save energy. Usually CPUs have more performance and less efficiency cores.

[Tuning your code’s performance for Apple silicon](https://developer.apple.com/documentation/apple-silicon/tuning-your-code-s-performance-for-apple-silicon):
> Task’s QoS class influences whether the system runs that task on a performance core (P core) or efficiency core (E core). For example, the system is more likely to run Background tasks on E cores to maximize battery life. If you don’t assign QoS classes, your app’s responsiveness and efficiency may suffer as a result.

It means it would be not efficient to dispatch heavy work to the all available CPU cores. Instead it's better to use only performance cores. [sysctl](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man3/sysctl.3.html) is to the rescue. In order to get number of performance cores need to run with `hw.perflevel0.physicalcpu` parameter. While for number of efficiency cores it's `hw.perflevel1.physicalcpu`. Which in summary should be equal to `cpuCount` we got earlier. And here is Swift code to actually get this info from withing the running app:

{% highlight swift %}
func getNumberOfPerformanceCores() -> Int32? {
    var cpuPerformanceCount: Int32? = nil
    var perflevel0: Int32 = 0
    var size = MemoryLayout<Int64>.size
    if sysctlbyname("hw.perflevel0.physicalcpu", &perflevel0, &size, nil, 0) != 0 {
        print("Couldn't get number of performance cores")
    } else {
        cpuPerformanceCount = perflevel0
    }
    return cpuPerformanceCount
}

let cpuCount = getNumberOfPerformanceCores()
{% endhighlight %}

This approach was tested on an app which process audio. Investigation via Xcode Instruments and measuring time confirmed best performance achieved when dispatching tasks to a number of threads which is equal to number of performance cores available while keeping UI responsive.

Thanks for reading.

---

### Questions or proposals? Reach me out at [Linkedin](https://www.linkedin.com/in/serhii-kyrylenko-232189110)

