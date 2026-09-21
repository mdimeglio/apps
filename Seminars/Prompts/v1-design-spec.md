# macOS PDF Presenter — Product and Implementation Design

**Status:** Initial V1 specification  
**Audience:** Codex / implementation agent  
**Platform:** macOS 27 only for V1  
**Primary use case:** Presenting PDF slide decks in Zoom from a native macOS app  
**Working title:** S

---

## 1. Purpose

Build a small, native macOS application for presenting PDF slide decks cleanly and reliably.

The initial motivation is that general-purpose PDF viewers are not designed around slide presentation. The app should feel more like QuickTime Player than Preview or Acrobat: the content dominates the window, interface chrome stays out of the way, the window naturally matches the slide aspect ratio, and advancing slides is visually seamless.

The most important product qualities are:

1. **Native Mac feel.** Follow current macOS 27 conventions and use the latest appropriate SwiftUI/AppKit APIs.
2. **Flicker-free slide changes.** Moving between pages must not produce flashes, blank frames, transient rescaling, or other visual artefacts.
3. **Minimal presentation-focused UI.** The PDF page is the app; controls are hidden until needed.
4. **Excellent Zoom window-sharing behaviour.** In windowed mode, the presentation window should match the slide aspect ratio so the shared window contains only the presentation surface.
5. **A deliberately small V1.** Implement the core experience well before adding presenter tools, annotations, remotes, or other advanced features.

---

## 2. Target user and scenario

The initial user is a mathematician who presents talks from PDF slide decks. The slides may be generated from LaTeX or handwritten and exported to PDF.

Decks do not contain embedded animations. However, they often use **builds across consecutive PDF pages**: for example, page 1 contains the first bullet, page 2 contains the first two bullets, and so on. Because most of the pixels are identical across these pages, any flash or intermediate frame is especially noticeable.

The first target scenario is:

- open a PDF;
- share the app's presentation window in Zoom;
- navigate using the keyboard;
- keep application controls hidden during the talk;
- optionally enter normal macOS full screen.

Future in-person presentation features may be added later, including an iPhone/iPad remote and a presenter view, but they are outside V1.

---

## 3. V1 scope

### 3.1 In scope

V1 must support:

- opening local PDF files;
- opening PDFs through the app's **File > Open…** command;
- appearing in Finder's **Open With** list for PDF files without attempting to become the default PDF application;
- opening a PDF passed to the app from Finder;
- multiple independent document windows, each showing a different PDF;
- read-only presentation of PDFs;
- a window whose aspect ratio is derived from the first page of the PDF;
- minimal, auto-hiding window chrome;
- standard macOS window controls, including close, minimise, and full screen;
- keyboard slide navigation;
- standard macOS full-screen behaviour;
- flicker-free page transitions;
- pre-rendering/caching of neighbouring pages;
- fitting pages without cropping;
- rendering that remains visually stable during PDF “build” sequences.

### 3.2 Explicitly out of scope for V1

Do **not** add the following unless required for the core implementation:

- presenter notes or presenter view;
- thumbnail navigator;
- slide overview/grid;
- drawing or freehand annotations;
- laser-pointer mode;
- iPhone/iPad remote;
- editing PDFs;
- transitions or animations between slides;
- embedded PDF links or rich PDF interaction;
- remembering arbitrary custom window sizes;
- a complex preferences UI;
- support for old macOS releases;
- Mac App Store packaging as part of the initial implementation.

These are possible follow-up features, not V1 requirements.

---

## 4. Interaction model

### 4.1 Opening a document

The app should behave like a normal document-oriented Mac app, but documents are read-only.

Expected flows:

- Launching a PDF with this app from Finder opens that PDF directly.
- Right-clicking a PDF in Finder and choosing **Open With** should list the app.
- **File > Open…** presents the standard system file picker for PDFs.
- If the app is launched without a document, present a simple, native way to choose a PDF to open. Avoid unnecessary “new document” affordances because the app does not create PDFs.
- More than one PDF may be open at once, with one independent presentation window per document.

### 4.2 Initial window state

Opening a document should create a normal, resizable window rather than immediately entering full screen.

The initial window should:

- use the first PDF page to establish the window's content aspect ratio;
- open at a generous size that fits comfortably on the current display;
- preserve that aspect ratio while the user resizes the window;
- not remember custom window sizes in V1.

The goal is similar to QuickTime Player: the window naturally takes on the geometry of the media being shown.

### 4.3 Window chrome

The slide should visually occupy the window.

At rest:

- title bar and presentation controls should be hidden;
- the slide surface should appear clean enough to share directly in Zoom.

When the pointer moves over the window:

- standard window controls should appear as a translucent overlay;
-  controls should disappear after a 2 second delay from the last pointer movement;
- normal windowed presentation should be a first-class experience, not merely a prelude to full screen;
- use current macOS 27 design conventions, including Liquid Glass where appropriate;
- rely on native SwiftUI/system controls where possible rather than imitating system chrome manually.

The UI should feel like a contemporary Apple media app, not a conventional PDF editor.

---

## 5. Navigation

V1 keyboard bindings:

