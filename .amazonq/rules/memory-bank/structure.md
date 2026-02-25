# Amazon Q for VS Code - Project Structure

## Repository Overview

Monorepo using npm workspaces containing the Amazon Q VS Code extension and shared core library.

## Root Directory Structure

```
amazon-q-vscode/
├── packages/           # Main extension packages
│   ├── amazonq/        # Amazon Q extension entry point
│   └── core/           # Shared core library (aws-core-vscode)
├── plugins/            # Custom ESLint plugins
├── scripts/            # Build and utility scripts
├── src.gen/            # Generated AWS SDK clients
├── docs/               # Documentation
├── designs/            # Design documents
├── buildspec/          # CI/CD build specifications
└── .github/            # GitHub workflows and templates
```

## Package: amazonq (Extension Entry Point)

```
packages/amazonq/
├── src/                # Extension source (thin wrapper)
├── resources/          # Icons, fonts, marketplace assets
├── scripts/            # Package-specific build scripts
├── test/               # Unit and E2E tests
├── package.json        # VS Code extension manifest
└── webpack.config.js   # Bundle configuration
```

## Package: core (Shared Library)

```
packages/core/src/
├── amazonq/            # Amazon Q chat and auth components
├── amazonqGumby/       # Code transformation (Q Transform)
├── amazonqScan/        # Security scanning
├── codewhisperer/      # Inline suggestions engine
├── codewhispererChat/  # Chat controllers and views
├── auth/               # Authentication (SSO, credentials)
├── shared/             # Utilities, clients, telemetry
├── awsService/         # AWS service integrations
├── lambda/             # Lambda function support
├── login/              # Login webview components
├── notifications/      # Notification system
└── test/               # Test suites
```

## Core Components

### Authentication (`auth/`, `amazonq/auth/`)

-   SSO and Builder ID authentication flows
-   Credential providers and caching
-   Connection state management

### Chat System (`codewhispererChat/`, `amazonq/webview/`)

-   Chat controllers and message handling
-   Mynah UI integration
-   Session storage and context management

### Inline Completions (`codewhisperer/`)

-   InlineCompletionItemProvider implementation
-   Recommendation handling and caching
-   Reference tracking and code coverage

### Security (`codewhisperer/service/`)

-   Security scan handlers
-   Diagnostic providers
-   Issue tree view and code actions

### Transformation (`amazonqGumby/`)

-   Transform by Q orchestration
-   Transformation hub UI
-   Job history and telemetry

### AWS Services (`awsService/`)

-   CloudFormation, Lambda, S3, ECS integrations
-   Explorer nodes and commands
-   Service-specific wizards

## Architectural Patterns

### Extension Activation

-   `extensionNode.ts` / `extensionWeb.ts` entry points
-   Lazy activation via `onStartupFinished`
-   Feature-based activation modules

### Webview Architecture

-   Vue.js 3 for complex UIs
-   Mynah UI for chat interface
-   Message-based communication with extension host

### Client Architecture

-   AWS SDK v3 clients with custom wrappers
-   Streaming clients for chat responses
-   Generated clients in `src.gen/@amzn/`

### State Management

-   Global state via VS Code Memento API
-   Session-based chat storage
-   Credential caching with TTL
