This document describes the current serializer API (both committed and forward-looking) and provides information on associated features.

Main API issue: https://github.com/dotnet/corefx/issues/34372
_note: there will be additional issues added here for new features_

Design points:
- Due to time constraints, and to gather feedback, the feature set is intended to a minimum viable product for 3.0.
  - An expectation is that a significant percent of Json.NET consumers would be able to use this, especially for ASP.NET scenarios. However, with the 3.0 release being a minimum viable product, that percent is not known. However, the percent will never be 100%, and that is neither a goal nor a long-term goal, because Json.NET has [many features](https://www.newtonsoft.com/json/help/html/JsonNetVsDotNetSerializers.htm) and some are not target requirements.
- Simple POCO object scenarios are targeted. These are typically used for DTO scenarios.
- The API designed to be extensible for new features in subsequent releases and by the community.
- Design-time attributes for defining the various options, but still support modifications at run-time.
- High performance - Minimal CPU and memory allocation overhead on top of the lower-level reader\writer.

# Programming Model
Let's start with coding examples before the formal API is provided.

Using a simple POCO class:
```cs
    public class Person
    {
        public string FirstName { get; set; }
        public string LastName { get; set; }
        public DateTime BirthDay { get; set; }
        public PhoneNumber PrimaryPhoneNumber { get; set; } // PhoneNumber is a custom data type
    }
```

To deserialize JSON bytes into a POCO instance:
```cs
    ReadOnlySpan<byte> utf8= ...
    Person person = JsonSerializer.Parse<Person>(utf8);
```

To serialize an object to JSON bytes:
```cs
    Person person = ...
    byte[] utf8 = JsonSerializer.ToBytes(person);
```

Custom attributes are used to specify serialization semantics. To change the options so that the JSON property names are CamelCased (e.g. "FirstName" to "firstName") an attribute can be applied to the class:
```cs
    [JsonCamelCasingConverter]
    public class Person
````
or to selected property(s):
```cs
    public class Person
    {
...
        [JsonCamelCasingConverter] public string FirstName { get; set; }
...
    }
```

If CamelCasing needs to be specified at run-time, the attribute can be specified globally on an options instance:
```cs
    var options = new JsonSerializerOptions();
    options.PropertyNamePolicyDefault = new JsonCamelCasingAttribute();
    Person poco = JsonSerializer.Parse<Person>(utf8, options);
```
or for a given POCO class:
```cs
    options.GetClassInfo(typeof(Person)).PropertyNamePolicyDefault = new JsonCamelCasingAttribute();
```
or to property(s):
```cs
    options.GetClassInfo(typeof(Person)).GetProperty("LastName").PropertyNamePolicyDefault = new JsonCamelCasingAttribute();
```

To specify a custom PhoneNumber data type converter:
```cs
     // All properties on Person that are of type PhoneNumber will use this converter.
    [MyPhoneNumberConverter]
    public class Person
````
or to property(s):
```cs
    public class Person
    {
...
        [MyPhoneNumberConverter] public PhoneNumber PrimaryPhoneNumber { get; set; }
...
    }
```
or at run-time globally:
```cs
    options.AddDataTypeConverter(new MyPhoneNumberConverterAttribute());
```
or for a given POCO class:
```cs
    options.GetClassInfo(typeof(Person)).AddDataTypeConverter(new MyPhoneNumberConverterAttribute());
```
or to property(s):
```cs
    options.GetClassInfo(typeof(Person)).GetProperty("PrimaryPhoneNumber").DataTypeConverter = new MyPhoneNumberConverterAttribute();
```

# API
## JsonSerializer
This static class is the main entry point.

The string-based `Parse()` and `ToString()` are convenience methods for strings, but slower than using the `<byte>` flavors because UTF8 must be converted to\from UTF16.

```cs
namespace System.Text.Json.Serialization
{
    public static class JsonSerializer
    {
        public static object Parse(ReadOnlySpan<byte> utf8Json, Type returnType, JsonSerializerOptions options = null);
        public static TValue Parse<TValue>(ReadOnlySpan<byte> utf8Json, JsonSerializerOptions options = null);

        public static object Parse(string json, Type returnType, JsonSerializerOptions options = null);
        public static TValue Parse<TValue>(string json, JsonSerializerOptions options = null);

        public static ValueTask<object> ReadAsync(Stream utf8Json, Type returnType, JsonSerializerOptions options = null, CancellationToken cancellationToken = default(CancellationToken));
        public static ValueTask<TValue> ReadAsync<TValue>(Stream utf8Json, JsonSerializerOptions options = null, CancellationToken cancellationToken = default(CancellationToken));

        // Naming of `ToBytes` TBD based on API review; may want to expose a char8[] in addition to byte[].
        public static byte[] ToBytes(object value, Type type, JsonSerializerOptions options = null);
        public static byte[] ToBytes<TValue>(TValue value, JsonSerializerOptions options = null);

        public static string ToString(object value, Type type, JsonSerializerOptions options = null);
        public static string ToString<TValue>(TValue value, JsonSerializerOptions options = null);

        public static Task WriteAsync(object value, Type type, Stream utf8Json, JsonSerializerOptions options = null, CancellationToken cancellationToken = default(CancellationToken));
        public static Task WriteAsync<TValue>(TValue value, Stream utf8Json, JsonSerializerOptions options = null, CancellationToken cancellationToken = default(CancellationToken));
    }
}
```

_note: Json.Net also has a [JsonSerializer class](https://www.newtonsoft.com/json/help/html/T_Newtonsoft_Json_JsonSerializer.htm) although it is instance-based not static. We may want to rename our static class to avoid collisions._

## JsonSerializerOptions
This class contains the options that are used during (de)serialization.

The design-time attributes can be overridden at runtime here or disabled by using the constructor `JsonSerializerOptions(disableDesignTimeAttributes:true)`.

If an instance of `JsonSerializerOptions` is not specified when calling read\write then a default instance is used which is immutable and private. Having a global\static instance is not a viable feature because of unintended side effects when more than one area of code changes the same settings. Having a instance specified per thread\context mechanism is possible, but will only be added pending feedback. It is expected that ASP.NET and other consumers that have non-default settings maintain their own global, thread or stack variable and pass that in on every call. ASP.NET and others may also want to read a .config file at startup in order to initialize the options instance.

An instance of this class and exposed objects will be immutable once (de)serialization has occurred. This allows the instance to be shared globally with the same settings without the worry of side effects. The immutability is also desired with a future code-generation feature. Due to the immutable nature and fine-grain control over options through Attributes, it is expected that the instance is shared across users and applications. We may provide a Clone() method pending feedback.

For performance, when a `JsonSerializerOptions` instance is used, it should be cached or re-used especially when run-time attributes are added because when that occurs, caches are held by the instance instead of being global.

```cs
namespace System.Text.Json.Serialization
{
    public class JsonSerializerOptions
    {
        public JsonSerializerOptions() { }

        // Default value is 16K
        public int DefaultBufferSize { get; set; }

        // These are passed to the Utf8JsonReader\Utf8JsonWriter
        public WriteIndented { get; set; }
        public JsonCommentHandling ReadCommentHandling { get; set; }
        public int MaxReadDepth { get; set; }
		
        // Get the metadata for a type. Used to add run-time attributes to the class or inspect the metadata.
        public JsonTypeInfo GetTypeInfo(Type type);

        // Review note: the members below are used for run-time extensibility.
        // These could all be replaced with a loosely-typed "AddAttribute()" call as in previous
        // iterations of this design. However based on feedback that was not intuitive enough so
        // now each extension point is explicit through the use of a method or property.

        // The metadata set by the members below is used when the same attribute is not present at
        // design-time for a given POCO class or property.

        // Add a data type converter (e.g. a PhoneNumber converter)
        public void AddDataTypeConverter(JsonDataTypeConverterAttribute attribute);

        // Consolidated options for property values
        public JsonPropertyValueAttribute PropertyPolicyDefault { get; set; }

        // Property name policy (e.g. enable camel-casing)
        public JsonPropertyNamePolicyAttribute PropertyNamePolicyDefault { get; set; }
    }
}
```
## JsonPropertyValueAttribute (consolidated options for property values)
This attribute specifies the options that pertain to a POCO property value.

It contains simple primitive values; more complex or extensible ones have their own attributes. Combining multiple settings prevents an explosion of perhaps 10-20 additional attributes\classes in the future.

```cs
namespace System.Text.Json.Serialization
{
    [AttributeUsage(AttributeTargets.Class | AttributeTargets.Property, AllowMultiple = false)]
    public class JsonPropertyValueAttribute : Attribute
    {
        // A null value indicates no setting which can be overridden at a higher level.
        // All bool? properties assume false semantics by default.

        public JsonPropertyValueAttribute() { }

        public bool? CaseInsensitivePropertyName { get; set; }
        public bool? IgnoreNullValueOnRead { get; set; }

        // Several more options TBD
    }
```

## Property Name feature
These attributes determine how a property name can be different between Json and POCO. New attributes (for example, to support a richer camel-casing approach or to add snake-casing) can easily be added by anyone by creating a new attribute derived from `JsonPropertyNamePolicyAttribute`.

_Review note: currently these attributes do not return a "converter" type like other attributes since there is reason such as generics that force it; however for consistency and perhaps re-use outside of "attributes" perhaps the Read\Write methods should exist on a new converter type_

```cs
namespace System.Text.Json.Serialization
{
    [AttributeUsage(AttributeTargets.Class | AttributeTargets.Property, AllowMultiple = false)]
    public class JsonCamelCasingConverterAttribute : JsonPropertyNamePolicyAttribute
    {
        public JsonCamelCasingConverterAttribute();

        public override string Read(string value);
        public override string Write(string value);
    }

    [AttributeUsage(AttributeTargets.Property, AllowMultiple = false)]
    public class JsonPropertyNameAttribute : JsonPropertyNamePolicyAttribute
    {
        public JsonPropertyNameAttribute();
        public JsonPropertyNameAttribute(string name);

        public string Name { get; set; }

        public override string Read(string value);
        public override string Write(string value);
    }
}

namespace System.Text.Json.Serialization.Policies
{
    [AttributeUsage(AttributeTargets.Class | AttributeTargets.Property, AllowMultiple = false)]
    public abstract class JsonPropertyNamePolicyAttribute : Attribute
    {
        public bool? CaseInsensitive { get; set; }

        public abstract string Read(string value);
        public abstract string Write(string value);
    }
}
```

## Date Converter feature
Dates are not part of JSON, so we need a converter. Dates are typically JSON strings. Currently there is an internal `DateTime` converter that currently only supports a limited ISO 8601 format of `"yyyy'-'MM'-'dd'T'HH':'mm':'ss'.'fffffffK"` however this is being replaced with a more general-purpose ISO implemenation via https://github.com/dotnet/corefx/issues/34690.

If the internal converter is not sufficient, such as when the format is a custom format or not ISO 8601 compatible, a developer can add a new type that derives from `JsonDataTypeConverterAttribute` and specifies `PropertyType=typeof(DateTime)` on that attribute.

## Enum Converter feature
By default, Enums are treated as longs in the JSON. This is most efficient and supports bit-wise attributes without any extra work.

This attribute, through `TreatAsString=true` allows Enums to be written with their string-based literal value. If this is not sufficient, a developer can derive from `JsonDataTypeConverterAttribute` and have that attribute specify `PropertyType=typeof(Enum)` or the specific Type of enum to define behavior for. Currently bit-wise string-based attributes are not supported (Json.Net supports this through a comma-separated list).

```cs
namespace System.Text.Json.Serialization
{
    [AttributeUsage(AttributeTargets.Class | AttributeTargets.Property, AllowMultiple = true)]
    public sealed class JsonEnumConverterAttribute: JsonDataTypeConverterAttribute
    {
        public JsonEnumConverterAttribute();
        public JsonEnumConverterAttribute(bool treatAsString = default);

        public bool TreatAsString { get; set; }

        public override System.Text.Json.Serialization.Policies.JsonDataTypeConverter<object> GetConverter();
        public override System.Text.Json.Serialization.Policies.JsonDataTypeConverter<TValue> GetConverter<TValue>();
    }
}
```

## ICollection and Array Converter feature
By default there is an internal ICollection converter that supports any concrete class that implements IList and an Array converter that supports jagged arrays (`foo[][]`) but not multidimensional (`foo[,]`).

## Enumerable Converter extensibility feature
The abstract `JsonEnumerableConverterAttribute` attribute is derived from to determine how an IEnumerable is converted by default during deserialization. This policy can be used when the collection type does not implement IList, the property returns an abstract type or interface, or for immutable collection types or similar collections which require all values passed into the constructor.

This attribute is used internally by the default Array converter since Arrays do not implement IList and because the size must be known while creating the Array.

## Policy namespace
This namespace contains classes that are not normally used or seen by the consumer. Instead, they are used by an author when deriving\extending existing attributes. Having them in a separate namespace reduces the class count of the main namespace.

```cs
namespace System.Text.Json.Serialization.Policies
{
    [AttributeUsage(AttributeTargets.Class | AttributeTargets.Property, AllowMultiple = true)]
    public abstract class JsonEnumerableConverterAttribute: Attribute
    {
        public EnumerableConverterAttribute();
        public Type EnumerableType { get; protected set; }
        public JsonEnumerableConverter CreateConverter();
    }

    public abstract class JsonEnumerableConverter
    {
        protected JsonEnumerableConverter();
        public abstract IEnumerable CreateFromList(Type elementType, IList sourceList);
    }

    // Base Attribute class for data type converters.
    [AttributeUsage(AttributeTargets.Class | AttributeTargets.Property, AllowMultiple = true)]
    public abstract class JsonDataTypeConverterAttribute : Attribute
    {
        public Type PropertyType {get; protected set;}
        public abstract JsonDataTypeConverter<TValue> GetConverter<TValue>;
    }

    // Base data type converter class.
    // Review note: this may be refactored to add an abstraction on top of the reader and writer.
    //  This would allow for a single Write() method instead of two (it will handle property name or lack of)
    //  and prevent potential misuses of exposing the Utf8JsonReader and Utf8JsonWriter directly.
    public abstract class JsonDataTypeConverter<TValue>
    {
        // Returning false here will case a JsonReaderException to be thrown.
        public abstract bool TryRead(Type valueType, ref Utf8JsonReader reader, out TValue value);

        // An implementation of this calls the lower-level reader\writer `WriteXXXValue()` methods.
        public abstract void Write(TValue value, ref Utf8JsonWriter writer);

        // An implementation of this calls the lower-level reader\writer `WriteXXX()` methods.
        // Review note: ideally this `Write()` which takes the property name doesn't need to exist; doing that required the lower-level reader\writer to add functionality.
        public abstract void Write(Span<byte> escapedPropertyName, TValue value, ref Utf8JsonWriter writer);
    }
}
```

## Extensibility and manual (de)serialization
There are several scenarios where lower-level callbacks are necessary. Here are some examples:
- OnDeserializing: initialize collections and other properties before serialization occurs.
- OnDeserialized: handle "underflow" scenarios by defaulting unassigned properties; set calculated properties.
- OnSerializing: assign default values as appropriate and write any properties not present in public properties.
- OnSerialized: write any additional properties that occur after the normal properties are serialized.
- OnPropertyRead: manually deserialize the property.
- OnPropertyWrite: manually serialize the property.
- OnMissingPropertyRead: manually deserialize a missing property.
- OnMissingPropertyWrite: manually serialize missing properties.
- OnMissingPropertyDeserialized: apply "overflow" json data and assign to the poco object, or remember it for later for serialization.

From a design standpoint, these callbacks could occur in several locations:
1) On the actual POCO instance.
2) On a converter type for the POCO.
3) As an event raised from the options class or elsewhere.

