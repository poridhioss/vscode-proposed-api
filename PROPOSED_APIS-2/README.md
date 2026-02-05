# VS Code Proposed APIs Part-2

Search & Files APIs provide powerful capabilities for finding files and searching text content across workspaces. These APIs are essential for code navigation, refactoring tools, and AI-powered code understanding. This guide covers the 6 search-related APIs in depth, showing how Puku uses them and how to modify them.

![](./images/image-1.svg)

## Quick Reference Table

| API | Purpose | Puku Usage |
|-----|---------|------------|
| `findFiles2` | Find files with multiple glob patterns | File discovery, context gathering |
| `findTextInFiles` | Search text content (callback-based) | Code search, reference finding |
| `findTextInFiles2` | Search text content (AsyncIterable) | Modern streaming search |
| `textSearchProvider` | Register custom text search | Custom search backends |
| `textSearchProvider2` | Updated text search provider | Enhanced search features |
| `aiTextSearchProvider` | Register AI-powered search | Semantic code search |

## File Locations in Puku Editor

```
src/chat/
├── package.json                                    ← API declarations
├── src/extension/
│   └── vscode.proposed.*.d.ts                     ← Copied type definitions
└── src/platform/search/
    ├── common/searchService.ts                    ← Search interface
    ├── vscode/baseSearchServiceImpl.ts            ← Base implementation
    └── vscode-node/searchServiceImpl.ts           ← Node.js implementation
```

## API 1: findFiles2

### What Problem Does It Solve?

The stable `vscode.workspace.findFiles()` only accepts a single glob pattern. What if you need to find all `.ts` AND `.js` files in one query? Or exclude multiple directories at once?

The `findFiles2` API solves this by accepting arrays of glob patterns for both inclusion and exclusion, making complex file searches possible in a single call.

**Real-world scenario:** Puku Editor needs to gather context from all source files. Instead of making separate queries for `.ts`, `.tsx`, `.js`, and `.jsx` files, it can use `findFiles2` with `['**/*.ts', '**/*.tsx', '**/*.js', '**/*.jsx']` in one call.

![](./images/image-2.svg)

### API Definition

**Definition File:** `vscode.proposed.findFiles2.d.ts`

**Location in VS Code:** `src/vscode-dts/vscode.proposed.findFiles2.d.ts`

**Location in Puku:** `src/chat/src/extension/vscode.proposed.findFiles2.d.ts`

```typescript
declare module 'vscode' {

    export interface FindFiles2Options {
        /**
         * Glob patterns to exclude. Combined with logical AND.
         */
        exclude?: GlobPattern[];

        /**
         * Which exclude settings to follow. Defaults to searchAndFilesExclude.
         */
        useExcludeSettings?: ExcludeSettingOptions;

        /**
         * Maximum number of results. Defaults to 20000.
         */
        maxResults?: number;

        /**
         * Which ignore files to respect (.gitignore, .ignore).
         */
        useIgnoreFiles?: {
            local?: boolean;    // Workspace root
            parent?: boolean;   // Parent directories
            global?: boolean;   // Global ignore files
        };

        /**
         * Whether to follow symlinks.
         */
        followSymlinks?: boolean;
    }

    export enum ExcludeSettingOptions {
        None = 1,                    // Don't use any excludes
        FilesExclude = 2,            // Use files.exclude setting
        SearchAndFilesExclude = 3    // Use both files.exclude and search.exclude
    }

    export namespace workspace {
        /**
         * Find files across all workspace folders.
         * @param filePattern Array of glob patterns (combined with OR)
         * @param options Search options
         * @param token Cancellation token
         */
        export function findFiles2(
            filePattern: GlobPattern[],
            options?: FindFiles2Options,
            token?: CancellationToken
        ): Thenable<Uri[]>;
    }
}
```

**What this API adds:**

The `findFiles2` API enhances file searching in **three key ways**:

#### 1. Include Patterns with OR Logic

![](./images/image-3.svg)

When you pass an array of include patterns like `['*.ts', '*.js', '*.tsx']`, a file is included if it matches **any one** of the patterns. This is OR logic - the file `app.ts` matches because it satisfies `*.ts`, even though it doesn't match `*.js` or `*.tsx`. Previously with `findFiles`, you'd need three separate API calls and then merge the results manually.

#### 2. Exclude Patterns with AND Logic

![](./images/image-4.svg)

The exclude patterns work differently - they use AND logic. When you specify `exclude: ['*test*', '*spec*']`, a file is only excluded if it matches **all** the patterns simultaneously. For example:
- `app.test.ts` is **kept** because it only matches `*test*`, not `*spec*`
- `app.spec.ts` is **kept** because it only matches `*spec*`, not `*test*`
- `app.test.spec.ts` is **excluded** because it matches both `*test*` AND `*spec*`

This AND logic for excludes is intentional - it prevents overly aggressive filtering where files matching just one pattern would be removed.

#### 3. Fine-grained Ignore File Control

![](./images/image-5.svg)

When searching for files, you often want to skip files that are in `.gitignore` - like `node_modules/`, `dist/`, or `.env` files. But `.gitignore` files can exist at multiple levels in your file system, and sometimes you need control over which ones to respect.

The `useIgnoreFiles` option lets you configure this precisely:

- **`local`**: The `.gitignore` in your workspace root (e.g., `my-app/.gitignore`). This typically contains project-specific ignores like `node_modules/`, `dist/`, and `coverage/`.

- **`parent`**: Any `.gitignore` files in directories above your workspace (e.g., `projects/.gitignore`). Useful in monorepos where parent directories might have shared ignore rules.

