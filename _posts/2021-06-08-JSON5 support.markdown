---
layout: post
title:  "JSON5 support"
date:   2021-06-08 05:20:00 +0200
tags: Swift, JSON
---
`@available(iOS 15.0, OSX 12.0, tvOS 11.0, watchOS 8.0, Xcode 13.0 *)`

[JSON5 specification](https://spec.json5.org)  has been around for few years now. Finally Apple brings support of this standard to it's platforms developers. Starting from iOS 15.0 (and other platforms oviously) [JSONSerialization](https://developer.apple.com/documentation/foundation/jsonserialization) and [JSONDecoder](https://developer.apple.com/documentation/foundation/jsondecoder) now support decoding from JSON5.

In order to leverage this functionality using `JSONDecoder` object, there is a need to explicitly turn it on via [allowsJSON5](https://developer.apple.com/documentation/foundation/jsondecoder/3766916-allowsjson5) boolean property. Minimum implementation example can be:

{% highlight swift %}
import Foundation

struct GroceryProduct: Codable {
    var name: String
    var points: Int
    var description: String?
}

let json = """
{
    "name": "Durian",
    // JSON5 comment
    "points": +600,
    "description": 'A fruit with a "distinctive" scent.' // pay attention to quotes here
}
""".data(using: .utf8)!

let decoder = JSONDecoder()
decoder.allowsJSON5 = true
let product = try decoder.decode(GroceryProduct.self, from: json)

print(product)
{% endhighlight %}

Output is:
`GroceryProduct(name: "Durian", points: 600, description: Optional("A fruit with a \"distinctive\" scent."))`

But what's a big deal of enabling this support? For me the most valuable specification of `JSON5` is [comments](https://spec.json5.org/#comments):

{% highlight swift %}
// This is a single line comment.

/* This is a multi-
   line comment. */
{% endhighlight %}

Some other features are: quotes for strings, hexadecimal digits, leading decimal point, positive sign and more.

Thanks for reading.

---

### Source code

You can find `SWIFT5.playground` [on GitHub repo](https://github.com/Devepre/blog_sources/tree/main/).

### Questions or proposals? Reach me out at [Linkedin](https://www.linkedin.com/in/serhii-kyrylenko-232189110)
