---
layout: default
title: "Beam Programming Guide"
permalink: /documentation/programming-guide/
redirect_from:
  - /learn/programming-guide/
  - /docs/learn/programming-guide/
---

# Apache Beam Programming Guide

The **Beam Programming Guide** is intended for Beam users who want to use the
Beam SDKs to create data processing pipelines. It provides guidance for using
the Beam SDK classes to build and test your pipeline. It is not intended as an
exhaustive reference, but as a language-agnostic, high-level guide to
programmatically building your Beam pipeline. As the programming guide is filled
out, the text will include code samples in multiple languages to help illustrate
how to implement Beam concepts in your pipelines.

<nav class="language-switcher">
  <strong>Adapt for:</strong>
  <ul>
    <li data-type="language-java" class="active">Java SDK</li>
    <li data-type="language-py">Python SDK</li>
  </ul>
</nav>

**Table of Contents:**
* TOC
{:toc}


## 1. Overview

To use Beam, you need to first create a driver program using the classes in one
of the Beam SDKs. Your driver program *defines* your pipeline, including all of
the inputs, transforms, and outputs; it also sets execution options for your
pipeline (typically passed in using command-line options). These include the
Pipeline Runner, which, in turn, determines what back-end your pipeline will run
on.

The Beam SDKs provide a number of abstractions that simplify the mechanics of
large-scale distributed data processing. The same Beam abstractions work with
both batch and streaming data sources. When you create your Beam pipeline, you
can think about your data processing task in terms of these abstractions. They
include:

* `Pipeline`: A `Pipeline` encapsulates your entire data processing task, from
  start to finish. This includes reading input data, transforming that data, and
  writing output data. All Beam driver programs must create a `Pipeline`. When
  you create the `Pipeline`, you must also specify the execution options that
  tell the `Pipeline` where and how to run.

* `PCollection`: A `PCollection` represents a distributed data set that your
  Beam pipeline operates on. The data set can be *bounded*, meaning it comes
  from a fixed source like a file, or *unbounded*, meaning it comes from a
  continuously updating source via a subscription or other mechanism. Your
  pipeline typically creates an initial `PCollection` by reading data from an
  external data source, but you can also create a `PCollection` from in-memory
  data within your driver program. From there, `PCollection`s are the inputs and
  outputs for each step in your pipeline.

* `Transform`: A `Transform` represents a data processing operation, or a step,
  in your pipeline. Every `Transform` takes one or more `PCollection` objects as
  input, performs a processing function that you provide on the elements of that
  `PCollection`, and produces one or more output `PCollection` objects.

* I/O `Source` and `Sink`: Beam provides `Source` and `Sink` APIs to represent
  reading and writing data, respectively. `Source` encapsulates the code
  necessary to read data into your Beam pipeline from some external source, such
  as cloud file storage or a subscription to a streaming data source. `Sink`
  likewise encapsulates the code necessary to write the elements of a
  `PCollection` to an external data sink.

A typical Beam driver program works as follows:

* Create a `Pipeline` object and set the pipeline execution options, including
  the Pipeline Runner.
* Create an initial `PCollection` for pipeline data, either using the `Source`
  API to read data from an external source, or using a `Create` transform to
  build a `PCollection` from in-memory data.
* Apply **Transforms** to each `PCollection`. Transforms can change, filter,
  group, analyze, or otherwise process the elements in a `PCollection`. A
  transform creates a new output `PCollection` *without consuming the input
  collection*. A typical pipeline applies subsequent transforms to the each new
  output `PCollection` in turn until processing is complete.
* Output the final, transformed `PCollection`(s), typically using the `Sink` API
  to write data to an external source.
* **Run** the pipeline using the designated Pipeline Runner.

When you run your Beam driver program, the Pipeline Runner that you designate
constructs a **workflow graph** of your pipeline based on the `PCollection`
objects you've created and transforms that you've applied. That graph is then
executed using the appropriate distributed processing back-end, becoming an
asynchronous "job" (or equivalent) on that back-end.

## 2. Creating a pipeline

