# Chapter 15: Using the HTTP Request Node in n8n
## Connect APIs and External Services

## Learning Objectives
By the end of this chapter, you will be able to:
- Explain what an API does and how it differs from a webhook.
- Describe the purpose of the HTTP Request node.
- Configure a request method, URL, authentication, query parameters, headers, and body.
- Use an API's documentation to configure a request.
- Import a cURL command to configure the node.
- Recognize common HTTP Request node use cases, including retrieving data, downloading files, and pagination.

## 1. What Is an API?
An **API (Application Programming Interface)** allows different applications to exchange data or request actions from each other.

An API commonly follows a request-and-response pattern:
1. A client sends a request to a service.
2. The service processes the request.
3. The service returns data or a confirmation.

Examples:
- **Weather service:** request a forecast for a location and receive weather data.
- **Maps service:** send coordinates and receive directions.
- **CRM:** send customer details and receive confirmation that they were saved.

APIs make it possible for applications to exchange information or trigger actions automatically.

![](n8n_http_node_1.png)

## 2. API vs Webhook

| API request | Webhook |
|---|---|
| A client asks a service for data or an action. | A service sends data when a configured event occurs. |
| Often initiated by the workflow when it needs information. | Often triggered by an event, such as an order or payment. |
| Analogy: making a phone call to ask a question. | Analogy: receiving an alert or text message. |

Examples of webhook events include an online store notifying a workflow when an order is placed, a payment service notifying it when a payment succeeds, or a chat application notifying it when a message arrives.

This chapter focuses on making API requests with n8n's **HTTP Request** node. A webhook can start or notify a workflow; an HTTP request lets a workflow call an external API.

## 3. What Is the HTTP Request Node?
The **HTTP Request** node is a built-in API client in n8n. It sends HTTP requests to external services and returns their responses. It is especially useful when:
- n8n does not have a dedicated node for the service or operation you need.
- A dedicated integration does not expose a particular API feature.
- You need to call a REST API directly.

The node's configuration should follow the external service's API documentation. The API documentation tells you the endpoint, supported method, required parameters, authentication method, headers, and request body.

## 4. Main Configuration Fields

### 4.1 Method
The HTTP method describes the operation being requested.

| Method | Common purpose |
|---|---|
| `GET` | Retrieve data |
| `POST` | Submit data or create a resource |
| `PUT` | Update or replace a resource, depending on the API |
| `DELETE` | Request deletion of a resource |

The API's documentation is the authority for which method to use. Some services implement deletion as a status update rather than permanent deletion, so follow the service's documented behavior.

### 4.2 URL
The URL is the endpoint receiving the request. Different API operations may use different endpoints. For example, a service may have one endpoint for a one-day forecast and another for a ten-day forecast.

Use the exact endpoint and path required by the API documentation.

### 4.3 Authentication
Some APIs expose public endpoints, while others require proof that the request is authorized.

The HTTP Request node offers predefined credentials for supported services and generic authentication options for other APIs.

Common authentication patterns include:
- **None:** for endpoints that do not require authentication.
- **Basic authentication:** usually a username and password.
- **Bearer token:** sends a token that the service validates.
- **OAuth2:** obtains and uses authorization through an OAuth flow.
- **API key:** a service-specific key, which may be sent in a query parameter or header.

Use the method required by the API. Store secrets in n8n credentials where appropriate rather than hard-coding them into workflow fields or code. Do not share real API keys in screenshots, exported examples, or public repositories.

### 4.4 Query Parameters
Query parameters are key-value pairs included in the URL, commonly after a `?`. They can specify which record to retrieve, the location, the page number, the page size, or other filters.

For example:

```text
https://api.example.com/weather?location=NewYork&units=metric
```

In n8n, enable **Send Query Parameters** and enter each parameter's name and value in the node's fields. Prefer this structured configuration to manually assembling a long URL when practical.

### 4.5 Headers
Headers provide additional information about a request. An API may require headers for authentication, content type, or response format.

Examples include:
- `Authorization: Bearer <token>`
- `Content-Type: application/json`
- `Accept: application/json`

Only send the headers required by the service's documentation. The response may also contain headers; n8n can be configured to include response headers when needed.

### 4.6 Body
The request body carries data sent to the API. It is often used with `POST` and `PUT`, although the exact requirements depend on the service.

The HTTP Request node supports different body formats, such as:
- JSON
- Form URL-encoded data
- Form data
- n8n binary data for file-related requests

For example, a JSON request body might look like this:

```json
{
  "name": "Riya",
  "email": "riya@example.com"
}
```

Do not add a body simply because the field is available. If the endpoint expects only a `GET` request with query parameters, follow that documented format.

## 5. Example: Fetching a Weather Forecast

The example in this chapter uses a weather API to demonstrate the request configuration. The same steps apply to many APIs.

### Step 1: Read the API documentation
Find the specific operation you want, such as a one-day forecast. Note:
- HTTP method
- Endpoint URL
- Required query parameters
- Authentication requirements
- Any required headers or body

### Step 2: Add the HTTP Request node
Create a workflow and add a Manual Trigger for testing. Add an **HTTP Request** node and connect it to the trigger.

### Step 3: Set the method and URL
Select the documented method, such as `GET`, and enter the endpoint URL.

