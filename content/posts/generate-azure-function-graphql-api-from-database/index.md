---
title: 'Generate Azure Function GraphQL API From Database'
date: 2022-07-18T09:30:33+02:00
draft: false
cover:
  image: 'images/_cover.png'
  alt: 'Generate Azure Function GraphQL API From Database'
images:
  - 'images/_cover.png'
tags: ['Azure', 'Azure Function', 'TypeScript', 'JavaScript', 'Node.js']
---

I remember when I first used a GraphQL API in a React SPA. It was a totally different experience from the one I was used to. The flexibility and ease of API discovery with GraphiQL blew my mind. I was using and creating ODATA APIs by then and my first thought was these must be pretty much the same. In fact, the concept is similar. Give front-end developers an API that will allow them to make requests as they want, and get only the data that they want. However, the approach is very different.

Since then I had a few tries with building GraphQL APIs in .NET using [Hot Chocolate](https://chillicream.com/docs/hotchocolate) or [GraphQL .NET](https://graphql-dotnet.github.io/). None of them convinced me to change the approach to how I build my APIs.

Recently, I started a project where we are building the Node.js API that provides data from an existing SQL database. Our team chose to use [Prisma](https://www.prisma.io/) ORM to generate the database client. When I was looking for more information about this ORM I found out that it plays nicely with GraphQL servers and libraries. It can even **generate GraphQL schema and resolvers based on a database schema**. Then I recalled my past GraphQL fascination and decided to give it a shot.

In this article, I am building a PoC that even goes further. Let's make this API serverless by hosting it on Azure Function App.

[Here](https://github.com/rithala/graphql-az-function) you can find the source code of this demo app.

## Why do that?

Why not :) There are many use cases when you need to deliver quickly the API for existing data. Or you want to focus on the database design and the API just transports data back and forth between clients and a database. This is also a good starting point for developing a new API.

For sure in a real-world scenario, there is much more that has to be done like authorization, side effects or integrations with other systems.

## Prerequisites

- [Node.js LTS](https://nodejs.org/en/)
- [Azure Functions Core Tools](https://github.com/Azure/azure-functions-core-tools)
- Database, you can use the academic classic's [Northwind](https://github.com/Microsoft/sql-server-samples/tree/master/samples/databases/northwind-pubs)
- Azure CLI or Azure Functions VS Code extension to deploy the app to Azure

## Prepare database

First, you need a database. In this example, I use the Northwind sample database hosted in Azure. Here is the [official quickstart guide](https://docs.microsoft.com/en-us/azure/azure-sql/database/single-database-create-quickstart?view=azuresql&tabs=azure-portal). You can create a simple DTU-based basic single instance for a testing purpose. Remember to enable **SQL authentication** in the SQL server configuration. For now, Prisma does not support Azure AD authentication. Note down the **server name**, **database name**, **admin user name** and **password**. You will need this to create the connection URL.

Now you can populate the database. Copy the SQL from [Azure SQL Northwind sample](<https://raw.githubusercontent.com/microsoft/sql-server-samples/master/samples/databases/northwind-pubs/instnwnd%20(Azure%20SQL%20Database).sql>) and execute on the created database using SQL Management Studio, Azure Data Studio or just simply in Azure Portal using Query Editor in the database resource. If you are connecting for the first time you will be asked to whitelist the IP address in the Azure SQL server firewall settings.

![Northwind SQL script in the query editor](./images/img1.jpg#center)

The script execution may take a while. The created database schema should look like this.

![Created database schema](./images/img2.jpg#center)

## Azure Function project setup

I recommend using the [Azure Functions VS Code extension](https://docs.microsoft.com/en-us/azure/azure-functions/functions-develop-vs-code?tabs=nodejs) to create the project.

Press ```CTRL + SHIFT + P``` type ```az``` and select create a new project.
![Create an Azure Function project](./images/img3.jpg#center)
Use the current dictionary or select a different one.
![Create an Azure Function project](./images/img4.jpg#center)
Select TypeScript.
![Create an Azure Function project](./images/img5.jpg#center)
Now you can select the HTTP trigger function to create.
![Create an Azure Function project](./images/img6.jpg#center)
Provide the name.
![Create an Azure Function project](./images/img7.jpg#center)
This step is quite important. If you want to secure your API yourself or use the App Service [built-in authentication](https://docs.microsoft.com/en-us/azure/app-service/overview-authentication-authorization) (Easy Auth) then choose the "Anonymous" authorization level. In this demo, I selected the "Function" level for simplicity. You will use the function key in the "x-functions-key" header to authenticate requests.
![Create an Azure Function project](./images/img8.jpg#center)

## Installing dependencies

There is a list of libraries that have to be installed. The most important ones are:

- [prisma](https://www.npmjs.com/package/prisma) - previously described ORM
- [graphql@15](https://www.npmjs.com/package/graphql/v/15.8.0) - GraphQL language runtime. Version 15 is required by other used packages.
- [type-graphql](https://www.npmjs.com/package/type-graphql) - Creating GraphQL schema using TypeScript decorators.
- [typegraphql-prisma](https://www.npmjs.com/package/typegraphql-prisma) - Generating type-graphql classes based on Prisma model.
- [apollo-server-azure-functions](https://www.npmjs.com/package/apollo-server-azure-functions) - Host Apollo (popular GraphQL server) on an Azure Function App.
- [reflect-metadata](https://www.npmjs.com/package/reflect-metadata) - Metadata reflection polyfill.

You can use two following commands to install all required dependencies.

```bash
npm i apollo-server-azure-functions reflect-metadata graphql@15 graphql-fields graphql-scalars type-graphql class-validator @prisma/client
npm i -D prisma typegraphql-prisma @types/graphql-fields
```

## TypeScript configuration

TypeGraphQL package [requires additional TypeScript configuration](https://typegraphql.com/docs/installation.html) to work correctly.

So you need to change the `tsconfig.json` file from this:

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "target": "es6",
    "outDir": "dist",
    "rootDir": ".",
    "sourceMap": true,
    "strict": false
  }
}
```

To make it look like this:

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "outDir": "dist",
    "rootDir": ".",
    "sourceMap": true,
    "strict": false,
    "target": "es2018",
    "lib": ["es2018", "esnext.asynciterable"],
    "esModuleInterop": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```

## Prisma project initialization

Prisma needs its configuration. The easiest way to start is to run the CLI command.

```bash
npx prisma init --datasource-provider sqlserver
```

This creates two files. The `prisma/schema.prisma` schema file and `.env` for keeping the design-time database URL.

You should have kept the values that I mentioned during creating the database. Edit `.env` file using those values.

```
DATABASE_URL=sqlserver://{server name}.database.windows.net:1433;database={database name};user={admin user name};password={admin password};encrypt=true
```

Also, you need to change the Prisma schema file to include the GraphQL generator in it. Let's change `prisma/schema.prisma` to include this.

```
generator client {
  provider = "prisma-client-js"
}

generator typegraphql {
  provider = "typegraphql-prisma"
  output   = "../generated/type-graphql"
}

datasource db {
  provider = "sqlserver"
  url      = env("DATABASE_URL")
}
```

The next steps are to pull the database schema and generate the database client and GraphQL schema. Run these commands.

```bash
npx prisma db pull
npx prisma generate
```

If the command run successfully you should see many files generated under `generated/type-graphql` folder.

## GraphQL function

Finally, it is time to wire all things together and test the app. The first step is to make some changes to the function definition located in `GraphQL/function.json`.

```json
{
  "bindings": [
    {
      "authLevel": "function",
      "type": "httpTrigger",
      "direction": "in",
      "name": "req",
      "methods": ["get", "post", "options"]
    },
    {
      "type": "http",
      "direction": "out",
      "name": "$return"
    }
  ],
  "scriptFile": "../dist/GraphQL/index.js"
}
```

The difference between the default configurations is out the binding name changed to `$return` to take the output from the returned value, and added `options` to methods to support CORS.

Replace the `GraphQL/intex.ts` content with the following code.

```typescript
import 'reflect-metadata';
import { PrismaClient } from '@prisma/client';
import { buildSchema } from 'type-graphql';
import { AzureFunction, Context, HttpRequest } from '@azure/functions';
import { ApolloServer } from 'apollo-server-azure-functions';
import { resolvers } from '../generated/type-graphql';

async function getClient() {
  const prisma = new PrismaClient();
  await prisma.$connect();

  return prisma;
}

async function buildGraphQlSchema() {
  return await buildSchema({
    resolvers,
    validate: false,
  });
}

// Lazy initialization of Prisma client and GraphQL schema
const prismaClientPromise = getClient();
const graphqlSchemaPromise = buildGraphQlSchema();

async function buildApolloHandler() {
  const schema = await graphqlSchemaPromise;
  const prismaClient = await prismaClientPromise;

  const server = new ApolloServer({
    schema: schema,
    csrfPrevention: true,
    cache: 'bounded',
    // enabled introspection to test the deployed function in Apollo Studio Sandbox
    introspection: true,
    context: {
      prisma: prismaClient,
    },
  });

  return server.createHandler({
    disableHealthCheck: true,
  });
}

// Lazy initialization of Apollo Azure Function handler
const apolloHandlerPromise = buildApolloHandler();

export const graphqlHandler: AzureFunction = async function (
  context: Context,
  req: HttpRequest
): Promise<void> {
  // Resolve Apollo handler instance
  const handler = await apolloHandlerPromise;
  return await new Promise((resolve, reject) => {
    const contextWithPromisifiedDone = {
      ...context,
      // This hack is required as we require some async operation prior using handler.
      done: (err?: Error | string | null, result?: any) => {
        if (err) {
          reject(err);
          return;
        }
        resolve(result);
      },
    };

    // Run Apollo handler
    handler(contextWithPromisifiedDone, req);
  });
};
```

Ok, try to run the app.

```bash
npm run start
```

Unfortunately, quickly after the app starts the error message appears.

```log
UnhandledPromiseRejectionWarning: Error: Some errors occurred while generating GraphQL schema:
  Input Object type CustomerCustomerDemoUpdateManyMutationInput must define one or more fields.
    Input Object type EmployeeTerritoriesUpdateManyMutationInput must define one or more fields.
```

After short googling, you can find out this is related to Prisma's [issue#4004](https://github.com/prisma/prisma/issues/4004). For now, there is no well no workaround. You can try to automate finding empty input classes and add some dummy property. In this case, I added a dummy property to `CustomerCustomerDemoUpdateManyMutationInput.ts` and `EmployeeTerritoriesUpdateManyMutationInput.ts` manually.

```typescript
// generated/type-graphql/resolvers/inputs/CustomerCustomerDemoUpdateManyMutationInput.ts

import * as TypeGraphQL from 'type-graphql';
import * as GraphQLScalars from 'graphql-scalars';
import { Prisma } from '@prisma/client';
import { DecimalJSScalar } from '../../scalars';

@TypeGraphQL.InputType('CustomerCustomerDemoUpdateManyMutationInput', {
  isAbstract: true,
})
export class CustomerCustomerDemoUpdateManyMutationInput {
  @TypeGraphQL.Field((_type) => String, {
    nullable: false,
  })
  DoNotUseThisInputType!: string;
}
```

```typescript
// generated/type-graphql/resolvers/inputs/EmployeeTerritoriesUpdateManyMutationInput.ts

import * as TypeGraphQL from 'type-graphql';
import * as GraphQLScalars from 'graphql-scalars';
import { Prisma } from '@prisma/client';
import { DecimalJSScalar } from '../../scalars';

@TypeGraphQL.InputType('EmployeeTerritoriesUpdateManyMutationInput', {
  isAbstract: true,
})
export class EmployeeTerritoriesUpdateManyMutationInput {
  @TypeGraphQL.Field((_type) => String, {
    nullable: false,
  })
  DoNotUseThisInputType!: string;
}
```


Second try with ```npm run start``` and now without errors! Open the following URL in the browser ```http://localhost:7071/api/GraphQL```. You should be redirected to Apollo Studio Sandbox. Now you can explore the API schema.

![Apollo Studio Sandbox](./images/img9.jpg#center)

The generated API is quite rich with queries and mutations. The API supports full CRUD operations. You can get, create, update, remove, get rows with related entities, filter them or even do aggregations like COUNT, AVG, SUM, MAX or MIN.

![Apollo Studio Sandbox](./images/img10.jpg#center)

## Publish the function app to Azure

First, you need to create the function app in the resource group. I tested it on the Windows consumption plan only.

![Create Azure Function app parameters](./images/img11.jpg#center)

After the function is created go to the created resource and open the "Configuration" pane. You got to add the "**DATABASE_URL**" to the application settings. Take the value from the ```.env``` file.

![DATABASE_URL setting](./images/img12.jpg#center)

The last thing to do here is to select the "General settings" tab and change the platform to **64 Bit**. Prisma binaries only work with **64-bit** platforms. Remember to save changes before leaving the configuration.

![64-bit platform](./images/img13.jpg#center)

To use this API in Apollo Studio you have to change the CORS settings. Go to ```API -> CORS```, remove the default rule and add a new one.
![CORS settings](./images/img17.jpg#center)

As you are already here, go to the ```Functions -> App keys``` pane and note down the default key.

![Copy default app key](./images/img14.jpg#center)

It is time to go back to VS Code and deploy the code to Azure. Again, press ```CTRL + SHIFT + P``` type ```az``` and select ```Azure Functions: Deploy to Function App```. You might be asked to log in, then chose the target subscription and previously created function app. Confirm that you are aware that deployment will erase the target function app.

![Confirmation](./images/img15.jpg#center)

It takes a while to deploy the function app. When it is done you can test the deployed function in Apollo Studio Sandbox. Open ```https://studio.apollographql.com/sandbox/explorer``` in the browser and click the cogwheel button to open the sandbox settings. Change the endpoint to the deployed function URL and add the ```x-functions-key``` header with the copied value from ```Functions -> App keys```.

![Sandbox settings](./images/img16.jpg#center)

After saving changes you should be connected to the deployed function app.

## Summary

After spending a few minutes you got a fully working API with all CRUD methods and even more. It is just the beginning. You can extend the generated model with custom resolvers and role-based authorization and we are getting closer to a rich API build pretty quickly.