- **`global`**: Your personal global gitignore at `~/.gitignore` (or `~/.config/git/ignore`). This usually contains OS-specific files like `.DS_Store` (macOS) or `Thumbs.db` (Windows).

**When would you disable a level?**

Your global `~/.gitignore` often contains personal preferences that shouldn't apply to every project. For example, you might globally ignore all `*.log` files to keep searches clean. But when debugging, you may need to search through those log files in a specific project. By setting `global: false`, you bypass your personal ignore rules while still respecting the project's local `.gitignore` (which ignores `node_modules/`, `dist/`, etc.). This gives you the flexibility to temporarily include files that are normally hidden, without affecting the project-level ignore rules that keep large folders like `node_modules/` out of your search results.

### How Puku Editor Uses It

**Main Implementation:** [baseSearchServiceImpl.ts](src/chat/src/platform/search/vscode/baseSearchServiceImpl.ts)

```typescript
export class BaseSearchServiceImpl extends AbstractSearchService {
    override findFiles(
        filePattern: vscode.GlobPattern | vscode.GlobPattern[],
        options?: vscode.FindFiles2Options,
        token?: vscode.CancellationToken
    ): Thenable<vscode.Uri[]> {
        // Convert single pattern to array for findFiles2
        const filePatternToUse = Array.isArray(filePattern) ? filePattern : [filePattern];
        return vscode.workspace.findFiles2(filePatternToUse, options, token);
    }
}
```

The base implementation wraps the proposed `findFiles2` API. It accepts either a single pattern or an array of patterns, normalizes them into an array, and passes them directly to `findFiles2`. This provides a consistent interface for the rest of the codebase.

**Extended Implementation:** `src/chat/src/platform/search/vscode-node/searchServiceImpl.ts` 

```typescript
export class SearchServiceImpl extends BaseSearchServiceImpl {
    override async findFiles(
        filePattern: vscode.GlobPattern | vscode.GlobPattern[],
        options?: vscode.FindFiles2Options,
        token?: vscode.CancellationToken
    ): Promise<vscode.Uri[]> {
        // Add Copilot ignore patterns to excludes
        const copilotIgnoreExclude = await this._ignoreService.asMinimatchPattern();
        if (options?.exclude) {
            options = {
                ...options,
                exclude: copilotIgnoreExclude
                    ? options.exclude.map(e => combineGlob(e, copilotIgnoreExclude))
                    : options.exclude
            };
        }
        return await super.findFiles(filePattern, options, token);
    }
}
```

The extended implementation adds Puku-specific behavior on top of the base class. It integrates with the ignore service to automatically exclude files matching `.copilotignore` patterns. This ensures that sensitive or irrelevant files are never included in search results, even if the caller doesn't explicitly exclude them.

### Pattern Combination Logic

Understanding how patterns combine is crucial:

![](./images/image-6.svg)

## API 2: findTextInFiles

### What Problem Does It Solve?

The stable VS Code API provides `vscode.workspace.openTextDocument()` and `TextDocument.getText()` - but these only work with files that are already open in the editor. What if you need to search for a string across thousands of files in your workspace, including files you've never opened?

The `findTextInFiles` API solves this by providing full-workspace text search with:

- **Regex support** - Search with patterns like `function\s+\w+` to find all function declarations
- **Case sensitivity control** - Match `Error` vs `error` as needed
- **Word boundary matching** - Find `log` without matching `logging` or `catalog`
- **Result streaming** - Get results via callback as they're found, not all at once
- **Include/exclude patterns** - Limit search to specific file types or folders

**Real-world scenario:** When Puku Editor needs to find all usages of a function across the codebase (not just open files), it uses `findTextInFiles` to search every file and stream results back as they're found.

![](./images/image-7.svg)

### API Definition

**Definition File:** `vscode.proposed.findTextInFiles.d.ts`

**Location in Puku:** `src/chat/src/extension/vscode.proposed.findTextInFiles.d.ts`

