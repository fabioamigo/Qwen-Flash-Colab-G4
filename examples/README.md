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

You can also select the provider/model with `/model`. Pi uses `system` messages and `max_tokens` for this server. The compatibility mapping sends thinking controls inside `chat_template_kwargs`, preserving thinking and mapping Pi's selected effort. No top-level OpenAI `reasoning_effort` is required. Start with `medium`; this example does not declare extra `xhigh` or `max` levels. Older Pi versions may require upgrading rather than accepting these newer fields silently.

## OpenCode

Install [OpenCode](https://opencode.ai/docs/), then use [opencode/opencode.json](opencode/opencode.json). From a clone of this repository, you can select the example without replacing your global configuration:

```bash
export OPENCODE_CONFIG="$PWD/examples/opencode/opencode.json"
# Change into the project you want OpenCode to work on, if necessary.
opencode --model colab-g4/pennyroyal
```

Alternatively, merge the example into your project's `opencode.json` or `~/.config/opencode/opencode.json`. OpenCode merges configuration sources, so existing project settings may override values. The example reads both `COLAB_BASE_URL` and `COLAB_API_KEY` from the environment. It uses `@ai-sdk/openai-compatible` for `/chat/completions` and selects the same Colab model for both normal and small-model tasks.

The model declares `reasoning_content` as its interleaved reasoning field. This example relies on the notebook's default **thinking enabled, preserve thinking enabled, medium effort**. It does not promise that OpenCode's generic reasoning variants map to the custom Qwen chat template. The 600-second client timeout does not override tunnel or server timeouts.

## Context, output, and validation

Both examples advertise the notebook's **524,288-token total context** and use a **32,768-token output ceiling** as a client configuration choice, not a claim about the model's maximum output capability. Prompt, tool definitions, conversation, and generated tokens must fit the total context together. If you change the server context, update both client configurations. Pi's zero cost entries represent no per-token API billing; they do not mean the Colab runtime is free.

Text and image inputs are declared to match the notebook's multimodal setup. Actual vision, multi-turn reasoning replay, and tool execution still require end-to-end checks on the running server. These examples have been checked against current upstream configuration documentation and parsed as JSON; they have not been run against a live Colab endpoint. The notebook itself is unchanged by this addition.

## Sources and credits

- [Pi](https://pi.dev/) and [earendil-works/pi custom models](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/models.md): provider configuration and thinking compatibility.
- [OpenCode providers](https://opencode.ai/docs/providers/#custom-provider), [configuration](https://opencode.ai/docs/config/), and [configuration schema](https://opencode.ai/config.json): custom OpenAI-compatible providers, environment substitution, and model metadata.
- [Cloudflare Quick Tunnels](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/trycloudflare/): SSE transport limitation.
- [Main README](../README.md#credits-and-inspiration): Pennyroyal, SGLang, RadixArk, FR-Spec, and the notebook's other upstream credits.
