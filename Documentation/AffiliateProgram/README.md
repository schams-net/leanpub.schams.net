# Affiliate Program

## Description

This API lets partners trigger a discount for a specific product at Leanpub.
The contact details of the recipient are passed in the payload to the API.
If successful, the function generates a coupon and sends an email to the recipient.
The email contains the link to Leanpub that lets the recipient redeem the discount when they purchase the product.

## API Endpoint

URI: `https://leanpub.schams.net/api/v2/PartnerDiscountRequest`

Only `POST` requests are supported. Any other method results in a "Missing Authentication Token" error (HTTP status code: 403).

## Authentication

All requests to this API endpoint require a valid **API key** and **partner identifier**.
The client must send the API key in the HTTP header. The partner identifier is part of the payload as shown below.

## Sandbox Mode

All requests are initially made in a sandbox mode by default.
In this mode, the API accepts and processes the requests the same way as in production.
This also applies to the responses from the API.

However, while in sandbox mode, the function does not create discount coupons at Leanpub.
It does not send the confirmation email with a dummy coupon code to the recipient's email address.
Instead, the function generates an email to a pre-defined address.

Discount coupons are only created at Leanpub if the sandbox mode is disabled.
This requires that the partner account has been set to *production-ready* and that the requests to the API have the property `sandboxMode` set to `false`.
If the property is not set, or set to `true`, or the partner account is restricted (not production-ready), the system automatically falls back to the sandbox mode.

## Request

### HTTP Header

The following HTTP header is required:

```text
x-api-key: <string>
```

### HTTP Body

The API expects the following JSON data as the payload of the HTTP request:

```text
{
  "productId": "<string>",
  "partnerId": "<string>",
  "origin": "<string>",
  "transactionId": "<string>",
  "sandboxMode": <boolean>,
  "recipient": {
    "firstName": "<string>",
    "lastName": "<string>",
    "email": "<string>"
  }
}
```

### Payload

| Property        | Data type | Required | Description                         | Character set                |
| :-------------- | :-------- | :------: | :---------------------------------- | :--------------------------- |
| `productId`     | string    | yes      | Product identifier                  | ASCII                        |
| `partnerId`     | string    | yes      | Partner identifier                  | Extended ASCII               |
| `origin`        | string    | yes      | One of the valid origin systems     | ASCII                        |
| `transactionId` | string    | no       | Arbitrary identifier                | ASCII                        |
| `sandboxMode`   | boolean   | no       | *see section "Sandbox Mode" above*  | `true` or `false`            |
| `recipient`     | array     | yes      | *see table below*                   |                              |

### Sub-property of `recipient`

| Property        | Data type | Required | Description                         | Character set                |
| :-------------- | :-------- | :------: | :---------------------------------- | :--------------------------- |
| `firstName`     | string    | yes      | Recipient's first name              | Extended ASCII               |
| `lastName`      | string    | yes      | Recipient's last name               | Extended ASCII               |
| `email`         | string    | yes      | Recipient's email address           | Extended ASCII               |

### Example Request

```bash
curl -s -X POST \
  --header 'x-api-key: aaaabbbbccccddddeeeeffff12345678' \
  --data '{"partnerId": "acme", "origin": "shop.example.com", "productId": "foobar", "recipient": {"firstName": "John", "lastName": "Smith", "email": "john@example.com"}}' \
  https://leanpub.schams.net/api/v2/PartnerDiscountRequest
```

## Response

The API always responses with JSON data.

```text
{
  "code": <integer>,
  "message": "<string>",
  "requestId": "<string>"
}
```

| Code | Message                | Notes                                        |
| ---: | :--------------------- | :------------------------------------------- |
|    0 | OK                     | Request acknowledged, *see notes below*      |
|  401 | Invalid partner ID     |                                              |
|  402 | Missing partner ID     |                                              |
|  403 | Invalid API key        |                                              |
|  404 | Invalid origin         |                                              |
|  405 | Invalid product ID     |                                              |
|  406 | Invalid recipient(s)   |                                              |
|  407 | Permission denied      | Attempt to leave sandbox mode refused        |
|  501 | Failed to schedule job | API error                                    |

A malformed payload or invalid HTTP request can result in a different error, e.g. "Missing authentication token" or "internal server error".
The `requestId` represents a unique identifier that can be used for debugging purposes.

## Additional Notes

When the API receives a discount request and the request passes the validation checks, the function schedules a job and sends the response to the client straight away.
Different functions handle the pending jobs, create the discount coupons at Leanpub, and send the emails to the recipient.
As this is an asynchronous process, the API function that receives the request is not aware of the results of other functions.
Therefore, a message "OK" (code: 0) in the response does not mean that a coupon was already created and an email sent to the recipient.
An "OK" means that the API received and successfully validated the request (acknowledgement).
