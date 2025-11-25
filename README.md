# @enorim/di

A lightweight dependency injection container for TypeScript/JavaScript applications. It provides a flexible and type-safe way to manage dependencies, supporting service registration, dependency resolution, scoping, lazy initialization, and lifecycle management.

## Design Philosophy

The container is designed to be **statically configured** - all service registrations should be done upfront during application initialization. The container should not be modified at runtime by adding or removing services dynamically. This approach ensures:

- **Predictable behavior**: Service dependencies are known at startup
- **Better performance**: No runtime container modifications
- **Easier debugging**: Clear service topology
- **Type safety**: All dependencies are resolved at container creation time

## Features

- **Type-safe**: Full TypeScript support with strong typing
- **Dependency Resolution**: Automatic resolution of service dependencies
- **Service Identifiers**: Support for interface-based service registration
- **Scoping**: Hierarchical service scopes for modular applications
- **Lazy Initialization**: Services are created only when needed
- **Lifecycle Management**: Automatic disposal of services with cleanup support
- **Variants**: Multiple implementations for the same service
- **Error Handling**: Comprehensive error messages for common issues

## Basic Usage

### Container Configuration

Configure your container once during application startup:

```typescript
import { Container } from '@enorim/di';

// Configure container at startup - do this ONCE
function configureContainer(): Container {
  const container = new Container();

  // Register all your services upfront
  container
    .add(DatabaseService)
    .add(UserService, [DatabaseService])
    .add(EmailService)
    .add(NotificationService, [EmailService]);

  return container;
}

// Use throughout your application
const container = configureContainer();
const provider = container.provider();
```

### Simple Service Registration

```typescript
import { Container } from '@enorim/di';

// Define your services
class DatabaseService {
  connect() {
    console.log('Connected to database');
  }
}

class UserService {
  constructor(private db: DatabaseService) {}

  getUser(id: string) {
    this.db.connect();
    return { id, name: 'John Doe' };
  }
}

// Create container and register services
const container = new Container();
container.add(DatabaseService).add(UserService, [DatabaseService]);

// Create provider and get services
const provider = container.provider();
const userService = provider.get(UserService);
console.log(userService.getUser('123'));
```

### Service Identifiers (Interface-based)

Use service identifiers to register implementations for interfaces:

```typescript
import { Container, createIdentifier } from '@enorim/di';

// Define interface and identifier
interface Logger {
  log(message: string): void;
}
const Logger = createIdentifier<Logger>('Logger');

// Implement the interface
class ConsoleLogger implements Logger {
  log(message: string) {
    console.log(message);
  }
}

class ApiService {
  constructor(private logger: Logger) {}

  fetch() {
    this.logger.log('Fetching data...');
  }
}

// Register services
const container = new Container();
container.addImpl(Logger, ConsoleLogger).add(ApiService, [Logger]);

const provider = container.provider();
const api = provider.get(ApiService);
api.fetch(); // Logs: "Fetching data..."
```

## Advanced Features

### Service Variants

Register multiple implementations of the same service:

```typescript
interface Storage {
  save(data: string): void;
}
const Storage = createIdentifier<Storage>('Storage');

class LocalStorage implements Storage {
  save(data: string) {
    localStorage.setItem('data', data);
  }
}

class RemoteStorage implements Storage {
  save(data: string) {
    // Save to remote server
  }
}

const container = new Container();
container
  .addImpl(Storage('local'), LocalStorage)
  .addImpl(Storage('remote'), RemoteStorage);

const provider = container.provider();
const localStorage = provider.get(Storage('local'));
const remoteStorage = provider.get(Storage('remote'));
```

### Collection Dependencies

Inject all implementations of a service:

```typescript
interface Plugin {
  name: string;
  execute(): void;
}
const Plugin = createIdentifier<Plugin>('Plugin');

class PluginManager {
  constructor(private plugins: Plugin[]) {}

  runAll() {
    this.plugins.forEach((plugin) => plugin.execute());
  }
}

const container = new Container();
container
  .addImpl(Plugin('auth'), { name: 'Auth', execute: () => {} })
  .addImpl(Plugin('logging'), { name: 'Logging', execute: () => {} })
  .add(PluginManager, [[Plugin]]); // Note the double brackets for collection

const provider = container.provider();
const manager = provider.get(PluginManager);
manager.runAll(); // Executes all plugins
```

