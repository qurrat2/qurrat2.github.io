---
layout: post
title: "Migrating from .NET Framework 4.x to .NET 8: What Actually Changes"
date: 2026-04-27
description: "A practical field guide from someone who spent years on both sides of the divide. What ports well, what needs work, and what you rewrite from scratch."
categories: dotnet
tags: [dotnet, csharp, aspnetcore, migration, webapi]
---

I spent five years building enterprise apps on .NET Framework 4.x (WebForms, MVC 5, IIS, SQL Server, stored procedures, Crystal Reports) and the last year building new ones on ASP.NET Core 8. Here is what I gathered over the past year.

## What actually changed at the architecture level

Stop thinking about .NET 8 as "a new version." It is a new process model with many of the old names kept for continuity. Once you internalise the shape below, the rest of the migration clicks into place.

### 1. Process model

**Then:** your app was a DLL loaded by IIS through an HTTP module pipeline. IIS owned the process, authentication, and thread management. You wrote ASPX pages or controllers; IIS handled everything around them.

**Now:** your app is a console application that hosts Kestrel, a cross-platform HTTP server, inside a `WebApplication` built from `WebApplication.CreateBuilder`. You can still sit behind IIS as a reverse proxy, but the hosting contract inverted: you own the process, IIS is optional.
The single biggest consequence: your app is self-contained. You can run `dotnet run` on Linux, Windows, macOS, or inside a container. No more "works on my IIS."

### 2. Configuration

**Then:** `web.config` XML with `<appSettings>`, `<connectionStrings>`, and transforms for each environment.

**Now:** `appsettings.json`, layered with `appsettings.Development.json`, `appsettings.Production.json`, environment variables, and user-secrets. All glued together through `IConfiguration`.

```csharp
// .NET Framework 4.x
var conn = ConfigurationManager.ConnectionStrings["Default"].ConnectionString;
var timeout = int.Parse(ConfigurationManager.AppSettings["RequestTimeout"]);

// .NET 8
var conn = builder.Configuration.GetConnectionString("Default");
var timeout = builder.Configuration.GetValue<int>("RequestTimeout");
```

The payoff is bigger than it looks. Typed config, environment-aware overrides, and no XML transforms. Secrets stop living in source control the day you move to user-secrets locally and environment variables in prod.

### 3. Dependency injection

**Then:** third-party container (Autofac, Unity, Ninject) wired in `Global.asax` or `App_Start`. Every team picked a different one.

**Now:** built in. `builder.Services.AddScoped<IX, Y>()`. Scoped, Transient, Singleton lifetimes are first-class.

This single change is responsible for more of .NET Core's cleanliness than anything else. There is a standard default way to do DI now, even though third-party containers are still supported.

### 4. Pipeline

**Then:** HTTP modules and handlers in `web.config`, order defined by XML, cross-cutting concerns like auth or logging written as `IHttpModule`.

**Now:** middleware. Ordered, explicit, and written in C#.

```csharp
// .NET Framework 4.x — HttpModule registered in web.config
public class RequestLoggingModule : IHttpModule
{
    public void Init(HttpApplication context)
    {
        context.BeginRequest += (s, e) => { /* log */ };
    }
    public void Dispose() { }
}

// .NET 8 — middleware in Program.cs
app.Use(async (context, next) =>
{
    var sw = Stopwatch.StartNew();
    await next();
    logger.LogInformation("{Path} took {Ms}ms", context.Request.Path, sw.ElapsedMilliseconds);
});
```

Order matters; you can see the order at a glance; you can unit-test each middleware. This is a quality-of-life jump.

### 5. Startup

**Then:** `Global.asax.cs` with `Application_Start`, plus `App_Start/RouteConfig.cs`, `App_Start/WebApiConfig.cs`, `App_Start/BundleConfig.cs`, and a dozen partial friends.

**Now:** `Program.cs`. All of it. One file for small apps, refactored into extension methods for large ones.

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(/* options */);

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

Every concern is in one readable file. A senior engineer can audit startup in ninety seconds.

## What ports well

Not everything is a rewrite. These are the parts that survive the trip.

### Pure C# business logic

If your domain classes, services, and rules are plain C# with no dependency on `System.Web` or `System.Configuration`, they move across untouched. This is the biggest argument for writing your business logic as pure C# from day one: future-you gets a free upgrade path.

### ADO.NET with stored procedures

`SqlConnection`, `SqlCommand`, `SqlParameter`, `DataReader`. The programming model is largely unchanged, though modern apps typically use `Microsoft.Data.SqlClient` and benefit from improved async and performance behaviour. If your reporting hot paths call stored procedures via ADO.NET, they compile on .NET 8 with little more than a namespace swap from `System.Data.SqlClient` to the recommended `Microsoft.Data.SqlClient` NuGet package. Your T-SQL is untouched.

```csharp
// Runs identically on both .NET Framework 4.x and .NET 8
using var conn = new SqlConnection(connectionString);
using var cmd = new SqlCommand("sp_GetAssetsByDepartment", conn)
{
    CommandType = CommandType.StoredProcedure
};
cmd.Parameters.AddWithValue("@DepartmentId", id);
await conn.OpenAsync();
using var reader = await cmd.ExecuteReaderAsync();
// ...
```

This matters for anyone coming from a heavy-SQL enterprise shop. Your SQL muscle does not atrophy; it transfers.

### JSON serialisation (mostly)

`Newtonsoft.Json` still works on .NET 8 and will for a long time. `System.Text.Json` is the new default, and the migration is usually a find-and-replace plus a handful of custom converters. Plan for it, but do not fear it.

### Entity Framework (with asterisks)

**EF6 (6.4+)** can run on .NET 8, but it's not cross-platform-first and is not recommended for new development. EF Core is the long-term direction. The APIs look similar but the model under the hood is different.

