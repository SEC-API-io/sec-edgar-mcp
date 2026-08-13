# Connect AWS Bedrock AgentCore Gateway to SEC EDGAR Data with MCP

Amazon Bedrock AgentCore Gateway turns Lambda functions, REST APIs and MCP
servers into one MCP endpoint for your agents. This server becomes an **MCP
server target**.

## Prerequisites

- An AgentCore Gateway with inbound authorization configured.
- The AWS CLI, or boto3, with the `bedrock-agentcore-control` client.
- An IAM role that carries the gateway permissions AWS lists on the MCP server
  targets page.
- A gateway that negotiates one of the supported MCP protocol versions:
  2026-07-28, 2025-11-25, 2025-06-18 or 2025-03-26.
- A sec-api key. Get one at [sec-api.io](https://sec-api.io/profile).

## Config location

There is no file. You create a credential provider and a gateway target through
the control-plane API. Put both in the same account and Region as the gateway.

## Steps

1. Store the key as an API key credential provider. Keep the
   `credentialProviderArn` from the response.

```bash
aws bedrock-agentcore-control create-api-key-credential-provider \
  --name sec-api-key \
  --api-key YOUR_API_KEY
```

2. Create the target.

```bash
aws bedrock-agentcore-control create-gateway-target \
  --gateway-identifier "your-gateway-id" \
  --name "secapi" \
  --target-configuration '{
      "mcp": {
          "mcpServer": {
              "endpoint": "https://api.sec-api.io/mcp"
          }
      }
  }' \
  --credential-provider-configurations '[{
      "credentialProviderType": "API_KEY",
      "credentialProvider": {
          "apiKeyCredentialProvider": {
              "providerArn": "arn:aws:bedrock-agentcore:us-west-2:123456789012:token-vault/default/apikeycredentialprovider/sec-api-key",
              "credentialLocation": "HEADER",
              "credentialParameterName": "Authorization",
              "credentialPrefix": "Bearer"
          }
      }
  }]'
```

To send the key in the URL instead, set `credentialLocation` to
`QUERY_PARAMETER`, `credentialParameterName` to `apiKey`, and drop
`credentialPrefix`.

3. Poll `GetGatewayTarget` until `status` is `READY`. Creation indexes the tool
   list for you.

## Reload

Call `SynchronizeGatewayTargets` with your gateway identifier and the target ID
whenever the server's tool list changes. It returns HTTP 202 and runs
asynchronously.

## Verify

POST `tools/list` to your gateway MCP endpoint. Count the tools whose names start
with `secapi___`. Expect **49**.

## First prompt

Through your agent:

> List Apple's three most recent 10-K filings.

Expect one `secapi___filing-search` call. The answer names three filings with
accession numbers, filing dates and document URLs.

## Quirks

- **Tool names get a prefix.** The gateway builds names as
  `${target_name}___${tool_name}`, with three underscores. `filing-search`
  becomes `secapi___filing-search`. Strip the prefix in your own code.
- **Use a header, not the query string.** AWS states the endpoint URL must be
  encoded and the gateway reuses it as given. A bare URL plus a header credential
  avoids the question entirely.
- **Do not enable MCP sessions for this target.** This server is stateless. It
  issues no `Mcp-Session-Id`.
- **IAM SigV4 does not work.** SigV4 outbound auth needs a target that verifies
  SigV4 signatures. This server does not. Use API key or OAuth.
- **One credential provider per target.** `credentialProviderConfigurations`
  takes exactly one item.
- **Keep ListingMode at DEFAULT.** This server needs no user token, so the
  control-plane synchronization works. DYNAMIC mode drops semantic search.
- **Results arrive as one text block.** No tool declares an output schema, so
  there is no `structuredContent`.

## Removal

```bash
aws bedrock-agentcore-control delete-gateway-target \
  --gateway-identifier "your-gateway-id" \
  --target-id "your-target-id"

aws bedrock-agentcore-control delete-api-key-credential-provider \
  --name sec-api-key
```

## Source

[MCP servers targets | Amazon Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-target-MCPservers.html)
and
[Define the gateway target configuration](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-add-target-api-target-config.html).