The current design and prototype supports (1) and (2) through the use of interfaces either applied to a POCO or to a Converter.

These extensibility interfaces exist in the `System.Text.Json.Serialization.Converters` namespace which is used by those that need to author code to perform manual deserialization. This namespace is not used by normal consumers. In the future it will also contain several simple static methods to help with per-property converters such as Enums and DateTime.

```cs
namespace System.Text.Json.Serialization.Converters
{
    public interface IJsonTypeConverterOnDeserialized
    {
        void OnDeserialized(object obj, JsonClassInfo jsonClassInfo, JsonSerializerOptions options);
    }

    public interface IJsonTypeConverterOnDeserializing
    {
        void OnDeserializing(object obj, JsonClassInfo jsonClassInfo, JsonSerializerOptions options);
    }

    public interface IJsonTypeConverterOnSerialized
    {
        void OnSerialized(object obj, JsonClassInfo jsonClassInfo, in Utf8JsonWriter writer, JsonSerializerOptions options);
    }

    public interface IJsonTypeConverterOnSerializing
    {
        void OnSerializing(object obj, JsonClassInfo jsonClassInfo, in Utf8JsonWriter writer, JsonSerializerOptions options);
    }

   // TBD: the other interfaces mentioned earlier to support OnPropertyRead etc...
}
```

