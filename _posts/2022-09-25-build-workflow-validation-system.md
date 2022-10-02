---
layout: post
title:  "Build a workflow validation system"
date:   2022-09-25
---

I'm currently working as a Lead core engineer at [Reelevant](https://reelevant.com), we're building a mar-tech product which help companies to generate individualized contents at scale.

Our customers have a large amount of data on their clients (tracking website navigation, shop purchases, recommendations…), we store those data for them and allow them to, based on data, generate the right and unique content for each client.

Those data are unique for each customer we have, which add complexity to using them, each source has is own fields, his own available filters… everything must be dynamic.

To help them generate and show the right content, we provide a workflow builder where they can build a complex workflow with their business logic, based on their data.

## Our workflow system

We've designed a workflow engine that is running workflows defined by a [Directed Acyclic Graph](https://en.wikipedia.org/wiki/Directed_acyclic_graph) where we have 3 different type of [vertices](https://en.wikipedia.org/wiki/Vertex_(graph_theory)):
- Data vertices in charge of retrieving data from our customer configured data sources
- Logic vertices in charge of splitting the workflow in multiple branch based on business logic (e.g. conditions on data, A/B testing...)
- Output vertices in charge of generating a content

Our workflow engine:
- is able to retrieve data with business logic defined by the customer (e.g. some filters on a specific category, get seen products...)
- is able to use data to retrieve more data (e.g. get a product recommendation based on the latest category seen by the client),
- is able to do condition on data (e.g. do we found a seen product? is the client seeing the content between 23 sept. and 26 sept.?)
- is able to use data to generate a content (e.g. show the latest seen product image)

## Our requirements

Since building a workflow could be quite technical and hard without right informations on the used data, our Workflow builder had to guide our customers and provide them useful insights on invalid behavior or on data structure.

To be as flexible as possible and build strictly valid workflow we wanted to have strict definition of our vertices (we call them node), what configuration do they need, which output do they produce if they are data nodes...

Having a strict definition for our nodes is allowing us to perform validation when the workflow is build, allowing to warn the user if is doing something wrong before even running the workflow.

## 1st step: Using Typescript

At Reelevant, our main language is Typescript, we use it in our backend and in our React frontend. It allows us to easily code share utils and typings and everyone in the tech team is able to work in both codebases.

The main idea was to be able to define our nodes with Typescript, since Typescript doesn't help us perform validation at runtime we've searched a runtime type system based on Typescript. And we've found [io-ts](https://github.com/gcanti/io-ts) which is quite used and was fulfilling our requirements (now there is also [zod](https://github.com/colinhacks/zod) which wasn't ready/known when we first started the project in early 2020). 

With io-ts we could easily define each node configuration that could be used to perform runtime validation.

```ts
import * as t from 'io-ts'

export class MyDataNode extends DataNode {
  public static readonly optionsType = t.type({
    dataSourceId: t.string,
    filters: t.array(t.type({
      field: t.string,
      operator: t.string,
      value: t.unknown
    }))
  })
}
```

## 2nd step: Type-checking node

Doing static validation is a first step in ensuring we have a valid workflow. The next step is be able to validate each node based on how it is configured. 

For example, with a given data node, if the user configured a specific data source, we want to ensure every filter he's configured are valid for this specific data source.

The way we do that, is to dynamically generate an io-ts type based on the current configuration of the node.
And since only some of our nodes have dynamic options type, that’s why using io-ts type for both is great because it allows us to rely on the same validation mechanism (through io-ts) whenever the node options type is dynamic or not

```ts
import * as t from 'io-ts'

export class MyDataNode extends DataNode {
  public static readonly optionsType = t.type({
    dataSourceId: t.string,
    filters: t.array(t.type({
      field: t.string,
      operator: t.string,
      value: t.unknown
    }))
  })

  constructor (private readonly options: t.TypeOf<typeof MyDataNode['optionsType']>) {}

  public async getDynamicOptionsType (): t.Mixed {
    const filters = await this.services.Datasource.fetchFilters({ id: this.options.dataSourceId })
    return t.type({
	  dataSourceId: t.literal(this.options.dataSourceId),
	  filters: t.array(
	    t.union(filters.map(filter => t.type({
	      field: t.literal(filter.field),
	      operator: t.union(filter.operators.map(t.literal)),
	      value: filter.type === 'string' ? t.string : t.number
	    })))
	  )
	})
  }
}
```