```typescript
declare module 'vscode' {

    export interface FindTextInFilesOptions {
        /**
         * A {@link GlobPattern glob pattern} that defines the files to search for.
         */
        include?: GlobPattern;

        /**
         * A {@link GlobPattern glob pattern} that defines files and folders to exclude.
         */
        exclude?: GlobPattern;

        /**
         * Whether to use the default and user-configured excludes. Defaults to true.
         */
        useDefaultExcludes?: boolean;

        /**
         * The maximum number of results to search for.
         */
        maxResults?: number;

        /**
         * Whether external files that exclude files, like .gitignore, should be respected.
         * See the vscode setting `"search.useIgnoreFiles"`.
         */
        useIgnoreFiles?: boolean;

        /**
         * Whether global files that exclude files, like .gitignore, should be respected.
         * See the vscode setting `"search.useGlobalIgnoreFiles"`.
         */
        useGlobalIgnoreFiles?: boolean;

        /**
         * Whether files in parent directories that exclude files, like .gitignore, should be respected.
         * See the vscode setting `"search.useParentIgnoreFiles"`.
         */
        useParentIgnoreFiles?: boolean;

        /**
         * Whether symlinks should be followed while searching.
         * See the vscode setting `"search.followSymlinks"`.
         */
        followSymlinks?: boolean;

        /**
         * Interpret files using this encoding.
         * See the vscode setting `"files.encoding"`.
         */
        encoding?: string;

        /**
         * Options to specify the size of the result text preview.
         */
        previewOptions?: TextSearchPreviewOptions;

        /**
         * Number of lines of context to include before each match.
         */
        beforeContext?: number;

        /**
         * Number of lines of context to include after each match.
         */
        afterContext?: number;
    }

    export interface TextSearchQuery {
        /**
         * The text pattern to search for.
         */
        pattern: string;

        /**
         * Whether or not `pattern` should match multiple lines of text.
         */
        isMultiline?: boolean;

        /**
         * Whether or not `pattern` should be interpreted as a regular expression.
         */
        isRegExp?: boolean;

        /**
         * Whether or not the search should be case-sensitive.
         */
        isCaseSensitive?: boolean;

        /**
         * Whether or not to search for whole word matches only.
         */
        isWordMatch?: boolean;
    }

    export namespace workspace {
        /**
         * Search text in files across all {@link workspace.workspaceFolders workspace folders} in the workspace.
         * @param query The query parameters for the search - the search string, whether it's case-sensitive, or a regex, or matches whole words.
         * @param callback A callback, called for each result
         * @param token A token that can be used to signal cancellation to the underlying search engine.
         * @return A thenable that resolves when the search is complete.
         */
        export function findTextInFiles(
            query: TextSearchQuery,
            callback: (result: TextSearchResult) => void,
            token?: CancellationToken
        ): Thenable<TextSearchComplete>;

        /**
         * Search text in files across all {@link workspace.workspaceFolders workspace folders} in the workspace.
         * @param query The query parameters for the search - the search string, whether it's case-sensitive, or a regex, or matches whole words.
         * @param options An optional set of query options. Include and exclude patterns, maxResults, etc.
         * @param callback A callback, called for each result
         * @param token A token that can be used to signal cancellation to the underlying search engine.
         * @return A thenable that resolves when the search is complete.
         */
        export function findTextInFiles(
            query: TextSearchQuery,
            options: FindTextInFilesOptions,
            callback: (result: TextSearchResult) => void,
            token?: CancellationToken
        ): Thenable<TextSearchComplete>;
    }
}
```

**What this API adds:**

The `findTextInFiles` API provides workspace-wide text search with a callback-based result streaming model. As matches are found, your callback receives each result immediately - you don't have to wait for the entire search to complete. This is efficient for large codebases where you want to show results progressively. The `TextSearchResult` can be either a `TextSearchMatch` (actual match with preview) or `TextSearchContext` (surrounding context lines).

### How Puku Editor Uses It

**Main Implementation:** `src/chat/src/platform/search/vscode/baseSearchServiceImpl.ts`

```typescript
// Base implementation - direct passthrough
async findTextInFiles(
    query: vscode.TextSearchQuery,
    options: vscode.FindTextInFilesOptions,
    progress: vscode.Progress<vscode.TextSearchResult>,
    token: vscode.CancellationToken
): Promise<vscode.TextSearchComplete> {
    return await vscode.workspace.findTextInFiles(
        query,
        options,
        result => progress.report(result),
        token
    );
}
```

This base implementation provides a thin wrapper around the VS Code API. It forwards the search query and options directly to `findTextInFiles`, and converts each result into a progress report. The callback-to-progress adapter (`result => progress.report(result)`) allows the service to integrate with VS Code's standard progress reporting system used throughout the editor.

**Extended with Ignore Support:** `src/chat/src/platform/search/vscode-node/searchServiceImpl.ts`

```typescript
// Filters results through Copilot ignore service
override async findTextInFiles(
    query: vscode.TextSearchQuery,
    options: vscode.FindTextInFilesOptions,
    progress: vscode.Progress<vscode.TextSearchResult>,
    token: vscode.CancellationToken
): Promise<vscode.TextSearchComplete> {
    const jobs: Promise<void>[] = [];
    const ignoreSupportedProgress: vscode.Progress<vscode.TextSearchResult> = {
        report: async (value) => {
            jobs.push((async () => {
                // Skip results from ignored files
                if (await this._ignoreService.isPukuIgnored(value.uri)) {
                    return;
                }
                progress.report(value);
            })());
        }
    };
    const result = await super.findTextInFiles(query, options, ignoreSupportedProgress, token);
    await Promise.all(jobs);
    return result;
}
```

This extended implementation adds Puku-specific filtering on top of the base search. It intercepts each streaming result and checks whether the file should be ignored using Puku's ignore service. Results from ignored files are silently dropped, while valid results pass through to the original progress handler. The `Promise.all(jobs)` ensures all async ignore checks complete before the search is considered finished.

### Result Types

![](./images/image-8.svg)

The diagram shows the two types of results you receive from text search. A `TextSearchMatch` represents an actual match - it includes the file URI, the exact character ranges where the match was found, and a preview of the matching text. A `TextSearchContext` represents surrounding lines when you use `beforeContext` or `afterContext` options - it gives you the file URI, the full line text, and the line number, helping users understand the match in context.

## API 3: findTextInFiles2

### What Problem Does It Solve?

The original `findTextInFiles` uses callbacks, which can be awkward to work with. `findTextInFiles2` modernizes the API by returning an `AsyncIterable` for results, making it easier to use with `for await...of` loops and async generators.

**Real-world scenario:** When building a streaming UI that shows search results as they arrive, `AsyncIterable` provides a cleaner programming model than callbacks.

#### Callback Pattern (findTextInFiles)

![](./images/image-9.svg)

With the callback pattern, you pass `query + options` to the search function along with a callback. As the search runs, it fires `callback(result)` for each match found (Result 1, Result 2, ... Result N). The function returns a `Promise` that resolves when the search completes. The problem: your callback fires unpredictably, making it hard to coordinate with other async operations or stop early.

