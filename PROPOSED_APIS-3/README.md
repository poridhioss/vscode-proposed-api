# VS Code Proposed APIs Part-3

Terminal APIs provide powerful capabilities for interacting with VS Code's integrated terminal. These APIs are essential for AI-powered code editors that need to understand terminal context, capture command output, and provide intelligent command suggestions. This guide covers the 4 terminal-related APIs in depth, showing how Puku uses them and how to modify them.

![](./images/image-1.svg)

The diagram provides an overview of the four terminal APIs. `terminalSelection` reads text the user has highlighted. `terminalDataWriteEvent` streams raw output data as it appears. `terminalExecuteCommandEvent` provides structured information when commands complete (requires shell integration). `terminalQuickFixProvider` lets you suggest fixes when commands fail.

## Quick Reference Table

| API | Purpose | Puku Usage |
|-----|---------|------------|
| `terminalSelection` | Read text selected by user in terminal | Context gathering for AI |
| `terminalDataWriteEvent` | Stream raw terminal output data | Buffer terminal output for context |
| `terminalExecuteCommandEvent` | Receive notifications when commands complete | Track command history and results |
| `terminalQuickFixProvider` | Register provider for terminal error fixes | AI-powered command suggestions |

## File Locations in Puku Editor

```
src/chat/
├── package.json                                    ← API declarations
├── src/extension/
│   └── vscode.proposed.terminal*.d.ts             ← Copied type definitions
└── src/platform/terminal/
    ├── common/terminalService.ts                  ← Terminal interface
    └── vscode/
        ├── terminalBufferListener.ts              ← Buffer management
        └── terminalServiceImpl.ts                 ← Implementation
```

## API 1: terminalSelection

### What Problem Does It Solve?

How do you know what text the user has selected in the terminal? The stable API doesn't expose this information at all. The `terminalSelection` API adds a simple `selection` property to the `Terminal` interface, letting you read whatever text the user has highlighted.

**Real-world scenario:** When a user selects an error message in the terminal and asks the AI "What does this mean?", Puku Editor uses `terminalSelection` to capture the exact text they've selected, providing precise context for the AI response.

![](./images/image-2.svg)

The diagram shows how the API works. Access the active terminal via `window.activeTerminal`, then read the `selection` property. It returns either a `string` containing the highlighted text (like an error message) or `undefined` if nothing is selected. This simple read-only property enables features like "explain selected error" or "fix this command".

### API Definition

**Definition File:** `vscode.proposed.terminalSelection.d.ts`

**Location in Puku:** `src/chat/src/extension/vscode.proposed.terminalSelection.d.ts`

```typescript
declare module 'vscode' {

    // https://github.com/microsoft/vscode/issues/188173

    export interface Terminal {
        /**
         * The selected text of the terminal or undefined if there is no selection.
         */
        readonly selection: string | undefined;
    }
}
```

**What this API adds:**

The `terminalSelection` API is beautifully simple - it adds just one property to the existing `Terminal` interface. The `selection` property returns the text the user has highlighted in the terminal, or `undefined` if nothing is selected. This read-only property updates automatically as the user changes their selection.

### How Puku Editor Uses It

**Main Implementation:** `src/chat/src/platform/terminal/vscode/terminalBufferListener.ts`

```typescript
export function getActiveTerminalSelection(): string {
    try {
        return window.activeTerminal?.selection ?? '';
    } catch {
        // In case the API isn't available
        return '';
    }
}
```

This helper function provides safe access to the terminal selection. It first checks if there's an active terminal using `window.activeTerminal`, then reads the `selection` property. The nullish coalescing operator (`??`) returns an empty string if selection is undefined.

The try-catch wrapper ensures graceful degradation - since this is a proposed API, it might not be available in all environments. If the API isn't available, the function returns an empty string instead of crashing the extension.

### Code Example: Using Terminal Selection

