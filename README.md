# Tech Design: Equipment App

| | |
|---|---|
| **Author** | _your name_ |
| **Status** | Draft |
| **Last updated** | _date_ |
| **Reviewers** | _Shell owners, Purse owners, tech lead_ |

> Lines marked **TBD** are things I still need to confirm with the team. Fill them in before the review.

---

## 1. Overview and context

### What we are building

Equipment is a new remote app in our micro-frontend setup. It will handle **TBD: short description, e.g. viewing and managing equipment records**. Users will reach it from the Shell app, the same way they reach Purse today.

### Where it fits

We already have two apps:

- **Shell**: the main app. It owns the domain, the top-level routing and (TBD) the login flow.
- **Purse**: an existing remote app built on Next.js 13.

Equipment will be the second remote app behind the Shell.

### Goals

- Ship Equipment as its own app that can be built and deployed without touching Purse.
- Keep the setup as close to Purse as possible, so the team only has to learn one pattern.
- Keep the changes needed in the Shell small.

### Non-goals

- Sharing live components between Equipment and Purse at runtime.
- Moving Purse to a different integration approach.
- Rewriting the Shell.

### People involved

| Role | Person / team |
|---|---|
| Product owner | TBD |
| Equipment dev team | TBD |
| Shell owners | TBD |
| Purse owners | TBD |

---

## 2. Requirements

### Functional

- TBD: list the main features, for example:
  - List and search equipment
  - View equipment details
  - Create and edit equipment
  - Role-based access (who can view, who can edit)
- Users must be able to move between Shell, Purse and Equipment without logging in again.

### Non-functional

| Area | Requirement |
|---|---|
| Performance | TBD, e.g. first page load under 2.5s on a normal connection |
| Availability | Equipment going down must not break Shell or Purse |
| Browser support | Same as Purse (TBD: list) |
| Accessibility | TBD, e.g. WCAG 2.1 AA |
| SEO | TBD, probably not needed if pages are behind login |

### Assumptions and constraints