The `Pipeline` abstraction encapsulates all the data and steps in your data
processing task. Your Beam driver program typically starts by constructing a
<span class="language-java">[Pipeline]({{ site.baseurl }}/documentation/sdks/javadoc/{{ site.release_latest }}/index.html?org/apache/beam/sdk/Pipeline.html)</span>
<span class="language-py">[Pipeline](https://github.com/apache/beam/blob/master/sdks/python/apache_beam/pipeline.py)</span>
object, and then using that object as the basis for creating the pipeline's data
sets as `PCollection`s and its operations as `Transform`s.

To use Beam, your driver program must first create an instance of the Beam SDK
class `Pipeline` (typically in the `main()` function). When you create your
`Pipeline`, you'll also need to set some **configuration options**. You can set
your pipeline's configuration options programatically, but it's often easier to
set the options ahead of time (or read them from the command line) and pass them
to the `Pipeline` object when you create the object.

```java
// Start by defining the options for the pipeline.
PipelineOptions options = PipelineOptionsFactory.create();

// Then create the pipeline.
Pipeline p = Pipeline.create(options);
```
```py
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets.py tag:pipelines_constructing_creating
%}
```

### 2.1. Configuring pipeline options

Use the pipeline options to configure different aspects of your pipeline, such
as the pipeline runner that will execute your pipeline and any runner-specific
configuration required by the chosen runner. Your pipeline options will
potentially include information such as your project ID or a location for
storing files.

When you run the pipeline on a runner of your choice, a copy of the
PipelineOptions will be available to your code. For example, you can read
PipelineOptions from a DoFn's Context.

#### 2.1.1. Setting PipelineOptions from command-line arguments

While you can configure your pipeline by creating a `PipelineOptions` object and
setting the fields directly, the Beam SDKs include a command-line parser that
you can use to set fields in `PipelineOptions` using command-line arguments.

To read options from the command-line, construct your `PipelineOptions` object
as demonstrated in the following example code:

```java
MyOptions options = PipelineOptionsFactory.fromArgs(args).withValidation().create();
```
```py
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets.py tag:pipelines_constructing_creating
%}
```

This interprets command-line arguments that follow the format:

```
--<option>=<value>
```

> **Note:** Appending the method `.withValidation` will check for required
> command-line arguments and validate argument values.

Building your `PipelineOptions` this way lets you specify any of the options as
a command-line argument.

> **Note:** The [WordCount example pipeline]({{ site.baseurl }}/get-started/wordcount-example)
> demonstrates how to set pipeline options at runtime by using command-line
> options.

#### 2.1.2. Creating custom options

You can add your own custom options in addition to the standard
`PipelineOptions`. To add your own options, define an interface with getter and
setter methods for each option, as in the following example:

```java
public interface MyOptions extends PipelineOptions {
    String getMyCustomOption();
    void setMyCustomOption(String myCustomOption);
  }
```
```py
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets.py tag:pipeline_options_define_custom
%}
```

You can also specify a description, which appears when a user passes `--help` as
a command-line argument, and a default value.

You set the description and default value using annotations, as follows:

```java
public interface MyOptions extends PipelineOptions {
    @Description("My custom command line argument.")
    @Default.String("DEFAULT")
    String getMyCustomOption();
    void setMyCustomOption(String myCustomOption);
  }
```
```py
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets.py tag:pipeline_options_define_custom_with_help_and_default
%}
```

{:.language-java}
It's recommended that you register your interface with `PipelineOptionsFactory`
and then pass the interface when creating the `PipelineOptions` object. When you
register your interface with `PipelineOptionsFactory`, the `--help` can find
your custom options interface and add it to the output of the `--help` command.
`PipelineOptionsFactory` will also validate that your custom options are
compatible with all other registered options.

{:.language-java}
The following example code shows how to register your custom options interface
with `PipelineOptionsFactory`:

```java
PipelineOptionsFactory.register(MyOptions.class);
MyOptions options = PipelineOptionsFactory.fromArgs(args)
                                                .withValidation()
                                                .as(MyOptions.class);
```

Now your pipeline can accept `--myCustomOption=value` as a command-line argument.

## 3. PCollections

The <span class="language-java">[PCollection]({{ site.baseurl }}/documentation/sdks/javadoc/{{ site.release_latest }}/index.html?org/apache/beam/sdk/values/PCollection.html)</span>
<span class="language-py">`PCollection`</span> abstraction represents a
potentially distributed, multi-element data set. You can think of a
`PCollection` as "pipeline" data; Beam transforms use `PCollection` objects as
inputs and outputs. As such, if you want to work with data in your pipeline, it
must be in the form of a `PCollection`.

After you've created your `Pipeline`, you'll need to begin by creating at least
one `PCollection` in some form. The `PCollection` you create serves as the input
for the first operation in your pipeline.

### 3.1. Creating a PCollection

You create a `PCollection` by either reading data from an external source using
Beam's [Source API](#pipeline-io), or you can create a `PCollection` of data
stored in an in-memory collection class in your driver program. The former is
typically how a production pipeline would ingest data; Beam's Source APIs
contain adapters to help you read from external sources like large cloud-based
files, databases, or subscription services. The latter is primarily useful for
testing and debugging purposes.

#### 3.1.1. Reading from an external source

To read from an external source, you use one of the [Beam-provided I/O
adapters](#pipeline-io). The adapters vary in their exact usage, but all of them
from some external data source and return a `PCollection` whose elements
represent the data records in that source.

Each data source adapter has a `Read` transform; to read, you must apply that
transform to the `Pipeline` object itself.
<span class="language-java">`TextIO.Read`</span>
<span class="language-py">`io.TextFileSource`</span>, for example, reads from an
external text file and returns a `PCollection` whose elements are of type
`String`, each `String` represents one line from the text file. Here's how you
would apply <span class="language-java">`TextIO.Read`</span>
<span class="language-py">`io.TextFileSource`</span> to your `Pipeline` to create
a `PCollection`:

```java
public static void main(String[] args) {
    // Create the pipeline.
    PipelineOptions options =
        PipelineOptionsFactory.fromArgs(args).create();
    Pipeline p = Pipeline.create(options);

    // Create the PCollection 'lines' by applying a 'Read' transform.
    PCollection<String> lines = p.apply(
      "ReadMyFile", TextIO.read().from("protocol://path/to/some/inputData.txt"));
}
```
```py
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets.py tag:pipelines_constructing_reading
%}
```

See the [section on I/O](#pipeline-io) to learn more about how to read from the
various data sources supported by the Beam SDK.

#### 3.1.2. Creating a PCollection from in-memory data

{:.language-java}
To create a `PCollection` from an in-memory Java `Collection`, you use the
Beam-provided `Create` transform. Much like a data adapter's `Read`, you apply
`Create` directly to your `Pipeline` object itself.

{:.language-java}
As parameters, `Create` accepts the Java `Collection` and a `Coder` object. The
`Coder` specifies how the elements in the `Collection` should be
[encoded](#element-type).

{:.language-py}
To create a `PCollection` from an in-memory `list`, you use the Beam-provided
`Create` transform. Apply this transform directly to your `Pipeline` object
itself.

The following example code shows how to create a `PCollection` from an in-memory
<span class="language-java">`List`</span><span class="language-py">`list`</span>:

```java
public static void main(String[] args) {
    // Create a Java Collection, in this case a List of Strings.
    static final List<String> LINES = Arrays.asList(
      "To be, or not to be: that is the question: ",
      "Whether 'tis nobler in the mind to suffer ",
      "The slings and arrows of outrageous fortune, ",
      "Or to take arms against a sea of troubles, ");

    // Create the pipeline.
    PipelineOptions options =
        PipelineOptionsFactory.fromArgs(args).create();
    Pipeline p = Pipeline.create(options);

    // Apply Create, passing the list and the coder, to create the PCollection.
    p.apply(Create.of(LINES)).setCoder(StringUtf8Coder.of())
}
```
```py
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets.py tag:model_pcollection
%}
```

### 3.2. PCollection characteristics

A `PCollection` is owned by the specific `Pipeline` object for which it is
created; multiple pipelines cannot share a `PCollection`. In some respects, a
`PCollection` functions like a collection class. However, a `PCollection` can
differ in a few key ways:

#### 3.2.1. Element type

The elements of a `PCollection` may be of any type, but must all be of the same
type. However, to support distributed processing, Beam needs to be able to
encode each individual element as a byte string (so elements can be passed
around to distributed workers). The Beam SDKs provide a data encoding mechanism
that includes built-in encoding for commonly-used types as well as support for
specifying custom encodings as needed.

#### 3.2.2. Immutability

A `PCollection` is immutable. Once created, you cannot add, remove, or change
individual elements. A Beam Transform might process each element of a
`PCollection` and generate new pipeline data (as a new `PCollection`), *but it
does not consume or modify the original input collection*.

#### 3.2.3. Random access

A `PCollection` does not support random access to individual elements. Instead,
Beam Transforms consider every element in a `PCollection` individually.

#### 3.2.4. Size and boundedness

A `PCollection` is a large, immutable "bag" of elements. There is no upper limit
on how many elements a `PCollection` can contain; any given `PCollection` might
fit in memory on a single machine, or it might represent a very large
distributed data set backed by a persistent data store.

A `PCollection` can be either **bounded** or **unbounded** in size. A
**bounded** `PCollection` represents a data set of a known, fixed size, while an
**unbounded** `PCollection` represents a data set of unlimited size. Whether a
`PCollection` is bounded or unbounded depends on the source of the data set that
it represents. Reading from a batch data source, such as a file or a database,
creates a bounded `PCollection`. Reading from a streaming or
continously-updating data source, such as Pub/Sub or Kafka, creates an unbounded
`PCollection` (unless you explicitly tell it not to).

The bounded (or unbounded) nature of your `PCollection` affects how Beam
processes your data. A bounded `PCollection` can be processed using a batch job,
which might read the entire data set once, and perform processing in a job of
finite length. An unbounded `PCollection` must be processed using a streaming
job that runs continuously, as the entire collection can never be available for
processing at any one time.

When performing an operation that groups elements in an unbounded `PCollection`,
Beam requires a concept called **windowing** to divide a continuously updating
data set into logical windows of finite size.  Beam processes each window as a
bundle, and processing continues as the data set is generated. These logical
windows are determined by some characteristic associated with a data element,
such as a **timestamp**.

#### 3.2.5. Element timestamps

Each element in a `PCollection` has an associated intrinsic **timestamp**. The
timestamp for each element is initially assigned by the [Source](#pipeline-io)
that creates the `PCollection`. Sources that create an unbounded `PCollection`
often assign each new element a timestamp that corresponds to when the element
was read or added.

> **Note**: Sources that create a bounded `PCollection` for a fixed data set
> also automatically assign timestamps, but the most common behavior is to
> assign every element the same timestamp (`Long.MIN_VALUE`).

Timestamps are useful for a `PCollection` that contains elements with an
inherent notion of time. If your pipeline is reading a stream of events, like
Tweets or other social media messages, each element might use the time the event
was posted as the element timestamp.

You can manually assign timestamps to the elements of a `PCollection` if the
source doesn't do it for you. You'll want to do this if the elements have an
inherent timestamp, but the timestamp is somewhere in the structure of the
element itself (such as a "time" field in a server log entry). Beam has
[Transforms](#transforms) that take a `PCollection` as input and output an
identical `PCollection` with timestamps attached; see [Assigning
Timestamps](#adding-timestamps-to-a-pcollections-elements) for more information
about how to do so.

## 4. Transforms

Transforms are the operations in your pipeline, and provide a generic
processing framework. You provide processing logic in the form of a function
object (colloquially referred to as "user code"), and your user code is applied
to each element of an input `PCollection` (or more than one `PCollection`).
Depending on the pipeline runner and back-end that you choose, many different
workers across a cluster may execute instances of your user code in parallel.
The user code running on each worker generates the output elements that are
ultimately added to the final output `PCollection` that the transform produces.

The Beam SDKs contain a number of different transforms that you can apply to
your pipeline's `PCollection`s. These include general-purpose core transforms,
such as [ParDo](#pardo) or [Combine](#combine). There are also pre-written
[composite transforms](#composite-transforms) included in the SDKs, which
combine one or more of the core transforms in a useful processing pattern, such
as counting or combining elements in a collection. You can also define your own
more complex composite transforms to fit your pipeline's exact use case.

### 4.1. Applying transforms

To invoke a transform, you must **apply** it to the input `PCollection`. Each
transform in the Beam SDKs has a generic `apply` method <span class="language-py">(or pipe operator `|`)</span>.
Invoking multiple Beam transforms is similar to *method chaining*, but with one
slight difference: You apply the transform to the input `PCollection`, passing
the transform itself as an argument, and the operation returns the output
`PCollection`. This takes the general form:

```java
[Output PCollection] = [Input PCollection].apply([Transform])
```
```py
[Output PCollection] = [Input PCollection] | [Transform]
```

Because Beam uses a generic `apply` method for `PCollection`, you can both chain
transforms sequentially and also apply transforms that contain other transforms
nested within (called [composite transforms](#composite-transforms) in the Beam
SDKs).

How you apply your pipeline's transforms determines the structure of your
pipeline. The best way to think of your pipeline is as a directed acyclic graph,
where the nodes are `PCollection`s and the edges are transforms. For example,
you can chain transforms to create a sequential pipeline, like this one:

```java
[Final Output PCollection] = [Initial Input PCollection].apply([First Transform])
.apply([Second Transform])
.apply([Third Transform])
```
```py
[Final Output PCollection] = ([Initial Input PCollection] | [First Transform]
              | [Second Transform]
              | [Third Transform])
```

The resulting workflow graph of the above pipeline looks like this:

[Sequential Graph Graphic]

However, note that a transform *does not consume or otherwise alter* the input
collection--remember that a `PCollection` is immutable by definition. This means
that you can apply multiple transforms to the same input `PCollection` to create
a branching pipeline, like so:

```java
[Output PCollection 1] = [Input PCollection].apply([Transform 1])
[Output PCollection 2] = [Input PCollection].apply([Transform 2])
```
```py
[Output PCollection 1] = [Input PCollection] | [Transform 1]
[Output PCollection 2] = [Input PCollection] | [Transform 2]
```

The resulting workflow graph from the branching pipeline above looks like this:

[Branching Graph Graphic]

You can also build your own [composite transforms](#composite-transforms) that
nest multiple sub-steps inside a single, larger transform. Composite transforms
are particularly useful for building a reusable sequence of simple steps that
get used in a lot of different places.

### 4.2. Core Beam transforms

Beam provides the following core transforms, each of which represents a different
processing paradigm:

* `ParDo`
* `GroupByKey`
* `CoGroupByKey`
* `Combine`
* `Flatten`
* `Partition`

#### 4.2.1. ParDo

`ParDo` is a Beam transform for generic parallel processing. The `ParDo`
processing paradigm is similar to the "Map" phase of a Map/Shuffle/Reduce-style
algorithm: a `ParDo` transform considers each element in the input
`PCollection`, performs some processing function (your user code) on that
element, and emits zero, one, or multiple elements to an output `PCollection`.

`ParDo` is useful for a variety of common data processing operations, including:

* **Filtering a data set.** You can use `ParDo` to consider each element in a
  `PCollection` and either output that element to a new collection, or discard
  it.
* **Formatting or type-converting each element in a data set.** If your input
  `PCollection` contains elements that are of a different type or format than
  you want, you can use `ParDo` to perform a conversion on each element and
  output the result to a new `PCollection`.
* **Extracting parts of each element in a data set.** If you have a
  `PCollection` of records with multiple fields, for example, you can use a
  `ParDo` to parse out just the fields you want to consider into a new
  `PCollection`.
* **Performing computations on each element in a data set.** You can use `ParDo`
  to perform simple or complex computations on every element, or certain
  elements, of a `PCollection` and output the results as a new `PCollection`.

In such roles, `ParDo` is a common intermediate step in a pipeline. You might
use it to extract certain fields from a set of raw input records, or convert raw
input into a different format; you might also use `ParDo` to convert processed
data into a format suitable for output, like database table rows or printable
strings.

When you apply a `ParDo` transform, you'll need to provide user code in the form
of a `DoFn` object. `DoFn` is a Beam SDK class that defines a distributed
processing function.

> When you create a subclass of `DoFn`, note that your subclass should adhere to
> the [Requirements for writing user code for Beam transforms](#requirements-for-writing-user-code-for-beam-transforms).

##### 4.2.1.1. Applying ParDo

Like all Beam transforms, you apply `ParDo` by calling the `apply` method on the
input `PCollection` and passing `ParDo` as an argument, as shown in the
following example code:

```java
// The input PCollection of Strings.
PCollection<String> words = ...;

// The DoFn to perform on each element in the input PCollection.
static class ComputeWordLengthFn extends DoFn<String, Integer> { ... }

// Apply a ParDo to the PCollection "words" to compute lengths for each word.
PCollection<Integer> wordLengths = words.apply(
    ParDo
    .of(new ComputeWordLengthFn()));        // The DoFn to perform on each element, which
                                            // we define above.
```
```py
# The input PCollection of Strings.
words = ...

# The DoFn to perform on each element in the input PCollection.
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets_test.py tag:model_pardo_pardo
%}
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets_test.py tag:model_pardo_apply
%}
```

In the example, our input `PCollection` contains `String` values. We apply a
`ParDo` transform that specifies a function (`ComputeWordLengthFn`) to compute
the length of each string, and outputs the result to a new `PCollection` of
`Integer` values that stores the length of each word.

##### 4.2.1.2. Creating a DoFn

The `DoFn` object that you pass to `ParDo` contains the processing logic that
gets applied to the elements in the input collection. When you use Beam, often
the most important pieces of code you'll write are these `DoFn`s--they're what
define your pipeline's exact data processing tasks.

> **Note:** When you create your `DoFn`, be mindful of the [Requirements
> for writing user code for Beam transforms](#requirements-for-writing-user-code-for-beam-transforms)
> and ensure that your code follows them.

{:.language-java}
A `DoFn` processes one element at a time from the input `PCollection`. When you
create a subclass of `DoFn`, you'll need to provide type parameters that match
the types of the input and output elements. If your `DoFn` processes incoming
`String` elements and produces `Integer` elements for the output collection
(like our previous example, `ComputeWordLengthFn`), your class declaration would
look like this:

```java
static class ComputeWordLengthFn extends DoFn<String, Integer> { ... }
```

{:.language-java}
Inside your `DoFn` subclass, you'll write a method annotated with
`@ProcessElement` where you provide the actual processing logic. You don't need
to manually extract the elements from the input collection; the Beam SDKs handle
that for you. Your `@ProcessElement` method should accept an object of type
`ProcessContext`. The `ProcessContext` object gives you access to an input
element and a method for emitting an output element:

{:.language-py}
Inside your `DoFn` subclass, you'll write a method `process` where you provide
the actual processing logic. You don't need to manually extract the elements
from the input collection; the Beam SDKs handle that for you. Your `process`
method should accept an object of type `element`. This is the input element and
output is emitted by using `yield` or `return` statement inside `process`
method.

```java
static class ComputeWordLengthFn extends DoFn<String, Integer> {
  @ProcessElement
  public void processElement(ProcessContext c) {
    // Get the input element from ProcessContext.
    String word = c.element();
    // Use ProcessContext.output to emit the output element.
    c.output(word.length());
  }
}
```
```py
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets_test.py tag:model_pardo_pardo
%}
```

{:.language-java}
> **Note:** If the elements in your input `PCollection` are key/value pairs, you
> can access the key or value by using `ProcessContext.element().getKey()` or
> `ProcessContext.element().getValue()`, respectively.

A given `DoFn` instance generally gets invoked one or more times to process some
arbitrary bundle of elements. However, Beam doesn't guarantee an exact number of
invocations; it may be invoked multiple times on a given worker node to account
for failures and retries. As such, you can cache information across multiple
calls to your processing method, but if you do so, make sure the implementation
**does not depend on the number of invocations**.

In your processing method, you'll also need to meet some immutability
requirements to ensure that Beam and the processing back-end can safely
serialize and cache the values in your pipeline. Your method should meet the
following requirements:

{:.language-java}
* You should not in any way modify an element returned by
  `ProcessContext.element()` or `ProcessContext.sideInput()` (the incoming
  elements from the input collection).
* Once you output a value using `ProcessContext.output()` or
  `ProcessContext.sideOutput()`, you should not modify that value in any way.

##### 4.2.1.3. Lightweight DoFns and other abstractions

If your function is relatively straightforward, you can simplify your use of
`ParDo` by providing a lightweight `DoFn` in-line, as
<span class="language-java">an anonymous inner class instance</span>
<span class="language-py">a lambda function</span>.

Here's the previous example, `ParDo` with `ComputeLengthWordsFn`, with the
`DoFn` specified as
<span class="language-java">an anonymous inner class instance</span>
<span class="language-py">a lambda function</span>:

```java
// The input PCollection.
PCollection<String> words = ...;

// Apply a ParDo with an anonymous DoFn to the PCollection words.
// Save the result as the PCollection wordLengths.
PCollection<Integer> wordLengths = words.apply(
  "ComputeWordLengths",                     // the transform name
  ParDo.of(new DoFn<String, Integer>() {    // a DoFn as an anonymous inner class instance
      @ProcessElement
      public void processElement(ProcessContext c) {
        c.output(c.element().length());
      }
    }));
```

```py
# The input PCollection of strings.
words = ...

# Apply a lambda function to the PCollection words.
# Save the result as the PCollection word_lengths.
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets_test.py tag:model_pardo_using_flatmap
%}
```

If your `ParDo` performs a one-to-one mapping of input elements to output
elements--that is, for each input element, it applies a function that produces
*exactly one* output element, you can use the higher-level
<span class="language-java">`MapElements`</span><span class="language-py">`Map`</span>
transform. <span class="language-java">`MapElements` can accept an anonymous
Java 8 lambda function for additional brevity.</span>

Here's the previous example using <span class="language-java">`MapElements`</span>
<span class="language-py">`Map`</span>:

```java
// The input PCollection.
PCollection<String> words = ...;

// Apply a MapElements with an anonymous lambda function to the PCollection words.
// Save the result as the PCollection wordLengths.
PCollection<Integer> wordLengths = words.apply(
  MapElements.into(TypeDescriptors.integers())
             .via((String word) -> word.length()));
```

```py
# The input PCollection of string.
words = ...

# Apply a Map with a lambda function to the PCollection words.
# Save the result as the PCollection word_lengths.
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets_test.py tag:model_pardo_using_map
%}
```

{:.language-java}
> **Note:** You can use Java 8 lambda functions with several other Beam
> transforms, including `Filter`, `FlatMapElements`, and `Partition`.

#### 4.2.2. GroupByKey

`GroupByKey` is a Beam transform for processing collections of key/value pairs.
It's a parallel reduction operation, analogous to the Shuffle phase of a
Map/Shuffle/Reduce-style algorithm. The input to `GroupByKey` is a collection of
key/value pairs that represents a *multimap*, where the collection contains
multiple pairs that have the same key, but different values. Given such a
collection, you use `GroupByKey` to collect all of the values associated with
each unique key.

`GroupByKey` is a good way to aggregate data that has something in common. For
example, if you have a collection that stores records of customer orders, you
might want to group together all the orders from the same postal code (wherein
the "key" of the key/value pair is the postal code field, and the "value" is the
remainder of the record).

Let's examine the mechanics of `GroupByKey` with a simple example case, where
our data set consists of words from a text file and the line number on which
they appear. We want to group together all the line numbers (values) that share
the same word (key), letting us see all the places in the text where a
particular word appears.

Our input is a `PCollection` of key/value pairs where each word is a key, and
the value is a line number in the file where the word appears. Here's a list of
the key/value pairs in the input collection:

```
cat, 1
dog, 5
and, 1
jump, 3
tree, 2
cat, 5
dog, 2
and, 2
cat, 9
and, 6
...
```

`GroupByKey` gathers up all the values with the same key and outputs a new pair
consisting of the unique key and a collection of all of the values that were
associated with that key in the input collection. If we apply `GroupByKey` to
our input collection above, the output collection would look like this:

```
cat, [1,5,9]
dog, [5,2]
and, [1,2,6]
jump, [3]
tree, [2]
...
```

Thus, `GroupByKey` represents a transform from a multimap (multiple keys to
individual values) to a uni-map (unique keys to collections of values).

#### 4.2.3. CoGroupByKey

`CoGroupByKey` joins two or more key/value `PCollection`s that have the same key
type, and then emits a collection of `KV<K, CoGbkResult>` pairs. [Design Your
Pipeline]({{ site.baseurl }}/documentation/pipelines/design-your-pipeline/#multiple-sources)
shows an example pipeline that uses a join.

Given the input collections below:
```
// collection 1
user1, address1
user2, address2
user3, address3

// collection 2
user1, order1
user1, order2
user2, order3
guest, order4
...
```

`CoGroupByKey` gathers up the values with the same key from all `PCollection`s,
and outputs a new pair consisting of the unique key and an object `CoGbkResult`
containing all values that were associated with that key. If you apply
`CoGroupByKey` to the input collections above, the output collection would look
like this:
```
user1, [[address1], [order1, order2]]
user2, [[address2], [order3]]
user3, [[address3], []]
guest, [[], [order4]]
...
````

> **A Note on Key/Value Pairs:** Beam represents key/value pairs slightly
> differently depending on the language and SDK you're using. In the Beam SDK
> for Java, you represent a key/value pair with an object of type `KV<K, V>`. In
> Python, you represent key/value pairs with 2-tuples.

#### 4.2.4. Combine

<span class="language-java">[`Combine`]({{ site.baseurl }}/documentation/sdks/javadoc/{{ site.release_latest }}/index.html?org/apache/beam/sdk/transforms/Combine.html)</span>
<span class="language-py">[`Combine`](https://github.com/apache/beam/blob/master/sdks/python/apache_beam/transforms/core.py)</span>
is a Beam transform for combining collections of elements or values in your
data. `Combine` has variants that work on entire `PCollection`s, and some that
combine the values for each key in `PCollection`s of key/value pairs.

When you apply a `Combine` transform, you must provide the function that
contains the logic for combining the elements or values. The combining function
should be commutative and associative, as the function is not necessarily
invoked exactly once on all values with a given key. Because the input data
(including the value collection) may be distributed across multiple workers, the
combining function might be called multiple times to perform partial combining
on subsets of the value collection. The Beam SDK also provides some pre-built
combine functions for common numeric combination operations such as sum, min,
and max.

Simple combine operations, such as sums, can usually be implemented as a simple
function. More complex combination operations might require you to create a
subclass of `CombineFn` that has an accumulation type distinct from the
input/output type.

##### 4.2.4.1. Simple combinations using simple functions

The following example code shows a simple combine function.

```java
// Sum a collection of Integer values. The function SumInts implements the interface SerializableFunction.
public static class SumInts implements SerializableFunction<Iterable<Integer>, Integer> {
  @Override
  public Integer apply(Iterable<Integer> input) {
    int sum = 0;
    for (int item : input) {
      sum += item;
    }
    return sum;
  }
}
```

```py
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets_test.py tag:combine_bounded_sum
%}
```

##### 4.2.4.2. Advanced combinations using CombineFn

For more complex combine functions, you can define a subclass of `CombineFn`.
You should use `CombineFn` if the combine function requires a more sophisticated
accumulator, must perform additional pre- or post-processing, might change the
output type, or takes the key into account.

A general combining operation consists of four operations. When you create a
subclass of `CombineFn`, you must provide four operations by overriding the
corresponding methods:

1. **Create Accumulator** creates a new "local" accumulator. In the example
   case, taking a mean average, a local accumulator tracks the running sum of
   values (the numerator value for our final average division) and the number of
   values summed so far (the denominator value). It may be called any number of
   times in a distributed fashion.

2. **Add Input** adds an input element to an accumulator, returning the
   accumulator value. In our example, it would update the sum and increment the
   count. It may also be invoked in parallel.

3. **Merge Accumulators** merges several accumulators into a single accumulator;
   this is how data in multiple accumulators is combined before the final
   calculation. In the case of the mean average computation, the accumulators
   representing each portion of the division are merged together. It may be
   called again on its outputs any number of times.

4. **Extract Output** performs the final computation. In the case of computing a
   mean average, this means dividing the combined sum of all the values by the
   number of values summed. It is called once on the final, merged accumulator.

The following example code shows how to define a `CombineFn` that computes a
mean average:

```java
public class AverageFn extends CombineFn<Integer, AverageFn.Accum, Double> {
  public static class Accum {
    int sum = 0;
    int count = 0;
  }

  @Override
  public Accum createAccumulator() { return new Accum(); }

  @Override
  public Accum addInput(Accum accum, Integer input) {
      accum.sum += input;
      accum.count++;
      return accum;
  }

  @Override
  public Accum mergeAccumulators(Iterable<Accum> accums) {
    Accum merged = createAccumulator();
    for (Accum accum : accums) {
      merged.sum += accum.sum;
      merged.count += accum.count;
    }
    return merged;
  }

  @Override
  public Double extractOutput(Accum accum) {
    return ((double) accum.sum) / accum.count;
  }
}
```
```py
pc = ...
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets_test.py tag:combine_custom_average_define
%}
```

If you are combining a `PCollection` of key-value pairs, [per-key
combining](#combining-values-in-a-keyed-pcollection) is often enough. If
you need the combining strategy to change based on the key (for example, MIN for
some users and MAX for other users), you can define a `KeyedCombineFn` to access
the key within the combining strategy.

##### 4.2.4.3. Combining a PCollection into a single value

Use the global combine to transform all of the elements in a given `PCollection`
into a single value, represented in your pipeline as a new `PCollection`
containing one element. The following example code shows how to apply the Beam
provided sum combine function to produce a single sum value for a `PCollection`
of integers.

```java
// Sum.SumIntegerFn() combines the elements in the input PCollection. The resulting PCollection, called sum,
// contains one value: the sum of all the elements in the input PCollection.
PCollection<Integer> pc = ...;
PCollection<Integer> sum = pc.apply(
   Combine.globally(new Sum.SumIntegerFn()));
```
```py
# sum combines the elements in the input PCollection.
# The resulting PCollection, called result, contains one value: the sum of all
# the elements in the input PCollection.
pc = ...
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets_test.py tag:combine_custom_average_execute
%}
```

##### 4.2.4.4. Combine and global windowing

If your input `PCollection` uses the default global windowing, the default
behavior is to return a `PCollection` containing one item. That item's value
comes from the accumulator in the combine function that you specified when
applying `Combine`. For example, the Beam provided sum combine function returns
a zero value (the sum of an empty input), while the max combine function returns
a maximal or infinite value.

To have `Combine` instead return an empty `PCollection` if the input is empty,
specify `.withoutDefaults` when you apply your `Combine` transform, as in the
following code example:

```java
PCollection<Integer> pc = ...;
PCollection<Integer> sum = pc.apply(
  Combine.globally(new Sum.SumIntegerFn()).withoutDefaults());
```
```py
pc = ...
sum = pc | beam.CombineGlobally(sum).without_defaults()
```

##### 4.2.4.5. Combine and non-global windowing

If your `PCollection` uses any non-global windowing function, Beam does not
provide the default behavior. You must specify one of the following options when
applying `Combine`:

* Specify `.withoutDefaults`, where windows that are empty in the input
  `PCollection` will likewise be empty in the output collection.
* Specify `.asSingletonView`, in which the output is immediately converted to a
  `PCollectionView`, which will provide a default value for each empty window
  when used as a side input. You'll generally only need to use this option if
  the result of your pipeline's `Combine` is to be used as a side input later in
  the pipeline.

##### 4.2.4.6. Combining values in a keyed PCollection

After creating a keyed PCollection (for example, by using a `GroupByKey`
transform), a common pattern is to combine the collection of values associated
with each key into a single, merged value. Drawing on the previous example from
`GroupByKey`, a key-grouped `PCollection` called `groupedWords` looks like this:
```
  cat, [1,5,9]
  dog, [5,2]
  and, [1,2,6]
  jump, [3]
  tree, [2]
  ...
```

In the above `PCollection`, each element has a string key (for example, "cat")
and an iterable of integers for its value (in the first element, containing [1,
5, 9]). If our pipeline's next processing step combines the values (rather than
considering them individually), you can combine the iterable of integers to
create a single, merged value to be paired with each key. This pattern of a
`GroupByKey` followed by merging the collection of values is equivalent to
Beam's Combine PerKey transform. The combine function you supply to Combine
PerKey must be an associative reduction function or a subclass of `CombineFn`.

```java
// PCollection is grouped by key and the Double values associated with each key are combined into a Double.
PCollection<KV<String, Double>> salesRecords = ...;
PCollection<KV<String, Double>> totalSalesPerPerson =
  salesRecords.apply(Combine.<String, Double, Double>perKey(
    new Sum.SumDoubleFn()));

// The combined value is of a different type than the original collection of values per key. PCollection has
// keys of type String and values of type Integer, and the combined value is a Double.
PCollection<KV<String, Integer>> playerAccuracy = ...;
PCollection<KV<String, Double>> avgAccuracyPerPlayer =
  playerAccuracy.apply(Combine.<String, Integer, Double>perKey(
    new MeanInts())));
```
```py
# PCollection is grouped by key and the numeric values associated with each key
# are averaged into a float.
player_accuracies = ...
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets_test.py tag:combine_per_key
%}
```

#### 4.2.5. Flatten

<span class="language-java">[`Flatten`]({{ site.baseurl }}/documentation/sdks/javadoc/{{ site.release_latest }}/index.html?org/apache/beam/sdk/transforms/Flatten.html)</span>
<span class="language-py">[`Flatten`](https://github.com/apache/beam/blob/master/sdks/python/apache_beam/transforms/core.py)</span> and
is a Beam transform for `PCollection` objects that store the same data type.
`Flatten` merges multiple `PCollection` objects into a single logical
`PCollection`.

The following example shows how to apply a `Flatten` transform to merge multiple
`PCollection` objects.

```java
// Flatten takes a PCollectionList of PCollection objects of a given type.
// Returns a single PCollection that contains all of the elements in the PCollection objects in that list.
PCollection<String> pc1 = ...;
PCollection<String> pc2 = ...;
PCollection<String> pc3 = ...;
PCollectionList<String> collections = PCollectionList.of(pc1).and(pc2).and(pc3);

PCollection<String> merged = collections.apply(Flatten.<String>pCollections());
```

```py
# Flatten takes a tuple of PCollection objects.
# Returns a single PCollection that contains all of the elements in the
{%
github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets.py tag:model_multiple_pcollections_flatten
%}
```

##### 4.2.5.1. Data encoding in merged collections

By default, the coder for the output `PCollection` is the same as the coder for
the first `PCollection` in the input `PCollectionList`. However, the input
`PCollection` objects can each use different coders, as long as they all contain
the same data type in your chosen language.

##### 4.2.5.2. Merging windowed collections

When using `Flatten` to merge `PCollection` objects that have a windowing
strategy applied, all of the `PCollection` objects you want to merge must use a
compatible windowing strategy and window sizing. For example, all the
collections you're merging must all use (hypothetically) identical 5-minute
fixed windows or 4-minute sliding windows starting every 30 seconds.

If your pipeline attempts to use `Flatten` to merge `PCollection` objects with
incompatible windows, Beam generates an `IllegalStateException` error when your
pipeline is constructed.

#### 4.2.6. Partition

<span class="language-java">[`Partition`]({{ site.baseurl }}/documentation/sdks/javadoc/{{ site.release_latest }}/index.html?org/apache/beam/sdk/transforms/Partition.html)</span>
<span class="language-py">[`Partition`](https://github.com/apache/beam/blob/master/sdks/python/apache_beam/transforms/core.py)</span>
is a Beam transform for `PCollection` objects that store the same data
type. `Partition` splits a single `PCollection` into a fixed number of smaller
collections.

`Partition` divides the elements of a `PCollection` according to a partitioning
function that you provide. The partitioning function contains the logic that
determines how to split up the elements of the input `PCollection` into each
resulting partition `PCollection`. The number of partitions must be determined
at graph construction time. You can, for example, pass the number of partitions
as a command-line option at runtime (which will then be used to build your
pipeline graph), but you cannot determine the number of partitions in
mid-pipeline (based on data calculated after your pipeline graph is constructed,
for instance).

The following example divides a `PCollection` into percentile groups.

```java
// Provide an int value with the desired number of result partitions, and a PartitionFn that represents the
// partitioning function. In this example, we define the PartitionFn in-line. Returns a PCollectionList
// containing each of the resulting partitions as individual PCollection objects.
PCollection<Student> students = ...;
// Split students up into 10 partitions, by percentile:
PCollectionList<Student> studentsByPercentile =
    students.apply(Partition.of(10, new PartitionFn<Student>() {
        public int partitionFor(Student student, int numPartitions) {
            return student.getPercentile()  // 0..99
                 * numPartitions / 100;
        }}));

// You can extract each partition from the PCollectionList using the get method, as follows:
PCollection<Student> fortiethPercentile = studentsByPercentile.get(4);
```
```py
# Provide an int value with the desired number of result partitions, and a partitioning function (partition_fn in this example).
# Returns a tuple of PCollection objects containing each of the resulting partitions as individual PCollection objects.
students = ...
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets.py tag:model_multiple_pcollections_partition
%}

# You can extract each partition from the tuple of PCollection objects as follows:
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets.py tag:model_multiple_pcollections_partition_40th
%}
```

### 4.3. Requirements for writing user code for Beam transforms

When you build user code for a Beam transform, you should keep in mind the
distributed nature of execution. For example, there might be many copies of your
function running on a lot of different machines in parallel, and those copies
function independently, without communicating or sharing state with any of the
other copies. Depending on the Pipeline Runner and processing back-end you
choose for your pipeline, each copy of your user code function may be retried or
run multiple times. As such, you should be cautious about including things like
state dependency in your user code.

In general, your user code must fulfill at least these requirements:

* Your function object must be **serializable**.
* Your function object must be **thread-compatible**, and be aware that *the
  Beam SDKs are not thread-safe*.

In addition, it's recommended that you make your function object **idempotent**.

> **Note:** These requirements apply to subclasses of `DoFn` (a function object
> used with the [ParDo](#pardo) transform), `CombineFn` (a function object used
> with the [Combine](#combine) transform), and `WindowFn` (a function object
> used with the [Window](#windowing) transform).

#### 4.3.1. Serializability

Any function object you provide to a transform must be **fully serializable**.
This is because a copy of the function needs to be serialized and transmitted to
a remote worker in your processing cluster. The base classes for user code, such
as `DoFn`, `CombineFn`, and `WindowFn`, already implement `Serializable`;
however, your subclass must not add any non-serializable members.

Some other serializability factors you should keep in mind are:

* Transient fields in your function object are *not* transmitted to worker
  instances, because they are not automatically serialized.
* Avoid loading a field with a large amount of data before serialization.
* Individual instances of your function object cannot share data.
* Mutating a function object after it gets applied will have no effect.
* Take care when declaring your function object inline by using an anonymous
  inner class instance. In a non-static context, your inner class instance will
  implicitly contain a pointer to the enclosing class and that class' state.
  That enclosing class will also be serialized, and thus the same considerations
  that apply to the function object itself also apply to this outer class.

#### 4.3.2. Thread-compatibility

Your function object should be thread-compatible. Each instance of your function
object is accessed by a single thread on a worker instance, unless you
explicitly create your own threads. Note, however, that **the Beam SDKs are not
thread-safe**. If you create your own threads in your user code, you must
provide your own synchronization. Note that static members in your function
object are not passed to worker instances and that multiple instances of your
function may be accessed from different threads.

#### 4.3.3. Idempotence

It's recommended that you make your function object idempotent--that is, that it
can be repeated or retried as often as necessary without causing unintended side
effects. The Beam model provides no guarantees as to the number of times your
user code might be invoked or retried; as such, keeping your function object
idempotent keeps your pipeline's output deterministic, and your transforms'
behavior more predictable and easier to debug.

### 4.4. Side inputs

In addition to the main input `PCollection`, you can provide additional inputs
to a `ParDo` transform in the form of side inputs. A side input is an additional
input that your `DoFn` can access each time it processes an element in the input
`PCollection`. When you specify a side input, you create a view of some other
data that can be read from within the `ParDo` transform's `DoFn` while procesing
each element.

Side inputs are useful if your `ParDo` needs to inject additional data when
processing each element in the input `PCollection`, but the additional data
needs to be determined at runtime (and not hard-coded). Such values might be
determined by the input data, or depend on a different branch of your pipeline.


#### 4.4.1. Passing side inputs to ParDo

```java
  // Pass side inputs to your ParDo transform by invoking .withSideInputs.
  // Inside your DoFn, access the side input by using the method DoFn.ProcessContext.sideInput.

  // The input PCollection to ParDo.
  PCollection<String> words = ...;

  // A PCollection of word lengths that we'll combine into a single value.
  PCollection<Integer> wordLengths = ...; // Singleton PCollection

  // Create a singleton PCollectionView from wordLengths using Combine.globally and View.asSingleton.
  final PCollectionView<Integer> maxWordLengthCutOffView =
     wordLengths.apply(Combine.globally(new Max.MaxIntFn()).asSingletonView());


  // Apply a ParDo that takes maxWordLengthCutOffView as a side input.
  PCollection<String> wordsBelowCutOff =
  words.apply(ParDo
      .of(new DoFn<String, String>() {
          public void processElement(ProcessContext c) {
            String word = c.element();
            // In our DoFn, access the side input.
            int lengthCutOff = c.sideInput(maxWordLengthCutOffView);
            if (word.length() <= lengthCutOff) {
              c.output(word);
            }
          }
      }).withSideInputs(maxWordLengthCutOffView)
  );
```
```py
# Side inputs are available as extra arguments in the DoFn's process method or Map / FlatMap's callable.
# Optional, positional, and keyword arguments are all supported. Deferred arguments are unwrapped into their
# actual values. For example, using pvalue.AsIteor(pcoll) at pipeline construction time results in an iterable
# of the actual elements of pcoll being passed into each process invocation. In this example, side inputs are
# passed to a FlatMap transform as extra arguments and consumed by filter_using_length.
words = ...
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets_test.py tag:model_pardo_side_input
%}

# We can also pass side inputs to a ParDo transform, which will get passed to its process method.
# The first two arguments for the process method would be self and element.

{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets_test.py tag:model_pardo_side_input_dofn
%}
...
```

#### 4.4.2. Side inputs and windowing

A windowed `PCollection` may be infinite and thus cannot be compressed into a
single value (or single collection class). When you create a `PCollectionView`
of a windowed `PCollection`, the `PCollectionView` represents a single entity
per window (one singleton per window, one list per window, etc.).

Beam uses the window(s) for the main input element to look up the appropriate
window for the side input element. Beam projects the main input element's window
into the side input's window set, and then uses the side input from the
resulting window. If the main input and side inputs have identical windows, the
projection provides the exact corresponding window. However, if the inputs have
different windows, Beam uses the projection to choose the most appropriate side
input window.

For example, if the main input is windowed using fixed-time windows of one
minute, and the side input is windowed using fixed-time windows of one hour,
Beam projects the main input window against the side input window set and
selects the side input value from the appropriate hour-long side input window.

If the main input element exists in more than one window, then `processElement`
gets called multiple times, once for each window. Each call to `processElement`
projects the "current" window for the main input element, and thus might provide
a different view of the side input each time.

If the side input has multiple trigger firings, Beam uses the value from the
latest trigger firing. This is particularly useful if you use a side input with
a single global window and specify a trigger.

### 4.5. Additional outputs

While `ParDo` always produces a main output `PCollection` (as the return value
from `apply`), you can also have your `ParDo` produce any number of additional
output `PCollection`s. If you choose to have multiple outputs, your `ParDo`
returns all of the output `PCollection`s (including the main output) bundled
together.

#### 4.5.1. Tags for multiple outputs

```java
// To emit elements to multiple output PCollections, create a TupleTag object to identify each collection
// that your ParDo produces. For example, if your ParDo produces three output PCollections (the main output
// and two additional outputs), you must create three TupleTags. The following example code shows how to
// create TupleTags for a ParDo with three output PCollections.

  // Input PCollection to our ParDo.
  PCollection<String> words = ...;

  // The ParDo will filter words whose length is below a cutoff and add them to
  // the main ouput PCollection<String>.
  // If a word is above the cutoff, the ParDo will add the word length to an
  // output PCollection<Integer>.
  // If a word starts with the string "MARKER", the ParDo will add that word to an
  // output PCollection<String>.
  final int wordLengthCutOff = 10;

  // Create three TupleTags, one for each output PCollection.
  // Output that contains words below the length cutoff.
  final TupleTag<String> wordsBelowCutOffTag =
      new TupleTag<String>(){};
  // Output that contains word lengths.
  final TupleTag<Integer> wordLengthsAboveCutOffTag =
      new TupleTag<Integer>(){};
  // Output that contains "MARKER" words.
  final TupleTag<String> markedWordsTag =
      new TupleTag<String>(){};

// Passing Output Tags to ParDo:
// After you specify the TupleTags for each of your ParDo outputs, pass the tags to your ParDo by invoking
// .withOutputTags. You pass the tag for the main output first, and then the tags for any additional outputs
// in a TupleTagList. Building on our previous example, we pass the three TupleTags for our three output
// PCollections to our ParDo. Note that all of the outputs (including the main output PCollection) are
// bundled into the returned PCollectionTuple.

  PCollectionTuple results =
      words.apply(ParDo
          .of(new DoFn<String, String>() {
            // DoFn continues here.
            ...
          })
          // Specify the tag for the main output.
          .withOutputTags(wordsBelowCutOffTag,
          // Specify the tags for the two additional outputs as a TupleTagList.
                          TupleTagList.of(wordLengthsAboveCutOffTag)
                                      .and(markedWordsTag)));
```

```py
# To emit elements to multiple output PCollections, invoke with_outputs() on the ParDo, and specify the
# expected tags for the outputs. with_outputs() returns a DoOutputsTuple object. Tags specified in
# with_outputs are attributes on the returned DoOutputsTuple object. The tags give access to the
# corresponding output PCollections.

{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets_test.py tag:model_pardo_with_tagged_outputs
%}

# The result is also iterable, ordered in the same order that the tags were passed to with_outputs(),
# the main tag (if specified) first.

{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets_test.py tag:model_pardo_with_tagged_outputs_iter
%}
```

#### 4.5.2. Emitting to multiple outputs in your DoFn

```java
// Inside your ParDo's DoFn, you can emit an element to a specific output PCollection by passing in the
// appropriate TupleTag when you call ProcessContext.output.
// After your ParDo, extract the resulting output PCollections from the returned PCollectionTuple.
// Based on the previous example, this shows the DoFn emitting to the main output and two additional outputs.

  .of(new DoFn<String, String>() {
     public void processElement(ProcessContext c) {
       String word = c.element();
       if (word.length() <= wordLengthCutOff) {
         // Emit short word to the main output.
         // In this example, it is the output with tag wordsBelowCutOffTag.
         c.output(word);
       } else {
         // Emit long word length to the output with tag wordLengthsAboveCutOffTag.
         c.output(wordLengthsAboveCutOffTag, word.length());
       }
       if (word.startsWith("MARKER")) {
         // Emit word to the output with tag markedWordsTag.
         c.output(markedWordsTag, word);
       }
     }}));
```

```py
# Inside your ParDo's DoFn, you can emit an element to a specific output by wrapping the value and the output tag (str).
# using the pvalue.OutputValue wrapper class.
# Based on the previous example, this shows the DoFn emitting to the main output and two additional outputs.

{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets_test.py tag:model_pardo_emitting_values_on_tagged_outputs
%}

# Producing multiple outputs is also available in Map and FlatMap.
# Here is an example that uses FlatMap and shows that the tags do not need to be specified ahead of time.

{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets_test.py tag:model_pardo_with_undeclared_outputs
%}
```

### 4.6. Composite transforms

Transforms can have a nested structure, where a complex transform performs
multiple simpler transforms (such as more than one `ParDo`, `Combine`,
`GroupByKey`, or even other composite transforms). These transforms are called
composite transforms. Nesting multiple transforms inside a single composite
transform can make your code more modular and easier to understand.

The Beam SDK comes packed with many useful composite transforms. See the API
reference pages for a list of transforms:
  * [Pre-written Beam transforms for Java]({{ site.baseurl }}/documentation/sdks/javadoc/{{ site.release_latest }}/index.html?org/apache/beam/sdk/transforms/package-summary.html)
  * [Pre-written Beam transforms for Python]({{ site.baseurl }}/documentation/sdks/pydoc/{{ site.release_latest }}/apache_beam.transforms.html)

#### 4.6.1. An example composite transform

The `CountWords` transform in the [WordCount example program]({{ site.baseurl }}/get-started/wordcount-example/)
is an example of a composite transform. `CountWords` is a `PTransform` subclass
that consists of multiple nested transforms.

In its `expand` method, the `CountWords` transform applies the following
transform operations:

  1. It applies a `ParDo` on the input `PCollection` of text lines, producing
     an output `PCollection` of individual words.
  2. It applies the Beam SDK library transform `Count` on the `PCollection` of
     words, producing a `PCollection` of key/value pairs. Each key represents a
     word in the text, and each value represents the number of times that word
     appeared in the original data.

Note that this is also an example of nested composite transforms, as `Count`
is, by itself, a composite transform.

Your composite transform's parameters and return value must match the initial
input type and final return type for the entire transform, even if the
transform's intermediate data changes type multiple times.

```java
  public static class CountWords extends PTransform<PCollection<String>,
      PCollection<KV<String, Long>>> {
    @Override
    public PCollection<KV<String, Long>> expand(PCollection<String> lines) {

      // Convert lines of text into individual words.
      PCollection<String> words = lines.apply(
          ParDo.of(new ExtractWordsFn()));

      // Count the number of times each word occurs.
      PCollection<KV<String, Long>> wordCounts =
          words.apply(Count.<String>perElement());

      return wordCounts;
    }
  }
```

```py
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets.py tag:pipeline_monitoring_composite
%}
```

#### 4.6.2. Creating a composite transform

To create your own composite transform, create a subclass of the `PTransform`
class and override the `expand` method to specify the actual processing logic.
You can then use this transform just as you would a built-in transform from the
Beam SDK.

{:.language-java}
For the `PTransform` class type parameters, you pass the `PCollection` types
that your transform takes as input, and produces as output. To take multiple
`PCollection`s as input, or produce multiple `PCollection`s as output, use one
of the multi-collection types for the relevant type parameter.

The following code sample shows how to declare a `PTransform` that accepts a
`PCollection` of `String`s for input, and outputs a `PCollection` of `Integer`s:

```java
  static class ComputeWordLengths
    extends PTransform<PCollection<String>, PCollection<Integer>> {
    ...
  }
```

```py
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets_test.py tag:model_composite_transform
%}
```

Within your `PTransform` subclass, you'll need to override the `expand` method.
The `expand` method is where you add the processing logic for the `PTransform`.
Your override of `expand` must accept the appropriate type of input
`PCollection` as a parameter, and specify the output `PCollection` as the return
value.

The following code sample shows how to override `expand` for the
`ComputeWordLengths` class declared in the previous example:

```java
  static class ComputeWordLengths
      extends PTransform<PCollection<String>, PCollection<Integer>> {
    @Override
    public PCollection<Integer> expand(PCollection<String>) {
      ...
      // transform logic goes here
      ...
    }
```

```py
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets_test.py tag:model_composite_transform
%}
```

As long as you override the `expand` method in your `PTransform` subclass to
accept the appropriate input `PCollection`(s) and return the corresponding
output `PCollection`(s), you can include as many transforms as you want. These
transforms can include core transforms, composite transforms, or the transforms
included in the Beam SDK libraries.

**Note:** The `expand` method of a `PTransform` is not meant to be invoked
directly by the user of a transform. Instead, you should call the `apply` method
on the `PCollection` itself, with the transform as an argument. This allows
transforms to be nested within the structure of your pipeline.

#### 4.6.3. PTransform Style Guide

The [PTransform Style Guide]({{ site.baseurl }}/contribute/ptransform-style-guide/)
contains additional information not included here, such as style guidelines,
logging and testing guidance, and language-specific considerations.  The guide
is a useful starting point when you want to write new composite PTransforms.

## 5. Pipeline I/O

When you create a pipeline, you often need to read data from some external
source, such as a file in external data sink or a database. Likewise, you may
want your pipeline to output its result data to a similar external data sink.
Beam provides read and write transforms for a [number of common data storage
types]({{ site.baseurl }}/documentation/io/built-in/). If you want your pipeline
to read from or write to a data storage format that isn't supported by the
built-in transforms, you can [implement your own read and write
transforms]({{site.baseurl }}/documentation/io/io-toc/).

### 5.1. Reading input data

Read transforms read data from an external source and return a `PCollection`
representation of the data for use by your pipeline. You can use a read
transform at any point while constructing your pipeline to create a new
`PCollection`, though it will be most common at the start of your pipeline.

```java
PCollection<String> lines = p.apply(TextIO.read().from("gs://some/inputData.txt"));
```

```py
lines = pipeline | beam.io.ReadFromText('gs://some/inputData.txt')
```

### 5.2. Writing output data

Write transforms write the data in a `PCollection` to an external data source.
You will most often use write transforms at the end of your pipeline to output
your pipeline's final results. However, you can use a write transform to output
a `PCollection`'s data at any point in your pipeline.

```java
output.apply(TextIO.write().to("gs://some/outputData"));
```

```py
output | beam.io.WriteToText('gs://some/outputData')
```

### 5.3. File-based input and output data

#### 5.3.1. Reading from multiple locations

Many read transforms support reading from multiple input files matching a glob
operator you provide. Note that glob operators are filesystem-specific and obey
filesystem-specific consistency models. The following TextIO example uses a glob
operator (\*) to read all matching input files that have prefix "input-" and the
suffix ".csv" in the given location:

```java
p.apply(“ReadFromText”,
    TextIO.read().from("protocol://my_bucket/path/to/input-*.csv");
```

```py
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets.py tag:model_pipelineio_read
%}
```

To read data from disparate sources into a single `PCollection`, read each one
independently and then use the [Flatten](#flatten) transform to create a single
`PCollection`.

#### 5.3.2. Writing to multiple output files

For file-based output data, write transforms write to multiple output files by
default. When you pass an output file name to a write transform, the file name
is used as the prefix for all output files that the write transform produces.
You can append a suffix to each output file by specifying a suffix.

The following write transform example writes multiple output files to a
location. Each file has the prefix "numbers", a numeric tag, and the suffix
".csv".

```java
records.apply("WriteToText",
    TextIO.write().to("protocol://my_bucket/path/to/numbers")
                .withSuffix(".csv"));
```

```py
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets.py tag:model_pipelineio_write
%}
```

### 5.4. Beam-provided I/O transforms

See the [Beam-provided I/O Transforms]({{site.baseurl }}/documentation/io/built-in/)
page for a list of the currently available I/O transforms.

## 6. Data encoding and type safety

When Beam runners execute your pipeline, they often need to materialize the
intermediate data in your `PCollection`s, which requires converting elements to
and from byte strings. The Beam SDKs use objects called `Coder`s to describe how
the elements of a given `PCollection` may be encoded and decoded.

> Note that coders are unrelated to parsing or formatting data when interacting
> with external data sources or sinks. Such parsing or formatting should
> typically be done explicitly, using transforms such as `ParDo` or
> `MapElements`.

{:.language-java}
In the Beam SDK for Java, the type `Coder` provides the methods required for
encoding and decoding data. The SDK for Java provides a number of Coder
subclasses that work with a variety of standard Java types, such as Integer,
Long, Double, StringUtf8 and more. You can find all of the available Coder
subclasses in the [Coder package](https://github.com/apache/beam/tree/master/sdks/java/core/src/main/java/org/apache/beam/sdk/coders).

{:.language-py}
In the Beam SDK for Python, the type `Coder` provides the methods required for
encoding and decoding data. The SDK for Python provides a number of Coder
subclasses that work with a variety of standard Python types, such as primitive
types, Tuple, Iterable, StringUtf8 and more. You can find all of the available
Coder subclasses in the
[apache_beam.coders](https://github.com/apache/beam/tree/master/sdks/python/apache_beam/coders)
package.

> Note that coders do not necessarily have a 1:1 relationship with types. For
> example, the Integer type can have multiple valid coders, and input and output
> data can use different Integer coders. A transform might have Integer-typed
> input data that uses BigEndianIntegerCoder, and Integer-typed output data that
> uses VarIntCoder.

### 6.1. Specifying coders

The Beam SDKs require a coder for every `PCollection` in your pipeline. In most
cases, the Beam SDK is able to automatically infer a `Coder` for a `PCollection`
based on its element type or the transform that produces it, however, in some
cases the pipeline author will need to specify a `Coder` explicitly, or develop
a `Coder` for their custom type.

{:.language-java}
You can explicitly set the coder for an existing `PCollection` by using the
method `PCollection.setCoder`. Note that you cannot call `setCoder` on a
`PCollection` that has been finalized (e.g. by calling `.apply` on it).

{:.language-java}
You can get the coder for an existing `PCollection` by using the method
`getCoder`. This method will fail with an `IllegalStateException` if a coder has
not been set and cannot be inferred for the given `PCollection`.

Beam SDKs use a variety of mechanisms when attempting to automatically infer the
`Coder` for a `PCollection`.

{:.language-java}
Each pipeline object has a `CoderRegistry`. The `CoderRegistry` represents a
mapping of Java types to the default coders that the pipeline should use for
`PCollection`s of each type.

{:.language-py}
The Beam SDK for Python has a `CoderRegistry` that represents a mapping of
Python types to the default coder that should be used for `PCollection`s of each
type.

{:.language-java}
By default, the Beam SDK for Java automatically infers the `Coder` for the
elements of a `PCollection` produced by a `PTransform` using the type parameter
from the transform's function object, such as `DoFn`. In the case of `ParDo`,
for example, a `DoFn<Integer, String>` function object accepts an input element
of type `Integer` and produces an output element of type `String`. In such a
case, the SDK for Java will automatically infer the default `Coder` for the
output `PCollection<String>` (in the default pipeline `CoderRegistry`, this is
`StringUtf8Coder`).

{:.language-py}
By default, the Beam SDK for Python automatically infers the `Coder` for the
elements of an output `PCollection` using the typehints from the transform's
function object, such as `DoFn`. In the case of `ParDo`, for example a `DoFn`
with the typehints `@beam.typehints.with_input_types(int)` and
`@beam.typehints.with_output_types(str)` accepts an input element of type int
and produces an output element of type str. In such a case, the Beam SDK for
Python will automatically infer the default `Coder` for the output `PCollection`
(in the default pipeline `CoderRegistry`, this is `BytesCoder`).

> NOTE: If you create your `PCollection` from in-memory data by using the
> `Create` transform, you cannot rely on coder inference and default coders.
> `Create` does not have access to any typing information for its arguments, and
> may not be able to infer a coder if the argument list contains a value whose
> exact run-time class doesn't have a default coder registered.

{:.language-java}
When using `Create`, the simplest way to ensure that you have the correct coder
is by invoking `withCoder` when you apply the `Create` transform.

### 6.2. Default coders and the CoderRegistry

Each Pipeline object has a `CoderRegistry` object, which maps language types to
the default coder the pipeline should use for those types. You can use the
`CoderRegistry` yourself to look up the default coder for a given type, or to
register a new default coder for a given type.

`CoderRegistry` contains a default mapping of coders to standard
<span class="language-java">Java</span><span class="language-py">Python</span>
types for any pipeline you create using the Beam SDK for
<span class="language-java">Java</span><span class="language-py">Python</span>.
The following table shows the standard mapping:

{:.language-java}
<table>
  <thead>
    <tr class="header">
      <th>Java Type</th>
      <th>Default Coder</th>
    </tr>
  </thead>
  <tbody>
    <tr class="odd">
      <td>Double</td>
      <td>DoubleCoder</td>
    </tr>
    <tr class="even">
      <td>Instant</td>
      <td>InstantCoder</td>
    </tr>
    <tr class="odd">
      <td>Integer</td>
      <td>VarIntCoder</td>
    </tr>
    <tr class="even">
      <td>Iterable</td>
      <td>IterableCoder</td>
    </tr>
    <tr class="odd">
      <td>KV</td>
      <td>KvCoder</td>
    </tr>
    <tr class="even">
      <td>List</td>
      <td>ListCoder</td>
    </tr>
    <tr class="odd">
      <td>Map</td>
      <td>MapCoder</td>
    </tr>
    <tr class="even">
      <td>Long</td>
      <td>VarLongCoder</td>
    </tr>
    <tr class="odd">
      <td>String</td>
      <td>StringUtf8Coder</td>
    </tr>
    <tr class="even">
      <td>TableRow</td>
      <td>TableRowJsonCoder</td>
    </tr>
    <tr class="odd">
      <td>Void</td>
      <td>VoidCoder</td>
    </tr>
    <tr class="even">
      <td>byte[ ]</td>
      <td>ByteArrayCoder</td>
    </tr>
    <tr class="odd">
      <td>TimestampedValue</td>
      <td>TimestampedValueCoder</td>
    </tr>
  </tbody>
</table>

{:.language-py}
<table>
  <thead>
    <tr class="header">
      <th>Python Type</th>
      <th>Default Coder</th>
    </tr>
  </thead>
  <tbody>
    <tr class="odd">
      <td>int</td>
      <td>VarIntCoder</td>
    </tr>
    <tr class="even">
      <td>float</td>
      <td>FloatCoder</td>
    </tr>
    <tr class="odd">
      <td>str</td>
      <td>BytesCoder</td>
    </tr>
    <tr class="even">
      <td>bytes</td>
      <td>StrUtf8Coder</td>
    </tr>
    <tr class="odd">
      <td>Tuple</td>
      <td>TupleCoder</td>
    </tr>
  </tbody>
</table>

#### 6.2.1. Looking up a default coder

{:.language-java}
You can use the method `CoderRegistry.getDefaultCoder` to determine the default
Coder for a Java type. You can access the `CoderRegistry` for a given pipeline
by using the method `Pipeline.getCoderRegistry`. This allows you to determine
(or set) the default Coder for a Java type on a per-pipeline basis: i.e. "for
this pipeline, verify that Integer values are encoded using
`BigEndianIntegerCoder`."

{:.language-py}
You can use the method `CoderRegistry.get_coder` to determine the default Coder
for a Python type. You can use `coders.registry` to access the `CoderRegistry`.
This allows you to determine (or set) the default Coder for a Python type.

#### 6.2.2. Setting the default coder for a type

To set the default Coder for a
<span class="language-java">Java</span><span class="language-py">Python</span>
type for a particular pipeline, you obtain and modify the pipeline's
`CoderRegistry`. You use the method
<span class="language-java">`Pipeline.getCoderRegistry`</span>
<span class="language-py">`coders.registry`</span>
to get the `CoderRegistry` object, and then use the method
<span class="language-java">`CoderRegistry.registerCoder`</span>
<span class="language-py">`CoderRegistry.register_coder`</span>
to register a new `Coder` for the target type.

The following example code demonstrates how to set a default Coder, in this case
`BigEndianIntegerCoder`, for
<span class="language-java">Integer</span><span class="language-py">int</span>
values for a pipeline.

```java
PipelineOptions options = PipelineOptionsFactory.create();
Pipeline p = Pipeline.create(options);

CoderRegistry cr = p.getCoderRegistry();
cr.registerCoder(Integer.class, BigEndianIntegerCoder.class);
```

```py
apache_beam.coders.registry.register_coder(int, BigEndianIntegerCoder)
```

#### 6.2.3. Annotating a custom data type with a default coder

{:.language-java}
If your pipeline program defines a custom data type, you can use the
`@DefaultCoder` annotation to specify the coder to use with that type. For
example, let's say you have a custom data type for which you want to use
`SerializableCoder`. You can use the `@DefaultCoder` annotation as follows:

```java
@DefaultCoder(AvroCoder.class)
public class MyCustomDataType {
  ...
}
```

{:.language-java}
If you've created a custom coder to match your data type, and you want to use
the `@DefaultCoder` annotation, your coder class must implement a static
`Coder.of(Class<T>)` factory method.

```java
public class MyCustomCoder implements Coder {
  public static Coder<T> of(Class<T> clazz) {...}
  ...
}

@DefaultCoder(MyCustomCoder.class)
public class MyCustomDataType {
  ...
}
```

{:.language-py}
The Beam SDK for Python does not support annotating data types with a default
coder. If you would like to set a default coder, use the method described in the
previous section, *Setting the default coder for a type*.

## 7. Windowing

窗口根据单个元素的时间戳来细分一个` pcollection `。转化即是聚合多样元素，隐式的作用于每一个基础窗口，例如方法` groupbykey `和`Combine`，－它们将每个PCollection处理成多个、有限的窗口，尽管整个集合本身可能是无界的。

一个相关的概念，称为触发器，决定何时计算聚合的结果，而这些数据是无界数据。您可以使用触发器来完善您的PCollection的窗口策略。触发器允许您处理延延迟达的数据或提供早期结果。有关更多信息，请参见触发器部分。

### 7.1. Windowing 基础

一些Beam转化，例如GroupByKey和Combine，通过一个通用的键来分组多个元素。通常，分组操作是将所有在整个数据集中具有相同键的元素分组。对于无界数据集，不可能收集所有的元素，因为新元素不断被添加，并且可能是无限的(例如，流数据)。‘如果您使用的是无界的PCollection，则窗口尤其有用。

在Beam模型中，任何PCollection(包括无界的PCollection)都可以被划分为逻辑窗口。PCollection中的每个元素都根据PCollection的窗口函数分配给一个或多个窗口，每个单独的窗口都包含一个有限数量的元素。分组转化然后在每个窗口的基础上考虑每个PCollection的元素。例如，GroupByKey通过键和窗口来隐式地将PCollection的元素分组。

**注意:** Beam的默认窗口行为是将一个PCollection的所有元素分配到一个全局窗口中，并丢弃最近的数据，即使是无界的PCollection。在对无界的PCollection使用如GroupByKey等分组转化之前，，您必须至少做以下的一个:
* 设置一个非全局的窗口功能。查看设置您的PCollection的窗口函数(设置-您的pcollections-windowing-function)。
* 设置一个非默认触发器(触发器)。这允许全局窗口在其他条件下发出结果，因为默认的窗口行为(等待所有数据到达)将永远不会发生。

如果您没有为您的无界PCollection设置非全局窗口函数或非默认触发器，并随后使用诸如GroupByKey或组合之类的分组转化，那么您的管道将在构建过程中产生错误，您的job将失败。

#### 7.1.1. Windowing 约束

在为PCollection设置了窗口函数之后，在下一次将分组转化应用到PCollection时，该窗口才起作用。
窗口分组发生在需要的基础上。如果您使用窗口转化设置一个窗口函数，那么每个元素都被分配到一个窗口，但是窗口不会被考虑到GroupByKey，或者在窗口和键之间合并聚合。这可能对您的管道有不同的影响。  
考虑下面图中的示例管道:
[![管道应用窗口图]({{ "/images/windowing-pipeline-unbounded.png" | prepend: site.baseurl }} "Pipeline applying windowing")

**Figure:** Pipeline 应用窗口

在上面的管道中，我们通过使用`KafkaIO`来读取一组键/值对来创建一个无界的PCollection，然后使用窗口转化向该集合应用一个窗口函数。
之后，我们将一个ParDo应用到集合中，并使用GroupByKey对ParDo的结果进行分组。
窗口函数对ParDo转化没有影响，因为在GroupByKey需要它们之前，窗口实际上不会被使用。
然而，随后应用了GroupByKey分组的转化结果——数据按键和窗口分组。

#### 7.1.2. 有界窗口应用

您可以在有界的PCollection中使用固定大小的数据集。
但是，请注意，窗口只考虑与PCollection的每个元素相关联的内隐时间戳，而创建固定数据集(例如TextIO)的数据源将相同的时间戳分配给每个元素。这意味着所有的元素都默认是一个全局窗口的默认部分。

为了使用固定数据集的窗口，您可以将自己的时间戳分配给每个元素。使用带`DoFn`的`ParDo`转化,将为每个元素输出一个新的时间戳(例如,在Beam SDK for java中[时间戳]({{ site.baseurl }}/documentation/sdks/javadoc/{{ site.release_latest}}/index.html?org/apache/beam/sdk/transforms/WithTimestamps.html)转化)。

演示如何使用有界的PCollection来影响您的操作
管道处理数据，考虑以下管道:
![无窗口的`GroupByKey`和`ParDo`应用到有界的集合上]({{ "/images/unwindowed-pipeline-bounded.png" | prepend: site.baseurl }} "GroupByKey and ParDo without windowing, on a bounded collection")

**Figure:** 有界集上应用无窗口的`GroupByKey` 与 `ParDo`

在上面的管道中，我们通过使用TextIO读取一组键/值对来创建一个有界的PCollection。然后，我们使用GroupByKey对集合进行分组，并将ParDo转化应用到分组的PCollection。在本例中，GroupByKey创建了一个惟一键的集合，然后ParDo对每一个键应用一次。

请注意，即使您没有设置一个窗口函数，仍然有一个窗口—您的PCollection中的所有元素都被分配给一个全局窗口。
现在，考虑相同的管道，但是使用一个窗口函数:

![带窗口的`GroupByKey`和`ParDo`应用到有界的集合上]({{ "/images/windowing-pipeline-bounded.png" | prepend: site.baseurl }} "带窗口的`GroupByKey`和`ParDo`应用到有界的集合上")

**Figure:** 带窗口的`GroupByKey`和`ParDo`应用到有界的集合上

与前面一样，管道创建了一个有界的键/值对集合。然后，我们为该PCollection设置了一个窗口函数(设置您的Pcollections-windowing-function)。根据窗口的功能对PCollection的元素进行GroupByKey分组转化。随后对每个窗口的每个键多次应用ParDo转化。

### 7.2. 预设 windowing functions

您可以定义不同类型的窗口来划分您的PCollection元素。Beam提供了几个窗口功能，包括:

*  固定时间窗口
*  滑动时间窗口
*  会话窗口
*  全局窗口
*  基于日历的窗户(暂不支持Python)

如果您有更复杂的需求，您也可以定义自己的`windowsFn`。   
注意，每个元素逻辑上可以属于多个窗口，这取决于你使用的窗口函数。例如，滑动时间窗口会创建重叠的窗口，其中一个元素可以分配给多个窗口。

#### 7.2.1. 固定时间窗口（Fixed time windows）

最简单的窗口形式是使用**固定时间窗口**:给定一个时间戳的`PCollection`，它可能会不断地更新，每个窗口可能会捕获到(例如)所有带有时间戳在时间间隔为5分钟时间窗口内的所有元素。

固定时间窗口表示数据流中不重叠的时间间隔。考虑以5分钟时间间隔的windows:在您的无界`PCollection`中所有的元素中，从0:00:00到(但不包括)0:05:00属于第一个窗口，从0:05:00到(但不包括)0:10的时间间隔内的元素属于第二个窗口，以此类推。

![图：时间间隔为30s的固定时间窗口]({{ "/images/fixed-time-windows.png" | prepend: site.baseurl }} "Fixed time windows, 30s in duration")

**图:** 固定时间窗口, 时间间隔30s.

#### 7.2.2. 滑动时间窗口（Sliding time windows）

一个**滑动时间窗**口也表示数据流中的时间间隔;然而，滑动时间窗口可以重叠。例如，每个窗口可能捕获5分钟的数据，但是每隔10秒就会启动一个新窗口。
滑动窗口开始的频率称为周期。
因此，我们的示例是一个窗口持续时间为5分钟和周期为10秒的滑动窗口。

由于多个窗口重叠，数据集中的大多数元素将属于多个窗口。这种类型的窗口对于获取运行数据的平均值是很有用的;在我们的示例中使用滑动时间窗口，您可以计算过去5分钟数据的运行平均值，每10秒更新一次。

![图：华东时间窗口, 时间间隔1min,周期30s]({{ "/images/sliding-time-windows.png" | prepend: site.baseurl }} "Sliding time windows, with 1 minute window duration and 30s window period")

**图:** 华东时间窗口, 时间间隔1min,周期30s.

#### 7.2.3. 会话窗口（Session windows）

一个**会话窗口**定义了包含在另一个元素的某个间隙时间内的元素的窗口。会话窗口应用于每一个基础键上，对于不定期分发的数据非常有用。例如，代表用户鼠标活动的数据流可能有很长一段时间的空闲时间，其中穿插了大量的点击。
如果数据在最小指定的间隔时间之后到达，这将启动一个新窗口的开始。

![时间间隔一分钟的会话窗口]({{ "/images/session-windows.png" | prepend: site.baseurl }} "Session windows, with a minimum gap duration")

**图:** 会话窗口，时间间隔1min. 注意，根据其数据分布，每个数据键都有不同的窗口。

#### 7.2.4. 全局窗口（The single global window）

默认情况下，`PCollection`中的所有数据都被分配给单个全局窗口，而延迟数据将被丢弃。    
如果数据集是固定大小的，那么您可以使用全局窗口缺省值来进行`PCollection`。
如果您使用的是一个无界的数据集(例如来自流数据源)，那么您可以使用单全局窗口，但是在应用`GroupByKey`和`Combine`等聚合转化时要特别谨慎。     
带有默认触发器的单一全局窗口通常要求在处理前全部数据集不可能持续更新数据的可用数据集。     
在无界的`PCollection`上使用全局窗口执行聚合转化时，您应该为该`PCollection`指定一个非默认触发器。

### 7.3. 设置 PCollection's windowing function

您可以通过应用窗口转化为`PCollection`设置窗口函数。当您应用窗口转化时，您必须提供一个窗口`fn`。
`windows fn`决定了您的`PCollection`将用于后续分组转化的窗口函数，例如固定的或滑动的时间窗口。   
当您设置一个窗口函数时，您可能还想为您的`PCollection`设置一个触发器。触发器决定了每个单独的窗口被聚合和释放的时间，并帮助改进窗口在计算较晚的数据和计算早期结果中的执行性能。有关更多信息，请参阅触发器(触发器)部分。

#### 7.3.1. 固定时间窗口（Fixed-time windows）

下面的示例代码展示了如何应用窗口来划分PCollection
到时间间隔为1min的固定窗户上，:

```java
    PCollection<String> items = ...;
    PCollection<String> fixed_windowed_items = items.apply(
        Window.<String>into(FixedWindows.of(Duration.standardMinutes(1))));
```
```py
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets_test.py tag:setting_fixed_windows
%}
```

#### 7.3.2. 滑动时间窗口（Sliding time windows）

下面的示例代码展示了如何应用窗口将`PCollection`分割为**##滑动时间窗口**。每个窗口长度为30分钟，每5秒启动一个新窗口:

```java
    PCollection<String> items = ...;
    PCollection<String> sliding_windowed_items = items.apply(
        Window.<String>into(SlidingWindows.of(Duration.standardMinutes(30)).every(Duration.standardSeconds(5))));
```
```py
{% github_sample   /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets_test.py tag:setting_sliding_windows
%}
```

#### 7.3.3. 会话窗口（Session windows）

下面的示例代码展示了如何应用窗口将PCollection划分为会话窗口，其中每个会话必须被至少10分钟的时间间隔分隔开:
```java
    PCollection<String> items = ...;
    PCollection<String> session_windowed_items = items.apply(
        Window.<String>into(Sessions.withGapDuration(Duration.standardMinutes(10))));
```
```py
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets_test.py tag:setting_session_windows
%}
```
注意，会话是每个键-集合中的每个键将根据数据分布有自己的会话分组。

#### 7.3.4. 全局窗口（Single global window）

如果您的`PCollection`是有界的(大小是固定的)，您可以将所有的元素分配到一个全局窗口中。下面的示例代码展示了如何为PCollection设置单一的全局窗口:

```java
    PCollection<String> items = ...;
    PCollection<String> batch_items = items.apply(
        Window.<String>into(new GlobalWindows()));
