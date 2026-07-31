# Protect an MCP Proxy with rate limiting and PII filtering

## Overview

This guide adds two protections to an MCP Proxy API, on top of authentication and access control: a **rate limit** that caps how many tool calls each caller can make, and a **PII filter** that redacts sensitive data from tool responses before they reach the calling agent.

### Why add these protections

Access control decides *which* tools a caller can invoke. It doesn't cap *how often* a caller invokes them, and it doesn't inspect *what* a tool returns. Two gaps remain:

* **Resource abuse.** A compromised or looping agent can hammer a permitted tool, driving cost and load. A per-caller rate limit contains it.
* **Data exposure.** A permitted read can still return names, emails, or account numbers to an agent and the model behind it. A response-side PII filter redacts that data at the gateway.

Both the `rate-limit` and `pii-filtering` policies declare support for the MCP proxy entrypoint, so you apply them to an MCP Proxy API the same way you apply any policy.

## Prerequisites

* An MCP Proxy API. To create one, see [Convert REST APIs to an MCP Server](convert-your-apis-to-mcp-servers.md).
* For PII filtering: an **AI Model Token Classification** resource on the API. See [AI resources](AI-resources/README.md).

## Rate-limit tool calls per caller

The `rate-limit` policy is available in the Community and Enterprise editions. Add it to the MCP Proxy API's flow, on the request phase.

<!-- TODO: verify the Policy Studio steps and field labels against the console source. -->

To cap each caller independently rather than the API as a whole, set the policy's rate key to a caller attribute, for example:

```json
{
  "addHeaders": true,
  "rate": {
    "limit": 10,
    "periodTime": 60,
    "periodTimeUnit": "SECONDS",
    "key": "{#context.attributes['user']}"
  }
}
```

Each caller gets its own counter. When a caller exceeds the limit within the window, the gateway returns `HTTP 429` until the window resets.

<!-- TODO: confirm the caller attribute exposed on an MCP Proxy flow (author verified {#context.attributes['user']} on a preview build, not on this GA version). -->

## Redact PII from tool responses

`pii-filtering` is an Enterprise policy (`feature=apim-policy-pii-filtering`). It uses an AI Model Token Classification resource to detect and redact PII in the payload. The model runs inside the gateway, so no payload leaves the boundary to be classified.

1. Add an **AI Model Token Classification** resource to the API and select a PII detection model.
2. Add the `pii-filtering` policy to the API's flow and reference that resource:

    ```json
    {
      "resourceName": "<your-token-classification-resource>",
      "categories": ["PERSON", "EMAIL", "PHONE", "FINANCIAL_ACCOUNT", "GOVERNMENT_ID", "LOCATION"],
      "threshold": 0.5,
      "skipResponsePayloadFiltering": false
    }
    ```

With `skipResponsePayloadFiltering` set to `false`, the policy scans tool responses and replaces detected PII in the selected categories with `[REDACTED]` before the response leaves the gateway.

<!-- TODO: verify the Policy Studio steps, resource creation UI, and the model options against the console source. -->

## Verify the protections

* **Rate limit.** Call a permitted tool repeatedly as one caller. After the configured limit within the window, the next call returns `HTTP 429`.
* **PII filtering.** Invoke a tool whose response contains a value in one of the selected categories, such as an email address. The response returns that value as `[REDACTED]`.

## Considerations

* PII detection runs a classification model per response. It's CPU-bound, so size the gateway for your expected request rate.
