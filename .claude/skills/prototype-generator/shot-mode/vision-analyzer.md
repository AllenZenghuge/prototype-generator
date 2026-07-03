# Vision Analyzer — Pixel-Level Screenshot Analysis

Sends screenshots to a configurable vision model for structured UI extraction.

## Model Selection

Read `prototype/prototype.config.json` → `vision_model`. Determine provider:

| Provider | API Call Method |
|----------|----------------|
| qwen3-vl-plus | OpenAI-compatible API via `Bash` tool with curl |
| claude-vision | Read the image file and pass to Claude's vision capability directly |
| gpt-4o | OpenAI API via `Bash` tool with curl |

---

## Phase 0 — Context Gathering (CRITICAL — do before any API call)

**Do NOT send screenshots to the vision model immediately.** First, extract all available context:

### 0.1 Read screenshot filenames and user messages

List all screenshots in `prototype/shots/` and note their filenames. Chinese filenames often carry semantic meaning (e.g., "内容区列表字段数据-前部分.png" = "content area list field data - front part").

### 0.2 Parse user context from the conversation

Extract from the user's message:
- Which part of the UI each screenshot covers (e.g., "前 10 列在截图 A，剩余列在截图 B")
- What to focus on (e.g., "只关注列头，忽略分页区域")
- What was already identified from other screenshots (e.g., "前 10 列已从另一张图识别完毕")
- Any known column/field names mentioned by the user

### 0.3 Confirm with user before sending

Present the extracted context to the user:

```
"Based on your screenshots and description, I understand:
 - Screenshot A ({filename}): {what it shows — e.g. left portion of wide table, columns 1-10}
 - Screenshot B ({filename}): {what it shows — e.g. right portion, columns 11+, plus pagination}
 - Special focus: {any user-specified focus areas}
 - Known context: {any info user already provided}

 I'll send this context to the vision model. Continue?"
```

**Only proceed after user confirmation.** This prevents context loss between the user's description and the vision model's prompt.

### 0.4 Construct the context-aware prompt

For each screenshot, build a prompt that includes:
1. What this screenshot shows (from filename + user description)
2. What to focus on / what to ignore
3. What was already identified from other screenshots (don't re-identify)
4. Any specific questions to answer

**Example for a split wide table:**
```
"Screenshot A shows the LEFT portion (columns 1-10) of a wide data table.
Screenshot B shows the RIGHT portion (columns 11+) of the same table.
Columns 1-10 were already identified from Screenshot A as: 操作, 状态, ...
For Screenshot B: focus ONLY on the column headers. List every column header
beyond column 10. Ignore the pagination area at the bottom."
```

---

## Phase 1 — Pixel Analysis (with context)

### 1.1 Qwen3-VL-Plus API Call

When provider is `qwen3-vl-plus`:

```bash
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
          {"type": "text", "text": "{CONTEXT_AWARE_PROMPT}"}
        ]
      }
    ],
    "max_tokens": 4096
  }'
```

### 1.2 Context-Aware Prompt Template

Replace `{CONTEXT_AWARE_PROMPT}` with a prompt built from Phase 0 context:

```
{SCREENSHOT_CONTEXT}

{SPECIFIC_FOCUS_INSTRUCTIONS}

{Known columns/areas already identified from other screenshots}

Analyze this UI screenshot and output a structured JSON description:

{
  "layout": {
    "type": "left-right | top-bottom | single-column | left-center-right",
    "header": { "height": "px", "background_color": "hex", "elements": [...] },
    "sidebar": { "width": "px", "background_color": "hex", "position": "left|right|none" },
    "content": { "background_color": "hex" }
  },
  "components": [
    {
      "type": "table | form | button-icon | button-text | button-text-icon | input | select | tag | tab | card | modal | drawer | stat-card | breadcrumb | pagination | chart | tree | timeline | checkbox | icon | avatar | badge | dropdown",
      "label": "human-readable name",
      "position": { "x": "px", "y": "px", "width": "px", "height": "px" },
      "style": {
        "background_color": "hex",
        "text_color": "hex",
        "border_color": "hex",
        "font_size": "px",
        "font_weight": "normal|medium|bold",
        "border_radius": "px",
        "padding": "px"
      },
      "state": "default|hover|focus|active|disabled|error"
    }
  ],
  "colors": { "primary": "hex", "background": "hex", "surface": "hex", ... },
  "typography": { "heading_size": "px", "body_size": "px", "caption_size": "px" },
  "spacing": { "component_gap": "px", "section_padding": "px" },
  "data_examples": { "table_rows": [...], "labels": [...], "numbers": [...] }
}

CRITICAL:
- ALL color values in hex (#RRGGBB)
- ALL measurements in px
- If asked to focus on column headers, list EVERY column — do not skip any
- Include visible text content as data_examples for realistic mock data
```

### 1.3 Parse the Response

Parse the JSON response. For each component:
1. Map raw pixel values to the nearest design system token
2. Flag values that don't align to 4px grid
3. Note components that match spec-defined patterns

---

## Phase 2 — Save Results

After receiving the vision model response, save the raw result to:

```
prototype/shots/vision_result_{timestamp}.json
```

This allows:
- Reuse without re-calling the API
- Incremental refinement (add columns, fix details)
- Audit trail of what the model saw

---

## Handling Multiple Screenshots

**Critical rule for multi-screenshot scenarios:**

1. **Never batch unrelated screenshots.** Each gets its own context-aware prompt.
2. **For split views** (e.g., wide table split across 2 screenshots):
   - Process sequentially: identify left portion first, then feed that context into the right portion's prompt
   - Tell the model explicitly: "Columns 1-10 were already identified as: [list]. Now identify columns 11+"
3. **Cross-reference**: identify shared components across screenshots
4. **Deduplicate**: shared components only need to be generated once

## Error Handling

| Scenario | Action |
|----------|--------|
| API key missing | Prompt: "Vision model API key not configured. Run /prototype-generator config to set it." |
| API returns error | Show error message. Offer fallback: "Vision analysis failed. Continue with spec-only generation?" |
| JSON parse failure | Show raw response excerpt. Ask: "Could not parse vision model output. Retry?" |
| Image too large | Compress to < 5MB before base64 encoding |
| Zero components detected | Ask: "No UI components detected. Is this a valid UI screenshot?" |
| Model missed columns/areas | Ask user to describe what was missed. Re-analyze with focused context in the prompt. |
