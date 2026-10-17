# Clean Architecture Cookie Authentication MVC

A sample ASP.NET Core MVC application demonstrating **form-based login, Cookie Authentication, and Role-Based Authorization** using a layered Clean Architecture approach.

The project separates presentation, application logic, domain rules, and infrastructure concerns to improve maintainability, testability, and scalability.

## 📑 Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [How It Works](#how-it-works)
  - [Cookie Authentication](#cookie-authentication)
  - [Role-Based Authorization](#role-based-authorization)
  - [Logout](#logout)
- [Running the Tests](#running-the-tests)
- [Technology Stack](#technology-stack)
- [Security Best Practices](#security-best-practices)
- [Future Enhancements](#future-enhancements)

---

## Overview

This project demonstrates how to implement secure form-based authentication in ASP.NET Core MVC using the built-in Cookie Authentication middleware.

After a successful login, the application creates an authenticated session using an encrypted authentication cookie. Role-based authorization restricts access to protected pages and controller actions according to the user's assigned roles.

### Key Features

- Form-based user login.
- ASP.NET Core Cookie Authentication.
- Role-Based Access Control (RBAC).
- Protected MVC controllers and actions using `[Authorize]`.
- Custom login, logout, and access-denied routes.
- Layered Clean Architecture.
- Separation of concerns across application layers.
- Support for unit and integration testing.
- Extensible architecture for future Identity and database integration.

## Architecture

The solution follows Clean Architecture principles, keeping business rules independent of external frameworks and infrastructure.

```text
┌──────────────────────────────────────────────────┐
│                  Web / MVC Layer                  │
│        Controllers, Views, View Models            │
├──────────────────────────────────────────────────┤
│                Application Layer                  │
│         Use Cases, DTOs, Service Interfaces       │
├──────────────────────────────────────────────────┤
│                  Domain Layer                     │
│          Entities, Value Objects, Rules           │
├──────────────────────────────────────────────────┤
│               Infrastructure Layer                │
│       EF Core, Repositories, Identity, Data       │
└──────────────────────────────────────────────────┘
```

### Architecture Principles

- **Separation of concerns:** Each layer has a clearly defined responsibility.
- **Dependency inversion:** Core business logic should not depend directly on infrastructure implementations.
- **Testability:** Application services and domain rules can be tested independently.
- **Maintainability:** New authentication providers and data sources can be introduced with minimal impact on other layers.

Dependencies should flow inward toward the domain and application abstractions, with infrastructure implementations supplied through dependency injection.

## Project Structure

The following structure illustrates the intended organization. Adjust folder and project names to match the actual solution.

```text
CleanArchitecture.CookieAuth.Mvc/
│
├── src/
│   ├── CleanArchitecture.Web/
│   │   ├── Controllers/
│   │   │   ├── AccountController.cs
│   │   │   ├── HomeController.cs
│   │   │   └── AdminController.cs
│   │   ├── Views/
│   │   │   ├── Account/
│   │   │   ├── Home/
│   │   │   └── Shared/
│   │   ├── Models/
│   │   ├── Program.cs
│   │   └── appsettings.json
│   │
│   ├── CleanArchitecture.Application/
│   │   ├── Interfaces/
│   │   ├── Services/
│   │   └── DTOs/
│   │
│   ├── CleanArchitecture.Domain/
│   │   ├── Entities/
│   │   └── Interfaces/
│   │
│   └── CleanArchitecture.Infrastructure/
│       ├── Data/
│       ├── Repositories/
│       └── Identity/
│
└── tests/
    └── CleanArchitecture.Tests/
```

**Note:** This is a reference structure. Not every listed controller, directory, or class necessarily exists in the current implementation.

## Prerequisites

Install the following tools:

- [.NET 9 SDK](https://dotnet.microsoft.com/download/dotnet/9.0)
- Visual Studio 2022 or Visual Studio Code with the C# extension
- Git
- A compatible database, if database-backed authentication is configured

Verify your .NET SDK installation:

```bash
dotnet --version
```

The installed SDK should support the target framework configured in the project.

## Getting Started

### 1. Clone the Repository

Replace the placeholder with your actual repository URL.

```bash
git clone <repository-url>
cd CleanArchitecture.CookieAuth.Mvc
```

### 2. Restore Dependencies

```bash
dotnet restore
```

### 3. Build the Solution

```bash
dotnet build
```

Resolve any compilation errors before running the application.

### 4. Configure Application Settings

Review `appsettings.json` and `appsettings.Development.json`, if present.

Configure the required settings for your environment, such as:

- Logging levels.
- Database connection strings, if applicable.
- Authentication and authorization options.

Do not commit passwords, production connection strings, or other secrets to source control.

### 5. Apply Database Migrations

If the project uses Entity Framework Core and migrations are configured, run the appropriate migration command.

For example:

```bash
dotnet ef database update \
  --project src/CleanArchitecture.Infrastructure \
  --startup-project src/CleanArchitecture.Web
```

This command assumes the infrastructure project contains the migrations and the web project provides the required design-time configuration. Adjust it to match the actual solution.

If authentication currently uses in-memory or hard-coded demonstration users, database migrations may not be required.

### 6. Run the Application

```bash
dotnet run --project src/CleanArchitecture.Web
```

Use the HTTPS URL printed in the terminal to open the application in your browser.

For example:

```text
https://localhost:5001
```

The actual port may differ depending on the project's launch settings.

## How It Works

### Cookie Authentication

ASP.NET Core Cookie Authentication manages the authenticated session by issuing an encrypted authentication cookie after successful sign-in.

Register the authentication middleware and configure its routes in `Program.cs`.

Example configuration:

```csharp
using Microsoft.AspNetCore.Authentication.Cookies;

builder.Services
    .AddAuthentication(
        CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(options =>
    {
        options.LoginPath = "/Account/Login";
        options.LogoutPath = "/Account/Logout";
        options.AccessDeniedPath = "/Account/AccessDenied";

        options.ExpireTimeSpan = TimeSpan.FromMinutes(60);
        options.SlidingExpiration = true;

        options.Cookie.HttpOnly = true;
        options.Cookie.SecurePolicy =
            CookieSecurePolicy.Always;
        options.Cookie.SameSite = SameSiteMode.Lax;
    });
```

Ensure the middleware pipeline includes authentication and authorization in the correct order:

```csharp
app.UseRouting();

app.UseAuthentication();
app.UseAuthorization();

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");
```

After validating the submitted credentials, the application can create a `ClaimsPrincipal` and sign in the user.

Example:

```csharp
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Authentication.Cookies;
using System.Security.Claims;

var claims = new List<Claim>
{
    new Claim(ClaimTypes.Name, username),
    new Claim(ClaimTypes.Role, "Admin")
};

var identity = new ClaimsIdentity(
    claims,
    CookieAuthenticationDefaults.AuthenticationScheme);

var principal = new ClaimsPrincipal(identity);

await HttpContext.SignInAsync(
    CookieAuthenticationDefaults.AuthenticationScheme,
    principal);
```

The example demonstrates the sign-in mechanism. In a real application, validate the user's credentials against a trusted user store before creating the claims. Never grant a role based solely on user-supplied input.

### Role-Based Authorization

Role-based authorization restricts access to controllers or actions based on claims associated with the authenticated user.

Example: Allow only administrators to access the administration dashboard.

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

[Authorize(Roles = "Admin")]
public IActionResult AdminDashboard()
{
    return View();
}
```

Example: Allow administrators and managers to access reports.

```csharp
[Authorize(Roles = "Admin,Manager")]
public IActionResult Reports()
{
    return View();
}
```

Example: Require authentication for an entire controller.

```csharp
[Authorize]
public class DashboardController : Controller
{
    public IActionResult Index()
    {
        return View();
    }
}
```

### Authorization Behavior

| Scenario | Expected behavior |
|---|---|
| Unauthenticated user accesses a protected page | Redirected to the login page for a normal browser request |
| Authenticated user has the required role | Access is granted |
| Authenticated user lacks the required role | Access is denied or redirected to the access-denied page |
| User signs out | Authentication cookie is cleared through the configured sign-out flow |

Exact behavior can vary depending on cookie events, endpoint configuration, and framework version.

### Logout

The application should sign out the user using the same authentication scheme used during login.

Example:

```csharp
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<IActionResult> Logout()
{
    await HttpContext.SignOutAsync(
        CookieAuthenticationDefaults.AuthenticationScheme);

    return RedirectToAction("Login", "Account");
}
```

Use a POST request for logout and include anti-forgery protection when using MVC forms.

## Running the Tests

Run all tests from the solution root:

```bash
dotnet test
```

To build and test the solution together:

```bash
dotnet build
dotnet test --no-build
```

If the test project exists at `tests/CleanArchitecture.Tests`, you can target it directly:

```bash
dotnet test tests/CleanArchitecture.Tests
```

Recommended test scenarios include:

- Valid login credentials.
- Invalid login credentials.
- Access to authenticated pages.
- Access to administrator-only pages.
- Access-denied behavior for users without the required role.
- Successful logout and subsequent access to protected pages.
- Anti-forgery validation for form submissions.

## Technology Stack

| Technology | Purpose |
|---|---|
| ASP.NET Core MVC | Web application and controller-based routing |
| .NET 9 | Application runtime and framework |
| C# 13 | Programming language |
| Cookie Authentication | Authenticated browser sessions |
| Claims-Based Identity | Represents user identities and roles |
| Role-Based Authorization | Controls access to protected resources |
| Entity Framework Core | Database access, if configured |
| Bootstrap 5 | Responsive UI styling, if included |
| xUnit / NUnit / MSTest | Testing, depending on the configured test project |

Version and package details should be verified against the project's `.csproj` files.

## Security Best Practices

- Validate credentials using a trusted user store.
- Never store plain-text passwords.
- Use a secure password hashing mechanism or a supported identity framework.
- Enforce HTTPS in deployed environments.
- Protect authentication cookies with `HttpOnly`, `Secure`, and an appropriate `SameSite` policy.
- Keep authentication and authorization middleware correctly configured.
- Assign roles only after trusted server-side validation.
- Protect state-changing form submissions against CSRF.
- Store secrets outside source control.
- Apply least-privilege access policies.
- Configure cookie expiration and sign-out behavior appropriately.

## Future Enhancements

Potential enhancements include:

- Integrate ASP.NET Core Identity.
- Add database-backed user and role management.
- Implement password reset and account recovery.
- Add account lockout and login throttling.
- Add audit logging for authentication events.
- Implement policy-based authorization.
- Add comprehensive unit and integration tests.
- Introduce CI/CD pipelines using GitHub Actions or Azure DevOps.
- Add centralized exception handling and structured logging.

## License

This project is intended for learning and demonstration purposes. Add an appropriate `LICENSE` file if you plan to distribute or reuse the code publicly.
