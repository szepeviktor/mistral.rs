# HTTP server

Mistral.rs provides a lightweight OpenAI API compatible HTTP server based on [axum](https://github.com/tokio-rs/axum). The request and response formats are supersets of the OpenAI API.

The API consists of the following endpoints. They can be viewed in your browser interactively by going to `http://localhost:<port>/docs`.

> ℹ️  Besides the HTTP endpoints described below `mistralrs-server` can also expose the same functionality via the **MCP protocol**.  
> Enable it with `--mcp-port <port>` and see [MCP_SERVER.md](MCP_SERVER.md) for details.

## Additional object keys

To support additional features, we have extended the completion and chat completion request objects. Both have the same keys added:

- `top_k`: `int` | `null`. If non null, it is only relevant if positive.
- `grammar`: `{"type" : "regex" | "lark" | "json_schema" | "llguidance", "value": string}` or `null`. Grammar to use. This is mutually exclusive to the OpenAI-compatible `response_format`.
- `min_p`: `float` | `null`. If non null, it is only relevant if 1 >= min_p >= 0.
- `enable_thinking`: `bool`, default to `false`. Enable thinking for models that support it.

## Model Parameter Validation

Mistral.rs validates that the `model` parameter in API requests matches the model that was actually loaded by the server. This ensures requests are processed by the correct model and prevents confusion.

**Behavior:**
- If the `model` parameter matches the loaded model name, the request proceeds normally
- If the `model` parameter doesn't match, the request fails with an error message indicating the mismatch
- The special model name `"default"` can be used to bypass this validation entirely

**Examples:**
- ✅ Request with `"model": "meta-llama/Llama-3.2-3B-Instruct"` when `meta-llama/Llama-3.2-3B-Instruct` is loaded → **succeeds**
- ❌ Request with `"model": "gpt-4"` when `mistral-7b-instruct` is loaded → **fails**
- ✅ Request with `"model": "default"` regardless of loaded model → **always succeeds**

**Usage:** Use `"default"` in the model field when you need to satisfy API clients that require a model parameter but don't need to specify a particular model. This is demonstrated in all the examples below.

## `POST`: `/v1/chat/completions`
Process an OpenAI compatible request, returning an OpenAI compatible response when finished. Please find the official OpenAI API documentation [here](https://platform.openai.com/docs/api-reference/chat). To control the interval keep-alive messages are sent, set the `KEEP_ALIVE_INTERVAL` environment variable to the desired time in ms.

To send a request with the Python `openai` library:

```python
import openai

client = openai.OpenAI(
    base_url="http://localhost:8080/v1", # "http://<Your api-server IP>:port"
    api_key = "EMPTY"
)

completion = client.chat.completions.create(
model="default",
messages=[
    {"role": "system", "content": "You are Mistral.rs, an AI assistant."},
    {"role": "user", "content": "Write a story about Rust error handling."}
]
)

print(completion.choices[0].message)
```

Or with `curl`:
```bash
curl http://localhost:8080/v1/chat/completions \
-H "Content-Type: application/json" \
-H "Authorization: Bearer EMPTY" \
-d '{
"model": "default",
"messages": [
{
    "role": "system",
    "content": "You are Mistral.rs, an AI assistant."
},
{
    "role": "user",
    "content": "Write a story about Rust error handling."
}
]
}'
```

A streaming request can also be created by setting `"stream": true` in the request JSON. Please see [this](https://cookbook.openai.com/examples/how_to_stream_completions) guide.

## `GET`: `/v1/models`
Returns the running models. 

Example with `curl`:
```bash
curl http://localhost:<port>/v1/models
```

## `GET`: `/` or `/health`
Returns the server health.

Example with `curl`:
```bash
curl http://localhost:<port>/health
```

## `GET`: `/docs`
Returns OpenAPI API docs via SwaggerUI.

Example with `curl`:
```bash
curl http://localhost:<port>/docs
```

## `POST`: `/v1/completions`
Process an OpenAI compatible completions request, returning an OpenAI compatible response when finished. Please find the official OpenAI API documentation [here](https://platform.openai.com/docs/api-reference/completions). 

To send a request with the Python `openai` library:

```python
import openai

client = openai.OpenAI(
    base_url="http://localhost:8080/v1", # "http://<Your api-server IP>:port"
    api_key = "EMPTY"
)

completion = client.completions.create(
    model="default",
    prompt="What is Rust?",
    max_tokens=256,
    frequency_penalty=1.0,
    top_p=0.1,
    temperature=0,
)

print(completion.choices[0].message)
```

Or with `curl`:
```bash
curl http://localhost:8080/v1/completions \
-H "Content-Type: application/json" \
-H "Authorization: Bearer EMPTY" \
-d '{
"model": "default",
"prompt": "What is Rust?"
}'
```


## `POST`: `/v1/responses`
Create a response using the OpenAI-compatible Responses API. Please find the official OpenAI API documentation [here](https://platform.openai.com/docs/api-reference/responses). 

To send a request with the Python `openai` library:

```python
import openai

client = openai.OpenAI(
    base_url="http://localhost:8080/v1",
    api_key = "EMPTY"
)

# First turn
resp1 = client.responses.create(
    model="default",
    input="Apples are delicious!"
)
print(resp1.output_text)

# Follow-up - no need to resend the first message
resp2 = client.responses.create(
    model="default",
    previous_response_id=resp1.id,
    input="Can you eat them?"
)
print(resp2.output_text)
```

Or with `curl`:
```bash
curl http://localhost:8080/v1/responses \
-H "Content-Type: application/json" \
-H "Authorization: Bearer EMPTY" \
-d '{
"model": "default",
"input": "Tell me about Rust programming"
}'

# Follow-up using previous_response_id
curl http://localhost:8080/v1/responses \
-H "Content-Type: application/json" \
-H "Authorization: Bearer EMPTY" \
-d '{
"model": "default",
"previous_response_id": "resp_12345-uuid-here",
"input": "What makes it memory safe?"
}'
```

The API also supports multimodal inputs (images, audio) and streaming responses by setting `"stream": true` in the request JSON.

## `GET`: `/v1/responses/{response_id}`
Retrieve a previously created response by its ID.

Example with `curl`:
```bash
curl http://localhost:8080/v1/responses/resp_12345-uuid-here \
-H "Authorization: Bearer EMPTY"
```

## `DELETE`: `/v1/responses/{response_id}`
Delete a stored response and its associated conversation history.

Example with `curl`:
```bash
curl -X DELETE http://localhost:8080/v1/responses/resp_12345-uuid-here \
-H "Authorization: Bearer EMPTY"
```

## `POST`: `/re_isq`
Reapply ISQ to the model if possible. Pass the names as a JSON object with the key `ggml_type` to a string (the quantization level).

Example with `curl`:
```bash
curl http://localhost:<port>/re_isq -H "Content-Type: application/json" -H "Authorization: Bearer EMPTY" -d '{"ggml_type":"4"}'
```
