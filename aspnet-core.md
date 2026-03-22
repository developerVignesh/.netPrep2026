# 🌐 ASP.NET Core – Interview Preparation Guide

> Each question: 📌 Definition | 🔍 Why It Exists | 🏭 Real-Time Use Case (full code) | ⚙️ Production Implementation

---

## Q52. Explain request pipeline in ASP.NET Core.

### 📌 Definition
The **ASP.NET Core request pipeline** is an ordered chain of middleware components. Each receives an `HttpContext`, optionally processes the request, calls the next middleware (`await next(context)`), and optionally processes the response on the way back. The pipeline is built in `Program.cs` using `app.Use()`, `app.Run()`, and `app.UseXxx()` extension methods. Execution is bidirectional: middleware runs pre-next logic on the incoming request, and post-next logic on the outgoing response —forming a nested "Russian doll" or "onion" structure. Routing, authentication, authorization, exception handling, and endpoint execution are all implemented as middleware.

### 🔍 Why It Exists / Problem It Solves
```csharp
// Without middleware pipeline: cross-cutting concerns scatter across every controller
// With middleware pipeline: apply once, affects all requests

// Execution order diagram:
// Request →  [ExceptionHandler] → [Auth] → [Routing] → [RateLimit] → [Endpoint]
// Response ←  [ExceptionHandler] ← [Auth] ← [Routing] ← [RateLimit] ← [Endpoint]

// app.Use() — two-way middleware (pre + post)
app.Use(async (context, next) =>
{
    // PRE — request coming in
    Console.WriteLine($"Before: {context.Request.Method} {context.Request.Path}");
    await next(context); // Call next middleware
    // POST — response going out
    Console.WriteLine($"After: Status {context.Response.StatusCode}");
});

// app.Run() — terminal middleware (no next call)
app.Run(async context =>
{
    await context.Response.WriteAsync("Hello from terminal middleware");
});
```

### 🏭 Real-Time Use Case — Full Implementation

```csharp
// Program.cs — complete production trading API pipeline
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme).AddJwtBearer(...);
builder.Services.AddAuthorization();
builder.Services.AddControllers();
builder.Services.AddResponseCompression();
builder.Services.AddHealthChecks().AddSqlServer(connStr);

var app = builder.Build();

// Pipeline ORDER matters — each line adds a middleware
app.UseExceptionHandler("/error");          // 1. Catch all unhandled exceptions
app.UseHsts();                              // 2. HTTP Strict Transport Security header
app.UseHttpsRedirection();                  // 3. Redirect HTTP → HTTPS
app.UseResponseCompression();               // 4. Gzip/Brotli compress responses
app.UseStaticFiles();                       // 5. Serve static files (wwwroot)
app.UseRouting();                           // 6. Match URL to endpoint (sets route data)
app.UseCors("AllowTradingUI");             // 7. CORS headers (after routing for endpoint policies)
app.UseAuthentication();                    // 8. Who are you? (JWT validation)
app.UseAuthorization();                     // 9. What can you do? (policy checks)
app.MapHealthChecks("/health");             // 10. Health endpoint (bypasses auth above via MapHealthChecks options)
app.MapControllers();                       // 11. Route to controller actions (terminal)

app.Run();
```

### ⚙️ Production-Grade Implementation

| Order | Middleware | Why this position |
|---|---|---|
| 1 | `UseExceptionHandler` | Must wrap everything — catches all downstream exceptions |
| 2-3 | HSTS / HTTPS redirect | Security — run before any logic |
| 6 | `UseRouting` | Must precede `UseAuthorization` (needs route data) |
| 8 | `UseAuthentication` | Before authorization — must identify user first |
| 9 | `UseAuthorization` | After identity is established |

---

## Q53. Middleware vs Filters.

### 📌 Definition
- **Middleware** operates at the **HTTP level** — it sees raw `HttpContext`, runs for every request, and has no knowledge of controllers, actions, or MVC concepts. It is registered globally in the pipeline.
- **Filters** operate at the **MVC/controller action level** — they run only for requests that reach the MVC pipeline and have access to `ActionContext`, model binding results, `ActionResult`, exception details, and controller metadata. Filters can be scoped globally, per-controller, or per-action.

### 🔍 Why It Exists — Filter Types

```csharp
// 5 Filter types (run in this order):
// 1. Authorization filters    — [Authorize], custom IAuthorizationFilter
// 2. Resource filters         — before/after model binding; can short-circuit
// 3. Action filters           — before/after action method execution
// 4. Exception filters        — catch exceptions from action/result filters
// 5. Result filters           — before/after IActionResult execution

// When to use Middleware vs Filter:
// Middleware: logging, CORS, compression, auth tokens, rate limiting
// Filters   : model validation, action-level cache, action-level audit, result transformation
```

### 🏭 Real-Time Use Case — Full Implementation

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;

namespace TradingAPI.Filters
{
    // ── Action Filter: validate order and log execution time ───────────────
    public sealed class OrderValidationActionFilter : IAsyncActionFilter
    {
        private readonly ILogger<OrderValidationActionFilter> _logger;
        public OrderValidationActionFilter(ILogger<OrderValidationActionFilter> logger) => _logger = logger;

        public async Task OnActionExecutionAsync(ActionExecutingContext context, ActionExecutionDelegate next)
        {
            // PRE-ACTION: validate model state
            if (!context.ModelState.IsValid)
            {
                context.Result = new UnprocessableEntityObjectResult(context.ModelState);
                return; // Short-circuit — action never runs
            }

            var sw = System.Diagnostics.Stopwatch.StartNew();
            var executed = await next(); // Execute the action
            sw.Stop();

            // POST-ACTION: log result
            if (executed.Exception != null)
                _logger.LogError(executed.Exception, "Action failed: {Action}", context.ActionDescriptor.DisplayName);
            else
                _logger.LogInformation("Action {Action} completed in {Ms}ms",
                    context.ActionDescriptor.DisplayName, sw.ElapsedMilliseconds);
        }
    }

    // ── Exception Filter: handle domain exceptions at MVC level ───────────
    public sealed class TradingExceptionFilter : IExceptionFilter
    {
        public void OnException(ExceptionContext context)
        {
            if (context.Exception is OrderValidationException ex)
            {
                context.Result = new BadRequestObjectResult(new { error = ex.Message, orderId = ex.OrderId });
                context.ExceptionHandled = true; // Mark as handled — no further propagation
            }
            // Other exception types propagate to global exception middleware
        }
    }

    // Register globally in Program.cs:
    // builder.Services.AddControllers(opts => {
    //     opts.Filters.Add<OrderValidationActionFilter>();
    //     opts.Filters.Add<TradingExceptionFilter>();
    // });

    // Or per-controller / per-action:
    [ServiceFilter(typeof(OrderValidationActionFilter))]
    public class OrderController : ControllerBase
    {
        [HttpPost]
        public IActionResult PlaceOrder([FromBody] OrderRequest req)
            => Ok(new { orderId = Guid.NewGuid() });
    }

    public sealed class OrderValidationException : Exception
    {
        public string OrderId { get; }
        public OrderValidationException(string orderId, string msg) : base(msg) => OrderId = orderId;
    }
    public record OrderRequest(string Symbol, decimal Price, int Quantity);
}
```

### ⚙️ Production-Grade Implementation

| Aspect | Middleware | Filters |
|---|---|---|
| **Scope** | All requests (HTTP) | MVC requests only |
| **Access to** | `HttpContext` only | `ActionContext`, model binding, route data |
| **Short-circuit** | Return without calling `next()` | Set `context.Result` |
| **Injection** | Constructor injection | Constructor or `IServiceFilter` |
| **Per-action** | ❌ Global only | ✅ Via attributes |

---

## Q54. Routing – conventional vs attribute routing.

### 📌 Definition
- **Conventional routing** defines URL patterns in `Program.cs` that map to controller/action names by convention. Useful for MVC applications with predictable URL patterns (e.g., `/{controller}/{action}/{id?}`).
- **Attribute routing** decorates controllers and actions directly with `[Route]`, `[HttpGet]`, `[HttpPost]` etc. — giving precise, explicit control over URL shapes. Required for REST APIs where resource-centric URLs don't follow controller/action naming conventions.

### 🔍 Why It Exists / Problem It Solves

```csharp
// Conventional routing (MVC apps):
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");
// GET /Order/Details/42 → OrderController.Details(id: 42)
// GET /Trade/History    → TradeController.History()

// ❌ Problem: REST APIs have resource-centric URLs that don't map to controller/action names:
// GET /api/orders/ORD-001/executions — conventional routing can't express this cleanly

// ✅ Attribute routing — precise URL control:
[ApiController]
[Route("api/orders")]
public class OrderController : ControllerBase
{
    [HttpGet("{orderId}")]                        // GET /api/orders/ORD-001
    public IActionResult Get(string orderId) => Ok();

    [HttpGet("{orderId}/executions")]             // GET /api/orders/ORD-001/executions
    public IActionResult GetExecutions(string orderId) => Ok();

    [HttpPost]                                    // POST /api/orders
    public IActionResult Place([FromBody] OrderRequest req) => Created($"/api/orders/ORD-NEW", null);

    [HttpDelete("{orderId}")]                     // DELETE /api/orders/ORD-001
    public IActionResult Cancel(string orderId) => NoContent();
}
```

### 🏭 Real-Time Use Case — Full Implementation

```csharp
[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]  // Versioned route: /api/v1/orders
[ApiVersion("1.0")]
public class OrdersController : ControllerBase
{
    // Route constraints — enforce parameter types at routing level
    [HttpGet("{orderId:length(8,20)}")]          // orderId must be 8-20 chars
    [ProducesResponseType(typeof(OrderDto), 200)]
    [ProducesResponseType(404)]
    public async Task<IActionResult> GetOrder(string orderId, CancellationToken ct)
    {
        // orderId is guaranteed valid format by route constraint
        var order = await _repo.FindAsync(orderId, ct);
        return order is null ? NotFound() : Ok(order);
    }

