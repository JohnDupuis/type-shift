# Type Shift
Evolutionary Type Converters for input validation and transformation

## Installation
`npm install --save type-shift`
`yarn add type-shift`

## What is Type Shift
Type Shift is a tool for building type converters and validators for schemas that will evolve over time. With Type Shift you declare a converter that "pulls" various historical input schemas into the current schema.

### Why should I use Type Shift instead of X
Type Shift is explicitly designed as single directional input validators / converters for schemas that change over time. This is very useful if you control both the reader and the writer of data. A good example is internal apis, or validating data stored in untyped storage such as IndexedDB. In these cases even though you are both the producer and consumer of the data you cannot rely on that data being valid due to changes in your application over time or other people or processes deciding to change your data on you. It is often a lot of effort to use explicit schema versioning in these situations.

With Type Shift you define a converter that knows how to infer new schema fields from previous versions of the schema, these are then used to pull data into the new format so your application only needs to think about it's newest schema version. Type Shift can also be used as an adaptor to transform one schema into another, however for anything other than trivial cases it's often easier to define these transformations with code after type checking an input.

Type Shift is intentionally focused on uni-directional conversions. This makes it easy to define a series of transformations without needing to define a way to "un-transform" data. Some use cases would benefit from being able to convert data from and to the same schema in which case Type Shift may not be the right tool.

## Usage

### Converting a Value
Converters have 3 different methods for converting values depending on your needs.
- `convert` Takes a value and returns a value or throws and Error. In most cases this is the best option.
- `tryConvert` Take a value and returns a `ConverterResult` which is either successful with a value or not successful with errors. If you're building an API and want to respond with a 400 error with the issues this may be easier to use.
- `tryConvertNode` Takes a node and returns a `ConverterResult` of a Node. If you're writing your own subclass of Converter you must implement this method, however generally you should never need to call it or implement it.

### Creating a Converter
Creating a converter is generally done through composition. This is accomplished via the `pipe` and `default` methods on the Converter class.

```ts
import * as t from 'type-shift';

// use pipe to chain converters, pipe takes either another Converter or a function
// when passing a function it will be called with the value resulting from the previous
// converter.
const numberAsString = t.string.pipe(s => {
  // check that this is actually a number
  const n = Number(s);
  if (isNaN(n)) {
    // A converter error with appropriate data will be created for us
    // If we wanted to surface our own values for the expected and actual
    // values we could throw an error with those fields, such as an AssertionError
    throw new Error();
  }
  return n;
}, 'numberAsString');

// default specifies a value, (or converter) to use in place of a missing input
// the value is passed through the converter it is defined on so you should specify
// a default input to the converter, not a default result.
export const valueObject = t.strict({ value: numberAsString.default('0') });
```

### Built-in converters
The Following converters are shipped with Type Shift
- `never` fails to convert any value except the missing value
- `unknown` successfully converts any present value
- `number` successfully converts values of type number
- `string` successfully converts values of type string
- `boolean` successfully converts values of type boolean
- `null` successfully converts values equal to null
- `undefined` successfully converts values equal to undefined
- `literal(value)` successfully converts values equal to the given value using strict (===) equality
- `oneOf(...values)` successfully converts values equal to one of the given values using strict (===) equality

### Unions of Basic Types
You'll often build off of basic types such as `string`, `number`, and `boolean`. It is often useful to express that you can accept 2 or more of these types. For this purpose the basic types support an `or` join to create a union type.

```ts
import * as t from 'type-shift';

// trys to match string, then null. Fail if both fail.
const stringOrNull = t.string.or(t.null);
```

### Advanced
#### Nodes
You should generally never need to interact with the node class unless you're extending Converter.

```ts
import * as t from 'type-shift';

export class SetConverter<T> implements t.Converter<Set<T>, unknown> {
  public readonly name: string;
  private readonly elementConverter: t.Converter<T, unknown>;

  constructor(elementConverter: t.Converter<T, unknown>) {
    super();
    this.elementConverter = elementConverter;
    this.name = `Set<${elementConverter.name}>`;
  }

  public tryConvertNode(node: t.Node<unknown>): t.ConverterResult<t.Node<Set<T>>> {
    // nodes could be present or missing, check isMissingValue on the node or
    // use the ifPresent method to get a value
    return node.ifPresent(
      p => {
        // present nodes have a value
        if (p.value instanceof Set) {
          const results = Array.from(p.value.value()).map((v, i) => (
            // nodes can create a child node with the current node as a parent.
            // the first argument is either a Navigable Path element, string, or number.
            this.elementConverter.tryConvertNode(p.child(i, v))
          ));
          if (results.every(({ success }) => success)) {
            // setValue creates a new node with a new value but the same path and parent
            return t.success(p.setValue(
              new Set(results.filter(
                // whenever you have a node you may be dealing with
                // a missing node and need to handle that fact
                ({ value }) => !value.isMissingValue
              ).map(
                ({ value }) => value.value
              ));
            ));
          } else {
            return t.failed(...results.filter(
              ({ success }) => !success
            ).flatMap(
              ({ errors }) => errors
            ));
          }
        }
        return t.failed(
          // converter errors should encode that path and value data from a node
          new t.ConverterError(p.path, 'Set', p.value)
        );
      },
      m => t.failed(
        // If a node doesn't have a value just omit the argument in ConverterError
        new t.ConverterError(m.path, 'Set')
      )
    )
  }
}
```

## Concepts

### Node
Because Type Shift is about transforming data it is often useful to be able to pull data from other fields in an input. To enable this we represent the value passed to converters as a Node in a Tree. Nodes have a parent, value, and path. Converters can use the data in a Node to traverse the tree to find values. The path data encoded in the node is useful for ensuring that error messages are specific about what field is incorrect. Nodes represent either present or missing data, allowing them to encode the fact that a field that was requested on an input was not found, and allowing converters downstream to take appropriate action.

Nodes use **Navigation** to represent the part of the path they are at. Navigation is a way of encoding both the string representation of a path and how to get from one node to another. For instance the `RootNavigation` can walk up the tree of Nodes to reach the root, while the `DotNavigation` follows object properties.

### Converter
The Converter is the core of Type Shift. It provides a way to transform a Node of one type into a Node of another type. In the case of validation that may be transforming an unknown type into a known type or it may be transforming data in other ways in the case of a type adaptor. Beyond the methods related to converting values a converter provides the methods `pipe` and `default`. `pipe` is used to chain converters creating a new converter, `default` is used to specify an input in the case of a missing value.