## What ports with work

### ASP.NET MVC 5 → ASP.NET Core MVC

Concepts are the same: controllers, actions, views, model binding, filters. The APIs moved. `System.Web.Mvc.Controller` is now `Microsoft.AspNetCore.Mvc.Controller`. `HttpGet`/`HttpPost` attributes live in a different namespace. Action results, filters, and `HttpContext` all have cleaner equivalents.

A reasonable rule of thumb is one week per developer per 20-30 controllers, assuming the business logic is already factored out.

### Web API 2 → ASP.NET Core Web API

Easier than MVC 5. The programming model is very similar, and ASP.NET Core unified MVC and Web API into one stack, so you lose a layer of confusion.

### Forms auth / cookies → JWT or cookies on Core

ASP.NET Core still supports cookie authentication. If your enterprise app used forms auth on an internal intranet, you can keep cookies. If you are exposing APIs to JavaScript or mobile clients, JWT is the clean answer.

In HOP, the portfolio API I link to at the end, I went with `AddAuthentication().AddJwtBearer()` because the clients are HTTP API consumers. BCrypt for password hashing on the user entity, JWT issued by an `IJwtTokenService` in the Infrastructure layer. A couple hundred lines, all inspectable in one folder.

## What does not port

Be honest with yourself about these. Mis-scoping a migration by pretending things will port is a classic project failure mode.

### WebForms

There is no forward path. WebForms is not in .NET Core or .NET 8 and never will be. `Page_Load`, `ViewState`, server controls, code-behind: gone.

Your options for a WebForms app are:
1. **Stay on .NET Framework 4.8** (still supported as part of the Windows OS lifecycle and receiving security updates).
2. **Rewrite the UI** in ASP.NET Core MVC or Blazor. Keep the business logic and database.
3. **Blazor Server** is the closest "feel" to WebForms (server-rendered components with stateful interactions) without the page lifecycle nightmare.

### WCF server-side

WCF service hosts do not run on .NET 8. Clients have a compatibility package. For server-side, your choices are:
1. **CoreWCF** (community-driven, .NET Foundation project) if you need to keep existing contracts and bindings during the move.
2. **gRPC** if the clients are internal and you control both ends.
3. **ASP.NET Core Web API** (REST or JSON-RPC) for everything else.

### Windows-specific APIs

`System.Drawing`, certain `System.DirectoryServices` paths, COM interop, Crystal Reports: these either do not exist on non-Windows .NET 8 or require platform-specific NuGet packages. Crystal Reports specifically does not have a clean .NET 8 path; you either keep that module on .NET Framework 4.8 or replace the reporting stack.

## A decision framework

When a stakeholder asks "should we migrate?" walk them through this ladder. Pick the first answer that matches.

1. **Does it use WebForms or self-host WCF heavily?** → Rewrite is the honest answer. Typical scope: months, not weeks.
2. **Is it MVC 5 / Web API 2 / EF6 / ADO.NET?** → Incremental refactor. Move to ASP.NET Core MVC, upgrade to EF Core or keep EF6 behind the shim. Typical scope: 4 to 8 weeks for a medium app.
3. **Is it a business-logic library plus a thin API veneer?** → Port the library as-is, rewrite the API in ASP.NET Core. Typical scope: 1 to 3 weeks.
4. **Is the app stable, in maintenance, and not blocking anyone?** → Stay on .NET Framework 4.8. It is still supported. Migrating for the sake of migrating burns budget that could fund a new product.

## What the other side looks like

In HOP I deliberately picked the layout I wish enterprise .NET Framework apps had used:

```
src/
├── HOP.Api              ASP.NET Core host, controllers, Swagger, JWT
├── HOP.Application      Interfaces and service contracts
├── HOP.Domain           Pure C# entities, no external deps
├── HOP.Infrastructure   EF Core, persistence, auth services
└── HOP.Contracts        Request/response DTOs
```

Dependency direction points inward. The domain knows nothing about HTTP or EF Core. Five projects sounds heavy until you realise every modern .NET team converges on a version of this. The enterprise .NET Framework projects I worked on had the same concerns smeared across three folders inside one assembly; the layering was implicit and inconsistent between modules. Making it explicit is the biggest delta in readability.

## Migration checklist (a saner starting point)

1. Inventory your dependencies. Run `upgrade-assistant` (Microsoft's tool) in analysis mode. It tells you which NuGet packages have .NET 8-compatible versions and which do not.
2. Extract pure business logic into its own project first, on .NET Framework. This project becomes your bridge: it compiles against both frameworks via multi-targeting.
3. Write the new .NET 8 host alongside the old one. Run them both against the same database in a staging environment.
4. Migrate one endpoint at a time. Keep a reverse-proxy routing rule in front of both so the switch is per-route, not per-app.
5. Kill the old host only after every route has a green-light period in production.

This is the Strangler Fig pattern. It is boring, slow, and it does not fail spectacularly the way a big-bang rewrite does.

## Closing

Migration is not a technology decision. It is a business decision about which capabilities you need next and how much legacy friction you are willing to live with. The technology side is a solved problem.

If your app is on life support and nobody is asking for new features, staying on .NET Framework 4.8 is a perfectly professional answer. If your team is shipping new features and the framework is getting in the way, .NET 8 pays back the migration cost in months, not years.

Most shops are somewhere in the middle. The right answer there is almost always "incremental refactor, behind the Strangler Fig." Boring and effective.

---

*I build .NET backends for enterprise and SaaS clients. If you are scoping a migration or need a second opinion on an architecture, [say hi on LinkedIn](https://linkedin.com/in/qurrat2) or have a look at [HOP](https://github.com/qurrat2/HOP-Healthcare-Ops-Platform), the portfolio project this post draws on.*
