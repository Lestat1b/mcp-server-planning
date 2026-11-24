# MCP Server Planning — AKC Breed Explorer

## Project Overview

This project extends the existing **AKC Breed Explorer** RESTful application with a Model Context Protocol (MCP) server. The MCP server provides an AI client with a standardized way to search, retrieve, compare, create, update, and delete breed profiles through tools, read-only resources, and reusable prompts.

The design builds on the existing Flask application. Flask consumes Dog API v2 for public breed and group information, filters records identified by the provider as AKC-recognized, normalizes the data, and supplements it with locally maintained breed notes. The MCP server communicates with the Flask REST API; it does not call Dog API v2 directly.

This is an educational breed-research tool, not an official American Kennel Club registry. Breed considerations are not veterinary advice or guarantees about an individual dog.

---

## a. Underlying API Selection

### Selected public API: Dog API v2

- **Base URL:** `https://dogapi.dog/api/v2`
- **Documentation:** `https://dogapi.dog/docs/api-v2`
- **Purpose:** Supplies structured dog breed records, breed groups, characteristics, images, and dog facts.
- **Authentication:** Public access without an API key is documented by the provider.
- **Response format:** JSON using JSON:API conventions.

Dog API v2 is appropriate for an AI-assisted breed research application because its structured records can provide breed names, descriptions, life-expectancy ranges, weight ranges, traits, images, and recognition information. The existing Flask service filters records using the provider's `recognized_by` information and normalizes them for the AKC Breed Explorer interface.

Dog API v2 is a read-oriented upstream source. It does not provide the application's locally curated pros and cons or the complete create, update, and delete operations required for this assignment. The architecture therefore has two REST layers:

1. **Dog API v2** supplies public breed and group data.
2. **The project's Flask REST API** provides normalized breed resources and CRUD operations for locally maintained breed profiles.

The MCP server calls the Flask API. Flask continues to retrieve public information from Dog API v2 and manage locally stored profile records. This avoids duplicating the public API integration in the MCP server.

### Resource identification and statelessness

The Flask API identifies resources through stable resource-oriented paths:

- `/api/breeds` identifies the breed collection.
- `/api/breeds/<slug>` identifies one breed profile.
- `/api/groups` identifies breed groups.
- `/api/facts` identifies dog facts.

Each HTTP request includes the parameters or JSON body needed for that operation. The Flask service does not depend on a previous browser or MCP request to understand the current request. This is REST statelessness. A short-lived cache of public Dog API responses may improve performance, but it does not represent a user's conversational state.

### CRUD extension to the Flask REST API

The existing project plans `GET /api/breeds`, `GET /api/breeds/<slug>`, `GET /api/groups`, `GET /api/facts`, and `POST /api/breeds`. To satisfy the four CRUD specifications, the Flask API will be extended as follows:

| CRUD operation | HTTP method and route | Purpose |
|---|---|---|
| Read collection | `GET /api/breeds` | Return breed profiles; support optional `q`, `group`, `size`, and `activity` filters. |
| Read one | `GET /api/breeds/<slug>` | Return one normalized breed profile or HTTP 404. |
| Create | `POST /api/breeds` | Validate and create a locally maintained breed profile; return HTTP 201 and the created profile. |
| Update | `PUT /api/breeds/<slug>` | Validate a complete replacement profile and update the matching local profile; return the updated profile or HTTP 404. |
| Delete | `DELETE /api/breeds/<slug>` | Delete a locally maintained profile; return HTTP 204 or HTTP 404 if it does not exist. |

`PUT` and `DELETE` apply only to locally maintained profiles, not records owned by Dog API v2. Public breed records remain read-only. The Flask service must distinguish locally managed profiles from upstream records so an AI client cannot overwrite or delete public source data.

The existing project uses `data/breeds.json` for locally maintained profiles. The update and delete implementations will read the file, locate a profile by normalized slug, validate the operation, and write the resulting collection safely. A database would be a stronger persistence option for a multi-user production deployment.

Before public deployment, write endpoints must enforce authentication and authorization, validate all fields on the server, protect against unsafe file access, and record appropriate audit information. The coursework application is a local demonstration and should not expose unauthenticated write access to the internet.

