# contentful.js [![Build Status](https://travis-ci.org/contentful/contentful.js.png?branch=master)](https://travis-ci.org/contentful/contentful.js)

Javascript client for [Contentful's](https://www.contentful.com) Content Delivery API:

- [Documentation](https://www.contentful.com/developers/documentation/content-delivery-api)
- [Example Apps](http://contentful.github.io/contentful.js/example/)
- [Tests](https://github.com/contentful/contentful.js/tree/master/test/integration) running in node and browsers via [BrowserStack](http://browserstack.com)

Supported browsers/environments:

- Chrome
- Firefox
- IE10
- node.js >= 0.8

## Install

In node, using [npm](http://npmjs.org):

``` sh
npm install contentful
```

In a browser, using [bower](http://bower.io):

``` sh
bower install contentful
# After installing, add this as a script tag:
# <script src="components/contentful/dist/contentful.min.js"></script>
```

Latest [contentful.min.js](https://raw.github.com/contentful/contentful.js/master/dist/contentful.min.js).

Note: The next minor version release of `dist/contentful.min.js` will
be much smaller. Please use a package manager to keep your JS
dependencies up to date and get the newest version right when it's
ready!

## Functionality

### Supported

* .space() = get details of current space
* .contentTypes() = get content types of current space
* .entries() = get entries of current space
* .assets() = get entries of current space
* .sync() = get all the data in a space

## API

This API documentation uses [Flow](http://flowtype.org/) for documenting the
types of inputs & outputs.

### Common types

There are 2 types that are re-used throughout the API: `Link` and `Collection`.

#### Link type

A `Link` is a reference to another resource, it's used in resource metadata to
refer to the space (and possible content type) of the resource.

```js
declare class Link<LinkType> {
  sys: {
    type: "Link",
    id: string,
    linkType: LinkType
  }
}
```

#### Collection type

The second re-usable type is `Collection`. Methods with a plural name such as
`entries` and `assets` return a (promise for) a `Collection` of a given type.
This collection contains properties that can be used for pagination as well as
full representations of the resources themselves.

```js
declare class Collection<T> {
  sys: {
    type: "Array",
  },
  total: number,
  skip: number,
  limit: number,
  items: Array<T>
}
```

### `createClient(opts: ClientOpts): Client`

Creates a new `Client` instance using an API key and space ID.

```js
var contentful = require('contentful');

type ClientOpts = {
  // ID of Space
  space: string,

  // A valid access token within the Space
  accessToken: string,

  // Enable or disable SSL. Enabled by default.
  secure?: boolean,

  // Set an alternate hostname, Default is 'cdn.contentful.com'
  host?: string
}

declare class Client = {
  space(): Promise<Space>;
  sync(params: {initial?: boolean, nextSyncToken?: string}): Collection<{}>
  entry(id: string): Promise<Asset>;
  entries(params: {}): Promise<Asset>;
  asset(id: string): Promise<Asset>;
  assets(params: {}): Promise<Asset>;
  contentType(id: string): Promise<ContentType>;
  contentTypes(): Promise<Collection<ContentType>>;
}

var client: Client = contentful.createClient({
  space: 'cfexampleapi',
  accessToken: 'b4c0n73n7fu1',
});
```

### `Client#space() : Promise<Space>`

```js
client.space() //=> Promise<Space>

type Space = {
  sys: {
    type: "Space",
    id: string
  },
  name: string,
  locales: Array<{ code: string name: string }>
}
```

### `Client#entry(id: string): Promise<Entry>`

Fetch one entry by ID from the space.

```js
client.entry('nyancat') //=> Promise<Entry>

type DateTime = string // An ISO8601 timestamp

type Entry {
  sys: {
    type: "Entry",
    id: string,
    space: Link<"Space">,
    contentType: Link<"ContentType">,
    createdAt: DateTime,
    updatedAt: DateTime,
    revision: number
  },
  fields: {
    // Field names correspond to FieldDefinition#id
    [fieldName: string]: FieldValue
  }
}

type FieldValue
  = string 
  | number
  | boolean
  | { lat: number, lon: number } // Location fields
  | DateTime                     // Date strings may not have a time component
  | Entry
  | Asset
  | Array<string>
  | Array<Entry>
  | Array<Asset>
```

### `Client#entries(params?: {}): Promise<Collection<Entry>>`

Return a (possibly filtered) [collection](#collection-type) of entries. The
optional `params` argument should be an object containing URL query parameters
used to filter the entries. The available search parameters are documented in
the [Content Delivery API documentation][search-parameters].

```js
client.entries({content_type: 'cat'}) //=> Promise<Collection<Entry>>
```

### `Client#asset(id: string): Promise<Asset>`

Retrieve a single asset by ID.

```js
client.asset('nyancat') //=> Promise<Asset>

type Asset = { 
  sys: {
    type: "Asset",
    id: string,
    space: Link<"Space">,
    createdAt: DateTime,
    updatedAt: DateTime,
    revision: 1
  },
  fields: {
    title: string,
    description: string,
    file: {
      fileName: "nyancat.png",
      contentType: "image/png",
      details: {
        // only present if asset is an image
        image?: {
          width: 250,
          height: 250
        },
        size: 12273
      },
      url: string  // a protocol relative URL
    }
  }
}
```

### `Client#assets(params: {}): Promise<Collection<Asset>>`

Return a [collection](#collection-type) of assets. The parameters supplied will
become the query string of the API request, so e.g. `{query: "cat"}` becomes
`&query=cat`. The available search parameters are documented in the
[Content Delivery API documentation][search-parameters].

```js
client.assets() //=> Promise<Collection<Asset>>
```

[search-parameters]: http://docs.contentfulcda.apiary.io/#reference/search-parameters

### `Client#contentType(id: string) : Promise`

```js
client.contentType('cat') //=> Promise<ContentType>

type ContentType = {
  sys: {
    type: "ContentType",
    id: string
  },
  name: string,
  description: string,
  fields: Array<FieldDefinition>
}

type FieldDefinition = SimpleFieldDefinition | ArrayFieldDefinition

type SimpleFieldDefinition = {
  id: string,
  name: string,
  type: "Symbol" | "Text" | "Number" | "Integer" | "Date" | "Location" | "Link"
  linkType?: "Entry" | "Asset"
}

type ArrayFieldDefinition = {
  id: string,
  name: string,
  type: "Array",
  items: {
    type: "Symbol" | "Link",
    linkType?: "Entry" | "Asset"
  }
}
```

### `Client#contentTypes(): Collection<ContentType>`

Returns a [collection](#collection-type) containing all ContentTypes in the space.

```js
client.contentTypes() //=> Promise<Collection<ContentType>>
```

### `Client#sync(params: {initial?: boolean, nextSyncToken?: string}): SyncResponse

The sync method can be used to keep a local copy of your space content up to
date (using the [Sync API][sync-api]). 

```js
client.sync({initial: true}) //=> Promise<SyncResponse>

type SyncResponse = Collection<SyncItem> & { nextSyncToken: string }

type SyncItem = Entry | Asset | Deleted<"Entry"> | Deleted<"Asset">

declare class DeletedEntry {
  sys: {
    id: string
    type: "DeletedEntry",
    updatedAt: DateTime
  }
}

declare class DeletedAsset {
  sys: {
    id: string
    type: "DeletedAsset",
    updatedAt: DateTime
  }
}
```

[sync-api]: http://docs.contentfulcda.apiary.io/#reference/synchronization

## License

MIT