## 3rd step: Type-checking workflow

The next and the final step is to ensure that if a node is using another node (e.g. a data node using another one to retrieve some data), the used output is compatible with the expected config.

For this step, we do the same as for the config, we define our output type in node definition:

```ts
import * as t from 'io-ts'

export class MyDataNode extends DataNode {
  public static readonly optionsType = ...
  public static readonly outputType = t.array(t.record(t.string, t.union([t.string, t.number])))

  constructor (private readonly options: t.TypeOf<typeof MyDataNode['optionsType']>) {}

  public async getDynamicOptionsType (): t.Mixed {
    ...
  }

  public async getDynamicOutputType (): t.Mixed {
    const schema = await this.services.Datasource.fetchSchema({ id: this.options.dataSourceId })
    return t.array(t.type(schema.reduce<t.Props>((outputType, property) => {
      outputType[property.name] = property.type === 'string' ? t.string : t.number
      return outputType
    }, {})))
  }
}
```

This way we have an option type and an output type. So when a user use a data node for another data node, we can compare both io-ts type and see if they match (e.g. if the filter value is expected to be a `t.string`, the property from the output of the dependency should be compatible with `t.string`, so it could be `t.string` or `t.literal(<string>)`).

io-ts doesn’t provide a way to compare two io-ts type so we’ve built our own compare utility on top of io-ts types, using io-ts internals. Each io-ts type instance contain a `tag` property which identify the type. 
This information allows us to retrieve type definition and compare it with another type definition.

```ts
function areCompatibleTypes (expectedType: t.Mixed, foundType: t.Mixed): boolean {
  if (expectedType._tag === 'UnionType') {
    return expectedType.types.some(type => areCompatibleTypes(type, foundType))
  }
  if (foundType._tag === 'UnionType') {
    return foundType.types.every(type => areCompatibleTypes(expectedType, type))
  }
  if (expectedType._tag === 'StringType') {
    return foundType._tag === 'String' || (foundType._tag === 'LiteralType' && typeof foundType.value === 'string')
  }
  ...
  return false
}
```

The last part is being able to compare some properties of a type and not the full type itself.
For example, when we configure a node, we want to be able to validate the property we’re setting without validating the whole node which could be not entirely configured.
So we’ve built an util to retrieve the io-ts type of a property (even nested properties) using io-ts internals.

```ts
function getPropertyType (type: t.Mixed, path: string[]): t.Mixed | null {
  let currentType = type
  while (path.length > 0) {
    const key = path.shift()
    // Each io-ts type instance have a `_tag` property
    // which identify them. 
    // InterfaceType = t.type(...)
    if (currentType._tag === 'InterfaceType') {
      // For InterfaceType with have `props` which is
      // defined properties for the type
      if (typeof currentType.props[key] === 'undefined') {
        return null
      }
      currentType = currentType.props[key]
      continue
    }
    if (currentType._tag === 'UnionType') {
      // For UnionType, we have `types` which
      // is the list of io-ts type in the union
      // So we try to find union element where the key
      // exists.
      // And we build back an union with matching types
      const foundTypes = currentType.types
        .map(type => getPropertyType(type, [key]))
        .filter(type => type !== null)
      if (foundTypes.length === 0) {
        return null
      }
      currentType = foundTypes.length === 1 ? foundTypes : t.union(foundTypes)
      continue
    }
    ...
    return null
  }
  return currentType
}
```

We were now able to validate each workflow node options, and if a workflow node depends on one another, validate that the dependency is valid (the output type of the dependency must be compatible with the option type of the dependant). 

## Conclusion

Defining everything with io-ts allows us to perform strict and dynamic validation on our workflows, we're able to dynamically create io-ts type, assert two types are compatible or not and provide useful errors to our customers if they are doing something wrong.

Additionally since we’re able to detect compatible node for a given node, we can hide uncompatible ones to our customers and much more (e.g. errors detection, automatic configuration…) leading to a better UX for them. 