```typescript
import * as vscode from 'vscode';

// Command to explain selected terminal text
vscode.commands.registerCommand('myext.explainTerminalSelection', async () => {
    const terminal = vscode.window.activeTerminal;

    if (!terminal) {
        vscode.window.showErrorMessage('No active terminal');
        return;
    }

    const selection = terminal.selection;

    if (!selection) {
        vscode.window.showInformationMessage('No text selected in terminal');
        return;
    }

    // Use the selection for AI context
    console.log(`User selected: ${selection}`);

    // Example: Send to chat participant
    vscode.commands.executeCommand('workbench.action.chat.open', {
        query: `@terminal Explain this error: ${selection}`
    });
});
```

## API 2: terminalDataWriteEvent

### What Problem Does It Solve?

The stable VS Code Terminal API has a fundamental asymmetry: you can **write** to terminals using `Terminal.sendText()`, but you cannot **read** what comes out. When a command runs, its output appears on screen but extensions have no way to access it. The `terminalDataWriteEvent` API bridges this gap by firing an event whenever data is written to any terminal, giving you access to the raw output stream.

**Key limitations of stable API:**
- `Terminal.sendText()` - Can send input
- Reading terminal output - Not available
- Accessing command results - Not available

**Real-world scenario:** When a user asks "Why did my build fail?", the AI needs to see the actual error messages. Puku Editor uses `terminalDataWriteEvent` to maintain a rolling buffer of recent terminal output, capturing error messages, stack traces, and command results that provide essential context for AI assistance.

![](./images/image-3.svg)

The diagram shows how to use the API. First, subscribe to `window.onDidWriteTerminalData` with a callback function. When any terminal receives output, your callback receives a `TerminalDataWriteEvent` with two properties:
- **`terminal`** - The Terminal instance that received the data (useful when multiple terminals are open)
- **`data`** - The raw string exactly as written, which may include plain text (`'Hello World\n'`), ANSI escape codes for colors (`'\x1b[32mSuccess\x1b[0m'`), or error messages (`'npm ERR! code ENOENT'`)

![](./images/image-4.svg)

The diagram shows the complete data flow from shell to Puku's buffer:

1. **Shell Process** - When you run a command like `npm test`, the shell writes output to stdout/stderr
2. **Pseudo-Terminal (PTY)** - The PTY acts as an intermediary between the shell and VS Code. The shell writes to the slave side, and VS Code reads from the master side. This is how terminal emulators work on Unix-like systems.
3. **VS Code Terminal** - The master side of the PTY sends data to two places:
   - **Terminal UI Display** - What you see on screen
   - **onDidWriteTerminalData** - The event that extensions can listen to
4. **Puku Processing** - Puku listens to the event, strips ANSI codes, and stores clean text in a per-terminal buffer

The PTY is crucial because it's where VS Code intercepts the raw data stream. Without this hook at the PTY level, extensions would have no way to access terminal output.

### API Definition

**Definition File:** `vscode.proposed.terminalDataWriteEvent.d.ts`

**Location in Puku:** `src/chat/src/extension/vscode.proposed.terminalDataWriteEvent.d.ts`

```typescript
declare module 'vscode' {

    // https://github.com/microsoft/vscode/issues/78502
    //
    // This API is still proposed but we don't intent on promoting it to stable due to problems
    // around performance. See #145234 for a more likely API to get stabilized.

    export interface TerminalDataWriteEvent {
        /**
         * The {@link Terminal} for which the data was written.
         */
        readonly terminal: Terminal;
        /**
         * The data being written.
         */
        readonly data: string;
    }

    namespace window {
        /**
         * An event which fires when the terminal's child pseudo-device is written to (the shell).
         * In other words, this provides access to the raw data stream from the process running
         * within the terminal, including VT sequences.
         */
        export const onDidWriteTerminalData: Event<TerminalDataWriteEvent>;
    }
}
```

**What this API adds:**

The `terminalDataWriteEvent` API provides a global event that fires whenever any terminal receives output data. The event includes which terminal received the data and the raw string content including VT/ANSI escape sequences. Note the comment in the API - VS Code doesn't plan to stabilize this API due to performance concerns. They recommend using `terminalExecuteCommandEvent` (with shell integration) instead for most use cases.

### How Puku Editor Uses It

**Main Implementation:** `src/chat/src/platform/terminal/vscode/terminalBufferListener.ts`

```typescript
const terminalBuffers: Map<Terminal, string[]> = new Map();

function appendLimitedWindow<T>(target: T[], data: T) {
    target.push(data);
    if (target.length > 40) {
        target.shift();
    }
}
```

