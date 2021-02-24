---
layout: post
title:  "Declaring relative size to main screen size for SwiftUI view"
date:   2021-02-24 07:00:00 +0200
tags: SwiftUI GeometryReader View
---
`@available(iOS 13.0, macOS 10.15, tvOS 13.0, watchOS 6.0, *)`

Sometimes there is a need to declare size of particular `View` to be relative to the main screen of application. While it's really easy to achieve for UIKit based apps by using globally shared singletone `UIScreen.main.bounds` property, it's not a case for SwiftUI based app, since UIKit objects only relevant to iOS\iPadOS\Mac Catalyst app. In addition things are going even [more crazy](https://stackoverflow.com/a/57442517/7524146), when you need to be notified about device rotation in order to grab updated width/height values form the `UIScreen.main`.

It would be great to handle this situation purely within SwiftUI tools without relaying on platform-specific `#if #else` statements switching between `UIScreen`, `NSScreen`, `WKInterfaceDevice` and wrapping `UIHostingController`. Using `GeometryReader` you can achieve the goal. Let's see a way how it can be implemented.

Let's assume some view need to be 60% of main screen width. The very naive implementation can be:

{% highlight swift %}
var body: some View {
    GeometryReader { proxy in
        Rectangle()
            .frame(width: proxy.size.width * 0.6)
    }
}
{% endhighlight %}

![Code example](/assets/images/code_00001.png)

The problem is that works only for top level content view. If you are applying this technique deeper in the view hierarchy, chances it will work wrong are very high:

{% highlight swift %}
var body: some View {
    // This is top view
    HStack {
        Text("Hello World").padding()
        
        // This is our view down in the hierarchy
        // For simplicity it's inplace instead of being declared as new view struct
        GeometryReader { proxy in
            Rectangle()
                .foregroundColor(.secondary)
                .frame(width: proxy.size.width * 0.6)
        }
    }
}
{% endhighlight %}

![Code example](/assets/images/code_00002.png)

As you can see from the screenshot, now 60% of width is divided between view itself plus `TextView` from the top level of view hierarchy. And it's not surprise, if you are familiar with how [GeometryReader](https://developer.apple.com/documentation/swiftui/geometryreader) actually works. WIth this knowledge in mind, let's wrap top container view into `GeometryReader` and pass proxy size data down to the hierarchy as environment object. It will allow to use size info from the top level view anywhere deep in the view hierarchy.

First of all, lets create ObservableObject to store size info:

{% highlight swift %}
import SwiftUI

class SizeProvider: ObservableObject {
    struct GeometryProxyInfo {
        let size: CGSize
        let safeAreaInsets: EdgeInsets
        
        init(_ proxy: GeometryProxy?) {
            size = proxy?.size ?? .zero
            safeAreaInsets = proxy?.safeAreaInsets ?? .init()
        }
    }
    
    @Published var proxyInfo: GeometryProxyInfo = GeometryProxyInfo(nil)
    
    func update(with proxy: GeometryProxy) -> Self {
        self.proxyInfo = GeometryProxyInfo(proxy)
        return self
    }
}
{% endhighlight %}

SubViiew will leverage knowledge of top view's size and use it to make specific frame size:

{% highlight swift %}
import SwiftUI

struct SubView: View {
    @EnvironmentObject var viewData: SizeProvider
    let multiplier: CGFloat = 0.6
    
    var body: some View {
        Rectangle()
            .foregroundColor(.secondary)
            .overlay(Text("Details View width:\n\(viewData.proxyInfo.size.width)"))
            .frame(width: viewData.proxyInfo.size.width * multiplier)
    }
}
{% endhighlight %}

And the top level view implementation may look like this:

{% highlight swift %}
import SwiftUI

struct ContentView: View {
    @StateObject var sizeProvider: SizeProvider = SizeProvider()
    
    var body: some View {
        GeometryReader { proxy in
            HStack() {
                Text("Top view")
                    .padding()
                    .background(Color.green)
                    .cornerRadius(20)
                SubView()
                    .environmentObject(sizeProvider.update(with: proxy))
            }
        }
    }
}
{% endhighlight %}

So far so good. Actually it's not! This code will lead to endless cycle of view updates and system will never be able to actually draw the view. That is because every new input from `GeometryReader` will cause `sizeProvider` to update within size from `GeometryProxy` which will lead to `body` reevaluating since `sizeProvider` is `StateObject`. And again. And again.
Instead of `sizeProvider` being wrapped as `StateObject`, let's make it just a regular property. And even more, let's delegate creating of this property to the very top level - `SceneDelegate` in case of iOS 13 or `Scene` in case of iOS 14:

{% highlight swift %}
import SwiftUI

@main
struct CoolApp: App {
    let sizeProvider: SizeProvider = SizeProvider()
    
    var body: some Scene {
        WindowGroup {
            ContentView(sizeProvider: sizeProvider)
        }
    }
}
{% endhighlight %}

and change `@StateObject var sizeProvider: SizeProvider = SizeProvider()` to `let sizeProvider: SizeProvider` in `ContentView`. That's it. Every view down in the hierarchy may use size of `GeometryReader` from the very top view. In addition screen rotation changes automatically handled by SwiftUI framework without any extra line of code on client side.

One consideration here is that `GeometryProxy` size property will contain size of the window excluding safe area insets. That is the reason why `SizeProvider` has `let safeAreaInsets: EdgeInsets` property in addition to `let size: CGSize`. In order to get full size of the app window you need to take into account safe area insets, if they are present in top view.

Result in iOS app:
![Code example](/assets/images/code_00003.png)

Result in macOS app:
![Code example](/assets/images/code_00004.png)

Thanks for reading.

---

### Source code

You can find [complete project on GitHub](https://github.com/Devepre/blog_sources) targeting macOS and iOS platforms under **"RelativeViewSize"** folder.

### Questions or proposals? Reach me out at [Linkedin](https://www.linkedin.com/in/serhii-kyrylenko-232189110)