#### AsyncIterable Pattern (findTextInFiles2)

![](./images/image-10.svg)

With the AsyncIterable pattern, you pass `query + options` and get back a `Response` object with two properties: `.result` is an `AsyncIterable` you can iterate with `for await`, and `.complete` is a `Promise` for completion status. This design gives you control - you can `break` out of the loop early, use `continue` to skip results, or `return` to exit. The code reads like synchronous iteration but handles async streaming.

### API Definition

**Definition File:** `vscode.proposed.findTextInFiles2.d.ts`

**Location in Puku:** `src/chat/src/extension/vscode.proposed.findTextInFiles2.d.ts`

```typescript
declare module 'vscode' {

    export interface FindTextInFilesOptions2 {
        /**
         * Array of include patterns (OR logic).
         */
        include?: GlobPattern[];

        /**
         * Array of exclude patterns (AND logic).
         */
        exclude?: GlobPattern[];

        /**
         * Which exclude settings to follow.
         */
        useExcludeSettings?: ExcludeSettingOptions;

        /**
         * Maximum results. Defaults to 20000.
         */
        maxResults?: number;

        /**
         * Ignore file settings.
         */
        useIgnoreFiles?: {
            local?: boolean;
            parent?: boolean;
            global?: boolean;
        };

        /**
         * Follow symlinks.
         */
        followSymlinks?: boolean;

        /**
         * File encoding.
         */
        encoding?: string;

        /**
         * Preview options.
         */
        previewOptions?: {
            numMatchLines?: number;
            charsPerLine?: number;
        };

        /**
         * Lines of context around matches.
         */
        surroundingContext?: number;
    }

    export interface FindTextInFilesResponse {
        /**
         * AsyncIterable of search results.
         */
        results: AsyncIterable<TextSearchResult2>;

        /**
         * Promise that resolves when search completes.
         */
        complete: Thenable<TextSearchComplete2>;
    }

    export namespace workspace {
        /**
         * Search text in files with AsyncIterable results.
         */
        export function findTextInFiles2(
            query: TextSearchQuery2,
            options?: FindTextInFilesOptions2,
            token?: CancellationToken
        ): FindTextInFilesResponse;
    }
}
```

**What this API adds:**

The `findTextInFiles2` API improves on the original in several ways. First, it returns a `FindTextInFilesResponse` object with both an `AsyncIterable<TextSearchResult2>` for streaming results and a `complete` promise for when the search finishes. Second, like `findFiles2`, it accepts arrays of include/exclude patterns. Third, it uses the updated `TextSearchResult2` type which includes `TextSearchMatch2` and `TextSearchContext2` classes with cleaner APIs.

### How Puku Editor Uses It

**Implementation:** `src/chat/src/platform/search/vscode/baseSearchServiceImpl.ts`

```typescript
findTextInFiles2(
    query: vscode.TextSearchQuery2,
    options?: vscode.FindTextInFilesOptions2,
    token?: vscode.CancellationToken
): vscode.FindTextInFilesResponse {
    return vscode.workspace.findTextInFiles2(query, options, token);
}
```

This implementation provides a direct passthrough to the VS Code API. Unlike the callback-based `findTextInFiles`, there's no need for a progress adapter - the `AsyncIterable` in the response handles streaming naturally. Callers can iterate over results using `for await...of` loops, making the code cleaner and easier to reason about.

### Key Differences from findTextInFiles

The main difference is how results are delivered. The original `findTextInFiles` uses a callback function that fires for each result, while `findTextInFiles2` returns an `AsyncIterable` that you can iterate with `for await...of`. This makes the code more readable and easier to integrate with modern async/await patterns.

Pattern handling is also improved. While `findTextInFiles` accepts a single `GlobPattern` for include and exclude, `findTextInFiles2` accepts arrays of patterns. This means you can search multiple file types (like `['**/*.ts', '**/*.js']`) without needing separate searches or complex glob syntax.

Context lines work differently too. The original API has separate `beforeContext` and `afterContext` options, while `findTextInFiles2` uses a single `surroundingContext` value that applies to both before and after. The result types are also updated - `TextSearchMatch2` and `TextSearchContext2` have cleaner APIs than their predecessors.

## API 4: textSearchProvider

### What Problem Does It Solve?

VS Code's built-in search uses ripgrep for lightning-fast text searching on local files. However, ripgrep only works with the local file system - it can't search files on remote servers, cloud storage, or virtual file systems. The `textSearchProvider` API solves this by letting you register a custom search implementation for any URI scheme.

When you register a provider for a scheme like `ssh://` or `cloud://`, VS Code delegates all search operations for that scheme to your provider. Your provider receives the search query (pattern, case sensitivity, regex options), folder scope, include/exclude patterns, and a progress callback to stream results back. This enables seamless search integration regardless of where the files actually live.

**Real-world scenarios:**
- **Remote development** - Search files on an SSH server without downloading them locally
- **Cloud storage** - Query a cloud API (GitHub, S3, Azure Blob) for text matches
- **Virtual file systems** - Search generated or computed content that doesn't exist on disk
- **Database-backed content** - Search text stored in databases as if they were files

![](./images/image-11.svg)

The diagram shows how VS Code routes search requests based on URI scheme. When you register a provider for a scheme like `ssh://` or `cloud://`, VS Code calls your provider instead of the built-in ripgrep whenever files with that scheme need to be searched. The `file://` scheme uses the built-in provider by default, but you can override it with your own implementation.

### API Definition

