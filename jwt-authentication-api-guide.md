# JWT Authentication and Authorization: API Integration Guide

## 1. Overview

This guide explains how to authenticate with a REST API using JSON Web Tokens (JWT), obtain an access token, authorize API requests, and retrieve an invoice.

It is intended for developers integrating an API into a backend application or an automated service.

The examples use a fictional billing API. Endpoint URLs, request fields, and response payloads are illustrative and must be adapted to the authentication contract of the actual API.

By the end of this guide, you will know how to:

- Understand authentication, authorization, and JWTs.
- Obtain an access token.
- Include a Bearer token in an API request.
- Understand access-token expiration and refresh-token handling.
- Troubleshoot common authentication and authorization errors.

## 2. Authentication and authorization

Authentication and authorization are related but serve different purposes.

| Concept        | Description                                                                                          |
| -------------- | ---------------------------------------------------------------------------------------------------- |
| Authentication | Verifies the identity of a user or application.                                                      |
| Authorization  | Determines which resources and operations that identity can access.                                  |
| Access token   | A credential presented to an API to authorize a request.                                             |
| Refresh token  | A credential used to obtain a new access token when the authorization server supports token renewal. |
| JWT            | A token format that contains claims, such as an issuer, subject, audience, or expiration time.       |

A JWT is commonly used as an access token, but not all access tokens are JWTs. A refresh token may also use a different format.

**Important:** A valid access token does not automatically grant access to every endpoint. The API must validate the token and enforce the required permissions.

## 3. How the authentication flow works

For a server-to-server integration using the OAuth 2.0 client credentials grant, the typical flow is:

1. Register the client application with the authorization server.
2. Obtain the client credentials.
3. Exchange the credentials for an access token.
4. Include the access token in requests to protected API endpoints.
5. Obtain another access token when the current one expires.

The client credentials grant is intended for confidential clients, such as backend services that can securely protect their credentials.

A refresh token is generally not issued in this flow. The client normally requests another access token using its client credentials.

For applications acting on behalf of a user, an authorization-code flow with PKCE may be more appropriate. The token lifecycle depends on the chosen OAuth flow and the API's authentication implementation.

## 4. Obtain an access token

Before calling a protected endpoint, obtain an access token from the authorization server.

### Prerequisites

Ensure that you have:

- A registered client application.
- A client ID and client secret.
- The authorization server's token endpoint.
- Any required audience, scope, or other parameters specified by the API provider.

### Request an access token

The following example illustrates an OAuth 2.0 client credentials request.

```bash
curl --request POST \
  --url 'https://auth.example.com/oauth/token' \
  --header 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'grant_type=client_credentials' \
  --data-urlencode 'client_id=<client_id>' \
  --data-urlencode 'client_secret=<client_secret>'
```

The token endpoint URL and required parameters depend on the authorization server. Some providers require client authentication through an HTTP Authorization header rather than including credentials in the request body.

### Example response

A successful request typically returns HTTP `200 OK` with a JSON response similar to the following:

```json
{
  "access_token": "<access_token>",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

The response fields mean:

- `access_token`: the credential used to access protected resources.
- `token_type`: the token type expected by the API, commonly `Bearer`.
- `expires_in`: the token's validity period in seconds, if provided.

In this example, the token is valid for 3,600 seconds (one hour).

Store the token securely and use it only for its intended audience and permissions.

## 5. Understand JWT claims

A JSON Web Token commonly consists of three Base64URL-encoded parts separated by periods:

```text
header.payload.signature
```

The parts contain:

- **Header:** metadata such as the signing algorithm and token type.
- **Payload:** claims describing the token, such as its subject, issuer, audience, and expiration.
- **Signature:** allows the recipient to verify the token's integrity and authenticity.

For example, a JWT payload might contain the following claims:

```json
{
  "iss": "https://auth.example.com",
  "sub": "billing-service",
  "aud": "billing-api",
  "exp": 1790000000,
  "scope": "invoice:read"
}
```

These values are illustrative. Actual claims and their meanings depend on the token issuer and API implementation.

**Security note:** Base64URL encoding is not encryption. Do not place passwords, client secrets, or other confidential information in a JWT payload. A decoded payload must not be trusted until the token has been properly validated.

The API should validate the signature, allowed signing algorithm, issuer, audience, expiration, and any other required claims before accepting a JWT.

## 6. Call a protected endpoint

Once you have an access token, include it in the `Authorization` header of each request to a protected endpoint.

### Retrieve an invoice (example from Billown project to get an invoice)

Send a `GET` request to the invoice endpoint, replacing `<invoice_id>` with the identifier of the invoice.

```bash
curl --request GET \
  --url 'https://api.example.com/v1/invoices/<invoice_id>' \
  --header 'Authorization: Bearer <access_token>' \
  --header 'Accept: application/json'
