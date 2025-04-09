
# Frontend Architecture: NX Monorepo (React + TypeScript)

This documentation describes the architectural structure and design principles used in this NX-based frontend monorepo. It enforces separation of concerns, domain-driven business logic organization, and maintainability at scale.

## 🔧 High-Level Structure

```
apps/
└── {app}/                     # Application entry (e.g., shop, admin)
    └── src/
        └── main.tsx          # Main entry point

libs/
└── {app}/
    ├── features/             # UI features (smart components)
    ├── entities/             # Domain logic per domain
    ├── ui/                   # App ui-kit
    ├── util/                 # Shared utilities (hooks, helpers, services, constants)
    └── infra/                # Infrastructure (3rd party integration wrappers)
```

---

## 🧩 Layer Responsibilities

### `features/`
- Contains UI **use-cases**.
- Feature exposes only its root component (features do not export hooks, state, utils and etc).
- No business logic here.
- Local-first: if the code is not domain related, put it inside the feature folder.
- May have local state. 

**Typical structure for a feature:**

```
features/ProductList/
├── components/               # UI elements (e.g., ProductItem)
├── hooks/                    # Feature-specific logic
├── helpers/                  # Non-domain display helpers
├── types/                    # Feature-specific types
├── ProductList.tsx           # Entry point for feature
└── index.ts                  # Feature public API
```

### `entities/`
- Centralized **business logic** organized by domain (`product`, `user`, `order`, etc.).
- Contains:
  - `model/`: type definitions and invariants
  - `api/`: API services or fetchers
  - `state/`: Zustand/Redux logic or slices
  - `helpers/`: business-rule-related helpers
  - `hooks/`: reusable hooks to access the data which **don't** perform any logic except requesting/sending the data.
  - `schemas/`: validation schemas (like zod or yup) for domain entities.
  - `data-sources/`: data source-like objects.

**Example:**

```
entities/product/
├── model/                   # product.model.ts
├── api/                     # product.api.ts
├── state/                   # product.slice.ts
├── helpers/                 # getProductDisplayName.ts
├── data-sources/            # product-data-source.ts
└── hooks/                   # useLoadProductsQuery.ts
```

### `ui/`
- Ui-kit of the app.
- Domain independent.
- Don't have an access to external data sources (api, global state)
- May have nested components (mostly for code splitting)

```
ui/Modal/
├── components/ # child components of the Modal
├── Modal.tsx 
├── Modal.css
└── index.ts
```

### `util/`
- Shared non-domain utilities.

Structure:

```
util/
├── hooks/                   # e.g., useMediaQuery
├── helpers/                 # e.g., formatDate
├── services/                # e.g., BrowserStorageService
└── constants/               # global configuration values
```

> ✅ Domain-specific utilities **never** go here. Keep this layer clean.

### `infra/`
- Integration with external tools, SDKs, services.
- No business rules here.

Examples:

```
infra/
├── logging/                 # e.g., sentry.ts
├── analytics/              # amplitude.ts, posthog.ts
├── i18n/                   # i18nConfig.ts
├── web-socket/             # WebSocketClient.ts
└── api-clients/            # auto-generated clients
```

---

## 🔄 Relationships & Rules

- `features/` ➡ may import from `entities/`, `ui/`, `util/`, `infra/`
- `entities/` ➡ may import from `utils/` and `infra/`, should never import from `features/`, `ui/`
- `ui/` ➡ may import from `utils/`, should never import from `features/`, `entities/`
- `util/` ➡ may import from `infra/`, should never import from `features/`, `entities/`, `ui`
- `infra` ➡ shouldn't use any other lib

Enforce import boundaries via ESLint rules or Nx module boundaries.

---

## 🧠 Decision Rationale

- **Domain-Driven Design (DDD) Alignment** — Organizing business logic explicitly by domain (entities) clearly separates stable domain rules from volatile presentation details. This increases clarity, maintainability, and scalability
- **Clear UI-Logic Separation** — Decoupling UI-specific logic (features) from the underlying domain logic (entities) allows independent evolution of UI without risking unintended business rule changes. `UserPageId` and `UserEditProfilePage` features may use the same `UserAPI`. Feature may use logic from multiple domains. For example `UserAppartmentCard` feature may use both `UserAPI` and `AppartmentAPI` to disaply the data.
- **Local-First Development Approach** — Promotes initially colocating related logic closely with the consumer, reducing premature abstraction. This approach avoids over-engineering while making refactoring straightforward when reuse becomes evident.
- **Controlled Reusability and Boundary Enforcement** — Strict import rules between layers (features, entities, ui, util, infra) explicitly prevent tight coupling, circular dependencies, and ensure architectural consistency through automated enforcement.
- **Relativly simple structure with strict rules** — Prevents from chaos and allows easily find files.

---

## ✅ Best Practices