    [HttpGet("symbol/{symbol:alpha:minlength(3)}")]  // symbol: alpha only, min 3 chars
    public async Task<IActionResult> GetBySymbol(string symbol, CancellationToken ct)
    {
        var orders = await _repo.GetBySymbolAsync(symbol, ct);
        return Ok(orders);
    }

    // Route with optional parameter
    [HttpGet("desk/{deskId?}")]                  // deskId is optional
    public IActionResult GetByDesk(string? deskId = null) => Ok(deskId ?? "all");
}
```

### ⚙️ Production-Grade Implementation

| Constraint | Example | Validates |
|---|---|---|
| `:int` | `{id:int}` | Integer only |
| `:guid` | `{id:guid}` | GUID format |
| `:length(min,max)` | `{orderId:length(8,20)}` | String length range |
| `:alpha` | `{symbol:alpha}` | Letters only |
| `:range(min,max)` | `{qty:range(1,10000)}` | Numeric range |
| `:regex(pattern)` | `{code:regex(^[A-Z]{{3}})` | Regex match |

---

## Q55. Model binding – how it works internally.

### 📌 Definition
**Model binding** is the ASP.NET Core process that automatically maps HTTP request data (route values, query strings, request body, headers, form data) to action method parameters and complex objects. The **model binding pipeline** uses a configured set of `IModelBinder` implementations — each knows how to bind a specific source. The framework inspects action parameter types and attributes (`[FromRoute]`, `[FromQuery]`, `[FromBody]`, `[FromHeader]`, `[FromForm]`, `[FromServices]`) to determine which binder to use. For `[FromBody]`, the registered input formatter (JSON by default via `System.Text.Json`) deserializes the request stream into the target type. Binding occurs before the action executes; if binding fails, `ModelState.IsValid` is `false`.

### 🔍 Why It Exists / Problem It Solves

```csharp
// Without model binding — manual parsing per endpoint:
app.MapPost("/orders", async (HttpContext ctx) =>
{
    using var reader  = new StreamReader(ctx.Request.Body);
    string    json    = await reader.ReadToEndAsync();
    var order = System.Text.Json.JsonSerializer.Deserialize<OrderRequest>(json);
    string    orderId = ctx.Request.RouteValues["orderId"]?.ToString() ?? "";
    // brittle, repetitive, no validation
});

// ✅ With model binding — automatic, validated, decoupled:
[HttpPost("{deskId}/orders")]
public IActionResult PlaceOrder(
    [FromRoute]  string       deskId,           // from URL route
    [FromQuery]  string?      priority,          // from ?priority=HIGH
    [FromHeader] string?      clientId,          // from X-Client-Id header
    [FromBody]   OrderRequest order,             // from JSON request body
    [FromServices] IRiskEngine risk)             // from DI container
{
    // All params automatically bound and available — ModelState validated
}
```

### 🏭 Real-Time Use Case — Full Implementation

```csharp
using System.ComponentModel.DataAnnotations;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.ModelBinding;

namespace TradingAPI.ModelBinding
{
    // ── Request model with validation attributes ───────────────────────────
    public sealed class PlaceOrderRequest
    {
        [Required, MinLength(3), MaxLength(20)]
        public string Symbol { get; init; } = "";

        [Required, Range(0.01, 10_000_000)]
        public decimal Price { get; init; }

        [Required, Range(1, 10_000)]
        public int Quantity { get; init; }

        [Required, RegularExpression("^(BUY|SELL)$")]
        public string Side { get; init; } = "";

        // Computed — excluded from binding
        [BindNever]
        public decimal TradeValue => Price * Quantity;
    }

    // ── Custom model binder: bind symbol from header if not in body ────────
    public sealed class SymbolModelBinder : IModelBinder
    {
        public Task BindModelAsync(ModelBindingContext ctx)
        {
            // Try header first, fall back to query string
            string? symbol = ctx.HttpContext.Request.Headers["X-Symbol"].FirstOrDefault()
                          ?? ctx.HttpContext.Request.Query["symbol"].FirstOrDefault();

            if (string.IsNullOrEmpty(symbol))
            {
                ctx.Result = ModelBindingResult.Failed();
                return Task.CompletedTask;
            }
            ctx.Result = ModelBindingResult.Success(symbol.ToUpperInvariant());
            return Task.CompletedTask;
        }
    }

    // ── Controller using complex model binding ─────────────────────────────
    [ApiController]
    [Route("api/orders")]
    public class OrderController : ControllerBase
    {
        [HttpPost]
        public IActionResult PlaceOrder([FromBody] PlaceOrderRequest req)
        {
            // [ApiController] auto-returns 400 if ModelState invalid (no manual check needed)
            return Ok(new
            {
                OrderId    = Guid.NewGuid(),
                Symbol     = req.Symbol,
                TradeValue = req.TradeValue
            });
        }

        // Custom binder via attribute
        [HttpGet("by-symbol")]
        public IActionResult GetBySymbol(
            [ModelBinder(typeof(SymbolModelBinder))] string symbol)
            => Ok(new { Symbol = symbol });
    }
}
```

### ⚙️ Production-Grade Implementation

| Source Attribute | Binds from | When to use |
|---|---|---|
| `[FromRoute]` | URL path segment | Resource identity in REST URLs |
| `[FromQuery]` | Query string | Filters, pagination, sorting |
| `[FromBody]` | Request body (JSON) | Complex request objects (POST/PUT) |
| `[FromHeader]` | HTTP header | Auth tokens, client IDs, correlation IDs |
| `[FromForm]` | Multipart form data | File uploads, form submissions |
| `[FromServices]` | DI container | Service injection into action params |

---

## Q56. Model validation – pipeline.

### 📌 Definition
**Model validation** in ASP.NET Core automatically validates action parameters after model binding using Data Annotations (`[Required]`, `[Range]`, `[MaxLength]`, `[RegularExpression]`) and custom `IValidatableObject` / `ValidationAttribute` implementations. With `[ApiController]`, if `ModelState.IsValid` is `false` after binding and validation, the framework automatically short-circuits the action and returns an RFC 7807 problem details `400 Bad Request` response — the action method never executes. The validation pipeline runs: (1) model binding, (2) validation attributes checked per property, (3) `IValidatableObject.Validate()` for cross-property validation, (4) model state populated with all errors.

### 🔍 Why It Exists / Problem It Solves

```csharp
// ❌ Manual validation in every action — brittle, repetitive:
public IActionResult PlaceOrder(OrderRequest req)
{
    if (string.IsNullOrEmpty(req.Symbol)) return BadRequest("Symbol required");
    if (req.Price <= 0) return BadRequest("Price must be positive");
    // ... 10 more conditions
}

// ✅ Declarative validation — DRY, testable:
public class OrderRequest : IValidatableObject
{
    [Required] public string Symbol   { get; init; } = "";
    [Range(0.01, 1_000_000)] public decimal Price { get; init; }
    [Range(1, 50_000)] public int Quantity { get; init; }

    // Cross-property validation (can't express with attributes alone)
    public IEnumerable<ValidationResult> Validate(ValidationContext ctx)
    {
        if (Price * Quantity > 10_000_000m)
            yield return new ValidationResult(
                "Trade value exceeds ₹10Cr limit",
                new[] { nameof(Price), nameof(Quantity) });

        if (Symbol == "ILLIQUID" && Quantity > 100)
            yield return new ValidationResult(
                "Max 100 lots for illiquid symbols",
                new[] { nameof(Symbol), nameof(Quantity) });
    }
}
```

### 🏭 Real-Time Use Case — Full Implementation

```csharp
using System.ComponentModel.DataAnnotations;
using Microsoft.AspNetCore.Mvc;

namespace TradingAPI.Validation
{
    // Custom validation attribute — reusable across models
    public sealed class TradingSymbolAttribute : ValidationAttribute
    {
        private static readonly HashSet<string> _validSymbols =
            new() { "NIFTY50", "BANKNIFTY", "SENSEX", "RELIANCE", "TCS", "INFY" };

        protected override ValidationResult? IsValid(object? value, ValidationContext ctx)
        {
            if (value is string symbol && _validSymbols.Contains(symbol.ToUpperInvariant()))
                return ValidationResult.Success;

            return new ValidationResult($"'{value}' is not a recognized trading symbol");
        }
    }

    public sealed class OrderRequest : IValidatableObject
    {
        [Required(ErrorMessage = "Symbol is required")]
        [TradingSymbol]
        public string Symbol { get; init; } = "";

        [Required]
        [Range(0.01, 1_000_000, ErrorMessage = "Price must be between ₹0.01 and ₹10,00,000")]
        public decimal Price { get; init; }

        [Range(1, 10_000)]
        public int Quantity { get; init; }

        [Required, RegularExpression("^(BUY|SELL|BUY_LIMIT|SELL_LIMIT)$")]
        public string OrderType { get; init; } = "";

        public IEnumerable<ValidationResult> Validate(ValidationContext ctx)
        {
            decimal tradeValue = Price * Quantity;
            if (tradeValue > 5_000_000m)
                yield return new ValidationResult(
                    $"Trade value ₹{tradeValue:N0} exceeds maximum ₹50L per order",
                    new[] { nameof(Price), nameof(Quantity) });
        }
    }

    [ApiController]             // Auto 400 on ModelState invalid
    [Route("api/orders")]
    public class OrderController : ControllerBase
    {
        [HttpPost]
        public IActionResult PlaceOrder([FromBody] OrderRequest req)
        {
            // If we reach here, req is FULLY VALIDATED — [ApiController] filtered bad requests
            return Ok(new { OrderId = Guid.NewGuid(), req.Symbol, req.Price, req.Quantity });
        }
    }
}
```

### ⚙️ Production-Grade Implementation

| Validation type | Mechanism | Cross-property? |
|---|---|---|
| Required, Range, MaxLength | `DataAnnotations` attributes | ❌ |
| Custom attribute | Inherit `ValidationAttribute` | ❌ |
| Cross-field rules | `IValidatableObject.Validate()` | ✅ |
| Service-dependent validation | `FluentValidation` library | ✅ |

Configure custom error response format:
```csharp
builder.Services.AddControllers()
    .ConfigureApiBehaviorOptions(opts =>
        opts.InvalidModelStateResponseFactory = ctx =>
            new UnprocessableEntityObjectResult(new
            {
                errors = ctx.ModelState
                    .Where(e => e.Value?.Errors.Count > 0)
                    .ToDictionary(
                        e => e.Key,
                        e => e.Value!.Errors.Select(x => x.ErrorMessage))
            }));
```

---

---

## Q57. JWT Authentication – how it works internally.

### 📌 Definition
**JWT (JSON Web Token)** is a compact, URL-safe token format used for authentication and information exchange. A JWT consists of three base64url-encoded parts separated by dots: **Header** (algorithm + type), **Payload** (claims: sub, iss, exp, roles, custom), and **Signature** (HMAC or RSA of header+payload). The server validates JWTs by re-computing the signature using the secret key — no database lookup needed (stateless). In ASP.NET Core, JWT authentication is implemented via `UseAuthentication()` + `AddJwtBearer()` which registers an `IAuthenticationHandler` that extracts the `Authorization: Bearer <token>` header, validates the signature/expiry/issuer/audience, and populates `HttpContext.User` with claims from the payload.

### 🔍 Why It Exists / Problem It Solves

```
JWT structure:
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9         ← Header (base64): {"alg":"HS256","typ":"JWT"}
.eyJzdWIiOiJUUkFERVItNDIiLCJyb2xlIjoiVHJhZGVyIiwiZXhwIjoxNzA5MDAwMDAwfQ== ← Payload
.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c  ← Signature (HMAC-SHA256)

Payload (decoded):
{ "sub": "TRADER-42", "role": "Trader", "desk": "EQUITY", "exp": 1709000000 }
```

### 🏭 Real-Time Use Case — Full Implementation

```csharp
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Microsoft.IdentityModel.Tokens;

namespace TradingAPI.Auth
{
    // ── JWT Settings POCO ─────────────────────────────────────────────────
    public sealed class JwtSettings
    {
        public string Secret    { get; init; } = "";  // min 32 chars for HMAC-SHA256
        public string Issuer    { get; init; } = "TradingAPI";
        public string Audience  { get; init; } = "TradingClients";
        public int    ExpiryMinutes { get; init; } = 60;
    }

    // ── Token service ─────────────────────────────────────────────────────
    public sealed class JwtTokenService
    {
        private readonly JwtSettings _settings;
        private readonly SymmetricSecurityKey _key;

        public JwtTokenService(JwtSettings settings)
        {
            _settings = settings;
            _key      = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(settings.Secret));
        }

        public string GenerateToken(string traderId, string role, string desk)
        {
            var claims = new[]
            {
                new Claim(JwtRegisteredClaimNames.Sub, traderId),
                new Claim(ClaimTypes.Role, role),
                new Claim("desk", desk),
                new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
            };

            var credentials = new SigningCredentials(_key, SecurityAlgorithms.HmacSha256);

            var token = new JwtSecurityToken(
                issuer:   _settings.Issuer,
                audience: _settings.Audience,
                claims:   claims,
                expires:  DateTime.UtcNow.AddMinutes(_settings.ExpiryMinutes),
                signingCredentials: credentials);

            return new JwtSecurityTokenHandler().WriteToken(token);
        }
    }

    // ── Program.cs JWT registration ───────────────────────────────────────
    public static class JwtStartup
    {
        public static void AddJwtAuth(WebApplicationBuilder builder, JwtSettings jwtSettings)
        {
            builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
                .AddJwtBearer(opts =>
                {
                    opts.TokenValidationParameters = new TokenValidationParameters
                    {
                        ValidateIssuer           = true,
                        ValidateAudience         = true,
                        ValidateLifetime         = true,
                        ValidateIssuerSigningKey = true,
                        ValidIssuer              = jwtSettings.Issuer,
                        ValidAudience            = jwtSettings.Audience,
                        IssuerSigningKey         = new SymmetricSecurityKey(
                            Encoding.UTF8.GetBytes(jwtSettings.Secret)),
                        ClockSkew                = TimeSpan.FromSeconds(30)  // Allow 30s clock drift
                    };

                    opts.Events = new JwtBearerEvents
                    {
                        OnAuthenticationFailed = ctx =>
                        {
                            if (ctx.Exception is SecurityTokenExpiredException)
                                ctx.Response.Headers["Token-Expired"] = "true";
                            return Task.CompletedTask;
                        }
                    };
                });

            builder.Services.AddAuthorization(opts =>
            {
                opts.AddPolicy("EquityDesk", p => p.RequireClaim("desk", "EQUITY"));
                opts.AddPolicy("SeniorTrader", p => p.RequireRole("SeniorTrader").RequireClaim("desk"));
            });
        }
    }

    // ── Auth + protected controllers ──────────────────────────────────────
    [ApiController, Route("api/auth")]
    public class AuthController : ControllerBase
    {
        private readonly JwtTokenService _tokenSvc;
        public AuthController(JwtTokenService tokenSvc) => _tokenSvc = tokenSvc;

        [HttpPost("login")]
        [AllowAnonymous]
        public IActionResult Login([FromBody] LoginRequest req)
        {
            if (req.Username == "trader1" && req.Password == "pass")
            {
                string token = _tokenSvc.GenerateToken("TRADER-42", "Trader", "EQUITY");
                return Ok(new { token, expiresIn = 3600 });
            }
            return Unauthorized();
        }
    }

    [ApiController, Route("api/orders"), Authorize]
    public class OrderController : ControllerBase
    {
        [HttpPost]
        [Authorize(Policy = "EquityDesk")]  // Only EQUITY desk traders
        public IActionResult PlaceOrder()
        {
            string traderId = User.FindFirstValue(ClaimTypes.NameIdentifier)!;
            string desk     = User.FindFirstValue("desk")!;
            return Ok(new { traderId, desk, status = "Placed" });
        }
    }

    public record LoginRequest(string Username, string Password);
}
```

### ⚙️ Production-Grade Implementation

| Security concern | Recommendation |
|---|---|
| **Key length** | Min 256-bit for HMAC-SHA256 — use `secrets.json` or Key Vault |
| **Token expiry** | Short-lived (15-60 min) access token + longer refresh token |
| **Refresh tokens** | Store hashed in DB — revokable, one-time use |
| **Signature algorithm** | RS256 (RSA) for distributed verification (API gateway + services) |
| **ClockSkew** | Set to 30s — allow slight clock drift between servers |

---

## Q58. Authorization – policies, roles, claims.

### 📌 Definition
ASP.NET Core authorization provides three levels of increasing sophistication: **Role-based** (`[Authorize(Roles = "Admin")]`) — checking if the user's `role` claim contains specified values; **Policy-based** (`[Authorize(Policy = "EquityDesk")]`) — named rules composed of multiple requirements (roles, claims, age, custom logic) evaluated by `IAuthorizationHandler`; and **Resource-based** authorization — checking permissions against a specific resource instance (e.g., can this user view THIS specific order). All authorization runs after authentication populates `HttpContext.User`.

### 🔍 Why It Exists / Problem It Solves

```csharp
// Role-based (simple, sufficient for admin/user split):
[Authorize(Roles = "Admin")]
public IActionResult AdminDashboard() => Ok();