### Scoped Services

Create hierarchical service scopes:

```typescript
import { Container, createScope } from '@enorim/di';

const workspaceScope = createScope('workspace');
const pageScope = createScope('page', workspaceScope);

class SystemService {
  appName = 'MyApp';
}

class WorkspaceService {
  constructor(public system: SystemService) {}
  name = 'Default Workspace';
}

class PageService {
  constructor(
    public system: SystemService,
    public workspace: WorkspaceService
  ) {}
  title = 'Default Page';
}

const container = new Container();
container.add(SystemService);
container.scope(workspaceScope).add(WorkspaceService, [SystemService]);
container.scope(pageScope).add(PageService, [SystemService, WorkspaceService]);

// Create providers with different scopes
const rootProvider = container.provider();
const workspaceProvider = container.provider(workspaceScope, rootProvider);
const pageProvider = container.provider(pageScope, workspaceProvider);

// Services are only available in their respective scopes
const system = rootProvider.get(SystemService); // ✓ Available
const page = pageProvider.get(PageService); // ✓ Available in page scope
// rootProvider.get(WorkspaceService); // ✗ Throws ServiceNotFoundError
```

### Lazy Initialization and Factory Functions

Services can be created using factory functions for lazy initialization:

```typescript
interface Config {
  apiUrl: string;
}
const Config = createIdentifier<Config>('Config');

class ApiClient {
  constructor(private config: Config) {}
}

const container = new Container();
container.addImpl(Config, (provider) => ({
  apiUrl: process.env.API_URL || 'http://localhost:3000',
}));
container.add(ApiClient, [Config]);

// Config is only created when ApiClient is requested
const provider = container.provider();
const client = provider.get(ApiClient);
```

### Service Override

Override existing service implementations:

```typescript
class ProductionLogger implements Logger {
  log(message: string) {
    console.log(`[PROD] ${message}`);
  }
}

class TestLogger implements Logger {
  log(message: string) {
    // Silent logger for tests
  }
}

const container = new Container();
container.addImpl(Logger, ProductionLogger);

// Override for testing
container.override(Logger, TestLogger);

const provider = container.provider();
const logger = provider.get(Logger); // Gets TestLogger
```

### Lifecycle Management

Services implementing `Symbol.dispose` are automatically cleaned up:

```typescript
class DatabaseConnection {
  connected = false;

  connect() {
    this.connected = true;
    console.log('Database connected');
  }

  [Symbol.dispose]() {
    this.connected = false;
    console.log('Database disconnected');
  }
}

const container = new Container();
container.add(DatabaseConnection);

const provider = container.provider();
const db = provider.get(DatabaseConnection);
db.connect();

// Clean up all services
provider.dispose(); // Logs: "Database disconnected"
```

## Best Practices

1. **Static Configuration**: Configure the entire container once at application startup

   ```typescript
   // ✅ Good: Configure everything upfront
   const container = new Container();
   container.add(ServiceA).add(ServiceB, [ServiceA]).add(ServiceC, [ServiceB]);

   // ❌ Avoid: Runtime modifications
   // Don't do this after the container is in use
   // container.add(NewService);
   ```

2. **Separation of Configuration and Usage**: Keep container setup separate from business logic

   ```typescript
   // config/container.ts
   export function createAppContainer(): Container {
     const container = new Container();
     // All registrations here
     return container;
   }

   // main.ts
   import { createAppContainer } from './config/container';
   const container = createAppContainer();
   const provider = container.provider();
   ```

3. **Use Scopes for Modules**: Organize services by feature using scopes

   ```typescript
   const authScope = createScope('auth');
   const userScope = createScope('user');

   container
     .scope(authScope)
     .add(AuthService)
     .scope(userScope)
     .add(UserService, [AuthService]);
   ```

4. **Lifecycle Management**: Implement cleanup for resources
   ```typescript
   class ResourceService {
     [Symbol.dispose]() {
       // Cleanup resources
     }
   }
   ```
