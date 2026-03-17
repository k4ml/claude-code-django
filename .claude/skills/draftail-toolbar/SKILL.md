---
name: wagtail-draftail-custom-entity
description: >
  Add custom entity toolbar buttons to Wagtail's Draftail rich text editor.
  Use this skill whenever implementing a new inline entity feature in a Wagtail
  project — icon pickers, badge inserters, custom inline widgets, or any toolbar
  button that inserts a Draft.js entity. Covers all three layers: Python
  registration in wagtail_hooks.py, JavaScript plugin (source + decorator) in
  styler-i18n-merge.js, and CSS. Trigger this skill at the first mention of
  "Draftail", "rich text toolbar", "custom entity", "inline icon", or "wagtail
  rich text button".
---

# Wagtail Draftail Custom Entity Toolbar Button

A reference skill for implementing custom entity buttons in Wagtail's Draftail
rich text editor. Entities are inline objects (icons, badges, custom widgets)
inserted into rich text that round-trip correctly through Wagtail's HTML ↔
contentstate converter.

---

## Architecture Overview

Three mandatory layers, plus optional settings:

| Layer | File | Responsibility |
|---|---|---|
| Python | `wagtail_hooks.py` | Feature registration, HTML ↔ contentstate converters |
| JavaScript | `styler-i18n-merge.js` | Draftail plugin: source (picker UI) + decorator (editor render) |
| CSS | `styler.css` | Inline display + modal/grid styles |
| Settings | `settings.py` | Declare feature in editor feature list |

All custom styler features live in `src/wagtail_styler/`.

---

## Step 1 — Python (`wagtail_hooks.py`)

### 1a. Helper: build choices at request time

Scan a directory and return `[{name, label, url}]`. Called from
`insert_editor_js` to inject data into `window.*`.

```python
def get_my_icon_choices():
    icons_dir = Path(settings.BASE_DIR) / "static" / "images" / "my-icons"
    if not icons_dir.exists():
        return []
    allowed_ext = {".png", ".jpg", ".jpeg", ".svg", ".webp", ".gif"}
    choices = []
    for p in sorted(icons_dir.iterdir()):
        if p.is_file() and p.suffix.lower() in allowed_ext:
            choices.append({
                "name": p.stem.lower(),
                "label": p.stem.replace("-", " ").replace("_", " ").title(),
                "url": static(f"images/my-icons/{p.name}"),
            })
    return choices
```

### 1b. Element handler — DB HTML → contentstate

```python
class MyIconElementHandler(InlineEntityElementHandler):
    mutability = "MUTABLE"  # Fine even though JS creates IMMUTABLE entities

    def get_attribute_data(self, attrs):
        return {
            "icon": attrs.get("data-my-icon"),
            "url": attrs.get("data-my-icon-url"),
        }
```

### 1c. Decorator — contentstate → DB HTML

```python
def my_icon_to_db_decorator(props):
    icon_name = props.get("icon")
    icon_url = props.get("url")
    if not icon_name:
        return DOM.create_element("span", {}, props["children"])
    if not icon_url:
        icon_url = static(f"images/my-icons/{icon_name}.png")
    return DOM.create_element(
        "span",
        {
            "data-my-icon": icon_name,
            "data-my-icon-url": icon_url,
            "style": {
                "display": "inline-block",
                "width": "20px",
                "height": "20px",
                "vertical-align": "middle",
                "background-image": f'url("{icon_url}")',
                "background-repeat": "no-repeat",
                "background-position": "center",
                "background-size": "contain",
            },
        },
        props["children"],  # passes the \u200b placeholder through unchanged
    )
```

### 1d. Register the feature

Inside `register_rich_text_features`:

```python
features.register_editor_plugin(
    "draftail",
    "my_icon",
    draftail_features.EntityFeature(
        {"type": "MY_ICON", "label": "🔶", "description": _("Insert icon")},
        js=[STYLER_JS],            # ← REQUIRED (see gotchas)
        css={"all": [STYLER_CSS]}, # ← REQUIRED
    ),
)
features.register_converter_rule(
    "contentstate",
    "my_icon",
    {
        "from_database_format": {
            "span[data-my-icon]": MyIconElementHandler("MY_ICON")
        },
        "to_database_format": {
            "entity_decorators": {"MY_ICON": my_icon_to_db_decorator}
        },
    },
)
```