The buffer uses a sliding window approach. Each terminal gets its own buffer (Map key), and data is stored as an array of strings. The `appendLimitedWindow` function keeps only the last 40 data events - roughly twice the typical visible terminal area - preventing unbounded memory growth.

```typescript
export function installTerminalBufferListeners(): Disposable[] {
    return [
        window.onDidWriteTerminalData(e => {
            let dataBuffer = terminalBuffers.get(e.terminal);
            if (!dataBuffer) {
                dataBuffer = [];
                terminalBuffers.set(e.terminal, dataBuffer);
            }
            appendLimitedWindow(dataBuffer, removeAnsiEscapeCodes(e.data));
        }),
        window.onDidCloseTerminal(e => {
            terminalBuffers.delete(e);
        })
    ];
}
```

This function sets up two event listeners. The `onDidWriteTerminalData` handler stores each data chunk after stripping ANSI escape codes (color codes, cursor movements, etc.) for clean text. The `onDidCloseTerminal` handler cleans up the buffer when a terminal closes, preventing memory leaks.

```typescript
export function getBufferForTerminal(terminal?: Terminal, maxChars: number = 16000): string {
    if (!terminal) {
        return '';
    }
    const buffer = terminalBuffers.get(terminal);
    if (!buffer) {
        return '';
    }
    const joined = buffer.join('');
    const start = Math.max(0, joined.length - maxChars);
    return joined.slice(start);
}
```

The `getBufferForTerminal` function retrieves buffered content for a specific terminal. It joins all stored chunks and returns up to `maxChars` characters (default 16000) from the end, ensuring the most recent output is included when providing context to AI features.


### Performance Considerations

Commands that produce high-volume output like `cat large_file.txt` or continuous logging can fire thousands of data events per second, each triggering your event handler. This is why VS Code doesn't plan to stabilize this API - the performance impact can be severe. Extensions should employ mitigation strategies like limiting buffer size (Puku keeps only the last 40 chunks), throttling processing, or filtering relevant data. For most use cases, VS Code recommends using shell integration with `terminalExecuteCommandEvent` instead, which fires once per command completion rather than many times per output chunk.

## API 3: terminalExecuteCommandEvent

### What Problem Does It Solve?

While `terminalDataWriteEvent` gives you raw output streams, it doesn't tell you **when a command starts or ends**, **what the exit code was**, or **which output belongs to which command**. The `terminalExecuteCommandEvent` API solves this by firing once per command completion with structured information about the command execution.

**Key limitations of terminalDataWriteEvent:**
- Raw stream - No command boundaries
- No exit code - Can't tell if command succeeded
- No command association - Output mixed together
- Performance issues - Fires too frequently

**What terminalExecuteCommandEvent provides:**
- Fires **once** when command completes
- Includes **exit code** (0 = success)
- Provides **clean output** (no ANSI codes)
- Associates output with **specific command**

**Real-world scenario:** When the user runs `npm test` and it fails, Puku Editor captures the command, its exit code, and output. The AI can then provide context-aware help like "Your test failed because Jest couldn't find the module 'lodash'."

![](./images/image-5.svg)

The diagram shows the API structure. Subscribe to `window.onDidExecuteTerminalCommand` to receive a `TerminalExecutedCommand` object when any command completes. Unlike `terminalDataWriteEvent` which fires many times per command, this fires **exactly once** per command with all information bundled together:
- **`terminal`** - Which terminal ran the command
- **`commandLine`** - The full command string (e.g., `'npm test'`)
- **`cwd`** - Working directory where the command ran
- **`exitCode`** - Result code (0 = success, non-zero = failure)
- **`output`** - Clean text output without ANSI codes

![](./images/image-6.svg)

The diagram shows how data flows through multiple paths simultaneously. When the **Shell Process** executes a command like `npm test`, it produces output that needs to reach both the user's screen and any listening extensions.

**Shell Integration Scripts** are the key enabler for this API. These are special scripts that VS Code injects into supported shells (bash, zsh, fish, PowerShell) when shell integration is enabled. They emit invisible OSC 633 escape sequences at critical moments - when a command starts (`OSC 633;A`) and when it ends (`OSC 633;D;1` where 1 is the exit code). These markers are embedded directly in the terminal output stream but are invisible to users.

