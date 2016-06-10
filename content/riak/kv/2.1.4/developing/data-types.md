---
title_supertext: "Developing with Riak KV"
title: "Data Types"
description: ""
project: "riak_kv"
project_version: "2.1.4"
menu:
  riak_kv-2.1.4:
    name: "Data Types"
    identifier: "developing_data_types"
    weight: 102
    parent: "developing"
toc: true
aliases:
  - /riak/2.1.4/dev/using/data-types
  - /riak/kv/2.1.4/dev/using/data-types
  - /riak/2.1.4/dev/data-modeling/data-types
  - /riak/kv/2.1.4/dev/data-modeling/data-types
canonical_link: "https://docs.basho.com/riak/kv/latest/developing/data-types"
---

[wiki crdt]: https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type#Others
[concept crdt]: ../../learn/concepts/crdts
[ops bucket type]: ../../using/cluster-operations/bucket-types

Riak KV 2.0+ introduces Riak-specific data types based on [convergent replicated data types (CRDTs)][wiki crdt]. While Riak KV was built as a data-agnostic key/value store, Riak Data Types enable you to use Riak KV as a _data-aware_ system and perform transactions on 5 CRDT-inspired data types:

- Flags
- Registers
- [Counters](#counters)
- [Sets](#sets)
- [Maps](#maps)

Counters, Sets, and Maps can be used as bucket-level data types or types that you can interact with directly.

Flags and Registers, however, must be [embedded in maps](#maps).

> For more information on how CRDTs work in Riak KV see [Concepts: Data Types][concept crdt].

> **Note on counters in earlier versions of Riak**
>
> Counters are the one CRDT that was available in Riak prior to 2.0
(introduced in version 1.4). The implementation of counters in version 2.0
has been almost completely revamped, so if you are using Riak version
2.0 or later we strongly recommend that you follow the usage
documentation here rather than documentation for the older version of
counters.

## Setting Up Buckets to Use Riak Data Types

In order to use Riak Data Types:

1. Create a bucket with the `datatype` parameter set to either: `counter`, `map`, or `set`.
2. Confirm the bucket was properly configured.
3. Activate the bucket-type.

### Creating a Bucket with a Data Type

you must first create a [bucket type][ops bucket type] that sets the `datatype` bucket parameter to either `counter`, `map`, or `set`.

The following would create a separate bucket type for each of the three
bucket-level data types:

```bash
riak-admin bucket-type create maps '{"props":{"datatype":"map"}}'
riak-admin bucket-type create sets '{"props":{"datatype":"set"}}'
riak-admin bucket-type create counters '{"props":{"datatype":"counter"}}'
```

> **Note**
>
> The names `maps`, `sets`, and `counters` are _not_ reserved
terms. You are always free to name bucket types whatever you like, with
the exception of `default`.

### Confirm Bucket configuration

Once you've created a bucket with a Riak Data Type, you can check
to make sure that the bucket property configuration associated with that
type is correct. This can be done through the `riak-admin` interface:

```bash
riak-admin bucket-type status maps
```

This will return a list of bucket properties and their associated values
in the form of `property: value`. If our `maps` bucket type has been set
properly, we should see the following pair in our console output:

```
datatype: map
```

### Active Bucket-type

If a bucket type has been properly constructed, it needs to be activated
to be usable in Riak. This can also be done using the `bucket-type`
command interface:

```bash
riak-admin bucket-type activate maps
```

To check whether activation has been successful, simply use the same
`bucket-type status` command shown above.

## Data Types and Search

Riak Data Types can be searched just like any other object, but with the
added benefit that your Data Type is indexed as a different type by Solr,
the search platform undergirding Riak Search.

In our Search documentation we offer a [full tutorial](../../usage/searching-data-types) as well as a variety of [examples](../../usage/search/#data-types-and-search-examples), including code
samples from each of our official client libraries.

## Usage Examples

The examples below show you how to use Riak Data Types at the
application level using each of Basho's [officially supported Riak clients](../../client-libraries).

All examples will use the bucket type names from above (`counters`, `sets`, and `maps`). You're free to substitute your own bucket type names if you wish.

> **Getting started with Riak clients**
>
> If you are connecting to Riak using one of Basho's official [client libraries](../../client-libraries), you can find more information about getting started with your client in our [Developing with Riak KV: Getting Started](../../getting-started) section.

## Data Types and Context

When performing normal key/value updates in Riak, we advise that you use
[causal context](/riak/kv/2.1.4/learn/concepts/causal-context), which enables Riak to make intelligent decisions
behind the scenes about which object values should be considered more
causally recent than others in cases of conflict. In some of the
examples above, you saw references to **context** metadata included with
each Data Type stored in Riak.

Data Type contexts are similar to [causal context](/riak/kv/2.1.4/learn/concepts/causal-context) in that they are
opaque (i.e. not readable by humans) and also perform a similar function
to that of causal context, i.e. they inform Riak which version of the
Data Type a client is attempting to modify. This information is required
by Riak when making decisions about convergence.

In the example below, we'll fetch the context from the user data map we
created for Ahmed, just to see what it looks like:

```java
// Using the "ahmedMap" Location from above:

FetchMap fetch = new FetchMap.Builder(ahmedMap).build();
FetchMap.Response response = client.execute(fetch);
Context ctx = response.getContext();
System.out.prinntln(ctx.getValue().toString())

// An indecipherable string of Unicode characters should then appear
```

```ruby
bucket = client.bucket('users')
ahmed_map = Riak::Crdt::Map.new(bucket, 'ahmed_info', 'maps')
ahmed_map.instance_variable_get(:@context)

# => "\x83l\x00\x00\x00\x01h\x02m\x00\x00\x00\b#\t\xFE\xF9S\x95\xBD3a\x01j"
```

```php
$map = (new \Basho\Riak\Command\Builder\FetchMap($riak))
    ->atLocation($location)
    ->build()
    ->execute()
    ->getMap();

echo $map->getContext(); // g2wAAAACaAJtAAAACLQFHUkv4m2IYQdoAm0AAAAIxVKxCy5pjMdhCWo=
```

```python
bucket = client.bucket_type('maps').bucket('users')
ahmed_map = Map(bucket, 'ahmed_info')
ahmed_map.context

# g2wAAAABaAJtAAAACCMJ/vlTlb0zYQFq
```

```csharp
// https://github.com/basho/riak-dotnet-client/blob/develop/src/RiakClientExamples/Dev/Using/DataTypes.cs

// Note: using a previous UpdateMap or FetchMap result
Console.WriteLine(format: "Context: {0}", args: Convert.ToBase64String(result.Context));

// Output:
// Context: g2wAAAACaAJtAAAACLQFHUkv4m2IYQdoAm0AAAAIxVKxCy5pjMdhCWo=
```

```javascript
var options = {
    bucketType: 'maps',
    bucket: 'customers',
    key: 'ahmed_info'
};

client.fetchMap(options, function (err, rslt) {
    if (err) {
        throw new Error(err);
    }

    logger.info("context: '%s'", rslt.context.toString('base64'));
});

// Output:
// context: 'g2wAAAACaAJtAAAACLQFHUmjDf4EYTBoAm0AAAAIxVKxC6F1L2dhSWo='
```

```erlang
%% You cannot fetch a Data Type's context directly using the Erlang
%% client. This is actually quite all right, as the client automatically
%% manages contexts when making updates.
```

<div class="note">
<div class="title">Context with the Ruby, Python, and Erlang clients</div>
In the Ruby, Python, and Erlang clients, you will not need to manually
handle context when making Data Type updates. The clients will do it all
for you. The one exception amongst the official clients is the Java
client. We'll explain how to use Data Type contexts with the Java client
directly below.
</div>

#### Context with the Java and PHP Clients

With the Java and PHP clients, you'll need to manually fetch and return Data Type
contexts for the following operations:

* Disabling a flag within a map
* Removing an item from a set (whether the set is on its own or within a
  map)
* Removing a field from a map

Without context, these operations simply will not succeed due to the
convergence logic driving Riak Data Types. The example below shows you
how to fetch a Data Type's context and then pass it back to Riak. More
specifically, we'll remove the `paid_account` flag from the map:

```java
// This example uses our "ahmedMap" location from above:

FetchMap fetch = new FetchMap.Builder(ahmedMap)
    .build();
FetchMap.Response response = client.execute(fetch);
Context ctx = response.getContext();
MapUpdate removePaidAccountField = new MapUpdate()
        .removeFlag("paid_account");
UpdateMap update = new UpdateMap.Builder(ahmedMap, removePaidAccountField)
        .withContext(ctx)
        .build();
client.execute(update);
```


```php
$map = (new \Basho\Riak\Command\Builder\FetchMap($riak))
    ->atLocation($location)
    ->build()
    ->execute()
    ->getMap();

$updateSet = (new \Basho\Riak\Command\Builder\UpdateSet($riak))
    ->remove('opera');

(new \Basho\Riak\Command\Builder\UpdateMap($riak))
    ->updateSet('interests', $updateSet)
    ->atLocation($location)
    ->withContext($map->getContext())
    ->build()
    ->execute();
```