**Definition File:** `vscode.proposed.textSearchProvider.d.ts`

**Location in Puku:** `src/chat/src/extension/vscode.proposed.textSearchProvider.d.ts`

```typescript
declare module 'vscode' {

    // https://github.com/microsoft/vscode/issues/59921

    /**
     * The parameters of a query for text search.
     */
    export interface TextSearchQuery {
        pattern: string;
        isMultiline?: boolean;
        isRegExp?: boolean;
        isCaseSensitive?: boolean;
        isWordMatch?: boolean;
    }

    /**
     * Options common to file and text search.
     */
    export interface SearchOptions {
        folder: Uri;
        includes: GlobString[];
        excludes: GlobString[];
        useIgnoreFiles: boolean;
        followSymlinks: boolean;
        useGlobalIgnoreFiles: boolean;
        useParentIgnoreFiles: boolean;
    }

    /**
     * Options to specify the size of the result text preview.
     */
    export interface TextSearchPreviewOptions {
        matchLines: number;
        charsPerLine: number;
    }

    /**
     * Options that apply to text search.
     */
    export interface TextSearchOptions extends SearchOptions {
        maxResults: number;
        previewOptions?: TextSearchPreviewOptions;
        maxFileSize?: number;
        encoding?: string;
        beforeContext?: number;
        afterContext?: number;
    }

    /**
     * Information collected when text search is complete.
     */
    export interface TextSearchComplete {
        limitHit?: boolean;
        message?: TextSearchCompleteMessage | TextSearchCompleteMessage[];
    }

    /**
     * A preview of the text result.
     */
    export interface TextSearchMatchPreview {
        text: string;
        matches: Range | Range[];
    }

    /**
     * A match from a text search.
     */
    export interface TextSearchMatch {
        uri: Uri;
        ranges: Range | Range[];
        preview: TextSearchMatchPreview;
    }

    /**
     * A line of context surrounding a TextSearchMatch.
     */
    export interface TextSearchContext {
        uri: Uri;
        text: string;
        lineNumber: number;
    }

    export type TextSearchResult = TextSearchMatch | TextSearchContext;

    /**
     * A TextSearchProvider provides search results for text results inside files in the workspace.
     */
    export interface TextSearchProvider {
        /**
         * Provide results that match the given text pattern.
         * @param query The parameters for this query.
         * @param options A set of options to consider while searching.
         * @param progress A progress callback that must be invoked for all results.
         * @param token A cancellation token.
         */
        provideTextSearchResults(
            query: TextSearchQuery,
            options: TextSearchOptions,
            progress: Progress<TextSearchResult>,
            token: CancellationToken
        ): ProviderResult<TextSearchComplete>;
    }

    export namespace workspace {
        /**
         * Register a text search provider.
         * Only one provider can be registered per scheme.
         * @param scheme The provider will be invoked for workspace folders that have this file scheme.
         * @param provider The provider.
         * @return A {@link Disposable} that unregisters this provider when being disposed.
         */
        export function registerTextSearchProvider(
            scheme: string,
            provider: TextSearchProvider
        ): Disposable;
    }
}
```

**What this API adds:**

The `textSearchProvider` API lets you completely replace VS Code's search behavior for a specific URI scheme. When VS Code needs to search files with your scheme (e.g., `myscheme://`), it calls your provider's `provideTextSearchResults` method instead of using ripgrep. You receive the search query, options (folder, includes, excludes, etc.), and a progress reporter to stream results back.

## API 5: textSearchProvider2

### What Problem Does It Solve?

The original `textSearchProvider` API has limitations when working with multi-root workspaces. It receives a single folder in the options, so providers must be called multiple times for multi-folder searches. The `textSearchProvider2` API modernizes this with a `folderOptions` array that contains all folders in a single call, allowing more efficient batch operations.

Additionally, the original API uses older types (`TextSearchMatch`, `TextSearchContext`) that require manual object construction. The updated API uses `TextSearchMatch2` and `TextSearchContext2` classes with constructors, making it easier to create properly structured results. The types also align with `findTextInFiles2`, so the same result objects work across both searching and providing.

**Key improvements over textSearchProvider:**
- **Multi-folder support** - `folderOptions` array instead of single folder, enabling batch operations
- **Per-folder ignore settings** - Each folder can have its own `useIgnoreFiles` configuration
- **Class-based results** - `TextSearchMatch2` and `TextSearchContext2` are classes with constructors
- **Unified context** - Single `surroundingContext` replaces separate before/after settings

### API Definition

**Definition File:** `vscode.proposed.textSearchProvider2.d.ts`

**Location in Puku:** `src/chat/src/extension/vscode.proposed.textSearchProvider2.d.ts`

