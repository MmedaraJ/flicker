**AI helped in producing parts of this README**

# Flicker

A Roku channel for browsing Flickr's photo library. Features a grid based home screen with categorized rows and a detail view for individual photos.

https://github.com/user-attachments/assets/1059866e-7588-41b9-b28e-8ce778f9e465

## Setup

1. Clone the repository
2. Run `npm install`
3. Copy `.env.example` to `.env` and add your credentials:
   ```
   ROKU_HOST=<your-roku-ip>
   ROKU_PASSWORD=<your-dev-password>
   FLICKR_API_KEY=<your-flickr-api-key>
   ```
4. Run `npm run deploy` or run with vscode

## Testing

```
npm test
```

Or test with vscode run option

**This repo has over 80% test coverage**

Tests use the [Rooibos](https://github.com/georgejecook/rooibos) framework. Test files are located in `source/tests/` mirroring the source structure.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                           mainScene                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────────┐  │
│  │ ScreenStack │  │ ContentTask │  │        UI Screens           │  │
│  │   Logic     │  │   Logic     │  │  (gridScreen/detailsScreen) │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          Task Layer                                 │
│  ┌─────────────────┐  ┌──────────────────┐  ┌───────────────────┐   │
│  │ mainLoaderTask  │  │ rowBatchLoader   │  │ rowPaginationTask │   │
│  │ (initial load)  │  │ (vertical pages) │  │ (horizontal pages)│   │
│  └─────────────────┘  └──────────────────┘  └───────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Service Layer                                │
│  ┌─────────────────┐  ┌──────────────────┐  ┌───────────────────┐   │
│  │ RowLoaderService│  │  ImageService    │  │  NetworkService   │   │
│  └─────────────────┘  └──────────────────┘  └───────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       Provider Layer                                │
│  ┌─────────────────┐  ┌──────────────────┐                          │
│  │ ProviderFactory │─▶│  FlickrProvider  │                          │
│  └─────────────────┘  └──────────────────┘                          │
└─────────────────────────────────────────────────────────────────────┘
```

## The Data Journey

### 1. App Launch

When the app starts, `mainScene.init()` orchestrates three things:
- Initializes the screen stack for navigation
- Shows the grid screen (empty, with loading indicator)
- Kicks off `ContentTask.start()` to fetch initial data

The UI remains responsive while data loads in the background via `mainLoaderTask`.

### 2. Initial Content Load

`mainLoaderTask` doesn't fetch all rows at once. Instead:

1. `HomePageConfig.getRowBatch(0)` returns the first batch of row configurations (10 rows)
2. `RowLoaderService.loadRows()` takes these configs and fires parallel network requests
3. Each request hits the Flickr API through `ProviderFactory` → `FlickrProvider`
4. Responses are parsed into `ContentNode` trees and returned to the UI

**Why parallel requests?** Loading 10 rows sequentially would be noticeably slow. `NetworkService.startParallelRequests()` fires all requests simultaneously and collects responses as they arrive, reducing perceived load time significantly.

### 3. Provider Abstraction

Row configurations use an action string format: `"flickr.search"`, `"flickr.popular"`, etc.

```
action: "flickr.search" → provider: "flickr", method: "search"
```

`ProviderFactory` parses this and routes to the appropriate provider. This abstraction exists so the app could support multiple image sources (Unsplash, Pexels, etc.) without touching UI code. Currently only Flickr is implemented, but the pattern is established.

**Trade-off:** This adds a layer of indirection for a single provider. However, it keeps the door open for expansion and separates API specific logic from business logic.

### 4. Two Types of Pagination

The app implements two distinct pagination strategies:

#### Vertical Pagination (Loading More Rows)

As users scroll down through rows, `gridScreen` monitors focus position. When approaching the bottom:

```
if rowIndex >= loadedRowCount - rowThreshold then loadMoreRows()
```

`rowBatchLoaderTask` fetches the next batch of row configurations (another 10 rows) and appends them to the content tree. This continues until all rows are loaded.

**Why batch rows?** Loading 100 rows upfront, for example, would mean 100 parallel API calls. This is excessive and slow. Batching balances initial load time against scroll smoothness.

#### Horizontal Pagination (Loading More Items in a Row)

Each row starts with 10 items. As users scroll right within a row:

```
if itemIndex >= itemCount - itemThreshold then loadMoreForRow()
```

`rowPaginationTask` fetches the next page (10 more items) for that specific row. This continues up to `MAX_PER_ROW` (50 items).

**Why cap at 50?** Roku devices have memory constraints. Unbounded horizontal scrolling could degrade performance. 50 items provides variety without risking memory pressure.

### 5. The Grid Screen

`gridScreen` is the main UI component. It manages:
- A `RowList` for displaying content
- Focus tracking for both pagination triggers
- Task lifecycle (creating, observing, cleaning up pagination tasks)

When an item is selected, `onGridScreenItemSelected` extracts the row and item index, then tells `DetailsScreen` to show that specific image.

### 6. Screen Stack Navigation

`ScreenStack` manages a simple navigation stack. When showing a new screen, it's pushed onto the stack. The back button pops screens off. This pattern keeps navigation logic centralized and consistent.

### 7. Details Screen

The detail view receives a single `ContentNode` with image metadata. It displays the larger image and metadata and fetches additional info (size, dimensions) via `imageInfoTask`.

## Technical Decisions

### Separation of Concerns

- **Tasks** handle async operations (network calls happen off the render thread)
- **Services** contain business logic (parsing, data transformation)
- **Providers** encapsulate API specific details
- **UI Logic** files keep component scripts focused on rendering

This separation makes the codebase testable. Services and providers can be unit tested without spinning up SceneGraph components.

### Configuration Driven Rows

Row definitions live in `HomePageConfig.getRows()`. Adding a new category is a one line change:

```brightscript
{ title: "New Category", action: "flickr.search", params: { tags: "keyword" } }
```

No UI code changes required. The same config drives both the initial load and pagination metadata.

### Secrets Management

API keys are stored in `.env` (gitignored) and injected at build time into a generated `secrets.bs` file. This keeps credentials out of version control while maintaining a simple build process.

**Trade-off:** The key still ends up in the deployed package. For a public API like Flickr this is acceptable. For sensitive credentials, a server side proxy would be the proper solution.

## What Could Be Added

- **Caching:** Store fetched images/metadata locally to reduce API calls on repeat visits
- **Error states:** Currently failed requests are silently skipped. User facing error messages could improve UX, depending on the desired user experience
- **Search functionality:** Allow users to search for specific tags or keywords
- **Favorites:** Let users save images locally using Roku's registry

These were omitted to keep scope focused, but the architecture supports adding them without major refactoring.