```
```py
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets_test.py tag:setting_global_window
%}
```

### 7.4. 水印与延迟数据（Watermarks and late data）

在任何数据处理系统中,有一定的时间差数据事件(**“事件时间”**,由时间戳数据元素本身)和实际数据元素的时间可能在`Pipeline`的任何阶段发生或被处理(**处理时间**,由系统上的时钟处理的元素)。
此外，也不能保证数据事件将在您的`Pipeline`中以与它们生成相同的顺序出现。    
例如，假设我们有一个使用固定时间窗口的`PCollection`，有5分钟的窗口。
对于每个窗口，`Beam`必须在给定的窗口范围内收集所有的事件时间时间戳(例如，在第一个窗口的0:00到4:59之间)。
在该范围之外的时间戳(从5点或以后的数据)属于不同的窗口。

然而，数据并不总是保证按时间顺序到达管道，或者总是以可预测的间隔到达。
Beam追踪的是一个水印，这是系统的概念，即当某个窗口中的所有数据都以期望的时间到达管道时。
从而把时间戳在水印后的数据被认为是`延迟数据`。

在我们的示例中，假设我们有一个简单的水印，它假定数据时间戳(事件时间)和数据出现在管道中的时间(处理时间)之间大约有30秒的延迟时间，那么Beam将在5:30关闭第一个窗口。
如果数据记录在5:34到达，但是有一个时间戳将它放在0:00-4:59窗口(例如，3:38)，那么该记录就是较晚的数据。

> **注意:** 为了简单起见，我们假设我们使用的是一个非常简单的水印，用来估计延迟时间。  

在实践中，您的PCollection的数据源决定了水印，而水印可以更精确或更复杂。    
`Beam` 的默认窗口配置尝试确定所有数据何时到达(基于数据源的类型)，然后在窗口的末端向前推进水印。
这种默认配置不允许延迟数据。
触发器(触发器)允许您修改和细化`PCollection`的窗口策略。
您可以使用触发器来决定每个单独的窗口何时聚合并报告其结果，包括窗口如何释放延迟的元素。

#### 7.4.1. 延迟数据管理

> **Note:** 管理延迟数据在Python的Beam SDK中暂不支持。

当你设置你的窗口处策略时，你可以通过调用`.withAllowedLateness`来允许处理延迟数据。下面的代码示例演示了一个窗口策略，该策略允许在窗口结束后的两天内进行后期数据。

```java
    PCollection<String> items = ...;
    PCollection<String> fixed_windowed_items = items.apply(
        Window.<String>into(FixedWindows.of(Duration.standardMinutes(1)))
              .withAllowedLateness(Duration.standardDays(2)));