// Policy-based (composable, testable requirements):
// Register:
options.AddPolicy("CanPlaceLargeOrders", policy => policy
    .RequireRole("SeniorTrader")
    .RequireClaim("desk")
    .RequireClaim("riskApproved", "true")
    .AddRequirements(new MaxOrderValueRequirement(5_000_000m)));

// Apply:
[Authorize(Policy = "CanPlaceLargeOrders")]
public IActionResult PlaceLargeOrder() => Ok();
```

### 🏭 Real-Time Use Case — Full Implementation

```csharp
using Microsoft.AspNetCore.Authorization;
using System.Security.Claims;

namespace TradingAPI.Authorization
{
    // Custom requirement
    public sealed class MaxOrderValueRequirement : IAuthorizationRequirement
    {
        public decimal MaxValue { get; }
        public MaxOrderValueRequirement(decimal max) => MaxValue = max;
    }

    // Custom handler — evaluates the requirement
    public sealed class MaxOrderValueHandler : AuthorizationHandler<MaxOrderValueRequirement>
    {
        protected override Task HandleRequirementAsync(
            AuthorizationHandlerContext ctx,
            MaxOrderValueRequirement req)
        {
            var maxValueClaim = ctx.User.FindFirstValue("maxOrderValue");
            if (maxValueClaim != null && decimal.TryParse(maxValueClaim, out decimal userMax))
            {
                if (userMax >= req.MaxValue)
                    ctx.Succeed(req); // Requirement met
                else
                    ctx.Fail(new AuthorizationFailureReason(this,
                        $"User max order value {userMax:C} < required {req.MaxValue:C}"));
            }
            return Task.CompletedTask;
        }
    }

    // Resource-based authorization — per-resource permission check
    public sealed class OrderAuthorizationHandler
        : AuthorizationHandler<OperationAuthorizationRequirement, Order>
    {
        protected override Task HandleRequirementAsync(
            AuthorizationHandlerContext ctx,
            OperationAuthorizationRequirement operation,
            Order resource)
        {
            string? userId = ctx.User.FindFirstValue(ClaimTypes.NameIdentifier);
            if (operation.Name == "View"   && (resource.TraderId == userId || ctx.User.IsInRole("Admin")))
                ctx.Succeed(operation);
            if (operation.Name == "Cancel" && resource.TraderId == userId && resource.Status == "OPEN")
                ctx.Succeed(operation);
            return Task.CompletedTask;
        }
    }

    // ServiceCollections registration (Program.cs):
    // builder.Services.AddScoped<IAuthorizationHandler, MaxOrderValueHandler>();
    // builder.Services.AddScoped<IAuthorizationHandler, OrderAuthorizationHandler>();
    // builder.Services.AddAuthorization(opts => {
    //     opts.AddPolicy("LargeOrderTrader", p => p
    //         .RequireRole("SeniorTrader")
    //         .AddRequirements(new MaxOrderValueRequirement(1_000_000m)));
    // });