- Purse is on Next.js 13, so Equipment should use the same version to avoid mismatches.
- Equipment has to be served under the Shell's domain.
- Purse does not have `@module-federation/nextjs-mf` in its `package.json`, and its build does not produce a `remoteEntry.js`. So it is not using Module Federation. *(TBD: confirm `basePath` in Purse's `next.config.js` to be sure it is Multi-Zones.)*
- We do not have the Shell repo, so Shell changes will need to be done by its owners.

---

## 3. Architecture

### High-level picture

```
                  Browser
                     |
                     v
              +-------------+
              |    Shell    |   (owns the domain and routing)
              +-------------+
                |           |
      /purse/*  |           |  /equipment/*
                v           v
          +---------+   +-----------+
          |  Purse  |   | Equipment |
          | (Next)  |   |  (Next)   |
          +---------+   +-----------+
                |           |
                v           v
             Backend APIs / services
```

### How Equipment plugs in

Equipment will be a normal, standalone Next.js app. It will use a `basePath` of `/equipment`, so all its pages and static files live under that path. The Shell forwards any request that starts with `/equipment` to the Equipment deployment.

This is the Multi-Zones approach. Section 9 compares it with Module Federation and explains why we picked it.

### Routing

- Equipment owns every URL under `/equipment/*`.
- No other app should use that prefix.
- Inside Equipment, pages live at their normal paths (`/list`, `/[id]`). Next.js adds the `/equipment` prefix on its own.

### Navigation

- Moving **inside** Equipment is a normal client-side navigation (fast, no reload). Use `next/link`.
- Moving **between** apps (Equipment to Purse, or Equipment to Shell) is a full page load. Use a plain `<a href>` for these links, not `next/link`. Using `next/link` across apps will not work properly because each app only knows its own routes.

### Changes needed in the Shell

The Shell owners need to add:

1. A rewrite rule for `/equipment` and `/equipment/:path*`.
2. An environment variable that holds the Equipment deployment URL (for example `EQUIPMENT_URL`).
3. A link or menu entry pointing to `/equipment`.

---

## 4. Tech stack

| Area | Choice | Notes |
|---|---|---|
| Framework | Next.js 13 | Same version as Purse |
| UI library | React (version used by Purse) | TBD: check the exact version |
| Language | TypeScript | TBD: confirm Purse also uses it |
| Styling | TBD | Use the same approach as Purse |
| State | TBD | Keep local state small; server data via the data-fetching library |
| Data fetching | TBD | e.g. SWR or React Query, matching Purse |
| Forms | TBD | |
| Testing | Jest + React Testing Library, Playwright or Cypress for e2e | TBD: match Purse |
| Lint / format | ESLint + Prettier | Copy Purse's config |
| Node version | TBD | Same as Purse |
| Package manager | TBD | Same as Purse |

The rule is simple: **if Purse already made a choice, we reuse it** unless there is a clear reason not to. Any difference should be listed here with a reason.

---

## 5. Shared concerns across zones

Since Shell, Purse and Equipment are separate apps, they do not share memory or code at runtime. These are the things we have to line up on purpose.

### Authentication and session

- TBD: how does login work today? (cookie, token, who issues it)
- Ideal setup: the Shell handles login and sets a cookie on the shared domain. Equipment reads that cookie on the server (in `getServerSideProps` or middleware) and on API calls.
- Equipment must not have its own separate login.
- Token refresh should be handled in one place (TBD: Shell or backend).

### Shared UI and design system

- TBD: is there a shared component package today?
- If yes, Equipment installs it and follows the same version rule as Purse.
- If no, we should create a small private package for common pieces (buttons, inputs, table, theme tokens) instead of copy-pasting between apps.

### Layout (header, footer, menu)

- TBD: who renders the header and menu, the Shell or each zone?
- Because a hard navigation reloads the page, the header can flicker if each app renders its own. We need to check how Purse handles this and do the same.

### State

- Anything kept only in memory (React state, Redux store) is lost when the user goes to another app.
- If data must survive a jump between apps, keep it in the URL, a cookie, or the backend.

### Shared utilities and types

- API types, date formatting, constants and similar code go in a shared package, or are duplicated with care if the amount is small.

### Analytics, logging, error tracking

- Use the same tools and the same event naming as Purse (TBD: which tools).
- Add an `app` tag (`equipment`) so events can be filtered.

---

## 6. Application structure

### Folder layout (proposed)

```
equipment/
├── public/
├── src/
│   ├── pages/            # Next.js routes
│   │   ├── _app.tsx
│   │   ├── index.tsx     # equipment list
│   │   └── [id].tsx      # equipment details
│   ├── components/       # UI pieces used by pages
│   ├── features/         # grouped by feature (list, details, forms)
│   ├── services/         # API calls
│   ├── hooks/
│   ├── types/
│   └── utils/
├── next.config.js
├── package.json
└── tsconfig.json
```

*TBD: adjust to match Purse's layout so developers can move between the two easily. Also confirm if Purse uses the `pages` router or the `app` router.*

### `next.config.js` (main parts)

```js
module.exports = {
  basePath: '/equipment',
  // Optional: use a separate prefix for static files so they
  // do not clash with other apps.
  // assetPrefix: '/equipment-static',
};
```

### Pages

| Route (public URL) | Purpose | Rendering |
|---|---|---|
| `/equipment` | List of equipment | TBD (SSR or CSR) |
| `/equipment/[id]` | Equipment details | TBD |
| TBD | | |

### Environment variables

| Name | Purpose |
|---|---|
| `NEXT_PUBLIC_API_URL` | Backend API base URL |
| TBD | |

---

## 7. Repository and delivery

### Repository

Equipment gets its **own repository** (polyrepo), the same way Purse has its own.

Why:

- The zones only depend on each other through URLs, so they do not need to live together.
- Each team can deploy without waiting for the others.
- We do not have to move Purse anywhere.

A monorepo (for example with Turborepo or Nx) would help if we end up with many zones and a lot of shared code. Right now that is not the case, so we skip it. We can revisit this later.

### Branching and review

- TBD: follow the same branching model as Purse (for example `main` + short-lived feature branches).
- All changes go through pull requests with at least one review.

### CI/CD

Pipeline steps:

1. Install dependencies
2. Lint and type check
3. Run unit tests
4. Build
5. Deploy to the target environment

| Environment | Deployed from | Notes |
|---|---|---|
| Dev | feature branches / `develop` | TBD |
| Staging | `main` | TBD |
| Production | tagged release | TBD |

Rollback: redeploy the previous build. Because Equipment is a separate deployment, rolling it back does not affect Shell or Purse.

### Local development

- Run Equipment on its own port (for example `3002`) with `npm run dev`. Open it at `http://localhost:3002/equipment`.
- To test it together with the Shell, point the Shell's `EQUIPMENT_URL` to `http://localhost:3002`.
- TBD: check how Purse is run locally with the Shell and use the same setup.

---

## 8. Security

- **Authentication**: Equipment trusts the session set by the Shell/login service. It must check the session on the server for every protected page and API call, not only in the browser.
- **Authorization**: role checks happen on the backend. The UI can hide buttons, but that is only for looks, not security.
- **Cookies**: `HttpOnly`, `Secure` and a correct `SameSite` value. The cookie domain must allow all three apps to read it (TBD: confirm current settings).
- **CORS**: since Equipment is served from the same domain through the Shell, most calls are same-origin. If Equipment calls a different origin, the backend must allow it explicitly.
- **Headers**: set CSP and other security headers. TBD: check if the Shell sets these globally or each app does it.
- **Secrets**: never put them in `NEXT_PUBLIC_*` variables. Keep them in the deployment environment or a secrets manager.
- **Dependencies**: run `npm audit` (or the team's scanner) in CI.
- **Input handling**: validate on the server, and escape anything that is rendered as HTML.

---

## 9. Solution via both Multi-Zones and Module Federation

There are two ways to plug Equipment into the Shell. Below is how each one would work for us.

### Option A: Multi-Zones

**How it works**

Equipment is a standalone Next.js app with `basePath: '/equipment'`. The Shell forwards `/equipment/*` requests to it.

**What we would do**

Equipment `next.config.js`:

```js
module.exports = {
  basePath: '/equipment',
};
```

Shell `next.config.js` (done by Shell owners):

```js
module.exports = {
  async rewrites() {
    return [
      {
        source: '/equipment',
        destination: `${process.env.EQUIPMENT_URL}/equipment`,
      },
      {
        source: '/equipment/:path*',
        destination: `${process.env.EQUIPMENT_URL}/equipment/:path*`,
      },
    ];
  },
};
```

**Pros**

- Built into Next.js. No extra plugin.
- No webpack config to write or maintain.
- Stable across Next.js upgrades.
- Each app has its own dependencies, so no version clashes on React.
- Easy to debug, since a problem stays inside one app.
- Same approach as Purse (as far as we can tell), so no new pattern for the team.

**Cons**

- Going between apps is a full page load, so in-memory state is lost.
- We cannot put a live Purse component inside an Equipment page.
- Shared UI has to come from a package, and each app upgrades it on its own.
- Every zone needs a unique `basePath`.

### Option B: Module Federation

**How it works**

Equipment is built as a *remote* that exposes modules through a `remoteEntry.js` file. The Shell (the *host*) loads that file at runtime and imports the exposed components. Next.js does not support this itself, so it needs the `@module-federation/nextjs-mf` plugin on top of webpack.

**What we would do**

Equipment `next.config.js`:

```js
const NextFederationPlugin = require('@module-federation/nextjs-mf');

module.exports = {
  webpack(config) {
    config.plugins.push(
      new NextFederationPlugin({
        name: 'equipment',
        filename: 'static/chunks/remoteEntry.js',
        exposes: {
          './EquipmentList': './src/components/EquipmentList',
        },
        shared: {
          // React and other shared libraries go here
        },
      })
    );
    return config;
  },
};
```

The Shell would also need a matching `remotes` entry, and its own federation setup. Purse would probably need one too if the Shell expects every remote to work this way. (This is example code only. The exact options depend on the plugin version.)

**Pros**

- Components can be loaded live from one app inside another.
- Navigation can feel like one single-page app, with no full reload.
- Shared libraries like React can be loaded once.

**Cons**

- No official Next.js support. It depends on a third-party plugin.
- The plugin has to match the Next.js version. Older plugin versions stopped working on newer Next 13 releases, so upgrades can break it.
- We need to understand webpack config, `exposes`, `remotes` and `shared` settings.
- Server-side rendering with federation is the hardest part to get right.
- React version mismatches between host and remote cause runtime errors (for example "invalid hook call").
- Purse and the Shell would likely need changes too, which is a lot of work for one new app.
- Changing an exposed component can break the Shell at runtime instead of at build time.

### Side-by-side

| | Multi-Zones | Module Federation |
|---|---|---|
| Support in Next.js | Built in | Third-party plugin |
| Needs webpack knowledge | No | Yes |
| Share live components | No | Yes |
| Moving between apps | Full page load | Can be client-side |
| Setup effort | Low | High |
| Upgrade risk | Low | Medium to high |
| Dependency isolation | Full | Shared, must be kept in sync |
| Changes needed in Purse | None | Probably yes |
| Changes needed in Shell | One rewrite rule | Federation setup + remote config |

### Decision

We go with **Multi-Zones**.

Reasons:

1. Purse does not use Module Federation, so Multi-Zones is the pattern that already fits.
2. It needs the smallest change in the Shell and none in Purse.
3. It is built into Next.js and is less likely to break when we upgrade.
4. The team does not need to learn webpack federation.
5. We do not currently need to show live Purse components inside Equipment.

### When to look at this again

- If we need to share live components or state between Equipment and Purse.
- If the full page load between apps becomes a real problem for users.
- If we end up with many zones that share a lot of UI.

---

## 1. Why we are writing this

We are building a micro-frontend system with three apps:

- **Shell**: the main app. It owns the layout and navigation and brings the other apps together.
- **ABC**: a remote app. Already built, using Next.js 13 with the App Router.
- **XYZ**: a remote app. Not built yet.

Before we write code for the Shell and XYZ, we need to decide where the code lives. This choice is hard to undo later, because it affects CI/CD, how we share code, how we version things, and how teams work day to day.

There are two options:

- **Polyrepo**: one Git repository per app.
- **Monorepo**: one Git repository that holds all the apps and the shared packages.

This section explains both, shows how each would look for us, and lists the pros and cons. It does not depend on how the apps are integrated at runtime. That is covered separately.

---

## 2. Polyrepo

### What it is

Each app has its own repository, its own `package.json`, its own pipeline and its own release cycle.

```
repo: shell      -> Next.js app
repo: abc        -> Next.js 13 app (exists today)
repo: xyz        -> Next.js app
repo: ui-shared  -> (optional) shared components, published to a private npm registry
```

### How it would work for us

- ABC stays where it is. We don't touch it.
- Shell and XYZ each get a new repo.
- Each repo builds, tests and deploys on its own.
- If we need shared code (design system, auth helpers, types), we publish it as a versioned package to a private registry such as GitHub Packages, npm private or Artifactory. Each app installs it like any other dependency.

### Pros

- **No migration for ABC.** It keeps its repo, history and pipeline. This is a real saving, since ABC is already built.
- **True independence.** Each team owns its repo, its release schedule and its tooling. One team's broken build does not block the others.
- **Clear ownership and access control.** Permissions, code owners and branch rules are simple because they are per repo.
- **Small and fast per repo.** Clone, install and CI times stay low because each repo only has one app.
- **Fits independent deployments.** The apps are deployed separately, so keeping the repos separate feels natural.
- **Easy to hand over.** A repo can move to another team or vendor without untangling anything.

### Cons

- **Shared code is painful.** Every change to a shared package needs: change, publish, bump the version in each consuming app, redeploy. This is slow, and people tend to skip it.
- **Version drift.** After a few months, each app may be on a different version of React, Next.js or the shared UI package. Nothing forces them to stay in line.
- **Problems show up late.** If one app upgrades a shared library in a way that breaks another app, we may only find out in staging or production, not at build time.
- **Cross-app changes are hard.** A change that touches Shell and XYZ together needs two PRs in two repos, merged and deployed in the right order.
- **Duplicated setup.** ESLint, TypeScript, Prettier, CI templates and Docker files get copied into every repo and slowly drift apart.
- **Harder to see the whole system.** Searching for "who uses this function" across repos is manual work.

---

## 3. Monorepo

### What it is

All apps live in one repository. Shared code lives in internal packages in the same repo. A workspace tool (pnpm workspaces, plus Turborepo or Nx) manages dependencies and builds.

```
repo: frontend-platform
  apps/
    shell/
    abc/
    xyz/
  packages/
    ui/              -> shared components
    auth/            -> shared auth helpers
    config/          -> shared eslint, tsconfig, prettier
    types/           -> shared types
```

### How it would work for us

- We move ABC into `apps/abc`. We should keep its Git history (for example with `git subtree` or `git filter-repo`) so we don't lose blame and log.
- Shell and XYZ are created directly in `apps/`.
- Shared code is imported from `packages/*` using workspace links. No publishing step.
- Each app still builds into its own output and is deployed on its own. The repo is shared, but the deployment is not. A monorepo does **not** mean a monolith.
- CI uses the build tool to detect which apps changed (`turbo run build --filter=...[origin/main]`, or `nx affected`) so we only build and test what is needed.

### Pros

- **Shared code is easy.** Change a shared package and every app sees it immediately, in the same PR. No publishing, no version bumps.
- **One version of key dependencies.** We can pin `react`, `react-dom` and `next` once at the root, which removes most of the version drift.
- **Atomic changes.** One PR can update Shell and XYZ together. Reviewers see the full impact in one place.
- **Problems show up early.** If a change breaks another app, CI for that PR fails. We find out before merge.
- **One setup for tooling.** One ESLint config, one TypeScript base config, one CI pipeline template.
- **Easy to search and refactor.** Renaming a shared function across all apps is one change.
- **Build caching.** Turborepo and Nx cache results, so unchanged apps don't rebuild.

### Cons

- **ABC needs to be migrated.** It means moving the code, fixing paths, and updating its pipeline and deploy setup. This is real work and has some risk, even if small.
- **We are pushed toward alignment.** If ABC must stay on Next 13 for a long time while we want a newer version for XYZ, a single-version policy gets awkward. Workspaces can hold different versions per app, but then we lose part of the benefit.
- **Tooling has a learning curve.** Someone has to own the workspace setup, the task graph, caching and CI filters. If it is set up badly, builds get slow.
- **CI can get slow as the repo grows.** It needs "affected only" builds and remote caching to stay fast.
- **Access control is coarser.** Everyone with access to the repo can see all apps. We can use `CODEOWNERS` to protect folders, but it is not the same as separate repos.
- **Larger blast radius for mistakes.** A bad change to a shared package can break all apps at once. Good tests and review rules are needed.
- **Teams are coupled in small ways.** Shared lockfile, shared main branch, shared CI queue.

---

## 4. Side by side

| Topic | Polyrepo | Monorepo |
| --- | --- | --- |
| Setup effort for ABC | None | Medium (migration needed) |
| Sharing code | Publish packages, bump versions | Import from workspace |
| Dependency alignment (React, Next) | Manual, easy to drift | Enforced at the root |
| Cross-app change | Several PRs, ordered deploys | One PR |
| Where breakages are caught | Often late (staging or prod) | In CI, before merge |
| Team independence | High | Medium |
| Access control | Per repo | Per folder (CODEOWNERS) |
| CI speed at scale | Fast per repo | Needs caching and affected builds |
| Tooling complexity | Low per repo, duplicated | Higher, but done once |
| Independent deploys | Yes | Yes (if pipelines are per app) |

---

## 5. Proposed direction

This is a proposal for discussion, not a final decision.

**Lean towards a monorepo** if:

- The same team (or closely working teams) owns all three apps.
- We expect a shared design system and shared auth.
- We want strict control over React and Next.js versions.

**Lean towards polyrepo** if:

- Different teams or vendors own different apps and need full autonomy.
- ABC will not be moved and must keep its own pipeline.
- Shared code is small and rarely changes.

A middle path is possible: keep ABC in its own repo for now, put Shell and XYZ together in a new monorepo, and publish the shared UI package to a private registry. We can move ABC in later once we are comfortable.

---

## Risks and open questions

| Item | Owner | Status |
|---|---|---|
| Confirm Purse uses `basePath` (Multi-Zones) | TBD | Open |
| Confirm how login and cookies work across apps | TBD | Open |
| Confirm who renders header and menu | TBD | Open |
| Shell owners agree to add the rewrite rule | TBD | Open |
| Next.js version and lock file match Purse | TBD | Open |
| Shared UI package: does one exist? | TBD | Open |

## References

- Next.js docs: Multi-Zones
- Purse repository: TBD (link)
- Backend API docs: TBD (link)