```

当您在“PCollection”上设置`.withAllowedLateness`时，允许延迟从第一个“PCollection”中派生出来的任何一个子“PCollection”上前向传播。如果您希望在以后的管道中能更改允许的延迟，那么您必须使用`Window.configure().withAllowedLateness()`来明确地实现。

### 7.5. 将时间戳添加到PCollection的元素中

无界源为每个元素提供时间戳。根据您的无界源，您可能需要配置如何从原始数据流中提取时间戳。

然而，有界源(例如来自`TextIO`的文件)不提供时间戳。
如果需要时间戳，必须将它们添加到`PCollection`的元素中。

您可以通过应用[ParDo](#ParDo)转化来为`PCollection`的元素分配新的时间戳，该转化将使用您设置的时间戳来输出新元素。

一个可能的示例，如果您的管道从输入文件读取日志记录，并且每个日志记录都包含一个时间戳字段;
由于您的管道从一个文件中读取记录，则文件源不会自动地分配时间戳。
您可以从每个记录中解析时间戳字段，并使用`DoFn`的`ParDo`转化将时间戳附加到您的`PCollection`中的每个元素。

```java
      PCollection<LogEntry> unstampedLogs = ...;
      PCollection<LogEntry> stampedLogs =
          unstampedLogs.apply(ParDo.of(new DoFn<LogEntry, LogEntry>() {
            public void processElement(ProcessContext c) {
              // 从当前正在处理的日志条目中提取时间戳。
              Instant logTimeStamp = extractTimeStampFromLogEntry(c.element());
              // 使用ProcessContext.outputWithTimestamp(而不是ProcessContext.output)
              //发送带有时间戳的实体。
              c.outputWithTimestamp(c.element(), logTimeStamp);
            }
          }));