The **Pseudo-Terminal (PTY)** combines everything into a single stream: the start marker, the actual command output, and the end marker with exit code. This combined stream flows to VS Code's terminal implementation.

**VS Code Terminal** receives this stream and processes it in two ways simultaneously. The **Terminal Display** renders the visible output for the user, while the **OSC 633 Parser** scans for the invisible markers. When the parser detects an end marker, it knows a command has completed and can extract the exit code directly from the marker sequence.

The **Event** system then builds a complete `TerminalExecutedCommand` object containing the terminal reference, command line, working directory, exit code, and clean output (with ANSI codes already stripped). This object is passed to all registered listeners.

**Puku** listens to this event and stores each command in its per-terminal history buffer using the same `appendLimitedWindow()` function, keeping the last 40 commands available for AI context.

**What are OSC 633 sequences?** OSC stands for "Operating System Command" - a category of escape sequences used for terminal-to-application communication. The 633 identifier is VS Code's specific sequence for shell integration. When you press Enter to run a command, the shell emits `OSC 633 ; A` to mark the start. After the command finishes, it emits `OSC 633 ; D ; <exit_code>` to mark the end and report the result. Additional sequences like `OSC 633 ; B` (prompt started) and `OSC 633 ; C` (command line captured) provide even richer context.

**Key difference from API 2:** The fundamental distinction is event frequency and data quality. `terminalDataWriteEvent` fires on every PTY write - potentially hundreds of times per command - and provides raw chunks that may contain partial lines and ANSI codes. In contrast, `terminalExecuteCommandEvent` fires exactly once per command with a clean, structured object that includes the exit code. However, this API requires shell integration to work. If shell integration is disabled, unsupported, or the user is running a non-integrated shell, the event will never fire and you must fall back to `terminalDataWriteEvent`.

### API Definition

**Definition File:** `vscode.proposed.terminalExecuteCommandEvent.d.ts`

**Location in Puku:** `src/chat/src/extension/vscode.proposed.terminalExecuteCommandEvent.d.ts`

```typescript
declare module 'vscode' {

    // https://github.com/microsoft/vscode/issues/145234

    export interface TerminalExecutedCommand {
        /**
         * The {@link Terminal} the command was executed in.
         */
        terminal: Terminal;
        /**
         * The full command line that was executed, including both the command and the arguments.
         */
        commandLine: string | undefined;
        /**
         * The current working directory that was reported by the shell. This will be a {@link Uri}
         * if the string reported by the shell can reliably be mapped to the connected machine.
         */
        cwd: Uri | string | undefined;
        /**
         * The exit code reported by the shell.
         */
        exitCode: number | undefined;
        /**
         * The output of the command when it has finished executing. This is the plain text shown in
         * the terminal buffer and does not include raw escape sequences. Depending on the shell
         * setup, this may include the command line as part of the output.
         */
        output: string | undefined;
    }

    export namespace window {
        /**
         * An event that is emitted when a terminal with shell integration activated has completed
         * executing a command.
         *
         * Note that this event will not fire if the executed command exits the shell, listen to
         * {@link onDidCloseTerminal} to handle that case.
         *
         * @deprecated Use {@link window.onDidStartTerminalShellExecution}
         */
        export const onDidExecuteTerminalCommand: Event<TerminalExecutedCommand>;
    }
}
```

**What this API adds:**

The `terminalExecuteCommandEvent` API provides structured information about executed commands, unlike the raw stream from `terminalDataWriteEvent`. Key advantages:

- **Clean output** - No ANSI escape codes, just plain text
- **Exit code** - Know if the command succeeded or failed
- **Working directory** - Know where the command ran
- **Command line** - Know exactly what was executed

Note: This API is deprecated in favor of `onDidStartTerminalShellExecution`, but is still used in Puku.

### How Puku Editor Uses It

**Main Implementation:** `src/chat/src/platform/terminal/vscode/terminalBufferListener.ts`

