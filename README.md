# SGA — Asset and Maintenance Management

SGA is an internal application for managing physical assets, functional systems, maintenance work, building inspections and technical knowledge.

Built with **Payload CMS, Next.js and PostgreSQL**, it combines a practical asset register with maintenance records, technical diagrams, building plans, inspection reports and traceable decisions.

The project is under active development. Its catalogue and operational workflows are being tested with real equipment and service records before wider use.

**Documentation updated:** 2026/11/19.

> **Current highlights:** dashboard maps for assets requiring attention, improved asset search and filters, building inspections with interactive plans and PDF previews, a searchable Markdown knowledge base, and a scheduled maintenance generation command.

## Contents

- [Purpose and design principles](#purpose-and-design-principles)
- [Domain model](#domain-model)
- [Collections](#collections)
- [Asset identification and documentation](#asset-identification-and-documentation)
- [Functional systems and technical diagrams](#functional-systems-and-technical-diagrams)
- [Building plans and asset locations](#building-plans-and-asset-locations)
- [Tickets and interventions](#tickets-and-interventions)
- [Service agreements and planned occurrences](#service-agreements-and-planned-occurrences)
- [Asset status history](#asset-status-history)
- [Building inspections](#building-inspections)
- [Knowledge base](#knowledge-base)
- [Roles and access control](#roles-and-access-control)
- [Frontend](#frontend)
- [Administration forms](#administration-forms)
- [Project organisation](#project-organisation)
- [Development](#development)
- [Database changes and backups](#database-changes-and-backups)
- [Git workflow](#git-workflow)
- [Validation checklist](#validation-checklist)
- [Known limitations and next steps](#known-limitations-and-next-steps)
- [Development conventions](#development-conventions)

## Purpose and design principles

SGA brings together information that previously lived in Drupal, emails, reports and individual knowledge. Drupal is an extraction source for existing data; it is not part of the application's future operation.

The main principles are:

- **Maintenance comes first.** Models identify equipment and collect reference documents. Detailed technical specifications stay in manuals unless a structured field serves a concrete purpose.
- **Planning and execution are separate.** A maintenance occurrence belongs to a reference period. Work can happen before or after that period without moving the original commitment.
- **An intervention records facts.** It describes work performed, its date, provider, documents and results.
- **A ticket organises a problem or request.** Routine maintenance can be recorded without creating a ticket.
- **Current asset status comes from history.** It is a calculated projection of confirmed events, not an independently editable value.
- **Simple work should remain simple.** Coordination can complete and validate straightforward cases in one action.
- **Relationships have an owning record.** Joins present related records without storing a second editable copy of the relationship.

Code, field names and machine identifiers are in **English**. User-facing labels are in **European Portuguese**.

## Domain model

| Concept | Responsibility |
| --- | --- |
| Entity | A manufacturer, supplier, service provider or other organisation. One entity can have several roles. |
| Asset type | A reusable classification with visual presentation settings. |
| Asset model | Product identification, manufacturer, reference image, documents and notes. |
| Asset | An individual physical item with its own code, location, history and photographs. |
| Functional system | A meaningful operational unit, such as a fire detection or refrigeration system. |
| System component | An asset's or group of assets' participation in a functional system, including functional connections. |
| Ticket | A fault, request or corrective process that may require several interventions. |
| Intervention | A dated record of work performed. |
| Service agreement | The provider, scope, procedure and contractual terms of a service. |
| Maintenance plan | A recurring schedule and equipment scope within a service agreement. |
| Planned occurrence | One expected execution associated with a year, quarter or month. |
| Asset status event | A confirmed, dated change in the condition of an asset. |
| Building inspection | A dated visit documenting the condition of a building or space. |
| Inspection point | A specific observation, with optional plan coordinates, photographs and asset references. |
| Knowledge article | An internal note or guide with a category, tags and controlled visibility. |

System components are independent records. Their relationship to the functional system is stored on the component and displayed through a join in the system form.

Interventions are also independent records, even when created through a ticket or planned occurrence. Their source relationship allows them to appear inside that source's form while remaining searchable on their own.

## Collections

| Collection slug | Purpose |
| --- | --- |
| `users` | Authentication, user accounts and roles. |
| `manufacturers` | Unified entities and their roles. The historical slug is retained even though the collection is labelled **Entidades**. |
| `asset-types` | Asset classifications, icons, colours and map marker shapes. |
| `asset-models` | Shared product identity and reference documentation. |
| `assets` | Individual physical assets. |
| `functional-systems` | Functional systems. |
| `system-components` | System membership and functional connections. |
| `connection-types` | Reusable connection classifications. |
| `building-plans` | Composite building plans uploaded as SVG or PNG. |
| `departments` | Organisational allocation. |
| `cost-centers` | Cost allocation. |
| `tickets` | Faults and work requests. |
| `interventions` | Work performed and its results. |
| `service-agreements` | Maintenance and service agreements. |
| `maintenance-plans` | Scope and recurrence rules within service agreements. |
| `planned-occurrences` | Expected maintenance executions. |
| `inspections` | Building inspections and issuance records. |
| `inspection-points` | Observations linked to an inspection, with location, photographs and related records. |
| `knowledge-articles` | Searchable internal Markdown articles. |
| `asset-status-events` | Authoritative asset status history. |
| `asset-status-batches` | Controlled bulk status registration. |
| `asset-compliance-records` | Compliance records; their operational workflow still needs further consolidation. |
| `images` | Shared uploaded images and generated image variants. |
| `documents` | Documents, document classification and document date. |
| `document-types` | Reusable document classifications. |
| `media` | Additional media collection retained from the application setup. Confirm its references before consolidating it with the dedicated upload collections. |

The exact `slug` in each collection file remains authoritative.

## Asset identification and documentation

### Identity

Each asset receives an automatically generated **eight-character code**, using uppercase letters and digits while excluding ambiguous characters. The database enforces uniqueness.

The code is the stable identifier. It does not change when the asset moves or receives a different internal designation. Duplication must generate a new code.

A serial number is optional. Small interchangeable devices, such as smoke detectors and alarm buttons, are still recorded as individual assets. A generic model can be used when the exact product cannot be identified.

The current administrative display label is:

```text
Model · Asset code · Serial number (when available)
```

The asset detail page uses the code as its main heading, with the model and internal designation presented separately. Including the serial number in selectors helps match equipment against service reports.

### Reference and instance information

- The **model's featured image** is the reference image on the asset page.
- The asset's photographs appear in its own gallery: identification labels, serial plates, installation details and other particulars of the physical item. Gallery photographs open in a modal with fit-to-window and original-size views.
- Model documents are shared through the relationship rather than copied into every asset.
- Documents support a **document type** and **document date**. Upload metadata also provides filename, MIME type and file size.
- Model notes and asset observations use rich text.
- Asset dates include manufacture, installation and warranty expiry.

User-facing dates should use **`yyyy/MM/dd`**. Operational forms normally request a day rather than a time. Audit timestamps retain the registration time.

Images support focal-point editing and configured image variants. New variants require processing; changing image size configuration does not automatically regenerate every existing upload.

## Functional systems and technical diagrams

A system component records:

- its functional system;
- a designation or zone;
- one or more individually registered assets;
- an optional preceding asset;
- an array of connections, each with source port, connection type and destination port.

Several connections can describe the same functional relationship. For example, an HVAC pair can have an electrical connection and two refrigerant pipes without merging them into one text value.

The Mermaid diagram is generated from these records. It presents component designations and asset identification, with connection labels such as:

```text
Zone 12 ► Fire-resistant cable
```

The viewer supports zoom controls, reset, panning and fullscreen presentation. At 100%, the diagram uses the available width; tall diagrams can be scrolled. Asset status supplies an additional visual reading of the system's condition.

### Electrical supply

Electrical supply belongs to the **asset**, through its `powerFeeds` array. Each feed can identify the source asset, AC/DC current type, ports, connection type and verification level.

This allows a system diagram to include relevant upstream electrical assets, including boards outside the functional system. Functional membership does not need to be duplicated merely to display an electrical dependency.

Verification distinguishes a presumed connection from one identified in documents or confirmed on site. Technical diagrams represent the information recorded in SGA; incomplete paths remain incomplete until investigated.

The diagram definition is prepared from application data and Mermaid renders its SVG in the client component. Persisted diagram snapshots and render caching are not yet implemented.

## Building plans and asset locations

Each building uses a **single composite plan**. The main ground-floor drawing and inset drawings for upper floors or mezzanines can coexist in that image.

This avoids duplicating most of a complex across several nearly identical floor plans.

- Plans accept SVG or PNG.
- The asset form provides a visual location picker.
- Positions are stored as normalised `x` and `y` coordinates between `0` and `1`.
- The location also records the building plan, level and area or room.
- The system page combines its assets on the appropriate plans.
- On system maps, **shape identifies asset type** and **colour identifies asset status**.
- Individual asset pages and dashboard maps can use the type's icon.
- Set **Presentation order** (`sortOrder`) on a building plan to control its position in system and dashboard maps. Lower numbers come first; ties are sorted by name. Missing values are treated as 100.

Static asset icons belong in **`public/asset-icons/` at the project root**, not inside `src/` or the uploads directory.

When replacing a plan, retain its proportions and the positions of the building features if existing asset coordinates should remain valid. Replace the uploaded file through the existing plan record to preserve its relationships.

## Tickets and interventions

### Tickets

Tickets identify a problem or request and collect its interventions, decisions and status proposals.

Codes follow:

```text
T-YYMMDD-XXXX
```

The date comes from the entered occurrence/communication date rather than simply the system's creation timestamp. Existing codes remain stable after creation.

The resolution workflow supports:

| Action | Result |
| --- | --- |
| Start or resume | Work in progress. |
| Put on hold | Work waiting, with a reason. |
| Propose resolution | Resolution awaiting coordination's validation. |
| Validate | Ticket closed. |
| Return | Work resumes, with a reason. |
| Resolve and close directly | Coordination completes a simple case in one action. |
| Reopen | Work resumes while preserving earlier decisions. |

A resolution proposal requires a supporting intervention that belongs to the ticket. Closing also checks for outstanding asset status proposals.

Actions are recorded in a workflow history containing the actor, effective date, registration timestamp, transition, supporting intervention and decision note. A closed ticket must be reopened before changing its interventions.

### Interventions

Intervention codes follow:

```text
I-YYMMDD-XXXX
```

The date comes from `performedAt`. The intervention records its summary, actual date, provider, scope, work details and supporting documents.

An intervention can originate from a **ticket** or a **planned occurrence**, but not both at once. The current model also permits a direct record without either origin.

Scope follows **System → Component → Assets**:

- a system alone covers the system;
- a component narrows that scope;
- explicitly selected assets narrow it further;
- assets can also be selected directly without a system.

When creating an intervention without its own scope, the source supplies the initial system, component and assets. This is a one-time copy, not a permanently synchronised mirror. Hook-based inheritance becomes visible after saving; the existing ticket-scope button can populate the form explicitly before saving.

## Service agreements and planned occurrences

### Agreements

A service agreement records its provider, work description, scope, maintenance instructions, dates and contractual documents. Each agreement can have several maintenance plans, so different groups of assets can follow different schedules under one contract. Validity is derived from the agreement dates; a future end date does not mark the agreement as ended today.

The identifying label follows:

```text
AS-XXXX-Provider
```

The editable short description remains in `title`; the composed identifier is stored in `displayLabel`.

### Recurrence and occurrences

A plan defines a first year and month, an interval in months and the equipment covered by the rule. Its preview lists the expected periods. Coordination can generate occurrences from the plan form; the same generator is also available from the command line.

Generation considers **24 calendar months**, from the current month through the following 23, in the Europe/Lisbon timezone. It respects the known start and end months of the agreement. The recurrence stays anchored to the plan's calendar even when an intervention happens early or late.

Generated monthly occurrences are identified by plan and period. Running generation again skips existing keys. If an agreement already has a manually created monthly occurrence without a plan for a period the generator needs to fill, generation reports a conflict. Associate the existing occurrence with its intended plan before retrying.

Occurrences can still be created manually, including without an agreement or plan. They may inherit scope, instructions and a work description from their source. Annual and quarterly reference periods remain available for manually recorded work. An optional sequence distinguishes multiple maintenance reviews in the same reference period; it does not count technicians' individual visits.

An agreement-based monthly occurrence can have a label such as:

```text
AS-XXXX-Provider-2025/05
```

### Scheduled generation

The command uses `SGA_MAINTENANCE_JOB_EMAIL` to look up a dedicated SGA account with coordinator permissions. Set it in the server environment used by the job, alongside the application's database configuration. The command scans plans with recurrence enabled. It logs a run ID and per-plan results, continues after an individual plan failure, and returns a non-zero exit status if any plan fails.

```sh
SGA_MAINTENANCE_DRY_RUN=1 pnpm run maintenance:generate
pnpm run maintenance:generate
```

The first command checks the horizon and conflicts without creating occurrences; the second creates missing records. Run them from the project directory with the intended database configured.

The repository includes [systemd service and timer units](deploy/systemd/). Once installed and enabled, the timer runs daily at **03:20 Europe/Lisbon** and can catch up after downtime. Review the service user, working directory and Node.js path before installing on another host. The units in Git do not confirm that the timer is active on a particular server.

```sh
systemctl list-timers sga-maintenance-generation.timer
journalctl -u sga-maintenance-generation.service -n 50
```

Deactivating a plan or recording an agreement's end affects future generation. Occurrences that were already created remain in the record and should be reviewed separately.

### Completion and validation

| State | Meaning |
| --- | --- |
| `planned` | Work remains to be completed. |
| `pending-validation` | Completion has been proposed. |
| `completed` | Completion has been validated. |
| `waived` | The occurrence was explicitly dispensed with. |
| `cancelled` | The occurrence was cancelled. |

Technicians can propose completion using a supporting intervention. Coordination can validate, return, dispense with, cancel or reopen an occurrence, and can complete and validate a simple case directly.

- The original reference period is retained.
- The effective completion date is separate from the validation date.
- Validation checks all associated interventions for pending asset status proposals.
- Confirmed asset status changes remain separate from maintenance completion.
- Interventions remain editable while validation is pending, allowing outstanding proposals to be processed.
- Completed, waived or cancelled occurrences must be reopened before changing their interventions.
- **Add note** appends information without rewriting a decision or changing the occurrence's state.

### Example: two visits for one maintenance occurrence

A scheduled refrigeration review identifies a leak. The technicians investigate on the first visit and return with additional equipment to locate and repair it.

Both interventions can belong to the same planned occurrence. They may propose different asset states at their respective effective dates. The final intervention can support completion of the maintenance commitment.

Finding a fault can be the result of a successfully performed inspection. Maintenance completion therefore does **not** require every asset to be operational. A corrective issue needing its own management can follow a separate ticket; a dedicated origin link for that follow-up remains a future enhancement.

## Asset status history

Supported states are:

| Value | Meaning |
| --- | --- |
| `operational` | Operational. |
| `restricted` | Operational with restrictions. |
| `out-of-service` | Out of service. |
| `under-repair` | Under repair. |
| `retired` | Retired. |

Tickets and interventions contain **asset status proposals**. A proposal can select multiple assets sharing the proposed status, effective date and justification. Different outcomes or reasons should use separate proposal rows.

Confirmation creates an individual `asset-status-events` record for each selected asset. A `sourceKey` prevents the same confirmed proposal from creating the same event again.

The current proposal-generated key format is:

```text
<source-collection>:<source-id>:<proposal-id>:<asset-id>
```

`assets.currentStatus` is calculated from the latest event by effective date. Registration timestamp and record ID resolve ties. Registering an older event therefore does not automatically replace a more recent effective state.

Initial states can be registered in batches. Dates can be known dates or explicitly convention-based reference dates. The asset timeline visually distinguishes a reference date using hatching.

Historical events and confirmed proposals are currently protected against ordinary editing and deletion, including by administrators. A dedicated administrative correction mechanism has been discussed but is **not implemented**. Direct SQL changes bypass application hooks and require checking source relationships, idempotency keys and the current-state projection.

## Building inspections

Inspections document the condition of buildings and spaces independently of the asset register. An inspection records the date, observer, building, context and overall summary. Its observation points are separate records shown through a join in the inspection form.

Each point can record a category, description, level or area, position on the plan and captioned photographs. Asset references are optional. An inspection can be motivated by a ticket, and corrective tickets can reference the inspection or specific points. The observations remain as evidence of what was seen, even after the work is closed.

The frontend pairs a zoomable, marked plan with the observation list. Selecting a marker highlights its observation, and photographs can be opened in a viewer. A PDF preview includes the annotated plan and illustrated observations when available.

Coordination or administration can issue an inspection once it has at least one saved observation. Issuance records the date and user and locks edits to the inspection and its observations. **The PDF endpoint generates a preview on demand**; issuance does not store a fixed PDF or a snapshot of the underlying plan and image files.

## Knowledge base

Articles store Markdown with a title, summary, category and comma-separated tags. The frontend renders GitHub Flavored Markdown and combines text search with tag filtering. Article visibility applies in server queries, including to the articles used to build the tag list.

| Visibility | Readers |
| --- | --- |
| Administrator | Administrators only. |
| Restricted | Administrators and coordinators. |
| Internal | Administrators and coordinators can read drafts; other SGA users see published articles only. |

Published means available inside SGA. The article collection retains up to 25 versions per document. Version history is restricted to administrators because older revisions may contain restricted content.

The knowledge base is for technical notes and guides. Controlled quality documents with formal approvals and effective revisions are a separate design topic.

## Roles and access control

Roles are stored on users and can be combined:

| Role | Main responsibility |
| --- | --- |
| `admin` | User administration, configuration and authorised management operations. |
| `coordinator` | Catalogue/planning management, work coordination and validation. |
| `technician` | Consultation, operational records and proposals. |
| `viewer` | Read access through the frontend; no administration-panel access. |

`src/lib/roleAccess.ts` applies policies where collections are registered in `payload.config.ts`.

| Policy | Main behaviour |
| --- | --- |
| `users` | User management is visible to administrators. Existing self-account restrictions are preserved. |
| `catalog` | Assets and systems are visible to technicians for consultation; management belongs to coordination and administration. |
| `reference` | Reference collections are maintained by coordination/administration and hidden from the technician's menu. Read access remains available to relationships. |
| `attachments` | Operational users can upload files; existing-file updates belong to coordination/administration. |
| `work` | Technicians can create and work on operational records, subject to workflow safeguards. |
| `planning` | Coordination creates occurrences. Technicians can submit actions while planning and scope fields are protected. |
| `history` | Confirmed history is readable, with creation restricted and ordinary updates/deletions blocked. |
| `batches` | Bulk state registration belongs to coordination/administration. |
| `agreements` | Agreements are maintained by coordination/administration and hidden from the technician's menu. Read access remains available. |

Policies restrict existing access functions rather than replacing their protections. They also preserve hooks. For example, an administrator's collection-level delete permission does not bypass a hook protecting a confirmed record.

Menu visibility is not a confidentiality boundary: an agreement hidden from a technician's menu remains readable under the current agreement policy. Use server-side read restrictions if the data itself must become inaccessible, accounting for dependent forms and frontend pages.

The server controls validation permissions. Role-specific filtering of every option in action selectors is not yet complete; visibility of an option does not grant permission to execute it.

Trusted internal status projection updates use the existing event hook and transaction context. Ordinary asset edits cannot independently set `currentStatus`.

User profiles include professional identity and contact details. Employment status is recorded separately from roles and does not itself revoke access. An internal responsible person can be assigned to work and planning records; when unset, the frontend can resolve responsibility from the source record.

## Frontend

| Route | Content |
| --- | --- |
| `/` | Maintenance dashboard: work queues, pending validation, recent interventions and asset attention maps. |
| `/assets` | Search by code, model, serial number or internal designation; combine asset type, model and building filters with pagination. |
| `/assets/[code]` | Identity, model reference image, location, photo gallery, documents, work records and status timeline. |
| `/asset-models/[id]` | Model reference information and documentation. |
| `/systems`, `/systems/[id]` | System directory, functional diagrams, asset plans and the latest ten tickets and interventions. |
| `/systems/[id]/register` | Paginated system work records. |
| `/tickets/[code]` | Ticket details, interventions and resolution history. |
| `/interventions`, `/interventions/[code]` | Searchable intervention directory and individual work records. |
| `/service-agreements`, `/service-agreements/[id]` | Agreement directory, provider and equipment details, plans and a calendar comparing planned periods with actual work. |
| `/planned-occurrences/[id]` | Reference period, interventions and completion history. |
| `/entities`, `/entities/[id]` | Entity directory, logos, agreements, documents and associated interventions. |
| `/inspections`, `/inspections/[code]` | Inspection directory, interactive plan and observation photographs. |
| `/inspections/[code]/pdf` | On-demand inspection PDF preview. |
| `/knowledge`, `/knowledge/[id]` | Searchable articles, tag filters and rendered Markdown. |

The application header shows the signed-in user and offers sign-out. Links between records stay in the frontend; links to edit or manage a record are shown where the user's role permits them.

### Assets requiring attention

The dashboard maps assets with confirmed states of **Restricted**, **Out of service** or **Under repair**. Icons identify asset types; marker colours identify states. Each building has a companion table, and assets without usable coordinates or an available plan stay visible in a separate list. Assets with no recorded state are counted separately. The maps represent asset condition, which can differ from the status or existence of a ticket.

## Administration forms

Operational forms use visible, labelled sections with related fields arranged in rows. A form's colour fades from its identification section towards its consultation fields. Embedded records and proposals can use their own accent so the reader can distinguish them from the surrounding form. These presentation groups preserve the existing stored field paths and server-side permissions.

## Project organisation

| Location | Responsibility |
| --- | --- |
| `src/payload.config.ts` | Payload setup, collection registration and applied role policies. |
| `src/collections/` | Collection schemas, relationships and collection-specific hooks. |
| `src/components/` | Admin controls and frontend components. |
| `src/lib/permissions.ts` | Shared role checks. |
| `src/lib/roleAccess.ts` | Collection policies and protected occurrence planning fields. |
| `src/lib/systemScope.ts` | Cascading filters and source-scope inheritance. |
| `src/lib/generateDatedCode.ts` | Ticket/intervention identifiers based on effective dates. |
| `src/lib/workStatusProposals.ts` | Multi-asset proposals, confirmation and status event creation. |
| `src/lib/ticketWorkflow.ts` | Ticket transitions, audit history and intervention safeguards. |
| `src/lib/occurrenceWorkflow.ts` | Occurrence completion, validation, notes and safeguards. |
| `src/lib/maintenanceRecurrence.ts` | Calendar recurrence calculations. |
| `src/lib/maintenancePlanning.ts` | Shared generator, plan scope and occurrence identification. |
| `scripts/generateMaintenanceOccurrences.ts` | Maintenance generation command and run logging. |
| `deploy/systemd/` | Service and timer units for scheduled generation. |
| `src/lib/inspectionIssuance.ts` | Inspection issuance and observation safeguards. |
| `src/lib/inspectionReport.tsx` | Inspection PDF presentation. |
| `src/lib/knowledge.ts` | Article visibility, searchable text and tag helpers. |
| `src/lib/buildSystemDiagram.ts` | Mermaid definition generation. |
| `src/app/(frontend)/` | Application pages and frontend styles. |
| `src/app/(payload)/` | Payload admin/API integration. |
| `src/payload-types.ts` | Generated collection types. |
| `public/asset-icons/` | Static SVG icons. |
| Upload directories | User-uploaded images, documents, plans and generated variants. |

Upload paths are defined by each collection's `staticDir`. Include every configured upload location in backup and restore procedures.

## Development

### Recorded environment

The SGA development installation is located at `/var/www/projects/sga` on the shared Ubuntu server and uses PostgreSQL database `sga`.

The application uses port **3001**. Other Payload projects have separate repositories/directories, databases and ports.

The last shared dependency baseline was:

| Dependency | Recorded version |
| --- | --- |
| Payload and associated packages | `3.90.2` |
| Next.js | `16.3.3` |
| React / React DOM | `19.2.6` |
| TypeScript | `5.7.3` |
| Sharp | `0.35.4` |
| Database adapter | `@payloadcms/db-postgres` |

Node.js 24 and pnpm 10 have been used on the development server. **`package.json`, the lockfile and the actual installed runtime take precedence over this recorded baseline.** This README does not imply an upgrade has been performed.

### Start locally

```sh
cd /var/www/projects/sga
pnpm install
pnpm dev
```

Open `http://localhost:3001` or `/admin` through the configured local/VPN/port-forwarding connection. `localhost` refers to the machine or forwarded endpoint being used.

Set `DATABASE_URL` to a **PostgreSQL** connection string and configure a strong `PAYLOAD_SECRET` before starting the application. The current `.env.example` has a leftover MongoDB example; use the PostgreSQL adapter configuration in `src/payload.config.ts` as the source of truth. Keep credentials and secrets outside Git.

### Generate and verify

After schema changes:

```sh
pnpm exec payload generate:types
```

After adding or changing registered admin components:

```sh
pnpm exec payload generate:importmap
```

Check the project:

```sh
pnpm exec tsc --noEmit
pnpm lint
pnpm build
```

The repository also defines integration and end-to-end test commands:

```sh
pnpm run test:int
pnpm run test:e2e
```

The integration suite includes recurrence calendar and horizon checks, plus knowledge visibility, search-text and tag helper tests. These checks do not exercise the server timer or every workflow and access boundary.

### Deployment and upgrades

`pnpm dev` is the development server. Production deployment and schema migrations need their own reviewed procedure.

Keep Payload packages on matching versions and commit the lockfile. Test upgrades against a separate database and upload copy. A Git branch alone does not isolate the data used by the application.

No Payload 4 upgrade is assumed by this documentation.

## Database changes and backups

PostgreSQL development schema push has been used during implementation. Renaming a field or changing its type may be interpreted as dropping data or creating a different column.

- Inspect destructive schema warnings against the intended change.
- Regenerated TypeScript types do not themselves migrate database contents.
- Prepare the migration baseline before adopting a controlled production migration workflow.
- Keep the established catalogue intact when resetting small experimental workflow records.
- Review references before deleting records involved in proposals, validation or historical events.

The existing backup approach stores timestamped database and file archives under `$HOME/backups/sga/`. Twice-daily automation was set up during development; verify the current cron configuration and recent output files on the server.

A recoverable backup needs:

1. A restorable PostgreSQL dump.
2. Every upload directory, including originals and generated files where required.
3. The matching application source revision and lockfile.
4. Environment/secrets configuration stored securely outside the repository.

Git protects source history; it does not replace database or upload backups. Cloud copying with tools such as rclone has been explored, but off-site success must be checked rather than assumed. A restore test is the meaningful check that the backup set is usable.

## Git workflow

Use a second terminal for Git while the development server runs in the first.

Before a substantial change:

```sh
git status
git switch -c feat/short-description
```

After implementation and verification:

```sh
git diff
git add path/to/changed-file README.md
git diff --cached --stat
git diff --cached
git commit -m "Describe the completed behaviour"
git push -u origin HEAD
```

Stage other intentional project changes explicitly. Review the staged diff before committing. Do not commit `.env`, database dumps, backups, uploads, build output or dependencies. Static icons in `public/asset-icons/` are source assets and should be versioned.

Use English commit messages describing a coherent change rather than one commit per edited file. Suggested documentation commit:

```text
Update SGA documentation for inspections and maintenance automation
```

## Validation checklist

Use these checks when changing workflow or access code:

- [ ] The TypeScript check passes after regenerating types/import maps where required.
- [ ] An administrator can manage users; technicians cannot assign or change roles.
- [ ] A technician can consult an asset but cannot edit its catalogue data.
- [ ] System/component selection correctly filters the asset selector.
- [ ] An intervention inherits source scope once, without changing earlier historical records when the source later changes.
- [ ] Confirming a multi-asset proposal creates one event per asset; saving again does not create duplicates.
- [ ] A backdated event preserves the correct current state and timeline ordering.
- [ ] Confirming a status proposal still updates the asset after catalogue access restrictions are applied.
- [ ] Ticket resolution and occurrence completion require a valid supporting intervention.
- [ ] Pending asset status proposals block final validation.
- [ ] Return and reopen actions retain the earlier decision history.
- [ ] An occurrence note preserves its state, completion data and existing decisions.
- [ ] Closed/completed work protects its interventions until reopened.
- [ ] Technician menus and custom controls follow the same permissions as the server.
- [ ] The frontend still displays readable relationships, attachments, maps and diagrams for each intended role.
- [ ] Maintenance generation respects agreement dates; a dry run writes nothing and subsequent runs do not duplicate occurrences.
- [ ] The dashboard respects building order, keeps unplotted assets in the list and reports unknown states separately.
- [ ] An inspection point appears on the plan and in the PDF preview with the correct photographs; issuance prevents further edits.
- [ ] Knowledge article details, search results and tag counts respect each role’s visibility.

## Known limitations and next steps

- Verify the maintenance timer, logs and backup restore procedure on the deployment host.
- Add notifications for operational work and failures. The current Payload configuration does not define an email adapter.
- Add an explicit originating-intervention relationship for corrective tickets where useful.
- Provide audited administrative corrections that reconcile confirmed proposals, historical events and current-state projections.
- Consolidate the asset compliance workflow.
- Consider historical system snapshots and diagram caching after measuring real workloads.
- Persist issued inspection reports and the exact plan and photographs used if a fixed report archive becomes necessary.
- Keep formal quality-document approval and controlled revision management separate from ordinary knowledge articles.

## Development conventions

- Prefer small, understandable changes with an explicit purpose.
- Keep machine names and code in English; use European Portuguese for application labels.
- Use `yyyy/MM/dd` for displayed operational dates and preserve effective dates separately from audit timestamps.
- Reuse entities, classifications, models and documents instead of creating near-duplicates.
- Keep each physical asset individually traceable, including assets without serial numbers.
- Preserve one owning relationship and use joins for reverse views.
- Keep derived fields under server control.
- Share the request context in related writes when they must participate in the same transaction.
- Do not use hidden menu entries as a substitute for API access control.
- For small disposable test datasets, prefer a deliberate reset over permanent compatibility code. Review each case against its dependencies and preserve established data.
- Keep this README aligned with the code and distinguish implemented features from planned work.
