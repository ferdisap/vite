# Vite Environment API

:::warning Low-level API
Inital work for this API was introduced in Vite 5.1 with the name [Vite Runtime API](./api-vite-runtime.md). In Vite 5.2 the API was reviewed and renamed as Vite Environemnt API. It still remains an experimental feature. We are gathering feedback about the revised proposal [here](https://github.com/vitejs/vite/discussions/15774). There will probably be breaking changes to it in Vite 5.3, so make sure to pin the Vite version to `~5.2.0` when using it. This is a low-level API meant for library and framework authors. If your goal is to create an application, make sure to check out the higher-level SSR plugins and tools at [Awesome Vite SSR section](https://github.com/vitejs/awesome-vite#ssr) first.
:::

A single Vite dev server can be used to interact with different module excecution environments concurrently. We'll use the word environment to refer to a configured Vite processing pipeline that can resolve ids, load and process source code and is connected to a runtime were the code is executed. Some examples of environments are: browser, ssr, workerd, rsc. The transformed source code is called a module, and the relationship between the modules processed in a each environment are kept in a module graph. The code for these modules are sent to the runtimes associated to each environment to be executed. When a module is evaluated, the runtime will request its imported modules triggering the processing of a section of the module graph.

A Vite Module Runner allows running any code by processing it with Vite plugins first. It is different from `server.ssrLoadModule` because the runtime implementation is decoupled from the server. This allows library and framework authors to implement their own layer of communication between the Vite server and the runner. The browser communicates communicates with its corresponding environment using the server Web Socket and through HTTP requests. The SSR Module runner can directly do function calls to process modules. Other environments could run modules connecting to an edge runtime like workerd, or a Worker Thread in the same node process as Vitest does.

All these environments share Vite's HTTP server, middlewares and Web Socket. The resolved config and plugins pipeline are also shared, but plugins can use `apply` so its hooks are only called for certain environments. The environment can also be accessed inside hooks for fine grained control.

![Vite Environments](../images/vite-environments.svg)

## Using environments in the Vite server

A Vite dev server exposes two environments by default named `'browser'` and `'ssr'`. The browser environment runs in client apps that have imported the `/@vite/client` module. The SSR environment runs in the same runtime as the Vite server (for example, in node) and allows application servers to be used to render requests during dev with full HMR support. We'll discuss later how frameworks and users can create and register new environments.

The available environments can be accessed using the `server.environments` read only array:

```js
server.environments.forEach((environment) => log(environment.name))
```

Each environment can also be accessed through its name:

```js
server.environment('browser').transformRequest(url)
```

An environment is an instance of the `ModuleExecutionEnvironment` class:

```ts
class ModuleExecutionEnvironment {
  /**
   * Unique identifier for the environment in a Vite server.
   * By default Vite exposes 'browser' and 'ssr' environments.
   * The ecosystem has concensus on other environments, like 'workerd'.
   */
  name: string
  /**
   * Communication channel to send and receive messages from the
   * associated module runner in the target runtime.
   */
  hot: HMRChannel | null
  /**
   * Graph of module nodes, with the imported relationship between
   * processed modules and the cached result of the processed code.
   */
  moduleGraph: ModuleGraph
  /**
   * Trigger the execution of a module using the associated module runner
   * in the target runtime.
   */
  run: ModuleRunFunction
  /**
   * Resolved config options for this environment. Options at the server
   * global scope are taken as defaults for all enviroments, and can
   * be overriden (resolve conditions, external, optimizedDeps)
   */
  config: ResolvedEnvironmentConfig

  constructor({ name, hot, run, config }: ModuleEnvironmentOptions)

  /**
   * Resolve the URL to an id, load it, and process the code using the
   * plugins pipeline. The module graph is also updated.
   */
  async transformRequest(url: string)

  /**
   * Register a request to be processed with low priority. This is useful
   * to avoid waterfalls. The Vite server has information about the imported
   * modules by other requests, so it can warmup the module graph so the
   * modules are already processed when they are requested.
   */
  async warmupRequest(url: string)

  /**
   * Fetch information about a module from the module runner without running it.
   */
  async fetch(url: string)
}
```

An environment instance in the Vite server let's you process a URL using the `environment.transformRequest(url)` method. This function will use the plugin pipeline to resolve the `url` to a module `id`, load it (reading the file from the file system or through a plugin that implements a virtual module), and then transform the code. While transforming the module, imports and othe metadata will be recorded in the environment module graph by creating or updating the corresponding module node. When processing is done, the transform result is also stored in the module.

But the environment instance can't execute the code itself, as the runtime where the module will be run could be different from the one the Vite server is running in. This is the case for the browser environment. When a html is loaded in the browser, its scripts are executed triggering the evaluation of the entire static module graph. Each imported URL generates a request to the Vite server to get the module code, that ends up handled by the Transform Middleware by calling `server.environment('browser').transformRequest(url)`. The connection between the environment instance in the server and the module runner in the browser is carried out through HTTP in this case.

:::info transformRequest naming
We are using `transformRequest(url)` and `warmupRequest(url)` in the current version of this proposal so it is easier to discuss and understand for users used to Vite's current API. Before releasing, we can take the opportunity to review these names too. For example, it could be named `environment.processModule(url)` or `environment.loadModule(url)` taking a page from Rollup's `context.load(id)` in plugin hooks. For the moment, we think keeping the current names and delaying this discussion is better.
:::

For the default SSR environment, Vite creates a module runner that implements evaluation using `new AsyncFunction` running in the same runtime as the server. This runner is an instance of `ModuleRunner` that exposes:

```ts
class ModuleRunner {
  /**
   * URL to execute. Accepts file path, server path, or id relative to the root.
   * Returns an instantiated module (same as in ssrLoadModule)
   */
  public async executeUrl(url: string): Promise<Record<string, any>>
  /**
   * Entry point URL to execute. Accepts file path, server path or id relative to the root.
   * In the case of a full reload triggered by HMR, this is the module that will be reloaded.
   * If this method is called multiple times, all entry points will be reloaded one at a time.
   * Returns an instantiated module
   */
  public async executeEntrypoint(url: string): Promise<Record<string, any>>
  /**
   * Other ModuleRunner methods...
   */
```

The SSR module runner instance is accessible through `server.ssrModuleRunner`. It isn't part of its associated environment instance because, as we explained before, other environments' module runners could live in a different runtime (for example for the browser, a module runner in a worker thread as used in Vitest, or an edge runtime like workerd). The communication between `server.ssrModuleRunner` and `server.environment('ssr')` is implemented through direct function calls. Given a Vite server configured in middleware mode as described by the [SSR setup guide](#setting-up-the-dev-server), let's implement the SSR middleware using the environment API. Error handling is omitted.

```js
app.use('*', async (req, res, next) => {
  const url = req.originalUrl

  // 1. Read index.html
  let template = fs.readFileSync(path.resolve(__dirname, 'index.html'), 'utf-8')

  // 2. Apply Vite HTML transforms. This injects the Vite HMR client,
  //    and also applies HTML transforms from Vite plugins, e.g. global
  //    preambles from @vitejs/plugin-react
  template = await server.transformIndexHtml(url, template)

  // 3. Load the server entry. executeEntryPoint(url) automatically transforms
  //    ESM source code to be usable in Node.js! There is no bundling
  //    required, and provides full HMR support.
  const { render } = await server.ssrModuleRunner.executeEntryPoint(
    '/src/entry-server.js',
  )

  // 4. render the app HTML. This assumes entry-server.js's exported
  //     `render` function calls appropriate framework SSR APIs,
  //    e.g. ReactDOMServer.renderToString()
  const appHtml = await render(url)

  // 5. Inject the app-rendered HTML into the template.
  const html = template.replace(`<!--ssr-outlet-->`, appHtml)

  // 6. Send the rendered HTML back.
  res.status(200).set({ 'Content-Type': 'text/html' }).end(html)
})
```

:::info ssrModuleRunner vs ssrEnvironment

An alternative is to expose `server.ssrEnvironment` that implements both `ModuleExecutionEnvironment` and `ModuleRunner` as in this case they are both in the same runtime. In this case, the code above would be:

```js
const { render } = await server.ssrEnvironment.executeEntryPoint(
  '/src/entry-server.js',
)
```

This may be an interesting idea if we would like to move other SSR related methods like `ssrFixStacktrace` from the server to the `ssrEnvironment`. And we could also have `server.browserEnvironment` that extends the environment with browser specific functionality.

For this proposal, `server.ssrModuleRunner` is used to highlight the separation between the Environment instance in the server and the module runner instance even if they are both available in the server for SSR.

:::

## Plugins and environments

The Vite server has a shared plugin pipeline, but when a module is processed it is always done in the context of a given environment.

### The `hotUpdate` hook

- **Type:** `(ctx: HotContext) => Array<ModuleNode> | void | Promise<Array<ModuleNode> | void>`
- **See also:** [HMR API](./api-hmr)

The `hotUpdate` hook allows plugins to perform custom HMR update handling for a given environment. When a file changes, the HMR algorithm is run for each environment in serie according to the order in `server.environments`, so the `hotUpdate` hook will be called multiple times. The hook receives a context object with the following signature:

```ts
interface HotContext {
  file: string
  timestamp: number
  environment: string
  modules: Array<ModuleNode>
  read: () => string | Promise<string>
  server: ViteDevServer
}
```

- `environment` is the module execution environment where a file update is currently being processed.

- `modules` is an array of modules in this environment that are affected by the changed file. It's an array because a single file may map to multiple served modules (e.g. Vue SFCs).

- `read` is an async read function that returns the content of the file. This is provided because on some systems, the file change callback may fire too fast before the editor finishes updating the file and direct `fs.readFile` will return empty content. The read function passed in normalizes this behavior.

The hook can choose to:

- Filter and narrow down the affected module list so that the HMR is more accurate.

- Return an empty array and perform a full reload:

  ```js
  hotUpdate({ environment, modules, timestamp }) {
    if (environment !== 'browser')
      return

    server.environment(environment).hot.send({ type: 'full-reload' })
    // Invalidate modules manually
    const invalidatedModules = new Set()
    for (const mod of modules) {
      server.environment(environment).moduleGraph.invalidateModule(
        mod,
        invalidatedModules,
        timestamp,
        true
      )
    }
    return []
  }
  ```

- Return an empty array and perform complete custom HMR handling by sending custom events to the client:

  ```js
  hotUpdate({ environment }) {
    if (environment !== 'browser')
      return

    server.environment(environment).hot.send({
      type: 'custom',
      event: 'special-update',
      data: {}
    })
    return []
  }
  ```

  Client code should register corresponding handler using the [HMR API](./api-hmr) (this could be injected by the same plugin's `transform` hook):

  ```js
  if (import.meta.hot) {
    import.meta.hot.on('special-update', (data) => {
      // perform custom update
    })
  }
  ```

### Using `apply`

In the examples of the previous section, we used a guard in the `hotUpdate` hook to only process updates from the browser environment. If a plugin is specific to only some of the available environments, `apply` can be used to avoid guarding each hook and improve performance.

```js
function ssrOnlyPlugin() {
  return {
    name: 'ssr-only-plugin',
    apply({ environment }) => environment === 'ssr',
    // unguarded hooks...
  }
}
```

### Accessing the current environment in hooks

The `environment` instance is passed as a parameter to the `options` of `resolveId`, `load`, and `transform`. It can also be accessed using the plugin hook context by `this.environment`.

A plugin could use the `environment` instance to:

- Only apply logic for certain environments.
- Change the way they work depending the configuration for the environment, that can be accessed using `environment.config`. The vite core resolve plugin modifies the way it resolves ids based on `environment.config.resolve.conditions` for example.

:::info environment in hooks

We could also pass an environment object that has `{ name, config }`. In a previous iteration the environment instance was passed directly as a parameter but we also need to pass the `environment` name during build time to hooks.

:::

## Environment Configuration

All environments share the Vite server configuration, but there are certain options that can be overriden per environment.

```js
export default {
  resolve: {
    conditions: [] // shared by all environments
  },
  environment: {
    browser: {
      resolve: {
        conditions: [] // override for the browser environment
      }
    }
    ssr: {
      optimizeDeps: {} // override for the ssr environment
    },
    workerd: {
      noExternal: true // override for a third party environment
    }
  }
}
```

The subset of options that can be overriden are resolved and are available at `environment.config`.

:::info What options can be overriden?

Vite has already allowed defining some config for the [SSR environment](https://vitejs.dev/config/ssr-options.html). Initially, these are the options that would be available to be overriden by any environment. Except for `ssr.target: 'node' | 'webworker'`, that could be deprecated as `webworker` could be implemented as a separate environment.

We could discuss what other options we should allow to be overriden, although maybe it is better to add them when there are requested by users later on. Some examples: `define`, `resolve.alias`, `resolve.dedupe`, `resolve.mainFields`, `resolve.extensions`.

:::

## Separate module graphs

Vite currently has a mixed browser and ssr module graph. Given an unprocessed or invalidated node, it isn't possible to know if it corresponds to the browser, ssr, or both environments. Module nodes have some properties prefixed, like `clientImportedModules` and `ssrImportedModules` (and `importedModules` that returns the union of both). `importers` contains all importers from both the browser and ssr environment for each module node. A module node also have `transformResult` and `ssrTransformResult`.

In this proposal, each environment has its own module graph (and a backward compatibility layer will be implemented to give time to the ecosystem to migrate). All module graphs have the same signature, so generic algorithms can be implemented to crawl or query the graph without depending on the environment. `hotUpdate` is a good example. When a file is modified, the module graph of each environment will be used to discovered the affected modules and perform HMR for each environment independendently.

Each module is represented by a `ModuleNode` instance. Modules may be registered in the graph without yet being processed (`transformResult` would be `null` in that case). `importers` and `importedModules` are also updated after the module is processed.

```ts
class ModuleNode {
  environment: string

  url: string
  id: string | null = null
  file: string | null = null

  type: 'js' | 'css'

  importers = new Set<ModuleNode>()
  importedModules = new Set<ModuleNode>()
  importedBindings: Map<string, Set<string>> | null = null

  info?: ModuleInfo
  meta?: Record<string, any>
  transformResult: TransformResult | null = null

  acceptedHmrDeps = new Set<ModuleNode>()
  acceptedHmrExports: Set<string> | null = null
  isSelfAccepting?: boolean
  lastHMRTimestamp = 0
  lastInvalidationTimestamp = 0
}
```

`environment.moduleGraph` is an instance of `ModuleGraph`:

```ts
export class ModuleGraph {
  environment: string

  urlToModuleMap = new Map<string, ModuleNode>()
  idToModuleMap = new Map<string, ModuleNode>()
  etagToModuleMap = new Map<string, ModuleNode>()
  fileToModulesMap = new Map<string, Set<ModuleNode>>()

  constructor(
    environment: string,
    resolveId: (url: string) => Promise<PartialResolvedId | null>,
  )

  async getModuleByUrl(rawUrl: string): Promise<ModuleNode | undefined>

  getModulesByFile(file: string): Set<ModuleNode> | undefined

  onFileChange(file: string): void

  invalidateModule(
    mod: ModuleNode,
    seen: Set<ModuleNode> = new Set(),
    timestamp: number = Date.now(),
    isHmr: boolean = false,
  ): void

  invalidateAll(): void

  async updateModuleInfo(
    mod: ModuleNode,
    importedModules: Set<string | ModuleNode>,
    importedBindings: Map<string, Set<string>> | null,
    acceptedModules: Set<string | ModuleNode>,
    acceptedExports: Set<string> | null,
    isSelfAccepting: boolean,
  ): Promise<Set<ModuleNode> | undefined>

  async ensureEntryFromUrl(
    rawUrl: string,
    setIsSelfAccepting = true,
  ): Promise<ModuleNode>

  createFileOnlyEntry(file: string): ModuleNode

  async resolveUrl(url: string): Promise<ResolvedUrl>

  updateModuleTransformResult(
    mod: ModuleNode,
    result: TransformResult | null,
  ): void

  getModuleByEtag(etag: string): ModuleNode | undefined
}
```

## Registering environments

There is a new plugin hook called `registerEnvironment` that is called after `configResolved`:

```js
function workedPlugin() {
  return {
    name: 'vite-plugin-workerd',
    registerEnvironment(server, environments) {
      const workedEnvironment = new WorkerdEnvironment(server)
      // This environment has 'workerd' as its name, a convention agreed upon by the ecosystem
      // connect workerdEnviroment logic to its associated workerd module runner
      environments.push(workedEnvironment)
    },
  }
}
```

The environment will be accesible in middlewares or plugin hooks through `server.environment('workerd')`.

## Creating new environments

One of the goals of this feature is to provide a customizable API to process and run code. A Vite dev server provides browser and ssr environments out of the box, but users can build new environments using the exposed primitives.

```ts
import { createModuleExectutionEnvironment } from 'vite'

const environment = createModuleExecutionEnvironment({
  name: 'workerd',
  hot: null,
  config: {
    resolve: { conditions: ['custom'] }
  },
  run(url) {
    return rpc().runModule(url)
  }
}) => ViteModuleExecutionEnvironment
```

## `ModuleRunner`

A module runner is instantiated in the target runtime. All APIs in the next section are imported from `vite/module-runner` unless stated otherwise. This export entry point is kept as lightweight as possible, only exporting the minimal needed to create runners in the

**Type Signature:**

```ts
export class ModuleRunner {
  constructor(
    public options: ModuleRunnerOptions,
    public evaluator: ModuleEvaluator,
    private debug?: ModuleRunnerDebugger,
  ) {}
  /**
   * URL to execute. Accepts file path, server path, or id relative to the root.
   */
  public async executeUrl<T = any>(url: string): Promise<T>
  /**
   * Entry point URL to execute. Accepts file path, server path or id relative to the root.
   * In the case of a full reload triggered by HMR, this is the module that will be reloaded.
   * If this method is called multiple times, all entry points will be reloaded one at a time.
   */
  public async executeEntrypoint<T = any>(url: string): Promise<T>
  /**
   * Clear all caches including HMR listeners.
   */
  public clearCache(): void
  /**
   * Clears all caches, removes all HMR listeners, and resets source map support.
   * This method doesn't stop the HMR connection.
   */
  public async destroy(): Promise<void>
  /**
   * Returns `true` if the runtime has been destroyed by calling `destroy()` method.
   */
  public isDestroyed(): boolean
}
```

The module evaluator in `ModuleRunner` is responsible for executing the code. Vite exports `ESModulesEvaluator` out of the box, it uses `new AsyncFunction` to evaluate the code. You can provide your own implementation if your JavaScript runtime doesn't support unsafe evaluation.

The two main methods that a module runner exposes are `executeUrl` and `executeEntrypoint`. The only difference between them is that all modules executed by `executeEntrypoint` will be reexecuted if HMR triggers `full-reload` event. Be aware that Vite module runners don't update the `exports` object when this happens, and instead it overrides it. You would need to run `executeUrl` or get the module from the `moduleCache` again if you rely on having the latest `exports` object.

**Example Usage:**

```js
import { ModuleRunner, ESModulesEvaluator } from 'vite/environment'
import { root, fetchModule } from './rpc-implementation.js'

const moduleRunner = new ModuleRunner(
  {
    root,
    fetchModule,
    // you can also provide hmr.connection to support HMR
  },
  new ESModulesEvaluator(),
)

await moduleRunner.executeEntrypoint('/src/entry-point.js')
```

## `ModuleRunnerOptions`

```ts
export interface ModuleRunnerOptions {
  /**
   * Root of the project
   */
  root: string
  /**
   * A method to get the information about the module.
   */
  fetchModule: FetchFunction
  /**
   * Configure how source maps are resolved. Prefers `node` if `process.setSourceMapsEnabled` is available.
   * Otherwise it will use `prepareStackTrace` by default which overrides `Error.prepareStackTrace` method.
   * You can provide an object to configure how file contents and source maps are resolved for files that were not processed by Vite.
   */
  sourcemapInterceptor?:
    | false
    | 'node'
    | 'prepareStackTrace'
    | InterceptorOptions
  /**
   * Disable HMR or configure HMR options.
   */
  hmr?:
    | false
    | {
        /**
         * Configure how HMR communicates between the client and the server.
         */
        connection: HMRRuntimeConnection
        /**
         * Configure HMR logger.
         */
        logger?: false | HMRLogger
      }
  /**
   * Custom module cache. If not provided, it creates a separate module cache for each ViteRuntime instance.
   */
  moduleCache?: ModuleCacheMap
}
```

## `ModuleEvaluator`

**Type Signature:**

```ts
export interface ModuleEvaluator {
  /**
   * Evaluate code that was transformed by Vite.
   * @param context Function context
   * @param code Transformed code
   * @param id ID that was used to fetch the module
   */
  evaluateModule(
    context: ModuleRunnerContext,
    code: string,
    id: string,
  ): Promise<any>
  /**
   * evaluate externalized module.
   * @param file File URL to the external module
   */
  evaluateExternalModule(file: string): Promise<any>
}
```

Vite exports `ESModulesEvaluator` that implements this interface by default. It uses `new AsyncFunction` to evaluate code, so if the code has inlined source map it should contain an [offset of 2 lines](https://tc39.es/ecma262/#sec-createdynamicfunction) to accommodate for new lines added. This is done automatically in the server SSR environment. If your runner implementation doesn't have this constraint, you should use `fetchModule` (exported from `vite`) directly.

## HMRModuleRunnerConnection

**Type Signature:**

```ts
export interface HMRModuleRunnerConnection {
  /**
   * Checked before sending messages to the client.
   */
  isReady(): boolean
  /**
   * Send message to the client.
   */
  send(message: string): void
  /**
   * Configure how HMR is handled when this connection triggers an update.
   * This method expects that connection will start listening for HMR updates and call this callback when it's received.
   */
  onUpdate(callback: (payload: HMRPayload) => void): void
}
```

This interface defines how HMR communication is established. Vite exports `ServerHMRConnector` from the main entry point to support HMR during Vite SSR. The `isReady` and `send` methods are usually called when the custom event is triggered (like, `import.meta.hot.send("my-event")`).

`onUpdate` is called only once when the new runtime is initiated. It passed down a method that should be called when connection triggers the HMR event. The implementation depends on the type of connection (as an example, it can be `WebSocket`/`EventEmitter`/`MessageChannel`), but it usually looks something like this:

```js
function onUpdate(callback) {
  this.connection.on('hmr', (event) => callback(event.data))
}
```

The callback is queued and it will wait for the current update to be resolved before processing the next update. Unlike the browser implementation, HMR updates in Vite Runtime wait until all listeners (like, `vite:beforeUpdate`/`vite:beforeFullReload`) are finished before updating the modules.

## Backward Compatibility

The current Vite server API will be deprecated but keep working during the next major.

- `server.transformRequest(url)` <br> -> `server.environment('browser').transformRequest(url)`
- `server.transformRequest(url, { ssr: true })` <br> -> `server.environment('ssr').tranformRequest(url)`
- `server.warmupRequest(url)` <br> -> `server.environment('browser').warmupRequest(url)`
- `server.ssrLoadModule(url)` <br> -> `server.ssrModuleRunner.executeEntryPoint(url)`
- `server.moduleGraph` <br> -> `server.environment(name).moduleGraph`
- `handleHotUpdate` <br> -> `hotUpdate`

Maybe we could also have:

- `server.open(url)` <br> -> `server.environment('browser').run(url)`

The `server.moduleGraph` will keep returning a mixed view of the browser and ssr module graphs. Proxy module nodes will be returned so all functions keep returning mixed module nodes. The same scheme is used for the module nodes passed to `handleHotUpdate`. This is the most difficult change to get right regarding backward compatibility. We may need to accept small breaking changes when we release the API in Vite 6, making it opt in until then when releasing the API as experimental in Vite 5.2.

## Open Questions

There are some open questions and alternative as info boxes interlined in the guide.

Names for concepts and the API are the best we could currently find, what we should keep discussing before releasing if we end up adopting this proposal.