```

The API validates the access token and checks whether the authenticated client has permission to read the requested invoice.

### Example response

A successful request returns HTTP `200 OK`.

```json
{
  "id": "inv_12345",
  "status": "paid",
  "currency": "EUR",
  "total": 150.0
}
```

The `total` field represents the invoice amount in the currency specified by `currency`.

### Authentication versus authorization failures

A request can fail even if an access token is present.

- **Authentication failure:** the token is missing, invalid, expired, or otherwise rejected.
- **Authorization failure:** the token is valid, but the client does not have permission to perform the requested operation.

For example, a client with the `invoice:read` permission may be allowed to retrieve an invoice but not delete it.

## 7. Handle access-token expiration and refresh tokens

Access tokens are usually short-lived. When a token expires, the API may reject requests made with it.

### Client credentials flow

For the client credentials grant shown in Section 4, request a new access token using the registered client credentials.

Do not assume that a refresh token will be returned.

### Flows that support refresh tokens

In flows where the authorization server issues a refresh token, the client can use that token to request a new access token.

An illustrative refresh request might look like this:

```bash
curl --request POST \
  --url 'https://auth.example.com/oauth/token' \
  --header 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'grant_type=refresh_token' \
  --data-urlencode 'refresh_token=<refresh_token>'
```

The exact parameters and client-authentication requirements depend on the authorization server.

A successful response may include a new access token and, when refresh-token rotation is supported, a replacement refresh token.

When using refresh tokens:

- Store them securely and restrict access to them.
- Follow the provider's refresh-token rotation and expiration rules.
- Replace the previous refresh token when the server issues a new one.
- If renewal fails because the refresh token has expired or been revoked, follow the application's reauthentication process.

Never send a refresh token to a protected business API endpoint unless that endpoint explicitly implements such a function. Refresh tokens are intended for the authorization server.

## 8. Handle common errors

| HTTP status                 | Meaning                                                      | Recommended action                                                                              |
| --------------------------- | ------------------------------------------------------------ | ----------------------------------------------------------------------------------------------- |
| `400 Bad Request`           | The request is malformed or contains invalid parameters.     | Verify the request body, grant type, and required fields.                                       |
| `401 Unauthorized`          | Authentication failed, or the access token was rejected.     | Check the token, its expiration, and the Authorization header. Obtain a new token if necessary. |
| `403 Forbidden`             | The request was understood, but access is not permitted.     | Check the required scopes, roles, and resource-level permissions.                               |
| `404 Not Found`             | The resource does not exist or is not exposed to the caller. | Verify the resource ID and consult the API's error-handling contract.                           |
| `429 Too Many Requests`     | The request rate exceeds the permitted limit.                | Respect rate-limit information and retry according to the API's guidance.                       |
| `500 Internal Server Error` | The server encountered an unexpected error.                  | Check the service status and retry safely according to the API's retry policy.                  |

The actual status code and response body depend on the API implementation.

## 9. Troubleshooting

### The token endpoint returns `400 Bad Request`

1. Verify that the grant type is supported.
2. Confirm that all required parameters are present.
3. Check the request's content type.
4. Consult the authorization server's error response for details.

### The API returns `401 Unauthorized`

1. Confirm that the `Authorization` header is present.
2. Check that the header uses the `Bearer` scheme.
3. Verify that the token has not expired.
4. Confirm that the token was issued by a trusted issuer and is intended for the target API.
5. Obtain a new access token and retry the request if appropriate.

### The API returns `403 Forbidden`

1. Verify that the client has the required scope or permission.
2. Check whether access is restricted by the requested resource, tenant, or account.
3. Ask the API administrator to review the client's authorization configuration if needed.

Obtaining a new token will not resolve a missing-permission error unless the token is reissued with the required permissions.

## 10. Security best practices

- Always use HTTPS for token and API requests.
- Store client secrets and tokens in a secure secrets manager or an equivalent protected environment.
- Never commit credentials to Git repositories.
- Avoid logging access tokens, refresh tokens, or client secrets.
- Request only the scopes needed by the application.
- Validate JWT signatures and claims using a trusted JWT library.
- Follow the authorization server's token expiration, revocation, and refresh policies.
- Avoid automatically retrying non-idempotent operations unless the API supports safe retries.

## 11. Next steps

After successfully authenticating and retrieving an invoice, consult the API reference for other available operations.

Before integrating with a production service, verify the actual token endpoint, supported OAuth grant types, required scopes, token lifetime, refresh behavior, and error-response format with the API provider's documentation.
