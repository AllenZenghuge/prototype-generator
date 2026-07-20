# Config Management

## Configuration File

`prototype/prototype.config.json` stores all skill configuration. If the file does not exist, run the first-time wizard.

### Default config values:

```json
{
  "vision_model": {
    "provider": "qwen3.7-max",
    "api_key": "",
    "base_url": "https://ws-udt9j53khqthrs7z.cn-beijing.maas.aliyuncs.com/compatible-mode/v1",
    "model_name": "qwen3.7-max-2026-06-08",
    "timeout_ms": 90000,
    "retry_max": 3,
    "retry_delay_ms": 5000
  },
  "default_spec": "./规范.md",
  "output_dir": "./output",
  "resolution": ["1366x768", "1440x900", "1920x1080"],
  "verification": {
    "enabled": true,
    "engine": "playwright"
  }
}
```

## First-Time Wizard

When `prototype/prototype.config.json` does not exist:

1. Create the `prototype/` directory with subdirectories: `docs/`, `shots/`, `output/css/`, `output/js/`, `output/pages/`
2. Ask the user to configure the vision model:

```
"First time setup! Which vision model provider for screenshot analysis?
1. Qwen3.7-Max (default, needs API key)
2. Claude Vision (built-in, no API key needed)
3. GPT-4o (needs API key)

Enter number:"
```

3. If provider needs API key, prompt: `"Enter your API key for {provider}:"`
4. Write `prototype/prototype.config.json` with the selected values
5. Confirm: `"Configuration saved to prototype/prototype.config.json. Place your UI spec as prototype/规范.md to get started."`

## /prototype:config Operations

### View current config
When called with no arguments, read and display `prototype/prototype.config.json`:
```
"Current configuration:
  Vision model: {provider} ({model_name})
  API timeout: {timeout_ms}ms, retry: {retry_max}× (delay {retry_delay_ms}ms)
  Default spec: {path}
  Output dir: {path}
  Resolutions: {list}
  Verification: {'enabled' / 'disabled'} (engine: {engine})"
```

### Change vision model: `/prototype:config vision`
1. Show current vision model config
2. Ask: `"Change to which provider? (qwen3.7-max / claude-vision / gpt-4o)"`
3. If API key needed, prompt for it
4. Update `prototype/prototype.config.json`

### Change API retry settings: `/prototype:config retry`
1. Show current timeout/retry values
2. Ask: `"Change which? (timeout / retry_max / retry_delay)"`
3. Prompt for new value
4. Update `prototype/prototype.config.json`
5. Confirm: `"API retry settings updated: timeout={x}ms, retry={n}×, delay={y}ms"`

### Change spec path: `/prototype:config spec`
1. Prompt for new spec path
2. Update `default_spec` in config

### Toggle verification: `/prototype:config verify on|off`
1. Set `verification.enabled` to `true` or `false`
2. Confirm: `"Verification {'enabled' / 'disabled'}."`

### Change verification engine: `/prototype:config engine`
1. Show current engine
2. Ask: `"Switch to which engine? (playwright / agentcloak)"`
3. Update `verification.engine` in config