### Step 4: Configure the API key
If the API requires a key as a query parameter, enable **Send Query Parameters** and add the parameter name specified by the documentation. Enter your own valid key; do not copy a key from a tutorial or expose it publicly.

If the API instead expects a header or a credential type, configure it accordingly. Authentication methods are service-specific.

### Step 5: Execute and inspect the response
Run the node. A successful response may contain the requested forecast, including location, minimum and maximum temperatures, units, and conditions. Inspect the output JSON to identify fields that later workflow nodes can use.

The important pattern is:

```text
Manual Trigger
      |
      v
HTTP Request (GET weather endpoint)
      |
      v
Inspect response JSON
      |
      v
Use the returned data in later nodes
```

## 6. Configure a Request by Importing cURL

**cURL** is a command-line tool often used in API documentation to represent an HTTP request. A cURL example can include the URL, method, headers, query parameters, and body.

The HTTP Request node can import a supported cURL command and use it to populate its configuration.

### Workflow
1. Open the API documentation and find its cURL example.
2. Copy the complete command.
3. Open the HTTP Request node in n8n.
4. Choose the **Import cURL** option.
5. Paste the command and import it.
6. Review the generated configuration, especially authentication and any secret values.
7. Execute the node and inspect the response.

Importing cURL can save manual configuration time, especially for requests with several headers or query parameters. Always review the imported settings; a command may include credentials, and not every cURL option necessarily maps exactly to a node setting.

## 7. Other Common HTTP Request Use Cases

### 7.1 Retrieve JSON data
A basic request can call an endpoint and return data without query parameters, headers, or a body if the endpoint does not require them.

### 7.2 Retrieve a file
Some endpoints return a file rather than a JSON object. Configure the response format according to the API and the HTTP Request node's available options. A downloaded file can be returned as binary data for later workflow steps.

### 7.3 Follow redirects
An endpoint may redirect the request to another URL. The node can offer redirect-related options, including a maximum number of redirects. Configure these according to the endpoint's behavior and the options available in your n8n version.

### 7.4 Pagination
APIs often divide large result sets into pages. Query parameters may specify a page number and page size. A workflow can request successive pages and combine the results. The exact pagination strategy depends on the API: some use page numbers, while others use cursors or next-page URLs.

### 7.5 Include response metadata
Response headers and status information can help determine whether a request succeeded and understand the returned content. Enable the relevant response options when you need this information for troubleshooting or workflow logic.

## 8. Troubleshooting Checklist

### The request fails
- Confirm the URL and endpoint path.
- Verify the HTTP method against the API documentation.
- Check that all required query parameters are present.
- Confirm that authentication is configured correctly.
- Check required headers and body format.
- Inspect the response status and error message.

### The API returns an authorization error
- Confirm that the API key or token is valid and active.
- Verify whether the service expects the key in a query parameter, header, or credential field.
- Check the permissions associated with the credential.
- Avoid exposing secrets in workflow exports or screenshots.

### The request succeeds but returns unexpected data
- Check filters, location values, page number, and page size.
- Confirm that the chosen endpoint returns the type of data you expect.
- Inspect the response JSON or file output.
- Check whether pagination is needed to retrieve all records.

### Importing cURL does not produce the expected request
- Confirm that the complete command was copied.
- Review the imported method, URL, query parameters, headers, and body.
- Reconfigure credentials securely.
- Compare the node settings with the API documentation.

## 9. Good Practices
1. Start with the API documentation rather than guessing parameters.
2. Test a simple request before adding complexity.
3. Use n8n credentials for secrets when appropriate.
4. Do not expose real API keys in examples, screenshots, or repositories.
5. Use query parameters, headers, and body fields for their intended purposes.
6. Inspect the response before connecting more nodes.
7. Add pagination logic when an API returns results in pages.
8. Handle API errors and rate limits as part of a production workflow.
9. Review imported cURL settings before executing them.

## 10. Practice Questions

1. What does API stand for?
2. What is the main difference between an API request and a webhook?
3. What is the purpose of the HTTP Request node?
4. Which HTTP method is commonly used to retrieve data?
5. What is the difference between query parameters and headers?
6. When might a request body be required?
7. What is cURL, and how can it help configure an HTTP Request node?
8. Why should API keys be handled carefully?
9. Why might pagination be needed?
10. What should you inspect when an HTTP request fails?

### Answer Key
1. Application Programming Interface.
2. An API request is initiated to ask a service for data or an action; a webhook sends data when a configured event occurs.
3. It allows n8n to make HTTP requests to external services and use their responses.
4. `GET`.
5. Query parameters commonly provide values or filters in the URL; headers provide metadata or instructions for the request.
6. When the endpoint expects data to be sent in the request, often for operations such as creating or updating a resource.
7. cURL is a command-line tool that can represent an HTTP request; importing a cURL command can populate node settings.
8. A key can grant access to an API or account, so exposing it can allow unauthorized use.
9. An API may split large result sets into multiple pages.
10. Check the URL, method, parameters, authentication, headers, body, response status, and error message.

## Chapter Summary
The HTTP Request node lets n8n communicate with APIs even when a dedicated integration is unavailable or does not expose the required operation. A request is configured using the method, URL, authentication, query parameters, headers, and—when required—a body. API documentation is the primary guide for these settings. Importing cURL can accelerate setup, while inspecting responses and planning for pagination and errors helps make workflows more reliable.