```typescript
declare module 'vscode' {

    export interface TextSearchProviderOptions {
        folderOptions: {
            folder: Uri;
            includes: string[];
            excludes: GlobPattern[];
            followSymlinks: boolean;
            useIgnoreFiles: {
                local: boolean;
                parent: boolean;
                global: boolean;
            };
            encoding: string;
        }[];
        maxResults: number;
        previewOptions: {
            matchLines: number;
            charsPerLine: number;
        };
        maxFileSize: number | undefined;
        surroundingContext: number;
    }

    export class TextSearchMatch2 {
        constructor(uri: Uri, ranges: { sourceRange: Range; previewRange: Range }[], previewText: string);
        uri: Uri;
        ranges: { sourceRange: Range; previewRange: Range }[];
        previewText: string;
    }

    export class TextSearchContext2 {
        constructor(uri: Uri, text: string, lineNumber: number);
        uri: Uri;
        text: string;
        lineNumber: number;
    }

    /**
     * Keyword suggestion for AI search.
     */
    export class AISearchKeyword {
        constructor(keyword: string);
        keyword: string;
    }

    /**
     * A result payload for a text search.
     */
    export type TextSearchResult2 = TextSearchMatch2 | TextSearchContext2;

    /**
     * A result payload for an AI search.
     * Can be a match or a keyword suggestion.
     */
    export type AISearchResult = TextSearchResult2 | AISearchKeyword;

    /**
     * A TextSearchProvider provides search results for text results inside files in the workspace.
     */
    export interface TextSearchProvider2 {
        /**
         * Provide results that match the given text pattern.
         * @param query The parameters for this query.
         * @param options A set of options to consider while searching.
         * @param progress A progress callback that must be invoked for all results.
         * @param token A cancellation token.
         */
        provideTextSearchResults(
            query: TextSearchQuery2,
            options: TextSearchProviderOptions,
            progress: Progress<TextSearchResult2>,
            token: CancellationToken
        ): ProviderResult<TextSearchComplete2>;
    }

    export namespace workspace {
        /**
         * Register a text search provider.
         * Only one provider can be registered per scheme.
         */
        export function registertextSearchProvider2(
            scheme: string,
            provider: TextSearchProvider2
        ): Disposable;
    }
}
```

**What this API adds:**

The main improvements in `textSearchProvider2` are: multi-folder support (options contains `folderOptions` array instead of single folder), structured ignore file settings per folder, `TextSearchMatch2` and `TextSearchContext2` are now classes with constructors (easier to create), and `surroundingContext` replaces separate before/after context settings.

### How Puku Editor Uses It

