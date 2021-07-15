This document is a draft proposal for Yet Another (Test) Report Format (YARF). It is intended to represent test results
from a variety of test automation tools as well as manual tests.

# Requirements

Test reporting tools that process and display test results typically have multiple requirements:

* Group test results from multiple different testing tools and test executions.
* Streamable format. Efficient processing of large result sets is important to keep memory consumption low.
* Generate an aggregated test result for multiple tests
* Display details for individual tests
* Support multiple testing tools
* Display a hierarchy of tests
* Display a subset of tests
* Display attachments such as screenshots and logs gathered during test execution
* Grouping tests by tags
* Store test results in a database
* Compact data format

## Data format

YARF is represented as a "stream" of `TestNode` objects that describe tests. While the stream of objects is a linear sequence of nodes, their `id` and `parentId` properties can be used to construct a hierarchy (tree) of test nodes.

The stream can be encoded as [NDJSON](http://ndjson.org/), a [JSON](https://www.json.org/json-en.html) array or an array of objects.

Below is the format of the `TestNode` objects (in TypeScript) This can be translated to a more portable format such as [JSON Schema](https://json-schema.org/) later.

```typescript
/**
 * A test node is either a leaf in a tree, or a node containing other nodes.
 */
type TestNode = {
  // The ID of the test node. This must be unique within a test execution (message stream).
  // The ID *may* be different for a new execution of the same test.
  id: string,
  // The ID of the parent node. This is `undefined` for the root node
  parentId?: string
  // A string representing the type of the node. This is specific to the test automation tool.
  type: string,
  // A reference back to the source code where this node came from. The format of
  // this field is a URI. Examples:
  //   source-reference:file://example.file?line=12&column=13
  //   source-reference:java:com/example/Example#testMethod()
  // A separate sub-spec might be needed for these URIs - maybe there is already a spec
  // or defacto format we can use (like the URLs IDEs print in stack traces).
  sourceRef: string
  // A stable ID that does not change between test execution. This field will be empty until we've built a tool
  // that can compute stable IDs for tests even when its name and line number changes from one revision to another.
  entityId?: string
  // The human readable name of the test node. For file system nodes this is the file name. For classes
  // it is the class name, for methods it is the method name etc.
  name: string,
  // The timestamp of when this node started executing. Only present for leaf nodes.
  // Formatted as an ISO 8601 string (ideally with millisecond precision)
  timestamp: string,
  // How long it took to execute. Only present for leaf nodes.
  duration?: Duration,
  // The status of the test execution. Must be present for leaf nodes.
  // The field may be specified by container nodes.
  // If the field is absent for a container node, consumers should compute this
  // the aggregated result from child nodes.
  // TODO: Document the algorithm for computing aggregate results.
  result?: Status,
  // Attachments collected during test execution
  attachments: Attachment[]
  // Tags attached to the test node
  tags: string[]
}

/**
 * An attachment represents an arbitrary piece of data attached to a test node. It can be used for images (screenshots), logs etc.
 */
type Attachment = {
  // The body of the attachment.
  body: string
  // The content encoding of the body.
  contentEncoding: AttachmentContentEncoding
  // The IANA media type of the body or data behind the url
  mediaType: string
}

enum AttachmentContentEncoding {
  // The content should be interpreted as-is (as text)
  IDENTITY = 'IDENTITY',
  // The content is binary, encoded with Base64
  // https://datatracker.ietf.org/doc/html/rfc4648
  BASE64 = 'BASE64',
}

type Timestamp = {
  seconds: number
  nanos: number
}

type Duration = {
  seconds: number
  nanos: number
}
```

## Mapping

Each testing tool will have its own tool-specific format for representing tests and test results. The examples below illustrate how
these representations could be mapped to `TestNode` objects.

### File system

```
.                                    -> { id: 'a', parentId: undefined, type: 'directory' }
├── features                         -> { id: 'b', parentId: 'a', type: 'directory', name: 'features' }
│   └── bar                          -> { id: 'c', parentId: 'b', type: 'directory', name: 'bar' }
│       └── three.feature            -> { id: 'd', parentId: 'c', type: 'file',      name: 'three.feature' }
├── lib                              -> { id: 'e', parentId: 'a', type: 'directory' }
│   └── spec                         -> { id: 'f', parentId: 'h', type: 'directory' }
│       └── my_spec.rb               -> { id: 'g', parentId: 'i', type: 'file' }
└── src                              -> { id: 'h', parentId: 'a', type: 'directory' }
    └── test                         -> { id: 'i', parentId: 'k', type: 'directory' }
        └── java                     -> { id: 'j', parentId: 'l', type: 'directory' }
            └── com                  -> { id: 'k', parentId: 'm', type: 'directory' }
                └── bigcorp          -> { id: 'l', parentId: 'n', type: 'directory' }
                    └── MyTest.java  -> { id: 'm', parentId: 'o', type: 'file' }
```

### Cucumber

```gherkin
# features/bar/three.feature        
Feature: Three                    #  -> { id: 'n', parentId: 'c', type: 'gherkin-feature',  name: 'Feature: Three' }
  Scenario: Blah                  #  -> { id: 'o', parentId: 'n', type: 'gherkin-scenario', name: 'Scenario: Blah' }
    Given blah                    #  -> { id: 'p', parentId: 'o', type: 'gherkin-step', name: 'Given blah' }

  Scenario Outline: Blah          #  -> { id: 'q', parentId: 'n', type: 'gherkin-scenario-outline' }
    Given <snip>
    When <snap>

    Examples:
      | snip | snap  |
      | red  | green |            #  -> { id: 'r', parentId: 'q', type: 'gherkin-step', status: 'passed' }
                                  #  -> { id: 's', parentId: 'q', type: 'gherkin-step', status: 'failed' }

  Rule: Blah                      #  -> { id: 't', parentId: 'n', type: 'gherkin-rule' }
    Scenario: Blah                #  -> { id: 'u', parentId: 't', type: 'gherkin-scenario' }
```

### JUnit

```java
// src/test/java/com/bigcorp/MyTest.java
class MyTest {                   // -> { id: 'v', parentId: 'm', name: 'com.bigcorp.MyTest' }
    @Test
    public void something() {    // -> { id: 'w', parentId: 'v', status: 'passed', name: 'something()' }
    }
}
```

### RSpec

```ruby
# lib/spec/my_spec.rb
describe 'something' do           # -> { id: 'w', parentId: 'g', type: 'rspec-context' }
  it 'something else' do          # -> { id: 'x', parentId: 'w', type: 'rspec-example' }
  end
end
```

## Stream processing

Consumers of a `TestNode` stream should process the objects according to their needs. Below is an example of an imaginary application `JoyReports` that is only interested in rendering test results from Cucumber Scenarios:

```typescript
const joyReportsScenarioById = new Map<string, JoyReportScenario>()

for(const testNode of testNodes) {
  if(testNode.type === 'gherkin-scenario' || testNode.type === 'gherkin-scenario-outline') {
    joyReportsScenarioById.put(testNode.id, new JoyReportsScenario(testNode))
  }
  if(testNode.type === 'gherkin-step') {
    joyReportsScenario = joyReportsScenarioById.get(testNode.parentId)
    joyReportsScenario.addJoyReportsStep(new JoyReportsStep(testNode))
  }
}
```

## Mapping to YARF

In order to support the YARF specification, each testing tool will need a
corresponding mapping tool that translates the native test result format into YARF.

### Cucumber

Cucumber's native test result format is [Cucumber Messages](https://github.com/cucumber/common/tree/main/messages#readme).

The Cucumber Open project would build a separate library (probably in TypeScript) that can read a stream of Cucumber Messages and output a stream of `TestNode` objects.

This library could then be used in existing tools:

* Cucumber Reports could use it in a Lambda to produce YARF and store this in its DynamoDB database
* Cucumber Open's HTML reporter could use it to embed YARF in the statically generated HTML document
* Cucumber for JIRA (C4J) could use it

#### Different mappers

There is more than one way to map a Gherkin Document to a YARF streams, especially for `Background` and `Scenario Outline`. We could experiment with different mapping strategies. The nice thing is that the output format is the same, just the shape of the tree will differ. This means consumers (such as React components) would not need to be adapted to different mappers.

## High fidelity test rendering

While the YARF format can be used to build a UI to render tests, it doesn't contain all the information to provide a high
fidelity display of the *source* of the test. For example, the YARF format does not include Cucumber's `DataTable` or `DocString`.

In order to make a high fidelity display of a test, the source representation of the test must be used for high fidelity rendering.

To use Cucumber as a further example, Gherkin provides a `GherkinDocument` data structure that represents the full abstract syntax tree (AST) for a `.feature` file.

A `<GherkinDocument />` React component could be used to render the whole document, and a `<Scenario />` component could be used to
render just a `Scenario` deeper in that AST. When we render a `<Step />`, we want to display the `status`, which isn't present in the
AST, but it is present in one of the YARF `TestNode` objects.

This is where the `TestNode#sourceRef` property comes in handy. A `Map<string,TestNode>` map can be built by iterating over the `TestNode`
objects, and the `<Step />` component could use its `Step#id` to look up the corresponding `TestNode` in that Map.

## Stable test IDs

Some testing tools (like JUnit) are able to produce the same `TestNode#id` for several executions.
Some testing tools (like Cucumber) do not have this ability.

Since the `TestNode#id` field *may* be different for each test execution, it should only be used to build
a hierarchy of tests, and not for tracking the test across multiple exec

Some systems may want to report on the "flakiness" of a test over time, across multiple revisions of that test. For example, a test named
`my_little_test` on line `48` in revision `r10` may have moved to line `56` in `r11`, and in `r12` it may have been renamed to `my_medium_test`.

Still, it could be considered to be the same "conceptual" test. This is where the `TestNode#entityId` comes in. If this ID is stable
from test execution to test execution, across revisions, then flakiness and other trend metrics could be computed.

Computing a stable `entityId` is possible, using techniques such as [lhdiff](https://ieeexplore.ieee.org/document/6676894). This is something
we may want to add support for in a separate tool at some stage. The field can be left empty for now, but if such a tool becomes available,
it can be used to populate this field.

Alternatively, mappers could assign a value to this field using other less reliable algorithms.

## Relevant links

This document is based on the following discussions and documents

* https://github.com/cucumber/common/issues/1462
* https://github.com/cucumber/common/issues/753
* https://github.com/SmartBear/cucumber-result-components
* https://github.com/cucumber/common/tree/main/react/javascript#readme
* https://github.com/SmartBear/cucumber-reports/pull/706

# Team

We should reach out to open source communities as well as people in SmartBear who might be interested in shaping the spec

* https://twitter.com/_nicojs/status/1408581271178125313
* https://github.com/junit-team/junit5/issues/373
* Rspec team
* Selenium team (Simon Stewart)
* Tap team
* Swagger team
* CI server vendors
* Maven Surefire team
