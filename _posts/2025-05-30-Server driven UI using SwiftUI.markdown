---
layout: post
title:  "Server driven UI using SwiftUI"
date:   2025-05-30 05:20:00 +0200
tags: SwiftUI, Swift, UI
---
Few years ago server driven UI on Apple platforms was a hot topic. I was working on it even earlier - starting from 2019 with introducing of SwiftUI beta.

Project requirements were obvious: to be able to configure app UI from a server. Different clients have the same app which will show different UI depending on server response. Declarative nature of SwiftUI makes this task to achieve much easier rater than via UIKit. With a permissions from a customer I want to reveal approach I've took 6 years back. Let's get started.

Original wording for a server driven piece of UI was "Widget". However, since Apple has already introduced [WidgetKit](https://developer.apple.com/documentation/widgetkit) which make intensive use of "Widget", let's agree on another term - "Viewlet".

> Viewlet: a portmanteau of "view" and "widget," suggesting a smaller, configurable view. In certain content management systems, a region of a page where customizable content can be rendered.

Let's get started! SwiftUI is a declarative way to describe UI. Views are highly composable elements which can be described as a tree. Each view is a node or branch of a tree. Each view may be a standard view container like `VStack` or a custom view (viewlet). JSON is a neat option to describe this hieararchy in a human readable way. Pretty much any JSON editor offers text and tree view. The idea is to create JSON schema which can describe two entities:

1. Standard SwiftUI view views such as `ZStack`, `VStack`, `ScrollView`, `Spacer`, etc.

2. Custom (often complex) views which will be called `Viewlet`

Since application will never know view hierarchy beforehand, it need a way to parse JSON objects of any nestedness. It's a perfect scenario to use recursive algorithm. It need to traverse down view hierarchy (encoded as JSON file) untill object is a view but not a view contaier or element of array. As a result it's a perfect use case for Swift protocols and generics. Let's define those:

1. `StackInsertable` is a protocol to identify elements which can be inserted into a stack. Stack will contain elements which are part of view hierarchy. Each element can have children and each child can have its own children:
{% highlight swift %}
public protocol Childrenable {
    var children: [Self]? { get }
}

public protocol StackInsertable: Identifiable & Childrenable { }
{% endhighlight %}

Let's define first data type - enumeration `StackElement`:
{% highlight swift %}
public enum StackElement<InputObject, FinalObject> where InputObject: StackInsertable {
    case stackOperator(item: StackOperator)
    case inputObject(item: InputObject)
    case finalObject(item: FinalObject)
    
    public enum StackOperator {
        case opening
        case closing
    }
}
{% endhighlight %}

And a final protocol which defines ability to convert object `StackFromToConverter`:

{% highlight swift %}
public protocol StackFromToConverter {
    associatedtype Input: StackInsertable
    associatedtype Output
    
    func convert(inputObjectFromStack: Input) throws -> Output
    func convert(stack: [StackElement<Input, Output>]) throws -> Output
}
{% endhighlight %}

As we can see it has only two requirements:
1. To be able to convert some input to output (meaning single child)
2. To be able to convert array of `StackElement`s (meaning something that may have children)

Finally we can focus on a main part - algorithm itself. `RecursiveTransformer` class instance will encapsulate it. Before doing so, let's think about SOLID principles and demonstrate real application of some of those principles:
1. Single Responsibility. Let's make `RecursiveTransformer` simple and performing only one task - transforming some kind of input which can be described as something that have children elements into output.
2. Open-Closed. Algorithm is generic and encapsulted into class behaviour. In other words the algorithm itself is a closed part while input and output may be changed without modifications of the class code. However we still will be able to extend class functionality via Swift extensions if needed.
3. Interface segregation. Previously introduced `StackFromToConverter` is a very simple interface (protocol) which relays on other simple interfaces `StackInsertable` and `Childrenable`. Those will be used as generic constraints over the class.
4. Dependency Inversion. Instead of using concrete implementation when injecting dependency, class will use simple segregated protocol `StackFromToConverter`.

 This is where Swift Generics and Protocols shine. `RecursiveTransformer` will be generic over `StackFromToConverter` converter which will be injected via constructor using Dependency Injection technique.
 
 {% highlight swift %}
 public final class RecursiveTransformer<Converter> where Converter: StackFromToConverter {
    private let converter: Converter
    private var stack = [StackElement<Converter.Input, Converter.Output>]()
    
    public init(with converter: Converter) {
        self.converter = converter
    }
    ...
}
{% endhighlight %}

Here is implementation of the algorithm:

{% highlight swift %}
public func transform(input object: Converter.Input, deviceType: DeviceType) throws -> Converter.Output {
    /// Recursion exit point: stack has only one final object. It's result itself.
    if let first = stack.first,
       case let StackElement.finalObject(finalObject) = first {
        
        stack.removeAll()
        return finalObject
    }
    
    /// If there are children, need to pass those recursively down to proceed
    /// and push elements to the stack after proceeding recursively.
    if let children = object.children {
        var chunk = [StackElement<Converter.Input, Converter.Output>]()
        chunk.append(.inputObject(item: object))
        
        chunk.append(.stackOperator(item: StackElement.StackOperator.opening))
        for child in children {
            let finalObject = try transform(input: child, deviceType: deviceType)
            chunk.append(.finalObject(item: finalObject))
        }
        chunk.append(.stackOperator(item: StackElement.StackOperator.closing))
        
        /// Convert final item using factory and reutrn back.
        return try converter.convert(stack: chunk)
    } else {
        /// Otherwise current input object is ready for converting to the output object via factory.
        return try converter.convert(inputObjectFromStack: object)
    }
}
{% endhighlight %}

Now it's time to reveal implementation of `StackFromToConverter` which is tailored to create SwiftUI views. Here is a glympse view of it:

{% highlight swift %}
final class ScreenConverter: StackFromToConverter {
    func convert(stack: [StackItem]) throws -> Output {
        ...
        switch parentInputObject.type {
        case .zStack, .vStack, .hStack:
            return createStackType(from: parentInputObject, with: array)
            
        case .scrollView:
            return Output(
                ScrollView {
                    ForEach(array) { $0.element }
                }
                .erased()
            )
            ...
        case .navigationView:
            return Output(
                NavigationView {
                    ForEach(array) { $0.element }
                }
            )
            
        case .widget, .spacer, .unsupported:
            throw ConverterError.unsupportedParrentTypeElement(firstElement)
        }
    }
    
    func convert(inputObjectFromStack: Input) throws -> Output {
        switch inputObjectFromStack.type {
        case .widget:
            return Output(
                widgetCreator.getView(screenObject: inputObjectFromStack)
            )
            ...
        case .spacer:
            return Output(Spacer().erased())
        }
    }
    
    func createStackType(from object: Input, with array: [Holder<Output>]) -> Output {
        switch object.type {
        case .hStack:
            return Output(
                GeometryReader { geometry in
                    HStack(alignment: object.alignment.systemHStack, spacing: object.spacing) {
                        ForEach(array) { item in
                            let screenObject = object.children!.first { $0.id == item.id }!
                            item.element.sizedView(using: geometry.size, screenObject: screenObject)
                        }
                    }
                }.erased()
            )
            ...
        }
    }
}
{% endhighlight %}

Now let's take a look how simplified input JSON file may look like:
{% highlight json %}
{
    "layouts": [
        {
            "key": "ipad",
            "screen": {
                "type": "hStack",
                "children": [
                    {
                        "type": "navigationContainer",
                        "children": [
                            {
                                "type": "widget",
                                 "widgetID": "featureList.0"
                            }
                        ]
                    },
                    {
                        "type": "widget",
                        "widgetID": "featureList.0"
                    }
                ]
            }
        }
    ]
}
{% endhighlight %}

This example is describing layout **only for iPad** device. It's a HStack with 2 children: Navigation Container and Feature List Viewlet. Navigation itself has another child - Feature List Viewlet.

Rendered screen will look like:

![Code example](/assets/images/code_000011.png)

It was very shortened version which demonstrates core approach to a server driven UI using SwiftUI. Navigation, configuration, parsing, app behaviour using Redux approach and other topics were not covered here at all. For a full working prototype please take a look source code of a demo project.

Thanks for reading.

---

### Source code

You can find [complete project on GitHub](https://github.com/Devepre/blog_sources/tree/main/ViewletApp).

### Questions or proposals? Reach me out at [Linkedin](https://www.linkedin.com/in/serhii-kyrylenko-232189110)