| Input | Behaviour |
|---|---|
| Right Arrow | Next page |
| Space | Next page |
| Left Arrow | Previous page |
| Escape | Exit full screen when in full screen, following normal macOS conventions |

Do not invent additional shortcuts in V1 unless required by system behaviour.

### 5.1 Key repeat

Holding Left or Right should use normal macOS key-repeat behaviour:

- advance one page per repeat event;
- preserve page order;
- do not skip intermediate pages just to catch up;
- pre-render ahead in the direction of travel so repeat navigation remains smooth.

If the user requests a page that is not ready to display, prefer briefly keeping the current slide visible until the next slide is ready rather than showing a blank or partially rendered intermediate state.

---

## 6. Aspect-ratio and page-sizing rules

### 6.1 Windowed mode

The **first page** defines the presentation window's aspect ratio for the lifetime of that document window.

When the window is resized:

- maintain that fixed aspect ratio;
- the page should fit within the content area without cropping.

If a later page has a different aspect ratio:

- do **not** change the window's aspect ratio;
- fit that page as large as possible within the established presentation rectangle;
- preserve the full page;
- fill any unused region with black.

Most real decks are expected to have a consistent page size; this rule exists to make mixed-size PDFs behave predictably.

### 6.2 Full-screen mode

Use the standard macOS full-screen mechanism.

In full screen:

- do not crop the slide;
- scale it as large as possible while preserving its aspect ratio;
- fit to whichever display dimension is limiting;
-  fill letterbox/pillarbox regions with black;
- allow standard ways of leaving full screen, including Escape where appropriate.

## 7. Rendering requirements

Page changes must appear as a direct, clean replacement of the old slide with the new slide.

There must be no visible:

- blank frame;
- white or black flash;
- temporary low-resolution page;
- half-rendered page;
- relayout flash;
- transient change in page position;
- transient change in scaling;
- visible teardown/recreation of the PDF view;
- other presentation artefact.

For slides that differ only by a small “build” element, all unchanged content should remain visually stationary from one page to the next.

Window resizing must remain responsive without compromising final image quality. The user should perceive continuous content during the resize, followed by a crisp final image.

---

## 8. macOS implementation approach

Target **macOS 27 only**. There is no need to carry compatibility shims for older OS releases in V1.

General rule: use the newest stable, idiomatic APIs available in the currently installed Xcode/macOS 27 SDK, but verify exact API names and availability against the SDK rather than assuming an API from memory.

Do not implement Zoom-specific APIs or integrations. The app simply needs to present a clean shareable window.

### 8.1 SwiftUI first, AppKit where useful

Prefer:

- SwiftUI for the application structure, commands/menus, document/window composition, overlays, and general state;
- AppKit interop only where it materially improves native window behaviour or rendering control.

### 8.2 Document model

Treat PDFs as read-only documents.

Use the modern document/window APIs available for macOS 27 where they are a good fit. The implementation should support multiple open PDFs as separate windows.

### 8.3 PDF handling

Use PDFKit and/or Core Graphics for:

- opening PDF data;
- enumerating pages;
- reading page bounds;
- rendering pages.

Keep presentation state separate from the underlying PDF document object.

### 8.4 Concurrency

Off-screen rendering should not block the main thread.

Use current Swift concurrency patterns where appropriate:

- main actor for UI state and presentation swaps;
- background tasks/actors for page rendering and colour analysis;
- cancellation when cached renders are no longer useful, especially after resizing or changing direction quickly.

Do not introduce a large concurrency abstraction layer for V1; keep ownership and task lifetimes obvious.

---

## 9. Error handling and edge cases

Handle common problems simply and natively.

Examples:

- invalid/corrupt PDF: show a clear error and leave the app usable;
- zero-page PDF: treat as an error/unsupported document;
- later page with different aspect ratio: fit inside the first-page presentation ratio and fill background;
- extremely large page: render at the required display resolution, not arbitrary PDF source dimensions;
- navigation before first/after last page: remain on the current boundary page; no wrapping;
- resize invalidates target-size renders: keep displaying a scaled current render while replacements are generated.

---

## 10. Performance priorities

Order of priorities:

1. No flicker or blank frames.
2. Immediate-feeling navigation when neighbour pages are cached.
3. Correct geometry and pixel stability.
4. Crisp rendering.
5. Reasonable memory usage.
6. Fast initial open.

It is acceptable for first-open or an uncached jump to wait a fraction longer if the alternative is displaying an intermediate broken state.

---

## 11. Accessibility and native behaviour

Even though the visible UI is minimal, retain standard macOS behaviours:

- keyboard navigation must work reliably;
- menus should expose important commands where appropriate;
- standard window actions should remain accessible;
- system appearance changes should be handled naturally by system components;
- do not unnecessarily replace native controls with inaccessible custom-drawn equivalents.

---

## 12. Distribution strategy

Distribution is **not** part of the first coding milestone, but the project should avoid choices that make later distribution difficult.

Initially:

- build and run locally from Xcode;
- optimise for the developer's current Mac running macOS 27.

Later, for sharing with colleagues:

