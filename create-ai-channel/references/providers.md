# AI Provider Reference

## Anthropic Claude

```python
from roomkit import AnthropicAIProvider, AnthropicConfig

provider = AnthropicAIProvider(
    AnthropicConfig(
        api_key="sk-ant-...",           # Required: ANTHROPIC_API_KEY
        model="claude-sonnet-4-20250514",  # Default model
    )
)
```

**Models:** `claude-opus-4-20250514`, `claude-sonnet-4-20250514`, `claude-haiku-4-20250514`

**Environment variable:** `ANTHROPIC_API_KEY`

## OpenAI GPT

```python
from roomkit import OpenAIAIProvider, OpenAIConfig

provider = OpenAIAIProvider(
    OpenAIConfig(
        api_key="sk-...",              # Required: OPENAI_API_KEY
        model="gpt-4o",               # Default model
    )
)
```

**Models:** `gpt-4o`, `gpt-4o-mini`, `gpt-4-turbo`, `o1`, `o1-mini`

**Environment variable:** `OPENAI_API_KEY`

## Google Gemini

```python
from roomkit import GeminiAIProvider, GeminiConfig

provider = GeminiAIProvider(
    GeminiConfig(
        api_key="...",                 # Required: GEMINI_API_KEY
        model="gemini-2.0-flash",      # Default model
    )
)
```

**Models:** `gemini-2.0-flash`, `gemini-2.0-flash-lite`, `gemini-1.5-pro`, `gemini-1.5-flash`

**Environment variable:** `GEMINI_API_KEY`

## Mistral

```python
from roomkit import MistralAIProvider, MistralConfig

provider = MistralAIProvider(
    MistralConfig(
        api_key="...",                 # Required: MISTRAL_API_KEY
        model="mistral-large-latest",  # Default model
    )
)
```

**Models:** `mistral-large-latest`, `mistral-medium-latest`, `mistral-small-latest`, `open-mistral-nemo`

**Environment variable:** `MISTRAL_API_KEY`

## Azure OpenAI

```python
from roomkit import AzureAIProvider, AzureAIConfig

provider = AzureAIProvider(
    AzureAIConfig(
        api_key="...",                                    # Required
        endpoint="https://my-resource.openai.azure.com",  # Required
        deployment="gpt-4o",                              # Required: deployment name
        api_version="2024-02-15-preview",                 # API version
    )
)
```

**Environment variables:** `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_ENDPOINT`

## vLLM (Local)

```python
from roomkit import create_vllm_provider, VLLMConfig

provider = create_vllm_provider(
    VLLMConfig(
        base_url="http://localhost:8000/v1",  # vLLM server URL
        model="meta-llama/Llama-3-8B",        # Model name
    )
)
```

No API key required for local inference.

## Mock (Testing)

```python
from roomkit import MockAIProvider

# Fixed responses
provider = MockAIProvider(responses=["Hello!", "How can I help?"])

# Cycles through responses in order
```

## AIChannel Options

All providers are used through `AIChannel`:

```python
from roomkit.channels.ai import AIChannel
from roomkit import RetryPolicy

ai = AIChannel(
    "ai-main",
    provider=provider,
    system_prompt="You are a helpful assistant.",
    temperature=0.7,                              # 0.0-1.0
    max_tokens=1024,                              # Max response tokens
    max_context_events=50,                        # Conversation history window
    thinking_budget=4096,                         # Extended thinking tokens
    tool_handler=my_handler,                      # Tool execution callback
    max_tool_rounds=200,                          # Tool loop limit
    tool_loop_timeout_seconds=300.0,              # Tool loop timeout
    retry_policy=RetryPolicy(max_retries=3),      # Retry on failure
    fallback_provider=backup_provider,            # Fallback provider
)
```
