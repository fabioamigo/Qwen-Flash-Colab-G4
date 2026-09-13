# Pi and OpenCode clients

These examples connect to the notebook's OpenAI-compatible **Chat Completions** API. The served model ID is **`pennyroyal`**, not the Hugging Face repository name. Run the clients on the computer containing the code you want them to work on.

## Streaming prerequisite

**Both clients need an SSE-capable connection. The notebook's default Cloudflare Quick Tunnel does not officially support SSE.** A successful `/models` request or the notebook's non-streaming generation test is insufficient to establish compatibility with these agents.

Use an SSE-capable route to the running server, such as a configured named Cloudflare Tunnel forwarding to `http://127.0.0.1:8001` on the Colab VM, or a private port-forward you have already established. Named tunnels require separate Cloudflare setup; these configuration examples do not provision one. Keep the server's API-key authentication enabled. If running a client on the same VM, use `http://127.0.0.1:8001/v1` directly. On your laptop, localhost means your laptop, unless you have configured port forwarding.

Do not add `stream: false` to these examples: their adapters expect streaming responses. Even if a Quick Tunnel happens to pass a short stream, this remains unsupported by Cloudflare. See [Cloudflare's Quick Tunnel limitations](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/trycloudflare/).

## Credentials and connection check

In Bash on Linux, macOS, or WSL:

```bash
export COLAB_BASE_URL='https://YOUR-SSE-ENDPOINT.example.com/v1'
read -rsp 'Temporary API key printed by the notebook: ' COLAB_API_KEY
echo
export COLAB_API_KEY

curl --fail-with-body --max-time 30 "$COLAB_BASE_URL/models" \
  -H "Authorization: Bearer $COLAB_API_KEY"

curl --fail-with-body --no-buffer --max-time 120 \
  "$COLAB_BASE_URL/chat/completions" \
  -H "Authorization: Bearer $COLAB_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"model":"pennyroyal","messages":[{"role":"user","content":"Reply with READY."}],"max_tokens":32,"stream":true,"chat_template_kwargs":{"enable_thinking":false}}'
```

The second request should return SSE `data:` events and terminate with `data: [DONE]`. This is a basic transport check, not a complete tool-calling or long-generation test. A 401 usually means the API key changed; a 404 can indicate a missing or duplicated `/v1`; HTML, timeouts, or broken streams indicate a routing or transport problem.

Use the **temporary server API key**, never your Hugging Face token. Repeat the exports after the notebook rotates credentials or the endpoint changes. Do not commit real keys or executed notebook outputs.

## Pi (pi.dev)

Use a current Pi version supporting the documented `chat-template` compatibility settings and `$VARIABLE` credential syntax. See [installation](https://pi.dev/) and [custom model documentation](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/models.md).

Merge the `colab-g4` provider from [pi/models.json](pi/models.json) into `~/.pi/agent/models.json`. If you have no file yet, create the parent directory and copy the example there. Preserve your existing providers.

Replace `baseUrl` with your actual SSE-capable base URL, including `/v1`. Pi's documented credential interpolation applies to `apiKey` and headers; this example uses a literal URL instead of assuming environment expansion in `baseUrl`. Leave `apiKey` as `"$COLAB_API_KEY"`, and export that variable in the shell that starts Pi.

```bash
pi --provider colab-g4 --model pennyroyal --thinking medium
```

You can also select the provider/model with `/model`. Pi uses `system` messages and `max_tokens` for this server. The compatibility mapping sends thinking controls inside `chat_template_kwargs`, preserving thinking and mapping Pi's selected effort. No top-level OpenAI `reasoning_effort` is required. The selectable levels are `off`, `medium`, and `xhigh`; `minimal`, `low`, `high`, and `max` are disabled through `thinkingLevelMap`. Use `--thinking off`, `--thinking medium`, or `--thinking xhigh`. Older Pi versions may require upgrading rather than accepting these newer fields silently.

## OpenCode

Install [OpenCode](https://opencode.ai/docs/), then use [opencode/opencode.json](opencode/opencode.json). From a clone of this repository, you can select the example without replacing your global configuration:

```bash
export OPENCODE_CONFIG="$PWD/examples/opencode/opencode.json"
# Change into the project you want OpenCode to work on, if necessary.
opencode --model colab-g4/pennyroyal
```

Alternatively, merge the example into your project's `opencode.json` or `~/.config/opencode/opencode.json`. OpenCode merges configuration sources, so existing project settings may override values. The example reads both `COLAB_BASE_URL` and `COLAB_API_KEY` from the environment. It uses `@ai-sdk/openai-compatible` for `/chat/completions` and selects the same Colab model for both normal and small-model tasks.

The model declares `reasoning_content` as its interleaved reasoning field. The example explicitly sends `chat_template_kwargs` and defaults to **medium** effort. Its `off`, `medium`, and `xhigh` variants override those kwargs; unwanted generic levels are disabled. Select a variant using OpenCode's `variant_cycle` keybinding. `off` sends `enable_thinking: false` and `reasoning_effort: "none"`; the other two send `enable_thinking: true` and the selected effort. All three retain `preserve_thinking: true`. The 600-second client timeout does not override tunnel or server timeouts.

## Explicit sampling and thinking controls

Both examples set the same sampling values for every thinking level:

| API parameter | Value |
| --- | ---: |
| `temperature` | 1.0 |
| `top_p` | 0.95 |
| `top_k` | 20 |
| `presence_penalty` | 0.0 |
| `repetition_penalty` | 1.0 |

Pi uses model-level `samplingParams`, which override Pi-generated request fields. OpenCode uses model-level `options` with the raw API field names. The inspected OpenAI-compatible adapter spreads these custom options into the request body after standardized sampling fields. Do not nest them under `samplingParams` or `extra_body` in OpenCode. Existing agent, plugin, or project overrides can still change the final request.

| Selected level | `enable_thinking` | Template effort |
| --- | --- | --- |
| `off` | false | `none` |
| `medium` | true | `medium` |
| `xhigh` | true | `xhigh` |

Both clients use `preserve_thinking: true` and send the controls inside `chat_template_kwargs`. The server/template determines the effect of these controls; `xhigh` is not a fixed thinking-token budget.

OpenCode also sets `options.max_tokens: 131072` explicitly. Merely advertising `limit.output: 131072` does not eliminate the inspected OpenCode default output cap of 32,000 tokens. The custom body option overrides the adapter's standardized `max_tokens` value for this provider. This sets a request ceiling, not a requirement to generate that many tokens.

## Context, output, and validation

Both examples advertise the notebook's **524,288-token total context** and use a **131,072-token output ceiling** as a client configuration choice, not a claim about the model's maximum output capability. Prompt, tool definitions, conversation, and generated tokens must fit the total context together. If you change the server context, update both client configurations. Pi's zero cost entries represent no per-token API billing; they do not mean the Colab runtime is free.

Text and image inputs are declared to match the notebook's multimodal setup. Actual vision, multi-turn reasoning replay, and tool execution still require end-to-end checks on the running server. These examples have been checked against current upstream configuration documentation and parsed as JSON; they have not been run against a live Colab endpoint. The notebook itself is unchanged by this addition.

## Sources and credits

- [Pi](https://pi.dev/) and [earendil-works/pi custom models](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/models.md): provider configuration and thinking compatibility.
- [OpenCode providers](https://opencode.ai/docs/providers/#custom-provider), [configuration](https://opencode.ai/docs/config/), and [configuration schema](https://opencode.ai/config.json): custom OpenAI-compatible providers, environment substitution, and model metadata.
- [OpenCode request transforms](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/provider/transform.ts) and [AI SDK OpenAI-compatible request construction](https://github.com/vercel/ai/blob/main/packages/openai-compatible/src/chat/openai-compatible-chat-language-model.ts): provider-option forwarding and output cap behavior.
- [Cloudflare Quick Tunnels](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/trycloudflare/): SSE transport limitation.
- [Main README](../README.md#credits-and-inspiration): Pennyroyal, SGLang, RadixArk, FR-Spec, and the notebook's other upstream credits.