```typescript
const terminalCommands: Map<Terminal, TerminalExecutedCommand[]> = new Map();

export function installTerminalBufferListeners(): Disposable[] {
    return [
        window.onDidExecuteTerminalCommand(e => {
            let commands = terminalCommands.get(e.terminal);
            if (!commands) {
                commands = [];
                terminalCommands.set(e.terminal, commands);
            }
            appendLimitedWindow(commands, e);
        }),
    ];
}
```

Similar to the data buffer, Puku maintains a command history per terminal. When a command completes, the full `TerminalExecutedCommand` object (including command line, exit code, output, and working directory) is stored using the same limited window approach to prevent unbounded growth.

```typescript
export function getLastCommandForTerminal(terminal: Terminal): TerminalExecutedCommand | undefined {
    return terminalCommands.get(terminal)?.at(-1);
}

export function getActiveTerminalLastCommand(): TerminalExecutedCommand | undefined {
    const activeTerminal = window.activeTerminal;
    if (activeTerminal === undefined) {
        return undefined;
    }
    return terminalCommands.get(activeTerminal)?.at(-1);
}
```

These helper functions retrieve the most recent command. `getLastCommandForTerminal` takes a specific terminal, while `getActiveTerminalLastCommand` automatically uses the currently active terminal. The `.at(-1)` syntax returns the last element of the array - the most recently executed command.

## API 4: terminalQuickFixProvider

### What Problem Does It Solve?

When a command fails in the terminal, users face a frustrating workflow: read the error, search for solutions, copy a fix command, paste it back into the terminal. The `terminalQuickFixProvider` API lets extensions short-circuit this process by suggesting fixes directly in the terminal UI.

**The manual workflow without quick fixes:**
1. Command fails with error message
2. User reads and interprets the error
3. User searches documentation or Stack Overflow
4. User figures out the correct command
5. User types or pastes the fix

**With terminalQuickFixProvider:**
1. Command fails → Quick fix popup appears instantly
2. User clicks suggested fix → Done

This API is particularly powerful when combined with AI. Instead of pattern-matching specific errors to specific fixes, Puku uses AI to analyze any error and generate contextually appropriate suggestions.

**Real-world scenario:** User runs `sudo apt updatex` (a typo). Puku detects the error and shows two quick fix options:
- **Fix using Puku** - AI analyzes the error and suggests the corrected command (`sudo apt update`)
- **Explain using Puku** - AI explains what went wrong ("updatex is not a recognized operation, it looks like a typo for update")

![](./images/image-7.svg)

The diagram shows Puku's quick fix flow. When a command like `sudo apt updatex` fails with an error, VS Code captures this in a `TerminalCommandMatchResult` object. Puku's quick fix provider receives this and offers two AI-powered options:

**Fix using Puku** analyzes the error and suggests a corrected command. In this case, it detects "updatex" is a typo and suggests `sudo apt update`. Selecting this option inserts the corrected command into the terminal.

**Explain using Puku** opens the Chat panel and provides an AI explanation of what went wrong. The AI identifies that "updatex is not a recognized operation for apt" and explains it looks like a typo for "update".

### API Definition

**Definition File:** `vscode.proposed.terminalQuickFixProvider.d.ts`

**Location in Puku:** `src/chat/src/extension/vscode.proposed.terminalQuickFixProvider.d.ts`

