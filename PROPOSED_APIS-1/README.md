# Level 1: Foundation APIs

Foundation APIs are the simplest proposed APIs in Puku Editor. They provide basic infrastructure that higher-level features (chat, search, terminal) build upon. This guide covers these 5 APIs in depth, showing how Puku uses them and how to modify them in the future.

![Foundation APIs Overview](./images/4.svg)

## Quick Reference Table

| API | Purpose | Puku Usage |
|-----|---------|------------|
| `extensionsAny` | Access extensions across all hosts | Reserved for future |
| `authLearnMore` | Add help links to auth dialogs | Error notifications |
| `devDeviceId` | Alternative device identifier | Telemetry, experiments |
| `dataChannels` | Extension ↔ Workbench communication | Telemetry forwarding |
| `resolvers` | Remote development connections | Reserved for future |

## File Locations in Puku Editor

Before diving into each API, here's where you'll find the relevant files:

```
src/chat/
├── package.json                          ← API declarations
├── src/extension/
│   └── vscode.proposed.*.d.ts           ← Copied type definitions
└── src/platform/
    ├── env/vscode/envServiceImpl.ts     ← devDeviceId usage
    └── telemetry/vscode/                ← dataChannels usage
```

## API 1: extensionsAny

### What Problem Does It Solve?

VS Code runs extensions in separate processes called "Extension Hosts." By default, your extension can only see other extensions running in the same host. But what if you need to check if an extension is installed on a remote machine or in a different host?

The `extensionsAny` API solves this by letting you query extensions across ALL extension hosts—local, remote, and web.

**Real-world scenario:** Imagine Puku Editor needs to check if a language server extension is installed on a remote SSH connection. Without `extensionsAny`, this would be impossible.

![extensionsAny API](./images/5.svg)

> **In short:** Your extension can always see local extensions (A, B). With `extensionsAny`, it can also discover extensions on remote servers (C, D) - but only their metadata, not call their functions.

### API Definition

**Definition File:** `vscode.proposed.extensionsAny.d.ts`

**Location in VS Code:** `src/vscode-dts/vscode.proposed.extensionsAny.d.ts`

**Location in Puku:** `src/chat/src/extension/vscode.proposed.extensionsAny.d.ts`

