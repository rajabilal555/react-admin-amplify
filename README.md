# React Admin Amplify ![GitHub release (latest by date)](https://img.shields.io/github/v/release/MrHertal/react-admin-amplify) ![GitHub Workflow Status](https://img.shields.io/github/workflow/status/MrHertal/react-admin-amplify/Node.js%20CI)

AWS Amplify data provider for [react-admin](https://github.com/marmelab/react-admin).

- [Installation](#installation)
- [Usage](#usage)
- [Options](#options)
- [Pagination](#pagination)
- [Filter](#filter)
- [Sorting](#sorting)
- [Storage](#storage)
- [Admin queries](#admin-queries)

This library contains the data and auth providers that connect a [react-admin](https://github.com/marmelab/react-admin) frontend to an [Amplify](https://docs.amplify.aws) backend. It also includes some components that make things easier to set up.

A demo is available here: <https://master.d3os44oci7szj2.amplifyapp.com>. It demonstrates the use of this library with the [17 patterns GraphQL schema](https://docs.amplify.aws/cli/graphql-transformer/dataaccess).

Demo source code is here: <https://github.com/MrHertal/react-admin-amplify-demo>.

## How does it work

The data provider accepts GraphQL queries and mutations as parameters.
Queries and mutations are the one generated by the [Amplify CLI](https://docs.amplify.aws/cli/graphql/client-code-generation/).

Based on the resource that is required, the data provider is able to choose the right query and to fetch the data.
GraphQL queries are executed with the [Amplify GraphQL client](https://docs.amplify.aws/lib/graphqlapi/query-data/q/platform/js).

On the other hand, the auth provider uses the [Amplify Auth library](https://docs.amplify.aws/lib/auth/emailpassword/q/platform/js) to manage users sign-in and sign-out.

## Before installation

Please note that your Amplify backend, meaning the `amplify/` folder containing your GraphQL schema, can be located in a different repo than the react-admin one.

Starting from a [react-admin](https://marmelab.com/react-admin/Tutorial.html) project, install the Amplify library:

```sh
npm install aws-amplify
```

You will need the configuration file `aws-exports.js` of your Amplify backend, so that react-admin can connect to your API.

Finally, you will need the `queries.js` and `mutations.js` files generated by the [Amplify CLI](https://docs.amplify.aws/cli/graphql/client-code-generation/).

## Installation

```sh
npm install react-admin-amplify
```

## Usage

Simplest way to set things up is to use the `AmplifyAdmin` component:

```jsx
// in App.js
import { Amplify } from "aws-amplify";
import { Resource } from "react-admin";
import { AmplifyAdmin } from "react-admin-amplify";
import awsExports from "./aws-exports";
import * as mutations from "./graphql/mutations";
import * as queries from "./graphql/queries";

Amplify.configure(awsExports); // Configure Amplify the usual way

function App() {
  return (
    <AmplifyAdmin // Replace the Admin component of react-admin
      operations={{ queries, mutations }} // Pass the queries and mutations
      options={{ authGroups: ["admin"] }} // Pass the options
    >
      <Resource name="orders" />
      {/* Set the resources as you would do within Admin component */}
    </AmplifyAdmin>
  );
}

export default App;
```

Data and auth providers can also be set independantly using `buildDataProvider` or `buildAuthProvider`.

Code above is the equivalent of:

```jsx
// in App.js
import { Amplify } from "aws-amplify";
import { Admin, Resource } from "react-admin";
import { buildAuthProvider, buildDataProvider } from "react-admin-amplify";
import awsExports from "./aws-exports";
import * as mutations from "./graphql/mutations";
import * as queries from "./graphql/queries";

Amplify.configure(awsExports);

function App() {
  return (
    <Admin
      authProvider={buildAuthProvider({ authGroups: ["admin"] })}
      dataProvider={buildDataProvider({ queries, mutations })}
    >
      <Resource name="orders" />
    </Admin>
  );
}

export default App;
```

## Options

### Auth provider

`authGroups`: array of user groups, default: `[]`

Restrict access of your react-admin app to users belonging to one of these groups.

For example:

`authGroups: ["admin"]` - only users belonging to Cognito group `admin` will be able to sign in.

### Data provider

`authMode`: string, default: `AMAZON_COGNITO_USER_POOLS`

Authorization mode used by the Amplify GraphQL client.

`storageBucket`: string, optional

S3 bucket if using Storage, [see below](#storage).

`storageRegion`: string, optional

S3 region if using Storage, [see below](#storage).

`enableAdminQueries`: boolean, default: `false`

Enables managing Cognito users and groups, [see below](#admin-queries).

## Features

This section details some features of the library but also some limitations.

### Pagination

Total count is not supported by Amplify, see <https://github.com/aws-amplify/amplify-cli/issues/1865>.

That means that react-admin default pagination does not suit well. I suggest implementing a prev/next pagination like the one described in react-admin [documentation](https://marmelab.com/react-admin/ListTutorial.html#building-a-custom-pagination).

### Filter

In order to use react-admin filters, you will have to correctly set [@key directives](https://docs.amplify.aws/cli/graphql-transformer/key) in your schema.

Let's say you have a GraphQL schema that defines a type `Order`:

```graphql
type Order @model {
  id: ID!
  customerID: ID!
  accountRepresentativeID: ID!
  productID: ID!
  status: String!
  amount: Int!
  date: String!
}
```

To list orders in react-admin, you define a resource called `orders`:

```jsx
<Resource name="orders" list={OrderList} />
```

Data provider will execute the query `listOrders` by default, when no filters are applied:

```js
export const listOrders = /* GraphQL */ `
  query ListOrders(
    $filter: ModelOrderFilterInput
    $limit: Int
    $nextToken: String
  ) {
    listOrders(filter: $filter, limit: $limit, nextToken: $nextToken) {
      items {
        id
        customerID
        accountRepresentativeID
        productID
        status
        amount
        date
        createdAt
        updatedAt
      }
      nextToken
    }
  }
`;
```

Now you want to filter orders by product. You may think about passing a `$filter` argument to that query. Unfortunately, this would only filter the results after query has been executed.

You need to configure index structures in order to do that, using the `@key` directive:

```graphql
type Order
  @model
  @key(
    name: "byProduct"
    fields: ["productID", "id"]
    queryField: "ordersByProduct"
  ) {
  id: ID!
  customerID: ID!
  accountRepresentativeID: ID!
  productID: ID!
  status: String!
  amount: Int!
  date: String!
}
```

Amplify CLI will generate the query `ordersByProduct`:

```js
export const ordersByProduct = /* GraphQL */ `
  query OrdersByProduct(
    $productID: ID
    $id: ModelIDKeyConditionInput
    $sortDirection: ModelSortDirection
    $filter: ModelOrderFilterInput
    $limit: Int
    $nextToken: String
  ) {
    ordersByProduct(
      productID: $productID
      id: $id
      sortDirection: $sortDirection
      filter: $filter
      limit: $limit
      nextToken: $nextToken
    ) {
      items {
        id
        customerID
        accountRepresentativeID
        productID
        status
        amount
        date
        createdAt
        updatedAt
      }
      nextToken
    }
  }
`;
```

Finally in your react-admin app, set the filter this way:

```jsx
const OrderFilter = (props) => (
  <Filter {...props}>
    <TextInput
      source="ordersByProduct.productID"
      label="Product id"
      alwaysOn
      resettable
    />
  </Filter>
);
```

The source `ordersByProduct.productID` tells the data provider to execute `ordersByProduct` query, passing filter value as `productID` parameter.

#### AmplifyFilter

Things become more complex when you want to add several filters.

Let's say that you want to add another filter to the order resource. You want to be able to filter orders by customer and by date:

```graphql
type Order
  @model
  @key(
    name: "byProduct"
    fields: ["productID", "id"]
    queryField: "ordersByProduct"
  )
  @key(
    name: "byCustomerByDate"
    fields: ["customerID", "date"]
    queryField: "ordersByCustomerByDate"
  ) {
  id: ID!
  customerID: ID!
  accountRepresentativeID: ID!
  productID: ID!
  status: String!
  amount: Int!
  date: String!
}
```

In your react-admin app, filters are set this way:

```jsx
const OrderFilter = (props) => (
  <Filter {...props}>
    <TextInput
      source="ordersByProduct.productID"
      label="Product id"
      alwaysOn
      resettable
    />
    <TextInput
      source="ordersByCustomerByDate.customerID"
      label="Customer id"
      alwaysOn
      resettable
    />
    <DateInput source="ordersByCustomerByDate.date.eq" label="Date" alwaysOn />
  </Filter>
);
```

Please note that `date` field is a sort key, so you need to specify an operator to the query (`eq` in this example).

These filters may be confusing for the users because they would expect to filter orders by product, customer and date at the same time.
You need to hide product filter when customer filter is being used, because the query executed is `ordersByCustomerByDate` and not `ordersByProduct`.

`AmplifyFilter` component solves this issue by displaying or hiding filters automatically:

```jsx
import { AmplifyFilter } from "react-admin-amplify";

const OrderFilter = (props) => (
  <AmplifyFilter {...props}>
    <TextInput
      source="ordersByProduct.productID"
      label="Product id"
      alwaysOn
      resettable
    />
    <TextInput
      source="ordersByCustomerByDate.customerID"
      label="Customer id"
      alwaysOn
      resettable
    />
    <DateInput source="ordersByCustomerByDate.date.eq" label="Date" alwaysOn />
  </AmplifyFilter>
);
```

Check the demo to see it in action: <https://master.d3os44oci7szj2.amplifyapp.com>.

Demo source code is here: <https://github.com/MrHertal/react-admin-amplify-demo>.

### Sorting

Sorting data is possible with the sort key. Since default list queries (like `listOrders`) have no sort key, you cannot sort them.
Similarly to filters, sorting is based on [@key directives](https://docs.amplify.aws/cli/graphql-transformer/key) set in the GraphQL schema.

Let's look at `Order` schema again:

```graphql
type Order
  @model
  @key(
    name: "byProduct"
    fields: ["productID", "id"]
    queryField: "ordersByProduct"
  )
  @key(
    name: "byCustomerByDate"
    fields: ["customerID", "date"]
    queryField: "ordersByCustomerByDate"
  ) {
  id: ID!
  customerID: ID!
  accountRepresentativeID: ID!
  productID: ID!
  status: String!
  amount: Int!
  date: String!
}
```

With such a configuration, sorting by `id` is possible only when filtering orders by product, whereas sorting by date is only possible when filtering orders by customer.

In order to tell your react-admin app, you have to specify the query name in the `sortBy` prop:

```jsx
export const OrderList = (props) => {
  return (
    <List {...props} filters={<OrderFilter />}>
      <Datagrid>
        <TextField source="id" sortBy="ordersByProduct" sortable={true} />
        <DateField source="date" sortBy="ordersByCustomerByDate" sortable={true} />
      </Datagrid>
    </List>
  );
};
);
```

Just like filters, it is better for users to only allow sorting when it is available. To do that, you have to change dynamically the `sortable` prop, depending on the filter that is applied.

See a working example [on the demo](https://github.com/MrHertal/react-admin-amplify-demo/blob/master/src/components/Order.js).

### Storage

You can use Amplify Storage with that library to manage user files.

First [configure storage](https://docs.amplify.aws/lib/storage/getting-started/q/platform/js) in your Amplify project.

You will need to update your API schema to save files, for example:

```graphql
type User @model {
  id: ID!
  username: String!
  picture: S3Object
  documents: [S3Object!]
}

type S3Object {
  bucket: String!
  region: String!
  key: String!
}
```

`S3Object` is mandatory for the data provider to properly work.

You can then pass the S3 bucket and region to the data provider:

```jsx
// in App.js
import { Amplify } from "aws-amplify";
import { Resource } from "react-admin";
import { AmplifyAdmin } from "react-admin-amplify";
import awsExports from "./aws-exports";
import * as mutations from "./graphql/mutations";
import * as queries from "./graphql/queries";

Amplify.configure(awsExports);

function App() {
  return (
    <AmplifyAdmin
      operations={{ queries, mutations }}
      options={{
        authGroups: ["admin"],
        storageBucket: awsExports.aws_user_files_s3_bucket,
        storageRegion: awsExports.aws_user_files_s3_bucket_region,
      }}
    >
      <Resource name="orders" />
    </AmplifyAdmin>
  );
}

export default App;
```

#### Amplify inputs

```jsx
import { AmplifyFileInput, AmplifyImageInput } from "react-admin-amplify";

// ...

export const UserCreate = (props) => (
  <Create {...props}>
    <SimpleForm>
      <AmplifyImageInput source="picture" accept="image/*" />
      <AmplifyFileInput
        source="documents"
        accept="application/pdf"
        multiple={true}
        storageOptions={{ level: "private" }}
      />
    </SimpleForm>
  </Create>
);
```

`AmplifyImageInput` and `AmplifyFileInput` components accept same props as [ImageInput](https://marmelab.com/react-admin/ImageInput.html) and [FileInput](https://marmelab.com/react-admin/FileInput.html).

An additional prop `storageOptions` is available and is passed to [Storage.put](https://docs.amplify.aws/lib/storage/upload/q/platform/js).

#### Amplify fields

```jsx
import { AmplifyFileField, AmplifyImageField } from "react-admin-amplify";

// ...

export const UserShow = (props) => (
  <Show {...props}>
    <SimpleShowLayout>
      <AmplifyImageField source="picture" title="Avatar" addLabel={true} />
      <AmplifyFileField
        source="documents"
        storageOptions={{ level: "private" }}
        addLabel={true}
      />
    </SimpleShowLayout>
  </Show>
);
```

`AmplifyImageField` and `AmplifyFileField` components accept same props as [ImageField](https://marmelab.com/react-admin/ImageField.html) and [FileField](https://marmelab.com/react-admin/FileField.html).

An additional prop `storageOptions` is available and is passed to [Storage.get](https://docs.amplify.aws/lib/storage/download/q/platform/js).

### Admin queries

[Admin queries](https://docs.amplify.aws/cli/auth/admin) allow us to manage users and groups of a Cognito user pool. For example, you can list all signed up users in your react-admin app.

First [configure admin queries](https://docs.amplify.aws/cli/auth/admin#enable-admin-queries) in your Amplify project.

Don't forget to update the configuration file `aws-exports.js` if it was imported from another project.

Then you have to set the data provider option `enableAdminQueries`:

```jsx
// in App.js
import { Amplify } from "aws-amplify";
import { Resource } from "react-admin";
import { AmplifyAdmin } from "react-admin-amplify";
import awsExports from "./aws-exports";
import * as mutations from "./graphql/mutations";
import * as queries from "./graphql/queries";

Amplify.configure(awsExports);

function App() {
  return (
    <AmplifyAdmin
      operations={{ queries, mutations }}
      options={{
        authGroups: ["admin"],
        enableAdminQueries: true,
      }}
    >
      <Resource name="orders" />
    </AmplifyAdmin>
  );
}

export default App;
```

It tells the data provider to call the admin queries API when requested resources are `cognitoUsers` or `cognitoGroups`.

You can then add these two resources:

```jsx
// in App.js
import { Amplify } from "aws-amplify";
import { Resource } from "react-admin";
import {
  AmplifyAdmin,
  CognitoGroupList,
  CognitoUserList,
  CognitoUserShow,
} from "react-admin-amplify";
import awsExports from "./aws-exports";
import * as mutations from "./graphql/mutations";
import * as queries from "./graphql/queries";

Amplify.configure(awsExports);

function App() {
  return (
    <AmplifyAdmin
      operations={{ queries, mutations }}
      options={{
        authGroups: ["admin"],
        enableAdminQueries: true,
      }}
    >
      <Resource
        name="cognitoUsers"
        options={{ label: "Cognito Users" }}
        list={CognitoUserList}
        show={CognitoUserShow}
      />
      <Resource
        name="cognitoGroups"
        options={{ label: "Cognito Groups" }}
        list={CognitoGroupList}
      />
    </AmplifyAdmin>
  );
}

export default App;
```

`CognitoUserList`, `CognitoUserShow` and `CognitoGroupList` are provided by this library to help you quickly setting things up. You can replace them by your own components if you want to add some customizations.

## License

MIT
