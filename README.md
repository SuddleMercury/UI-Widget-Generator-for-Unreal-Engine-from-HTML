# Widget Generator — HTML → UMG Pipeline

**Author:** TheOne/SuddleMercury - DZ
**Version:** 2.0 · **Engine:** Unreal Engine 5.x · **Type:** Editor-only plugin

Turn HTML mock-ups into real Unreal **Widget Blueprints** — layout, textures, buttons,
progress bars, scroll boxes, panel hierarchy, and even **UMG animations generated from CSS
`@keyframes`** — in two steps: convert in the browser, import in the editor.

The pipeline has two halves:

1. **Offline Converter** (`Converter/index.html`) — a self-contained web page that renders
   your HTML screens, walks the layout, bakes textures, and exports a ZIP package
   (`manifest.json` + PNG textures). Runs entirely in your browser; nothing is uploaded.
2. **Widget Generator plugin** (this folder) — an Unreal Editor plugin that reads that ZIP
   and generates one Widget Blueprint per HTML screen, plus Texture2D assets, animations,
   and starter Blueprint graph logic.

---

## Installation (first time)

1. Copy the `WidgetGenerator` folder into your project's `Plugins/` directory
   (create `Plugins/` next to your `.uproject` if it doesn't exist).
2. Your project must be a **C++ project** so the editor can compile the plugin:
   - Already C++? Skip ahead.
   - Blueprint-only? Add any empty C++ class once via **Tools ▸ New C++ Class**, or
     right-click the `.uproject` → **Generate Visual Studio project files**.
     You need Visual Studio (Windows) with the "Game development with C++" workload.
3. Double-click your `.uproject`. Unreal reports *"The following modules are missing or built
   with a different engine version: WidgetGenerator. Would you like to rebuild them now?"*
   → click **Yes** and wait for the compile.
4. When the editor opens, you'll find **Tools ▸ Widget Generator** and a toolbar button.

**Important:** whenever you update the plugin's C++ source, repeat step 3. Running an old
binary with a new converter is the classic cause of "my hierarchy imported flat".

## Step 1 — Convert your HTML

1. Open **Tools ▸ Widget Generator** → **Open Offline Converter**
   (or double-click `Plugins/WidgetGenerator/Converter/index.html`).
   *Needs internet once, for the ZIP library from cdnjs.*
2. Click **Select folder** and pick the folder containing your HTML screens.
   Folder mode matters: it lets the converter embed referenced images as textures.
   Single-file mode skips external images (logged as "Skipped non-embeddable texture").
3. Set the batch name and resolution (should match your target UI resolution; 1920×1080 default).
4. **Batch Convert** → watch the log → **Download ZIP**.

Tip: if the converter ever seems dead (buttons doing nothing), press **F12 ▸ Console** in the
browser — a red error there pinpoints the problem immediately.

## Step 2 — Generate widgets in Unreal

1. **Tools ▸ Widget Generator**.
2. **Browse** to the downloaded ZIP.
3. Optionally set a batch name and the content path (default `/Game/UI/Generated`).
4. **Create Widgets**. Assets appear under `Content/UI/Generated/<batch>/`:
   - `WBP_<batch>_<screen>` — one Widget Blueprint per HTML file
   - `Textures/T_tex_*` — embedded images, baked gradients/shadows, rasterized SVGs
   The first widget opens automatically.

A `UWidgetGeneratorBlueprintLibrary::ImportWidgetGeneratorZip` Blueprint node is also
available if you want to script imports.

---

## Terminology (for UMG newcomers)

| Term | Meaning |
|---|---|
| **Widget Blueprint (WBP)** | A UMG asset defining a UI screen. Designer tab = visual layout, Graph tab = Blueprint logic. |
| **UMG** | Unreal Motion Graphics — Unreal's UI system. |
| **Canvas Panel** | A container placing children at exact X/Y positions. Used as the root of every generated screen, because HTML layouts arrive as absolute positions. |
| **Anchor** | The point of the screen a widget "sticks" to when resolution changes. Generated widgets anchor top-left. |
| **Editor plugin** | Runs only inside the Unreal Editor; never ships in a packaged game. |
| **STORE ZIP** | A ZIP with no compression. The importer reads only this kind — the converter always exports it (PNGs are internally compressed anyway). |

## Special HTML attributes (the important part)

Mark up your HTML with these and the generator builds smarter widgets:

| Attribute | Effect in Unreal |
|---|---|
| `data-ue-panel="Inventory"` | The element becomes a named **Canvas Panel** (`Panel_Inventory`) containing everything inside it — child positions are re-based relative to the panel, so one Set Visibility node shows/hides the whole group. Panels nest. |
| `data-ue-panel-default="open"` | Panel starts visible. |
| `data-ue-panel-display="block"` (or `flex`/`grid`) | A panel hidden in the HTML (`display:none` **or** `opacity:0`) still exports — shown temporarily for measurement (hidden-state transforms cleared), then imported **Collapsed**. Perfect for pause menus, popups, damage flashes. |
| `data-ue-toggle="Inventory"` / `data-action` / `onclick` / `role="button"` | The element becomes a clickable **Button**. With `data-ue-toggle`, working graph logic is generated: click → show/hide the matching panel. |
| `id="PB_HealthBar"` / `data-ue-widget="progress"` / `data-ue-progress="--health"` / `data-ue-progress-fill=".fill"` | The element becomes a real **Progress Bar**, keeping its id as the widget name. Percent comes from the CSS variable, a literal value, or the measured width of the fill child; fill color falls back to the fill gradient's end color. |
| `data-ue-bake="image"` (or `data-ue-widget="image"`) | The whole subtree is rasterized into ONE Image widget — ideal for compass strips, tick rulers, and decorative clusters that would otherwise become dozens of tiny widgets. Falls back to per-element import if the subtree references external image files. |
| Element `id`s | Ids become widget names (`TXT_HealthValue`, `IMG_PsyIcon` conventions pass through untouched) so Blueprint/C++ can find widgets by stable names. Inline `<svg>` icons are rasterized automatically and named from their (parent) id. |

## What converts

Text (size, weight, italic, color, alignment, letter-spacing), images (`<img>` + CSS
`background-image`), inline SVG icons, solid backgrounds, borders, rounded corners, simple
linear gradients (baked), box-shadows (baked to soft textures), buttons with real `:hover`
colors (press state derived), CSS `@keyframes` animations (opacity, translate, rotate,
scale — delay and iteration count preserved; `infinite` loops forever), scrollable regions
(`overflow-y: auto/scroll` → Scroll Box), progress bars, sliders, checkboxes, text inputs,
named panels with auto-wired toggle buttons, stacking order, opacity.

Not converted: icon fonts, radial/conic gradients, `backdrop-filter`/masks/blend modes,
3D transforms, skew, `<select>` dropdowns, video, JS-generated UI. Fonts map to the engine
default (Roboto) — bold/italic preserved.

## How animations arrive in Unreal

Each CSS animation becomes a **UMG animation** on the Widget Blueprint (visible in
**Window ▸ Animations** inside the widget editor), named `<Widget>_<keyframesName>`. The
generator also writes Blueprint logic: **Event Construct → Play Animation** for every
animation, with `animation-iteration-count` mapped to *Num Loops To Play* (`infinite` → 0 =
loop forever). Open the **Graph** tab to see or change the wiring. Easing uses Unreal's
smooth cubic keys, so `ease`/`linear` differences are approximated. `animation-delay`
becomes a hold at the start of the timeline.

## Prompting AI for conversion-friendly HTML

When asking an AI to generate UI screens for this pipeline, include these rules:

```
Create a self-contained 1920x1080 HTML game UI screen for Unreal Engine UMG conversion.
Return only valid final HTML with embedded CSS.

Rules:
- use real <button> elements for anything clickable
- mark every major section (menu, popup, tab, sidebar) with data-ue-panel="Name"
- panels that start hidden need data-ue-panel-display="block" so they still export
- controls that open/close a panel get data-ue-toggle="PanelName"
- name progress bars with ids like PB_HealthBar and dynamic text with ids like TXT_HealthValue
- tag dense decorative groups (tick strips, rulers) with data-ue-bake="image"
- keep important text and images as real DOM elements, present at initial page load
- use clean ids and class names (they become widget names in Unreal)
- solid colors, borders, rounded corners, and simple linear gradients convert best
- avoid: backdrop-filter, mask-image, blend modes, 3D transforms, pseudo-element-only
  content, and UI that only appears after JavaScript runs
```

## Package format (WGEN-2)

```
my_batch.zip                 (STORE / uncompressed entries only)
├── manifest.json            # screens + widget list (positions, colors in 0-1 sRGB)
└── textures/*.png           # deduplicated baked images
```

Widget types in the manifest: `text`, `rect`, `image`, `button`, `panel`, `scrollbox`,
`progress`, `slider`, `checkbox`, `textinput`. Coordinates are absolute pixels at the chosen
resolution; the importer re-bases children of panels/scrollboxes onto their container.
WGEN-1 packages (V1 converter) still import.

## For developers — extending the plugin

Source layout (`Source/WidgetGenerator/`):

- `WidgetGeneratorModule.cpp` — menu/toolbar registration, opens the dialog window.
- `SWidgetGeneratorDialog.cpp` — the Slate import window.
- `WidgetGenZipReader.cpp` — minimal stored-ZIP parser (no compression library needed).
- `WidgetGenImporter.cpp` — the heart: manifest parsing, texture creation, WidgetTree
  construction, panel re-basing, two-pass compile.
- `WidgetGenAnimationBuilder.cpp` — authors `UWidgetAnimation` MovieScene tracks
  (RenderOpacity float track + RenderTransform 2D transform track).
- `WidgetGenGraphBuilder.cpp` — generates Blueprint graph nodes (Construct→PlayAnimation,
  OnClicked→toggle visibility).
- `WidgetGeneratorBlueprintLibrary.cpp` — Blueprint-callable import wrapper.

Converter logic lives entirely in `Converter/index.html` (single file, no build step).
Ideas with headroom: font mapping UI, `:hover`-state textures, radial gradients,
WidgetSwitcher generation from tab groups, re-import that updates existing WBPs in place,
CSS `transition` → hover animations.
<img width="961" height="611" alt="image" src="https://github.com/user-attachments/assets/2a970bee-3fb6-41df-991d-7350f9d64aeb" />
<img width="604" height="329" alt="image" src="https://github.com/user-attachments/assets/e84d28cf-45fa-49c2-9477-baf9bd584411" />
<img width="934" height="413" alt="image" src="https://github.com/user-attachments/assets/8851708e-759e-4f61-9627-1ffba135e501" />
<img width="474" height="230" alt="image" src="https://github.com/user-attachments/assets/5914a500-c8ca-42d6-9fe6-5f77c58a4bab" />
<img width="650" height="464" alt="image" src="https://github.com/user-attachments/assets/2d16b15c-d680-4914-8792-effc36b42feb" />
<img width="587" height="405" alt="image" src="https://github.com/user-attachments/assets/57ca64fb-5baa-4a95-8938-608c9f278096" />
<img width="587" height="379" alt="image" src="https://github.com/user-attachments/assets/60fd6e2e-def6-4c85-bc82-b9d09293e63e" />

<img width="350" height="319" alt="image" src="https://github.com/user-attachments/assets/1103e276-9b82-48d1-954c-c6747bfe65d6" />
<img width="875" height="387" alt="image" src="https://github.com/user-attachments/assets/fd65236a-32fd-496c-a1f6-3b167fa06290" />
<img width="691" height="229" alt="image" src="https://github.com/user-attachments/assets/dc1671a7-36b0-41b9-9ed8-a7c81bc8f8d5" />
<img width="733" height="446" alt="image" src="https://github.com/user-attachments/assets/2b3c38c4-ce93-403c-849b-899d8e4b54af" />
<img width="538" height="445" alt="image" src="https://github.com/user-attachments/assets/6e78ea27-2ad1-4565-8b05-af1cef7b9483" />


<img width="746" height="587" alt="image" src="https://github.com/user-attachments/assets/667623b1-6b41-48b9-a5bc-68d73ffbc30a" />
<img width="1047" height="659" alt="image" src="https://github.com/user-attachments/assets/370f5384-23ae-4457-aa4d-927a6162060a" />

