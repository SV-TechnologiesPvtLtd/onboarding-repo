# Centralized Email Service

## 1. Overview

The **Centralized Email Service** is a shared repository designed to manage and maintain email templates for multiple products and applications.

Instead of creating and maintaining separate email templates for every project, this repository provides a centralized, reusable, component-based template architecture.

The goal is to maintain consistent email structures while allowing each product to have its own branding, theme, and styling.

For example, multiple products can share the same:

- Base email layout
- Header and footer components
- Common email styles
- Reusable email structures

At the same time, each product can define its own theme, including brand colors, typography, and other visual styles.

### Key objectives

- Centralize email templates across multiple projects.
- Create reusable layouts and components.
- Maintain product-specific branding and themes.
- Generate production-ready HTML email templates.
- Support backend integration, particularly with Go applications.
- Make template development, previewing, and maintenance easier.

> **Scope:** This repository is responsible for preparing reusable, styled email templates. Actual email delivery is handled by the consuming backend and its email provider.

---

## 2. Architecture and technology approach

The proposed implementation uses component-based HTML templates with external CSS stylesheets.

### 2.1 Primary approach: Go `html/template` + Juice

The primary approach is to use Go's standard `html/template` syntax for template definitions and [Juice](https://github.com/Automattic/juice) for CSS inlining.

The workflow is:

1. Write email templates using Go `html/template` syntax.
2. Create reusable components such as Header and Footer.
3. Define shared styles in `base.css`.
4. Define product-specific styles in separate theme stylesheets.
5. Use a Node.js/TypeScript build script to assemble the layout, components, and email content.
6. Process the assembled HTML through Juice to inline supported CSS rules.
7. Generate the final template files inside `dist/templates/`.
8. Allow the Go backend to parse and execute these compiled templates with dynamic data.

**Why this approach?**

- Go backends can consume the generated templates directly.
- Templates retain Go-compatible syntax.
- Shared components reduce duplication.
- External CSS makes styling easier to maintain.
- CSS inlining improves compatibility with email clients that have limited stylesheet support.

### 2.2 Alternative approach: Handlebars

