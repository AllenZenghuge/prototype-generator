# Vision Analyzer — Pixel-Level Screenshot Analysis

Sends screenshots to a configurable vision model for structured UI extraction.

## Model Selection

Read `prototype/prototype.config.json` → `vision_model`. Determine provider:

| Provider | API Call Method |
|----------|----------------|
| qwen3-vl-plus | OpenAI-compatible API via `Bash` tool with curl |
| claude-vision | Read the image file and pass to Claude's vision capability directly |
| gpt-4o | OpenAI API via `Bash` tool with curl |

## Qwen3-VL-Plus API Call (Default)

When provider is `qwen3-vl-plus`:

```bash
# For each screenshot in prototype/shots/:
curl -s "https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3-vl-plus",
    "messages": [
      {
        "role": "user",
        "content": [
          {"type": "image_url", "image_url": {"url": "data:image/png;base64,{BASE64_IMAGE}"}},
          {"type": "text", "text": "{ANALYSIS_PROMPT}"}
        ]
      }
    ],
    "max_tokens": 4096
  }'
```

### ANALYSIS_PROMPT (sent to vision model)

```
Analyze this UI screenshot and output a structured JSON description. Be precise with pixel values.

Output JSON schema:
{
  "layout": {
    "type": "left-right | top-bottom | single-column | left-center-right",
    "header": { "height": "px", "background_color": "hex", "elements": ["..."] },
    "sidebar": { "width": "px", "background_color": "hex", "position": "left|right|none" },
    "content": { "background_color": "hex" },
    "footer": { "height": "px", "visible": true|false }
  },
  "components": [
    {
      "type": "table | form | button | input | select | tag | tab | card | modal | drawer | stat-card | breadcrumb | pagination | chart | tree | timeline",
      "label": "human-readable name",
      "position": { "x": "px", "y": "px", "width": "px", "height": "px" },
      "style": {
        "background_color": "hex",
        "text_color": "hex",
        "border_color": "hex",
        "font_size": "px",
        "font_weight": "normal|medium|bold",
        "border_radius": "px",
        "padding": "px",
        "margin": "px"
      },
      "children": ["nested component descriptions"],
      "state": "default|hover|focus|active|disabled|error"
    }
  ],
  "colors": {
    "primary": "hex",
    "background": "hex",
    "surface": "hex",
    "text_primary": "hex",
    "text_secondary": "hex",
    "border": "hex",
    "success": "hex",
    "warning": "hex",
    "error": "hex"
  },
  "typography": {
    "heading_size": "px",
    "body_size": "px",
    "caption_size": "px",
    "font_family": "system|serif|mono"
  },
  "spacing": {
    "component_gap": "px",
    "section_padding": "px",
    "grid_columns": number
  },
  "data_examples": {
    "table_rows": ["example row 1 content", "example row 2 content"],
    "labels": ["visible text labels found in screenshot"],
    "numbers": ["number values with their format (currency/percentage/count)"]
  }
}

Important:
- ALL color values must be in hex format (#RRGGBB or #RRGGBBAA)
- ALL measurements in px
- If a component's state is not default, note it
- Include visible text content as data_examples for realistic mock data
- Note any icons or images with descriptions
```

### Parsing the Response

Parse the JSON response. For each component:
1. Map raw pixel values to the nearest design system token
2. Flag values that don't align to 4px grid
3. Note components that match spec-defined patterns (e.g., "this 32px blue button matches primary button spec")

### Handling Multiple Screenshots

If `prototype/shots/` contains multiple screenshots:
- Process each independently
- Cross-reference: identify shared components (same sidebar, same header)
- Deduplicate: shared components only need to be generated once
- Generate a unified page list from all screenshots

## Error Handling

| Scenario | Action |
|----------|--------|
| API key missing | Prompt: "Vision model API key not configured. Run /prototype:config vision to set it." |
| API returns error | Show error message. Offer fallback: "Vision analysis failed. Continue with spec-only generation? (y/n)" |
| JSON parse failure | Show raw response excerpt. Ask: "Could not parse vision model output. Retry? (y/n)" |
| Image too large | Compress to < 5MB before base64 encoding |
| Zero components detected | Ask: "No UI components detected in screenshot. Is this a valid UI screenshot? (y/n)" |
