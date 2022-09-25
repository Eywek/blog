---
layout: post
title:  "Build a workflow validation system"
date:   2022-09-25
categories: Typescript Workflow Engineering
---

# Build a workflow validation system

## Context

I'm currently working as a Lead core engineer at [Reelevant](https://reelevant.com), we're building a martech product which help companies to generate individualized contents at scale.

Our customers have a large amount of data on their clients (tracking website navigation, shop purchases, recommandations...), we store those data for them and allow them to, based on data, generate the right and unique content for each client.

To generate and show the right content, our customers needs to be able to build a workflow with their business logic, based on their data. And each data source is unique for each of our customer, each data source have his own fields and filter, everything must be dynamic.  

## Our workflow system

We've designed a workflow engine that run workflow defined by a [Directed Acyclic Graph](https://en.wikipedia.org/wiki/Directed_acyclic_graph) where we have 3 different type of [vertices](https://en.wikipedia.org/wiki/Vertex_(graph_theory)):
- Data vertices in charge of retrieving data from our customer configured data sources
- Logic vertices in charge of splitting the workflow in multiple branch based on business logic (i.e. conditions on data, A/B testing...)
- Output vertices in charge of generating a content

Our workflow engine:
- is able to retrieve data with business logic defined by the customer (i.e. some filters on a specific category, get seen products...)
- is able to use data to retrieve more data (i.e. get a product recommendation based on the latest category seen by the client),
- is able to do condition on data (do we found a seen product? is the client seeing the content between 23 sept. and 26 sept.?)
- is able to use data to generate a content (show the latest seen product image)

## What we wanted

Since our workflow engine is able to process some complex logic, we want to help our customers to build easily a workflow with a Workflow builder.

To be as flexible as possible and build strictly valid workflow we wanted to have strict definition of our vertices (we call them node), what configuration do they need, which output do they produce if they are data nodes...

Having a strict definition for our nodes is allowing us to perform validation when the workflow is build, allowing to warn the user if is doing something wrong before even running the workflow.

## 1st step: Using Typescript

At Reelevant, our main language is Typescript, we use it in our backend and in our React frontend. It allow us to easily code share utils and typings and everyone in the tech team is able to work in both codebases.

The main idea was to be able to define our nodes with Typescript, since Typescript doesn't help us to perform validation at runtime we've searched a runtime type system based on Typescript. And we've found [io-ts](https://github.com/gcanti/io-ts) which is quite used and was fulfilling our requirements (now there is also [zod](https://github.com/colinhacks/zod) which wasn't ready/known when we first started the project in early 2020). 

With io-ts we could easily define our node configurations and output and easily perform runtime validation on it. 

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

The way we do that is to dynamically generate an io-ts type based on the current configuration of the node, it allow us to leverage io-ts validation (our validation system is the same whenever we're validating statically the workflow or based on the configuration).

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

The next and the final step is to ensure that if a node is using another node (i.e. a data node using another one to retrieve some data), the used output is compatible with the expected config.

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

This way we have an option type and an output type. So when a user use a data node for another data node, we can compare both io-ts type and see if they match (i.e. if the filter value is expected to be a `t.string`, the property from the output of the dependency should be compatible with `t.string`, so it could be `t.string` or `t.literal(<string>)`).

The only issue is that io-ts don't provide a way to check if 2 type are compatible with each other. So we've built a `TypeManager` util to deal with io-ts type and perform validation on it.

For example, we have a method called `getPropertyType` which takes an io-ts type and a property path. The method will try to get the property type from the io-ts type:

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

And when we're able to retrieve a property type, we can use our `areCompatibleTypes` method to check if two type are compatible:

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

## Conclusion

Defining everything with io-ts allows us to perform strict and dynamic validation on our workflows, we're able to dynamically create io-ts type, assert two types are compatible or not and provide useful errors to our customers if they are doing something wrong.

Of course this whole type-checking system also allow us to forbid invalid configurations by showing to our customers only compatible configurations (i.e. we don't show an incompatible data source for a given filter).