    [ApiController, Route("api/orders"), Authorize]
    public sealed class OrderController : ControllerBase
    {
        private readonly IAuthorizationService _authz;
        public OrderController(IAuthorizationService authz) => _authz = authz;

        [HttpGet("{orderId}")]
        public async Task<IActionResult> Get(string orderId)
        {
            var order = new Order(orderId, User.FindFirstValue(ClaimTypes.NameIdentifier)!, "OPEN");
            // Resource-based: checks if current user can view THIS specific order
            var result = await _authz.AuthorizeAsync(User, order, new OperationAuthorizationRequirement { Name = "View" });
            return result.Succeeded ? Ok(order) : Forbid();
        }
    }

    public record Order(string Id, string TraderId, string Status);
}
```

### ⚙️ Production-Grade Implementation

| Authorization type | When to use |
|---|---|
| `[AllowAnonymous]` | Public endpoints (login, health, docs) |
| `[Authorize]` | Any authenticated user |
| `[Authorize(Roles = "X")]` | Simple role guard |
| `[Authorize(Policy = "X")]` | Composable multi-requirement rules |
| `IAuthorizationService.AuthorizeAsync(user, resource, op)` | Per-resource, data-driven authorization |

---

## Q59. Action Result types and when to use them.

### 📌 Definition
ASP.NET Core provides a hierarchy of `IActionResult` types for returning HTTP responses from controllers. Using specific `ActionResult<T>` vs raw `IActionResult` enables Swagger/OpenAPI to generate accurate response schemas. Common types: `Ok(data)` → 200, `Created(location, data)` → 201, `NoContent()` → 204, `BadRequest(errors)` → 400, `Unauthorized()` → 401, `Forbidden()` → 403, `NotFound()` → 404, `Conflict(data)` → 409, `UnprocessableEntity(errors)` → 422, `StatusCode(code, data)` → custom, `File(bytes, contentType)` → binary response.

### 🏭 Real-Time Use Case — Full Implementation

```csharp
[ApiController, Route("api/orders")]
public sealed class OrderController : ControllerBase
{
    // ActionResult<T> — typed: Swagger can infer response schema
    [HttpGet("{orderId}")]
    [ProducesResponseType(typeof(OrderDto), 200)]
    [ProducesResponseType(404)]
    public ActionResult<OrderDto> Get(string orderId)
    {
        var order = FindOrder(orderId);
        return order is null ? NotFound(new { orderId, error = "Order not found" }) : Ok(order);
    }

    [HttpPost]
    [ProducesResponseType(typeof(OrderDto), 201)]
    [ProducesResponseType(422)]
    public ActionResult<OrderDto> Create([FromBody] OrderRequest req)
    {
        var order = new OrderDto(Guid.NewGuid().ToString(), req.Symbol, req.Price);
        return CreatedAtAction(nameof(Get), new { orderId = order.Id }, order);
        // 201 Created with Location: /api/orders/ORD-001 header
    }

    [HttpDelete("{orderId}")]
    [ProducesResponseType(204)]
    [ProducesResponseType(404)]
    [ProducesResponseType(409)]
    public IActionResult Cancel(string orderId)
    {
        var order = FindOrder(orderId);
        if (order is null) return NotFound();
        if (order.Status == "FILLED") return Conflict(new { error = "Cannot cancel filled order" });
        return NoContent(); // 204 — success, no body
    }

    // Binary file response
    [HttpGet("{orderId}/report")]
    public IActionResult GetReport(string orderId)
    {
        byte[] pdfBytes = GeneratePdf(orderId);
        return File(pdfBytes, "application/pdf", $"order-{orderId}.pdf");
    }

    // Custom status code
    [HttpPost("{orderId}/partial-fill")]
    public IActionResult PartialFill(string orderId)
        => StatusCode(206, new { orderId, status = "PartiallyFilled" }); // 206 Partial Content

    private OrderDto? FindOrder(string id) => id == "ORD-999" ? null : new OrderDto(id, "NIFTY50", 24500m);
    private byte[] GeneratePdf(string id) => Array.Empty<byte>();
    public record OrderDto(string Id, string Symbol, decimal Price, string Status = "OPEN");
    public record OrderRequest(string Symbol, decimal Price);
}
```

---

## Q60. Content negotiation in ASP.NET Core.

### 📌 Definition
**Content negotiation** is the mechanism where the client indicates its preferred response format via the `Accept` header (e.g., `Accept: application/json` or `Accept: application/xml`) and the server selects the best matching output formatter. In ASP.NET Core MVC, this is handled by `ObjectResult` (returned by `Ok()`, `Created()`, etc.) which inspects the `Accept` header and runs registered formatters in priority order. By default, only JSON is registered; XML support requires adding `AddXmlSerializerFormatters()`.

```csharp
// Program.cs — enable XML in addition to JSON:
builder.Services.AddControllers()
    .AddXmlSerializerFormatters()    // Adds text/xml and application/xml formatters
    .AddXmlDataContractSerializerFormatters(); // Alternatively: DataContract XML

// Client requests XML:
// GET /api/orders/ORD-001
// Accept: application/xml
// → Response: <OrderDto><Id>ORD-001</Id><Symbol>NIFTY50</Symbol></OrderDto>

// Restrict available formats per controller:
[Produces("application/json")]   // Only JSON regardless of Accept header
public class OrderController { }

// Custom formatter (e.g., CSV for batch exports):
public sealed class CsvOutputFormatter : TextOutputFormatter
{
    public CsvOutputFormatter()
    {
        SupportedMediaTypes.Add("text/csv");
        SupportedEncodings.Add(Encoding.UTF8);
    }
    protected override bool CanWriteType(Type? type) => type == typeof(List<OrderDto>);
    public override async Task WriteResponseBodyAsync(OutputFormatterWriteContext ctx, Encoding selectedEncoding)
    {
        var orders = (List<OrderDto>)ctx.Object!;
        var csv = new StringBuilder("Id,Symbol,Price\n");
        foreach (var o in orders) csv.AppendLine($"{o.Id},{o.Symbol},{o.Price}");
        await ctx.HttpContext.Response.WriteAsync(csv.ToString(), selectedEncoding);
    }
}
```

---

## Q61. Versioning API.

### 📌 Definition
**API versioning** in ASP.NET Core allows multiple versions of an API to coexist — clients pin to a specific version; the server can evolve without breaking existing consumers. Versioning strategies: **URL path** (`/api/v1/orders`), **query string** (`/api/orders?api-version=1.0`), **header** (`Api-Version: 1.0`), **media type** (`Accept: application/vnd.trading.v1+json`). The `Asp.Versioning.Mvc` NuGet package (formerly `Microsoft.AspNetCore.Mvc.Versioning`) provides attributes, discovery, and deprecation support.

```csharp
// Program.cs:
builder.Services.AddApiVersioning(opts =>
{
    opts.DefaultApiVersion   = new ApiVersion(1, 0);
    opts.AssumeDefaultVersionWhenUnspecified = true;
    opts.ReportApiVersions   = true;    // Adds api-supported-versions response header
}).AddMvc();

// V1 controller:
[ApiController, Route("api/v{version:apiVersion}/orders")]
[ApiVersion("1.0")]
public class OrdersV1Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetOrders() => Ok(new { version = "v1", data = new[] { "ORD-001" } });
}

// V2 controller — extended response:
[ApiController, Route("api/v{version:apiVersion}/orders")]
[ApiVersion("2.0")]
[ApiVersion("1.0", Deprecated = true)]  // Mark v1 as deprecated on this controller
public class OrdersV2Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetOrders()
        => Ok(new { version = "v2", data = new[] { new { id = "ORD-001", status = "FILLED" } } });
}
// GET /api/v1/orders → v1 response
// GET /api/v2/orders → v2 response
// Response header: api-supported-versions: 1.0, 2.0 | api-deprecated-versions: 1.0
```

---

## Q62. Minimal APIs vs Controllers.

### 📌 Definition
**Minimal APIs** (introduced in .NET 6) define HTTP endpoints directly in `Program.cs` using lambda expressions — no controller classes, no attributes, minimal ceremony. Best for microservices, internal APIs, and small focused services where the MVC pattern adds unnecessary overhead. **Controllers** (`ControllerBase`) provide the full MVC pipeline: model binding, model validation, action filters, content negotiation, and rich convention-based routing. Best for larger APIs with complex business logic, shared filters, and OpenAPI documentation needs.

```csharp
// Minimal API — entire API in Program.cs:
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
var app = builder.Build();

app.MapGet("/api/orders/{orderId}", async (string orderId, IOrderRepository repo, CancellationToken ct)
    => await repo.FindAsync(orderId, ct) is { } order ? Results.Ok(order) : Results.NotFound());

app.MapPost("/api/orders", async (OrderRequest req, IOrderRepository repo)
    => Results.Created($"/api/orders/{req.Symbol}", await repo.CreateAsync(req)));

app.MapDelete("/api/orders/{orderId}", async (string orderId, IOrderRepository repo)
    => await repo.DeleteAsync(orderId) ? Results.NoContent() : Results.NotFound());

// Group with shared prefix/auth:
var orders = app.MapGroup("/api/v2/orders").RequireAuthorization().WithTags("Orders");
orders.MapGet("",   (IOrderRepository r) => r.GetAllAsync());
orders.MapGet("{id}", (string id, IOrderRepository r) => r.FindAsync(id));
```

| Dimension | Minimal API | Controller |
|---|---|---|
| **Boilerplate** | Minimal | More (class + attributes) |
| **Filters** | Endpoint filters | Action filters |
| **Model validation** | Manual | Auto (`[ApiController]`) |
| **Best for** | Microservices, small APIs | Large APIs, complex logic |

---

---

## Q63. Response caching and output caching.

### 📌 Definition
- **Response caching** — adds HTTP cache headers (`Cache-Control`, `Vary`, `Expires`) and optionally caches responses in memory server-side. Controlled via `[ResponseCache]` attribute on controllers.
- **Output caching** (ASP.NET Core 7+) — a full server-side output cache that stores the complete response body and bypasses endpoint execution on cache hits. More powerful than response caching: supports cache tags, eviction, policies, and varied caching by route/query/header.

```csharp
// Program.cs:
builder.Services.AddResponseCaching();
builder.Services.AddOutputCache(opts =>
{
    opts.AddBasePolicy(b => b.Cache());                           // Default: cache all
    opts.AddPolicy("Ticks", b => b.Cache().Expire(TimeSpan.FromSeconds(1)).Tag("ticks"));
    opts.AddPolicy("Reports", b => b.Cache().Expire(TimeSpan.FromMinutes(5)));
});
app.UseOutputCache();

