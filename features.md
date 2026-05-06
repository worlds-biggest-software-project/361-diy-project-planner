# DIY Project Planner — Feature & Functionality Survey

> Candidate #361 · Researched: 2026-05-04

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| HomeZada | Consumer web + mobile | Freemium; paid plans from ~$59/yr | https://www.homezada.com |
| Remodelum | Consumer web | Free (core); Premium tier | https://www.remodelum.com |
| BudgetMyReno | Consumer web | Freemium; 30-day free trial | https://budgetmyreno.com |
| MOREPHO | Consumer mobile (iOS + Android) | Free | https://www.morepho.com |
| Renovately | Consumer mobile (iOS) | Free (AI features may be paid) | https://apps.apple.com/us/app/renovately-home-renovation/id1440401045 |
| Magicplan | Professional mobile (iOS + Android) | Freemium; plans from ~$9.99/mo | https://magicplan.app |
| Planner 5D | Consumer + pro web + mobile | Freemium; plans from ~$7.99/mo | https://planner5d.com |
| Houzz | Consumer + pro web + mobile | Free for homeowners; Pro subscription | https://www.houzz.com |
| DIY Project Planner (Ahyslop) | Consumer mobile (Android) | Free with ads | https://play.google.com/store/apps/details?id=com.ahyslop.diyprojectplanner |
| Buildertrend | Professional web + mobile | Commercial; starts ~$199/mo | https://buildertrend.com |

---

## Feature Analysis by Solution

### HomeZada

**Core features**
- Home improvement project planning with tasks, phases, and milestones
- Budget creation with itemised material and labour line items
- Expense logging with receipt photo uploads
- Document storage (permits, warranties, contractor bids, photos)
- Home inventory and maintenance scheduling alongside projects
- ROI estimation for completed improvements
- Before/during/after photo documentation per project

**Differentiating features**
- Visual Design AI: upload a photo of a real room and test different design schemes
- Homeowner AI: generates material lists with estimated quantities and unit prices from project descriptions
- Integrates project data with whole-home financial reporting (home value tracking, net worth)

**UX patterns**
- Dashboard-centric home page showing active projects, upcoming maintenance, and budget summaries
- Structured project wizard guides users through phases: planning → budgeting → execution → close-out
- Web-first with companion mobile app; not optimised for on-site mobile use

**Integration points**
- Links out to retailer product pages for saved items
- PDF/CSV export of budgets and project summaries
- No publicly documented third-party API

**Known gaps**
- No supplier price comparison across multiple vendors
- No native contractor quote comparison module
- Offline mode limited; requires connectivity for most features
- No permit checklist or compliance reminder system

**Licence / IP notes**
- Proprietary SaaS; no open-source components identified

---

### Remodelum

**Core features**
- Renovation budget creation with category breakdown (labour, materials, permits, etc.)
- Real-time expense tracking vs. planned budget with variance display
- Invoice and payment schedule management
- Zip-code-based cost estimation using local construction cost data
- Contractor quote comparison
- Collaborative sharing with spouse, contractor, or designer

**Differentiating features**
- Cost estimator doubles as a live tracking budget, keeping planning and tracking in a single view
- Zip-code-accurate local cost benchmarks for common renovation tasks
- Payment schedule management tied to contractor milestones

**UX patterns**
- Clean single-page budget view; minimal onboarding friction
- Web-first; works across devices via browser (no dedicated mobile app listed)
- Focuses on financial management; no task scheduling or photo documentation

**Integration points**
- Document storage for invoices and contracts
- No retailer or contractor marketplace integration

**Known gaps**
- No task scheduling or dependency tracking
- No photo documentation per task or phase
- No permit/compliance reminders
- No offline capability
- No material quantity or bill-of-materials builder

**Licence / IP notes**
- Proprietary; free-tier model

---

### BudgetMyReno

**Core features**
- Unlimited projects and expense transactions
- Invoice scanning via photo (OCR auto-categorisation)
- Customisable expense categories with advanced search/filter
- Visual budget snapshot per category
- Payment schedule tracking with due-date management
- Audit trail of expense changes
- Collaborative project sharing