**API Registration:** [package.json:128](src/chat/package.json#L128)

```json
"enabledApiProposals": [
    "textSearchProvider2",
    // ... other proposals
]
```

The API is enabled in Puku's extension manifest. While Puku doesn't register a `TextSearchProvider2` provider directly, it uses the types from this API (`TextSearchMatch2`, `TextSearchContext2`) in its semantic search implementation.

**Type Usage:** [semanticSearchTextSearchProvider.ts](src/chat/src/extension/workspaceSemanticSearch/node/semanticSearchTextSearchProvider.ts)

```typescript
// Creating search results using TextSearchMatch2 class
const match: vscode.TextSearchMatch2 = new TextSearchMatch2(
    result.uri,
    ranges,
    result.preview.text
);
progress.report(match);
```

This code creates `TextSearchMatch2` instances to report semantic search results. The class constructor takes the file URI, an array of range pairs (source range and preview range), and the preview text. Using the class constructor ensures proper object structure instead of manually creating objects with the correct shape.

```typescript
// Creating matches from code chunks
const match: vscode.TextSearchMatch2 = new TextSearchMatch2(
    chunk.file,
    [{
        sourceRange: new VSCodeRange(
            chunk.range.startLineNumber,
            chunk.range.startColumn,
            chunk.range.endLineNumber,
            chunk.range.endColumn
        ),
        previewRange: this.getPreviewRange(rangeText, symbolsToHighlight),
    }],
    rangeText
);
progress.report(match);
```

When processing file chunks from the workspace chunk search, Puku creates `TextSearchMatch2` objects with the chunk's location as the source range and computed preview ranges that highlight relevant symbols. This allows semantic search results to integrate seamlessly with VS Code's search UI.

## API 6: aiTextSearchProvider

### What Problem Does It Solve?

Traditional text search is purely literal - searching for "error handling" only finds files containing that exact phrase. But developers often think in concepts, not exact text. When searching for "error handling", you might want to find try-catch blocks, error classes, logging calls, and validation code - even if none of them contain the words "error handling".

The `aiTextSearchProvider` API solves this by enabling semantic search that understands meaning, not just characters. Unlike `textSearchProvider` which receives a structured query (pattern, regex flags, case sensitivity), the AI provider receives the raw search string - allowing natural language queries like "how does authentication work" or "where are API endpoints defined".

**Key capabilities:**
- **Natural language queries** - Users can search using concepts and questions, not just code patterns
- **Semantic matching** - Uses embeddings/vectors to find conceptually related code
- **Keyword suggestions** - Can report `AISearchKeyword` objects to help users refine their search
- **Separate results section** - AI results appear in their own section in VS Code's Search panel, labeled with your provider's `name`

**Real-world scenarios:**
- Search "error handling" → finds try-catch blocks, error classes, logger.error() calls
- Search "user authentication" → finds login functions, JWT validation, session management
- Search "database queries" → finds ORM models, SQL builders, connection pools

![](./images/image-12.svg)

The diagram shows the API structure. First, you register your provider with a URI scheme using `registerAITextSearchProvider`. When a user searches, VS Code calls `provideAITextSearchResults` with the raw query string, folder options, a progress reporter, and cancellation token. Your provider processes the query and reports results via the progress callback - either `TextSearchMatch2` for code matches, `TextSearchContext2` for context lines, or `AISearchKeyword` for search suggestions. Results appear in a separate section labeled with your provider's `name`.

### API Definition

**Definition File:** `vscode.proposed.aiTextSearchProvider.d.ts`

**Location in Puku:** `src/chat/src/extension/vscode.proposed.aiTextSearchProvider.d.ts`

```typescript
// vscode.proposed.aiTextSearchProvider.d.ts
// version: 2

declare module 'vscode' {
    /**
     * An AITextSearchProvider provides additional AI text search results in the workspace.
     */
    export interface AITextSearchProvider {
        /**
         * The name of the AI searcher. Will be displayed as `{name} Results` in the Search View.
         */
        readonly name?: string;

        /**
         * Provide results that match the given text pattern.
         * @param query The parameter for this query.
         * @param options A set of options to consider while searching.
         * @param progress A progress callback that must be invoked for all results.
         * @param token A cancellation token.
         */
        provideAITextSearchResults(
            query: string,
            options: TextSearchProviderOptions,
            progress: Progress<TextSearchResult2>,
            token: CancellationToken
        ): ProviderResult<TextSearchComplete2>;
    }

    export namespace workspace {
        /**
         * Register an AI text search provider.
         * Only one provider can be registered per scheme.
         * @param scheme The provider will be invoked for workspace folders that have this file scheme.
         * @param provider The provider.
         * @return A {@link Disposable} that unregisters this provider when being disposed.
         */
        export function registerAITextSearchProvider(
            scheme: string,
            provider: AITextSearchProvider
        ): Disposable;
    }
}

// Note: AISearchKeyword and AISearchResult are defined in textSearchProvider2.d.ts
// export class AISearchKeyword { constructor(keyword: string); keyword: string; }
// export type AISearchResult = TextSearchResult2 | AISearchKeyword;
```

**What this API adds:**

The `aiTextSearchProvider` API enables semantic search in VS Code's Search panel. Unlike `textSearchProvider` which receives a structured `TextSearchQuery`, the AI provider receives a raw `string` query - allowing natural language searches. It can also report `AISearchKeyword` suggestions alongside matches, helping users refine their search. The results appear in a separate "AI Results" section in the Search panel.

### How Puku Editor Uses It

**Main Implementation:** `src/chat/src/extension/workspaceSemanticSearch/node/semanticSearchTextSearchProvider.ts`

Puku's semantic search is a sophisticated multi-stage pipeline that combines embeddings, LLM ranking, and AST analysis to provide intelligent code search. The implementation follows these steps:

1. **Input Processing** - Extract include/exclude patterns from folder options
2. **Semantic Chunk Search** - Generate embeddings and find ~50 related code chunks via vector similarity
3. **Keyword Extraction** - Use Tree-sitter AST to extract symbols, then LLM selects best keywords as `AISearchKeyword` suggestions
4. **LLM Ranking** - Send chunks to LLM which returns top ~10 file/query pairs with specific search terms
5. **Report Results** - Two paths: LLM-ranked results use `findTextInFiles` for exact positions; remaining chunks are reported directly with chunk ranges

#### Step 1: Extract Search Scope from Options

```typescript
provideAITextSearchResults(query: string, options: vscode.TextSearchProviderOptions,
    progress: vscode.Progress<vscode.TextSearchResult2>, token: vscode.CancellationToken) {

    const includes = new Set<vscode.GlobPattern>();
    const excludes = new Set<vscode.GlobPattern>();

    // Extract include/exclude patterns from each folder
    for (const folder of options.folderOptions) {
        folder.includes?.forEach(e => {
            includes.add(e.startsWith('*') ? e : new RelativePattern(folder.folder, e));
        });
        folder.excludes?.forEach(e => {
            excludes.add(typeof e === 'string' && !e.startsWith('*')
                ? new RelativePattern(folder.folder, e) : e);
        });
    }
}
```

The method first processes the `TextSearchProviderOptions` to extract include and exclude patterns from each workspace folder. Patterns are converted to `RelativePattern` objects when they're folder-specific.

#### Step 2: Semantic Chunk Search

```typescript
const result = await this.workspaceChunkSearch.searchFileChunks(
    {
        endpoint: await this.getEndpoint(),
        tokenBudget: MAX_CHUNK_TOKEN_COUNT,      // Token limit for results
        fullWorkspaceTokenBudget: MAX_CHUNK_TOKEN_COUNT,
        maxResults: MAX_CHUNKS_RESULTS,          // Max chunks to return
    },
    {
        rawQuery: query,
        resolveQueryAndKeywords: async () => ({
            rephrasedQuery: query,
            keywords: this.getKeywordsForContent(query),  // Extract identifiers
        }),
    },
    { globPatterns: { include, exclude } },
    token
);
```

The workspace chunk search service uses embeddings and vector similarity to find code chunks semantically related to the query. The `getKeywordsForContent` method extracts identifiers from the query using regex to help with keyword matching.

#### Step 3: Extract AI Keywords with Tree-sitter

```typescript
async treeSitterAIKeywords(query: string, progress: vscode.Progress<vscode.AISearchResult>,
    chunks: FileChunk[], token: vscode.CancellationToken) {

    const symbols = new Set<string>();
    for (const chunk of chunks) {
        const doc = await this.workspaceService.openTextDocumentAndSnapshot(chunk.file);
        const ast = this._parserService.getTreeSitterAST({
            languageId: doc.languageId,
            getText: () => doc.getText()
        });
        const symbolsInChunk = await ast?.getSymbols({
            startIndex: doc.offsetAt(new Position(chunk.range.startLineNumber, chunk.range.startColumn)),
            endIndex: doc.offsetAt(new Position(chunk.range.endLineNumber, chunk.range.endColumn)),
        });
        symbolsInChunk?.forEach(symbol => symbols.add(symbol.text));
    }

    // Use LLM to select best keywords from extracted symbols
    const keywordResult = await this.llmSelectKeywords(symbols, query);
    keywordResult.forEach(keyword => {
        progress.report(new AISearchKeyword(keyword));
    });
}
```

For each matched chunk, Puku parses the code using Tree-sitter to extract symbol names (functions, classes, variables). These symbols are sent to an LLM which selects the most relevant ones as keyword suggestions, reported via `AISearchKeyword`.

#### Step 4: LLM Ranking

```typescript
// Build prompt with chunks and query
const intent = this._intentService.getIntent('searchPanel', ChatLocation.Other);
const intentInvocation = await intent.invoke({ location: ChatLocation.Other, request });
const prompt = await intentInvocation.buildPrompt({ query, chunkResults }, progress, token);

// Call LLM to rank results
const fetchResult = await intentInvocation.endpoint.makeChatRequest(
    'searchPanel',
    prompt.messages,
    async (text, _, delta) => undefined,
    token,
    ChatLocation.Other,
    undefined,
    { temperature: 0.1 },  // Low temperature for consistent ranking
);

// Parse LLM response as JSON ranking
const rankingResults: IRankResult[] = JSON.parse(
    fetchResult.value.replace(/```(?:json)?/g, '').trim()
);
```

The chunks are sent to an LLM via the intent service with a specialized "searchPanel" prompt. The LLM returns a JSON array ranking the most relevant files with specific search queries for each.

#### Step 5: Report Results

```typescript
async reportSearchResults(rankingResults: IRankResult[], combinedChunks: FileChunk[],
    progress: vscode.Progress<vscode.AISearchResult>, token: vscode.CancellationToken) {

    // For LLM-ranked results: use findTextInFiles for exact positions
    await Promise.all(rankingResults.map(result =>
        this.searchService.findTextInFiles(
            { pattern: result.query, isRegExp: false },
            { useDefaultExcludes: true, maxResults: 20, include: result.file },
            onResult,  // Converts to TextSearchMatch2
            token
        )
    ));

    // For remaining chunks: report directly with AST-highlighted previews
    for (const chunk of combinedChunks.slice(rankingResults.length)) {
        const doc = await this.workspaceService.openTextDocumentAndSnapshot(chunk.file);
        const ast = this._parserService.getTreeSitterAST({ languageId: doc.languageId, getText: () => doc.getText() });
        const symbolsToHighlight = await ast?.getSymbols({ startIndex, endIndex });

        const match = new TextSearchMatch2(
            chunk.file,
            [{ sourceRange: chunkRange, previewRange: this.getPreviewRange(rangeText, symbolsToHighlight) }],
            rangeText
        );
        progress.report(match);
    }
}
```

Results are reported in two ways. For LLM-ranked results, `findTextInFiles` is called to get exact match positions in each file. For remaining chunks, results are reported directly with preview ranges computed from Tree-sitter symbol analysis to highlight the most relevant identifiers.

### Code Example: Implementing AI Search

```typescript
import * as vscode from 'vscode';