// Controller usage:
[ApiController, Route("api")]
public class MarketController : ControllerBase
{
    // Output cache: cache market prices for 1s, vary by symbol query param
    [HttpGet("prices")]
    [OutputCache(PolicyName = "Ticks", VaryByQueryKeys = new[] { "symbol" })]
    public async Task<IActionResult> GetPrice([FromQuery] string symbol, CancellationToken ct)
        => Ok(new { symbol, price = 24_500m + symbol.Length, cachedAt = DateTime.UtcNow });

    // Invalidate cache tag when price updated (trading system real-time scenario)
    [HttpPost("prices/invalidate")]
    public async Task<IActionResult> InvalidatePrices(
        [FromServices] IOutputCacheStore store, CancellationToken ct)
    {
        await store.EvictByTagAsync("ticks", ct);   // All "ticks" cache entries evicted
        return NoContent();
    }
}
```

### ⚙️ Production-Grade Implementation

| Scenario | Cache duration | Vary by |
|---|---|---|
| Market tick prices | 1-5 seconds | Symbol, exchange |
| End-of-day reports | 5-60 minutes | Date, report type |
| Reference data (holidays, instruments) | 1 hour – 24 hours | None |
| User-specific portfolio | Cache disabled | (per-user = no shared cache) |

---

## Q64. Dependency Injection in ASP.NET Core – deep dive.

*Covered extensively in Q22 (lifetimes) and Q23 (DI internals). ASP.NET Core adds:*

```csharp
// Keyed services (ASP.NET Core 8+) — inject specific implementation by key
builder.Services.AddKeyedSingleton<IExchangeConnector, NSEConnector>("NSE");
builder.Services.AddKeyedSingleton<IExchangeConnector, BSEConnector>("BSE");

// Resolve by key:
public class TradeRouter([FromKeyedServices("NSE")] IExchangeConnector nse,
                         [FromKeyedServices("BSE")] IExchangeConnector bse) { }

// IHostedService registration (background services):
builder.Services.AddHostedService<MarketFeedBackgroundService>();

// IOptions validation at startup:
builder.Services.AddOptions<TradingConfig>()
    .Bind(builder.Configuration.GetSection("Trading"))
    .ValidateDataAnnotations()
    .ValidateOnStart(); // Throws on startup if config invalid — fail fast
```

---

## Q65. Hosted services and BackgroundService.

### 📌 Definition
`IHostedService` is an interface for long-running background tasks that run for the lifetime of the application. `BackgroundService` is an abstract base class implementing `IHostedService`, exposing an `ExecuteAsync(CancellationToken)` method to override. In ASP.NET Core, hosted services start when the host starts and are gracefully stopped (CancellationToken signaled) when the host shuts down. Use cases: market data feed listeners, trade settlement processors, scheduled jobs, queue consumers.

### 🏭 Real-Time Use Case — Full Implementation

```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using System.Threading;
using System.Threading.Channels;

namespace TradingPlatform.BackgroundServices
{
    // ── Market price feed consumer — BackgroundService ─────────────────────
    public sealed class MarketFeedService : BackgroundService
    {
        private readonly Channel<string> _feed;
        private readonly ILogger<MarketFeedService> _logger;
        private readonly IServiceScopeFactory _scopeFactory;

        public MarketFeedService(
            Channel<string> feed,
            ILogger<MarketFeedService> logger,
            IServiceScopeFactory scopeFactory)
        {
            _feed         = feed;
            _logger       = logger;
            _scopeFactory = scopeFactory;
        }

        protected override async Task ExecuteAsync(CancellationToken stoppingToken)
        {
            _logger.LogInformation("Market Feed Service started");

            // Gracefully reads until stoppingToken is cancelled (app shutdown)
            await foreach (var tick in _feed.Reader.ReadAllAsync(stoppingToken))
            {
                try
                {
                    await ProcessTickAsync(tick, stoppingToken);
                }
                catch (Exception ex) when (!stoppingToken.IsCancellationRequested)
                {
                    // Log and continue — don't crash the service on per-tick errors
                    _logger.LogError(ex, "Failed to process tick: {Tick}", tick);
                }
            }

            _logger.LogInformation("Market Feed Service stopping");
        }

        private async Task ProcessTickAsync(string tick, CancellationToken ct)
        {
            // Scoped service for DB write — BackgroundService is singleton
            using var scope = _scopeFactory.CreateScope();
            var repo = scope.ServiceProvider.GetRequiredService<ITickRepository>();
            await repo.SaveAsync(tick, ct);
            _logger.LogDebug("Tick saved: {Tick}", tick);
        }
    }

    // ── Scheduled job service — runs on timer ──────────────────────────────
    public sealed class DailySettlementService : BackgroundService
    {
        private readonly ILogger<DailySettlementService> _logger;
        private readonly IServiceScopeFactory _scopeFactory;
        private readonly TimeOnly _runAt = new(15, 30, 0); // 3:30 PM

        public DailySettlementService(ILogger<DailySettlementService> logger, IServiceScopeFactory scopeFactory)
            => (_logger, _scopeFactory) = (logger, scopeFactory);

        protected override async Task ExecuteAsync(CancellationToken stoppingToken)
        {
            while (!stoppingToken.IsCancellationRequested)
            {
                var now  = TimeOnly.FromDateTime(DateTime.Now);
                var next = _runAt > now
                    ? DateTime.Today.Add(_runAt.ToTimeSpan())
                    : DateTime.Today.AddDays(1).Add(_runAt.ToTimeSpan());

                await Task.Delay(next - DateTime.Now, stoppingToken);

                _logger.LogInformation("Running daily settlement at {Time}", DateTime.Now);
                using var scope = _scopeFactory.CreateScope();
                var svc = scope.ServiceProvider.GetRequiredService<ISettlementService>();
                await svc.RunAsync(stoppingToken);
            }
        }
    }

    // Registration in Program.cs:
    // builder.Services.AddSingleton(Channel.CreateUnbounded<string>());
    // builder.Services.AddHostedService<MarketFeedService>();
    // builder.Services.AddHostedService<DailySettlementService>();

    public interface ITickRepository { Task SaveAsync(string tick, CancellationToken ct); }
    public interface ISettlementService { Task RunAsync(CancellationToken ct); }
}
```

### ⚙️ Production-Grade Implementation

| Pattern | Why |
|---|---|
| `BackgroundService` + `CancellationToken` | Graceful shutdown — complete in-flight work before stopping |
| `IServiceScopeFactory` in constructor | Singleton host service needing Scoped services (DbContext etc.) |
| `Channel<T>` for work queue | Lock-free, high-throughput producer/consumer between services |
| `try/catch` per work item | Don't let per-item errors crash the entire background loop |

---

## Q66. Health checks – implementation.

### 📌 Definition
**Health checks** in ASP.NET Core provide endpoints for container orchestrators (Kubernetes, Docker) and load balancers to probe application health. A health check is an `IHealthCheck` implementation that returns `HealthCheckResult.Healthy()`, `Degraded()`, or `Unhealthy()`. ASP.NET Core ships with built-in checks for SQL Server, Redis, Entity Framework, and external HTTP endpoints. Kubernetes uses two endpoints: `/health/live` (liveness — is the process running?) and `/health/ready` (readiness — is it ready to serve traffic?).

### 🏭 Real-Time Use Case — Full Implementation

```csharp
using Microsoft.Extensions.Diagnostics.HealthChecks;

namespace TradingPlatform.HealthChecks
{
    // Custom health check: verify exchange connectivity
    public sealed class ExchangeConnectivityCheck : IHealthCheck
    {
        private readonly HttpClient _http;
        public ExchangeConnectivityCheck(IHttpClientFactory factory) => _http = factory.CreateClient("NSE");

        public async Task<HealthCheckResult> CheckHealthAsync(
            HealthCheckContext ctx, CancellationToken ct = default)
        {
            try
            {
                var resp = await _http.GetAsync("/ping", ct).WaitAsync(TimeSpan.FromSeconds(3), ct);
                return resp.IsSuccessStatusCode
                    ? HealthCheckResult.Healthy("NSE exchange reachable")
                    : HealthCheckResult.Degraded($"NSE returned {resp.StatusCode}");
            }
            catch (Exception ex)
            {
                return HealthCheckResult.Unhealthy("Cannot reach NSE exchange", ex,
                    new Dictionary<string, object> { ["endpoint"] = "NSE /ping" });
            }
        }
    }

    // Program.cs registration:
    public static class HealthCheckStartup
    {
        public static void AddTradingHealthChecks(WebApplicationBuilder builder)
        {
            builder.Services.AddHealthChecks()
                .AddSqlServer(
                    builder.Configuration.GetConnectionString("OrderDb")!,
                    name: "orders-db", tags: new[] { "db", "ready" })
                .AddRedis(
                    builder.Configuration["Redis:ConnectionString"]!,
                    name: "cache", tags: new[] { "cache", "ready" })
                .AddCheck<ExchangeConnectivityCheck>("exchange", tags: new[] { "exchange", "ready" })
                .AddCheck("memory", () =>
                {
                    var allocated = GC.GetTotalMemory(forceFullCollection: false);
                    var limit     = 500L * 1024 * 1024; // 500MB threshold
                    return allocated < limit
                        ? HealthCheckResult.Healthy($"Memory: {allocated / 1024 / 1024}MB")
                        : HealthCheckResult.Degraded($"High memory: {allocated / 1024 / 1024}MB", null,
                            new Dictionary<string, object> { ["allocated"] = allocated });
                }, tags: new[] { "live" });
        }