---

## b. MCP Capability Mapping

The MCP server exposes three capability categories: tools, resources, and prompts. MCP messages use JSON-RPC 2.0 request and response structures. The MCP client discovers the server's available capabilities and invokes them by name rather than constructing arbitrary HTTP requests.

### MCP tools — `@mcp.tool`

Tools are executable functions. They call the project's Flask REST API using the appropriate HTTP methods. Input parameters are declared in Python type hints and validated by the MCP SDK's generated input schema, with additional validation inside each function.

| Tool | Parameters / input schema | REST operation | Return type |
|---|---|---|---|
| `search_breeds` | `query: str = ""`, `group: str = ""`, `size: str = ""`, `activity: str = ""` | `GET /api/breeds` with optional query parameters | JSON text containing count, source, and matching profiles |
| `get_breed_details` | `slug: str` | `GET /api/breeds/{slug}` | JSON text containing one profile |
| `list_breed_groups` | No required parameters | `GET /api/groups` | JSON text containing group names and source |
| `get_dog_facts` | `limit: int = 1`, valid range 1–5 | `GET /api/facts?limit={limit}` | JSON text containing facts |
| `create_breed_profile` | `profile: dict` with required profile fields | `POST /api/breeds` | JSON text containing the created profile |
| `update_breed_profile` | `slug: str`, `profile: dict` | `PUT /api/breeds/{slug}` | JSON text containing the updated profile |
| `delete_breed_profile` | `slug: str`, `confirm: bool` | `DELETE /api/breeds/{slug}` | Confirmation text identifying the deleted local profile |

The profile input includes `name`, `group`, `size`, `lifespan`, `activity`, `grooming`, `temperament`, `pros`, `cons`, and `akc_url`. The server requires `size` to be `Small`, `Medium`, or `Large`; `pros` and `cons` must be arrays of non-empty strings; required text fields cannot be blank; and the AKC reference URL must pass the Flask API's accepted URL validation.

The delete tool includes an explicit `confirm` parameter. It refuses to delete when confirmation is false. The model should explain the target profile and obtain user confirmation before invoking this destructive operation.

### MCP resources — `@mcp.resource`

Resources are read-only data sources that the model can inspect. They map to GET operations only and never create, update, or delete records.

| Resource URI | Description | Underlying REST request |
|---|---|---|
| `akcbreed://catalog` | Current normalized breed catalog | `GET /api/breeds` |
| `akcbreed://breeds/{slug}` | One breed profile | `GET /api/breeds/{slug}` |
| `akcbreed://groups` | Available breed groups | `GET /api/groups` |
| `akcbreed://facts` | A small set of dog facts | `GET /api/facts?limit=3` |

The `akcbreed://` URI scheme provides a recognizable namespace. The catalog URI identifies the collection, while the `{slug}` URI template identifies a specific breed. These predictable URIs allow an MCP client to discover resources and request only the data fragment it needs. Resource responses are serialized as JSON text with an `application/json` MIME type.

Resources do not expose write operations. A client that needs to change data must invoke the corresponding tool.

### MCP prompts — `@mcp.prompt`

Prompts are reusable conversation templates that help the model collect missing information and formulate a safe, well-structured request. A prompt does not itself execute a REST request or mutate data.

| Prompt name | Purpose and expected context |
|---|---|
| `breed_recommendation` | Guides the model to ask about living space, activity level, grooming tolerance, household needs, and preferred size before searching. |
| `breed_comparison` | Requests two or more breed names or slugs and guides comparison of lifespan, size, activity, grooming, temperament, pros, and cons. |
| `create_breed_profile_guide` | Guides a user through required profile fields and checks that they are present before the create tool is called. |
| `update_breed_profile_guide` | Requests the target slug and complete replacement profile, summarizes the intended change, and asks for confirmation before the update tool is called. |
| `delete_breed_profile_guide` | Requests the target slug, explains that deletion is destructive, and requires explicit confirmation before the delete tool is called. |

Prompts will tell the model that generated pros and cons are general considerations, not official AKC ratings. The model should distinguish provider-supplied information from locally curated notes and should not present breed traits as guarantees.