```typescript
declare module 'vscode' {

    // https://github.com/microsoft/vscode/issues/162950

    export type SingleOrMany<T> = T[] | T;

    export interface TerminalQuickFixProvider {
        /**
         * Provides terminal quick fixes
         * @param commandMatchResult The command match result for which to provide quick fixes
         * @param token A cancellation token indicating the result is no longer needed
         * @return Terminal quick fix(es) if any
         */
        provideTerminalQuickFixes(
            commandMatchResult: TerminalCommandMatchResult,
            token: CancellationToken
        ): ProviderResult<SingleOrMany<TerminalQuickFixTerminalCommand | TerminalQuickFixOpener | Command>>;
    }

    export interface TerminalCommandMatchResult {
        commandLine: string;
        commandLineMatch: RegExpMatchArray;
        outputMatch?: {
            regexMatch: RegExpMatchArray;
            outputLines: string[];
        };
    }

    export namespace window {
        /**
         * @param provider A terminal quick fix provider
         * @return A {@link Disposable} that unregisters the provider when being disposed
         */
        export function registerTerminalQuickFixProvider(
            id: string,
            provider: TerminalQuickFixProvider
        ): Disposable;
    }

    export class TerminalQuickFixTerminalCommand {
        /**
         * The terminal command to insert or run
         */
        terminalCommand: string;
        /**
         * Whether the command should be executed or just inserted (default)
         */
        shouldExecute?: boolean;
        constructor(terminalCommand: string, shouldExecute?: boolean);
    }

    export class TerminalQuickFixOpener {
        /**
         * The uri to open
         */
        uri: Uri;
        constructor(uri: Uri);
    }

    /**
     * A matcher that runs on a sub-section of a terminal command's output
     */
    interface TerminalOutputMatcher {
        /**
         * A string or regex to match against the unwrapped line. If this is a regex with the multiline
         * flag, it will scan an amount of lines equal to `\n` instances in the regex + 1.
         */
        lineMatcher: string | RegExp;
        /**
         * Which side of the output to anchor the {@link offset} and {@link length} against.
         */
        anchor: TerminalOutputAnchor;
        /**
         * The number of rows above or below the {@link anchor} to start matching against.
         */
        offset: number;
        /**
         * The number of wrapped lines to match against, this should be as small as possible for performance
         * reasons. This is capped at 40.
         */
        length: number;
    }

    enum TerminalOutputAnchor {
        Top = 0,
        Bottom = 1
    }
}
```

**What this API adds:**

The `terminalQuickFixProvider` API enables intelligent error handling in the terminal through a provider pattern. When you register a provider using `window.registerTerminalQuickFixProvider()`, VS Code calls your `provideTerminalQuickFixes()` method whenever a command matches your configured patterns. The `TerminalCommandMatchResult` parameter gives you both the command line and any regex matches from the command or its output, letting you understand exactly what failed and why.

The API supports three types of fixes: `TerminalQuickFixTerminalCommand` for suggesting terminal commands (with optional auto-execution), `TerminalQuickFixOpener` for opening files or URLs (useful for linking to documentation), and standard VS Code `Command` objects for triggering any editor action. You can return a single fix or an array of suggestions, giving users multiple options to resolve the error. The `TerminalOutputMatcher` configuration lets you target specific parts of command output - anchoring from top or bottom with configurable offset and length - so you can match against error messages without scanning the entire output buffer.

### How Puku Editor Uses It

**Main Implementation:** `src/chat/src/extension/conversation/vscode-node/terminalFixGenerator.ts`

Puku uses AI to generate intelligent terminal quick fixes. The implementation has two main parts: the orchestration function that manages the UI, and the generator class that handles AI integration.

**Part 1: Data Structures and State**

```typescript
const enum CommandRelevance {
    Low = 1,
    Medium = 2,
    High = 3,
}

export interface ICommandSuggestion {
    command: string;
    description: string;
    relevance: CommandRelevance;
}

// Global state to store the last command match result
export function setLastCommandMatchResult(value: vscode.TerminalCommandMatchResult) {
    lastCommandMatchResult = value;
}
export let lastCommandMatchResult: vscode.TerminalCommandMatchResult | undefined;
```

The `CommandRelevance` enum defines three priority levels for suggestions. The `ICommandSuggestion` interface represents a single fix with its command, description, and relevance score. The `lastCommandMatchResult` variable stores the most recent error context, set by the quick fix provider when an error is detected.

**Part 2: UI Orchestration**

