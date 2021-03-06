---
layout: default
title: "Beam WordCount Example"
permalink: get-started/wordcount-example/
redirect_from: /use/wordcount-example/
---

# Apache Beam WordCount Example

* TOC
{:toc}

<nav class="language-switcher">
  <strong>Adapt for:</strong> 
  <ul>
    <li data-type="language-java">Java SDK</li>
    <li data-type="language-py">Python SDK</li>
  </ul>
</nav>

The WordCount examples demonstrate how to set up a processing pipeline that can read text, tokenize the text lines into individual words, and perform a frequency count on each of those words. The Beam SDKs contain a series of these four successively more detailed WordCount examples that build on each other. The input text for all the examples is a set of Shakespeare's texts.

Each WordCount example introduces different concepts in the Beam programming model. Begin by understanding Minimal WordCount, the simplest of the examples. Once you feel comfortable with the basic principles in building a pipeline, continue on to learn more concepts in the other examples.

* **Minimal WordCount** demonstrates the basic principles involved in building a pipeline.
* **WordCount** introduces some of the more common best practices in creating re-usable and maintainable pipelines.
* **Debugging WordCount** introduces logging and debugging practices.
* **Windowed WordCount** demonstrates how you can use Beam's programming model to handle both bounded and unbounded datasets.

## MinimalWordCount

Minimal WordCount demonstrates a simple pipeline that can read from a text file, apply transforms to tokenize and count the words, and write the data to an output text file. This example hard-codes the locations for its input and output files and doesn't perform any error checking; it is intended to only show you the "bare bones" of creating a Beam pipeline. This lack of parameterization makes this particular pipeline less portable across different runners than standard Beam pipelines. In later examples, we will parameterize the pipeline's input and output sources and show other best practices.