**GitHub Issue:** [microsoft/vscode#145307](https://github.com/microsoft/vscode/issues/145307)

```typescript
declare module 'vscode' {

    // https://github.com/microsoft/vscode/issues/145307 @alexdima

    export interface Extension<T> {

        /**
         * `true` when the extension is associated to another extension host.
         *
         * *Note* that an extension from another extension host cannot export
         * API, e.g {@link Extension.exports its exports} are always `undefined`.
         */
        readonly isFromDifferentExtensionHost: boolean;
    }

    export namespace extensions {

        /**
         * Get an extension by its full identifier in the form of: `publisher.name`.
         *
         * @param extensionId An extension identifier.
         * @param includeDifferentExtensionHosts Include extensions from different extension host
         * @return An extension or `undefined`.
         */
        export function getExtension<T = any>(extensionId: string, includeDifferentExtensionHosts: boolean): Extension<T> | undefined;

        /**
         * All extensions across all extension hosts.
         *
         * @see {@link Extension.isFromDifferentExtensionHost}
         */
        export const allAcrossExtensionHosts: readonly Extension<void>[];

    }
}
```

**What this API adds:**

This API adds three things: First, `isFromDifferentExtensionHost` is a boolean property on any Extension object that tells you if that extension is running on a remote host (SSH, WSL, Container) - if `true`, you can see the extension but cannot call its functions. Second, `getExtension(id, true)` lets you pass `true` as a second parameter to search across all hosts instead of just local. Third, `allAcrossExtensionHosts` gives you an array of ALL extensions from ALL hosts (local + remote) at once.

### How Puku Editor Uses It

**Current Status:** Declared in `package.json` but not actively used.

```json
// src/chat/package.json
{
    "enabledApiProposals": [
        "extensionsAny",
        // ... other APIs
    ]
}
```

**Why Reserved:** This API is useful for remote development scenarios. Puku Editor may use it in the future to:
- Detect if required extensions are installed on remote hosts
- Coordinate with other extensions across hosts
- Provide better error messages when dependencies are missing

### Code Example: How to Use This API

```typescript
import * as vscode from 'vscode';

// Check if an extension is installed anywhere
function isExtensionInstalled(extensionId: string): boolean {
    // Without extensionsAny - only checks local host
    const localOnly = vscode.extensions.getExtension(extensionId);

    // With extensionsAny - checks all hosts
    const anywhere = vscode.extensions.getExtension(extensionId, true);

    return anywhere !== undefined;
}

// List all extensions across all hosts
function listAllExtensions(): void {
    const allExtensions = vscode.extensions.allAcrossExtensionHosts;

    for (const ext of allExtensions) {
        console.log(`${ext.id} - Remote: ${ext.isFromDifferentExtensionHost}`);
    }
}

// Check if we can access an extension's exports
function canAccessExtensionExports(extensionId: string): boolean {
    const ext = vscode.extensions.getExtension(extensionId, true);

    if (!ext) {
        return false;
    }

    // Can only access exports from same host
    return !ext.isFromDifferentExtensionHost;
}
```

### Important Limitations

| Limitation | Why It Matters |
|------------|----------------|
| **No export sharing** | Extensions from different hosts run in separate processes. You can see them but cannot call their APIs. |
| **Read-only access** | You can query extensions but cannot activate or control them. |
| **Performance cost** | Querying all hosts requires cross-process communication, which is slower than local queries. |

### How to Modify This API Usage

If you need to add new functionality using `extensionsAny`:

1. **Check the current .d.ts file** for available methods
2. **Create a service wrapper** (like `IExtensionDiscoveryService`) to abstract the API
3. **Handle the remote case** - remember you can't access exports from remote extensions
4. **Add error handling** for when extensions aren't found

## API 2: authLearnMore

### What Problem Does It Solve?

When VS Code shows an authentication dialog, users might be confused about why they need to sign in or what permissions are being requested. The `authLearnMore` API adds a "Learn More" link to these dialogs, helping users understand what's happening.

**Real-world scenario:** When Puku Editor asks for GitHub authentication, users might wonder "Why does this editor need my GitHub account?" A Learn More link can explain that it's needed for Copilot features.

![](./images/6.svg)

> **In short:** The `authLearnMore` API adds a "Learn More" link to authentication dialogs. When users click it, they're taken to documentation explaining why authentication is needed—reducing confusion and building trust.

### API Definition

**Definition File:** `vscode.proposed.authLearnMore.d.ts`

**Location in VS Code:** `src/vscode-dts/vscode.proposed.authLearnMore.d.ts`

**Location in Puku:** `src/chat/src/extension/vscode.proposed.authLearnMore.d.ts`

**GitHub Issue:** [microsoft/vscode#206587](https://github.com/microsoft/vscode/issues/206587)

```typescript
declare module 'vscode' {

    // https://github.com/microsoft/vscode/issues/206587

    export interface AuthenticationGetSessionPresentationOptions {
        /**
         * An optional Uri to open in the browser to learn more about this authentication request.
         */
        learnMore?: Uri;
    }
}
```

**How this connects to `getSession()`:**

`vscode.authentication.getSession()` is the main VS Code API for requesting user authentication. Extensions call it to get an authentication session (like GitHub login). It takes a provider ID, scopes, and options.

The stable API already defines `createIfNone` and `forceNewSession` options as either `boolean` OR `AuthenticationGetSessionPresentationOptions`:

```typescript
// From vscode.d.ts (STABLE API)
export interface AuthenticationGetSessionOptions {
    createIfNone?: boolean | AuthenticationGetSessionPresentationOptions;
    forceNewSession?: boolean | AuthenticationGetSessionPresentationOptions;
}
```

The proposed API adds `learnMore` to `AuthenticationGetSessionPresentationOptions`. So instead of passing `true`, you pass an object with `learnMore`:

```
Without proposed API:  createIfNone: true
With proposed API:     createIfNone: { learnMore: Uri }
```

### How Puku Editor Uses It

**Current Status:** Declared in `package.json` but not actively used.

```json
// src/chat/package.json
{
    "enabledApiProposals": [
        "authLearnMore",
        // ... other APIs
    ]
}
```

**Why Reserved:** Puku Editor currently implements "Learn More" links manually using stable APIs (like `showWarningMessage` + `env.openExternal`). The `authLearnMore` proposed API is reserved for future use when authentication dialogs need built-in help links.

**Files using the manual "Learn More" pattern (stable APIs):**
- `src/chat/src/extension/completions-core/vscode-node/lib/src/error/userErrorNotifier.ts` - Certificate error notifications
- `src/chat/src/extension/completions-core/vscode-node/extension/src/modelPicker.ts` - Model picker
- `src/chat/src/extension/inlineEdits/vscode-node/inlineEditProviderFeature.ts` - Inline edits

**Example of manual pattern (NOT using the proposed API):**
```typescript
// From userErrorNotifier.ts - manual "Learn More" using stable APIs
const learnMoreAction = { title: 'Learn more' };
this._notificationSender.showWarningMessage(errorMsg, learnMoreAction)
    .then(userResponse => {
        if (userResponse?.title === learnMoreAction.title) {
            return vscode.env.openExternal(Uri.parse(learnMoreLink));
        }
    });
```

**How you would use the proposed API:**
```typescript
// Using authLearnMore proposed API (not currently used in Puku)
const session = await vscode.authentication.getSession(
    'github',
    ['user:email', 'read:org'],
    {
        // Instead of createIfNone: true, pass an object with learnMore
        createIfNone: {
            learnMore: vscode.Uri.parse('https://docs.puku.sh/auth/why-github')
        }
    }
);
```

### Code Example: Implementing Learn More Links

```typescript
import * as vscode from 'vscode';

const HELP_URLS = {
    auth: 'https://docs.puku.sh/auth',
    permissions: 'https://docs.puku.sh/permissions',
    troubleshooting: 'https://docs.puku.sh/troubleshooting'
};

// Request authentication with help link
async function requestAuthWithHelp(): Promise<vscode.AuthenticationSession | undefined> {
    try {
        return await vscode.authentication.getSession(
            'puku',                    // Provider ID
            ['ai:chat', 'ai:edit'],    // Required scopes
            {
                // Pass presentation options object instead of true
                createIfNone: {
                    learnMore: vscode.Uri.parse(HELP_URLS.auth)
                }
            }
        );
    } catch (error) {
        // Show error with troubleshooting link
        const action = await vscode.window.showErrorMessage(
            'Authentication failed. Please try again.',
            'Learn More'
        );

        if (action === 'Learn More') {
            await vscode.env.openExternal(
                vscode.Uri.parse(HELP_URLS.troubleshooting)
            );
        }

        return undefined;
    }
}
```

### When to Use This API

Use this API when users might need context about authentication. For first-time authentication, link to "Why sign in?" documentation. When requesting new permissions, link to permission explanations. For authentication errors, provide a troubleshooting guide link. For complex auth flows like OAuth, link to a step-by-step guide.

### How to Modify This API Usage

To add new learn more links:

1. **Define your help URLs** in a central location (e.g., `src/chat/src/common/helpUrls.ts`)
2. **Add the `learnMore` option** to `getSession` calls
3. **Ensure URLs are valid** and documentation exists at those URLs
4. **Consider localization** - URLs might need to point to localized docs

## API 3: devDeviceId

### What Problem Does It Solve?

VS Code provides `vscode.env.machineId` for device identification, but it's designed for production use. The `devDeviceId` API provides an alternative identifier specifically for development and testing scenarios.

**Real-world scenario:** Imagine Puku Editor wants to test a new chat UI design. They need to split users into two groups - Group A sees the old UI, Group B sees the new UI. The problem: how do you ensure the same user always sees the same UI? If you randomly assign on each session, a user might see the old UI today and new UI tomorrow - that's confusing and breaks the experiment. The solution: use `devDeviceId` to generate a consistent hash. Since `devDeviceId` never changes for a device, the same user always lands in the same group across all sessions.

![](./images/8.svg)

> **In short:** `machineId` is for production telemetry - tracking how many users use VS Code, crash reports, and usage analytics that Microsoft collects (privacy-sensitive, don't use for experiments). `devDeviceId` is for development use cases like A/B testing and feature flags - it's safe to use without affecting production data.

![](./images/7.svg)

### API Definition

**Definition File:** `vscode.proposed.devDeviceId.d.ts`

**Location in VS Code:** `src/vscode-dts/vscode.proposed.devDeviceId.d.ts`

**Location in Puku:** `src/chat/src/extension/vscode.proposed.devDeviceId.d.ts`

```typescript
declare module 'vscode' {

    export namespace env {

        /**
         * An alternative unique identifier for the computer.
         */
        export const devDeviceId: string;
    }
}
```

**What this API adds:**

This API adds a single property `vscode.env.devDeviceId` - a string that uniquely identifies the device. Unlike `machineId` which is intended for production telemetry, `devDeviceId` is specifically for development scenarios like A/B testing, feature flags, and experiment assignment. The value is stable across sessions but may differ from `machineId`.

### How Puku Editor Uses It

**Main Implementation:** `src/chat/src/platform/env/vscode/envServiceImpl.ts`

```typescript
import * as vscode from 'vscode';

export class EnvServiceImpl implements IEnvService {
    // Stable production identifier
    public get machineId(): string {
        return vscode.env.machineId;
    }

    // Development/testing identifier (proposed API)
    public get devDeviceId(): string {
        return vscode.env.devDeviceId;
    }

    // Session-specific identifier
    public get sessionId(): string {
        return vscode.env.sessionId;
    }
}
```

**Other files using this API:**
- `src/chat/src/platform/telemetry/vscode-node/microsoftExperimentationService.ts` - A/B testing
- `src/chat/src/platform/telemetry/node/azureInsightsReporter.ts` - Telemetry reporting
- `src/chat/src/platform/authentication/node/copilotTokenManager.ts` - Token management

### Design Pattern: Service Abstraction

Puku Editor wraps the proposed API in a service interface. This pattern is important for:

1. **Testability** - Tests can provide mock implementations
2. **Future-proofing** - If the API changes, only one file needs updating
3. **Consistency** - All code accesses device IDs the same way

```typescript
// Interface definition (platform-agnostic)
export interface IEnvService {
    readonly machineId: string;
    readonly devDeviceId: string;
    readonly sessionId: string;
}

// Real implementation (uses proposed API)
export class EnvServiceImpl implements IEnvService {
    get devDeviceId(): string {
        return vscode.env.devDeviceId;
    }
    // ... other properties
}

// Test implementation (returns predictable values)
export class TestEnvService implements IEnvService {
    get devDeviceId(): string {
        return 'test-device-id-12345';
    }
    // ... other properties
}
```

### Code Example: Using devDeviceId for Experiments

```typescript
import { IEnvService } from './envService';

class ExperimentationService {
    constructor(private envService: IEnvService) {}

    // Consistently assign user to experiment group
    getExperimentGroup(experimentName: string): 'control' | 'treatment' {
        const deviceId = this.envService.devDeviceId;

        // Hash the device ID + experiment name for consistent assignment
        const hash = this.hashString(`${deviceId}:${experimentName}`);

        // 50/50 split
        return hash % 2 === 0 ? 'control' : 'treatment';
    }

    private hashString(str: string): number {
        let hash = 0;
        for (let i = 0; i < str.length; i++) {
            hash = ((hash << 5) - hash) + str.charCodeAt(i);
            hash |= 0; // Convert to 32-bit integer
        }
        return Math.abs(hash);
    }
}
```

### Difference Between machineId and devDeviceId

| Property | machineId | devDeviceId |
|----------|-----------|-------------|
| **Stability** | Very stable, rarely changes | May vary between sessions |
| **Purpose** | Production identification | Development/testing |
| **Privacy** | Hashed, anonymized | Alternative hash |
| **When to use** | Production telemetry | A/B testing, experiments |

### How to Modify This API Usage

To add new uses of `devDeviceId`:

1. **Use the `IEnvService` interface** instead of accessing `vscode.env.devDeviceId` directly
2. **Inject the service** via constructor or dependency injection
3. **Add tests** using `TestEnvService` with predictable values
4. **Document the use case** in the service or calling code

## API 4: dataChannels

### What Problem Does It Solve?

VS Code extensions run in a separate process from the main VS Code UI (workbench). Sometimes these two need to communicate—for example, when the workbench completes an authentication flow and needs to notify the extension.

The `dataChannels` API creates typed communication channels between the extension host and the workbench.

**Real-world scenario:** When a user completes OAuth authentication in the browser, VS Code's workbench receives the callback. It needs to forward the authentication token to the Puku extension. `dataChannels` provides this communication path.

![](./images/9.svg)

> **In short:** Extensions and the workbench run in separate processes. `dataChannels` creates a typed message bus between them - the workbench sends data, the extension receives it via events.

### API Definition

**Definition File:** `vscode.proposed.dataChannels.d.ts`

**Location in Puku:** `src/chat/src/extension/vscode.proposed.dataChannels.d.ts`

```typescript
declare module 'vscode' {
    export namespace env {
        /**
         * Get or create a typed data channel for communication with the workbench.
         * @param channelId Unique identifier for the channel
         */
        export function getDataChannel<T>(channelId: string): DataChannel<T>;
    }

    export interface DataChannel<T = unknown> {
        /**
         * Event fired when data is received on this channel.
         */
        readonly onDidReceiveData: Event<DataChannelEvent<T>>;
    }

    export interface DataChannelEvent<T> {
        /**
         * The data that was sent.
         */
        data: T;
    }
}
```

### How Puku Editor Uses It

**Main Implementation:** `src/chat/src/extension/telemetry/vscode/githubTelemetryForwardingContrib.ts`

This file uses `dataChannels` to receive telemetry data from the VS Code workbench and forward it to GitHub's telemetry service.

```typescript
import { env, Disposable } from 'vscode';

// Define the shape of telemetry data
interface IEditTelemetryData {
    eventName: string;
    data: Record<string, unknown>;
}

export class GithubTelemetryForwardingContrib extends Disposable {
    constructor(
        @ITelemetryService private readonly _telemetryService: ITelemetryService,
        @IGitService private readonly _gitService: IGitService,
    ) {
        super();

        // Create/get a typed channel for edit telemetry
        const channel = env.getDataChannel<IEditTelemetryData>('editTelemetry');

        // Listen for data from the workbench
        this._register(channel.onDidReceiveData((args) => {
            // Transform and forward the telemetry
            const data = translateToGithubProperties(args.data.data);

            this._telemetryService.sendGHTelemetryEvent(
                'vscode.' + args.data.eventName,
                data
            );
        }));
    }
}
```

### Design Pattern: Event-Driven Communication

The `dataChannels` API follows an event-driven pattern:

1. **Create a channel** with a unique ID
2. **Subscribe to events** using `onDidReceiveData`
3. **Process received data** in the event handler
4. **Clean up** by disposing the subscription when done

```typescript
import { env, Disposable } from 'vscode';

// 1. Define your data types for type safety
interface AuthCallbackData {
    token: string;
    userId: string;
    expiresAt: number;
}

// 2. Create a handler class
class AuthCallbackHandler extends Disposable {
    private _onAuthenticated = new vscode.EventEmitter<AuthCallbackData>();
    readonly onAuthenticated = this._onAuthenticated.event;

    constructor() {
        super();

        // 3. Get the channel
        const channel = env.getDataChannel<AuthCallbackData>('pukuAuth');

        // 4. Subscribe to events
        this._register(channel.onDidReceiveData((event) => {
            console.log('Received auth callback:', event.data);

            // 5. Process the data
            this.handleAuthCallback(event.data);
        }));
    }

    private handleAuthCallback(data: AuthCallbackData): void {
        // Validate token
        if (data.expiresAt < Date.now()) {
            console.error('Token already expired');
            return;
        }

        // Store token and notify listeners
        this._onAuthenticated.fire(data);
    }
}
```

### Channel Naming Convention

Use descriptive, namespaced channel names:

| Channel Name | Purpose | Data Type |
|--------------|---------|-----------|
| `editTelemetry` | Editor action telemetry | `{ eventName, data }` |
| `pukuAuth` | Authentication callbacks | `{ token, userId }` |
| `indexingProgress` | Semantic indexing status | `{ progress, status }` |
| `chatSession` | Chat session sync | `{ sessionId, messages }` |

### How to Modify This API Usage

To add a new data channel:

1. **Define the data interface** with TypeScript types
2. **Choose a unique channel name** using your extension's namespace
3. **Create the channel** using `env.getDataChannel<YourType>('channelName')`
4. **Subscribe to events** and handle data in callbacks
5. **Dispose properly** to avoid memory leaks

**Workbench Side:** The workbench code that sends data is in:
- `src/vscode/src/vs/workbench/services/chat/common/`


## API 5: resolvers

### What Problem Does It Solve?

When you open a folder in VS Code using SSH, WSL, or a container, VS Code needs to know HOW to connect to that remote machine. The `resolvers` API lets extensions define custom remote connection types - telling VS Code "when you see this URI scheme, here's how to connect."

**Real-world scenario:** Imagine Puku Editor wants to add a "Poridhi Cloud" feature where users can open projects hosted on Puku's servers. Without resolvers, VS Code wouldn't know how to connect to `poridhi-cloud://project123`. With resolvers, Puku can register a handler that says "when you see `poridhi-cloud://`, connect to our servers at this address."

![](./images/10.svg)

> **In short:** VS Code has built-in support for SSH, WSL, and containers. The `resolvers` API lets you add YOUR OWN remote connection types - like connecting to cloud servers, custom VMs, or proprietary infrastructure.

### API Definition

**Definition File:** `vscode.proposed.resolvers.d.ts`

This is a **large and complex API** with many interfaces. Here are the key parts:

```typescript
declare module 'vscode' {
    // Main resolver interface
    export interface RemoteAuthorityResolver {
        /**
         * Called when VS Code needs to connect to a remote.
         */
        resolve(
            authority: string,
            context: RemoteAuthorityResolverContext
        ): ResolverResult | Thenable<ResolverResult>;

        /**
         * Optional: Create tunnels for port forwarding.
         */
        tunnelFactory?: (
            tunnelOptions: TunnelOptions,
            tunnelCreationOptions: TunnelCreationOptions
        ) => Thenable<Tunnel> | undefined;

        /**
         * Optional: Provide candidate ports for auto-forwarding.
         */
        showCandidatePort?: (
            host: string,
            port: number,
            detail: string
        ) => Thenable<boolean>;
    }

    // Register a resolver
    export namespace workspace {
        export function registerRemoteAuthorityResolver(
            authorityPrefix: string,
            resolver: RemoteAuthorityResolver
        ): Disposable;
    }

    // Connection result
    export class ResolvedAuthority {
        readonly host: string;
        readonly port: number;
        readonly connectionToken: string | undefined;

        constructor(host: string, port: number, connectionToken?: string);
    }
}
```

**What this API adds:**

The resolvers API enables extensions to define custom remote connection types. The `RemoteAuthorityResolver` interface is the core - it has a `resolve()` method that VS Code calls when it encounters your custom URI scheme (like `poridhi-cloud://`). Your resolver returns a `ResolvedAuthority` containing the actual host, port, and optional connection token. The `tunnelFactory` method handles port forwarding - when a user wants to access port 3000 on the remote machine from their local browser, this creates the tunnel. The `showCandidatePort` method lets you filter which auto-detected ports should be suggested for forwarding. You register your resolver using `workspace.registerRemoteAuthorityResolver('your-scheme', resolver)`. The actual API file contains many more interfaces for advanced features like `ExecServer` (running commands remotely), `TunnelOptions` (tunnel configuration), and `RemoteFileSystem` (custom file access), but the simplified version above covers the essential parts for most use cases.

### How Puku Editor Uses It

**Current Status:** Declared in `package.json` but not actively implemented.

```json
// src/chat/package.json
{
    "enabledApiProposals": [
        "resolvers",
        // ... other APIs
    ]
}
```

**Future Potential Uses:**
- **Poridhi Cloud:** Connect to cloud-hosted development environments
- **AI Sandbox:** Run AI code suggestions in isolated remote containers
- **Collaborative Editing:** Connect multiple users to shared remote workspaces

### Code Example: Implementing a Basic Resolver

```typescript
import * as vscode from 'vscode';

// Register a resolver for 'poridhi-cloud' URIs
// e.g., vscode-remote://poridhi-cloud+project123/path/to/file
export function registerPukuCloudResolver(): vscode.Disposable {
    const resolver: vscode.RemoteAuthorityResolver = {
        async resolve(authority, context) {
            // authority = 'poridhi-cloud+project123'
            const projectId = authority.split('+')[1];

            // Show progress while connecting
            return vscode.window.withProgress({
                location: vscode.ProgressLocation.Notification,
                title: `Connecting to Poridhi Cloud: ${projectId}`,
                cancellable: true
            }, async (progress, token) => {
                progress.report({ message: 'Authenticating...' });

                // Get cloud connection details from your API
                const connection = await fetchCloudConnection(projectId);

                if (token.isCancellationRequested) {
                    throw new Error('Connection cancelled');
                }

                progress.report({ message: 'Establishing connection...' });

                // Return the resolved authority
                return new vscode.ResolvedAuthority(
                    connection.host,      // e.g., 'project123.puku.cloud'
                    connection.port,      // e.g., 443
                    connection.token      // Optional connection token
                );
            });
        },

        // Optional: Handle port forwarding
        async tunnelFactory(tunnelOptions, tunnelCreationOptions) {
            // Create a tunnel for the requested port
            const tunnel = await createTunnel(
                tunnelOptions.remoteAddress.host,
                tunnelOptions.remoteAddress.port
            );

            return {
                remoteAddress: tunnelOptions.remoteAddress,
                localAddress: { host: 'localhost', port: tunnel.localPort },
                dispose: () => tunnel.close()
            };
        }
    };

    return vscode.workspace.registerRemoteAuthorityResolver('poridhi-cloud', resolver);
}

// Helper function to fetch connection details
async function fetchCloudConnection(projectId: string): Promise<{
    host: string;
    port: number;
    token: string;
}> {
    const response = await fetch(`https://api.puku.sh/cloud/${projectId}/connect`);
    return response.json();
}
```


### How to Modify This API Usage

To implement remote development features:

1. **Study existing resolvers** - Look at SSH, WSL, and Dev Containers extensions
2. **Start simple** - Implement just `resolve()` first
3. **Add tunneling** - Implement `tunnelFactory` for port forwarding
4. **Handle errors** - Remote connections are prone to failures
5. **Test thoroughly** - Remote development has many edge cases


## Handling API Changes

### Detecting Changes

When updating VS Code, proposed APIs might change. Here's how to detect and handle changes:

1. **Check VS Code Release Notes**
   - Look for "Proposed API" section
   - Note any deprecations or breaking changes

2. **Compare .d.ts Files**
   ```bash
   # After updating VS Code source
   diff src/chat/src/extension/vscode.proposed.devDeviceId.d.ts \
        src/vscode/src/vscode-dts/vscode.proposed.devDeviceId.d.ts
   ```

3. **Watch GitHub Issues**
   - Each API has an associated GitHub issue (linked in the .d.ts file)
   - Subscribe to these issues for change notifications

### Migration Checklist

When a proposed API changes:

- [ ] Read the new .d.ts file to understand changes
- [ ] Update `src/chat/src/extension/vscode.proposed.*.d.ts`
- [ ] Update version in `package.json` if required (e.g., `chatProvider@4`)
- [ ] Search for all usages: `grep -r "apiName" src/chat/`
- [ ] Update each usage to match the new API
- [ ] Run TypeScript compilation to catch type errors
- [ ] Run tests to verify functionality
- [ ] Test manually in VS Code

### If an API is Removed

If a proposed API you depend on is removed:

1. **Check if it became stable** - It might be available without `enabledApiProposals`
2. **Find an alternative** - Look for similar proposed APIs
3. **Implement a workaround** - Some functionality can be achieved differently
4. **File a feature request** - Ask VS Code team for the functionality

## Document Information

**Last Updated:** February 2026

**Puku Editor Version:** 0.43.34

**VS Code Version:** 1.107.0
