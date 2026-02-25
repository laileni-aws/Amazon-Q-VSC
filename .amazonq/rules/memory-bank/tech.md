# Amazon Q for VS Code - Technology Stack

## Languages & Runtimes

-   TypeScript 5.0+ (primary language)
-   JavaScript (generated SDK clients)
-   Vue.js 3.3+ (webview components)
-   Node.js (extension host)

## VS Code Extension APIs

-   Minimum VS Code version: 1.83.0
-   Extension kind: workspace
-   Activation: `onStartupFinished`, `onUri`, `onCommand`, `onView`

## Build System

### Package Manager

-   npm 10.1.0+ with workspaces
-   Monorepo structure with `packages/*` and `plugins/*`

### Bundling

-   Webpack 5.95+ for extension bundling
-   esbuild-loader for fast TypeScript compilation
-   Vue Loader for .vue single-file components
-   Separate configs for Node.js and Web targets

### TypeScript Configuration

-   Strict mode enabled
-   ES2022 target
-   CommonJS module output

## Key Dependencies

### AWS SDKs

-   @aws-sdk/\* v3 clients (CloudFormation, Lambda, S3, etc.)
-   @amzn/codewhisperer-streaming (generated)
-   @amzn/amazon-q-developer-streaming-client (generated)

### UI Framework

-   @aws/mynah-ui 4.35+ (chat interface)
-   Vue 3.3+ (webviews)
-   @vscode/codicons (icons)

### Language Server

-   vscode-languageclient 9.0+
-   @aws/language-server-runtimes

### Testing

-   Mocha 10.1+ (test runner)
-   Sinon 14.0+ (mocking)
-   c8 (code coverage)
-   @vscode/test-electron (VS Code integration tests)

### Code Quality

-   ESLint 8.56+ with TypeScript parser
-   Prettier 3.3+ (formatting)
-   Husky 9.0+ (git hooks)
-   Custom eslint-plugin-aws-toolkits

## Development Commands

### Build

```bash
npm run compile              # Full production build
npm run compileDev           # Development build with source maps
npm run watch                # Watch mode for development
```

### Test

```bash
npm run test                 # Unit tests
npm run testE2E              # End-to-end tests
npm run testInteg            # Integration tests
npm run testWeb              # Web extension tests
```

### Lint & Format

```bash
npm run lint                 # Run ESLint
npm run lintfix              # Auto-fix lint issues
```

### Package

```bash
npm run package              # Create VSIX package
npm run vscode:prepublish    # Pre-publish build
```

### Utilities

```bash
npm run clean                # Clean build artifacts
npm run reset                # Clean and reinstall dependencies
npm run generateTelemetry    # Generate telemetry types
npm run scan-licenses        # Generate license reports
```

## Configuration Files

-   `tsconfig.json` - TypeScript configuration
-   `.eslintrc.js` - ESLint rules
-   `webpack.config.js` - Webpack bundling
-   `.prettierignore` - Prettier exclusions
-   `codecov.yml` - Code coverage settings
