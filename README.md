# DIY Project Planner

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, offline-capable home improvement planner that unifies material lists, cost tracking, and contractor comparison in one consumer-friendly tool.

DIY Project Planner is an open-source project management tool built specifically for homeowners and DIY enthusiasts running multi-trade renovations. It replaces the patchwork of spreadsheets, note apps, and browser tabs that homeowners currently use to plan budgets, source materials, and evaluate contractors.

---

## Why DIY Project Planner?

- Capable tools like Wrike, Smartsheet, and Buildertrend are enterprise-oriented, with steep learning curves and pricing (Buildertrend starts ~$199/mo) that exclude consumers.
- Consumer apps like the Ahyslop DIY Project Planner and MOREPHO focus on task lists or contractor referrals but lack budget vs. actual variance tracking and bill-of-materials builders.
- Financial-focused tools (Remodelum, BudgetMyReno) skip task scheduling, photo documentation, and permit reminders.
- Design-focused tools (Planner 5D, Houzz, Magicplan) excel at visualisation but offer no structured cost tracking or contractor management for homeowners.
- No surveyed competitor offers offline-first operation, supplier price comparison across retailers, or jurisdiction-aware permit checklists — all critical for real on-site DIY use.

---

## Key Features

### Planning & Task Management

- Project creation with phases, task lists, dependencies, and estimated durations
- Progress checkboxes and status tracking per task
- Push notifications for task deadlines, quote expiry, and contractor follow-up reminders
- Photo documentation (before / during / after) attached to tasks or phases

### Materials & Cost Control

- Bill-of-materials builder with quantity, unit, unit cost, and supplier fields per project phase
- Real-time budget vs. actual dashboard covering materials, tools, and labour
- Supplier price comparison: multiple price entries per material with auto-lowest-cost highlight
- PDF/CSV export of material lists and cost summaries for contractors or mortgage lenders

### Contractor Management

- Side-by-side contractor quote entry per trade
- Rating, review notes, and award tracking
- Role-based sharing so homeowners can grant view access to contractors or co-owners

### Compliance & On-Site Use

- Permit and compliance checklist with reminders keyed to project type
- Offline-first local-first storage with background cloud sync for reliable on-site use without connectivity

---

## AI-Native Advantage

AI-assisted project scoping turns a plain-language description (e.g. "renovate my kitchen") into a structured phase plan with task list, material list, and budget range. Further AI augmentation candidates include estimating material quantities from room dimensions, surfacing relevant permit requirements based on project type and location, suggesting contractor red-flag criteria by trade, detecting budget-overrun anomalies across expense categories, and recommending lower-cost alternative materials that still meet specification.

---

## Tech Stack & Deployment

- Mobile-first design for iOS and Android with a responsive web companion
- Local-first data sync (e.g. SQLite + cloud sync) for reliable offline access on job sites
- Integration hooks for home improvement retailers (Home Depot, Lowe's) for live price lookups where APIs are available
- PDF/CSV export for material lists and cost summaries
- Role-based sharing for homeowners, contractors, and co-owners
- Push notifications for deadlines, follow-ups, and quote expiry

---

## Market Context

The candidate-projects catalogue rates this project as Complexity 4, with High domain availability and Medium demand. All surveyed incumbents are proprietary SaaS or mobile applications; no open-source competitor was identified. Consumer pricing in adjacent tools ranges from free (ad-supported) through ~$7.99–$59/yr for design and budgeting freemium tiers, while professional construction platforms such as Buildertrend start at ~$199/mo — leaving an open gap for a free, open-source, AI-native consumer DIY tool.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