        public static void MapTradingHealthChecks(WebApplication app)
        {
            // Liveness: is the process alive? (lightweight — only "live" tagged checks)
            app.MapHealthChecks("/health/live", new() {
                Predicate = r => r.Tags.Contains("live"),
                ResponseWriter = HealthCheckResponseWriter.WriteHealthResponse
            });
            // Readiness: ready for traffic? (all checks including DB, cache, exchange)
            app.MapHealthChecks("/health/ready", new() {
                Predicate = r => r.Tags.Contains("ready"),
                ResponseWriter = HealthCheckResponseWriter.WriteHealthResponse
            });
        }
    }

    public static class HealthCheckResponseWriter
    {
        public static async Task WriteHealthResponse(HttpContext ctx, HealthReport report)
        {
            ctx.Response.ContentType = "application/json";
            var result = new
            {
                status  = report.Status.ToString(),
                checks  = report.Entries.Select(e => new {
                    name   = e.Key,
                    status = e.Value.Status.ToString(),
                    description = e.Value.Description,
                    duration = e.Value.Duration.TotalMilliseconds
                }),
                totalDuration = report.TotalDuration.TotalMilliseconds
            };
            await ctx.Response.WriteAsJsonAsync(result);
        }
    }
}
```

---

## Q67. Rate limiting in ASP.NET Core.

### 📌 Definition
ASP.NET Core 7+ includes built-in rate limiting middleware (`Microsoft.AspNetCore.RateLimiting`). Four policies: **Fixed window** (N requests per window), **Sliding window** (N requests per rolling window), **Token bucket** (tokens replenished at rate R, burst of B), **Concurrency** (max N simultaneous requests). Applied globally or per-endpoint, with configurable rejection responses and queue depth.

```csharp
// Program.cs:
builder.Services.AddRateLimiter(opts =>
{
    opts.RejectionStatusCode = 429;
    opts.OnRejected = async (ctx, ct) =>
    {
        ctx.HttpContext.Response.Headers["Retry-After"] = "60";
        await ctx.HttpContext.Response.WriteAsync("Rate limit exceeded", ct);
    };

    // Token bucket: 100 orders/min, burst of 20
    opts.AddTokenBucketLimiter("orders", o => {
        o.TokenLimit          = 20;    // Burst capacity
        o.ReplenishmentPeriod = TimeSpan.FromSeconds(1);
        o.TokensPerPeriod     = 2;     // ~120/min sustained
        o.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        o.QueueLimit          = 5;     // Queue up to 5 excess requests
    });

    // Per-user sliding window:
    opts.AddSlidingWindowLimiter("per-user", o => {
        o.PermitLimit         = 60;
        o.Window              = TimeSpan.FromMinutes(1);
        o.SegmentsPerWindow   = 6;
    });
});
app.UseRateLimiter();

// Apply to endpoint:
app.MapPost("/api/orders", (OrderRequest req) => Results.Ok())
   .RequireRateLimiting("orders");

[EnableRateLimiting("per-user")]
[ApiController, Route("api/market")]
public class MarketDataController : ControllerBase { }
```

---

## Q68. CORS – how to configure.

### 📌 Definition
**CORS (Cross-Origin Resource Sharing)** is a browser security mechanism that restricts JavaScript on `origin-a.com` from making requests to `origin-b.com` unless `origin-b.com` explicitly allows it via CORS response headers. ASP.NET Core handles CORS through the `UseCors()` middleware that adds `Access-Control-Allow-Origin`, `Access-Control-Allow-Headers`, and `Access-Control-Allow-Methods` headers based on configured policies.

```csharp
// Program.cs:
builder.Services.AddCors(opts =>
{
    // Strict production policy — only known trading UI origins
    opts.AddPolicy("TradingUI", policy => policy
        .WithOrigins("https://trade.example.com", "https://admin.example.com")
        .WithMethods("GET", "POST", "DELETE", "PUT")
        .WithHeaders("Content-Type", "Authorization", "X-API-Key")
        .SetPreflightMaxAge(TimeSpan.FromHours(1))  // Cache preflight for 1h
        .AllowCredentials());

    // Dev-only: allow any origin
    opts.AddPolicy("DevelopmentAll", policy => policy
        .AllowAnyOrigin()
        .AllowAnyMethod()
        .AllowAnyHeader());
});

// ⚠️ UseRouting must be BEFORE UseCors for endpoint-scoped CORS:
app.UseRouting();
app.UseCors("TradingUI");  // Apply global policy
app.UseAuthorization();

// Override per endpoint:
app.MapGet("/api/public-prices", GetPrices).RequireCors("DevelopmentAll");

[EnableCors("TradingUI")]
[DisableCors] // Disable CORS for specific action
public class OrderController : ControllerBase { }
```

---

## Q69. SignalR – real-time communication.

### 📌 Definition
**SignalR** is an ASP.NET Core library for adding real-time bidirectional communication between server and clients. It abstracts over three transport mechanisms (WebSockets preferred → Server-Sent Events → Long Polling) and provides: typed server-to-client method invocation, connection groups, hub abstractions, and horizontal scaling via Redis backplane. Use cases: live market price streaming, order status notifications, trading dashboards, chat.

### 🏭 Real-Time Use Case — Full Implementation

```csharp
using Microsoft.AspNetCore.SignalR;

namespace TradingPlatform.SignalR
{
    // ── Strongly-typed SignalR Hub ─────────────────────────────────────────
    public interface IMarketClient
    {
        Task OnPriceUpdate(PriceTick tick);
        Task OnOrderUpdate(OrderStatus status);
    }

    public sealed class MarketHub : Hub<IMarketClient>
    {
        private readonly ILogger<MarketHub> _logger;
        public MarketHub(ILogger<MarketHub> logger) => _logger = logger;

        public override async Task OnConnectedAsync()
        {
            string? desk = Context.User?.FindFirst("desk")?.Value ?? "PUBLIC";
            await Groups.AddToGroupAsync(Context.ConnectionId, desk);
            _logger.LogInformation("Client connected: {Id} → Group: {Desk}", Context.ConnectionId, desk);
            await base.OnConnectedAsync();
        }

        public override async Task OnDisconnectedAsync(Exception? ex)
        {
            _logger.LogInformation("Client disconnected: {Id}", Context.ConnectionId);
            await base.OnDisconnectedAsync(ex);
        }

        // Client can call this to subscribe to specific symbol
        public async Task SubscribeSymbol(string symbol)
        {
            await Groups.AddToGroupAsync(Context.ConnectionId, $"symbol:{symbol}");
            await Clients.Caller.OnPriceUpdate(new PriceTick(symbol, 24500m, DateTime.UtcNow));
        }
    }

    // ── Background service that broadcasts price updates ──────────────────
    public sealed class PriceBroadcastService : BackgroundService
    {
        private readonly IHubContext<MarketHub, IMarketClient> _hub;
        public PriceBroadcastService(IHubContext<MarketHub, IMarketClient> hub) => _hub = hub;

        protected override async Task ExecuteAsync(CancellationToken ct)
        {
            var rng = new Random();
            while (!ct.IsCancellationRequested)
            {
                var tick = new PriceTick("NIFTY50", 24_500m + (decimal)(rng.NextDouble() * 100), DateTime.UtcNow);
                // Broadcast to all clients subscribed to this symbol group
                await _hub.Clients.Group("symbol:NIFTY50").OnPriceUpdate(tick);
                await Task.Delay(500, ct); // Stream 2 ticks/sec
            }
        }
    }

    // Registration:
    // builder.Services.AddSignalR().AddStackExchangeRedis(connStr); // Scale-out via Redis
    // app.MapHub<MarketHub>("/hubs/market");

    public record PriceTick(string Symbol, decimal Price, DateTime Timestamp);
    public record OrderStatus(string OrderId, string Status, DateTime UpdatedAt);
}
```

---

## Q70. gRPC in ASP.NET Core.

### 📌 Definition
**gRPC** is a high-performance, contract-first, binary RPC framework using Protocol Buffers (Protobuf) for serialization and HTTP/2 for transport. In ASP.NET Core, gRPC services define contracts in `.proto` files; tooling generates C# server stubs and client proxies. Advantages over REST: 5-10x smaller payloads (binary vs JSON), strict schema contract (`.proto`), built-in streaming (unary, server streaming, client streaming, bidirectional), and strong typing. Use for internal microservice communication where performance and schema contracts matter.

```csharp
// orders.proto:
// syntax = "proto3";
// service OrderService {
//   rpc PlaceOrder   (PlaceOrderRequest)    returns (PlaceOrderResponse);
//   rpc StreamPrices (StreamPricesRequest)  returns (stream PriceTick);
// }
// message PlaceOrderRequest  { string symbol = 1; double price = 2; int32 quantity = 3; }
// message PlaceOrderResponse { string order_id = 1; string status = 2; }

// Server implementation:
public sealed class OrderGrpcService : OrderService.OrderServiceBase
{
    private readonly ILogger<OrderGrpcService> _logger;
    public OrderGrpcService(ILogger<OrderGrpcService> logger) => _logger = logger;

    public override Task<PlaceOrderResponse> PlaceOrder(PlaceOrderRequest req, ServerCallContext ctx)
    {
        _logger.LogInformation("gRPC PlaceOrder: {Symbol} x{Qty} @ {Price}", req.Symbol, req.Quantity, req.Price);
        return Task.FromResult(new PlaceOrderResponse { OrderId = Guid.NewGuid().ToString(), Status = "ACCEPTED" });
    }

    // Server streaming — push continuous price ticks to client
    public override async Task StreamPrices(StreamPricesRequest req, IServerStreamWriter<PriceTick> stream, ServerCallContext ctx)
    {
        var rng = new Random();
        while (!ctx.CancellationToken.IsCancellationRequested)
        {
            await stream.WriteAsync(new PriceTick { Symbol = req.Symbol, Price = 24500 + rng.Next(200) });
            await Task.Delay(100, ctx.CancellationToken);
        }
    }
}