**Differentiating features**
- Smart invoice scanning reduces manual data entry
- Full audit trail distinguishes it from simpler trackers
- Unlimited projects on free tier

**UX patterns**
- Web-based; responsive across devices
- Expense-entry-first UX; budget view is the primary screen
- 30-day premium trial with no credit card required lowers adoption friction

**Integration points**
- No documented API or retailer integrations

**Known gaps**
- No task management or scheduling
- No contractor comparison module
- No design or visualisation features
- No material list / bill-of-materials builder
- No offline access

**Licence / IP notes**
- Proprietary SaaS; freemium model

---

### MOREPHO

**Core features**
- Project cards per home area with description, photos (up to 14), and checklists
- Do-it-by date with daily savings goal calculation
- "Request a Pro" toggle to share project with local contractors for proposals
- Q&A between homeowner and pros on project cards
- Local hardware store recommendations (hand-curated)
- Done Archive for completed projects (useful for resale/tax documentation)

**Differentiating features**
- Direct contractor discovery and proposal request within the same app
- Q&A workflow between homeowner and pros before project award
- Done Archive framed as a resale and tax document

**UX patterns**
- Mobile-first card-based UX
- Progressive disclosure: simple checklist by default; detail added as needed
- Free with no account required to browse; account needed to request pros

**Integration points**
- Connects to local contractor and hardware store networks
- No documented API

**Known gaps**
- No budget tracking beyond daily savings goal
- No multi-phase scheduling or dependency management
- No material quantity builder
- No document or permit storage
- Very limited cost tracking compared to HomeZada or Remodelum
- No offline mode

**Licence / IP notes**
- Free app; proprietary; monetises via contractor referral network

---

### Renovately

**Core features**
- To-do lists with labour tasks and shopping (material) items distinguished
- Due dates and reminders per task
- Nested checklists within to-dos
- Real-time budget roll-up at task, project, and home level
- Contractor quote recording
- Mood board for inspirational photos
- Monthly improvement summary report
- Collaborative sharing with spouse or designer

**Differentiating features**
- Renovately AI: Gen AI co-pilot for brainstorming, auto-generating task lists, and instant budget guidance
- Clear distinction between labour tasks and material shopping items within a unified to-do
- Budget visible at task, project, and whole-home levels simultaneously

**UX patterns**
- iOS-only; mobile-first
- Task-entry-first; budget computed from tasks rather than entered separately
- AI co-pilot accessible inline within project planning flow

**Integration points**
- No documented third-party API or retailer integrations

**Known gaps**
- iOS-only (no Android, no web)
- No floor-plan or space visualisation
- No permit/compliance checklist
- No offline mode documented
- No supplier price comparison

**Licence / IP notes**
- Proprietary; free download with potential in-app purchases for AI tier

---

### Magicplan

**Core features**
- LiDAR-powered automatic floor-plan scanning (Apple RoomPlan API)
- AR corner/wall-detection room measurement
- 2D and 3D floor-plan creation and editing
- Object/equipment catalog (doors, windows, plumbing, appliances, furniture)
- Forms, checklists, and notes attached to rooms or objects
- 360° panorama capture
- Price lists and cost estimates per room
- Bluetooth laser device integration (Bosch, Leica, Stanley)
- Multiple export formats: PDF, DXF, PNG, ESX (Xactimate), FML (Cotality)

**Differentiating features**
- Millimetre-accurate measurements via Bluetooth laser pairing
- Xactimate/Cotality export targets insurance and restoration professionals
- LiDAR auto-scan distinguishes it from all consumer DIY apps

**UX patterns**
- Scan-first workflow; floor plan is the central artefact
- Primarily professional/trade UX; consumer use possible but secondary
- Photo and note attachment directly on floor-plan elements

**Integration points**
- Xactimate and Cotality (Verisk) integration
- 20+ Bluetooth laser device ecosystem
- No homeowner retailer integrations

**Known gaps**
- No budget tracking or cost management for homeowners
- No contractor comparison
- No permit reminder system
- No collaborative sharing with non-professionals
- Learning curve too high for casual DIY users

