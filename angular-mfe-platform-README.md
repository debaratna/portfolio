<div align="center">

<h1>⚡ angular-mfe-platform</h1>

<p><strong>Production-grade Angular Microfrontend platform using Webpack Module Federation</strong><br/>
Designed for distributed teams — independent deploys, shared design system, zero runtime coupling.</p>

[![Angular](https://img.shields.io/badge/Angular-17%2B-DD0031?style=flat-square&logo=angular)](https://angular.dev)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.x-3178C6?style=flat-square&logo=typescript)](https://typescriptlang.org)
[![NX](https://img.shields.io/badge/NX-Monorepo-143055?style=flat-square&logo=nx)](https://nx.dev)
[![Module Federation](https://img.shields.io/badge/Module%20Federation-Webpack%205-8DD6F9?style=flat-square&logo=webpack)](https://webpack.js.org/concepts/module-federation/)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)](LICENSE)

</div>

---

## 🎯 Why This Exists

Most Microfrontend tutorials show you _how to wire up_ Module Federation. They don't show you what happens when:

- Your Shell and 5 remotes need **shared OIDC auth state** without a global variable
- A remote fails to load and the **Shell needs to stay alive** and recoverable
- Two squads deploy simultaneously and **version skew breaks shared libraries**
- You need to enforce **GDPR-compliant lazy loading** (don't load a remote until consent is given)

This repo is the reference implementation I wish existed when I started this work at scale.

---

## 🏗️ Architecture Overview

```
┌──────────────────────────────────────────────────────────┐
│                     Shell Host App                        │
│  ┌───────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │  Auth Remote  │  │Catalog Remote│  │Dashboard Remote│  │
│  │  (MFE-Auth)   │  │  (MFE-Cat)   │  │  (MFE-Dash)   │  │
│  └───────────────┘  └──────────────┘  └───────────────┘  │
│                                                           │
│  Shared:  @angular/core • @angular/router • design-system │
│  Signals-based cross-MFE store (no NgRx, no events bus)  │
└──────────────────────────────────────────────────────────┘
```

**Key decisions documented in [`/docs/decisions/`](./docs/decisions/):**
- `ADR-001` — Why Module Federation over iframes or Web Components  
- `ADR-002` — Signals store vs NgRx for cross-MFE state  
- `ADR-003` — OIDC token propagation strategy across remotes  
- `ADR-004` — Consent-gated lazy loading for GDPR compliance

---

## 📦 Repo Structure (NX Monorepo)

```
angular-mfe-platform/
├── apps/
│   ├── shell/              # Host application — routing + layout
│   ├── mfe-auth/           # Remote: Login, SSO, session management
│   ├── mfe-catalog/        # Remote: Product catalog, search, filtering
│   └── mfe-dashboard/      # Remote: Analytics, KPI widgets
├── libs/
│   ├── shared-ui/          # Design system components (standalone)
│   ├── shared-auth/        # OIDC service, guards, interceptors
│   ├── shared-store/       # Angular Signals cross-MFE state
│   └── shared-utils/       # Common pipes, validators, helpers
├── docs/
│   └── decisions/          # ADR files
└── tools/
    └── generators/         # NX custom generators for new MFEs
```

---

## ✨ Key Features

### 🔌 Module Federation Shell
- Dynamic remote discovery via `manifest.json` — no hardcoded URLs
- Graceful fallback UI when a remote fails to load
- Shared singleton `@angular/core`, `@angular/router`, and design system
- Route-level preloading strategy for critical remotes

### 🧠 Signals-Based Cross-MFE Store
No NgRx. No event bus. No `BehaviorSubject` spaghetti.

```typescript
// libs/shared-store/src/lib/user.store.ts
import { signal, computed } from '@angular/core';

export const userStore = {
  profile: signal<UserProfile | null>(null),
  permissions: signal<Permission[]>([]),
  isAuthenticated: computed(() => userStore.profile() !== null),

  setProfile(profile: UserProfile) {
    userStore.profile.set(profile);
  },

  hasPermission(permission: Permission): boolean {
    return userStore.permissions().includes(permission);
  }
};
```

Any remote reads from the same signal. No providers. No injection tokens. No ceremony.

### 🔐 OIDC-Aware Routing
```typescript
// Auth guard that works across Shell and all remotes
@Injectable({ providedIn: 'root' })
export class AuthGuard implements CanActivate {
  canActivate(route: ActivatedRouteSnapshot): Observable<boolean> {
    return this.authService.isTokenValid().pipe(
      tap(valid => { if (!valid) this.authService.initiateLogin(route.url); })
    );
  }
}
```

### 🇪🇺 GDPR-Compliant Lazy Loading
Remotes that handle personal data are **not loaded** until explicit user consent is recorded. Consent state is persisted and checked before the router activates any consent-gated route.

### 🏎️ NX Build Performance
- `affected:build` — only rebuild what changed
- Remote build cache shared via NX Cloud
- Custom generator: `nx generate @mfe-platform/tools:mfe --name=new-feature` scaffolds a fully wired remote in seconds

---

## 🚀 Getting Started

```bash
# Clone
git clone https://github.com/debaratna/angular-mfe-platform.git
cd angular-mfe-platform

# Install
npm install

# Run everything (Shell + all remotes) concurrently
npx nx run-many --target=serve --all --parallel

# Shell runs at http://localhost:4200
# mfe-auth  → http://localhost:4201
# mfe-catalog → http://localhost:4202
# mfe-dashboard → http://localhost:4203
```

---

## 🧪 Testing Strategy

| Layer | Tool | Coverage Target |
|-------|------|----------------|
| Unit (services, stores) | Jest | 100% branch |
| Component | Angular Testing Library | Critical paths |
| Integration (MFE boundary) | Cypress Component | Shell ↔ Remote contracts |
| E2E | Playwright | Core user journeys |

```bash
npx nx run-many --target=test --all        # All unit tests
npx nx run shell-e2e:e2e                   # E2E suite
```

---

## 📐 Design Decisions

**Why not Web Components?** Angular's DI system doesn't cross the Web Component boundary cleanly. You lose guards, interceptors, and shared services — the things that make large apps maintainable.

**Why not iframes?** URL sync, deep linking, keyboard navigation, and accessibility all become your problem. The complexity cost is higher than Module Federation.

**Why Signals over NgRx?** For cross-MFE shared state, NgRx requires injecting a store into every remote — that means version-coupling your store library. A plain signal object imported as a singleton avoids this entirely.

---

## 🤝 Contributing

This is a reference architecture, not a framework. If you've solved a Microfrontend problem differently at scale, open a Discussion. I'm particularly interested in:
- Alternative shared-state patterns
- CI/CD strategies for independent remote deploys
- Multi-tenant Shell configurations

---

<div align="center">

**Built from lessons learned running Microfrontend platforms in production at enterprise scale.**  
*Part of [debaratna's](https://github.com/debaratna) open engineering work.*

</div>