// Registration:
// builder.Services.AddGrpc(opts => { opts.MaxReceiveMessageSize = 4 * 1024 * 1024; });
// app.MapGrpcService<OrderGrpcService>();
```

---

---

## Q71. Exception handling in ASP.NET Core.

### 📌 Definition
ASP.NET Core provides multiple layers for exception handling: **middleware** (`UseExceptionHandler`, `UseDeveloperExceptionPage`), **Problem Details** (RFC 7807 format for API error responses), **Exception Filters** (per-controller/action MVC-level), and **ProblemDetails service** (ASP.NET Core 7+). The recommended production approach: use `UseExceptionHandler("/error")` to catch all unhandled exceptions, then expose an `/error` endpoint that returns RFC 7807 `ProblemDetails` JSON.

```csharp
// Program.cs:
if (app.Environment.IsDevelopment())
    app.UseDeveloperExceptionPage(); // Detailed stack trace for devs
else
    app.UseExceptionHandler("/error"); // Redirect to error handler in prod

app.UseStatusCodePagesWithReExecute("/error/{0}"); // Handle 404, 405 etc.

// Error handler endpoint:
app.Map("/error", (HttpContext ctx) =>
{
    var ex = ctx.Features.Get<IExceptionHandlerFeature>()?.Error;
    return Results.Problem(
        title:      ex?.GetType().Name ?? "Error",
        detail:     ex?.Message ?? "An unexpected error occurred",
        statusCode: ex is OrderNotFoundException ? 404 :
                    ex is ArgumentException      ? 400 : 500);
});

// ASP.NET Core 7+ — ProblemDetails service (auto-generates problem details for ModelState errors too)
builder.Services.AddProblemDetails(opts =>
{
    opts.CustomizeProblemDetails = ctx =>
    {
        ctx.ProblemDetails.Extensions["correlationId"] = ctx.HttpContext.Items["CorrelationId"];
        ctx.ProblemDetails.Extensions["timestamp"]     = DateTime.UtcNow;
    };
});
```

---

## Q72. Logging with Serilog.

### 📌 Definition
Serilog is a popular structured logging library for .NET that integrates with ASP.NET Core's `ILogger<T>` abstraction. It offers rich sinks (Console, File, Seq, Elasticsearch, Application Insights), property enrichers, log filtering per namespace, and a rolling file approach for production.

```csharp
// NuGet: Serilog.AspNetCore, Serilog.Sinks.File, Serilog.Sinks.Seq

// Program.cs:
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft.AspNetCore", LogEventLevel.Warning)
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithThreadId()
    .WriteTo.Console(outputTemplate:
        "[{Timestamp:HH:mm:ss} {Level:u3}] [{CorrelationId}] {Message:lj}{NewLine}{Exception}")
    .WriteTo.File("logs/trading-.log",
        rollingInterval: RollingInterval.Day,
        retainedFileCountLimit: 30,
        outputTemplate: "{Timestamp:yyyy-MM-dd HH:mm:ss.fff} [{Level}] [{CorrelationId}] {Message}{NewLine}{Exception}")
    .WriteTo.Seq("http://seq-server:5341") // Centralized log server
    .CreateLogger();

builder.Host.UseSerilog();

// Request logging middleware:
app.UseSerilogRequestLogging(opts =>
{
    opts.MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000} ms";
    opts.EnrichDiagnosticContext = (diag, ctx) =>
    {
        diag.Set("ClientIP",      ctx.Connection.RemoteIpAddress);
        diag.Set("UserAgent",     ctx.Request.Headers.UserAgent.ToString());
        diag.Set("CorrelationId", ctx.Items["CorrelationId"]);
    };
});
```

---

## Q73. Response compression.

### 📌 Definition
Response compression reduces the byte size of HTTP responses (JSON, HTML, text) before sending over the network, improving throughput for bandwidth-constrained clients. ASP.NET Core's `ResponseCompressionMiddleware` applies gzip or Brotli encoding transparently when the client sends `Accept-Encoding: gzip, br`. Not beneficial for already-compressed content (images, PDFs, binary).

```csharp
// Program.cs:
builder.Services.AddResponseCompression(opts =>
{
    opts.EnableForHttps = true; // Enable for HTTPS (BREACH attack consideration — only for non-secret responses)
    opts.Providers.Add<BrotliCompressionProvider>();   // Better compression ratio
    opts.Providers.Add<GzipCompressionProvider>();
    opts.MimeTypes = ResponseCompressionDefaults.MimeTypes.Concat(new[]
    {
        "application/json",
        "text/csv",
        "application/x-protobuf"
    });
});
builder.Services.Configure<BrotliCompressionProviderOptions>(o => o.Level = System.IO.Compression.CompressionLevel.Fastest);
app.UseResponseCompression(); // Must be before UseStaticFiles, UseRouting
```

---

## Q74. IHttpClientFactory – why and how.

*Covered in Q48. ASP.NET Core adds typed clients:*

```csharp
// Typed client — encapsulates HttpClient + base URL + default headers
public sealed class MarketDataClient
{
    private readonly HttpClient _http;
    public MarketDataClient(HttpClient http) => _http = http;
    public async Task<decimal> GetLatestPrice(string symbol)
        => decimal.Parse(await _http.GetStringAsync($"/price/{symbol}"));
}

// Registration:
builder.Services.AddHttpClient<MarketDataClient>(c =>
{
    c.BaseAddress = new Uri("https://api.nse.co.in");
    c.DefaultRequestHeaders.Add("X-API-Key", "SECRET");
    c.Timeout = TimeSpan.FromSeconds(5);
})
.SetHandlerLifetime(TimeSpan.FromMinutes(5))        // Rotate handler every 5min (DNS changes)
.AddTransientHttpErrorPolicy(p =>                   // Retry on transient errors
    p.WaitAndRetryAsync(3, retry => TimeSpan.FromSeconds(Math.Pow(2, retry))));
```

---

## Q75. What is OpenAPI/Swagger?

### 📌 Definition
**OpenAPI** (formerly Swagger) is a specification for describing REST APIs in a machine-readable format (JSON/YAML). ASP.NET Core can auto-generate OpenAPI documents from your controllers (via `Swashbuckle.AspNetCore` or the built-in `Microsoft.AspNetCore.OpenApi` in .NET 9). The document describes all endpoints, request/response schemas, auth requirements, and allows generating client SDKs, API explorers, and contract tests.

```csharp
// Program.cs (.NET 9 built-in):
builder.Services.AddOpenApi();
app.MapOpenApi();                // /openapi/v1.json
app.UseSwaggerUI(c => c.SwaggerEndpoint("/openapi/v1.json", "Trading API v1"));

// Swashbuckle (pre-.NET 9):
builder.Services.AddSwaggerGen(opts =>
{
    opts.SwaggerDoc("v1", new() { Title = "Trading API", Version = "v1" });
    opts.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Type = SecuritySchemeType.Http,
        Scheme = "bearer",
        BearerFormat = "JWT"
    });
    opts.AddSecurityRequirement(new OpenApiSecurityRequirement { ... }); // Apply to all endpoints
    opts.IncludeXmlComments(Path.Combine(AppContext.BaseDirectory, "TradingAPI.xml")); // Add XML doc comments
});

// Controller: XML doc comments appear in Swagger UI
/// <summary>Place a new order on the exchange</summary>
/// <param name="req">Order details including symbol, price, and quantity</param>
/// <returns>Created order with assigned order ID</returns>
[HttpPost]
[ProducesResponseType(typeof(OrderDto), 201)]
public IActionResult PlaceOrder([FromBody] OrderRequest req) { ... }
```

---

## Q76-Q80. Session, cookies, HttpContext, ActionContext, and IWebHostEnvironment.

### Q76. Session management

```csharp
// Session requires distributed cache (Redis for multi-server):
builder.Services.AddStackExchangeRedisCache(o => o.Configuration = redisConn);
builder.Services.AddSession(opts => {
    opts.IdleTimeout = TimeSpan.FromMinutes(30);
    opts.Cookie.HttpOnly = true;
    opts.Cookie.IsEssential = true;
});
app.UseSession();

// Usage in controller:
HttpContext.Session.SetString("TraderId", "TRADER-42");
string? id = HttpContext.Session.GetString("TraderId");
```

### Q77. Cookie management

```csharp
// Set cookie:
Response.Cookies.Append("SessionToken", token, new CookieOptions {
    HttpOnly  = true,    // Not accessible from JS
    Secure    = true,    // HTTPS only
    SameSite  = SameSiteMode.Strict,
    Expires   = DateTimeOffset.UtcNow.AddHours(1)
});
// Read:
string? token = Request.Cookies["SessionToken"];
// Delete:
Response.Cookies.Delete("SessionToken");
```

### Q78. HttpContext deep dive

```csharp
// HttpContext: the central per-request container
var ctx = HttpContext;
ctx.Request.Method;             // GET, POST...
ctx.Request.Path;               // /api/orders
ctx.Request.Query["page"];      // Query string
ctx.Request.Headers["Auth"];    // Headers
ctx.Request.Body;               // Request stream
ctx.Response.StatusCode = 200;
ctx.Response.Headers["X-Id"] = "abc";
ctx.User;                       // ClaimsPrincipal (from auth)
ctx.Items["CorrelationId"];     // Per-request data bag
ctx.Connection.RemoteIpAddress; // Client IP
ctx.RequestAborted;             // CancellationToken — fires on client disconnect
ctx.RequestServices;            // Scoped DI container for this request
```

### Q79. IWebHostEnvironment

```csharp
// Injected to detect environment:
public class OrderController(IWebHostEnvironment env) : ControllerBase
{
    [HttpGet("debug")]
    public IActionResult Debug()
    {
        if (!env.IsDevelopment()) return Forbid();
        return Ok(new { env.EnvironmentName, env.ContentRootPath });
    }
}
// Environment set by ASPNETCORE_ENVIRONMENT env var: Development | Staging | Production
// Custom: ASPNETCORE_ENVIRONMENT=UAT → env.IsEnvironment("UAT")
```

---

## Q81-Q85. Security topics.

### Q81. HTTPS and HSTS

```csharp
// Force HTTPS everywhere:
app.UseHttpsRedirection();  // Redirect HTTP→HTTPS (301/302)
app.UseHsts();              // HTTP Strict Transport Security header (min 1 year for prod)
// appsettings.json: "Hsts": { "MaxAge": 365 (days), "IncludeSubDomains": true, "Preload": true }
```

### Q82. Data Protection API

```csharp
// Encrypts sensitive data (session tokens, auth cookies, anti-forgery tokens):
builder.Services.AddDataProtection()
    .PersistKeysToAzureBlobStorage(blobUri)   // Persist keys across restarts/pods
    .ProtectKeysWithAzureKeyVault(kvUri, credentials); // Encrypt keys at rest