A converter is specified through an attribute:
```cs
namespace System.Text.Json.Serialization
{
    [AttributeUsage(AttributeTargets.Class, AllowMultiple = false)]
    public class JsonClassAttribute : System.Attribute
    {
        public JsonClassAttribute() { }
        public Type ConverterType { get; set; }
    }
}
```
The `JsonClassInfo` type passed into the callbacks support exposing metadata on the object including its System.Type and exposes the type's properties and their converters, escaped name, etc:
```cs
namespace System.Text.Json.Serialization
{
    public sealed class JsonClassInfo
    {
        public System.Type Type { get; }

        public JsonPropertyInfo GetProperty(ReadOnlySpan<byte> propertyName);
        public JsonPropertyInfo GetProperty(string propertyName);

        public void AddDataTypeConverter(JsonDataTypeConverterAttribute attribute);
        public JsonPropertyValueAttribute PropertyPolicyDefault { get; set; }
        public JsonPropertyNamePolicyAttribute PropertyNamePolicyDefault { get; set; }
    }

    public abstract class JsonPropertyInfo
    {
        public string EscapedName { get; }
        public string Name { get; }
        public PropertyInfo PropertyInfo { get; }

        public JsonDataTypeConverterAttribute DataTypeConverter { get; set; }
        public JsonPropertyValueAttribute PropertyPolicyDefault { get; set; }
        public JsonPropertyNamePolicyAttribute PropertyNamePolicyDefault { get; set; }
    }

    // Each JsonPropertyInfo is an instance of this (to avoid boxing)
    public class JsonPropertyInfo<TValue> : JsonPropertyInfo
    {
        // This is the effective data type converter that will be used.
        public JsonDataTypeConverter<TValue> DataTypeConverter { get; }
    }
}
```

