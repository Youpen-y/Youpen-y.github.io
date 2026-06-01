---
title: "Shortcodes Reference"
date: 2025-01-01
draft: false
description: "Hugo shortcodes available in this site"
---

Hugo shortcodes extend Markdown with custom components.

- Use `{{</* */>}}` with percent delimiters for shortcodes that process inner content as Markdown.
- Use angle bracket delimiters for raw HTML shortcodes.

---

## Table

Define column widths for Markdown tables.

### Ratio Syntax (recommended)

Use colon-separated numbers. Percentages are calculated automatically:

```markdown
{{% table cols="2:3:1" %}}
| Name    | Description             | Status |
|---------|-------------------------|--------|
| Feature | A long description here | Active |
{{% /table %}}
```

Renders as a 3-column table with widths 33.33%, 50%, 16.67%.

### Percent Syntax (legacy)

Comma-separated percentage values:

```markdown
{{% table cols="20%, 60%, 20%" %}}
| Name    | Description             | Status |
|---------|-------------------------|--------|
| Feature | A long description here | Active |
{{% /table %}}
```

### Auto Layout

Omit `cols` to use browser auto-sizing based on content:

```markdown
{{% table %}}
| Key  | Value |
|------|-------|
| Name | Demo  |
{{% /table %}}
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `cols`    | No       | Column widths. Ratio (`"2:3:1"`) or percent (`"20%,60%,20%"`). |

---

## Embed

Embed external pages via iframe (desktop) or link card (mobile).

```markdown
{{< embed url="https://example.com" title="Example" height="400" >}}
```

| Parameter | Default          | Description |
|-----------|------------------|-------------|
| `url`     | (required)       | Page URL to embed. |
| `title`   | Same as `url`    | Display title. |
| `height`  | `"600"`          | Iframe height in pixels. |

---

## Mermaid

Render Mermaid diagrams with dark mode support.

```markdown
{{< mermaid >}}
graph LR
    A[Start] --> B{Decision}
    B -->|Yes| C[Action]
    B -->|No| D[End]
{{< /mermaid >}}
```

Mermaid JS is loaded once per page. Diagrams re-render on theme toggle.

---

## Tips

Colored callout boxes with four severity levels.

```markdown
{{< tips info "Did you know?" >}}
This is an informational tip.
{{< /tips >}}
```

```markdown
{{< tips warn "Caution" >}}
Be careful with this setting.
{{< /tips >}}
```

### Types

| Type      | Color  | Use case |
|-----------|--------|----------|
| `info`    | Blue   | Helpful notes |
| `warn`    | Yellow | Warnings |
| `error`   | Red    | Error messages |
| `success` | Green  | Success confirmations |

### Parameters

| Parameter | Position | Required | Description |
|-----------|----------|----------|-------------|
| type      | 1st      | Yes      | One of `info`, `warn`, `error`, `success` |
| title     | 2nd      | No       | Bold heading text |

---

## Figure

Enhanced image with optional caption, link, and attribution.

```markdown
{{< figure src="/images/photo.jpg" title="Sunset" caption="A beautiful sunset" >}}
```

### Parameters

| Parameter  | Default             | Description |
|------------|---------------------|-------------|
| `src`      | (required)          | Image source URL. |
| `alt`      |                     | Alt text. Falls back to `caption`. |
| `link`     | Same as `src`       | Click-through URL. |
| `target`   | `"_blank"`          | Link target. |
| `rel`      |                     | Link rel attribute. |
| `title`    |                     | Image title. |
| `caption`  |                     | Caption below image (Markdown supported). |
| `attr`     |                     | Attribution text. |
| `attrlink` |                     | Attribution link URL. |
| `width`    |                     | Image width. |
| `height`   |                     | Image height. |
| `class`    | `"image-container"` | CSS class for wrapper. |

---

## Plist

Property list rendered as a key-value table.

```markdown
{{< plist "Name:Alice" "Age:30" "City:Beijing" >}}
```

Renders as a two-column table with keys as header cells and values as data cells (Markdown supported).

---

## Raw

Output inner content as raw HTML, bypassing Markdown processing.

```html
{{< raw >}}
<div class="custom">Hello <strong>world</strong></div>
{{< /raw >}}
```

---

## Video

Embed an autoplay looping video.

```markdown
{{< video src="/videos/demo.mp4" >}}
```

The video element fills 100% width with controls, muted playback, looping, and autoplay enabled.