// Usage — protect/unprotect arbitrary payloads:
var protector = dataProtectionProvider.CreateProtector("OrderConfirmation");
string token  = protector.Protect("ORD-001:TRADER-42");   // Encrypted
string        = protector.Unprotect(token);                // Decrypted (throws if tampered)
```

### Q83. Anti-forgery tokens (CSRF)

```csharp
// Auto-included for Razor Pages and MVC forms
// For API: use JWT (stateless) — no need for CSRF tokens
// For cookie-auth APIs: add explicitly
builder.Services.AddAntiforgery(opts => opts.HeaderName = "X-CSRF-TOKEN");
[ValidateAntiForgeryToken] // Action filter validates token
```

### Q84. Secret management

```csharp
// Development: dotnet user-secrets init / dotnet user-secrets set "Jwt:Secret" "..."
// Production: Azure Key Vault, AWS Secrets Manager, HashiCorp Vault
builder.Configuration.AddAzureKeyVault(new Uri(kvUri), new DefaultAzureCredential());
// Access: configuration["Jwt:Secret"] — transparent, same as appsettings
```

### Q85. Input sanitization / XSS prevention

```csharp
// ASP.NET Core HTML-encodes by default in Razor views
// For JSON APIs: no XSS vector — JSON is not rendered as HTML by browser
// Sanitize only if storing user HTML and re-rendering:
// NuGet: HtmlSanitizer
var sanitizer = new HtmlSanitizer();
string safe = sanitizer.Sanitize(userHtml); // Strips dangerous tags/attrs
```

---

## Q86-Q90. Testing, ETag, Problem Details, Telemetry, Deployment.

### Q86. Integration testing

```csharp
// WebApplicationFactory<T> — full in-memory test server:
public sealed class OrderApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    public OrderApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.WithWebHostBuilder(b =>
            b.ConfigureServices(s =>
            {
                s.RemoveAll<IOrderRepository>();
                s.AddSingleton<IOrderRepository, InMemoryOrderRepository>(); // Test double
            })).CreateClient();
    }

    [Fact]
    public async Task PlaceOrder_ValidRequest_Returns201()
    {
        var resp = await _client.PostAsJsonAsync("/api/orders",
            new { symbol = "NIFTY50", price = 24500m, quantity = 10 });
        resp.StatusCode.Should().Be(HttpStatusCode.Created);
    }
}
```

### Q87. ETag and conditional requests

```csharp
// ETag: hash of response — client sends If-None-Match;
// server returns 304 Not Modified if unchanged (saves bandwidth)
[HttpGet("{orderId}")]
public IActionResult Get(string orderId)
{
    var order = FindOrder(orderId);
    string etag = $"\"{order.Version}\"";
    if (Request.Headers.IfNoneMatch == etag) return StatusCode(304);
    Response.Headers.ETag = etag;
    return Ok(order);
}
```

### Q88. Problem Details (RFC 7807)

```csharp
// Return standardized error format:
return Problem(
    type:   "https://trading.example.com/errors/order-not-found",
    title:  "Order Not Found",
    detail: $"No order with ID '{orderId}' exists",
    status: 404,
    instance: $"/api/orders/{orderId}");
// JSON: {"type":"...","title":"Order Not Found","status":404,"detail":"...","instance":"..."}
```

### Q89. OpenTelemetry / Distributed tracing

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddSqlClientInstrumentation()
        .AddOtlpExporter(o => o.Endpoint = new Uri("http://otel-collector:4317")))
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()
        .AddPrometheusExporter());
app.MapPrometheusScrapingEndpoint("/metrics");
```

### Q90. Deployment – Docker

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app --no-restore
FROM runtime AS final
WORKDIR /app
COPY --from=build /app .
ENV ASPNETCORE_URLS=http://+:8080
ENV ASPNETCORE_ENVIRONMENT=Production
EXPOSE 8080
ENTRYPOINT ["dotnet", "TradingAPI.dll"]
```

---

## Q91-Q101. Remaining ASP.NET Core Questions.

### Q91. Request size limits

```csharp
builder.Services.Configure<KestrelServerOptions>(o =>
    o.Limits.MaxRequestBodySize = 10 * 1024 * 1024); // 10MB
// Per-action override:
[RequestSizeLimit(5_000_000)]        // 5MB for this action
[DisableRequestSizeLimit]            // Unlimited (file uploads)
```

### Q92. File upload handling

```csharp
[HttpPost("upload")]
[RequestFormLimits(MultipartBodyLengthLimit = 50_000_000)] // 50MB
public async Task<IActionResult> Upload(IFormFile file, CancellationToken ct)
{
    if (file.Length == 0) return BadRequest("Empty file");
    var ext  = Path.GetExtension(file.FileName);
    if (ext is not ".csv" and not ".xlsx") return BadRequest("Only CSV/Excel");
    var path = Path.Combine("uploads", $"{Guid.NewGuid()}{ext}");
    await using var stream = System.IO.File.Create(path);
    await file.CopyToAsync(stream, ct);
    return Ok(new { path });
}
```

### Q93. Forwarded headers / load balancer

```csharp
// When behind NGINX/proxy: X-Forwarded-For, X-Forwarded-Proto
builder.Services.Configure<ForwardedHeadersOptions>(opts =>
{
    opts.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
    opts.KnownProxies.Add(IPAddress.Parse("10.0.0.1")); // Trust only known proxy IP
});
app.UseForwardedHeaders(); // Must be FIRST in pipeline
```

### Q94. Action execution filters order

```
Filter execution order:
1. Authorization filters     (global → controller → action)
2. Resource filters          (global → controller → action) — PRE
3. Model binding
4. Action filters            (global → controller → action) — PRE
5. Action method executes
4. Action filters            (action → controller → global) — POST (reverse)
3. Exception filters         (if exception)
2. Resource filters          (action → controller → global) — POST
1. Result filters            (global → controller → action) — PRE
   IActionResult executes
1. Result filters            (action → controller → global) — POST
```

### Q95. IActionFilter vs IAsyncActionFilter

```csharp
// Synchronous (runs filter code before/after action):
public class SyncFilter : IActionFilter {
    public void OnActionExecuting(ActionExecutingContext ctx) { }   // Before
    public void OnActionExecuted(ActionExecutedContext ctx)  { }    // After
}
// Async (preferred — await inside filter):
public class AsyncFilter : IAsyncActionFilter {
    public async Task OnActionExecutionAsync(ActionExecutingContext ctx, ActionExecutionDelegate next) {
        // Before
        var result = await next(); // Execute action
        // After — result.Result contains IActionResult
    }
}
```

### Q96. Response headers – Cache-Control semantics

```
Cache-Control: no-store                    → Don't cache at all (sensitive data)
Cache-Control: no-cache                    → Cache but always revalidate with server
Cache-Control: public, max-age=300         → Cache for 5 min (CDN-cacheable)
Cache-Control: private, max-age=60         → Browser only, 60s
Cache-Control: public, max-age=3600, must-revalidate  → 1 hour, then revalidate
```

### Q97. Dependency injection scopes in middleware

```csharp
// Middleware is singleton — cannot inject Scoped services in constructor
// ❌ Wrong:
public class MyMiddleware(RequestDelegate next, IOrderRepository repo) { } // Scoped in singleton!
// ✅ Correct — inject Scoped per-request in InvokeAsync:
public class MyMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext ctx, IOrderRepository repo) // Scoped injected here ✅
    {
        await next(ctx);
    }
}
```

### Q98. Action result short-circuiting

```csharp
// In IAsyncActionFilter: set context.Result to prevent action execution
public async Task OnActionExecutionAsync(ActionExecutingContext ctx, ActionExecutionDelegate next) {
    if (!ctx.HttpContext.User.Identity!.IsAuthenticated) {
        ctx.Result = new UnauthorizedResult(); // Action NEVER runs
        return;
    }
    await next(); // Action runs
}
```

### Q99. Custom InputFormatter (JSON5, YAML input)

```csharp
public sealed class YamlInputFormatter : InputFormatter
{
    public YamlInputFormatter() => SupportedMediaTypes.Add("application/x-yaml");
    public override async Task<InputFormatterResult> ReadRequestBodyAsync(InputFormatterContext ctx)
    {
        using var reader = new StreamReader(ctx.HttpContext.Request.Body);
        string yaml = await reader.ReadToEndAsync();
        var obj = YamlDeserializer.Deserialize(yaml, ctx.ModelType);
        return InputFormatterResult.Success(obj);
    }
}
// builder.Services.AddControllers(o => o.InputFormatters.Insert(0, new YamlInputFormatter()));
```

### Q100. Program.cs startup order

```csharp
// Correct Program.cs ordering:
var builder = WebApplication.CreateBuilder(args);
// 1. Add services (DI registrations)
builder.Services.AddControllers();
builder.Services.AddAuthentication(...);
// 2. Build the app
var app = builder.Build();
// 3. Configure middleware pipeline (ORDER MATTERS)
app.UseExceptionHandler("/error");
app.UseHttpsRedirection();
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
// 4. Run
app.Run();
```

### Q101. IStartupFilter (legacy) / WebApplication lifecycle

```csharp
// Run code on startup (EF migrations, warm-up):
builder.Services.AddHostedService<StartupTaskService>();
// Or via IHostApplicationLifetime events:
var lifetime = app.Services.GetRequiredService<IHostApplicationLifetime>();
lifetime.ApplicationStarted.Register(() => Console.WriteLine("App started"));
lifetime.ApplicationStopping.Register(() => Console.WriteLine("App stopping — drain connections"));
lifetime.ApplicationStopped.Register(() => Console.WriteLine("App stopped — cleanup done"));
```

---