To run this example, follow the instructions in the Quickstart for [Java]({{ site.baseurl }}/get-started/quickstart-java) or [Python]({{ site.baseurl }}/get-started/quickstart-py). To view the full code, see **[MinimalWordCount](https://github.com/apache/beam/blob/master/examples/java/src/main/java/org/apache/beam/examples/MinimalWordCount.java).**

**Key Concepts:**

* Creating the Pipeline
* Applying transforms to the Pipeline
* Reading input (in this example: reading text files)
* Applying ParDo transforms
* Applying SDK-provided transforms (in this example: Count)
* Writing output (in this example: writing to a text file)
* Running the Pipeline

The following sections explain these concepts in detail along with excerpts of the relevant code from the Minimal WordCount pipeline.

### Creating the Pipeline

The first step in creating a Beam pipeline is to create a `PipelineOptions` object. This object lets us set various options for our pipeline, such as the pipeline runner that will execute our pipeline and any runner-specific configuration required by the chosen runner. In this example we set these options programmatically, but more often command-line arguments are used to set `PipelineOptions`. 

You can specify a runner for executing your pipeline, such as the `DataflowRunner` or `SparkRunner`. If you omit specifying a runner, as in this example, your pipeline will be executed locally using the `DirectRunner`. In the next sections, we will specify the pipeline's runner.

```java
 PipelineOptions options = PipelineOptionsFactory.create();

    // In order to run your pipeline, you need to make following runner specific changes:
    //
    // CHANGE 1/3: Select a Beam runner, such as DataflowRunner or FlinkRunner.
    // CHANGE 2/3: Specify runner-required options.
    // For DataflowRunner, set project and temp location as follows:
    //   DataflowPipelineOptions dataflowOptions = options.as(DataflowPipelineOptions.class);
    //   dataflowOptions.setRunner(DataflowRunner.class);
    //   dataflowOptions.setProject("SET_YOUR_PROJECT_ID_HERE");
    //   dataflowOptions.setTempLocation("gs://SET_YOUR_BUCKET_NAME_HERE/AND_TEMP_DIRECTORY");
    // For FlinkRunner, set the runner as follows. See {@code FlinkPipelineOptions}
    // for more details.
    //   options.setRunner(FlinkRunner.class);
```

```py
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets.py tag:examples_wordcount_minimal_options
%}```

The next step is to create a Pipeline object with the options we've just constructed. The Pipeline object builds up the graph of transformations to be executed, associated with that particular pipeline.

```java
Pipeline p = Pipeline.create(options);
```

```py
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets.py tag:examples_wordcount_minimal_create
%}```

### Applying Pipeline Transforms

The Minimal WordCount pipeline contains several transforms to read data into the pipeline, manipulate or otherwise transform the data, and write out the results. Each transform represents an operation in the pipeline.

Each transform takes some kind of input (data or otherwise), and produces some output data. The input and output data is represented by the SDK class `PCollection`. `PCollection` is a special class, provided by the Beam SDK, that you can use to represent a data set of virtually any size, including unbounded data sets.

<img src="{{ "/images/wordcount-pipeline.png" | prepend: site.baseurl }}" alt="Word Count pipeline diagram">
Figure 1: The pipeline data flow.

The Minimal WordCount pipeline contains five transforms:

1.  A text file `Read` transform is applied to the Pipeline object itself, and produces a `PCollection` as output. Each element in the output PCollection represents one line of text from the input file. This example uses input data stored in a publicly accessible Google Cloud Storage bucket ("gs://").

    ```java
    p.apply(TextIO.read().from("gs://apache-beam-samples/shakespeare/*"))
    ```

    ```py
    {% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets.py tag:examples_wordcount_minimal_read
    %}```

2.  A [ParDo]({{ site.baseurl }}/documentation/programming-guide/#transforms-pardo) transform that invokes a `DoFn` (defined in-line as an anonymous class) on each element that tokenizes the text lines into individual words. The input for this transform is the `PCollection` of text lines generated by the previous `TextIO.Read` transform. The `ParDo` transform outputs a new `PCollection`, where each element represents an individual word in the text.

    ```java
    .apply("ExtractWords", ParDo.of(new DoFn<String, String>() {
        @ProcessElement
        public void processElement(ProcessContext c) {
            // \p{L} denotes the category of Unicode letters,
            // so this pattern will match on everything that is not a letter.
            for (String word : c.element().split("[^\\p{L}]+")) {
                if (!word.isEmpty()) {
                    c.output(word);
                }
            }
        }
    }))
    ```

    ```py
    # The Flatmap transform is a simplified version of ParDo.
    {% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets.py tag:examples_wordcount_minimal_pardo
    %}```

3.  The SDK-provided `Count` transform is a generic transform that takes a `PCollection` of any type, and returns a `PCollection` of key/value pairs. Each key represents a unique element from the input collection, and each value represents the number of times that key appeared in the input collection.

	In this pipeline, the input for `Count` is the `PCollection` of individual words generated by the previous `ParDo`, and the output is a `PCollection` of key/value pairs where each key represents a unique word in the text and the associated value is the occurrence count for each.

    ```java
    .apply(Count.<String>perElement())
    ```

    ```py
    {% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets.py tag:examples_wordcount_minimal_count
    %}```

4.  The next transform formats each of the key/value pairs of unique words and occurrence counts into a printable string suitable for writing to an output file.

	The map transform is a higher-level composite transform that encapsulates a simple `ParDo`. For each element in the input `PCollection`, the map transform applies a function that produces exactly one output element.

    ```java
    .apply("FormatResults", MapElements.via(new SimpleFunction<KV<String, Long>, String>() {
        @Override
        public String apply(KV<String, Long> input) {
            return input.getKey() + ": " + input.getValue();
        }
    }))
    ```

    ```py
    {% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets.py tag:examples_wordcount_minimal_map
    %}```

5.  A text file write transform. This transform takes the final `PCollection` of formatted Strings as input and writes each element to an output text file. Each element in the input `PCollection` represents one line of text in the resulting output file.

    ```java
    .apply(TextIO.write().to("wordcounts"));
    ```

    ```py
    {% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets.py tag:examples_wordcount_minimal_write
    %}```
    
Note that the `Write` transform produces a trivial result value of type `PDone`, which in this case is ignored.

### Running the Pipeline

Run the pipeline by calling the `run` method, which sends your pipeline to be executed by the pipeline runner that you specified when you created your pipeline.

```java
p.run().waitUntilFinish();
```

```py
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets.py tag:examples_wordcount_minimal_run
%}```

Note that the `run` method is asynchronous. For a blocking execution instead, run your pipeline appending the `waitUntilFinish` method.

## WordCount Example

This WordCount example introduces a few recommended programming practices that can make your pipeline easier to read, write, and maintain. While not explicitly required, they can make your pipeline's execution more flexible, aid in testing your pipeline, and help make your pipeline's code reusable.

This section assumes that you have a good understanding of the basic concepts in building a pipeline. If you feel that you aren't at that point yet, read the above section, [Minimal WordCount](#minimalwordcount).

To run this example, follow the instructions in the Quickstart for [Java]({{ site.baseurl }}/get-started/quickstart-java) or [Python]({{ site.baseurl }}/get-started/quickstart-py). To view the full code, see **[WordCount](https://github.com/apache/beam/blob/master/examples/java/src/main/java/org/apache/beam/examples/WordCount.java).**

**New Concepts:**

* Applying `ParDo` with an explicit `DoFn`
* Creating Composite Transforms
* Using Parameterizable `PipelineOptions`

The following sections explain these key concepts in detail, and break down the pipeline code into smaller sections.

### Specifying Explicit DoFns

When using `ParDo` transforms, you need to specify the processing operation that gets applied to each element in the input `PCollection`. This processing operation is a subclass of the SDK class `DoFn`. You can create the `DoFn` subclasses for each `ParDo` inline, as an anonymous inner class instance, as is done in the previous example (Minimal WordCount). However, it's often a good idea to define the `DoFn` at the global level, which makes it easier to unit test and can make the `ParDo` code more readable.

```java
// In this example, ExtractWordsFn is a DoFn that is defined as a static class:

static class ExtractWordsFn extends DoFn<String, String> {
    ...

    @ProcessElement
    public void processElement(ProcessContext c) {
        ...
    }
}
```

```py
# In this example, the DoFns are defined as classes:

{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets.py tag:examples_wordcount_wordcount_dofn
%}```

### Creating Composite Transforms

If you have a processing operation that consists of multiple transforms or `ParDo` steps, you can create it as a subclass of `PTransform`. Creating a `PTransform` subclass allows you to create complex reusable transforms, can make your pipeline's structure more clear and modular, and makes unit testing easier.

In this example, two transforms are encapsulated as the `PTransform` subclass `CountWords`. `CountWords` contains the `ParDo` that runs `ExtractWordsFn` and the SDK-provided `Count` transform.

When `CountWords` is defined, we specify its ultimate input and output; the input is the `PCollection<String>` for the extraction operation, and the output is the `PCollection<KV<String, Long>>` produced by the count operation.

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

public static void main(String[] args) throws IOException {
  Pipeline p = ...

  p.apply(...)
   .apply(new CountWords())
   ...
}
```

```py
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets.py tag:examples_wordcount_wordcount_composite
%}```

### Using Parameterizable PipelineOptions

You can hard-code various execution options when you run your pipeline. However, the more common way is to define your own configuration options via command-line argument parsing. Defining your configuration options via the command-line makes the code more easily portable across different runners.

Add arguments to be processed by the command-line parser, and specify default values for them. You can then access the options values in your pipeline code.

```java
public static interface WordCountOptions extends PipelineOptions {
  @Description("Path of the file to read from")
  @Default.String("gs://dataflow-samples/shakespeare/kinglear.txt")
  String getInputFile();
  void setInputFile(String value);
  ...
}

public static void main(String[] args) {
  WordCountOptions options = PipelineOptionsFactory.fromArgs(args).withValidation()
      .as(WordCountOptions.class);
  Pipeline p = Pipeline.create(options);
  ...
}
```

```py
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets.py tag:examples_wordcount_wordcount_options
%}```

## Debugging WordCount Example

The Debugging WordCount example demonstrates some best practices for instrumenting your pipeline code.

To run this example, follow the instructions in the Quickstart for [Java]({{ site.baseurl }}/get-started/quickstart-java) or [Python]({{ site.baseurl }}/get-started/quickstart-py). To view the full code, see **[DebuggingWordCount](https://github.com/apache/beam/blob/master/examples/java/src/main/java/org/apache/beam/examples/DebuggingWordCount.java).**

**New Concepts:**

* Logging
* Testing your Pipeline via `PAssert`

The following sections explain these key concepts in detail, and break down the pipeline code into smaller sections.

### Logging

Each runner may choose to handle logs in its own way.

```java
// This example uses .trace and .debug:

public class DebuggingWordCount {

  public static class FilterTextFn extends DoFn<KV<String, Long>, KV<String, Long>> {
    ...

    public void processElement(ProcessContext c) {
      if (...) {
        ...
        LOG.debug("Matched: " + c.element().getKey());
      } else {        
        ...
        LOG.trace("Did not match: " + c.element().getKey());
      }
    }
  }
}
```

```py
{% github_sample /apache/beam/blob/master/sdks/python/apache_beam/examples/snippets/snippets.py tag:example_wordcount_debugging_logging
%}```


#### Direct Runner

If you execute your pipeline using `DirectRunner`, it will print the log messages directly to your local console.

#### Dataflow Runner

If you execute your pipeline using `DataflowRunner`, you can use Stackdriver Logging. Stackdriver Logging aggregates the logs from all of your Dataflow job's workers to a single location in the Google Cloud Platform Console. You can use Stackdriver Logging to search and access the logs from all of the workers that Dataflow has spun up to complete your Dataflow job. Logging statements in your pipeline's `DoFn` instances will appear in Stackdriver Logging as your pipeline runs.

If you execute your pipeline using `DataflowRunner`, you can control the worker log levels. Dataflow workers that execute user code are configured to log to Stackdriver Logging by default at "INFO" log level and higher. You can override log levels for specific logging namespaces by specifying: `--workerLogLevelOverrides={"Name1":"Level1","Name2":"Level2",...}`. For example, by specifying `--workerLogLevelOverrides={"org.apache.beam.examples":"DEBUG"}` when executing this pipeline using the Dataflow service, Stackdriver Logging would contain only "DEBUG" or higher level logs for the package in addition to the default "INFO" or higher level logs.

The default Dataflow worker logging configuration can be overridden by specifying `--defaultWorkerLogLevel=<one of TRACE, DEBUG, INFO, WARN, ERROR>`. For example, by specifying `--defaultWorkerLogLevel=DEBUG` when executing this pipeline with the Dataflow service, Cloud Logging would contain all "DEBUG" or higher level logs. Note that changing the default worker log level to TRACE or DEBUG will significantly increase the amount of logs output.

#### Apache Spark Runner

> **Note:** This section is yet to be added. There is an open issue for this ([BEAM-792](https://issues.apache.org/jira/browse/BEAM-792)).

#### Apache Flink Runner

> **Note:** This section is yet to be added. There is an open issue for this ([BEAM-791](https://issues.apache.org/jira/browse/BEAM-791)).

#### Apache Apex Runner

> **Note:** This section is yet to be added. There is an open issue for this ([BEAM-2285](https://issues.apache.org/jira/browse/BEAM-2285)).

### Testing your Pipeline via PAssert

`PAssert` is a set of convenient PTransforms in the style of Hamcrest's collection matchers that can be used when writing Pipeline level tests to validate the contents of PCollections. `PAssert` is best used in unit tests with small data sets, but is demonstrated here as a teaching tool.

Below, we verify that the set of filtered words matches our expected counts. Note that `PAssert` does not produce any output, and pipeline will only succeed if all of the expectations are met. See [DebuggingWordCountTest](https://github.com/apache/beam/blob/master/examples/java/src/test/java/org/apache/beam/examples/DebuggingWordCountTest.java) for an example unit test.

```java
public static void main(String[] args) {
  ...
  List<KV<String, Long>> expectedResults = Arrays.asList(
        KV.of("Flourish", 3L),
        KV.of("stomach", 1L));
  PAssert.that(filteredWords).containsInAnyOrder(expectedResults);
  ...
}
```

```py
This feature is not yet available in the Beam SDK for Python.
```

## WindowedWordCount

This example, `WindowedWordCount`, counts words in text just as the previous examples did, but introduces several advanced concepts.

**New Concepts:**

* Unbounded and bounded pipeline input modes
* Adding timestamps to data
* Windowing
* Reusing PTransforms over windowed PCollections

The following sections explain these key concepts in detail, and break down the pipeline code into smaller sections.

### Unbounded and bounded pipeline input modes

Beam allows you to create a single pipeline that can handle both bounded and unbounded types of input. If the input is unbounded, then all PCollections of the pipeline will be unbounded as well. The same goes for bounded input. If your input has a fixed number of elements, it's considered a 'bounded' data set. If your input is continuously updating, then it's considered 'unbounded'.

Recall that the input for this example is a a set of Shakespeare's texts, finite data. Therefore, this example reads bounded data from a text file:

```java
public static void main(String[] args) throws IOException {
    Options options = ...
    Pipeline pipeline = Pipeline.create(options);

    PCollection<String> input = pipeline
      .apply(TextIO.read().from(options.getInputFile()))

```

```py
This feature is not yet available in the Beam SDK for Python.
```

### Adding Timestamps to Data

Each element in a `PCollection` has an associated **timestamp**. The timestamp for each element is initially assigned by the source that creates the `PCollection` and can be adjusted by a `DoFn`. In this example the input is bounded. For the purpose of the example, the `DoFn` method named `AddTimestampsFn` (invoked by `ParDo`) will set a timestamp for each element in the `PCollection`.
  
```java
.apply(ParDo.of(new AddTimestampFn()));
```

```py
This feature is not yet available in the Beam SDK for Python.
```

Below is the code for `AddTimestampFn`, a `DoFn` invoked by `ParDo`, that sets the data element of the timestamp given the element itself. For example, if the elements were log lines, this `ParDo` could parse the time out of the log string and set it as the element's timestamp. There are no timestamps inherent in the works of Shakespeare, so in this case we've made up random timestamps just to illustrate the concept. Each line of the input text will get a random associated timestamp sometime in a 2-hour period.

```java
static class AddTimestampFn extends DoFn<String, String> {
  private final Instant minTimestamp;
  private final Instant maxTimestamp;

  AddTimestampFn(Instant minTimestamp, Instant maxTimestamp) {
    this.minTimestamp = minTimestamp;
    this.maxTimestamp = maxTimestamp;
  }

  @ProcessElement
  public void processElement(ProcessContext c) {
    Instant randomTimestamp =
      new Instant(
          ThreadLocalRandom.current()
          .nextLong(minTimestamp.getMillis(), maxTimestamp.getMillis()));

    /**
     * Concept #2: Set the data element with that timestamp.
     */
    c.outputWithTimestamp(c.element(), new Instant(randomTimestamp));
  }
}
```

```py
This feature is not yet available in the Beam SDK for Python.
```

### Windowing

Beam uses a concept called **Windowing** to subdivide a `PCollection` according to the timestamps of its individual elements. PTransforms that aggregate multiple elements, process each `PCollection` as a succession of multiple, finite windows, even though the entire collection itself may be of infinite size (unbounded).

The `WindowedWordCount` example applies fixed-time windowing, wherein each window represents a fixed time interval. The fixed window size for this example defaults to 1 minute (you can change this with a command-line option). 

```java
PCollection<String> windowedWords = input
  .apply(Window.<String>into(
    FixedWindows.of(Duration.standardMinutes(options.getWindowSize()))));
```

```py
This feature is not yet available in the Beam SDK for Python.
```

### Reusing PTransforms over windowed PCollections

You can reuse existing PTransforms that were created for manipulating simple PCollections over windowed PCollections as well.

```java
PCollection<KV<String, Long>> wordCounts = windowedWords.apply(new WordCount.CountWords());
```

```py
This feature is not yet available in the Beam SDK for Python.
```

### Write Results to an Unbounded Sink

When our input is unbounded, the same is true of our output `PCollection`. We need to make sure that we choose an appropriate, unbounded sink. Some output sinks support only bounded output, while others support both bounded and unbounded outputs. By using a `FilenamePolicy`, we can use `TextIO` to files that are partitioned by windows. We use a composite `PTransform` that uses such a policy internally to write a single sharded file per window.

In this example, we stream the results to a BigQuery table. The results are then formatted for a BigQuery table, and then written to BigQuery using BigQueryIO.Write.

```java
  wordCounts
      .apply(MapElements.via(new WordCount.FormatAsTextFn()))
      .apply(new WriteOneFilePerWindow(output, options.getNumShards()));
```

```py
This feature is not yet available in the Beam SDK for Python.
```