```typescript
export async function generateTerminalFixes(instantiationService: IInstantiationService) {
    const commandMatchResult = lastCommandMatchResult;
    if (!commandMatchResult) {
        return;
    }

    // Generate fixes using AI
    const picksPromise = instantiationService
        .createInstance(TerminalQuickFixGenerator)
        .generateTerminalQuickFix(commandMatchResult, CancellationToken.None)
        .then(fixes => {
            // Sort by relevance (high first) and add separator labels
            const picks = (fixes ?? [])
                .sort((a, b) => b.relevance - a.relevance)
                .map(e => ({
                    label: e.command,
                    description: e.description,
                    suggestion: e
                }));
            // Insert separators between relevance groups
            // (actual code adds QuickPickItemKind.Separator items)
            return picks;
        });

    // Show quick pick with loading animation
    const pick = vscode.window.createQuickPick();
    pick.placeholder = 'Generating...';
    pick.busy = true;
    pick.show();

    // Wait for AI results
    pick.items = await picksPromise;
    pick.busy = false;

    // Handle selection
    await new Promise<void>(r => pick.onDidAccept(() => r()));
    const item = pick.activeItems[0];
    if (item && 'suggestion' in item) {
        // Auto-execute unless command has placeholders like {filename}
        const shouldExecute = !item.suggestion.command.match(/{.+}/);
        vscode.window.activeTerminal?.sendText(item.suggestion.command, shouldExecute);
    }
    pick.dispose();
}
```

This function orchestrates the user experience. It shows a quick pick immediately with a loading indicator while AI generates suggestions in the background. When the user selects a fix, it sends the command to the terminal. The `shouldExecute` logic checks for placeholder patterns like `{filename}` - if found, the command is inserted but not executed, letting the user fill in the placeholder first.

**Part 3: AI-Powered Fix Generation**

```typescript
class TerminalQuickFixGenerator {
    constructor(
        @IEndpointProvider private readonly _endpointProvider: IEndpointProvider,
        @IInstantiationService private readonly _instantiationService: IInstantiationService,
        @ILogService private readonly _logService: ILogService,
        @IWorkspaceService private readonly _workspaceService: IWorkspaceService,
    ) {}

    async generateTerminalQuickFix(
        commandMatchResult: vscode.TerminalCommandMatchResult,
        token: CancellationToken
    ): Promise<ICommandSuggestion[] | undefined> {
        // Step 1: Extract file paths mentioned in error output
        const unverifiedContextUris = await this._generateTerminalQuickFixFileContext(
            commandMatchResult, token
        );

        // Step 2: Verify which files actually exist
        const verifiedContextUris: Uri[] = [];
        const nonExistentContextUris: Uri[] = [];
        for (const uri of unverifiedContextUris) {
            try {
                const exists = await vscode.workspace.fs.stat(uri);
                if (exists.type === vscode.FileType.File) {
                    verifiedContextUris.push(uri);
                } else {
                    nonExistentContextUris.push(uri);
                }
            } catch {
                nonExistentContextUris.push(uri);
            }
        }

        // Step 3: Build prompt with error context and file information
        const endpoint = await this._endpointProvider.getChatEndpoint('copilot-fast');
        const promptRenderer = PromptRenderer.create(
            this._instantiationService, endpoint, TerminalQuickFixPrompt, {
                commandLine: commandMatchResult.commandLine,
                verifiedContextUris,
                nonExistentContextUris,
            }
        );
        const prompt = await promptRenderer.render(undefined, undefined);

        // Step 4: Call LLM for suggestions
        const fetchResult = await endpoint.makeChatRequest(
            'terminalQuickFixGenerator',
            prompt.messages,
            undefined,
            token,
            ChatLocation.Other
        );

        // Step 5: Parse JSON response (may be wrapped in markdown code block)
        const codeblocks = extractCodeBlocks(fetchResult.value);
        const json = JSON.parse(
            codeblocks.length > 0 ? codeblocks[0].code : fetchResult.value
        );

        // Step 6: Validate and return suggestions
        return json.map(entry => ({
            command: entry.command,
            description: entry.description,
            relevance: parseRelevance(entry.relevance)
        }));
    }
}
```

The `TerminalQuickFixGenerator` class performs the actual AI integration. The key insight is the **two-pass approach**: first it asks the LLM to extract file paths from the error message (`_generateTerminalQuickFixFileContext`), then it verifies which files exist vs don't exist. This distinction is crucial - telling the AI "file X doesn't exist" helps it suggest `touch X` or `mkdir` commands, while "file X exists but has errors" leads to different suggestions. The final LLM call receives this enriched context to generate relevant fix commands.

## Document Information

**Last Updated:** February 2026

**Puku Editor Version:** 0.43.34

**VS Code Version:** 1.107.0