### 1e. Add to default features list

```python
STYLER_FEATURES = [
    "color_custom",
    "highlight_custom",
    "fontsize_custom",
    "payment_icon",
    "inline_icon",
    "my_icon",   # ← add here
]
```

These are appended to `features.default_features` in `register_rich_text_features`
so all editors pick them up automatically.

### 1f. Inject choice data into JavaScript

```python
@hooks.register("insert_editor_js")
def editor_js():
    return format_html(
        '<script>window.wagtailStylerMyIcons = {};</script>',
        mark_safe(json.dumps(get_my_icon_choices())),
    )
```

**Only inject `window.*` data here — never a `<script src="...">` for STYLER_JS.
Loading it again here breaks save behaviour** (see gotchas).

### 1g. Cache-busting for the JS/CSS assets

```python
def versioned_static(asset_path):
    asset_url = static(asset_path)
    asset_file = Path(__file__).resolve().parent / "static" / asset_path
    if not asset_file.exists():
        return asset_url
    return f"{asset_url}?v={int(asset_file.stat().st_mtime)}"

STYLER_JS  = versioned_static("wagtail_styler/styler-i18n-merge.js")
STYLER_CSS = versioned_static("wagtail_styler/styler.css")
```

Computed once at module load. Touching the file updates mtime and forces
browsers to re-fetch.

---

## Step 2 — JavaScript (`styler-i18n-merge.js`)

All plugins live in one shared file used by all styler features.

### 2a. Add i18n strings

```js
var L = {
  en: {
    title_my_icon: "Insert icon",
    no_my_icons:   "No icons available",
  },
  ja: {
    title_my_icon: "アイコンを挿入",
    no_my_icons:   "利用可能なアイコンがありません",
  },
};
```

### 2b. Fetch choices from window

```js
function getMyIcons() {
  var source = window.wagtailStylerMyIcons;
  if (!Array.isArray(source)) return [];
  return source.filter(function (icon) {
    return icon && icon.name && icon.url;
  });
}
```

### 2c. Add a branch to `mergeDataFor`

```js
} else if (kind === "MY_ICON") {
  data.icon  = value.name;
  data.url   = value.url;
}
```

### 2d. Add a fast-path in `applyMergedEntity`

Icon entities always create a *new* entity — they must never go through the
normal "merge with existing entity" path used by COLOR/HILITE/FSIZE.

```js
if (kind === "MY_ICON") {
  var iconData = mergeDataFor(kind, value, {});
  content = content.createEntity("MY_ICON", "IMMUTABLE", iconData);
  var key   = content.getLastCreatedEntityKey();
  var style = editorState.getCurrentInlineStyle();
  var newContent = DraftJS.Modifier.replaceText(
    content, selection,
    "\u200b",   // ← MUST be \u200b — see \u200b rule below
    style, key
  );
  onComplete(DraftJS.EditorState.push(editorState, newContent, "insert-characters"));
  return;  // ← early return skips normal merge path
}
```

### 2e. Decorator — renders entity inside the editor

```js
function myIconDeco(nodeProps) {
  var d = nodeProps.data || getData(nodeProps) || {};
  if (!d.url || !d.icon) return React.createElement("span", null, nodeProps.children);
  return React.createElement(
    "span", { className: "ws-my-icon-wrap" },
    React.createElement("img", { src: d.url, alt: d.icon, className: "ws-my-icon-image" }),
    React.createElement("span", { className: "ws-my-icon-placeholder" }, nodeProps.children)
  );
}
```

### 2f. Source component — picker popup

Use a **class component** so each click creates a fresh instance (function
components get reused and can stale-close).