Note that `JsonClassInfo`, `JsonPropertyInfo`, and `JsonPropertyInfo<TValue>` are all used internally by the (de)serializer for its core implementation and thus are not created just for extensibility.

# API comparison to Json.NET
## Simple scenario
Json.NET:
```cs
    Person person = ...;
    string result = JsonConvert.SerializeObject(person);
    person = JsonConvert.DeserializeObject(person);
```

JsonSerializer:
```cs
    Person person = ...;
    byte[] result = JsonSerializer.ToBytes(person);
    person = JsonSerializer.Parse(result);
```
## Simple scenario with run-time settings
Json.NET:
```cs
    var settings = new JsonSerializerSettings();
    settings.NullValueHandling = NullValueHandling.Ignore;
    string result = JsonConvert.SerializeObject(person, settings);
```

JsonSerializer:
```cs
    var options = new JsonSerializerOptions();
    options.JsonPropertyValueAttribute = new JsonPropertyValueAttribute
    {
        IgnoreNullValueOnRead = true,
        IgnoreNullValueOnWrite = true
    };
    byte[] result = JsonSerializer.ToBytes(person, options);
```

Note that Json.NET also has a `JsonSerializer` class with instance methods for advanced scenarios. See also Json.NET [code samples](https://www.newtonsoft.com/json/help/html/Samples.htm) and [documentation](https://www.newtonsoft.com/json/help/html/R_Project_Documentation.htm).

## Callbacks
Json.NET:
```cs
    public class Person
    {
        private string _foo;

        [OnDeserializing()]
        internal void OnDeserializing(StreamingContext context)
        {
            _foo = "custom value";
        }
    }
```

JsonSerializer
```cs
    public class Person : IJsonTypeConverterOnDeserializing
    {
        private string _foo;

        void IJsonTypeConverterOnDeserializing.OnDeserializing(object obj, JsonClassInfo jsonClassInfo, JsonSerializerOptions options)
        {
            _foo = "Hello";
        }
    }
```
or using a POCO converter type (so the POCO class itself doesn't need to be modified)
```cs
    // Attribute can also be specified at run-time.
    [JsonClass(ConverterType = typeof(MyPersonConverter))]
    public class Person
    {
        private string _foo;

        public void SetFoo(string value)
        {
            _foo = value;
        }
    }

    internal struct MyPersonConverter : IJsonTypeConverterOnDeserializing
    {
        void IJsonTypeConverterOnDeserializing.OnDeserializing(object obj, JsonClassInfo jsonClassInfo, JsonSerializerOptions options)
        {
            ((MyPerson)obj).SetFoo("Hello");
        }
    }
```

A converter instance is created during deserialization and again during serialization. However since the same instance is used for the entire deserialization (and similarly for serialization), the instance of the converter can maintain state across callbacks, which is useful for logic such as remembering "overflow" property values as they are deserialized in order to use them later during OnDeserialized.

If the converter cannot assign "overflow" state to the POCO for later serialization, then we will need a feature to support the converter adding that state to an user-defined object that will be returned from the `JsonSerializer` "read" methods and then manually passed back to the "write" methods, or adding an attribute that can be applied to a property on the POCO type that represents an overflow property bag.

# Design notes
## Attribute Usage
All supported attributes are located in the `System.Text.Json.Serialization` namespace, although new ones can be added by the community through inheritance as previously discussed (for example for Enums, DateTime, PhoneNumber).

Having all attributes in a single namespace makes it intuitive to find the appropriate functionality. However it is different from some other serializers which re-use attributes from:
- [DataContract] from System.Runtime.Serialization.
- [DataMember] from System.Runtime.Serialization. Typically just the "Name" property is supported.
- [EnumMember] from System.Runtime.Serialization.
- [IgnoreDataMember] from System.Runtime.Serialization.
- [Serializable] and [NonSerializable] from System (for fields).
- ShouldSerializeXXX pattern.

Json.NET for example supports both their own `JsonPropertyAttribute` and `DataMemberAttribute` to specify the name override of a property.

Pending feedback, we may elect to support some of these other attributes to help with compatibility with existing POCOs types.

## Static typing \ polymorphic behavior
The Read\Write methods specify statically (at compile time) the POCO type through `<TValue>` which below is `<Person>`:
```cs
    Person person = JsonSerializer.Parse<Person>(utf8);
    JsonSerializer.ToBytes<Person>(person);

    // Due to generic inference, Write can be simplified as:
    JsonSerializer.ToBytes(person);
```

For Read, this means that if the `utf8` data has additional properties that originally came from a derived class (e.g. a Customer class that derived from Person), only a Person object is instantiated. This is fairly obvious, since there is no hint to say a Customer was originally serialized.

For Write, this means that even if the actual `person` object is a derived type (e.g. a Customer instance), only the data from Person (and base classes, but not derived classes) are serialized. This is as not as obvious as Read, since the Write method could have internally called `object.GetType()` to obtain the type.

Because serialization of an object tree in an opt-out model (all properties are serialized by default), this static typing helps prevent misuses such as accidental data exposure of a derived runtime-created type.

However, static typing also limits polymorphic scenarios. This overload can be used with `GetType()` to address this:
```cs
    JsonSerializer.ToBytes(person, person.GetType());
```

In this case, if `person` is actually a Customer object, the Customer will be serialized.

When POCO objects are returned from other properties or collection, they follow the same static typing based on the property's type or the collection's generic type. There is not a mechanism to support polymorphic behavior here, although that could be added in the future as an opt-in model.

## Async support for Stream and Pipe (Pipe support pending active discussions)
The design supports async methods that can stream with a buffer size around the size of the largest property value which means the capability to asynchronously (de)serialize a very large object tree with a minimal memory footprint.

Currently the async `await` calls on Stream and Pipe is based on a byte threshold determined by the current buffer size.

For the Stream-based async methods, the Stream's `ReadAsync()` \ `WriteAsync()` are awaited. There is no call to `FlushAsync()` - it is expected the consumer does this or uses a Stream or an adapter that can auto-flush.

## Layering of Attributes
Currently the information below applies to `JsonPropertyValueAttribute` and `JsonPropertyNameAttribute`, but the same rules would apply to any similar future property.

Using design-time attributes means that consumers of POCO types should not to apply any run-time options for common scenarios, making it easier for various consumers to properly use these types.

A given Attribute class can typically be applied to a Property or Class either at run-time or design-time. In addition at run-time a global attribute to the current JsonSerializerOptions instance can be specified. The lower-level (e.g. property) having precedence of the higher-level (e.g. class). This allows for global options with override capability.

Note: we could add design-time support for applying an attribute to an Assembly if that feature is desired.

For Attribute classes with several properties\options, a `null` value for a given setting indicates no setting, and a lower-level setting should be used.

The rules listed:
- Attributes added at run-time override design-time attributes when both applied to the same level (global, class or property).
- Attributes at a lower-level (e.g. property) have precedence over high-level (e.g. class).
- A null value of a property on an Attribute (if allowed) means there is no setting. This allows a given Attribute to contain more than one "independent" option and allows assigning that value somewhere else.

The attributes specified at run-time on `JsonSerializerOptions` instance have the least precedence and thus are the "default" values if no design-time attributes exist for them or other run-time attributes specify those same option(s) on a class or property.

## Performance
The goal is to have a super fast (de)serializer given the feature set with minimal overhead on top of the reader and writer.

Design notes:
- Uses IL Emit by default for object creation. Property set\get are handled through direct delegates. There is potential here for additional emit support (including build-time or code-gen support) and\or new reflection features.
- Avoid boxing when feasible. POCO properties that are value types should not be boxed (and unboxed in IL).
- No objects created on the heap during (de)serialization after warm-up, with the exception of the POCO instance and its related state during serialization.
- No expensive reflection or other calls after warm-up (all items cached).
- Optimized lookup for property names using an array of long integers as the key (containing the first 6 bytes of the name plus 2 bytes for size) with fallback to comparing to byte[] when the key is a match (no conversion from byte to string necessary for the name). For POCO types with a high count of properties, we may (TBD) use a secondary algorithm such as a hashtable.

# Pending features not covered here

## Pending features that will affect the public API

### Loosely-typed arrays and objects
Support for the equivalent of `JArray.Load(serializer)` and `JObject.Load(serializer)` to process out-of-order properties or loosely-typed objects that have to be manually (de)serialized.
### Underflow \ overflow feature
#### Default values \ underflow
Support default values for underflow (missing JSON data when deserializing), and trimming of default values when serializing.
#### Overflow
Ability to remember overflow (JSON data with no matching POCO property). Must support round-tripping meaning the ability to pass back this state (obtained during deserialize) back to a serialize method.
### Required Fields
Ability to specify which fields are required.
### Ignored Fields
Ability to specify which fields are to be ignored.

## Pending features that do not affect the public API

### IDictionary support
### DateTime and DateTimeOffset type 
### Default creation of types for collections of type IEnumerable<T>, IList<T> or ICollection<T>.
### Exception semantics for both serialization and deserialization. The reader exception will include message, line number, byte offline and property path. The writer exceptions will include message and property path.
#### Research need for having TryRead methods instead of throwing.

# Currently supported Types
- Array (see the feature notes)
- Boolean
- Byte
- Char (as a JSON string of length 1)
- DateTime (see the feature notes)
- Double
- Enum (see the feature notes)
- Int16
- Int32
- Int64
- IEnumerable (see the feature notes)
- IList (see the feature notes)
- Nullable < T >
- SByte
- Single
- String
- UInt16
- UInt32
- UInt64