### JSON-RPC and transport

MCP uses JSON-RPC 2.0 messages for capability discovery and operations such as listing tools, calling tools, listing resources, reading resources, and retrieving prompts. JSON-RPC defines the message format; it is not itself the network transport.

The planned deployment transport is **Streamable HTTP over HTTPS**. It supports a separately hosted MCP server that communicates with compatible remote MCP clients. The server will use the MCP Python SDK's Streamable HTTP transport. During local development, the server may also be launched with `stdio` for convenient testing in a local MCP host. The documented deployment target is Streamable HTTP. Server-Sent Events (SSE) will not be the primary transport.

---

## c. Interaction Data Flow (Textual)

### End-to-end request sequence

1. A user asks an AI application to find a breed, compare breeds, retrieve a dog fact, or manage a locally maintained profile.
2. The AI application acts as the **MCP Host**. Its MCP Client establishes a connection to the MCP Server using Streamable HTTP over HTTPS.
3. The MCP Client and Server exchange initialization and capability information using JSON-RPC 2.0. The client discovers available tools, resources, and prompts.
4. The model interprets the user's request. If required information is missing, it uses an appropriate prompt to ask follow-up questions.
5. The model requests a tool invocation through the MCP Client. The request includes the tool name and structured arguments.
6. The MCP Server validates the arguments against the tool's input schema and its own business rules.
7. The selected tool uses an asynchronous HTTP client (`httpx.AsyncClient`) to call the corresponding Flask REST endpoint. The MCP server does not call Dog API v2 directly.
8. The Flask API validates the REST request. For public breed data, Flask calls Dog API v2 as needed, normalizes the JSON:API response, and returns application JSON. For local profile changes, Flask validates and updates its local data store.
9. The MCP Server checks the HTTP status, parses the JSON response, and converts the result into MCP tool content, normally readable text containing formatted JSON.
10. The MCP Client delivers the tool result to the model. The model summarizes the result for the user, including source and limitations where appropriate.

### Textual architecture diagram

```text
User
  |
  v
AI Application / MCP Host
  |
  v
LLM <----> MCP Client
              |
              | MCP messages (JSON-RPC 2.0)
              | Streamable HTTP over HTTPS
              v
        MCP Server (Python)
        - @tool functions
        - @resource definitions
        - @prompt templates
        - input validation
        - error translation
              |
              | Asynchronous HTTP (httpx)
              v
        Flask REST API
        - GET /api/breeds
        - GET /api/breeds/<slug>
        - GET /api/groups
        - GET /api/facts
        - POST /api/breeds
        - PUT /api/breeds/<slug>
        - DELETE /api/breeds/<slug>
              |
        +-----+---------------------+
        |                           |
        v                           v
  Dog API v2 (GET)          Local profile storage
  breeds / groups / facts   data/breeds.json
```

### Asynchronous REST communication and statelessness

Tool functions will be asynchronous and use `httpx.AsyncClient` with a finite timeout. Each invocation sends all required parameters in that request, checks the response status, and closes the client appropriately. No tool depends on a previous tool call's local variables or conversational history. If a value from an earlier operation is needed, the model must pass it explicitly as a parameter.

A short-lived HTTP connection pool or cache may be used for efficiency, but it does not change the independent nature of each request. The MCP server will not store hidden conversational state to decide what a tool means.

### Input schema validation

The MCP SDK derives JSON input schemas from tool function type hints and descriptions. The MCP Server validates incoming JSON-RPC tool arguments against that schema before the tool function runs. The function then performs business validation, including checking slug format, accepted size values, list contents, URL format, and numeric ranges. The Flask API repeats validation at its own trust boundary. Validation in the MCP layer improves model feedback; validation in Flask protects the REST service from every client, not only MCP.

### Decorator implementation example

The following is representative FastMCP-style pseudocode. The final implementation will use the exact import and server initialization supported by the selected MCP Python SDK version.