```js
function MyIconSource(props) {
  React.Component.call(this, props);
  this._container = null;
  this._key = this._key.bind(this);
}
MyIconSource.prototype = Object.create(React.Component.prototype);
MyIconSource.prototype.constructor = MyIconSource;

MyIconSource.prototype._key = function (e) {
  if (!this._container) return;
  if (e.key === "Escape") {
    e.preventDefault();
    this._close();
    this.props.onComplete(this.props.editorState);
  }
};
MyIconSource.prototype._close = function () {
  if (this._container) {
    try { ReactDOM.unmountComponentAtNode(this._container); } catch (e) {}
    if (this._container.parentNode) this._container.parentNode.removeChild(this._container);
    this._container = null;
    document.removeEventListener("keydown", this._key);
  }
};
MyIconSource.prototype.componentDidMount = function () {
  this._container = document.createElement("div");
  this._container.className = "ws-modal-root";
  document.body.appendChild(this._container);
  document.addEventListener("keydown", this._key);

  var self = this;
  function onCancel() { self._close(); self.props.onComplete(self.props.editorState); }
  function onSelect(icon) {
    self._close();
    applyMergedEntity(self.props.editorState, "MY_ICON", icon, self.props.onComplete);
  }
  function PickerPanel() {
    var icons = getMyIcons();
    return React.createElement(
      "div", { className: "ws-modal-backdrop", onClick: onCancel },
      React.createElement(
        "div", { className: "ws-modal ws-payment-modal", onClick: stop },
        React.createElement(
          "div", { className: "ws-modal-form" },
          React.createElement("div", { className: "ws-modal-title" }, t("title_my_icon")),
          icons.length
            ? React.createElement(
                "div", { className: "ws-payment-grid" },
                icons.map(function (icon) {
                  return React.createElement(
                    "button",
                    {
                      key: icon.name, type: "button",
                      className: "ws-payment-option",
                      onClick: function () { onSelect(icon); },
                      title: icon.label || icon.name,
                    },
                    React.createElement("img", { src: icon.url, alt: icon.label || icon.name, className: "ws-payment-option-image" }),
                    React.createElement("span", { className: "ws-payment-option-label" }, icon.label || icon.name)
                  );
                })
              )
            : React.createElement("p", { className: "ws-payment-empty" }, t("no_my_icons")),
          React.createElement(
            "div", { className: "ws-actions" },
            React.createElement("button", { type: "button", className: "ws-btn", onClick: onCancel }, t("cancel"))
          )
        )
      )
    );
  }
  ReactDOM.render(React.createElement(PickerPanel), this._container);
};
MyIconSource.prototype.componentWillUnmount = function () { this._close(); };
MyIconSource.prototype.render = function () { return null; };
```

### 2g. Register the plugin

```js
registerPlugin(
  { type: "MY_ICON", source: MyIconSource, decorator: myIconDeco },
  "entityTypes"
);
```

Pass the **class itself** as `source` (not a `Source("MY_ICON", ...)` factory),
because icon pickers don't use the standard prompt-box UI pattern.

---

## Step 3 — CSS (`styler.css`)

```css
/* Inline display inside the editor */
.ws-my-icon-wrap        { display: inline-flex; align-items: center; vertical-align: middle; margin: 0 2px; }
.ws-my-icon-image       { width: 20px; height: 20px; object-fit: contain; display: inline-block; }
.ws-my-icon-placeholder { font-size: 0; line-height: 0; }
```

Modal/grid/button classes (`.ws-modal-backdrop`, `.ws-payment-grid`,
`.ws-payment-option`, etc.) are shared and already exist in the stylesheet.

---

## Step 4 — Settings (`settings.py`)

```python
WAGTAILADMIN_RICH_TEXT_EDITORS = {
    'default': {
        'WIDGET': 'wagtail.admin.rich_text.DraftailRichTextArea',
        'OPTIONS': {
            'features': [
                # ...existing features...
                'my_icon',
            ]
        }
    }
}
```

Also add `"my_icon"` to `STYLER_FEATURES` in `wagtail_hooks.py`.

---

## Critical Gotchas

### ⚠️ The `\u200b` placeholder rule

**Always use `"\u200b"` (U+200B zero-width space) as the placeholder character
in `replaceText`. Never use `" "` (space) or `"\u00a0"` (non-breaking space).**

Why: Wagtail's HTML→contentstate parser uses html5lib, which strips
whitespace-only text nodes. A regular space inside
`<span data-my-icon="..."> </span>` gets stripped, making
`entityRanges[n].length === 0`. Draftail has no text to attach the decorator
to, so the icon is invisible when re-opening the editor.