**Licence / IP notes**
- Proprietary; freemium with professional subscription tiers

---

### Planner 5D

**Core features**
- 2D and 3D floor-plan creation with drag-and-drop interface
- Catalogue of 8,400+ furniture and décor items
- AI-generated room layouts from text descriptions
- 4K photorealistic rendering
- AR room visualiser (see layout at real scale in your space)
- Switch between 2D/3D/flat perspective modes
- Cross-platform: web, Windows, Mac, iOS, Android
- Community templates and sharing

**Differentiating features**
- One of the largest furniture/item catalogs in consumer design tools
- Apple Vision Pro support for spatial computing previews
- AI layout generation from text prompt

**UX patterns**
- Design-first; floor plan is the starting point
- No prior CAD/design skill required
- Project sharing for feedback from friends or designers

**Integration points**
- No material procurement or retailer pricing integration
- No contractor directory integration
- No documented API for third-party tools

**Known gaps**
- No budget, cost tracking, or expense management
- No contractor management features
- No task scheduling or project phases
- No permit/compliance tools
- Design focus only; no project management

**Licence / IP notes**
- Proprietary; freemium; paid plans unlock rendering and premium items

---

### Houzz

**Core features**
- Browse 25 million+ high-resolution home design photos
- Shop 5 million+ products with Visual Match (image recognition)
- View in My Room 3D: AR product placement in your space
- Ideabooks for saving and organising design inspiration
- Directory of 3 million+ home professionals with reviews and portfolios
- Sketch and annotate directly on design photos
- Biweekly newsletter with renovation guides and home tours
- Houzz Pro (separate app): construction management, client portal, scheduling, invoicing for contractors

**Differentiating features**
- Largest consumer home design photo library in the market
- Visual Match AI finds purchasable products from any design photo
- Seamless inspiration-to-purchase flow
- Dual product: consumer Houzz + professional Houzz Pro with shared data

**UX patterns**
- Inspiration/browse-first; discovery is the dominant pattern
- Product purchasing is tightly integrated into the browsing experience
- Professional connection initiated organically from project/design context

**Integration points**
- E-commerce product purchasing within app
- Houzz Pro integrates with client billing and scheduling for contractors
- No documented public API for homeowner data

**Known gaps**
- No structured project budget or cost tracking for homeowners
- No material quantity/bill-of-materials builder
- No permit checklist
- No offline support
- Weak task/schedule management for homeowners (Houzz Pro has scheduling, but aimed at contractors)

**Licence / IP notes**
- Proprietary; free for homeowners; Houzz Pro is commercial subscription

---

### DIY Project Planner (Ahyslop, Android)

**Core features**
- Project list with icon and colour coding
- Task list per project with progress checkboxes
- Expense logging per project
- Basic photo attachment per project
- Simple notes field

**Differentiating features**
- Extremely lightweight; fast to get started
- Free with no account required

**UX patterns**
- List-first; minimal chrome
- Aimed at single-person DIY users; no collaboration

**Integration points**
- None

**Known gaps**
- No budget vs. actual variance tracking
- No contractor comparison
- No material quantity/bill-of-materials builder
- No permit/compliance tools
- No offline mode distinguishable from regular local storage
- No scheduling or dependency management
- Ad-supported; users report intrusive ads
- No iOS version

**Licence / IP notes**
- Free; proprietary; ad-supported

---

### Buildertrend

**Core features**
- End-to-end construction project management (scheduling, budgeting, subcontractor management)
- Homeowner client portal with progress updates and approval workflows
- Digital takeoff and estimating tools
- Document management and blueprint storage
- Change order management with client approval
- Time tracking for crews
- Job cost reporting (estimated vs. actual)
- Mobile app for field crews and homeowner view
- Online payment processing

**Differentiating features**
- Industry-leading construction scheduling with Gantt-style views
- Client portal bridges contractor workflow and homeowner visibility
- Integrated payment processing and lien waiver management

**UX patterns**
- Professional-first; homeowner portal is read-mostly (approvals and payments)
- Steep onboarding; designed for professional construction firms
- Gantt scheduling dominant UX metaphor