- Apply local-first principle: colocate small logic until reuse appears.
- If a helper uses a domain model, it belongs in `entities/{domain}/helpers`.
- Use `infra/` only for integration layers — no domain knowledge allowed.
- Prefer readability over premature abstraction.
- Keep api fetching hooks (react-query, swr) simple. Create hooks that only fetch/submit the data. Move adapters, validatiors, mappers outside.

---

## 🧪 Example Use Case: Checkout Feature

- `features/Checkout/useCheckout.ts`:
    - Uses `entities/order/api/createOrder.ts`
    - Uses `entities/user/helpers/getUserFullName.ts`
    - Uses `infra/logging/sentry.ts`
- No business logic lives in the feature. It only orchestrates data flow.

---

## 🏁 Summary

| Layer     | Responsibility                       | Domain-aware | Example                         |
|-----------|--------------------------------------|----------------|----------------------------------|
| features  | UI use-cases                         | ❌            | ProductList, Checkout            |
| entities  | Business logic per domain            | ✅            | product/api/, user/helpers/      |
| ui        | App UI-kit                           | ❌            | Button, Card, Modal              |
| util      | Generic reusable helpers/hooks       | ❌            | useMediaQuery, formatDate        |
| infra     | SDK wrappers, external integration   | ❌            | sentry.ts, i18n.ts, wsClient.ts  |

This architecture provides a consistent, maintainable, and scalable pattern for frontend applications.

## 📋 Example
```
my-monorepo/
├── apps/
│   └── shop/                      # Application entry point for "shop"
│       └── src/
│           ├── main.tsx         # Main entry point
│           └── App.tsx          # Root component (e.g., includes routing)
│
└── libs/
    └── shop/
        ├── features/            # UI use-case features (smart components)
        │   └── ProductList/     
        │       ├── components/  # Feature child components
        │       │   ├── ProductItem
        │       │   └── ProductListContainer
        │       ├── hooks/       # Feature-specific hooks
        │       │   └── useProductList.ts
        │       ├── helpers/     # Display helpers (non-domain logic)
        │       │   └── formatProductPrice.ts
        │       ├── types/       # Feature-specific TypeScript types
        │       │   └── product.types.ts
        │       ├── ProductList.tsx  # Root feature smart component
        │       └── index.ts         # Public export for the feature
        │
        ├── entities/            # Domain logic (business rules), by domain
        │   ├── product/
        │   │   ├── model/       # Domain models, types, invariants
        │   │   │   └── product.model.ts
        │   │   ├── api/         # API services or fetchers for products
        │   │   │   └── product.api.ts
        │   │   ├── state/       # Global state management (Redux slice /Zustand store)
        │   │   │   └── product.slice.ts
        │   │   ├── helpers/     # Business rule helpers
        │   │   │   └── getProductDisplayName.ts
        │   │   ├── data-sources/
        │   │   │   └── product-data-source.ts
        │   │   └── hooks/       # Domain-specific data fetching hooks (no extra logic)
        │   │       └── useLoadProductsQuery.ts
        │   ├── user/            # Domain: User
        │   │   ├── model/
        │   │   │   └── user.model.ts
        │   │   ├── api/
        │   │   │   └── user.api.ts
        │   │   ├── state/
        │   │   │   └── user.slice.ts
        │   │   └── helpers/
        │   │       └── getUserFullName.ts
        │   └── order/
        │       ├── model/
        │       │   └── order.model.ts
        │       ├── api/
        │       │   └── createOrder.ts
        │       ├── state/
        │       │   └── order.slice.ts
        │       └── helpers/
        │           └── calculateOrderTotal.ts
        │
        ├── ui/                  # App UI-kit
        │   ├── Button.tsx
        │   ├── Card.tsx
        │   └── Modal/
        │       ├── components/
        │       │   └── ModalHeader.tsx
        │       ├── Modal.tsx 
        │       ├── Modal.css
        │       └── index.ts
        │
        ├── util/                # Generic, non-domain utilities
        │   ├── hooks/
        │   │   └── useMediaQuery.ts
        │   ├── helpers/
        │   │   └── formatDate.ts
        │   ├── services/
        │   │   └── BrowserStorageService.ts
        │   └── constants/
        │       └── dateFormats.ts
        │
        └── infra/               # External integrations (no business logic)
            ├── logging/         # Logging service wrappers (e.g., Sentry)
            │   └── LoggingService.ts
            ├── analytics/       # Analytics service wrappers (e.g., Amplitude)
            │   └── AnalyticsService.ts
            ├── i18n/            # Internationalization configurations
            │   └── TranslationService.ts
            ├── web-socket/      # WebSocket client wrappers
            │   └── WebSocketClient.ts
            └── api-clients/     # Auto-generated or custom API client wrappers
                └── SwaggerClient.ts
```