```python
from fastmcp import FastMCP
import httpx
import json

mcp = FastMCP("AKC Breed Explorer MCP")
REST_API = "http://127.0.0.1:5000"

@mcp.tool(
    name="get_breed_details",
    description=(
        "Retrieve a normalized AKC Breed Explorer profile by its "
        "URL slug. Returns breed characteristics, source information, "
        "and available pros and cons."
    ),
)
async def get_breed_details(slug: str) -> str:
    """Fetch one breed profile from the Flask REST API."""
    if not slug or not slug.strip():
        return "Input error: provide a non-empty breed slug."

    async with httpx.AsyncClient(timeout=12.0) as client:
        response = await client.get(
            f"{REST_API}/api/breeds/{slug.strip()}"
        )

    response.raise_for_status()
    return json.dumps(response.json(), ensure_ascii=False, indent=2)
```

This example illustrates a descriptive tool name, clear description, typed parameter, asynchronous HTTP GET, and JSON serialization. The implementation will add centralized exception handling so HTTP errors return controlled, understandable messages rather than raw tracebacks.

---

## d. Prompt Context & Parameter Extraction

### AI form handling

A traditional web form asks a person to fill in fields. In an AI interface, a prompt guides the conversation so the model can identify the same required values from natural language, ask for missing information, and call a tool with structured arguments.

For example, if a user asks, "Find a medium-sized dog that is good for an active household," the `breed_recommendation` prompt guides the model to identify:

- Desired size: `Medium`
- Activity preference: active / high activity
- Other useful context: living space, grooming tolerance, household needs, and experience with dogs

The model asks follow-up questions when important details are missing rather than inventing values. It then calls `search_breeds` with the extracted parameters. The tool schema and Flask API validate the values.

For profile creation, the `create_breed_profile_guide` prompt elicits the name, group, size, lifespan, activity, grooming, temperament, pros, cons, and AKC reference URL. It organizes the information into the expected JSON object. The model must not call the create tool until required fields are present and valid.

For updates, the model collects the target slug and a complete replacement profile because the planned endpoint uses PUT semantics. It summarizes the replacement and requests user confirmation before calling the update tool. For deletion, the model identifies the target local profile, explains the consequence, and obtains explicit confirmation before calling the delete tool.

### Tool validation and user-friendly errors

The MCP tool functions validate:

- Required strings are present and are not only whitespace.
- Slugs contain only expected lowercase letters, digits, and hyphens.
- Size is one of `Small`, `Medium`, or `Large`.
- `pros` and `cons` are arrays containing non-empty strings.
- The AKC URL uses an accepted HTTPS AKC URL pattern.
- The facts limit is an integer from 1 through 5.
- Update and delete targets exist and are locally editable.
- Delete requests include `confirm=true`.

When the model supplies malformed data, the tool returns a concise correction message. For example:

```text
Input error: size must be Small, Medium, or Large.
Please correct the size value and try again.
```

If a breed slug does not exist, the tool reports that no matching profile was found and suggests using `search_breeds` to discover valid slugs.

### Separation of prompts, resources, and tools

Prompts only guide the conversation and help the model collect and organize information. They do not call REST endpoints and do not change application data. Resources expose read-only information through GET-backed operations. Tools perform the actual REST operations.

This separation is important for safety:

- **Prompt:** helps the model gather and confirm the information needed.
- **Resource:** provides read-only catalog, group, fact, or breed data.
- **Tool:** performs a defined operation, including GET retrieval or the state-changing POST, PUT, and DELETE operations.

A prompt cannot bypass tool validation. Every state-changing operation must pass through a specifically defined tool and the Flask API's validation and authorization controls.

### REST and MCP error propagation

The MCP server translates REST failures into concise messages that the model can understand:

| REST response or failure | MCP-facing message |
|---|---|
| HTTP 400 | `The REST API rejected the input. Check the required fields and allowed values: [safe error detail].` |
| HTTP 404 | `No breed profile was found for slug '[slug]'. Search the catalog for a valid slug.` |
| HTTP 409 | `A profile with this identifier already exists or the requested change conflicts with current data.` |
| HTTP 500 | `The breed service encountered an internal error. Please try again later. Reference: [request ID, if available].` |
| HTTP 502 / 503 / 504 | `The upstream breed data service is temporarily unavailable. Please retry later.` |
| Network timeout | `The breed service did not respond before the request timed out. Please retry.` |
| Invalid JSON response | `The breed service returned data in an unexpected format. The request could not be completed safely.` |