`\u200b` is not considered whitespace by html5lib and survives the round-trip,
giving `length: 1`.

Verify with:
```python
from wagtail.admin.rich_text.converters.contentstate import ContentstateConverter
import json
html = '<p><span data-my-icon="x" data-my-icon-url="/static/...">\u200b</span>text</p>'
result = json.loads(ContentstateConverter(['my_icon']).from_database_format(html))
assert result['blocks'][0]['entityRanges'][0]['length'] == 1
```

### ⚠️ `js` and `css` must be in every `EntityFeature`

```python
# ✅ Correct
draftail_features.EntityFeature(
    {"type": "MY_ICON", ...},
    js=[STYLER_JS],
    css={"all": [STYLER_CSS]},
)

# ❌ Wrong — popup will never open; registerPlugin is never called
draftail_features.EntityFeature({"type": "MY_ICON", ...})
```

Wagtail loads JS/CSS only at editor init based on active features. Without
these arguments, clicking the toolbar button does nothing.

### ⚠️ Don't load the script tag in `insert_editor_js`

```python
# ✅ Correct — only inject window data
@hooks.register("insert_editor_js")
def editor_js():
    return format_html(
        '<script>window.wagtailStylerMyIcons = {};</script>',
        mark_safe(json.dumps(get_my_icon_choices())),
    )

# ❌ Wrong — loading STYLER_JS twice breaks save behaviour
@hooks.register("insert_editor_js")
def editor_js():
    return format_html('<script src="{}"></script>', STYLER_JS)
```

### ⚠️ Icon entities need their own fast-path in `applyMergedEntity`

The normal merge flow updates an existing entity (used by COLOR/HILITE/FSIZE).
Icons always create a new entity. Without the early `return`, clicking to insert
a second icon overwrites the first.

### ⚠️ CSS attribute selector syntax in `from_database_format`

```python
"from_database_format": {
    "span[data-my-icon]": MyIconElementHandler("MY_ICON")
}
```

Wagtail's converter supports `tag[attr]` CSS attribute selectors. Only
`<span>` elements with `data-my-icon` are matched — other spans are unaffected.

---

## Migrating Existing DB Rows After Changing Placeholder

If you change the placeholder from `" "` to `"\u200b"` after data is already
saved:

```python
import re
from django.db import connection, transaction

with transaction.atomic():
    with connection.cursor() as cursor:
        cursor.execute(
            "SELECT id, my_field::text FROM my_table "
            "WHERE my_field::text LIKE '%%data-my-icon%%'"
        )
        for row_id, val in cursor.fetchall():
            fixed = re.sub(
                r'(<span[^>]*data-my-icon[^>]*>)[ \t\r\n]*(</span>)',
                u'\u0001\\1\u200b\\2',
                val
            ).replace('\u0001', '')
            if fixed != val:
                cursor.execute(
                    "UPDATE my_table SET my_field = %s::jsonb WHERE id = %s",
                    [fixed, row_id]
                )
```

---

## Testing Checklist

1. Insert an icon in the editor and save.
2. Confirm DB contains `<span data-my-icon="..." data-my-icon-url="...">\u200b</span>`:
   ```bash
   PGPASSWORD=secret psql -U tulip_dev -d tulip_dev -h localhost \
     -c "SELECT my_field FROM my_table WHERE my_field::text LIKE '%data-my-icon%';"
   ```
3. Confirm round-trip parser gives `length: 1`:
   ```bash
   uv run python manage.py shell -c "
   from wagtail.admin.rich_text.converters.contentstate import ContentstateConverter
   import json
   html = '<p><span data-my-icon=\"x\" data-my-icon-url=\"/static/x.png\">\u200b</span>after</p>'
   r = json.loads(ContentstateConverter(['my_icon']).from_database_format(html))
   print(r['blocks'][0]['entityRanges'])
   "
   ```
4. Re-open the editor — icon must be visible.
5. Cache-bust if needed: `touch src/wagtail_styler/static/wagtail_styler/styler-i18n-merge.js`