```
```py
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets_test.py tag:setting_timestamp
%}
```

## 8. Triggers

> **注意:** Triggers只能应用于Java的the Beam SDK。
> python的the Beam SDK不支持Triggers.

当收集和分组数据到窗口中时，Beam使用**tirggers**来确定什么时候发出每个窗口的汇总结果(称为a
*面板*)。如果你使用Beam的默认窗口配置和[默认触发器 default
trigger](#the-default-trigger)，当`beam`判断所有的[数据都已经到达（estimates all data has arrived）](#watermarks-and-late-data)，它会输出汇总结果
并丢弃该窗口的所有后续数据。

你可以为你的`PCollection`设置其他触发器来更改这个默认行为。
Beam提供了许多预置的触发器:

*   **基于事件时间的触发器**：正如每个数据元素的时间戳所暗示的那样，这些触发器运行在事件时间上。而且Beam的默认触发就是基于事件时间。

*   **基于处理时间的触发器**：这些触发器运行在处理时间——数据元素可以在传输过程中在任何给定阶段被处理
*   **数据驱动的触发器**：基于这些触发器通过检查到达每个窗口的数据来运行，一旦数据遇到某个特定属性就触发。目前，数据驱动的触发器仅支持遇到特定数字后触发。
*   **复合触发器**：这些触发器将多个触发器用不同的方式组合起来。


在较高的层次上，触发器提供了两个额外的功能，而不是简单地在窗口的末尾输出:

*   在给定窗口的所有数据都到达之前，触发器允许Beam发送早期结果。
例如，在一定时间流逝之后或者在特定数量的元素到达之后发出结果。
*   触发器允许在事件时间水印通过窗口结束后触发处理延迟数据。

这些功能允许你根据不同的用例来控制你数据流，并且平衡不同的影响因素:

*   **完整性:** 在你计算出结果之前，拥有所有数据有多么重要？
*   **延迟:** 你希望等待数据的时间有多长?例如，你是否会一直等到你认为你拿到所有的数据的时候?当它到达时，你处理数据吗?
*   **成本：** 你愿意花多少钱来降低延迟?

例如，一个要求时间敏感的系统更新可能会使用一个严格的基于时间的触发器——每N秒就会发出一个窗口，比起数据完整性更重视数据及时性。
一个重视数据完整性而不是结果的精确时间的系统可能会选择使用Beam的默认触发器，在窗口的最后面发出结果。

你还可以为一个[采用全局窗口的 global window for its windowing function](#windowing)无界`PCollection`设置触发器。在你希望你的数据管道能够为一个无界数据集提供定期更新的时候是非常有用的
——例如，提供当前时间或者更新的每N秒的所有数据的运行平均值，或者每N个元素。

### 8.1. 事件时间触发(Event time triggers)

`AfterWatermark`触发器在事件时间上运行。基于与数据元素相连的时间戳，在水印经过窗口后，`AfterWatermark`触发器会发出窗口的内容。[水印](#Watermark)是一种全局进度度量，在任何给定的点上，都是Beam在管道内输入完整性的概念。`AfterWatermark.pastEndOfWindow()`只有当水印经过窗口时才会起作用。
此外，如果您的管道在窗口结束之前或之后接收到数据，那么可以使用`.withEarlyFirings(trigger)` 与`.withLateFirings(trigger)`配置一个触发器去处理。
下面的示例展示了一个账单场景，并使用了早期和后期的[**Firings**]:

```java
  // 在月末创建一个账单
  AfterWatermark.pastEndOfWindow()
      // 在这个月，要接近实时的估计。
      .withEarlyFirings(
          AfterProcessingTime
              .pastFirstElementInPane()
              .plusDuration(Duration.standardMinutes(1))
      // Fire on any late data so the bill can be corrected.
      //对任何迟来的数据进行Fire，这样就可以纠正该账单。
      .withLateFirings(AfterPane.elementCountAtLeast(1))
```
```py
  # Beam SDK for Python 不支持Striggers
```

#### 8.1.1. 默认触发器

`PCollection`的默认触发器是基于事件时间的，当Beam的watermark经过窗口的末端时，会发出窗口的结果，然后每次延迟的数据到达时都会触发。

但是，如果您同时使用默认的窗口配置和默认触发器，默认触发器只会发送一次，而延迟的数据则会被丢弃。这是因为默认的窗口配置有一个允许的延迟值为0。有关修改此行为的信息，请参见处理延迟数据部分。

### 8.2. 处理时间触发器（**time trigger**）

`AfterProcessingTime`触发器在处理时间上运行。
例如，在接收到数据之后，`AfterProcessingTime.pastFirstElementInPane() `会释放一个窗口。处理时间由系统时钟决定，而不是数据元素的时间戳。
`AfterProcessingTime`对于来自窗口的早期结果非常有用，尤其是具有大型时间框架的窗口，例如一个全局窗口。

### 8.3. 数据驱动触发器(Data-driven triggers)

Beam提供了一种数据驱动触发器`AfterPane.elementCountAtLeast()`。这个触发器在元素计数上起作用;在当前panes至少收集了N个元素之后，它就会触发。这允许一个窗口可以释放早期的结果(在所有的数据积累之前)，如果您使用的是一个全局窗口，那么这个窗口就特别有用。

需要特别注意，例如，如果您使用`.elementCountAtLeast(50)`计数但是只有32个元素到达，那么这32个元素永远存在。如果32个元素对您来说很重要，那么考虑使用复合触发器(组合-触发器)来结合多个条件。这允许您指定多个触发条件，例如“当我接收到50个元素时，或者每1秒触发一次”。

### 8.4. 触发器设置


当您使用窗口转换为PCollection设置一个窗口函数时，您还可以指定一个触发器。
You set the trigger(s) for a `PCollection` by invoking the method
`.triggering()` on the result of your `Window.into()` transform, as follows:
您可以在`Window.into()`转化基础上调用`.triggering()`方法来为PCollection设置触发器(s)，如下:
```java
  PCollection<String> pc = ...;
  pc.apply(Window.<String>into(FixedWindows.of(1, TimeUnit.MINUTES))
                               .triggering(AfterProcessingTime.pastFirstElementInPane()
                                                              .plusDelayOf(Duration.standardMinutes(1)))
                               .discardingFiredPanes());
```
```py
  # Beam SDK for Python 不支持Striggers.
```
这个代码样例为`PCollection`设置了一个基于时间的触发器，在该窗口的第一个元素被处理后一分钟就会发出结果。
代码样例中的最后一行`.discardingFiredPanes()`是窗口的积累模式**accumulation mode**。

#### 8.4.1. 窗口积累模式[Window accumulation modes]

当您指定一个触发器时，您还必须设置窗口的累积模式。当触发器触发时，它会将窗口的当前内容作为一个panes发出。由于触发器可以多次触发，所以积累模式决定系统是否会在触发器触发时积累窗口panes，或者丢弃它们。
要设置一个窗口来积累触发器触发时产生的panes，请调用`.accumulatingFiredPanes()`当你设置触发器时。要设置一个窗口来丢弃被触发的panes，调用`.discardingFiredPanes()`。

让我们来看一个使用固定时间窗口和基于数据的触发器的`PCollection`的例子。这是您可能会做的事情，例如，每个窗口代表一个10分钟的运行平均值，但是您想要在UI中显示平均值的当前值，而不是每十分钟。我们将假设以下条件:

*   The `PCollection` 采用10-minute 固定时间窗口.
*   The `PCollection` 每次三个元素到达，可被仿佛触发的触发器.

下图显示了键X的数据事件，它们到达了PCollection并被分配给了windows。为了保持图表的简单，我们假定所有事件都按顺序到达了管道。
![图数据事件积累模式示例]({{ "/images/trigger-accumulation.png" | prepend: site.baseurl }} "Data events for accumulating mode example")

##### 8.4.1.1. 积累模式（Accumulating mode）

如果我们的触发器被设置为`.accumulatingFiredPanes`。每次触发时，触发器都会释放出如下的值。请记住，每当有三个元素到达时，触发器就会触发:
```
  First trigger firing:  [5, 8, 3]
  Second trigger firing: [5, 8, 3, 15, 19, 23]
  Third trigger firing:  [5, 8, 3, 15, 19, 23, 9, 13, 10]
```


##### 8.4.1.2. 丢弃模式

如果你的触发器设置成`.discardingFiredPanes`, 则在每一次触发时会释放如下值:

```
  First trigger firing:  [5, 8, 3]
  Second trigger firing:           [15, 19, 23]
  Third trigger firing:                         [9, 13, 10]
```

#### 8.4.2. 延迟数据处理

如果您希望您的管道处理watermark在窗口结束后到达的数据，那么您可以在设置窗口配置时应用一个允许的延迟。这使您的触发器有机会对最近的数据作出反应。如果设置了延迟置，默认的触发器将在任何延迟的数据到达时立即释放新的结果。

当你设置窗口功能时，使用`.withAllowedLateness()`设置允许延迟:

```java
  PCollection<String> pc = ...;
  pc.apply(Window.<String>into(FixedWindows.of(1, TimeUnit.MINUTES))
                              .triggering(AfterProcessingTime.pastFirstElementInPane()
                                                             .plusDelayOf(Duration.standardMinutes(1)))
                              .withAllowedLateness(Duration.standardMinutes(30));
```
```py
  # Beam SDK for Python 不支持Striggers
```

这允许延迟传播到所有的`PCollection`，这是由于将转换应用到原始的PCollection而产生的。如果您想要在您的管道中更改允许的延迟，您可以再次使用`Window.configure().withAllowedLateness()`。

### 8.5. 复合触发器Composite triggers

您可以组合多个触发器来形成复合触发器，并且可以指定触发器，在大多数情况下，或者在其他定制条件下，多次释放结果。

#### 8.5.1. Composite trigger 类型

Beam 包括如下类型 composite triggers:

*   你可以额外通过`.withEarlyFirings` 和`.withLateFirings`添加早期 firings 或者晚期 firings的`AfterWatermark.pastEndOfWindow`.
*   `Repeatedly.forever`指定一个永远执行的触发器。只要触发了触发器的条件，它就会产生一个窗口来释放结果，然后重新设置并重新启动。将`Repeatedly.forever`与`.orFinally`结合在一起，指定条件，使某个重复触发的触发器停止触发。
*   `AfterEach.inOrder`将多个触发器组合在一个特定的序列中。每当序列中的触发器发出一个窗口时，序列就会向下一个触发器前进。
*   `AfterFirst`获取多个触发器，并第一次发出它的任何一个参数触发器都是满意的。这相当于多个触发器的逻辑或操作。当所有的参数触发器都被满足时，
*   `AfterAll`就需要多个触发器并发出,这相当于多个触发器的逻辑和操作。
*   `orFinally` ，它可以作为最后的条件，使任何触发器在最后时刻触发，而不会再次触发。

#### 8.5.2. AfterWatermark.pastEndOfWindow的复合触发器

当Beam估计所有的数据都已经到达(例如，当水印通过窗口的末端)时，一些最有用的组合触发了一段时间，或者两者的结合，或者两者的结合:
*   推测触发(Speculative firings)能够在水印通过窗口的末端时，允许对部分结果进行更快的处理。
*   延迟触发（Late firings）在水印经过窗口后的延迟触发，以便处理延迟到达的数据。

你可以使用`AfterWatermark.pastEndOfWindow`表示这种模式。例如，下面的例子触发了以下条件下的代码:
*   根据Beam的估计，所有的数据都已经到达(水印通过窗口的末端)
*   任何时间延迟的数据到达，经过10分钟的延迟
*   两天之后，我们假设没有更多的数据将到达，触发器停止执行

```java
  .apply(Window
      .configure()
      .triggering(AfterWatermark
           .pastEndOfWindow()
           .withLateFirings(AfterProcessingTime
                .pastFirstElementInPane()
                .plusDelayOf(Duration.standardMinutes(10))))
      .withAllowedLateness(Duration.standardDays(2)));
```
```py
  # Beam SDK for Python 不支持Striggers
```

#### 8.5.3. 其他 composite triggers

您还可以构建其他类型的复合触发器。下面的示例代码显示了一个简单的复合触发器，只要该pane至少有100个元素，或者一分钟后就会触发。

```java
  Repeatedly.forever(AfterFirst.of(
      AfterPane.elementCountAtLeast(100),
      AfterProcessingTime.pastFirstElementInPane().plusDelayOf(Duration.standardMinutes(1))))
```
```py
  # Beam SDK for Python 不支持Striggers
```
