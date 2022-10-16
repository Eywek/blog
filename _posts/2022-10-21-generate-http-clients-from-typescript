---
layout: post
title:  "Generate HTTP clients from TypeScript types"
date:   2022-10-21
---

I'm currently working as a Lead core engineer at [Reelevant](https://reelevant.com), we're building a martech product which help companies to generate individualized contents at scale.

We're using [TypeScript](https://www.typescriptlang.org/) as a main language in both ours backend and frontend codebase. 

## Our requirements

We wanted to be able to use TypeScript types defined in our backend APIs in our frontend application but our backend and frontend aren't in the same Github repository. 

We also wanted to be able to expose our APIs routes easily to frontend developers through an HTTP client, removing the need to search for an endpoint definition in our backend code base.

And the last nice to have we wanted to setup is exposing a documentation to our customers' developers ([example](https://reelevant.readme.io/reference/list-invitation)).

## OpenAPI

The easier way to generate HTTP clients our documentation from API definition is using the [OpenAPI specification](https://swagger.io/specification/) (previously called Swagger). 

The idea is to create an OpenAPI specification file where we can define every HTTP endpoint with their configuration (method, authentification, body, query parameters...) and responses (http codes, body...). Then, when the file is created, you can use it with many tools to generate HTTP client, documentation or even generate code.

The main issue with OpenAPI files is the maintenance, if you're creating a OpenAPI specification by hand, it means you must think to update it each time you update your API code. And it also means needing to define twice your entities types (openapi spec + typescript). 

## Generating OpenAPI from Typescript

To avoid this issue and iterate faster in our backend codebase, we chose to generate our OpenAPI specifications based on our TypeScript code. The idea is simple, we use the typescript compiler to read our TypeScript code, resolve our types used in our API endpoints, and generate a OpenAPI specification from it.

To generate those files, we use the [typoa](https://github.com/eywek/typoa) library.

## Generating HTTP clients from OpenAPI spec

The next step for us was to generate TypeScript HTTP clients based on the OpenAPI specifications, where we would retrieve our TypeScript types defined in our backend codebase. 

The idea is the same as the OpenAPI specification generation, we read our OpenAPI spec, and generate TypeScript code which define a class where methods are calling our backend endpoints with [axios](https://github.com/axios/axios). And we also generate TypeScript types based on OpenAPI components. 

To generate those clients, we use the [oatyp](https://github.com/eywek/oatyp) library.

## Automate everything

The last step was to automate everything, when someone commit on our backend codebase, we should create a new version of our OpenAPI client. 

First, when we commit on our backend codebase, a Docker image is built with built service, the Docker image also contains the OpenAPI specification file which was generated during the image build. 

To generate automatically clients for each commit, we need to get the OpenAPI specification file from the Docker image, and use [oatyp](https://github.com/eywek/oatyp) to build the HTTP client, and then publish it to NPM. 

Our workflow is the following:

![](https://blog.eywek.fr/assets/images/openapi-clients-schema.png)

We have a service called `release-observer` which receive webhooks when a commit is issued in our Github organization and is configured to commit to another repository. We primarily use this service to deploy our services to our Kubernetes cluster (when a commit on master is done, the service update Kubernetes manifests to pull the new service image).

We have a `openapi-clients` repository where we have yaml files defining which docker image we use to retrieve the OpenAPI file for each client:

```yaml
# openapi-clients/packages/<service name>/ref.yaml
image:
	repository: eu.gcr.io/<docker image path>
	tag: 9b8c3cba1b4f60ea265c47fa5ef043bc4ea97030
openapi:
	path: /path/to/spec/in/image/openapi.yaml
```

When we commit on our backend codebase, the release-observer pick the commit hash and commit in our `openapi-clients`  to update the `image.tag`, which triggers a Github Actions which generate the HTTP client and publish it to NPM.

## Conclusion

Generating OpenAPI specifications from our TypeScript codebase allows us to easily generate HTTP clients for each commit leveraging Github Actions. 

Having OpenAPI specifications also allows us to build a documentation that we can expose to our customer's developers. 

Everything is automatic and doesn't need our intervention which is really great and confortable to focus on business needs when the team is small.
