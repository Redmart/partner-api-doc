---
title: API Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - shell
  - scala

toc_footers:
  - RedMart Partner API <i>version 0.3.0</i>
  - <a href='https://github.com/lord/slate'>Documentation Powered by Slate</a>

includes:
  - errors

search: true
---

# Introduction

This is the official documentation of RedMart's Marketplace **Partner API**. Using this private API, you'll be able to manage your inventory, products, orders, pickups and more!

We provide language bindings in Shell and Scala. You can view code samples in the dark pane on the right. Switch the programming language with the tabs at the top.

## Changelog

We will list any changes to the current version of the API here.

| Date       | Details of changes                                                                                               |
| ---------- | ---------------------------------------------------------------------------------------------------------------- |
| 2023-08-03 | Adds [Pickup Jobs](#pickup-jobs) API                                                                             |
|            | Adds new [scope](#scopes) for pickup jobs query                                                                  |
|            | Adds new [rate limits](#rate-limiting) for pickup jobs query                                                     |
| 2018-07-06 | Pre-release of RedMart Partner API Document Version 1                                                            |
| 2018-07-13 | Renames _Stocks_ as **Stock Lots**                                                                               |
|            | Identifies each Stock Lot by their **new `id` field** (rather than the previously used `availableForPickupFrom`) |
|            | Adds the error response code **409 Conflict** in [Update one Stock Lot](#update-one-stock-lot)                   |
| 2018-08-02 | Adds OAuth 2 [scopes](#scopes) to all endpoints to manage access rights of applications                          |
|            | Lists all available [Environments](#environments)                                                                |
| 2018-08-28 | Adds [Rate Limiting](#rate-limiting) section                                                                     |

## Environments

We currently provide the Partner API in one environment, **Production**.

| Environment | Hostname                 |
| ----------- | ------------------------ |
| Production  | partners-api.redmart.com |

# Registration

The Partner API uses the [OAuth2 authorization framework](https://tools.ietf.org/html/rfc6749) and supports the [Client Credentials](https://tools.ietf.org/html/rfc6749#section-4.4) flow.

In a nutshell :

1.  Contact RedMart Partner Support at <a href="mailto:rm_partnersupport@care.lazada.com">rm_partnersupport@care.lazada.com</a> to register your Client Application
2.  Save (securely) your CLIENT_ID and CLIENT_SECRET into your Client Application's database
3.  Client Application requests an Access Token from Partner API using its CLIENT_ID and CLIENT SECRET
4.  Client Application uses the Access Token to call Partner API endpoints
5.  Once Client Application starts receiving [_401 Unauthorized_](https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10.4.2) responses to its calls, it means the Access Token has expired. It then needs to request another Access Token (step 3)

# Request a Token

<a id="request-a-token"></a>

> To request a token, use this code:

```shell

curl --include -X POST \
     "https://{HOSTNAME}/oauth2/token" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     --data "grant_type=client_credentials" \
     --data "client_id={MY_CLIENT_ID}" \
     --data "client_secret={MY_CLIENT_SECRET}" \
     --data "scope={scopes}"
```

```scala
// TODO
```

> Make sure to replace `{MY_CLIENT_ID}` and `{MY_CLIENT_SECRET}` with the actual values provided during Registration and `{scopes}` with a space-separated list of [scopes](#scopes).

The Client Application makes a request to the `/oauth2/token` endpoint by sending the following parameters in the "application/x-www-form-urlencoded" format

| Parameter     | Required | Description                                                                           |
| ------------- | -------- | ------------------------------------------------------------------------------------- |
| client_id     | true     | The Client ID provided during Registration                                            |
| client_secret | true     | The Client Secret provided during Registration                                        |
| grant_type    | true     | The value MUST be set to "client_credentials"                                         |
| scope         | true     | A space-separated list of [scopes](#scopes), e.g. `read:product read:pickup-location` |

> [200 OK](https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10.2.1) response:

```json
{
  "token_type": "bearer",
  "access_token": "{access-token}",
  "expires_in": 7200 // this is in seconds
}
```

## Scopes

To call any endpoint of this API, your access token needs to have access to the scope that this specific endpoint requires. As a best practice, you should _always_ only request the smallest possible set of scopes for your application to work. This ensures the smallest possible impact in case anything goes wrong (see also [Principle of least privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege)).

| Scope                | Scope Description                                  |
| -------------------- | -------------------------------------------------- |
| read:pickup-location | Grants access to view details of a pickup location |
| read:product         | Grants access to view details of a product         |
| read:stock-lot       | Grants access to view stock lots of a product      |
| write:stock-lot      | Grants access to update stock lots of a product    |
| read:pickup-job      | Grants access to view pickup jobs of a store       |

# Rate Limiting

Each endpoint of this API limits how many times you can call it per second. Below is a summary of all existing endpoints and their respective rate limit.

| Endpoint                                              | Max number of calls per second |
| ----------------------------------------------------- | ------------------------------ |
| [Get all Pickup Locations](#get-all-pickup-locations) | 10                             |
| [Get one Pickup Location](#get-one-pickup-location)   | 10                             |
| [Get all Products](#get-all-products)                 | 10                             |
| [Get one Product](#get-one-product)                   | 10                             |
| [Get all Stock Lots](#get-all-stock-lots)             | 10                             |
| [Get one Stock Lot](#get-one-stock-lot)               | 10                             |
| [Update one Stock Lot](#update-one-stock-lot)         | 10                             |
| [Get Pickup Jobs](#get-pickup-jobs)                   | 1                              |
| [Get one Pickup Job](#get-one-pickup-job)             | 10                             |

Each (successful) response from the above endpoints contains 2 headers you can monitor to help control your rate

| http Header                  | Example Value | Description                                                                                 |
| ---------------------------- | ------------- | ------------------------------------------------------------------------------------------- |
| X-RateLimit-Limit-second     | 10            | max number of calls per second allowed for this particular endpoint. Always the same value. |
| X-RateLimit-Remaining-second | 9             | max _remaining_ number of calls allowed in the current second for this particular endpoint. |

If you exceed the rate limit of a particular endpoint, it'll keep responding http code [429](https://tools.ietf.org/html/rfc6585#section-4) until the current second is passed.

# Pickup Locations

A Product can be picked from within one Pickup Location.
A Marketplace Seller can have several Pickup Locations.

<aside class="warning">
To perform all <b>Pickup Locations</b> operations, you must always send your oauth2 access-token (see the <a href="#request-a-token">Request a Token</a> section).
Look at the 'Authorization' header in the code samples.
</aside>

## Get all Pickup Locations

<a id="get-all-pickup-locations"></a>

> Code samples

```shell

curl --include -X GET \
     "https://{HOSTNAME}/v1/pickup-locations?page=1&pageSize=50" \
     -H 'Accept: application/json' \
     -H 'Authorization: Bearer {access-token}'

```

```scala
// TODO
```

`GET /v1/pickup-locations`

_Querying and filtering pickup locations_

<h3 id="getpickup-locations-parameters">Query Parameters</h3>

| Parameter | In    | Required | Default | Description                                          |
| --------- | ----- | -------- | ------- | ---------------------------------------------------- |
| page      | query | false    | 1       | The page number, must be >= 1                        |
| pageSize  | query | false    | 50      | Number of items on one page, must be >= 1 and <= 100 |

> 200 Response

```json
{
  "page": 1,
  "pageSize": 50,
  "items": [
    {
      "addressLine1": "1 Main Street",
      "city": "Singapore",
      "name": "ABC Merchant",
      "country": "Singapore",
      "postalCode": "123456",
      "id": "23789",
      "addressLine2": "#02-15"
    }
  ],
  "total": 1
}
```

<h3 id="getpickup-locations-responses">Responses</h3>

| Status | Meaning                                                 | Description | Schema                                              |
| ------ | ------------------------------------------------------- | ----------- | --------------------------------------------------- |
| 200    | [OK](https://tools.ietf.org/html/rfc7231#section-6.3.1) | OK          | [Page*PickupLocation*](#schemapage_pickuplocation_) |

<aside class="warning">
To perform this operation, you must be authenticated by means of one of the following methods:
OAuth2 ( Scopes: read:pickup-location )
</aside>

## Get one Pickup Location

<a id="get-one-pickup-location"></a>

> Code samples

```shell

curl --include -X GET \
     "https://{HOSTNAME}/v1/pickup-locations/{pickupLocationId}" \
     -H 'Accept: application/json' \
     -H 'Authorization: Bearer {access-token}'

```

```scala
// TODO
```

`GET /v1/pickup-locations/{pickupLocationId}`

_Querying the details of a specific pickup location_

<h3 id="getpickup-locations-pickuplocationid-parameters">Path Parameters</h3>

| Parameter        | In   | Required | Description                                                                                |
| ---------------- | ---- | -------- | ------------------------------------------------------------------------------------------ |
| pickupLocationId | path | true     | The unique Identifier of the pickup-location, in the response example it's the value 23789 |

> 200 Response

```json
{
  "addressLine1": "1 Main Street",
  "city": "Singapore",
  "name": "ABC Merchant",
  "country": "Singapore",
  "postalCode": "123456",
  "id": "23789",
  "addressLine2": "#02-15"
}
```

> 404 Response

```json
{
  "title": "The requested resource could not be found"
}
```

<h3 id="getpickup-locations-pickuplocationid-responses">Responses</h3>

| Status | Meaning                                                        | Description | Schema                                  |
| ------ | -------------------------------------------------------------- | ----------- | --------------------------------------- |
| 200    | [OK](https://tools.ietf.org/html/rfc7231#section-6.3.1)        | OK          | [PickupLocation](#schemapickuplocation) |
| 404    | [Not Found](https://tools.ietf.org/html/rfc7231#section-6.5.4) | Not Found   | [Problem](#schemaproblem)               |

<aside class="warning">
To perform this operation, you must be authenticated by means of one of the following methods:
OAuth2 ( Scopes: read:pickup-location )
</aside>

<h1 id="mp-partner-api-service-products">Products</h1>

The Products API allows you to retrieve Products and update their inventory stocks.

<aside class="warning">
To perform all <b>Products</b> operations, you must always send your oauth2 access-token (see the <a href="#request-a-token">Request a Token</a> section).
Look at the 'Authorization' header in the code samples.
</aside>

## Get all Products

<a id="opIdgetProducts-page-pageSize-pickupLocations"></a>

> Code samples

```shell

curl --include -X GET \
     "https://{HOSTNAME}/v1/products?page=1&pageSize=50&pickupLocationIds=23789,23790" \
     -H 'Accept: application/json' \
     -H 'Authorization: Bearer {access-token}'

```

```scala
// TODO

```

`GET /v1/products`

_Query and filter products_

<h3 id="getproducts-page-pagesize-pickuplocations-parameters">Query Parameters</h3>

| Parameter       | In    | Type          | Required | Description                                                                                 |
| --------------- | ----- | ------------- | -------- | ------------------------------------------------------------------------------------------- |
| page            | query | integer       | false    | Page to start returning results from, must be >= 1                                          |
| pageSize        | query | integer       | false    | Number of items on one page, must be >= 1 and <= 100                                        |
| pickupLocations | query | array[string] | false    | The unique ids of pickup locations you want to retrieve the products from (comma-separated) |

> 200 Response

```json
{
  "page": 1,
  "pageSize": 50,
  "items": [
    {
      "barcodes": ["111122223333"],
      "status": {
        "type": "Enabled"
      },
      "title": "Chocolate Cereals",
      "rpc": 100150,
      "pickupLocations": [
        {
          "id": "23789"
        }
      ],
      "productCode": "abc-merch-specific-choc-cereals-code"
    },
    {
      "barcodes": ["444455556666"],
      "status": {
        "type": "Disabled"
      },
      "title": "Apple Cereals",
      "rpc": 100151,
      "pickupLocations": [
        {
          "id": "23789"
        }
      ],
      "productCode": "abc-merch-specific-choc-cereals-code"
    },
    {
      "barcodes": ["444455556666"],
      "status": {
        "type": "Discontinued"
      },
      "title": "Fresh Orange Juice",
      "rpc": 120350,
      "pickupLocations": [
        {
          "id": "23790"
        }
      ],
      "productCode": "abc-merch-specific-orange-juice-code"
    }
  ],
  "total": 3
}
```

<h3 id="getproducts-page-pagesize-pickuplocations-responses">Responses</h3>

| Status | Meaning                                                 | Description | Schema                                |
| ------ | ------------------------------------------------------- | ----------- | ------------------------------------- |
| 200    | [OK](https://tools.ietf.org/html/rfc7231#section-6.3.1) | OK          | [Page*Product*](#schemapage_product_) |

<aside class="warning">
To perform this operation, you must be authenticated by means of one of the following methods:
OAuth2 ( Scopes: read:product )
</aside>

## Get one Product

Get one Product _by RPC_ (RPC stands for 'RedMart Product Code')

<a id="opIdgetProducts-productId"></a>

> Code samples

```shell

curl --include -X GET \
     "https://{HOSTNAME}/v1/products/{productId}" \
     -H 'Accept: application/json' \
     -H 'Authorization: Bearer {access-token}'

```

```scala
// TODO

```

`GET /v1/products/{productId}`

_Querying the details of a specific product by RPC_

<h3 id="getproducts-productid-parameters">Path Parameters</h3>

| Parameter | In   | Type   | Required | Description                                                                             |
| --------- | ---- | ------ | -------- | --------------------------------------------------------------------------------------- |
| productId | path | string | true     | the RPC of the Product (so the RedMart-specific code, _not_ the merchant-specific code) |

> 200 Response

```json
{
  "barcodes": ["111122223333"],
  "status": {
    "type": "Enabled"
  },
  "title": "Chocolate Cereals",
  "rpc": 100150,
  "pickupLocations": [
    {
      "id": "23789"
    }
  ],
  "productCode": "abc-merch-specific-choc-cereals-code"
}
```

> 404 Response

```json
{
  "title": "The requested resource could not be found"
}
```

<h3 id="getproducts-productid-responses">Responses</h3>

| Status | Meaning                                                          | Description | Schema                    |
| ------ | ---------------------------------------------------------------- | ----------- | ------------------------- |
| 200    | [OK](https://tools.ietf.org/html/rfc7231#section-6.3.1)          | OK          | [Product](#schemaproduct) |
| 400    | [Bad Request](https://tools.ietf.org/html/rfc7231#section-6.5.1) | Bad Request | [Problem](#schemaproblem) |
| 404    | [Not Found](https://tools.ietf.org/html/rfc7231#section-6.5.4)   | Not Found   | [Problem](#schemaproblem) |

<aside class="warning">
To perform this operation, you must be authenticated by means of one of the following methods:
OAuth2 ( Scopes: read:product )
</aside>

## Get all Stock Lots

<a id="opIdgetProductsPickup-locationsStocks-productId-pickupLocationId"></a>

With this endpoint, retrieve all Stock Lots of a given product. For now, RedMart supports only _one_ Stock Lot per product.

> Code samples

```shell

curl --include -X GET \
     "https://{HOSTNAME}/v1/products/{productId}/pickup-locations/{pickupLocationId}/stock-lots" \
     -H 'Accept: application/json' \
     -H 'Authorization: Bearer {access-token}'

```

```scala
// TODO

```

`GET /v1/products/{productId}/pickup-locations/{pickupLocationId}/stock-lots`

_Querying all in-store stock lots levels for a specific product in a specific pickup location_

<h3 id="getproductspickup-locationsstocks-productid-pickuplocationid-parameters">Path Parameters</h3>

| Parameter        | In   | Type   | Required | Description                                                                             |
| ---------------- | ---- | ------ | -------- | --------------------------------------------------------------------------------------- |
| productId        | path | string | true     | the RPC of the Product (so the RedMart-specific code, _not_ the merchant-specific code) |
| pickupLocationId | path | string | true     | The unique id of the pickup location where the product is stored                        |

> 200 Response

```json
[
  {
    "id": "0",
    "quantityAtPickupLocation": 10,
    "quantityScheduledForPickup": 2,
    "quantityAvailableForSale": 8
  }
]
```

<h3 id="getproductspickup-locationsstocks-productid-pickuplocationid-responses">Responses</h3>

| Status | Meaning                                                          | Description | Schema                    |
| ------ | ---------------------------------------------------------------- | ----------- | ------------------------- |
| 200    | [OK](https://tools.ietf.org/html/rfc7231#section-6.3.1)          | OK          | Inline                    |
| 400    | [Bad Request](https://tools.ietf.org/html/rfc7231#section-6.5.1) | Bad Request | [Problem](#schemaproblem) |
| 404    | [Not Found](https://tools.ietf.org/html/rfc7231#section-6.5.4)   | Not Found   | [Problem](#schemaproblem) |

<h3 id="getproductspickup-locationsstocks-productid-pickuplocationid-responseschema">200 Response Schema</h3>

| Name                       | Type                          | Required | Restrictions | Description                                                                                                                                          |
| -------------------------- | ----------------------------- | -------- | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
|                            | [[StockLot](#schemastocklot)] | false    | none         | none                                                                                                                                                 |
| id                         | string                        | true     | none         | Identifier of the requested Stock Lot. For now always hardcoded to "0" (please note the `String` type, do **not** always expect it to be a number !) |
| quantityAtPickupLocation   | integer(int32)                | true     | none         | Number of items available in the pickup location                                                                                                     |
| quantityScheduledForPickup | integer(int32)                | true     | none         | Number of items that are scheduled for pickup in the next few days                                                                                   |
| quantityAvailableForSale   | integer(int32)                | true     | none         | Number of items that can currently still be ordered by customers                                                                                     |

<aside class="warning">
To perform this operation, you must be authenticated by means of one of the following methods:
OAuth2 ( Scopes: read:stock-lot )
</aside>

## Get one Stock Lot

<a id="opIdgetProductsPickup-locationsStocks1970-01-01T00:00:00Z-productId-pickupLocationId"></a>

For now, RedMart supports only _one_ Stock Lot per Product.

> Code samples

```shell

curl --include -X GET \
     "https://{HOSTNAME}/v1/products/{productId}/pickup-locations/{pickupLocationId}/stock-lots/0" \
     -H 'Accept: application/json' \
     -H 'Authorization: Bearer {access-token}'

```

```scala
// TODO

```

`GET /v1/products/{productId}/pickup-locations/{pickupLocationId}/stock-lots/0`

_Querying in-store stock levels for a product in a specific pickup location_

<h3 id="getproductspickup-locationsstocks1970-01-01t00:00:00z-productid-pickuplocationid-parameters">Path Parameters</h3>

| Parameter        | In   | Type   | Required | Description                                                                                                                                          |
| ---------------- | ---- | ------ | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| productId        | path | string | true     | the RPC of the Product (so the RedMart-specific code, _not_ the merchant-specific code)                                                              |
| pickupLocationId | path | string | true     | The unique id of the pickup location where the product is stored                                                                                     |
| id               | path | string | true     | Identifier of the requested Stock Lot. For now always hardcoded to "0" (please note the `String` type, do **not** always expect it to be a number !) |

> 200 Response

```json
{
  "id": "0",
  "quantityAtPickupLocation": 10,
  "quantityScheduledForPickup": 2,
  "quantityAvailableForSale": 8
}
```

> 404 Response

```json
{
  "title": "The requested resource could not be found"
}
```

<h3 id="getproductspickup-locationsstocks1970-01-01t00:00:00z-productid-pickuplocationid-responses">Responses</h3>

| Status | Meaning                                                          | Description | Schema                      |
| ------ | ---------------------------------------------------------------- | ----------- | --------------------------- |
| 200    | [OK](https://tools.ietf.org/html/rfc7231#section-6.3.1)          | OK          | [StockLot](#schemastocklot) |
| 400    | [Bad Request](https://tools.ietf.org/html/rfc7231#section-6.5.1) | Bad Request | [Problem](#schemaproblem)   |
| 404    | [Not Found](https://tools.ietf.org/html/rfc7231#section-6.5.4)   | Not Found   | [Problem](#schemaproblem)   |

### Response Headers

<a id="get-one-stock-of-a-product-response-headers"></a>

| Status | Header | Type   | Format | Description                                                 |
| ------ | ------ | ------ | ------ | ----------------------------------------------------------- |
| 200    | ETag   | string |        | Identifier that describes the latest state of this resource |

<aside class="warning">
To perform this operation, you must be authenticated by means of one of the following methods:
OAuth2 ( Scopes: read:stock-lot )
</aside>

## Update one Stock Lot

<a id="opIdpatchProductsPickup-locationsStocks1970-01-01T00:00:00Z-productId-pickupLocationId-If-Match-body"></a>

> Code samples

```shell

curl --include -X PATCH \
     "https://{HOSTNAME}/v1/products/{productId}/pickup-locations/{pickupLocationId}/stock-lots/0" \
     -H 'Content-Type: application/merge-patch+json' \
     -H 'Accept: application/json' \
     -H 'If-Match: "10"' \
     -H 'Authorization: Bearer {access-token}' \
     --data '{"quantityAtPickupLocation": 20}'
```

```scala
// TODO

```

`PATCH /v1/products/{productId}/pickup-locations/{pickupLocationId}/stock-lots/0`

_Updating a specific in-store stock lot's level for an RPC in a specific pickup location_

> Body parameter

```json
{
  "quantityAtPickupLocation": 20
}
```

<h3 id="patchproductspickup-locationsstocks1970-01-01t00:00:00z-productid-pickuplocationid-if-match-body-parameters">Path and Header Parameters</h3>

| Parameter        | In     | Type                                    | Required | Description                                                                                                                                                                                                                                               |
| ---------------- | ------ | --------------------------------------- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| productId        | path   | string                                  | true     | the RPC of the Product (so the RedMart-specific code, _not_ the merchant-specific code)                                                                                                                                                                   |
| pickupLocationId | path   | string                                  | true     | The unique id of the pickup location where the product is stored                                                                                                                                                                                          |
| id               | path   | string                                  | true     | Identifier of the requested Stock Lot. For now always hardcoded to "0" (please note the `String` type, do **not** always expect it to be a number !)                                                                                                      |
| If-Match         | header | string                                  | true     | The update request will only be processed if this value matches the latest `ETag` value of the resource (see <a href="#get-one-stock-of-a-product-response-headers">Response Headers</a> section of the [Get one Stock Lot](#get-one-stock-lot) endpoint) |
| body             | body   | [StockLotUpdate](#schemastocklotupdate) | true     | StockLotUpdate                                                                                                                                                                                                                                            |

> 200 Response

```json
{
  "id": "0",
  "quantityAtPickupLocation": 20,
  "quantityScheduledForPickup": 2,
  "quantityAvailableForSale": 18
}
```

> 409 Response

```json
{
  "title": "the request conflicted with the current state of the resource"
}
```

> 412 Response

```json
{
  "title": "the If-Match condition evaluated to false"
}
```

<h3 id="patchproductspickup-locationsstocks1970-01-01t00:00:00z-productid-pickuplocationid-if-match-body-responses">Responses</h3>

| Status | Meaning                                                                | Description                                                                                                                                                                                                                  | Schema                      |
| ------ | ---------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------- |
| 200    | [OK](https://tools.ietf.org/html/rfc7231#section-6.3.1)                | OK                                                                                                                                                                                                                           | [StockLot](#schemastocklot) |
| 400    | [Bad Request](https://tools.ietf.org/html/rfc7231#section-6.5.1)       | Bad Request                                                                                                                                                                                                                  | [Problem](#schemaproblem)   |
| 404    | [Not Found](https://tools.ietf.org/html/rfc7231#section-6.5.4)         | Not Found                                                                                                                                                                                                                    | [Problem](#schemaproblem)   |
| 409    | [Conflict](https://tools.ietf.org/html/rfc7231#section-6.5.8)          | The `quantityAtPickupLocation` value in your request is strictly lower than the `quantityScheduledForPickup` value. Set `quantityAtPickupLocation` to be at least equal to `quantityScheduledForPickup` before resubmitting. | [Problem](#schemaproblem)   |
| 412    | [Precondition Failed](https://tools.ietf.org/html/rfc7232#section-4.2) | The Stock Lot level has changed since the last time you read it, which may result in an inconsistent state. Re-read the Stock Lot, fetch its `Etag` and update your `If-Match` header before resubmitting.                   | [Problem](#schemaproblem)   |

### Response Headers

| Status | Header | Type   | Format | Description                                                 |
| ------ | ------ | ------ | ------ | ----------------------------------------------------------- |
| 200    | ETag   | string |        | Identifier that describes the latest state of this resource |

<aside class="warning">
To perform this operation, you must be authenticated by means of one of the following methods:
OAuth2 ( Scopes: write:stock-lot )
</aside>

# Pickup Jobs

The Pickup Jobs API allows you to retrieve scheduled or completed pickup jobs.

<aside class="success">
This API is <b>invite only</b>. If you want to use this api then please reach out to our support at <a href="mailto:rm_partnersupport@care.lazada.com">rm_partnersupport@care.lazada.com</a> and provide your store name and id.  
</aside>

<aside class="warning">
To perform all <b>Pickup Jobs</b> operations, you must always send your oauth2 access-token (see the <a href="#request-a-token">Request a Token</a> section).
Look at the 'Authorization' header in the code samples.
</aside>

## Get Pickup Jobs

<a id="get-all-pickup-jobs"></a>

> Code samples

```shell

curl --include -X GET \
     "https://{HOSTNAME}/v1/pickup-jobs?from=1682870400000&till=1685548800000&statuses=pending" \
     -H 'Accept: application/json' \
     -H 'Authorization: Bearer {access-token}'

```

```scala
// TODO
```

`GET /v1/pickup-jobs`

_Query and filter pickup jobs. Maximum allowable date range query is 31 days. Unable to retrieve data older than 180 days._

<h3 id="getpickup-jobs-from-till-statuses-parameters">Query Parameters</h3>

| Parameter | In    | Type          | Required | Description                                                                                                      |
| --------- | ----- | ------------- | -------- | ---------------------------------------------------------------------------------------------------------------- |
| from      | query | integer       | true     | Epoch milliseconds of start of date range query                                                                  |
| till      | query | integer       | false    | Epoch milliseconds of end of date range query                                                                    |
| statuses  | query | array[string] | false    | Comma-separated job statuses that you want to include. Refer to [JobStatus](#schemajobstatus) for valid statuses |

> 200 Response

```json
[
  {
    "scheduledAt": 0,
    "qtyFulfilledCount": 0,
    "amendabilityCutOffDate": 0,
    "preferredPickupTime": "string",
    "items": [
      {
        "name": "string",
        "qtyFulfilled": 0,
        "sku": "string",
        "size": "string",
        "shipmentsInfo": [
          {
            "qty": 0,
            "orderId": "string"
          }
        ],
        "minimumExpiryDate": "string",
        "qty": 0,
        "vpc": "string",
        "imageUrl": "string",
        "rpc": 0
      }
    ],
    "pickedAt": 0,
    "id": 0,
    "status": "string",
    "category": "string",
    "qtyCount": 0
  }
]
```

<h3 id="getpickup-jobs-from-till-statuses-responses">Responses</h3>

| Status | Meaning                                                          | Description | Schema                                |
| ------ | ---------------------------------------------------------------- | ----------- | ------------------------------------- |
| 200    | [OK](https://tools.ietf.org/html/rfc7231#section-6.3.1)          | OK          | [List of PickupJob](#schemapickupjob) |
| 400    | [Bad Request](https://tools.ietf.org/html/rfc7231#section-6.5.1) | Bad Request | [Problem](#schemaproblem)             |

<aside class="success">
This API is <b>invite only</b>. If you want to use this api then please reach out to our support at <a href="mailto:rm_partnersupport@care.lazada.com">rm_partnersupport@care.lazada.com</a> and provide your store name and id.  
</aside>

<aside class="warning">
To perform this operation, you must be authenticated by means of one of the following methods:
OAuth2 ( Scopes: read:pickup-job )
</aside>

## Get Pickup Job

<a id="get-one-pickup-job"></a>

> Code samples

```shell

curl --include -X GET \
     "https://{HOSTNAME}/v1/pickup-jobs/{pickupJobId}" \
     -H 'Accept: application/json' \
     -H 'Authorization: Bearer {access-token}'

```

```scala
// TODO
```

`GET /v1/pickup-jobs/{pickupJobId}`

_Querying one pickup job by pickupJobId_

<h3 id="getpickup-jobs-pickupjobid-parameters">Path Parameters</h3>

| Parameter   | In   | Type   | Required | Description                     |
| ----------- | ---- | ------ | -------- | ------------------------------- |
| pickupJobId | path | string | true     | Unique identifier of pickup job |

> 200 Response

```json
{
  "scheduledAt": 0,
  "qtyFulfilledCount": 0,
  "amendabilityCutOffDate": 0,
  "preferredPickupTime": "string",
  "items": [
    {
      "name": "string",
      "qtyFulfilled": 0,
      "sku": "string",
      "size": "string",
      "shipmentsInfo": [
        {
          "qty": 0,
          "orderId": "string"
        }
      ],
      "minimumExpiryDate": "string",
      "qty": 0,
      "vpc": "string",
      "imageUrl": "string",
      "rpc": 0
    }
  ],
  "pickedAt": 0,
  "id": 0,
  "status": "string",
  "category": "string",
  "qtyCount": 0
}
```

> 404 Response

```json
{
  "title": "Could not find store or pickup job"
}
```

<h3 id="getpickup-jobs-pickupjobid-responses">Responses</h3>

| Status | Meaning                                                        | Description                                | Schema                        |
| ------ | -------------------------------------------------------------- | ------------------------------------------ | ----------------------------- |
| 200    | [OK](https://tools.ietf.org/html/rfc7231#section-6.3.1)        | OK                                         | [PickupJob](#schemapickupjob) |
| 404    | [Not Found](https://tools.ietf.org/html/rfc7231#section-6.5.4) | Pickup job associated with store not Found | [Problem](#schemaproblem)     |

<aside class="success">
This API is <b>invite only</b>. If you want to use this api then please reach out to our support at <a href="mailto:rm_partnersupport@care.lazada.com">rm_partnersupport@care.lazada.com</a> and provide your store name and id.  
</aside>

<aside class="warning">
To perform this operation, you must be authenticated by means of one of the following methods:
OAuth2 ( Scopes: read:pickup-job )
</aside>

# _Schemas_

All schemas referenced in the APIs documented above

<h2 id="tocSpickuplocation">PickupLocation</h2>

<a id="schemapickuplocation"></a>

```json
{
  "addressLine1": "string",
  "city": "string",
  "name": "string",
  "country": "string",
  "postalCode": "string",
  "id": "string",
  "addressLine2": "string"
}
```

_PickupLocation_

### Properties

| Name         | Type   | Required | Restrictions | Description |
| ------------ | ------ | -------- | ------------ | ----------- |
| addressLine1 | string | true     | none         | none        |
| city         | string | true     | none         | none        |
| name         | string | true     | none         | none        |
| country      | string | true     | none         | none        |
| postalCode   | string | true     | none         | none        |
| id           | string | true     | none         | none        |
| addressLine2 | string | true     | none         | none        |

<h2 id="tocSstocklotupdate">StockLotUpdate</h2>

<a id="schemastocklotupdate"></a>

```json
{
  "quantityAtPickupLocation": 0
}
```

_StockLotUpdate_

### Properties

| Name                     | Type           | Required | Restrictions | Description                                      |
| ------------------------ | -------------- | -------- | ------------ | ------------------------------------------------ |
| quantityAtPickupLocation | integer(int32) | true     | none         | Number of items available in the pickup location |

<h2 id="tocSproblem">Problem</h2>

<a id="schemaproblem"></a>

```json
{
  "title": "string"
}
```

_Problem_

### Properties

| Name  | Type   | Required | Restrictions | Description |
| ----- | ------ | -------- | ------------ | ----------- |
| title | string | true     | none         | none        |

<h2 id="tocSpickuplocationid">PickupLocationId</h2>

<a id="schemapickuplocationid"></a>

```json
{
  "id": "string"
}
```

_PickupLocationId_

### Properties

| Name | Type   | Required | Restrictions | Description |
| ---- | ------ | -------- | ------------ | ----------- |
| id   | string | true     | none         | none        |

<h2 id="tocSproduct">Product</h2>

<a id="schemaproduct"></a>

```json
{
  "barcodes": ["string"],
  "status": {
    "type": "Enabled"
  },
  "title": "string",
  "rpc": 0,
  "pickupLocations": [
    {
      "id": "string"
    }
  ],
  "productCode": "string"
}
```

_Product_

### Properties

| Name            | Type                                          | Required | Restrictions | Description                                  |
| --------------- | --------------------------------------------- | -------- | ------------ | -------------------------------------------- |
| barcodes        | [string]                                      | true     | none         | none                                         |
| status          | [Status](#schemastatus)                       | true     | none         | none                                         |
| title           | string                                        | true     | none         | none                                         |
| rpc             | integer(int64)                                | true     | none         | The RPC (RedMart Product Code) of an item    |
| pickupLocations | [[PickupLocationId](#schemapickuplocationid)] | true     | none         | none                                         |
| productCode     | string                                        | true     | none         | Custom product code as defined by the seller |

<h2 id="tocSstatus">Status</h2>

<a id="schemastatus"></a>

```json
{
  "type": "Disabled"
}
```

_Status_

### Properties

| Name | Type   | Required | Restrictions | Description |
| ---- | ------ | -------- | ------------ | ----------- |
| type | string | true     | none         | none        |

#### Enumerated Values

| Property | Value        |
| -------- | ------------ |
| type     | Disabled     |
| type     | Discontinued |
| type     | Enabled      |

<h2 id="tocSstocklot">Stock Lot</h2>

<a id="schemastocklot"></a>

```json
{
  "id": "0",
  "quantityAtPickupLocation": 0,
  "quantityScheduledForPickup": 0,
  "quantityAvailableForSale": 0
}
```

_StockLot_

### Properties

| Name                       | Type           | Required | Restrictions | Description                                                        |
| -------------------------- | -------------- | -------- | ------------ | ------------------------------------------------------------------ |
| id                         | string         | true     | none         | Identifier of the Stock Lot. For now always hardcoded to "0"       |
| quantityAtPickupLocation   | integer(int32) | true     | none         | Number of items available in the pickup location                   |
| quantityScheduledForPickup | integer(int32) | true     | none         | Number of items that are scheduled for pickup in the next few days |
| quantityAvailableForSale   | integer(int32) | true     | none         | Number of items that can currently still be ordered by customers   |

<h2 id="tocSpage_pickuplocation_">Page_PickupLocation_</h2>

<a id="schemapage_pickuplocation_"></a>

```json
{
  "page": 0,
  "pageSize": 0,
  "items": [
    {
      "addressLine1": "string",
      "city": "string",
      "name": "string",
      "country": "string",
      "postalCode": "string",
      "id": "string",
      "addressLine2": "string"
    }
  ],
  "total": 0
}
```

_Page«PickupLocation»_

### Properties

| Name     | Type                                      | Required | Restrictions | Description |
| -------- | ----------------------------------------- | -------- | ------------ | ----------- |
| page     | integer(int32)                            | true     | none         | none        |
| pageSize | integer(int32)                            | true     | none         | none        |
| items    | [[PickupLocation](#schemapickuplocation)] | true     | none         | none        |
| total    | integer(int32)                            | false    | none         | none        |

<h2 id="tocSpage_product_">Page_Product_</h2>

<a id="schemapage_product_"></a>

```json
{
  "page": 0,
  "pageSize": 0,
  "items": [
    {
      "barcodes": ["string"],
      "status": {
        "type": "Enabled"
      },
      "title": "string",
      "rpc": 0,
      "pickupLocations": [
        {
          "addressLine1": "string",
          "city": "string",
          "name": "string",
          "country": "string",
          "postalCode": "string",
          "id": "string",
          "addressLine2": "string"
        }
      ],
      "productCode": "string"
    }
  ],
  "total": 0
}
```

_Page«Product»_

### Properties

| Name     | Type                        | Required | Restrictions | Description |
| -------- | --------------------------- | -------- | ------------ | ----------- |
| page     | integer(int32)              | true     | none         | none        |
| pageSize | integer(int32)              | true     | none         | none        |
| items    | [[Product](#schemaproduct)] | true     | none         | none        |
| total    | integer(int32)              | false    | none         | none        |

<h2 id="tocS_PickupJob">PickupJob</h2>
<!-- backwards compatibility -->
<a id="schemapickupjob"></a>
<a id="schema_PickupJob"></a>
<a id="tocSpickupjob"></a>
<a id="tocspickupjob"></a>

```json
{
  "scheduledAt": 0,
  "qtyFulfilledCount": 0,
  "amendabilityCutOffDate": 0,
  "preferredPickupTime": "string",
  "items": [
    {
      "name": "string",
      "qtyFulfilled": 0,
      "sku": "string",
      "size": "string",
      "shipmentsInfo": [
        {
          "qty": 0,
          "orderId": "string"
        }
      ],
      "minimumExpiryDate": "string",
      "qty": 0,
      "vpc": "string",
      "imageUrl": "string",
      "rpc": 0
    }
  ],
  "pickedAt": 0,
  "id": 0,
  "status": "string",
  "category": "string",
  "qtyCount": 0
}
```

_PickupJob_

### Properties

| Name                   | Type                              | Required | Restrictions | Description                                                                |
| ---------------------- | --------------------------------- | -------- | ------------ | -------------------------------------------------------------------------- |
| scheduledAt            | integer(int64)                    | true     | none         | Epoch milliseconds of scheduled pick up time                               |
| qtyFulfilledCount      | integer(int32)                    | true     | none         | Actual number of items picked up in the scheduled job                      |
| amendabilityCutOffDate | integer(int64)                    | false    | none         | Epoch milliseconds after which pickup job cannot be amended (if available) |
| preferredPickupTime    | string                            | false    | none         | Preferred pickup time/range specified by seller (if applicable)            |
| items                  | [[PickupItem](#schemapickupitem)] | true     | none         | List of items being picked up                                              |
| pickedAt               | integer(int64)                    | false    | none         | Epoch milliseconds of actual pickup time                                   |
| id                     | integer(int64)                    | true     | none         | Pickup job id                                                              |
| status                 | string                            | true     | none         | Pickup job status. Refer to [JobStatus](#schemajobstatus) for more details |
| category               | string                            | true     | none         | Category of products being picked up, e.g. dry, fresh, frozen, etc         |
| qtyCount               | integer(int32)                    | true     | none         | Number of items to be picked up in the scheduled job                       |

<h2 id="tocS_PickupItem">PickupItem</h2>
<!-- backwards compatibility -->
<a id="schemapickupitem"></a>
<a id="schema_PickupItem"></a>
<a id="tocSpickupitem"></a>
<a id="tocspickupitem"></a>

```json
{
  "name": "string",
  "qtyFulfilled": 0,
  "sku": "string",
  "size": "string",
  "shipmentsInfo": [
    {
      "qty": 0,
      "orderId": "string"
    }
  ],
  "minimumExpiryDate": "string",
  "qty": 0,
  "vpc": "string",
  "imageUrl": "string",
  "rpc": 0
}
```

_PickupItem_

### Properties

| Name              | Type                                  | Required | Restrictions | Description                                                  |
| ----------------- | ------------------------------------- | -------- | ------------ | ------------------------------------------------------------ |
| name              | string                                | true     | none         | Name of item                                                 |
| qtyFulfilled      | integer(int32)                        | false    | none         | Quantity of the item being picked up in actual job           |
| sku               | string                                | true     | none         | Stock Keeping Unit (SKU) code for the item                   |
| size              | string                                | false    | none         | The size or dimensions of the item (if available)            |
| shipmentsInfo     | [[ShipmentInfo](#schemashipmentinfo)] | false    | none         | List of orders for the pickup of this item                   |
| minimumExpiryDate | string                                | false    | none         | Minimum expiry date of item                                  |
| qty               | integer(int32)                        | true     | none         | Quantity of the item to be picked up in scheduled job        |
| vpc               | string                                | false    | none         | none                                                         |
| imageUrl          | string                                | false    | none         | The URL of the image associated with the item (if available) |
| rpc               | integer(int64)                        | true     | none         | The RPC (RedMart Product Code) of an item                    |

<h2 id="tocS_ShipmentInfo">ShipmentInfo</h2>
<!-- backwards compatibility -->
<a id="schemashipmentinfo"></a>
<a id="schema_ShipmentInfo"></a>
<a id="tocSshipmentinfo"></a>
<a id="tocsshipmentinfo"></a>

```json
{
  "qty": 0,
  "orderId": "string"
}
```

_ShipmentInfo_

### Properties

| Name    | Type           | Required | Restrictions | Description                    |
| ------- | -------------- | -------- | ------------ | ------------------------------ |
| qty     | integer(int32) | true     | none         | Quantity of item               |
| orderId | string         | true     | none         | Unique identifier of the order |

<h2 id="tocSjobstatus">JobStatus</h2>

<a id="schemajobstatus"></a>

```json
"pickedup"
```

_JobStatus_

### Possible Job Status

Query using values in string column.

| Status        | String      | Description     |
| ------------- | ----------- | --------------- |
| Pending       | "pending"   | Job pending     |
| PickupArrived | "arrived"   | Driver arrived  |
| PickedUp      | "pickedup"  | Items picked up |
| Cancelled     | "cancelled" | Job cancelled   |
| PickupFailed  | "failed"    | Pickup failed   |
