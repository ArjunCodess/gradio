# Server mode (`gr.Server`)

Tags: API, MCP, FASTAPI, SERVER

`gr.Server` is Gradio's backend without Gradio's UI. You keep the queue, streaming, MCP tools, ZeroGPU, and Spaces hosting, and you bring your own frontend (HTML, React, Svelte, or none at all).

Talk to it with the [JavaScript client](/guides/getting-started-with-the-js-client) or the [Python client](/guides/getting-started-with-the-python-client). That is the intended way to build a custom Gradio app: `gr.Server` on the Python side, `@gradio/client` in the browser.

To do this, instantiate `gr.Server` directly. It is a FastAPI server with Gradio's API engine built in.

## When to use `gr.Server`

Use `gr.Server` instead of `gr.Blocks` when any of the following apply:

- You want a **completely custom (potentially vibe-coded) UI** (your own HTML, React, Svelte, etc.) powered by Gradio's backend
- You want **full FastAPI control** (custom GET/POST routes, middleware, dependency injection) alongside Gradio API endpoints
- You're building a service to **host on Spaces** with or without ZeroGPU but don't need Gradio components

If you're happy with Gradio's built-in UI components, use `gr.Blocks`, `gr.ChatInterface`, or `gr.Interface` instead.

## Installation

`gr.Server` is included in the main Gradio package. If you want MCP support, install the extra:

```bash
pip install "gradio[mcp]"
```

## A Minimal Example

Here's the simplest possible `gr.Server` app — a single API endpoint with no UI:

```python
import gradio as gr

app = gr.Server()

@app.api(name="hello")
def hello(name: str) -> str:
    return f"Hello, {name}!"

app.launch()
```

That's it. When you run this script, you get:

- A Gradio API endpoint at `/gradio_api/call/hello` with queuing and SSE streaming
- Auto-generated API docs at `/gradio_api/info`
- A Python and JavaScript client that can call `/hello` by name

You can test it with the Gradio Python client:

```python
from gradio_client import Client

client = Client("http://localhost:7860")
result = client.predict("World", api_name="/hello")
print(result)  # "Hello, World!"
```

Or from JavaScript, which is what a custom frontend should use:

```js
import { Client } from "@gradio/client";

const app = await Client.connect("http://localhost:7860");
const result = await app.predict("/hello", { name: "World" });
console.log(result.data);  // ["Hello, World!"]
```

## Custom Routes

Since `gr.Server` inherits from FastAPI, you can add any route directly:

```python
import gradio as gr
from fastapi.responses import HTMLResponse

app = gr.Server()

@app.api(name="hello")
def hello(name: str) -> str:
    return f"Hello, {name}!"

@app.get("/", response_class=HTMLResponse)
async def homepage():
    return "<h1>Welcome to my API</h1>"

@app.get("/health")
async def health():
    return {"status": "ok"}

app.launch()
```

Your custom routes take priority over Gradio's default routes. For example, your `GET /` replaces Gradio's default UI page.

You can also use all standard FastAPI features — `app.add_middleware()`, `app.include_router()`, dependency injection, exception handlers, and so on.

## MCP Tools

To expose your API endpoints as MCP tools, add the `@app.mcp.tool()` decorator and pass `mcp_server=True` to `launch()`:

```python
import gradio as gr

app = gr.Server()

@app.mcp.tool(name="hello")
@app.api(name="hello")
def hello(name: str) -> str:
    """Greet someone by name."""
    return f"Hello, {name}!"

app.launch(mcp_server=True)
```

The `@app.mcp.tool()` and `@app.api()` decorators are independent — you can have API-only endpoints or MCP-only tools. Stack both when you want a function available through both.

## A Complete Example with the JavaScript Client

This example combines everything: custom HTML served at `/`, Gradio API endpoints with concurrency limits, MCP tools, and a custom REST endpoint, wired up with [the Gradio JavaScript client](/guides/getting-started-with-the-js-client).

$code_server_app

Run it with:

```bash
python run.py
```

Then open `http://localhost:7860` in your browser. The custom HTML page uses the `@gradio/client` JavaScript library to call the Gradio API endpoints. Meanwhile, the same endpoints are available as MCP tools and through the REST API at `/gradio_api/call/add` and `/gradio_api/call/multiply`.

Note: if your `gr.Server` app uses ZeroGPU, you _must_ call Gradio API endpoints through `@gradio/client` from the browser. The JavaScript client forwards the Hugging Face iframe auth headers needed for ZeroGPU quota handling.

## Concurrency and Streaming

`app.api()` supports all of the same concurrency and streaming options as `gr.api()`:

```python
@app.api(name="generate", concurrency_limit=2, stream_every=0.5)
async def generate(prompt: str):
    for token in model.generate(prompt):
        yield token
```

Generator functions automatically stream results via SSE, just like in a regular Gradio app. The `concurrency_limit` parameter controls how many concurrent calls to this endpoint are allowed. By default, this is set to 1, since many ML workloads that run on GPU can only support a single user at a time. However, you can increase this, or set to `None` to use FastAPI defaults, if you are e.g. calling an external API.

For the full API reference, see the [`Server` documentation](/docs/gradio/server).