class MyAISearchProvider implements vscode.AITextSearchProvider {
    readonly name = 'My AI Search';

    constructor(
        private embeddingService: EmbeddingService,
        private vectorStore: VectorStore
    ) {}

    async provideAITextSearchResults(
        query: string,
        options: vscode.TextSearchProviderOptions,
        progress: vscode.Progress<vscode.AISearchResult>,
        token: vscode.CancellationToken
    ): Promise<vscode.TextSearchComplete2> {
        // 1. Generate embedding for the query
        const queryEmbedding = await this.embeddingService.embed(query);

        // 2. Search vector store for similar code chunks
        const results = await this.vectorStore.search(queryEmbedding, {
            limit: options.maxResults,
            folders: options.folderOptions.map(f => f.folder)
        });

        // 3. Report keyword suggestions
        const keywords = this.extractKeywords(query);
        for (const keyword of keywords) {
            progress.report(new vscode.AISearchKeyword(keyword));
        }

        // 4. Report matching code chunks
        for (const result of results) {
            if (token.isCancellationRequested) break;

            const match = new vscode.TextSearchMatch2(
                result.uri,
                [{
                    sourceRange: result.range,
                    previewRange: new vscode.Range(0, 0, 0, result.preview.length)
                }],
                result.preview
            );
            progress.report(match);
        }

        return { limitHit: results.length >= options.maxResults };
    }

    private extractKeywords(query: string): string[] {
        // Extract important terms from natural language query
        return query.split(/\s+/).filter(word => word.length > 3);
    }
}

// Register the provider
export function activate(context: vscode.ExtensionContext) {
    const provider = new MyAISearchProvider(embeddingService, vectorStore);
    context.subscriptions.push(
        vscode.workspace.registerAITextSearchProvider('file', provider)
    );
}
```

This example shows the basic structure of an AI search provider. The `provideAITextSearchResults` method receives the raw query string (not a structured query object), allowing natural language processing. It generates an embedding for the query, searches a vector store for similar code chunks, reports keyword suggestions using `AISearchKeyword`, and finally reports the matching code as `TextSearchMatch2` objects. The provider is registered using `registerAITextSearchProvider` with the `file` scheme.

### Key Differences from textSearchProvider

| Aspect | textSearchProvider | aiTextSearchProvider |
|--------|-------------------|---------------------|
| **Query type** | `TextSearchQuery` (structured) | `string` (raw text) |
| **Query options** | Pattern, regex, case, word match | Natural language |
| **Result types** | `TextSearchResult` | `AISearchResult` (includes keywords) |
| **Results location** | Main search results | Separate "AI Results" section |
| **Use case** | Literal text search | Semantic/conceptual search |


## Document Information

**Last Updated:** February 2026

**Puku Editor Version:** 0.43.34

**VS Code Version:** 1.107.0