- use normal Apple code signing;
- enable an appropriate sandbox/hardened runtime configuration from the beginning where practical;
- notarise a release build;
- distribute a signed app, likely in a DMG;
- Mac App Store distribution is optional and not required.

Do not let packaging work distract from the V1 presentation experience.

---

## 13. Acceptance criteria for V1

V1 is complete when all of the following are true.

### File/document behaviour

- [ ] A PDF can be opened with File > Open….
- [ ] The app appears in Finder's Open With choices for PDFs.
- [ ] Opening a PDF from Finder opens it in the app.
- [ ] Multiple PDFs can be open in separate windows simultaneously.
- [ ] The app is read-only and does not present irrelevant “new PDF” creation UI.

### Window behaviour

- [ ] The first page sets the document window's aspect ratio.
- [ ] Manual resizing preserves that aspect ratio.
- [ ] Initial window size is large but fits on screen.
- [ ] Window controls can auto-hide so the page dominates the window.
- [ ] Standard macOS full screen works.
- [ ] Escape exits full screen according to normal macOS behaviour.

### Navigation

- [ ] Right Arrow advances one page.
- [ ] Space advances one page.
- [ ] Left Arrow moves back one page.
- [ ] Holding an arrow key follows ordinary key repeat and advances one page per repeat.
- [ ] Navigation stops at the first/last page rather than wrapping.

### Rendering

- [ ] Page changes never display a blank intermediate frame.
- [ ] There is no obvious flash between pages.
- [ ] “Build” sequences appear visually stable except for the content that actually changes.
- [ ] Current/adjacent pages are pre-rendered.
- [ ] If a requested page is not ready, the old page remains displayed until the new one is ready.
- [ ] No crossfade or transition animation is applied to ordinary page changes.

### Sizing/background

- [ ] Pages are never cropped in windowed or full-screen presentation.
- [ ] A later page with a different aspect ratio fits inside the established presentation rectangle.
- [ ] Letterbox/pillarbox regions are filled with black.
- [ ] Page render and background update atomically.

### Zoom use

- [ ] The window can be shared in Zoom without permanent toolbars/sidebars taking space from the slide.
- [ ] With chrome hidden, the shared window reads visually as the slide itself.

---

## 14. Testing strategy

Include tests where they provide useful confidence, but prioritise end-to-end visual behaviour over artificial unit-test coverage.

### 14.1 Unit-test candidates

- first-page aspect-ratio calculation;
- fit-without-cropping geometry;
- navigation boundary logic;
- render-cache keys/eviction;
- background-colour inference on synthetic images;
- direction-priority logic for pre-render requests.

### 14.2 Manual/visual tests

Create or collect a few small PDF fixtures:

1. 16:9 white-background slides with incremental builds;
4. a PDF whose first page is 16:9 and one later page has a different aspect ratio;
5. a high-detail mathematical slide with fine text/vector graphics.

For each:

- navigate slowly forward/backward;
- hold the arrow key to exercise repeat;
- reverse direction quickly;
- resize continuously;
- enter/leave full screen;
- share the window in Zoom and inspect the remote view if possible.

The decisive test is whether slide builds look like a stable slide with only the intended new content appearing.

---

## 15. Codex implementation instructions

Implement the **complete V1 in one pass** rather than stopping after tiny scaffolding milestones.

Before coding:

1. inspect the current macOS 27 / Xcode 27 SDK and project templates;
2. choose the most idiomatic current APIs available;
3. write a short implementation plan in the repository;
4. call out any places where SwiftUI alone is insufficient and a small AppKit bridge is preferable.

During implementation:

- optimise for correctness and native behaviour, not clever abstraction;
- keep the presentation renderer isolated and easy to reason about;
- avoid adopting a stock PDF viewer if it compromises the no-flicker requirement;
- use structured concurrency for off-main-thread rendering;
- make navigation deterministic;
- preserve the old fully rendered page until the replacement is complete;
- keep the codebase small enough to inspect manually;
- use red/green test driven development.

When finished:

1. build the project;
2. run tests;
3. fix compiler warnings/errors that are within project control;
4. provide concise run instructions;
5. summarise any known limitations;
6. identify the first things worth evaluating by hand, especially page-transition stability and window behaviour.

The expected output is a **complete Xcode project/repository that can be opened and run**, not merely code snippets.

---

## 16. Guidance for implementation decisions not specified here

When this document does not prescribe a detail, use the following order of preference:

1. standard macOS behaviour;
2. current Apple Human Interface Guidelines and current SDK conventions;
3. minimal UI;
4. predictable behaviour over configurability;
5. rendering stability over transition effects;
6. a small maintainable implementation over a general framework.

Do not add speculative features simply because they are easy to implement.

---

## 17. Future ideas — not V1

Possible later work, driven by real usage:

- letter box colouring to match slide background;
- temporary pen/highlighter annotations;
- laser-pointer mode;
- presenter display with current/next slide;
- iPhone/iPad remote for in-person talks;
- slide thumbnails/overview;
- jump-to-slide UI;
- remembered window size/state;
- presentation timer;
- richer keyboard shortcuts;
- persistent annotation layers;
- app packaging for wider distribution;
- App Store release if useful.

Do not implement these in v1.
