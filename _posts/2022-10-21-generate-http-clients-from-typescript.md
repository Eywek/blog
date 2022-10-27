---
layout: post
title:  "Generate HTTP clients from TypeScript types"
date:   2022-10-21
---

I'm currently working as a Lead core engineer at [Reelevant](https://reelevant.com), we're building a martech product which help companies to generate individualized contents at scale.

We're using [TypeScript](https://www.typescriptlang.org/) as a main language in both ours backend and frontend codebase. Each codebase was in a separate Github repository.

Few years ago, for our frontend engineers to use a newly created or updated API endpoint, they needed to look into the backend codebase to search for API types and endpoint. 

At this time, it wasn't a great developer experience and we wanted to simplify this process by exposing HTTP clients to our frontend engineers where they could simply call methods with right TypeScript types.

Additionally, at the same time, one of our business requirement was to expose our public API endpoints to our customers. We needed to set-up an up-to-date documentation. Writing it our-self and keeping up to date wasn't really something we've considered since we didn't have many resources and we needed to move quickly.

## OpenAPI

The easier and most common way to generate HTTP clients or documentation from an API definition is using the [OpenAPI specification](https://swagger.io/specification/) (previously called Swagger). 

The idea is to create an OpenAPI specification file where we can define every HTTP endpoint with their configuration (method, authentication, body, query parameters...) and responses (http codes, body...). Then, when the file is created, you can use it with many tools to generate HTTP client, documentation or even generate code.

The main issue we had with OpenAPI files is the maintenance, if you're creating an OpenAPI specification by hand, it means you must think to update it each time you update your API code. And it also means needing to define twice your entities types (openapi spec + typescript). 

## Generating OpenAPI from Typescript

To avoid this issue and iterate faster in our backend codebase, we chose to generate our OpenAPI specifications based on our TypeScript code. The idea is simple, we use the typescript compiler to read our TypeScript code, resolve our types used in our API endpoints, and generate an OpenAPI specification from it.

To generate those files, we use the [typoa](https://github.com/eywek/typoa) library.

## Generating HTTP clients from OpenAPI spec

The next step for us was to generate TypeScript HTTP clients based on the OpenAPI specifications, where we would retrieve our TypeScript types defined in our backend codebase. 

The idea is the same as the OpenAPI specification generation, we read our OpenAPI spec, and generate TypeScript code which define a class where methods are calling our backend endpoints with [axios](https://github.com/axios/axios). And we also generate TypeScript types based on OpenAPI components. 

To generate those clients, we use the [oatyp](https://github.com/eywek/oatyp) library.

## Automate everything

The last step was to automate everything, when someone commit on our backend codebase, we should create a new version of our OpenAPI client. 

Currently our setup was building a Docker image -for each service- when a commit is landed on master. And we had a service called `release-observer` which receive webhooks when a commit is issued in our Github organization and is configured to commit to another repository. We primarily use this service to deploy our services to our Kubernetes cluster (when a commit on master is done, the service update Kubernetes manifests to pull the new service image).

Our purpose here was to use this current setup as much as possible. And that's what we've done by adding the built OpenAPI specification file to each our Docker image. 

This way, we were able to retrieve an OpenAPI file for each service version easily. 

We also had setup a `openapi-clients` repository where we have yaml files defining which docker image we use to retrieve the OpenAPI file for each client:
```yaml
# openapi-clients/packages/<service name>/ref.yaml
image:
  repository: eu.gcr.io/<docker image path>
  tag: 9b8c3cba1b4f60ea265c47fa5ef043bc4ea97030
openapi:
  path: /path/to/spec/in/image/openapi.yaml
```

Updating this yaml file is triggering a GitHub Action which retrieve the OpenAPI file in the docker image, build the HTTP client and publish it to NPM.

And then, we've configured our `release-observer` service to update this yaml file when a commit is landed on master.

Which leads to the following workflow:

![](/assets/images/openapi-clients-schema.png)

## Conclusion

Generating OpenAPI specifications from our TypeScript codebase allows us to easily generate HTTP clients for each commit leveraging Github Actions. 

Having OpenAPI specifications also allows us to build a documentation that we can expose to our customer's developers. 

Everything is automatic and doesn't need our intervention which is really great and comfortable to focus on business needs when the team is small.