**Integration points**
- QuickBooks integration
- Xero integration
- Zapier support
- Open API (REST) for enterprise integrations

**Known gaps**
- Not a consumer product; pricing excludes DIY homeowners (~$199+/mo)
- Homeowner has no independent planning capability; must be invited by contractor
- No inspiration/design features
- No consumer-friendly material shopping

**Licence / IP notes**
- Proprietary SaaS; commercial pricing

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Task lists with progress tracking (checkboxes, status)
- Budget entry and expense logging
- Photo attachment to projects or tasks
- Basic project organisation by home area or room
- Collaboration/sharing with at least one other person (spouse, contractor)
- Mobile access (iOS and/or Android)

### Differentiating Features
- AI-generated task lists and cost estimates from project descriptions
- Local cost benchmarks calibrated to zip code or region
- Contractor quote comparison and proposal management within the app
- Invoice scanning (OCR) to reduce manual data entry
- AR room visualisation (see products/layouts at real scale)
- LiDAR-based accurate room measurement
- Full audit trail for expense changes
- Done Archive linked to resale and tax documentation

### Underserved Areas / Opportunities
- Integrated bill-of-materials builder with quantity calculations tied to room dimensions
- Supplier price comparison across multiple retailers (Home Depot, Lowe's, etc.) in real time
- Permit and compliance checklist with jurisdiction-aware reminders
- Offline-first design for reliable on-site use without connectivity
- Multi-phase scheduling with task dependencies for complex multi-trade renovations
- Contractor evaluation combining quote tracking, ratings, notes, and communication history in one place
- AI-guided scope breakdown: input "I want to renovate my kitchen" and receive a structured phase plan with task list, material list, and budget range
- Export of full project documentation (material lists, cost summaries) as PDF/CSV for contractors or mortgage lenders

### AI-Augmentation Candidates
- Auto-generating phased task lists from a plain-language project description
- Estimating material quantities from room dimensions or floor-plan data
- Surfacing relevant permit requirements based on project type and location
- Suggesting contractor questions or red-flag criteria based on trade type
- Identifying budget overruns early with anomaly detection across expense categories
- Recommending alternative materials at lower cost points while meeting spec requirements

---

## Legal & IP Summary

All surveyed tools are proprietary SaaS or mobile applications. No open-source licensing was identified among the primary competitors. None of the feature patterns reviewed (task lists, budget trackers, photo documentation, AI planning assistants) appear to be subject to specific patents in the public domain based on available information, though no formal patent search was conducted. Building on standard UX patterns (checklists, Kanban boards, budget dashboards) carries no known IP risk. Any AI-generated material list or cost-estimation algorithm developed independently would not infringe on existing tools. Developers should avoid directly cloning UI/UX trade dress or proprietary data formats. No regulatory framework (beyond general data-privacy requirements such as GDPR/CCPA) was identified as specific to this domain.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Project creation with phases and task lists (checkboxes, status, due dates)
- Bill-of-materials builder: items with quantity, unit, unit cost, and supplier fields
- Budget vs. actual expense tracker with category breakdown (labour, materials, permits, tools)
- Photo documentation attached to tasks or phases (before/during/after)
- Contractor quote entry and side-by-side comparison per trade
- Offline-capable local-first data storage with background sync

**Should-have (v1.1)**
- AI-assisted project scoping: generate task list and material list from a text description
- Permit/compliance checklist with reminders keyed to project type
- Supplier price comparison: multiple price entries per material item with auto-lowest-cost highlight
- PDF/CSV export of material lists and cost summaries
- Collaborative sharing with role-based access (owner, view-only, contractor)
- Push notifications for task deadlines, quote expiry, and contractor follow-up reminders

**Nice-to-have (backlog)**
- Room dimension input to auto-calculate material quantities (e.g., paint coverage, tile count)
- AR product placement to visualise finish choices in situ
- Integration with Home Depot / Lowe's APIs for live price lookups
- Done Archive with resale and tax documentation packaging
- ROI estimation per improvement based on home value benchmarks
- Voice-input task and expense logging for hands-free on-site use
