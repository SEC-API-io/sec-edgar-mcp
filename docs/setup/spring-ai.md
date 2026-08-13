# Connect Spring AI to SEC EDGAR Data with MCP

Spring AI adds AI building blocks to Spring Boot. Its MCP client boot starter
configures MCP clients from `application.yml` and exposes their tools as a
`ToolCallbackProvider`. It speaks Streamable HTTP.

## Prerequisites

- Java 17 or later, and a Spring Boot application.
- Spring AI 2.0.0 or later.
- An SEC-API.io key from [sec-api.io](https://sec-api.io/profile).

```xml
<dependency>
  <groupId>org.springframework.ai</groupId>
  <artifactId>spring-ai-starter-mcp-client</artifactId>
</dependency>
```

Use this starter, not the WebFlux one. The header customizer needs the JDK
`HttpClient` transport.

## Config location

Two files. `src/main/resources/application.yml` holds the connection, and a
`@Configuration` class holds the auth header, for example
`src/main/java/com/example/McpConfiguration.java`.

## Configuration

```yaml
spring:
  ai:
    mcp:
      client:
        type: SYNC
        request-timeout: 35s
        streamable-http:
          connections:
            sec-api:
              url: https://api.sec-api.io
              endpoint: /mcp
```

The key cannot go in the URL. See the quirks. Send it as a header:

```java
@Configuration
class McpConfiguration {
  @Bean
  McpSyncHttpClientRequestCustomizer secApiAuth(
      @Value("${SEC_API_KEY}") String apiKey) {
    return (builder, method, endpoint, body, context) ->
        builder.header("Authorization", "Bearer " + apiKey);
  }
}
```

## Reload

Restart the application. The starter connects at startup and lists the tools
once.

## Verify

Inject `SyncMcpToolCallbackProvider` and read
`provider.getToolCallbacks().length`. Expect **49**. HTTP 401 means the key is
wrong.

## First prompt

> Find the 3 most recent 10-K filings from Apple. Give the filing date and the
> accession number of each.

The model calls `filing-search` once with `ticker:AAPL AND formType:"10-K"`. It
answers with three rows. Each row carries a `filedAt` date and an accession
number such as `0000320193-25-000073`.

## Quirks

- **Never put `?apiKey=` in `url`.** Spring AI strips the query string, because
  `WebClient.baseUrl()` keeps only the scheme, the host and the port. The
  request then carries no key and the server answers 401. This is open issue
  [spring-ai#6505](https://github.com/spring-projects/spring-ai/issues/6505).
  The header customizer above is the workaround.
- **Pick SYNC or ASYNC once.** The starter cannot mix the two client types.
- **The server is stateless.** `GET` and `DELETE` return 404. There is no SSE
  stream to fall back to.
- Results arrive as one text block of stringified JSON. Errors return the text
  `sec-api error: <message>`.

## Removal

Delete the `sec-api` block under `streamable-http.connections` and the
customizer bean. Drop the starter dependency if no other MCP server remains.

## Source

[MCP Client Boot Starter, Spring AI reference](https://docs.spring.io/spring-ai/reference/api/mcp/mcp-client-boot-starter-docs.html)
