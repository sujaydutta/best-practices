# Feature Flagging Best Practices

> *Java · .NET · Python · Angular · React · Flipt · flagd · OpenFeature*

---

## Table of Contents

1. [Introduction & Strategic Context](#1-introduction--strategic-context)
2. [Core Architecture Principles](#2-core-architecture-principles)
3. [Patterns & Anti-Patterns](#3-patterns--anti-patterns)
4. [Java](#4-java)
5. [.NET](#5-net-c)
6. [Python](#6-python)
7. [Angular](#7-angular)
8. [React](#8-react)
9. [Open Source Self-Hosted Tools](#9-open-source-self-hosted-tools)
10. [Operational & Security Guidance](#10-operational--security-guidance)
11. [Governance & Team Standards](#11-governance--team-standards)
12. [Quick Reference](#12-quick-reference)

---

## 1. Introduction & Strategic Context

Feature flagging — also called feature toggles or feature switches — is one of the most powerful techniques in modern software delivery. When implemented correctly, flags decouple deployment from release, enabling continuous delivery, controlled rollouts, experimentation, and instant kill-switch capability.

This guide defines the architectural standards, patterns, anti-patterns, and platform-specific implementation guidance for feature flags across all technology stacks used in our organization.

> **💡 WHY IT MATTERS:** Teams with mature feature flag practices deploy 200× more frequently and recover from incidents 24× faster (DORA State of DevOps). Flags are not a testing tool — they are a delivery architecture decision.

### 1.1 Core Benefits

- Trunk-based development without long-lived feature branches
- Progressive delivery: canary, ring-based, percentage-based rollouts
- A/B testing and experimentation with statistical rigor
- Instant kill switches for high-risk changes without a rollback deploy
- Operational flags for circuit-breakers and infrastructure toggles
- Permission-based feature entitlements by user, role, or tenant

### 1.2 Flag Taxonomy

Not all flags are equal. Categorize every flag before creating it:

| Type | Purpose & Lifetime |
|---|---|
| **Release Flag** | Hides incomplete features in production. Short-lived — remove within 1–2 sprints of full rollout. |
| **Experiment Flag** | A/B or multivariate testing. Medium-lived — expires when experiment concludes. |
| **Ops Toggle** | Circuit-breaker, kill-switch, load shedding. Permanent infrastructure controls. |
| **Permission Flag** | Entitlement by user role, plan tier, or tenant. Long-lived but should be modeled as entitlements, not hardcoded flags. |
| **Infrastructure Flag** | Switches between implementations (e.g., new DB, cache strategy). Short-lived during migrations. |

---

## 2. Core Architecture Principles

### 2.1 Use a Centralized Flag Service

Never scatter flag definitions in config files, environment variables, or code constants. A centralized flag service provides:

- Single source of truth for flag states across all services and environments
- Real-time flag updates without redeployment (push or polling)
- Audit log of who changed what and when
- Targeting rules, percentage rollouts, and user segmentation in one place

> **✅ RECOMMENDED:** Evaluate LaunchDarkly, Unleash (self-hosted), Flagsmith, Flipt, or flagd for open-source/self-hosted deployments. For greenfield projects, OpenFeature's vendor-neutral SDK standard should be the default choice — it lets you swap providers without rewriting application code. See [Section 9](#9-open-source-self-hosted-tools) for a deep dive on Flipt and flagd.

### 2.2 The Flag Evaluation Contract

Flag evaluation must be: (1) fast and non-blocking, (2) fault-tolerant with safe defaults, and (3) side-effect free. A flag check must never make a network call in the hot path without a local cache.

```
// Pseudocode — universal flag evaluation contract
evaluate(flagKey, context, defaultValue):
  1. Check local in-memory cache (< 1ms)
  2. If cache miss or stale → async refresh from flag service
  3. On SDK error / timeout → return defaultValue (fail safe)
  4. Log evaluation event for analytics (non-blocking, async)
  5. Return boolean | string | number | JSON variant
```

### 2.3 Flag Hygiene: The Flag Lifecycle

Flags that outlive their purpose become technical debt. Enforce a lifecycle:

1. **Define flag:** document type, owner, expiry date, and safe default in flag registry.
2. **Gate the feature:** implement the flag check with a safe fallback.
3. **Roll out:** use targeting rules, percentage, or ring-based progression.
4. **Clean up:** once at 100% or retired, remove the flag and all conditional branches within one sprint.

> **⚠️ ANTI-PATTERN:** Flags with no expiry date, no owner, and no cleanup ticket are the #1 source of flag debt. Every flag must have a JIRA/GitHub issue for cleanup at creation time.

### 2.4 Context & Targeting

Flag evaluation context should always be explicit and structured. Pass a context object — never rely on global/thread-local state for user identity:

```
// Universal flag context structure
FlagContext {
  userId:      string    // stable unique identifier
  email:       string    // for email-based targeting
  tenantId:    string    // for B2B multi-tenant targeting
  environment: string    // prod | staging | dev
  appVersion:  string    // semver for version-based rollouts
  attributes:  Map<string, any>  // arbitrary custom attributes
}
```

### 2.5 Observability Requirements

Every flag evaluation should emit a structured log event. Aggregate these into a dashboard tracking:

- Evaluation volume per flag, per variant, per time window
- Default fallback rate (spikes indicate SDK or service issues)
- Flag change events correlated with error rate and latency charts
- Stale flag detection: flags with no evaluations in 30+ days

---

## 3. Patterns & Anti-Patterns

### 3.1 Recommended Patterns

#### Strangler Fig with Flags

When migrating a legacy system, wrap the old and new paths in a flag. Roll the flag forward incrementally, monitoring error rates at each step.

```javascript
if (flags.isEnabled('use-new-payment-engine', ctx)) {
  return newPaymentEngine.process(order);
} else {
  return legacyPaymentEngine.process(order);   // safe default
}
```

#### Dark Launch / Shadow Mode

Run both old and new implementations simultaneously. The new path's result is discarded but its latency, errors, and correctness are recorded. Roll forward only when metrics satisfy SLOs.

#### Ring-Based Rollout

Roll flags through deployment rings in sequence: internal employees → beta users → 1% → 5% → 25% → 100%. Define automated roll-forward gates based on error rate thresholds.

### 3.2 Anti-Patterns to Avoid

| Anti-Pattern | Why It's Dangerous & What to Do Instead |
|---|---|
| **Flag in hot loop** | Checking a flag 10,000×/sec without caching causes latency spikes. Cache evaluations per request, not per call. |
| **Nested flag logic** | `if(flagA && flagB \|\| flagC)` becomes unmaintainable. Collapse to a single flag or use a configuration object. |
| **Hardcoded user lists** | Embedding user IDs in code is not targeting — it's a bug waiting to happen. Use the flag service's targeting rules. |
| **Omitting safe defaults** | `evaluate('flag')` with no default will throw on SDK errors. Always provide a safe default matching the old behavior. |
| **Permanent release flags** | Flags that ship forever carry ongoing evaluation overhead and hidden complexity. Set and enforce expiry dates. |
| **Testing flags with env vars** | Environment variables require a redeploy to change. Use a real flag service even in test environments. |

---

## 4. Java

### 4.1 Recommended Libraries

- OpenFeature Java SDK (vendor-neutral, recommended for new projects)
- LaunchDarkly Java Server SDK 6.x
- Unleash Client Java
- FF4J (self-hosted, Spring-native)

### 4.2 Spring Boot Integration

Wrap the flag client as a `@Bean` and inject it. Never instantiate SDK clients in service methods — they are expensive and must be singletons.

```java
@Configuration
public class FeatureFlagConfig {
    @Bean
    public LDClient launchDarklyClient(@Value("${ld.sdk.key}") String sdkKey) {
        LDConfig config = new LDConfig.Builder()
            .events(Components.sendEvents().flushIntervalMillis(5000))
            .build();
        return new LDClient(sdkKey, config);
    }
}

@Service
public class CheckoutService {
    private final LDClient ldClient;

    public CheckoutService(LDClient ldClient) {
        this.ldClient = ldClient;
    }

    public Order processOrder(Order order, User user) {
        LDContext ctx = LDContext.builder(user.getId())
            .set("email", user.getEmail())
            .set("plan",  user.getPlan())
            .build();

        boolean useNewEngine = ldClient.boolVariation(
            "new-checkout-engine", ctx, false);  // safe default = false

        return useNewEngine
            ? newCheckout.process(order)
            : legacyCheckout.process(order);
    }
}
```

### 4.3 Java-Specific Guidance

- **Shutdown hook:** call `ldClient.close()` on application stop to flush events
- **Use `LDContext` (v6+)** over deprecated `LDUser` — it supports multi-context linking
- **Thread safety:** SDK clients are thread-safe; do not create per-request instances
- **Testing:** inject a mock or use the `TestData` data source from the SDK for unit tests
- **GraalVM / native image:** verify SDK compatibility; some reflection configs are needed

```java
// Unit testing with LaunchDarkly TestData source
TestData td = TestData.dataSource();
td.update(td.flag("new-checkout-engine").variationForAll(true));
LDClient testClient = new LDClient("sdk-key", new LDConfig.Builder()
    .dataSource(td).events(Components.noEvents()).build());
```

---

## 5. .NET (C#)

### 5.1 Recommended Libraries

- OpenFeature .NET SDK (vendor-neutral standard)
- LaunchDarkly .NET Server SDK 7.x
- Microsoft.FeatureManagement (built-in for ASP.NET Core)
- Unleash Client .NET

### 5.2 ASP.NET Core with Microsoft.FeatureManagement

For teams on ASP.NET Core, `Microsoft.FeatureManagement` integrates directly with the configuration system and supports filters (time window, browser, percentage, custom).

```csharp
// Program.cs / Startup.cs
builder.Services.AddFeatureManagement()
    .AddFeatureFilter<PercentageFilter>()
    .AddFeatureFilter<TimeWindowFilter>();

// appsettings.json
"FeatureManagement": {
  "NewDashboard": true,
  "BetaSearch": {
    "EnabledFor": [{ "Name": "Percentage", "Parameters": { "Value": 25 } }]
  }
}

// Controller usage
[HttpGet("/dashboard")]
public async Task<IActionResult> Dashboard(
    [FromServices] IFeatureManager featureManager)
{
    if (await featureManager.IsEnabledAsync("NewDashboard"))
        return View("NewDashboard");
    return View("Dashboard");
}

// Attribute-based flag gating on controller actions
[FeatureGate("BetaSearch")]
public IActionResult SearchBeta() => View();
```

### 5.3 .NET-Specific Guidance

- **`IFeatureManager`** is scoped per request — safe for user-context evaluation
- **Use `IFeatureManagerSnapshot`** when you need consistent evaluation within a single request scope
- **For LaunchDarkly:** register `LdClient` as singleton via DI; do not use static instances
- **OpenTelemetry:** emit custom spans for flag evaluations to trace flag impact on latency
- **Blazor/WASM:** flag evaluation must target the server; do not expose server-side SDK keys to the browser

---

## 6. Python

### 6.1 Recommended Libraries

- OpenFeature Python SDK
- LaunchDarkly Python Server SDK
- Flagsmith Python SDK
- Unleash Client Python

### 6.2 FastAPI / Django Integration

In Python services, initialize the SDK client once at application startup and share it via dependency injection or application state — not as a module-level global (which causes issues with async frameworks and testing).

```python
# FastAPI example — lifespan context for SDK lifecycle
from contextlib import asynccontextmanager
from fastapi import FastAPI, Depends
import ldclient
from ldclient.config import Config

@asynccontextmanager
async def lifespan(app: FastAPI):
    ldclient.set_config(Config(sdk_key=settings.LD_SDK_KEY))
    app.state.ld = ldclient.get()
    yield
    app.state.ld.close()   # flush events on shutdown

app = FastAPI(lifespan=lifespan)

def get_flag_client(request: Request):
    return request.app.state.ld

@app.get("/checkout")
def checkout(user_id: str, ld=Depends(get_flag_client)):
    ctx = Context.builder(user_id).build()
    use_new = ld.variation('new-checkout', ctx, False)
    return new_engine.run() if use_new else legacy_engine.run()
```

### 6.3 Python-Specific Guidance

- **Avoid module-level SDK initialization** — it runs at import time, making mocking in tests difficult
- **For async Django/FastAPI:** use async-compatible SDK or offload evaluations to a thread pool
- **Testing:** use the `TestData` data source or mock the `variation()` method with `unittest.mock`
- **Type annotations:** wrap flag values in typed helpers to avoid bool/string confusion across teams
- **Celery workers:** re-initialize the SDK within the worker process, not only in the main process

```python
# Typed flag helper pattern (prevents silent type errors)
def flag_bool(client, key: str, ctx, default: bool = False) -> bool:
    result = client.variation(key, ctx, default)
    if not isinstance(result, bool):
        logger.error('Flag %s returned non-bool: %r', key, result)
        return default
    return result
```

---

## 7. Angular

### 7.1 Recommended Approach

- OpenFeature Web SDK with Angular wrapper
- LaunchDarkly Angular SDK (wraps js-client-sdk)
- Flagsmith JavaScript SDK
- Custom lightweight service wrapping your backend flag proxy (preferred for security)

> **🔐 SECURITY CRITICAL:** Never use a server-side SDK key in Angular (browser). Use client-side environment keys only, which are designed to be public. Never expose flag targeting rules or user PII through the client SDK.

### 7.2 Angular Service Pattern

Encapsulate flag logic in a singleton Angular service. Use RxJS `BehaviorSubject` to reactively propagate flag state changes to all subscribers without polling.

```typescript
@Injectable({ providedIn: 'root' })
export class FeatureFlagService implements OnDestroy {
  private client: LDClient;
  private flags$ = new BehaviorSubject<Record<string, unknown>>({});

  constructor(private authService: AuthService) {
    const context: LDContext = {
      kind: 'user',
      key: this.authService.userId,
      email: this.authService.email,
    };
    this.client = initialize(environment.ldClientId, context);
    this.client.on('ready', () => this.syncFlags());
    this.client.on('change', () => this.syncFlags());
  }

  private syncFlags() {
    this.flags$.next(this.client.allFlags());
  }

  isEnabled(key: string, defaultValue = false): Observable<boolean> {
    return this.flags$.pipe(
      map(flags => (flags[key] as boolean) ?? defaultValue),
      distinctUntilChanged()
    );
  }

  ngOnDestroy() { this.client.close(); }
}

// Usage in component
export class CheckoutComponent {
  showNewCheckout$ = this.flags.isEnabled('new-checkout-ui');
  constructor(private flags: FeatureFlagService) {}
}

// Template
// <app-new-checkout *ngIf="showNewCheckout$ | async; else legacyTpl">
```

### 7.3 Angular-Specific Guidance

- **Use `ngIf` / `@if` for structural guards** — avoid hiding elements with CSS for gated features
- **Route guards:** implement `CanActivate` using flag service to block access to unreleased routes
- **SSR (Angular Universal):** evaluate flags server-side and embed in HTML to prevent flash-of-wrong-content
- **Change detection:** use `async` pipe to avoid manual subscriptions and memory leaks
- **Testing:** provide a mock `FeatureFlagService` in `TestBed` with preset `BehaviorSubject` values

---

## 8. React

### 8.1 Recommended Approach

- OpenFeature Web React SDK (vendor-neutral)
- LaunchDarkly React SDK (wraps js-client-sdk)
- Flagsmith React SDK
- React Context + custom hook wrapping a backend proxy flag endpoint

### 8.2 Context + Custom Hook Pattern

Expose flags via React Context at the app root and consume them with a typed hook. This enables clean component logic and straightforward testing with a mock provider.

```tsx
// FeatureFlagProvider.tsx
const FlagContext = createContext<Record<string, unknown>>({});

export function FeatureFlagProvider({ children }: { children: ReactNode }) {
  const { user } = useAuth();
  const [flags, setFlags] = useState<Record<string, unknown>>({});

  useEffect(() => {
    if (!user) return;
    const client = initialize(process.env.REACT_APP_LD_KEY!, {
      kind: 'user', key: user.id, email: user.email
    });
    client.waitForInitialization().then(() => setFlags(client.allFlags()));
    client.on('change', () => setFlags(client.allFlags()));
    return () => { client.close(); };
  }, [user?.id]);

  return <FlagContext.Provider value={flags}>{children}</FlagContext.Provider>;
}

// useFlag hook
export function useFlag<T>(key: string, defaultValue: T): T {
  const flags = useContext(FlagContext);
  return (flags[key] as T) ?? defaultValue;
}

// Component usage
function Checkout() {
  const showNewUI = useFlag('new-checkout-ui', false);
  return showNewUI ? <NewCheckout /> : <LegacyCheckout />;
}
```

### 8.3 React-Specific Guidance

- **Wrap the provider high in the tree** (app root) so flags are available on first render
- **For Next.js:** evaluate flags in `getServerSideProps` or Middleware and pass as props to prevent layout shift
- **React Query / SWR:** cache flag state server-side and revalidate on user identity change
- **Testing:** replace `FlagContext.Provider` with preset values in test wrappers — no SDK calls needed
- **Avoid re-renders:** memoize flag-derived values with `useMemo` when they drive expensive computations

```tsx
// Test wrapper — no SDK required
function renderWithFlags(ui: ReactElement, flags: Record<string, unknown>) {
  return render(
    <FlagContext.Provider value={flags}>{ui}</FlagContext.Provider>
  );
}

test('shows new checkout when flag enabled', () => {
  renderWithFlags(<Checkout />, { 'new-checkout-ui': true });
  expect(screen.getByTestId('new-checkout')).toBeInTheDocument();
});
```

---

---

## 9. Open Source Self-Hosted Tools

For teams that require full data sovereignty, no per-seat licensing, or GitOps-native flag management, open source self-hosted tools are a strong alternative to SaaS platforms. The two most architecturally significant options are **Flipt** and **flagd**.

### 9.1 Flipt

<https://flipt.io> · License: GPL-3.0 · Language: Go (single binary)

Flipt is a 100% open source, self-hosted feature flag platform with zero paid tiers. It is built as a single Go binary with no required external dependencies, making it the simplest option to operate. Flipt natively supports a **GitOps workflow** — flag definitions can be stored and versioned in Git repositories alongside application code, with changes flowing automatically into running instances.

**Strengths:**
- Single binary deployment — runs on bare metal, VMs, containers, or Kubernetes
- No seat limits, no paywalled features, no cloud dependency
- Git-native flag storage: flags live in YAML/JSON in your repo, fully auditable via `git log`
- Full OpenFeature provider support across Java, .NET, Python, Node.js, Go, Ruby, and browser JS
- SSO, SAML, and RBAC available for enterprise self-hosted deployments
- Local (in-process) flag evaluation — flags keep working even if the Flipt server is unreachable

**Trade-offs:**
- No built-in experimentation or A/B testing analytics — requires a separate data pipeline
- Smaller contributor base than Unleash or Flagsmith; features iterate more slowly
- Community support only (no paid SLA tier)

**Deployment:**

```bash
# Docker — single container, no external DB needed by default
docker run -d -p 8080:8080 -p 9000:9000 \
  -v $(pwd)/config:/etc/flipt \
  docker.flipt.io/flipt/flipt:latest

# Kubernetes via Helm
helm repo add flipt https://helm.flipt.io
helm install flipt flipt/flipt --namespace feature-flags --create-namespace
```

**GitOps flag definition (YAML in repo):**

```yaml
# flags.yaml — committed to your application repo
flags:
  - key: new-checkout-engine
    name: New Checkout Engine
    type: BOOLEAN_FLAG_TYPE
    enabled: true
    rules:
      - segment: beta-users
        rank: 1
  - key: dark-mode-ui
    name: Dark Mode UI
    type: BOOLEAN_FLAG_TYPE
    enabled: false
```

**Java integration via OpenFeature + Flipt provider:**

```java
// Add dependency: io.flipt:flipt-openfeature-provider-java
FliptProvider provider = new FliptProvider(
    FliptProviderConfig.builder()
        .host("http://flipt.internal:9000")
        .build()
);
OpenFeatureAPI.getInstance().setProvider(provider);
Client client = OpenFeatureAPI.getInstance().getClient();

MutableContext ctx = new MutableContext();
ctx.add("userId", user.getId());
ctx.add("email", user.getEmail());

boolean enabled = client.getBooleanValue("new-checkout-engine", false, ctx);
```

**Python integration via OpenFeature + Flipt provider:**

```python
from openfeature import api
from openfeature.contrib.provider.flipt import FliptProvider

api.set_provider(FliptProvider(address="http://flipt.internal:9000"))
client = api.get_client()

ctx = EvaluationContext(targeting_key=user_id, attributes={"email": email})
enabled = client.get_boolean_value("new-checkout-engine", False, ctx)
```

---

### 9.2 flagd

<https://flagd.dev> · License: Apache-2.0 · CNCF Project · Language: Go

flagd is the **official OpenFeature reference implementation** — a lightweight feature flag evaluation daemon following the Unix philosophy: do one thing well. It has no UI, no management console, and no built-in persistence layer. Instead, it reads flag definitions from one or more configurable **sync sources** (local files, HTTP endpoints, Kubernetes CRDs, or gRPC streams) and exposes evaluation over gRPC and HTTP.

This minimalism is its greatest strength: flagd slots into any existing infrastructure without dictating how you store or manage flags.

**Strengths:**
- Official OpenFeature reference implementation — highest spec compliance
- Multiple sync sources: local file, HTTP, Kubernetes custom resource, gRPC
- Three deployment modes: standalone service, Kubernetes sidecar (via OpenFeature Operator), or in-process evaluation engine embedded directly in the app
- Hot-reloads flag definitions — changes take effect immediately without restart
- Built-in Prometheus metrics on port `8014` and native OpenTelemetry support
- Zero vendor lock-in: pure OpenFeature API in application code

**Trade-offs:**
- No built-in UI or management console — you manage flag JSON/YAML files directly or pair with a UI layer
- No built-in targeting UI: targeting rules are written as JSON Logic expressions
- Requires operational maturity to manage flag files at scale without a dedicated management layer

**Deployment modes:**

```
Mode 1 — Standalone service (all languages call it over gRPC/HTTP)
  [App] ──gRPC──► [flagd process] ──reads──► [flags.json | HTTP | K8s CRD]

Mode 2 — Kubernetes sidecar (injected automatically by OpenFeature Operator)
  Pod: [App container] ──localhost:8013──► [flagd sidecar] ──► [FeatureFlag CRD]

Mode 3 — In-process (flagd evaluation engine embedded; no separate process)
  [App] ──► [flagd-core library] ──reads──► [local flags.json]
```

**Kubernetes deployment with OpenFeature Operator:**

```yaml
# FeatureFlag CRD — flag definitions as Kubernetes resources
apiVersion: core.openfeature.dev/v1beta1
kind: FeatureFlag
metadata:
  name: app-flags
spec:
  flagSpec:
    flags:
      new-checkout-engine:
        state: ENABLED
        variants:
          "true": true
          "false": false
        defaultVariant: "false"
        targeting:
          if:
            - in: ["@beta.example.com", { var: email }]
            - "true"
            - "false"
---
# Annotate your Deployment — flagd sidecar is injected automatically
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout-service
  annotations:
    openfeature.dev/enabled: "true"
    openfeature.dev/featureflagsource: "app-flags"
```

**Flag definition format (JSON Logic targeting rules):**

```json
{
  "flags": {
    "new-checkout-engine": {
      "state": "ENABLED",
      "variants": { "true": true, "false": false },
      "defaultVariant": "false",
      "targeting": {
        "if": [
          { "in": ["@beta.example.com", { "var": "email" }] },
          "true",
          "false"
        ]
      }
    }
  }
}
```

**Java integration (OpenFeature + flagd provider):**

```java
// flagd runs as a sidecar — connect to localhost
FlagdProvider provider = new FlagdProvider(
    FlagdOptions.builder()
        .host("localhost")
        .port(8013)
        .build()
);
OpenFeatureAPI.getInstance().setProvider(provider);
Client client = OpenFeatureAPI.getInstance().getClient();

MutableContext ctx = new MutableContext(user.getId());
ctx.add("email", user.getEmail());

boolean enabled = client.getBooleanValue("new-checkout-engine", false, ctx);
```

**.NET integration (OpenFeature + flagd provider):**

```csharp
// NuGet: OpenFeature.Contrib.Providers.Flagd
var provider = new FlagdProvider(new Uri("http://localhost:8013"));
await OpenFeature.Api.Instance.SetProviderAsync(provider);
var client = OpenFeature.Api.Instance.GetClient();

var ctx = EvaluationContext.Builder()
    .SetTargetingKey(userId)
    .Set("email", userEmail)
    .Build();

bool enabled = await client.GetBooleanValueAsync("new-checkout-engine", false, ctx);
```

**Angular / React integration (flagd web provider):**

```typescript
// flagd-web-provider connects to flagd's HTTP evaluation endpoint
// Run flagd with --metrics-port exposed, or use a backend proxy
import { OpenFeature } from "@openfeature/web-sdk";
import { FlagdWebProvider } from "@openfeature/flagd-web-provider";

await OpenFeature.setProviderAndWait(
  new FlagdWebProvider({ host: "flagd-proxy.internal", port: 8013, tls: true })
);
const client = OpenFeature.getClient();
const enabled = client.getBooleanValue("new-checkout-engine", false);
```

> **⚠️ NOTE FOR FRONTEND:** flagd's gRPC endpoint is not directly browser-accessible. Run flagd behind an Envoy or grpc-web proxy, or use the OpenFeature REST evaluation API (`--metrics-port`) for browser clients. Never expose an internal flagd instance directly to the public internet.

---

### 9.3 Choosing Between Flipt, flagd, and SaaS

| Criterion | Flipt | flagd | SaaS (LaunchDarkly, etc.) |
|---|---|---|---|
| **Setup complexity** | Low (single binary) | Medium (needs a sync source + optional operator) | Low (cloud-managed) |
| **UI / management console** | ✅ Built-in web UI | ❌ None — manage flag files directly | ✅ Full-featured |
| **OpenFeature compliance** | ✅ Full provider support | ✅ Reference implementation | ✅ Varies by vendor |
| **GitOps / flags as code** | ✅ Native Git storage | ✅ File/CRD sync | ⚠️ Varies |
| **Kubernetes-native** | ✅ Helm chart available | ✅ OpenFeature Operator (sidecar injection) | ⚠️ External dependency |
| **Experimentation / A/B** | ❌ Separate tooling needed | ❌ Separate tooling needed | ✅ Often built-in |
| **Audit log** | ✅ Via Git history | ✅ Via Git/CRD history | ✅ Built-in |
| **Cost** | Free (GPL-3.0) | Free (Apache-2.0) | Per-seat / usage-based |
| **Best for** | Teams wanting a full flag platform, self-hosted, with a UI | Cloud-native / K8s shops wanting maximum portability and no vendor coupling | Teams prioritizing experimentation, analytics, and managed ops |


## 10. Operational & Security Guidance

### 10.1 Environment Promotion Strategy

Flags must exist in all environments but with independent states. Never copy a production flag state directly — let each environment progress independently.

| Environment | Flag State Policy |
|---|---|
| **Development** | All flags default ON for feature teams. No targeting rules required. |
| **Staging / QA** | Mirror production flag states. Run full regression with both flag states. |
| **Pre-prod / Canary** | Progressive rollout validation. Enable for 5–10% of traffic. |
| **Production** | Flags start at 0%. Increment through ring policy. Monitor SLOs at each step. |

### 10.2 Security Considerations

- Never include SDK server keys in client-side code, mobile apps, or public repos
- Rotate SDK keys on a regular cadence (quarterly) and immediately after team departures
- Use a backend flag proxy to shield internal targeting rules from the client
- Treat flag state as sensitive config — apply the same access controls as secrets
- Audit flag changes: integrate flag change webhooks with your SIEM or security tooling

### 10.3 Performance Budgets

Flag evaluation overhead must be negligible. Establish and enforce these limits:

| Metric | Budget |
|---|---|
| Single flag evaluation (cached) | < 1 ms (target: < 0.1 ms) |
| SDK initialization time | < 500 ms (async, non-blocking) |
| Flag payload size (all flags) | < 50 KB compressed |
| Polling interval (if not streaming) | 30–60 seconds minimum |
| Max flags per service | < 50 active at any time (enforce via tooling) |

### 10.4 Incident Response with Flags

Feature flags are a first-class incident mitigation tool. For every high-risk feature:

1. Pre-create an ops kill-switch flag before the feature ships to production.
2. Add the kill-switch to the on-call runbook with clear instructions.
3. Test the kill-switch path in staging — confirm the fallback works correctly.
4. During an incident, disable the flag before investigating root cause.
5. Post-incident: analyze flag evaluation data to understand blast radius.

---

## 11. Governance & Team Standards

### 11.1 Flag Registry Requirements

Every flag must be registered in the team's flag registry (JIRA, Notion, or the flag service's built-in metadata) with:

- Flag key, type, and description
- Owner (team and individual engineer)
- Creation date and target removal date
- Linked deployment/feature ticket and cleanup ticket
- Safe default value rationale

### 11.2 Code Review Checklist for Flag PRs

Reviewers must verify the following before approving any PR that adds or modifies a flag:

- [ ] Safe default is the previous behavior (not the new behavior)
- [ ] Flag evaluation context is passed explicitly, not via globals
- [ ] No flag evaluation inside a tight loop without caching
- [ ] Unit tests cover both the flag-on and flag-off code paths
- [ ] Flag metadata is updated in the registry
- [ ] Cleanup ticket is linked in the PR description

### 11.3 Automated Governance

- **Static analysis:** detect flags older than 60 days with no cleanup ticket (CI check)
- **Dashboards:** weekly digest of stale flags (no evaluations in 30 days) to flag owners
- **Limits:** enforce < 50 active flags per service; alert at 40 (warn) and block new flag creation at 50
- **Chaos testing:** periodically flip random non-critical flags to validate fallback paths

---

## 12. Quick Reference

### 12.1 Platform SDK & Library Summary

| Platform | Recommended SDK / Library |
|---|---|
| **Java (Server)** | OpenFeature Java SDK · LaunchDarkly Java v6 · FF4J (Spring-native) |
| **.NET (Server)** | Microsoft.FeatureManagement · OpenFeature .NET · LaunchDarkly .NET v7 |
| **Python (Server)** | OpenFeature Python · LaunchDarkly Python · Flagsmith Python |
| **Angular (Client)** | LaunchDarkly Angular SDK · OpenFeature Web + Angular · Custom proxy service |
| **React (Client)** | LaunchDarkly React SDK · OpenFeature Web React · Custom Context hook |
| **Flipt (Self-hosted)** | flipt.io — single binary, Git-native, OpenFeature provider for all languages |
| **flagd (Self-hosted)** | flagd.dev — OpenFeature reference daemon, K8s sidecar, file/HTTP/CRD sync |
| **All (Vendor-neutral)** | OpenFeature spec (openfeature.dev) — evaluate first for new projects |

### 12.2 Decision Tree — Which Flag Type?

```
Is the feature incomplete but deployed?        → Release Flag        (short-lived)
Is this an A/B or multivariate experiment?    → Experiment Flag     (medium-lived)
Is this a kill-switch or circuit-breaker?     → Ops Toggle          (permanent)
Is this about user plan or permission?        → Permission Flag     (long-lived, model as entitlement)
Is this switching between implementations?    → Infrastructure Flag (short-lived)
```

> **🏆 GOLDEN RULE:** If you cannot answer *"When does this flag get removed?"* at creation time, do not create the flag until you can. Flags without a defined lifecycle are technical debt on day one.

---

*This document is maintained by the Staff Architecture team. Submit revisions via the architecture-docs repository.*