Another possible approach is to use [Handlebars](https://handlebarsjs.com/) as the template engine.

Handlebars is a widely used templating language with implementations available in different programming environments.

**Advantages:**

- Can be integrated with Node.js and other backend environments.
- Supports reusable partials and helpers.
- Does not depend on Go-specific template syntax.
- May be more suitable if the repository needs to serve projects using different backend stacks.

**Trade-off:**

A Go backend cannot directly execute Handlebars templates using Go's standard `html/template` package. It would need a compatible Handlebars implementation, a separate rendering service, or another integration layer.

For the initial implementation, Go `html/template` is the preferred approach because the primary consumers are Go backends.

---

## 3. Styling strategy

The styling architecture separates shared styles from product-specific themes.

### 3.1 Base styles (`base.css`)

Contains common styles used across email templates, such as:

- Email resets
- Default typography
- Common layout rules
- Container styles
- Responsive styles
- Shared component styles

### 3.2 Product themes (`styles/themes/`)

Each product can have its own stylesheet.

For example:

```text
styles/
├── base.css
└── themes/
    ├── invoice-studio.css
    ├── sv-holidays.css
    └── another-product.css
```

Product themes can define:

- Primary and secondary brand colors
- Background colors
- Typography overrides
- Button styles
- Product-specific visual elements

This separation allows multiple products to share the same layout and components without sharing the same branding.

---

## 4. Source code structure

```text
email-service/
├── .env.example                  # Environment configuration template
├── package.json                  # Dependencies and npm scripts
├── tsconfig.json                 # TypeScript compiler setup
├── server.ts                     # Local preview server
│
├── public/                       # Static public assets
│   ├── icons/                    # Social and UI icons
│   └── illustrations/            # Email illustrations
│
├── dist/                         # Generated build output
│   └── templates/
│       └── auth/
│           ├── invite.html       # Compiled Go template
│           └── reset-password.html
│
└── src/                          # Source templates and build scripts
    ├── app.ts                    # Express preview application
    ├── build-template.ts         # Template assembly and CSS inlining
    │
    ├── layouts/
    │   └── BaseLayout.html       # Master email layout
    │
    ├── components/
    │   ├── Header.html           # Reusable header definition
    │   └── Footer.html           # Reusable footer definition
    │
    ├── templates/
    │   └── auth/
    │       ├── invite.html       # Invitation email content
    │       └── reset-password.html
    │
    ├── styles/
    │   ├── base.css              # Shared email styles
    │   └── themes/
    │       └── invoice-studio.css
    │
    ├── services/
    │   └── render.service.ts     # Template assembly and preview rendering
    │
    └── routes/
        └── email.routes.ts       # Express preview and render endpoints
```

### Directory responsibilities

| Directory/file | Responsibility |
|---|---|
| `public/` | Stores static assets such as logos, icons, and illustrations. |
| `dist/templates/` | Contains compiled HTML templates consumed by Go backends. |
| `src/layouts/` | Contains master email layouts. |
| `src/components/` | Contains reusable email components. |
| `src/templates/` | Contains individual email template definitions. |
| `src/styles/` | Contains shared CSS and product-specific themes. |
| `src/services/` | Handles template assembly, CSS processing, and preview rendering. |
| `src/routes/` | Defines preview and rendering endpoints. |
| `build-template.ts` | Runs the template compilation and CSS-inlining process. |
| `app.ts` | Configures the local Express preview application. |
| `server.ts` | Starts the local development server. |

---

## 5. Template composition

The template system follows a component-based structure.

A master layout provides the common HTML structure, while individual email templates provide their own content. Reusable components such as Header and Footer can be included in the master layout.

This allows multiple email templates to share a consistent structure without repeating the same HTML.

### 5.1 Layout: the master HTML structure

The layout defines the common structure shared by multiple emails.

Example: `BaseLayout.html`

```html
{{define "base"}}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>{{.Subject}}</title>
</head>
<body>
    {{template "header" .}}

    <main>
        {{block "content" .}}
            Default content
        {{end}}
    </main>

    {{template "footer" .}}
</body>
</html>
{{end}}
```

Here:

- `define "base"` gives the master layout a name.
- `template "header" .` inserts the header component.
- `block "content" .` defines a replaceable content section.
- `template "footer" .` inserts the footer component.
- `.` represents the current data context.

### 5.2 Components: reusable HTML sections

Components are defined using `{{define}}` and included using `{{template}}`.

Example: `Header.html`

```html
{{define "header"}}
<header>
    <img src="{{.LogoURL}}" alt="{{.CompanyName}}">
</header>
{{end}}
```

Example: `Footer.html`

```html
{{define "footer"}}
<footer>
    <p>© {{.Year}} {{.CompanyName}}</p>
</footer>
{{end}}
```

These components can be reused by multiple email templates, such as invitation, password-reset, and verification emails.

### 5.3 Email templates: individual email content

Each email template defines its own content block.

Example: `templates/auth/invite.html`

```html
{{define "content"}}
<h1>Hello, {{.RecipientName}}!</h1>

<p>You have been invited to join {{.CompanyName}}.</p>

<a href="{{.InvitationURL}}">
    Accept Invitation
</a>

{{if .ExpiresAt}}
    <p>This invitation expires on {{.ExpiresAt}}.</p>
{{end}}
{{end}}
```

The `content` block is inserted into the content area of the master layout.

The invitation email does not need to duplicate the header, footer, or overall HTML structure.

---

## 6. Understanding Go `html/template`

Go provides a standard library package called `html/template` for generating HTML documents using reusable templates and dynamic data.

It supports:

- Named template definitions.
- Template composition using `template` and `block`.
- Reusable components using `define`.
- Dynamic data using `{{.Field}}`.
- Conditional rendering using `{{if}}` and `{{else}}`.
- Iteration using `{{range}}`.
- Context-aware HTML escaping to help prevent injection vulnerabilities.

### 6.1 The main template actions

| Action | Purpose | Example |
|---|---|---|
| `define` | Defines a named template. | `{{define "header"}}...{{end}}` |
| `template` | Executes another named template. | `{{template "header" .}}` |
| `block` | Defines a named template and provides default content. | `{{block "content" .}}...{{end}}` |
| `if` | Conditionally renders content. | `{{if .ExpiresAt}}...{{end}}` |
| `else` | Provides the alternative branch of an `if`. | `{{else}}...` |
| `range` | Iterates over a slice, array, map, or other supported collection. | `{{range .Features}}...{{end}}` |
| `with` | Executes a block with a new data context when the value is non-empty. | `{{with .User}}...{{end}}` |

### 6.2 How Go combines the templates

The layout, components, and email body are separate source files. Go does not automatically discover and combine them based on their directory structure.

The backend must parse the required files together into a template set, or the build process must assemble them into a self-contained compiled template.

Conceptually, the composition looks like this:

```text
BaseLayout.html
│
├── Header component
│
├── Content block
│   └── Invite email content
│
└── Footer component
        │
        ▼
Go template execution
        │
        ▼
Final invitation HTML
```

The `.` passed to each nested template supplies its data context. In this example, the header, content, and footer all receive the same root data object.

### 6.3 How dynamic data is passed to Go templates

The Go backend supplies data when executing the template.

Example:

```go
data := map[string]interface{}{
    "Subject":       "You are invited!",
    "RecipientName": "Shraddha",
    "CompanyName":   "SV Technologies",
    "LogoURL":       "https://example.com/logo.png",
    "InvitationURL": "https://example.com/invite/abc123",
    "Year":          2026,
    "ExpiresAt":     "November 30, 2026",
}
```

The template accesses these values using dot notation:

| Go template syntax | Example value |
|---|---|
| `{{.Subject}}` | You are invited! |
| `{{.RecipientName}}` | Shraddha |
| `{{.CompanyName}}` | SV Technologies |
| `{{.InvitationURL}}` | Invitation URL |
| `{{.Year}}` | 2026 |

Here, `.` represents the current data context passed to the template.

#### Conditional rendering

```html
{{if .ExpiresAt}}
    <p>Your invitation expires on {{.ExpiresAt}}.</p>
{{else}}
    <p>Your invitation does not have an expiration date.</p>
{{end}}
```

#### Iterating over data

```html
<ul>
    {{range .Features}}
        <li>{{.}}</li>
    {{end}}
</ul>
```

The same email template can therefore be reused with different data without changing its HTML source.

### 6.4 How the Go backend executes a template

After the build process generates the compiled template, the Go backend loads and executes it.

Example:

```go
package main

import (
    "bytes"
    "html/template"
    "log"
)

func main() {
    tmpl := template.Must(
        template.ParseFiles("dist/templates/auth/invite.html"),
    )

    data := map[string]interface{}{
        "Subject":       "You are invited!",
        "RecipientName": "Shraddha",
        "CompanyName":   "SV Technologies",
        "InvitationURL": "https://example.com/invite/abc123",
        "Year":          2026,
    }

    var buffer bytes.Buffer

    err := tmpl.ExecuteTemplate(&buffer, "base", data)
    if err != nil {
        log.Fatal(err)
    }

    emailHTML := buffer.String()

    // Pass emailHTML to the email-sending service.
}
```

What happens here:

1. `template.ParseFiles()` reads and parses the compiled HTML template.
2. `template.Must()` makes the application fail immediately if parsing returns an error.
3. The backend prepares the dynamic data.
4. `ExecuteTemplate()` executes the named master template (`base`) using the supplied data.
5. The rendered HTML is written into a buffer.
6. `buffer.String()` provides the final HTML as a string.

Go's `html/template` performs contextual HTML escaping on dynamic values. This helps protect against HTML injection when rendering user-provided content.

> **Important:** `ParseFiles()` parses the files supplied to it. It does not automatically load separate source files from sibling directories. The generated output must include all definitions needed by the selected email.

### 6.5 What happens after template execution?

At this point, the template has become an HTML document with its dynamic values populated.

**Before Go execution:**

```html
<h1>Hello, {{.RecipientName}}!</h1>

<p>You have been invited to join {{.CompanyName}}.</p>

<a href="{{.InvitationURL}}">Accept Invitation</a>
```

**After Go execution:**

```html
<h1>Hello, Shraddha!</h1>

<p>You have been invited to join SV Technologies.</p>

<a href="https://example.com/invite/abc123">
    Accept Invitation
</a>
```

The backend can now pass this HTML string to an email delivery provider, such as:

- Amazon SES
- SendGrid
- SMTP
- Resend

The email provider handles the actual delivery to the recipient.

---

## 7. Compilation and rendering pipeline

The complete process has two distinct stages: build time and runtime.

### 7.1 Build-time pipeline (Node.js / TypeScript)

```text
SOURCE FILES
│
├── layouts/BaseLayout.html
│   ├── {{template "header" .}}
│   ├── {{block "content" .}}
│   └── {{template "footer" .}}
│
├── components/
│   ├── Header.html
│   └── Footer.html
│
├── templates/auth/
│   └── invite.html
│
└── styles/
    ├── base.css
    └── themes/invoice-studio.css
          │
          ▼
NODE.JS / TYPESCRIPT BUILD PROCESS
          │
          ├── 1. Reads source files
          ├── 2. Assembles layout + components + email body
          ├── 3. Injects base and product-specific CSS
          ├── 4. Runs Juice CSS inlining
          ├── 5. Preserves Go template actions
          │
          ▼
COMPILED TEMPLATE
dist/templates/auth/invite.html
```

### 7.2 Runtime pipeline (Go backend)

```text
COMPILED TEMPLATE
dist/templates/auth/invite.html
          │
          ▼
GO BACKEND
          │
          ├── 1. Parses compiled template
          ├── 2. Prepares dynamic data
          ├── 3. Executes named master template
          ├── 4. Generates final HTML
          │
          ▼
EMAIL PROVIDER
          │
          ├── Amazon SES
          ├── SendGrid
          ├── SMTP
          └── Resend
          │
          ▼
RECIPIENT RECEIVES EMAIL
```

### 7.3 Build time vs runtime

| Build time (Node.js) | Runtime (Go) |
|---|---|
| Reads source files | Loads compiled templates |
| Combines layout and components | Supplies dynamic data |
| Includes email-specific content | Evaluates conditions and loops |
| Injects CSS | Executes template actions |
| Inlines CSS using Juice | Generates final email HTML |
| Generates Go-compatible templates | Passes HTML to the email provider |

**Important:** The Node.js build process does not generate the final personalized email. It generates a reusable template that the Go backend can execute whenever an email needs to be sent.

---

## 8. Implementation considerations

1. **Unique template names:** Each email should have its own template set or uniquely named content block. Multiple email templates defining `"content"` in the same Go template set can overwrite one another.
2. **Complete definitions:** The compiled output must contain all required named definitions, including `base`, `header`, `footer`, and the email-specific content definition.
3. **Named execution:** Execute the intended master template by name, for example, `ExecuteTemplate(&buffer, "base", data)`.
4. **Custom functions:** Register custom Go template functions before parsing if templates need them.
5. **Juice compatibility:** Test that CSS inlining preserves Go template actions and does not alter conditional or dynamic expressions.
6. **HTML escaping:** Use `html/template` for untrusted dynamic HTML values. Avoid bypassing escaping unless the content is trusted and intentionally marked safe.
7. **Static assets:** Use publicly accessible HTTPS URLs for email images and icons. Email clients generally cannot access local project assets.
8. **Email client compatibility:** Test generated emails in relevant clients because CSS support varies across email clients.

---

## 9. Key concepts to remember

| Concept | Responsibility |
|---|---|
| Layout | Defines the common structure shared by emails. |
| Components | Provide reusable HTML sections. |
| Email templates | Define the content specific to each email. |
| Base styles | Provide shared styling and email resets. |
| Product themes | Define product-specific branding. |
| Node.js build process | Assembles templates and prepares CSS. |
| Go backend | Supplies data and executes templates. |
| Email provider | Delivers the generated email. |

The separation of these responsibilities makes the centralized approach reusable across multiple products.

---

## 10. Development workflow

The following is the intended high-level workflow when working on this repository:

1. Create or update the required source HTML template.
2. Reuse the existing layout and components wherever possible.
3. Add or update shared CSS or the relevant product theme.
4. Run the template build script.
5. Inspect the generated file in `dist/templates/`.
6. Preview the email and check its layout and styling.
7. Verify that Go template actions remain intact.
8. Test the generated template with the Go backend and representative data.

Use the repository's actual npm scripts in `package.json` for the exact development, preview, and build commands.

---

## 11. Future improvements

Potential improvements for the centralized email service include:

- Support for additional product themes.
- More reusable email components.
- Standardized template data contracts between frontend/template development and backend teams.
- Automated validation of required template definitions.
- Automated CSS-inlining and build checks.
- Email preview and testing for multiple email clients.
- Documentation for integrating new projects.
- A template registry documenting available emails and their required data fields.

---

## 12. References

- [Go `html/template` documentation](https://pkg.go.dev/html/template)
- [Go `text/template` documentation](https://pkg.go.dev/text/template)
- [Juice – CSS inlining](https://github.com/Automattic/juice)
- [Handlebars documentation](https://handlebarsjs.com/)
