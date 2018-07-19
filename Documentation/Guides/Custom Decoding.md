### Custom Decoding with NetworkRequest

Since the introduction of `Codable`, parsing alternate representations of model objects has become drastically simpler. Hyperspace is no different - it heavily leans on `Codable` to make parsing network responses painless. By default, Hyperspace will opt to use the default instance of `JSONDecoder`. This object expects it's `Date`s represented as a time interval since 1970, it's data represented as a base-64 encoded `String` and will `throw` if encountering any non-conforming floats. 

If the objects you are trying to fetch with a `NetworkRequest` rely on a non-default setting of these properties, you can still use the default `Codable` extensions that are built into Hyperspace - you don't have to rewrite anything. 

If you are using `AnyNetworkRequest<T>`, there are two initializers:

```swift
public init(method: HTTP.Method, url: URL, headers: [HTTP.HeaderKey: HTTP.HeaderValue]?, body: Data?,
    cachePolicy: URLRequest.CachePolicy, timeout: TimeInterval, dataTransformer: @escaping (Data) -> Result<T, AnyError>)
    
public init(method: HTTP.Method, url: URL, headers: [HTTP.HeaderKey: HTTP.HeaderValue]?, body: Data?,
    cachePolicy: URLRequest.CachePolicy, timeout: TimeInterval, decoder: JSONDecoder)
```

The first of these initializers allows you to completely customize the data transformation block to your liking. This is the default initializer that is used with all types. The second initializer is an option only when `ResponseType` is `Decodable`. At this point you simply pass in the `JSONDecoder` instance you need, and Hyperspace will use it to do the rest.

If you are conforming to `NetworkRequest` with your own custom structs, you have similar options. First, you can implement the protocol requirement `transformData(_:)` however you wish - this is a great option if you have not yet gotten around to adopting `Codable` (or if you are parsing something other than JSON). In addition, the `NetworkRequestDefaults` struct contains two more options when you are working with a `ResponseType: Codable`:

```swift
public static func dataTransformer<ResponseType: Decodable, ErrorType: DecodingFailureInitializable>(for decoder: JSONDecoder) -> (Data) -> Result<ResponseType, ErrorType>
public static func dataTransformer<ResponseType: Decodable, ErrorType>(for decoder: JSONDecoder, 
                                                                  catchTransformer: @escaping (Swift.Error, Data) -> E) -> (Data) -> Result<ResponseType, ErrorType>

```

The first of these functions provides a defaul that can be used for any `Codable` type, as long as your error conforms to `DecodingFailureInitializable`. But, in the case your `ErrorType` does not conform, you can use the other function - in which you simply provide a closure to convert a generic error to your type: `(Swift.Error) -> E`

### Decoding With Containers

It is not uncommon to see JSON resembling below (where the object at 'root_key' is what you're trying to decode):

```
{
    "root_key": {
        "title": "Some Title!"
        "subtitle": "Some subtitle"
    }
}
```

To make this as painless as possible, Hyperspace has built support for `DecodableContainer`s. This is a protocol which contains a single child element (the element you want to decode) - all you have to do is specify the root key in the container's `CodingKeys`. For example:

```swift
struct MockDecodableContainer: DecodableContainer {
    var element: MockObject

    private enum CodingKeys: String, CodingKey {
        case element = "root_key"
    }
}
```

In the above example, the container will decode the `MockObject` type at the "root_key" key in the JSON. We've built support in to Hyperspace for these contains extensively:

In `JSONDecoder`:

```swift
func decode<T, U: DecodableContainer>(_ type: T.Type, from data: Data, with container: U.Type) throws -> T where T == U.ContainedType
```

In `AnyNetworkRequest`:

```swift
public init<U: DecodableContainer>(method: HTTP.Method, url: URL, headers: [HTTP.HeaderKey: HTTP.HeaderValue]?, body: Data?,
    cachePolicy: URLRequest.CachePolicy, timeout: TimeInterval, decoder: JSONDecoder, containerType: U.Type) where U.ContainedType == T
    
public init(method: HTTP.Method, url: URL, headers: [HTTP.HeaderKey: HTTP.HeaderValue]? = nil, body: Data? = nil, 
    cachePolicy: URLRequest.CachePolicy, timeout: TimeInterval, decoder: JSONDecoder, rootDecodingKey: String)
```

In addition to the ability for these containers to be generic over the `Decodable` type (as long as they all use the same `CodingKey.element` raw value) you can also simply specify a `String` here. Using this value will sacrifice some runtime performance, but allows you handle much less standard JSON.