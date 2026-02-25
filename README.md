# 🌐 NovelGlide EPUB Renderer - Web App

A sophisticated **web-based EPUB renderer** that runs inside the **Flutter WebView** to display book content with full HTML/CSS support, interactive pagination, and seamless Dart ↔ JavaScript communication.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Communication Service](#communication-service)
4. [Core Services](#core-services)
5. [API Routes & Messages](#api-routes--messages)
6. [Development Setup](#development-setup)
7. [Build & Deployment](#build--deployment)
8. [Integration with Flutter](#integration-with-flutter)
9. [Troubleshooting](#troubleshooting)
10. [Performance Optimization](#performance-optimization)

---

## Overview

The **NovelGlide EPUB Renderer** is a **TypeScript + Webpack** web application that:

- 📖 **Renders EPUB books** using the industry-standard `epub.js` library
- 🔌 **Communicates with Flutter** via JavaScript message channels
- ⚡ **Handles pagination** intelligently (smooth scroll, RTL support)
- 🔍 **Provides search** functionality across book content
- 🎤 **Enables text-to-speech** by extracting readable text nodes
- 🎨 **Applies styling** dynamically (font size, color, line height)
- 📱 **Adapts to screen** size changes (orientation, tablet mode)

### Why Web-Based?

**Advantages of rendering in WebView instead of native:**
- ✅ Full **HTML/CSS/JavaScript** support
- ✅ **Complex layouts** work as intended
- ✅ **Publisher formatting** respected
- ✅ **Pagination engine** handles text flow
- ✅ **Performance** optimized via browser
- ✅ **Reusable** across Android/iOS/Web

**Trade-offs:**
- ⚠️ Requires **WebView** on device
- ⚠️ **JavaScript bridge** adds overhead
- ⚠️ **Memory usage** higher than pure Dart

---

## Architecture

### System Design

```
Flutter App (Dart)
    ↓
WebViewController
    ↓
Local Web Server (http://localhost:8080)
    ↓
NovelGlide-EpubRenderer (this web app)
    ├─ HTML/CSS/JavaScript
    ├─ EPUB.js Library
    └─ Communication Service
    ↓
JavaScript ↔ Dart Bridge (postMessage API)
    ↓
Event Streams (Flutter)
```

### Component Structure

```
index.js (Entry point)
    ├─ ReaderApi (Main reader engine)
    ├─ CommunicationService (JS ↔ Dart bridge)
    ├─ SearchService (Search functionality)
    ├─ TtsService (Text extraction)
    └─ Utils
        ├─ BreadcrumbUtils (TOC navigation)
        ├─ TextNodeUtils (Text analysis)
        └─ DelayTimeUtils (Async timing)
```

### Directory Structure

```
NovelGlide-EpubRenderer/
├── src/
│   ├── index.html              # HTML entry point
│   ├── index.js                # JavaScript entry point
│   ├── ReaderApi.ts            # Main reader engine
│   ├── services/
│   │   ├── CommunicationService.ts    # JS ↔ Dart communication
│   │   ├── SearchService.ts           # Search in chapter/book
│   │   └── TtsService.ts              # Text extraction for TTS
│   ├── utils/
│   │   ├── BreadcrumbUtils.ts         # Table of contents
│   │   ├── TextNodeUtils.ts           # Text node analysis
│   │   └── DelayTimeUtils.ts          # Async utilities
│   └── sass/
│       └── main.sass                  # Styling
│
├── dist/                       # Built output
│   ├── index.html
│   ├── index.js
│   └── index.css
│
├── webpack.config.js           # Build configuration
├── tsconfig.json               # TypeScript configuration
├── package.json                # Dependencies
└── README.md                   # This file
```

---

## Communication Service

### Overview

The **CommunicationService** is the bridge between JavaScript (web) and Dart (mobile app).

**How it works:**

```
Dart App
    ↓ (sends message via WebView channel)
    ↓
JavaScript CommunicationService.receive()
    ↓
Matches route → calls registered callback
    ↓
Process (e.g., turn page, search)
    ↓
CommunicationService.send() → back to Dart
    ↓
Dart App (streams event to UI)
```

### API

```typescript
class CommunicationService {
    // Set the communication channel (called by Dart)
    setChannel(channel: any): void
    
    // Send a message to Dart
    static send(route: string, data: any = ""): void
    
    // Register callback for incoming message
    static register(route: string, callback: Function): void
    
    // Internal: receive message from Dart
    receive(route: string, data: any): void
}
```

### Message Format

**Dart → JavaScript:**
```json
{
  "route": "nextPage",
  "data": null
}
```

**JavaScript → Dart:**
```json
{
  "route": "setState",
  "data": {
    "startCfi": "epubcfi(/6/4[chap1]!/4/2,/1:0,/1:100)",
    "breadcrumb": "Chapter 1 > Section A",
    "chapterFileName": "chapter_01.xhtml",
    "chapterCurrentPage": 5,
    "chapterTotalPage": 20
  }
}
```

### Registering Routes

Routes are registered in the `ReaderApi` constructor:

```typescript
CommunicationService.register('main', this.main.bind(this));
CommunicationService.register('nextPage', this.nextPage.bind(this));
CommunicationService.register('goto', this.goto.bind(this));
CommunicationService.register('setFontSize', this.setFontSize.bind(this));
```

When Dart sends `{ route: 'nextPage' }`, the registered callback is invoked automatically.

---

## Core Services

### 1. ReaderApi — Main Reader Engine

**File:** `src/ReaderApi.ts`

Manages EPUB loading, pagination, navigation, and styling.

#### Key Properties

```typescript
public book: Book                  // EPUB.js book instance
public rendition: Rendition        // Rendering context
private isSmoothScroll: boolean   // Smooth scroll enabled?
private isRtl: Boolean            // Right-to-left layout?
private fontSize: number          // Current font size
private lineHeight: number        // Line height multiplier
```

#### Key Methods

```typescript
// Entry point (called when Dart initializes reader)
async main(data: {
  destination?: string    // CFI, href, or percentage
  savedLocation?: string  // EPUB location data
}): Promise<void>

// Navigation
async nextPage(): Promise<void>
async prevPage(): Promise<void>
async goto(destination: string): Promise<void>

// Styling
setFontSize(size: number): void
setFontColor(color: string): void
setLineHeight(multiplier: number): void
setSmoothScroll(enabled: boolean): void
```

#### Navigation Logic

**Next/Previous Page:**
```
1. Check RTL layout (right-to-left languages)
2. Call EPUB.js rendition.next() or rendition.prev()
3. Wait for 'relocated' event or 'scrollend' event
4. Emit state update (breadcrumb, CFI, page numbers)
```

**Goto Destination:**
```typescript
// Supports three formats:
await goto("epubcfi(/6/4...)");              // CFI pointer
await goto("chapter_01.xhtml");               // File href
await goto("0.5");                            // Percentage (0-1)
```

#### Page Calculation

Pages are calculated based on EPUB.js location data:

```typescript
get currentPage(): number {
  // Count columns/pages within current chapter
  // Handles both scrolling and pagination modes
}

get totalPage(): number {
  // Total pages in current chapter
  // Based on rendition width/height
}
```

#### State Synchronization

Automatically sends state to Dart on page change:

```typescript
private syncState(location: any) {
  const breadcrumb = BreadcrumbUtils.get(
    this.book.navigation.toc, 
    location.start.href
  );
  
  CommunicationService.send('setState', {
    startCfi: location.start.cfi,
    breadcrumb: breadcrumb,
    chapterFileName: location.start.href,
    chapterCurrentPage: this.currentPage,
    chapterTotalPage: this.totalPage,
  });
}
```

---

### 2. CommunicationService — JavaScript ↔ Dart Bridge

**File:** `src/services/CommunicationService.ts`

Singleton service for bidirectional messaging.

```typescript
// Receive from Dart → route to callback
receive(route: string, data: any): void
  → this.keyMap.get(route)(data)

// Send to Dart
send(route: string, data: any = ""): void
  → this.channel.postMessage(JSON.stringify({route, data}))

// Register callback
register(route: string, callback: Function): void
  → this.keyMap.set(route, callback)
```

**Features:**
- ✅ **Singleton pattern** (`getInstance()`)
- ✅ **Route-based dispatch** (map string routes to callbacks)
- ✅ **Fallback console logging** if channel unavailable
- ✅ **JSON serialization** for safe data transfer

**Channel Setup (called by Dart):**
```dart
// Flutter code:
_dataSource.setChannel();  // Establishes bidirectional channel
```

---

### 3. SearchService — Search Functionality

**File:** `src/services/SearchService.ts`

Provides chapter-wide and whole-book search.

```typescript
// Search in entire book
searchInWholeBook(q: string): void
  → Loads each spine item
  → Calls item.find(query)
  → Returns all results

// Search in current chapter only
searchInCurrentChapter(q: string): void
  → Gets current section from CFI
  → Calls item.find(query)
  → Returns chapter results
```

**Result Format:**
```typescript
// EPUB.js returns array of matches:
{
  cfi: "epubcfi(...)",
  excerpt: "...matching text content...",
  // Additional metadata
}
```

**Messaging:**
```typescript
CommunicationService.send('setSearchResultList', {
  searchResultList: results
});
```

**Performance Notes:**
- ⚠️ Whole-book search **blocks UI** while loading chapters
- ✅ Current chapter search is **fast** (~100ms)
- 💡 Consider implementing **async** search with progress indicator

---

### 4. TtsService — Text-to-Speech Support

**File:** `src/services/TtsService.ts`

Extracts readable text for text-to-speech engine.

```typescript
// Start TTS playback
play(): void
  → Filter visible text nodes
  → Sort by reading order
  → Send first paragraph to Dart

// Play next paragraph
async next(): Promise<void>
  → Removes first node from queue
  → If last node, go to next page
  → Send next paragraph to Dart

// Stop playback
stop(): void
  → Clear node queue
  → Reset state
```

**Text Node Filtering:**
```typescript
this.nodeList = ReaderApi.getInstance().textNodeList
  .filter((node) => {
    const hasContent = node.textContent.trim().length > 0;
    return TextNodeUtils.isVisible(node) && hasContent;
  });
```

**Auto Page-Turn:**
When reaching the last paragraph of a chapter:
1. Check if book ends
2. If yes: send `ttsEnd` to Dart (stops playback)
3. If no: automatically call `nextPage()`, continue TTS

**Messaging Flow:**
```
Dart: { route: 'ttsPlay' }
    ↓
JS: play() → extract visible text nodes
    ↓
JS: send('ttsPlay', 'First paragraph text...')
    ↓
Dart: TtsRepository reads message, plays audio
    ↓
Dart: send { route: 'ttsNext' }
    ↓
JS: next() → dequeue, send next paragraph
```

---

## API Routes & Messages

### Dart → JavaScript Routes

#### **'main'** — Initialize Reader
```json
{
  "route": "main",
  "data": {
    "destination": "epubcfi(...) | chapter.xhtml | 0.5",
    "savedLocation": "location-data-string"
  }
}
```
**Action:** Load book, restore location, emit ready
**Response:** `setState`, `loadDone`

#### **'nextPage'** — Go to Next Page
```json
{
  "route": "nextPage"
}
```
**Action:** Pagination (handles smooth scroll, RTL)
**Response:** `setState` (new location)

#### **'prevPage'** — Go to Previous Page
```json
{
  "route": "prevPage"
}
```
**Action:** Reverse pagination
**Response:** `setState` (new location)

#### **'goto'** — Jump to Position
```json
{
  "route": "goto",
  "data": "epubcfi(...) | chapter.xhtml | 0.75"
}
```
**Action:** Navigate to CFI, href, or percentage
**Response:** `setState` (new location)

#### **'setFontSize'** — Change Font Size
```json
{
  "route": "setFontSize",
  "data": 18
}
```
**Action:** Update font size (in pixels)
**Response:** None (visual change only)

#### **'setFontColor'** — Change Font Color
```json
{
  "route": "setFontColor",
  "data": "#000000"
}
```
**Action:** Update text color (hex code)
**Response:** None (visual change only)

#### **'setLineHeight'** — Change Line Height
```json
{
  "route": "setLineHeight",
  "data": 1.8
}
```
**Action:** Update line height multiplier
**Response:** None (visual change only)

#### **'setSmoothScroll'** — Toggle Smooth Scroll
```json
{
  "route": "setSmoothScroll",
  "data": true
}
```
**Action:** Enable/disable smooth scroll animation
**Response:** None (immediate effect)

#### **'searchInWholeBook'** — Search All Chapters
```json
{
  "route": "searchInWholeBook",
  "data": "query text"
}
```
**Action:** Search entire book
**Response:** `setSearchResultList` (all matches)

#### **'searchInCurrentChapter'** — Search This Chapter
```json
{
  "route": "searchInCurrentChapter",
  "data": "query text"
}
```
**Action:** Search current chapter only
**Response:** `setSearchResultList` (chapter matches)

#### **'ttsPlay'** — Start Text-to-Speech
```json
{
  "route": "ttsPlay"
}
```
**Action:** Begin TTS from current location
**Response:** `ttsPlay` (first paragraph text)

#### **'ttsNext'** — Play Next Paragraph
```json
{
  "route": "ttsNext"
}
```
**Action:** Advance to next readable paragraph
**Response:** `ttsPlay` (next paragraph) or `ttsEnd`

#### **'ttsStop'** — Stop Text-to-Speech
```json
{
  "route": "ttsStop"
}
```
**Action:** Halt TTS playback
**Response:** None

---

### JavaScript → Dart Routes

#### **'setState'** — Location Changed
```json
{
  "route": "setState",
  "data": {
    "startCfi": "epubcfi(/6/4[chap1]!/4/2,/1:0,/1:50)",
    "breadcrumb": "Chapter 1 > Introduction",
    "chapterFileName": "chapter_01.xhtml",
    "chapterCurrentPage": 3,
    "chapterTotalPage": 15
  }
}
```
**Frequency:** Every page change
**Use:** Update UI (page number, breadcrumb, etc.)

#### **'loadDone'** — Reader Ready
```json
{
  "route": "loadDone"
}
```
**Frequency:** Once (after initial load)
**Use:** Hide loading indicator, enable buttons

#### **'saveLocation'** — Location Data
```json
{
  "route": "saveLocation",
  "data": "location-serialized-data"
}
```
**Frequency:** Once (after location generation)
**Use:** Cache location for restoration

#### **'ttsPlay'** — Text Ready for Speech
```json
{
  "route": "ttsPlay",
  "data": "This is the paragraph text to be spoken aloud..."
}
```
**Frequency:** Every paragraph (TTS mode)
**Use:** Feed text to TTS engine

#### **'ttsEnd'** — TTS Complete
```json
{
  "route": "ttsEnd"
}
```
**Frequency:** Once (at book end during TTS)
**Use:** Stop playback, show completion message

#### **'ttsStop'** — TTS Interrupted
```json
{
  "route": "ttsStop"
}
```
**Frequency:** On orientation change during TTS
**Use:** Resume TTS after page recalculation

#### **'setSearchResultList'** — Search Results
```json
{
  "route": "setSearchResultList",
  "data": {
    "searchResultList": [
      {
        "cfi": "epubcfi(/6/4...)",
        "excerpt": "...matching text...",
        // additional metadata
      },
      // ... more results
    ]
  }
}
```
**Frequency:** Once per search
**Use:** Display results in UI

---

## Development Setup

### Prerequisites

```
Node.js: v18+
npm or yarn
```

### Installation

```bash
# Clone repository
cd /Volumes/Transcend/GitHub/NovelGlide-EpubRenderer

# Install dependencies
npm install

# or
yarn install
```

### Development Server

```bash
# Start webpack dev server (hot reload)
npm run dev

# Visit: http://localhost:8080
```

**Features:**
- 🔄 **Hot reload** on file change
- 📊 **Source maps** for debugging
- 🎨 **SASS compilation**
- 📦 **TypeScript** transpilation

### Project Configuration

**TypeScript** (`tsconfig.json`):
- Target: ES2020
- Module: ES6
- Strict mode: enabled

**Webpack** (`webpack.config.js`):
- Entry: `src/index.js`
- Output: `dist/index.js`
- Loaders: Babel, TypeScript, SASS
- Plugins: HtmlWebpackPlugin, MiniCssExtractPlugin

### Code Structure

**Main Entry:**
```javascript
// src/index.js
window.readerApi = ReaderApi.getInstance();
window.communicationService = CommunicationService.getInstance();
window.ttsService = TtsService.getInstance();
window.searchService = SearchService.getInstance();
```

**Adding New Service:**
1. Create service class with `getInstance()` singleton
2. Register routes in `CommunicationService`
3. Export in `index.js`

**Example:**
```typescript
// src/services/MyService.ts
export class MyService {
  constructor() {
    CommunicationService.register('myRoute', this.handle.bind(this));
  }
  
  handle(data: any): void {
    // Process request
    CommunicationService.send('myResponse', result);
  }
  
  static getInstance(): MyService {
    window.myService ??= new MyService();
    return window.myService;
  }
}
```

---

## Build & Deployment

### Production Build

```bash
# Build optimized version
npm run build

# Output: dist/
#   ├── index.html
#   ├── index.js (minified)
#   └── index.css (minified)
```

**Optimizations:**
- ✅ **Minification** (JavaScript, CSS)
- ✅ **Tree shaking** (unused code removal)
- ✅ **Source maps** (for debugging)
- ✅ **ES5 compatibility** (older devices)

### Deployment to Flutter

**Copy built files:**
```bash
# Copy dist/ to Flutter assets
cp -r dist/* /Volumes/Transcend/GitHub/novelglide-flutter/assets/renderer/
```

**Files needed in Flutter:**
```
assets/renderer/
├── index.html
├── index.js
└── index.css
```

**Loading in Flutter:**
```dart
// In reader_server_repository_impl.dart
const String rendererAssetPath = 'assets/renderer/index.html';
```

### Updating EPUB.js

The project uses `epub.js` v0.3.93. To update:

```bash
# Update to latest
npm update epubjs

# Check compatibility
npm test
```

---

## Integration with Flutter

### Communication Flow Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    Flutter App                          │
│                                                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │         ReaderScreen (UI)                        │  │
│  │  - Page number, breadcrumb, buttons              │  │
│  │  - Search results, TTS controls                  │  │
│  └──────────────────────────────────────────────────┘  │
│                       ↕                                  │
│  ┌──────────────────────────────────────────────────┐  │
│  │      ReaderCoreWebViewRepositoryImpl              │  │
│  │  - Navigation (next, prev, goto)                 │  │
│  │  - Styling (fontSize, color, etc.)              │  │
│  │  - Search, TTS                                   │  │
│  └──────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
                      ↕ (WebView channel)
┌──────────────────────────────────────────────────────────┐
│            WebViewController (Native)                    │
│                                                          │
│  - Hosts: http://localhost:8080                         │
│  - JavaScript bridge for postMessage                    │
└──────────────────────────────────────────────────────────┘
                      ↕ (HTTP)
┌──────────────────────────────────────────────────────────┐
│        Local Web Server (Shelf, Dart)                   │
│                                                          │
│  - Serves assets: HTML, CSS, JS, fonts, images          │
│  - Routes requests to appropriate files                 │
└──────────────────────────────────────────────────────────┘
                      ↕ (HTTP)
┌──────────────────────────────────────────────────────────┐
│   NovelGlide-EpubRenderer (This Web App)                 │
│                                                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │         ReaderApi (EPUB.js)                      │  │
│  │  - Book rendering, pagination                    │  │
│  │  - Navigation logic                              │  │
│  │  - Styling application                           │  │
│  └──────────────────────────────────────────────────┘  │
│                                                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │      CommunicationService                        │  │
│  │  - Route dispatch (route → callback)             │  │
│  │  - Message serialization                         │  │
│  │  - Bidirectional messaging                       │  │
│  └──────────────────────────────────────────────────┘  │
│                                                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Services: Search, TTS, Utils                    │  │
│  └──────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### Key Integration Points

**1. Initialization (Flutter)**
```dart
// In reader_core_webview_repository_impl.dart
await _dataSource.loadPage(serverUri);  // Load index.html

_dataSource.send(ReaderWebMessageDto(
  route: 'main',
  data: <String, String?>{
    'destination': pageIdentifier,
    'savedLocation': savedLocation,
  },
));
```

**2. Page Navigation**
```dart
// Dart requests next page
_dataSource.send(ReaderWebMessageDto(route: 'nextPage'));

// JavaScript responds
CommunicationService.send('setState', {...});

// Flutter receives and updates UI
_dataSource.onSetState.listen((state) {
  // Update page number, breadcrumb, etc.
});
```

**3. Search**
```dart
// Dart initiates search
_dataSource.send(ReaderWebMessageDto(
  route: 'searchInWholeBook',
  data: 'query',
));

// JavaScript returns results
CommunicationService.send('setSearchResultList', [...]);

// Flutter receives and displays
_dataSource.onSetSearchResultList.listen((results) {
  // Display search UI
});
```

### Debugging

**Enable WebView debugging:**

**Android:**
```bash
# View WebView console logs
adb logcat | grep chromium
```

**iOS:**
```
Safari → Develop → Select device → Select renderer page
```

**In Web App:**
```typescript
// Fallback console logging
if (!this.channel) {
  console.log(route, data);  // Logs to DevTools
}
```

---

## Troubleshooting

### Common Issues

#### 1. WebView Shows Blank Page

**Symptoms:** White screen, no content

**Diagnosis:**
```bash
# Check if server is running
curl http://localhost:8080/index.html

# Check WebView logs
adb logcat | grep webview
```

**Solutions:**
```typescript
// In ReaderApi.main()
console.log('Book loaded:', this.book);
console.log('Rendition ready:', this.rendition);

// Check if EPUB file exists
// Server must serve book.epub at /book.epub
```

**Common causes:**
- ❌ `book.epub` not found at expected path
- ❌ Server not running or wrong port
- ❌ EPUB file corrupted
- ❌ JavaScript errors in console

#### 2. Navigation Doesn't Work

**Symptoms:** Next/Previous buttons don't respond

**Debug:**
```typescript
// In ReaderApi.nextPage()
console.log('Current page:', this.currentPage);
console.log('Total page:', this.totalPage);
console.log('Is scrolling:', this.isScrolling);

// Check if rendition is ready
if (!this.rendition) {
  console.error('Rendition not initialized');
}
```

**Causes:**
- ❌ `isScrolling` flag stuck (true)
- ❌ `'relocated'` event not firing
- ❌ EPUB has unusual structure

#### 3. Search Returns No Results

**Symptoms:** Search works but shows nothing

**Debug:**
```typescript
// In SearchService.searchInWholeBook()
promiseList.forEach((p, i) => {
  p.then((results) => {
    console.log(`Spine item ${i}:`, results.length, 'matches');
  });
});
```

**Causes:**
- ❌ Case-sensitive search (implement case-insensitive)
- ❌ Special characters in query
- ❌ Text encoded differently

**Fix:**
```typescript
// Normalize search
const normalized = q.toLowerCase();
// Implement fuzzy matching
```

#### 4. TTS Doesn't Work

**Symptoms:** TTS menu appears but no audio

**Debug:**
```typescript
// In TtsService.play()
console.log('Visible nodes:', this.nodeList.length);
console.log('First node text:', this.nodeList[0]?.textContent);
```

**Causes:**
- ❌ No visible text nodes found
- ❌ `TextNodeUtils.isVisible()` too strict
- ❌ Text content is whitespace-only

#### 5. Performance Issues

**Symptoms:** Slow page turns, laggy scrolling

**Debug:**
```typescript
// Measure page turn time
const start = performance.now();
await this.nextPage();
console.log(`Page turn: ${performance.now() - start}ms`);
```

**Optimization tips:**
```typescript
// Debounce scroll events
private debounce(fn: Function, delay: number) {
  let timeout;
  return (...args) => {
    clearTimeout(timeout);
    timeout = setTimeout(() => fn(...args), delay);
  };
}

// Throttle state updates
const throttledSync = this.throttle(
  this.syncState.bind(this), 
  200
);
```

#### 6. Fonts Not Loading

**Symptoms:** Custom fonts appear as fallback

**Debug:**
```typescript
// Check if font file is served
fetch('/fonts/georgia.ttf')
  .then(r => console.log('Font served:', r.ok))
  .catch(e => console.error('Font load failed:', e));
```

**Causes:**
- ❌ Font files not in local server
- ❌ CSS @font-face path incorrect
- ❌ CORS headers missing

---

## Performance Optimization

### Measurement & Profiling

```typescript
// Measure operation time
const measure = (name: string, fn: () => void) => {
  const start = performance.now();
  fn();
  const duration = performance.now() - start;
  console.log(`${name}: ${duration.toFixed(2)}ms`);
};

measure('nextPage', () => this.nextPage());
measure('goto', () => this.goto('chapter_02.xhtml'));
```

### Optimization Strategies

**1. Lazy Load Chapters**
```typescript
// Don't load all chapters at startup
// Load on-demand via goto()
```

**2. Cache Location Data**
```typescript
// EPUB.js caches location data
// Reuse this.book.locations.save()
```

**3. Optimize Text Extraction**
```typescript
// Cache text nodes instead of recalculating
private cachedTextNodes: Array<Node> = [];

get textNodeList(): Array<Node> {
  if (this.cachedTextNodes.length === 0) {
    this.cachedTextNodes = this.extractTextNodes();
  }
  return this.cachedTextNodes;
}
```

**4. Debounce State Updates**
```typescript
// Don't fire setState on every scroll
// Batch updates and emit less frequently
private stateUpdateScheduled = false;

private syncState(location: any) {
  if (this.stateUpdateScheduled) return;
  
  this.stateUpdateScheduled = true;
  setTimeout(() => {
    CommunicationService.send('setState', {...});
    this.stateUpdateScheduled = false;
  }, 100);
}
```

**5. Use Smooth Scroll Sparingly**
```typescript
// Smooth scroll is expensive
// Disable for chapter transitions
if (doGotoNextChapter) {
  this.setSmoothScroll(false);
}
```

### Memory Management

**Indicators:**
- 📊 Monitor via Chrome DevTools → Memory
- ⚠️ Large EPUBs (>50MB) can consume 100MB+ RAM

**Best Practices:**
```typescript
// Unload chapters after use
await item.unload();

// Clear node cache on chapter change
this.cachedTextNodes = [];

// Use WeakMap for caches
private weakCache = new WeakMap();
```

---

## Testing

### Unit Testing Setup

```bash
# Add Jest
npm install --save-dev jest ts-jest

# Create jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'jsdom',
};
```

### Example Test

```typescript
// __tests__/ReaderApi.test.ts
describe('ReaderApi', () => {
  let reader: ReaderApi;

  beforeEach(() => {
    reader = ReaderApi.getInstance();
  });

  it('should initialize with a book', async () => {
    await reader.main({});
    expect(reader.book).toBeDefined();
    expect(reader.rendition).toBeDefined();
  });

  it('should navigate to next page', async () => {
    const initialPage = reader.currentPage;
    await reader.nextPage();
    // Page should change (or reach end)
    expect(reader.currentPage).not.toBe(initialPage);
  });
});
```

---

## Contributing

### Code Style

- Use **TypeScript** for new files
- Follow **camelCase** for methods/variables
- Document public methods with JSDoc
- Use **meaningful names** (not `a`, `b`, `x`)

### Git Workflow

```bash
# Create feature branch
git checkout -b feature/your-feature

# Develop and test
npm run dev

# Build and verify
npm run build

# Commit and push
git commit -m "feat: add new feature"
git push origin feature/your-feature
```

### Pull Request Checklist

- [ ] Code builds without warnings
- [ ] No console errors
- [ ] Tested on Android and iOS (if possible)
- [ ] Documented API changes
- [ ] Updated README if needed

---

## Resources

### Dependencies

- **[EPUB.js](https://github.com/futurepress/epub.js)** v0.3.93 — EPUB rendering
- **Webpack** v5 — Build tool
- **TypeScript** v5 — Language
- **Babel** v7 — ES5 compatibility
- **SASS** v1 — Styling

### Documentation

- [EPUB.js Docs](https://futurepress.github.io/epub.js/)
- [EPUB3 Spec](https://www.w3.org/publishing/epub3/)
- [WebView JavaScript Bridge](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage)

### Related Files

- **Flutter Integration:** `novelglide-flutter/lib/features/reader/`
- **Server Setup:** `novelglide-flutter/lib/features/reader/data/repositories/reader_server_repository_impl.dart`
- **WebView DataSource:** `novelglide-flutter/lib/features/reader/data/data_sources/reader_webview_data_source_impl.dart`

---

## License

Part of the NovelGlide project. See LICENSE file for details.

---

**Last Updated:** 2026-02-25  
**Maintained by:** NovelGlide Development Team  
**Framework:** TypeScript + Webpack + EPUB.js  
**Status:** Active Development
