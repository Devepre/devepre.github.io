---
layout: post
title:  "UserDefaults value updating SwiftUI View"
date:   2020-06-24 00:08:00 +0300
tags: SwiftUI UserDefaults PropertyWrapper
---
`@available(iOS 14.0, OSX 10.16, tvOS 14.0, watchOS 7.0, *)`

There is common use case when the app need to update `View` when some value stored from `UserDefaults` storage has been changed. `@AppStorage` property wrapper is for the rescue.

Minimum implementation example can be:

{% highlight swift %}
import SwiftUI

struct ContentView: View {
    
    @AppStorage("logged_in") var isLoggedIn = false
    
    var body: some View {
        Toggle("Is user logged in", isOn: $isLoggedIn)
    }
}
{% endhighlight %}

Few things to notice here:
- Initializer of property wrapper accepts `String` value which represents the `key` for storing.
- Type of the storing value is infered by the compiler automatically so you can omit generic parameter `@AppStorage<Bool>("logged_in") var isLoggedIn = false`
- Created property is of type Bool `var isLoggedIn: Bool { get nonmutating set }`. In fact property wrapper exposes `projectedValue` variable of type `Binding<Bool>`. That means `isLoggedIn` can be used with `$` sign for accessing it's `Binding`. That is exactly what `Toggle` needs as one of the arguments. Whenever user taps on the Toggle, it will switch `isLoggedIn`  variable. Framework will write new value using the appropriate `UserDefaults` key. And that will trigger current view update to reflect change. But that is the exact same behaviour for `Toggle` even without using fancy property wrapper - it toggles automatically when user tap on it. In order to illustrate bindinb even more clearly, let's add one more control.

{% highlight swift %}
import SwiftUI

struct ContentView: View {
    
    @AppStorage("logged_in") var isLoggedIn = false
    
    var body: some View {
        VStack {
            Toggle("Is user logged in", isOn: $isLoggedIn)
            Text(isLoggedIn ? "Logged in" : "Logged out")
        }
    }
}
{% endhighlight %}

This time since `Text` view is declaring to render different text depending on `isLoggedIn` variable, view is recreating each time that variable changes.

Another way to declare variable backed by `UserDefaults` is to specify initial value via initializer of property wrapper itself:

{% highlight swift %}
@AppStorage(wrappedValue: false, "logged_in") var isLoggedIn
{% endhighlight %}

Which one to use is only matter of personal taste. But what is worse to mention is the fact that actual writing of value into the disk only happens with mutating of property wrapper value **only after** initialization. Before that view will use `wrappedValue` from the memory but not from the actual disk storage. You should be aware of that especially if other parts of the application uses "old way" to access `UserDefaults` values like so: `UserDefaults.standard.value(forKey:)`. Consider such example:

{% highlight swift %}
import SwiftUI

struct ContentView: View {
    
    @AppStorage("logged_in") var isLoggedIn = false
    
    var body: some View {
        VStack {
            Toggle("Is user logged in", isOn: $isLoggedIn)
            Text(isLoggedIn ? "Logged in" : "Logged out")
            Button("Get value old way") {
                let isLoggedIn = UserDefaults.standard.value(forKey: "logged_in") as? Bool
                print(isLoggedIn)
            }
        }
    }
}
{% endhighlight %}

If user has never tap on `Toggle`, tap on the `Button` will always print *"nil"* despite `isLoggedIn` will be `false`. Also it's easy to check by exploring `.plist` file of `UserDefaults` store. It's located under *"~Documents/Library/Preferences"* directory of the Simulator. You can check path by printing out  `NSHomeDirectory()` somewhere in the app life cycle.

---

Although declaration of wrapper is generic `@frozen @propertyWrapper public struct AppStorage<Value> : DynamicProperty`, all available initializers has some constraint. It means you can't use `@AppStorage` property wrapper for any type you want. 

Out of the box `AppStorage` works with such types: `Bool, Int, Double, String, URL, Data` and `RawRepresentable`. If you will try to use this property wrapper for other type, compiler will complain with error message `No exact matches in call to initializer`.
Interesting case here is `RawRepresentable`. It's allowed to use Enums for storing values to the `UserDefaults` store, but Enum need to satisfy `RawRepresentable` protocol while `RawValue` should be either `String` or `Int`. Here is an example:

{% highlight swift %}
import SwiftUI

struct ContentView: View {
    
    enum UserType: String {
        case admin
        case member
        case guest
    }
    
    @AppStorage("user_role") var userType = UserType.member
    
    var body: some View {
        VStack {
            Text(userType.rawValue)
            Button("Make admin") {
                userType = .admin
            }
        }
    }
}
{% endhighlight %}

With that being said, it can be not only Enum but a Struct, for example. Just need to implement protocol:

{% highlight swift %}
import SwiftUI

struct ContentView: View {
    
    @AppStorage("user_role") var userType = TopUser(rawValue: 12)
    
    var body: some View {
        VStack {
            Text(String(userType.rawValue))
            Button("Make admin") {
                userType = TopUser(rawValue: 42)
            }
        }
    }
}

struct TopUser: RawRepresentable {
    var rawValue: Int
}
{% endhighlight %}
