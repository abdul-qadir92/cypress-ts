# Cypress TypeScript Sample Project

A sample Cypress end-to-end testing project configured with TypeScript, supporting both local execution and BrowserStack cloud testing.

> **Note:** This project uses a **non-root Cypress configuration** (`cypress/config/cypress.config.ts` instead of the default `cypress.config.ts` in the project root). This requires a **dual-path strategy** to work correctly in both local and BrowserStack environments due to different path resolution behaviors. If your Cypress config is in the standard root location, this setup is not necessary. See the [Dual-Path Strategy](#dual-path-strategy-for-local--browserstack-compatibility) section for details.

## Prerequisites

- Node.js (v14 or higher)
- npm or yarn
- BrowserStack account (for cloud testing)

## Installation

1. Install dependencies:

```bash
npm install
```

2. Install BrowserStack CLI globally (for cloud testing):

```bash
npm install -g browserstack-cypress-cli
```

## Running Tests

### Local Execution

#### Open Cypress Test Runner (Interactive Mode)

```bash
npm run cypress:open
```

#### Run Tests Headless

```bash
npm run cypress:run
```

#### Run Tests in Headed Mode

```bash
npm run cypress:run:headed
```

### BrowserStack Cloud Execution

#### Run tests on BrowserStack (async)

```bash
npm run browserstack
```

#### Run tests on BrowserStack (sync - waits for completion)

```bash
npm run browserstack:sync
```

#### Direct CLI usage with config file

For non-root config setups, you **must** set `BSTACK_USE_RELATIVE_PATHS=true` to enable relative path resolution:

```bash
# Basic run command for non-root config
BSTACK_USE_RELATIVE_PATHS=true browserstack-cypress run --ccf ./cypress/config/cypress.config.ts

# With sync mode (wait for results)
BSTACK_USE_RELATIVE_PATHS=true browserstack-cypress run --sync --ccf ./cypress/config/cypress.config.ts
```

See: [BrowserStack Configuration File Documentation](https://www.browserstack.com/docs/automate/cypress/configuration-file)

#### Running specific spec files

You can run specific specs using the `--spec` or `-s` flag:

```bash
# Run a single spec file
BSTACK_USE_RELATIVE_PATHS=true browserstack-cypress run --sync \
  --ccf ./cypress/config/cypress.config.ts \
  --spec "cypress/e2e/example.cy.ts"

# Run multiple spec files (comma-separated)
BSTACK_USE_RELATIVE_PATHS=true browserstack-cypress run --sync \
  --ccf ./cypress/config/cypress.config.ts \
  --spec "cypress/e2e/example.cy.ts,cypress/e2e/kitchen-sink.cy.ts"

# Run all specs matching a glob/regex pattern
BSTACK_USE_RELATIVE_PATHS=true browserstack-cypress run --sync \
  --ccf ./cypress/config/cypress.config.ts \
  --spec "cypress/e2e/**/*.cy.ts"

# Run specs in a specific folder
BSTACK_USE_RELATIVE_PATHS=true browserstack-cypress run --sync \
  --ccf ./cypress/config/cypress.config.ts \
  --spec "cypress/e2e/smoke/**/*"
```

You can also specify specs in `browserstack.json`:

```json
{
  "run_settings": {
    "specs": [
      "cypress/e2e/example.cy.ts",
      "cypress/e2e/kitchen-sink.cy.ts"
    ]
  }
}
```

See: [BrowserStack Spec Files Documentation](https://www.browserstack.com/docs/automate/cypress/specs-to-run)

## Project Structure

```
.
├── cypress/
│   ├── config/
│   │   └── cypress.config.ts  # Cypress configuration (non-root location)
│   ├── e2e/                   # Test files
│   │   ├── example.cy.ts
│   │   └── kitchen-sink.cy.ts
│   ├── fixtures/              # Test data
│   │   └── example.json
│   └── support/               # Support files
│       ├── commands.ts        # Custom commands
│       └── e2e.ts             # Global configuration
├── browserstack.json          # BrowserStack configuration
├── tsconfig.json              # TypeScript configuration
└── package.json
```

## Configuration

### Cypress Config Location

The Cypress configuration file is located at `cypress/config/cypress.config.ts` (not in the project root). This non-standard location requires special handling for path resolution.

### Dual-Path Strategy for Local + BrowserStack Compatibility

Cypress and BrowserStack resolve relative paths differently:

| Environment     | Path Resolution Behavior                          |
|-----------------|---------------------------------------------------|
| **Local Cypress** | Relative paths resolve from the **project root** |
| **BrowserStack**  | Relative paths resolve from the **config file location** |

To support both environments, the configuration uses a **dual-path strategy**:

#### Absolute Paths (Default - for Local Execution)

The config computes absolute paths using `__dirname`:

```typescript
const projectRoot = path.resolve(__dirname, '..', '..')
const cypressRoot = path.join(projectRoot, 'cypress')

const absolutePaths = {
  specPattern: path.join(cypressRoot, 'e2e', '**/*.cy.{js,jsx,ts,tsx}'),
  supportFile: path.join(cypressRoot, 'support', 'e2e.ts'),
  // ... other paths
}
```

#### Relative Paths (for BrowserStack Execution)

When running on BrowserStack, relative paths are used (relative to the config file at `cypress/config/`):

```typescript
const relativePaths = {
  specPattern: '../e2e/**/*.cy.{js,jsx,ts,tsx}',
  supportFile: '../support/e2e.ts',
  // ... other paths
}
```

#### Switching Between Path Modes

The `setupNodeEvents` hook checks for environment variables to switch to relative paths:

- `BSTACK_USE_RELATIVE_PATHS=true` (recommended for BrowserStack)
- `USE_RELATIVE_PATHS=true`
- `CYPRESS_USE_RELATIVE_PATHS=true`

**Important:** You must explicitly set `BSTACK_USE_RELATIVE_PATHS=true` when running on BrowserStack:

```bash
BSTACK_USE_RELATIVE_PATHS=true browserstack-cypress run
```

The npm scripts (`npm run browserstack` and `npm run browserstack:sync`) already include this environment variable.

### BrowserStack Configuration

#### Setting up `browserstack.json`

Before running tests on BrowserStack, configure the following in `browserstack.json`:

1. **Authentication** - Add your BrowserStack credentials:

```json
{
  "auth": {
    "username": "YOUR_BROWSERSTACK_USERNAME",
    "access_key": "YOUR_BROWSERSTACK_ACCESS_KEY"
  }
}
```

2. **Home Directory** (REQUIRED) - Set the absolute path to your local repository:

```json
{
  "run_settings": {
    "home_directory": "/absolute/path/to/your/cypress-repo"
  }
}
```

> **Why is `home_directory` required?**
>
> When BrowserStack CLI uploads your project, it zips the directory containing the Cypress config file. Since our config is at `cypress/config/cypress.config.ts`, only the `cypress/config/` folder would be uploaded by default - missing all tests, fixtures, and support files.
>
> Setting `home_directory` to the project root ensures the entire project is zipped and uploaded.
>
> See: [BrowserStack Home Directory Documentation](https://www.browserstack.com/docs/automate/cypress/home-directory)

#### Example Setup for New Developers

```bash
# 1. Clone the repository
git clone <repo-url>
cd cypress-ts

# 2. Install dependencies
npm install

# 3. Update browserstack.json with your details
# - Set auth.username and auth.access_key
# - Set run_settings.home_directory to your local repo path
#   Example: /Users/yourname/projects/cypress-ts

# 4. Run tests locally
npm run cypress:run

# 5. Run tests on BrowserStack
npm run browserstack:sync
```

## Writing Tests

Tests are written in TypeScript (`.ts` files) in the `cypress/e2e/` directory. Example:

```typescript
describe('My Test Suite', () => {
  it('should do something', () => {
    cy.visit('/')
    cy.get('h1').should('be.visible')
  })
})
```

## Custom Commands

Custom commands can be added in `cypress/support/commands.ts`. An example `dataCy` command is already included.

## Resources

- [Cypress Documentation](https://docs.cypress.io/)
- [TypeScript Documentation](https://www.typescriptlang.org/docs/)
- [BrowserStack Cypress Integration](https://www.browserstack.com/docs/automate/cypress/integrate-your-test)
- [BrowserStack Home Directory](https://www.browserstack.com/docs/automate/cypress/home-directory)
- [BrowserStack Configuration File](https://www.browserstack.com/docs/automate/cypress/configuration-file)
- [BrowserStack Spec Files](https://www.browserstack.com/docs/automate/cypress/specs-to-run)