The server will not expose stack traces, secrets, local file paths, or sensitive internal details to the model. It will preserve useful HTTP status information and safe error details. Transient failures may be retried a limited number of times for safe GET requests. POST, PUT, and DELETE will not be blindly retried because repeating a state-changing request could create duplicate or unintended effects.

For successful responses, the server parses the Flask JSON response and returns it as MCP text content, formatted as readable JSON. Tool results identify the data source when the REST response supplies it. The model can then summarize the result without changing the meaning of the source data.

---

## e. Technology Stack

| Component | Planned technology | Purpose |
|---|---|---|
| Programming language | Python 3.11 or newer | MCP server and HTTP integration |
| MCP implementation | MCP Python SDK or compatible FastMCP library | Defines tools, resources, prompts, and protocol behavior |
| MCP transport | Streamable HTTP over HTTPS | Connects a remote MCP client to the server |
| Local development transport | `stdio` | Supports local MCP host and test-client development |
| Existing REST framework | Flask | Hosts the AKC Breed Explorer REST API |
| Async HTTP client | `httpx` | Makes asynchronous requests from MCP tools to Flask |
| Data format | JSON / JSON:API upstream | Exchanges normalized profile data and public API data |
| Validation | Python type hints, MCP-generated JSON schemas, and explicit server-side checks | Validates tool arguments and profile data |
| Local persistence | Existing JSON file initially; database considered for production | Stores locally curated profiles |
| Testing | `pytest`, `pytest-asyncio`, and MCP SDK test client or in-memory client | Tests tool behavior, resources, prompts, and integration |
| Configuration | Python virtual environment and environment variables | Keeps URLs and deployment settings configurable |

The MCP server runs as a separate process from Flask. The REST base URL is configurable through an environment variable rather than hard-coded for deployment. HTTPS and authentication are required for a remotely accessible production service.

---

## f. Implementation Steps

1. **Review and extend the Flask REST API.**
   - Review the current AKC Breed Explorer routes and data model.
   - Confirm existing GET and POST behavior.
   - Implement `PUT /api/breeds/<slug>` for complete profile replacement.
   - Implement `DELETE /api/breeds/<slug>` for locally maintained profiles.
   - Ensure upstream Dog API records remain read-only.
   - Add server-side validation, clear status codes, and safe persistence behavior.
   - Add authentication and authorization before any public deployment of write operations.

2. **Create the MCP project structure.**
   - Create a new repository named `mcp-server-planning` for this planning deliverable.
   - For the eventual implementation, create a separate project directory or repository for the Python MCP server.
   - Add `mcp_server.py`, `requirements.txt`, `.env.example`, `README.md`, and a `tests/` directory.
   - Create and activate a Python virtual environment.
   - Install the MCP Python SDK / FastMCP, `httpx`, and testing dependencies.

3. **Configure the server and REST connection.**
   - Initialize the MCP server with a descriptive name.
   - Read the Flask REST API base URL from an environment variable.
   - Configure asynchronous HTTP timeouts and response handling.
   - Keep credentials and secrets out of source control.

4. **Implement the retrieval tools.**
   - Add `search_breeds`, `get_breed_details`, `list_breed_groups`, and `get_dog_facts`.
   - Add tool descriptions and typed input parameters.
   - Validate model input before making a REST request.
   - Preserve source attribution in returned data.

5. **Implement the CRUD tools.**
   - Add `create_breed_profile` using POST.
   - Add `update_breed_profile` using PUT.
   - Add `delete_breed_profile` using DELETE.
   - Require explicit confirmation for deletion and confirmation of the proposed replacement before updates.
   - Ensure these tools operate only on locally maintained profiles.

6. **Define read-only resources.**
   - Implement the catalog, individual breed, groups, and facts resource URIs.
   - Map each resource to the corresponding GET endpoint.
   - Return JSON with the `application/json` MIME type.
   - Add short-lived caching only if testing shows it is useful; do not cache write operations as though they were read-only.

7. **Create prompt templates.**
   - Implement recommendation, comparison, create, update, and delete prompts.
   - Specify the information each prompt should collect.
   - Instruct the model to ask for missing required information and distinguish public API data from local notes.
   - Ensure prompts guide the model but do not execute operations.

8. **Implement error handling and logging.**
   - Translate REST status codes, timeouts, invalid JSON, and connection failures into safe, readable MCP tool results.
   - Avoid returning stack traces or secrets.
   - Log operational events and request identifiers without logging unnecessary user data.
   - Limit retries to appropriate transient, read-only operations.

9. **Test the Flask REST API.**
   - Test GET collection and individual-profile routes.
   - Test valid and invalid POST requests.
   - Test valid PUT updates, missing profiles, and invalid replacement data.
   - Test DELETE success, missing profiles, and attempts to delete upstream-only records.
   - Verify local persistence remains valid after create, update, and delete operations.

10. **Test the MCP server.**
    - Use an MCP test client or in-memory client to initialize the server and discover tools, resources, and prompts.
    - Invoke every tool with valid and invalid parameters.
    - Read each resource URI and verify its JSON content.
    - Retrieve each prompt and verify that its instructions and expected inputs are clear.
    - Test HTTP 400, 404, 409, 500, timeout, and malformed-response scenarios.
    - Verify state-changing tools cannot bypass confirmation, validation, or authorization.

11. **Perform end-to-end integration testing.**
    - Start the Flask REST API and MCP server.
    - Connect an MCP client using the selected transport.
    - Ask the model to search for breeds, retrieve a profile, and compare results.
    - Create a test local profile, update it, and delete it after confirmation.
    - Confirm the model receives understandable results and errors.
    - Confirm the MCP server never accesses Dog API v2 directly.

12. **Review, document, and deploy.**
    - Document server setup, environment variables, transport, tool schemas, resource URIs, prompts, and error behavior.
    - Document the REST API contract, including GET, POST, PUT, and DELETE.
    - Review security controls, input validation, logging, and data-source limitations.
    - Run the complete test suite before release.
    - Deploy the Flask REST API and MCP server in a controlled environment with HTTPS and appropriate authentication.
    - Verify that the chosen MCP host can discover and invoke the deployed capabilities.

---

## Testing and Acceptance Criteria

The plan will be considered complete when:

- The Flask REST API supports the required GET, POST, PUT, and DELETE operations for the intended resources.
- Public Dog API records remain read-only, while local profiles can be created, updated, and deleted.
- The MCP client can discover all documented tools, resources, and prompts.
- Tool schemas reject missing, malformed, and out-of-range inputs.
- Resources return read-only data from the matching GET endpoints.
- Prompts guide parameter collection without directly changing data.
- Successful tool calls return structured, understandable JSON content.
- REST errors and network failures are translated into safe, useful messages.
- Integration tests demonstrate the complete path from MCP client through the MCP server and Flask API.
- Write operations require appropriate authorization and explicit confirmation for destructive actions.

---

## Conclusion

The AKC Breed Explorer MCP server provides a semantic interface between an AI model and the existing Flask REST application. Dog API v2 remains the public source for breed information, Flask remains responsible for normalization and local profile CRUD operations, and the MCP server exposes those capabilities through tools, read-only resources, and prompts.

Using JSON-RPC 2.0 over Streamable HTTP separates MCP message semantics from the network transport. Typed tool schemas, independent stateless requests, asynchronous HTTP calls, resource URI templates, and clear error translation make the integration understandable and predictable for an AI client. The design also separates read-only information from state-changing operations and reserves write access for validated, authorized tools.

## References

Dog API. (n.d.). *Dog API v2 documentation*. https://dogapi.dog/docs/api-v2

Model Context Protocol. (n.d.). *Model Context Protocol documentation*. https://modelcontextprotocol.io/

Model Context Protocol. (n.d.). *Python SDK*. GitHub. https://github.com/modelcontextprotocol/python-sdk

FastMCP. (n.d.). *FastMCP documentation*. https://gofastmcp.com/

HTTPX. (n.d.). *Async support*. https://www.python-httpx.org/async/

Flask. (n.d.). *Flask documentation*. https://flask.palletsprojects.com/

