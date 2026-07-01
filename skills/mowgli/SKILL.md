---
name: mowgli
description: Push and pull product designs between a real codebase and a Mowgli project using the mowgli CLI. Use when the user asks to sync, push, or pull screens/specs to or from Mowgli (https://mowgli.ai), or mentions "mowgli", "mowgli-cli", or a Mowgli project.
---

# Mowgli Sync

You are working with **Mowgli**, an AI-powered product design and scoping tool. This
skill lets you push designs from a real codebase into a Mowgli project (and pull them
back) through the `mowgli` CLI, which allows programmatic access to Mowgli.

Mowgli is a Figma-like canvas aimed at fast design iteration. Users can make
sweeping or surgical changes, export into a variety of formats, create
prototypes, and use the version history. A user might be using Mowgli to
experiment visually, redesign, compare, and ideate on an app, website or other
project. As the user's coding agent, you facilitate the interoperation between a
real codebase and the Mowgli project.

Read this whole document before acting. The single most important idea: a Mowgli
project is **spec-backed**. Screens, the SPEC.md, and the frontend.xml must always stay
in sync. The `edit` command does all three automatically, following best
practices.

---

## 1. What a Mowgli project is

A Mowgli project models a product's frontend as a small, self-contained set of files:

- **`SPEC.md`** - the markdown spec: elevator pitch, user journeys, and data model.
- **`frontend.xml`** - a `<frontend_common>` block (shared chrome notes) followed by a
  `<screens>` list. Each `<screen>` entry looks like:

  ```xml
  <screen name="Account Settings" component="SettingsScreen">
    <summary>One-line description of the screen.</summary>
    <contents>
      The overall content and design elements, deliberately staying away from tiny
      details (exact layout, colors, spacing). Markdown bullets are fine here.
    </contents>
    <state id="default" name="Active Subscriber">1-2 sentence description.</state>
    <state id="freePlan" name="Free Plan">...</state>
  </screen>
  ```

  - `component` matches the `.tsx` filename (`{component}.tsx`) and is the component name;
    `name` is the human-readable title.
  - `<summary>` is a one-liner; `<contents>` is the fuller, still-high-level description.
  - each `<state>` has an `id` (passed to the component's `state` prop), a `name`, and a
    1-2 sentence description as its body text.
- **`<id>.tsx`** - one React component per screen. Constraints on this code:
  - icons come **only** from `lucide-react`
  - styling is **pure Tailwind with exact classes** like `bg-[#abcdef]`, `text-[#0F172A]`
  - all Google Fonts are available via arbitrary font-family syntax, e.g. `font-['Open_Sans']`
  - the component takes a single `state` prop and renders the matching state

A canonical screen file looks like this:

```tsx
import * as React from 'react';
import * as Icons from 'lucide-react';

export type OnboardingScreenProps = { state: string };

/**
  ...docs about OnboardingScreen...
  States:
  - `step1`: Step 1, consisting of a name for the user's first event, and a skip
    button to avoid creating a first event.
  ...
*/
const OnboardingScreen = (props: OnboardingScreenProps) => (
  <div className="min-w-full h-full overflow-y-auto">
    <div className="bg-[#abcdef] text-[#112233] text-lg">Onboarding</div>
    {/* ... */}
  </div>
);

export default OnboardingScreen;
```

`frontend.xml`, especially the screen descriptions and their state lists, is used as a
load-bearing, abbreviated description of the screens in places where the full `.tsx`
would be too token-heavy. Keep it accurate.

Mowgli is **version controlled**: a project has a HEAD (current) version and a linear
history. Every `edit` produces a new version. There is project-level metadata (one
freeform string per project) and version-level metadata (one freeform string per
version) that you use to track the codebase <-> Mowgli relationship over time.

---

## 2. The CLI

Invoke without installing via `npx mowgli-cli ...`. Everything goes to stdout as
JSON or raw strings; the exit code signals success or failure. The server
defaults to `https://app.mowgli.ai`; override with `--base-url <url>` or
`$MOWGLI_BASE_URL`.

| Command | What it does |
| --- | --- |
| `mowgli login` / `mowgli logout` | Manage the stored credential (browser-approved CLI grant) |
| `mowgli projects` | List projects the credential can access |
| `mowgli -p <projectId> info` | Project info, including project-level interop metadata |
| `mowgli -p <projectId> set-meta [text]` | Read, or set (text arg or stdin) the project-level metadata |
| `mowgli -p <projectId> versions [--limit N --offset N]` | Paginated version history |
| `mowgli -p <projectId> version <n>` | Show one version |
| `mowgli -p <projectId> version <n> set-meta [text]` | Read, or set the version-level metadata |
| `mowgli -p <projectId> files [--version <v>]` | List logical files (SPEC.md, frontend.xml, screens) |
| `mowgli -p <projectId> read <path> [--version <v>]` | Print one file's contents |
| `mowgli -p <projectId> diff [--stat] [--from <v> --to <v>] [paths...]` | Unified diffs, or a `--stat` summary |
| `mowgli edit --lint -f <file>...` | Offline preflight: flag local files imported by the `-f` files but not provided (no API call) |
| `mowgli -p <projectId> edit [--instruction <text>] [-f <file>]...` | Apply a natural-language edit via the Mowgli agent |

### Command examples

```bash
# Who can I see? (find the target project's id)
mowgli projects

# First-contact recon on a project
mowgli -p <projectId> info                 # includes the project metadata string
mowgli -p <projectId> versions --limit 20  # history; check userdata on recent versions
mowgli -p <projectId> files                # what screens exist at HEAD
mowgli -p <projectId> read SPEC.md
mowgli -p <projectId> read frontend.xml
mowgli -p <projectId> read DashboardScreen.tsx

# What changed between two versions (cheap overview, then full patch)
mowgli -p <projectId> diff --stat --from 6
mowgli -p <projectId> diff --from 6 --to 9 DashboardScreen.tsx

# Write project metadata (first connect) - via stdin
echo "Codebase: github.com/acme/app (branch main, commit a1b2c3d).
Stack: React 18 + TS, Tailwind v3 semantic tokens, lucide-react. Font: Geist.
Path map: DashboardScreen.tsx <- src/pages/DashboardPage.tsx (+ ShellHeader, Dropdown).
Theme: light only; token->hex bg #FFFFFF, fg #0F172A, primary #E51A4C ..." \
  | mowgli -p <projectId> set-meta

# Apply an edit: instruction on stdin, real source files appended with -f
mowgli -p <projectId> edit \
  -f src/pages/DashboardPage.tsx \
  -f src/components/ShellHeader.tsx \
  -f tailwind.config.ts \
  < /tmp/dashboard-instruction.txt

# Record what you did on the resulting version
mowgli -p <projectId> version 7 set-meta "Synced DashboardScreen from commit a1b2c3d ..."
```

Notes on `edit`:
- The **instruction** comes from `--instruction` or piped **stdin**.
- Each `-f <file>` appends that file's contents to the instruction wrapped in
  `--- BEGIN FILE: <name> ---` / `--- END FILE: <name> ---` markers. Use `-f` to hand
  the agent the real codebase source (the screen, its dependencies, the theme).
- On success the response is the agent's free-text message plus a compact stat summary:
  the `vN -> vM` version bump, one `A/M/D <path>  +added -removed` line per file, the
  totals, and a pointer to `mowgli -p <id> diff --from N --to M` for the full patch. The
  full patch is NOT dumped inline; pull it with `diff` only if the summary looks wrong.
- **If there is no diff, the edit changed nothing** - read the message, fix the
  instruction/context, and rerun the whole thing as a fresh invocation.

---

## 3. Rules of Engagement with Mowgli projects

### Prerequisites

- **Project-level interop metadata.** A coding agent can read and write an arbitrary
  string associated with the project. An agent SHOULD always read and write this
  information when encountering a project that is not already in the context. This
  metadata should be used to identify the repository/codebase connected to the Mowgli
  project, the relationship of the Mowgli project to the real living product (faithful
  copy, smaller slice of a larger product, etc.), define component library and design
  system adherence of the real codebase, and provide information about path mappings and
  directory organization of the real codebase vs the Mowgli project.
- **Version-level interop metadata.** Mowgli is fully version controlled: a project has
  a HEAD (current) version and a linear version history logging every change to the
  project since creation. Version-level interop metadata is a free-form string
  associated with a version, that an agent can read and write. An agent SHOULD always
  read and write this information when performing any type of Mowgli <-> codebase sync.
  The agent SHOULD write information such as:
  - Context on the edits the agent has applied to the project (for example, to sync with
    the real codebase, or correct some discrepancy)
  - Current commit shorthash that corresponds to the version
  - Additional metadata that will be useful to the agent in the future, such as omitted
    features or intentional divergence from the real state

### The Rules of Engagement

Mowgli projects are spec-backed and rely on fidelity of the represented screens and
states.

Any and ALL CHANGES to the screens must be followed by appropriate changes to the
SPEC.md and frontend.xml files that fully reflect the changes in the application.

It's especially important to collect ambient info, even from the backend side of the
application, to understand the real user flows that a screen or feature participates in,
instead of just shipping the design to Mowgli. Mowgli is a SPEC-DRIVEN DESIGN TOOL, and
it relies on the SPEC.md being in sync with the screen designs to provide highly
contextual suggestions, brainstorming, and allow sweeping changes on projects.

Additionally, the screen information in frontend.xml, especially the list of
states, is used as load-bearing foundational documentation for the project,
providing a bird's eye view of the full application --- not just the happy path.
Ensure you discover ALL significant modals, dropdown menus, popovers, and all
relevant states and substates of a screen and faithfully represent them in
Mowgli.

Every change to a screen must be accompanied with corresponding changes to the
appropriate parts of frontend.xml. The `edit` command/endpoint can be used to
apply these changes in one fell swoop, as it's agent-backed.

All first accesses to a project must be accompanied by checking the project metadata,
and writing/updating it as necessary, to ensure there is no drift between what the agent
thinks about the project and what was previously recorded there.

All changes to the Mowgli project should be accompanied by a final version-metadata
write, noting down the changes made in brief form, the commit shorthash or other
versioning pointer linking the Mowgli and codebase versions, and any additional
information necessary to faithfully execute similar operations in the future.

## 4. Sync SOPs

### Mowgli -> Codebase sync

1. List all projects.
2. Identify the project that the user is referring to. If there are multiple and you are
   unsure, ask the user. Take note of the project ID.
3. Check the metadata of the selected project. If it contains any contradictory
   information, resolve this with the user.
4. If not already present, write metadata to the Mowgli project:
   - Current codebase (GitHub name, purpose, etc)
   - Software stack
   - Component libraries and other codebase-level details that are used, if any
   - Current commit (if any) at the time of this first sync
   - Any other top-level notes or observations that could be useful to
     recontextualize the project for a future coding agent run.
5. Pull version information and see if there are any version-level metadata tags.
   (Resolve any ambiguities with the user.)
6. Pull the spec and frontend.xml files from the Mowgli project and understand the
   project. If the tags from the previous step signify there was already work done, use
   the diff endpoints to see what has changed from the last known version and the new
   one. Take note of the changes.
7. As necessary, pull screen by screen, with diffs if needed, and apply changes to the
   codebase. The user has vetted the Mowgli state, so treat it as the source of truth and
   let the diff define the scope: translate exactly what changed and nothing more. Don't read intent
   into version titles or descriptions (they are context for understanding the
   diff, not extra instructions) - unless the user explicitly asks, or a change
   genuinely can't be applied faithfully without it.
8. Write version metadata at the end to signify that the version is up to date with the
   codebase, with date & time and commit info.

### Codebase -> Mowgli sync

1. List all projects and identify the target project, exactly as in steps 1-2 above.
   Take note of the project ID.
2. Check the project metadata. If it is missing or stale, write/update it (codebase
   identity, software stack, component libraries, and path mappings between the real
   codebase and the Mowgli project). Resolve any contradictions with the user before
   proceeding.
3. Pull version information and read the version-level metadata to find the last synced
   version and its recorded commit shorthash. This is the baseline the codebase will be
   diffed against.
4. Determine what changed in the codebase since that baseline (for example,
   `git diff <lastSyncedCommit>`), and map the changed source files to the affected
   Mowgli screens, plus SPEC.md and frontend.xml. If there is no recorded baseline, treat
   every relevant screen as changed.
5. For each affected screen, assemble the context the edit agent needs: the target
   Mowgli files as @-mentions (`@SPEC.md`, `@frontend.xml`, `@<Screen>.tsx`) and the real
   codebase code pasted inline, including the dependencies required to render it
   faithfully (theme files such as `tailwind.config.ts`, shared components such as
   `src/components/ui/button.tsx`, and anything else in the dependency tree). When ADDING
   a new screen to a project that already has screens, also @-mention at least one
   existing screen as do-not-touch reference - the endpoint requires it, and it anchors
   the new screen to the project's shared chrome and style.
6. Run `edit --lint`, batching no more than 4-5 large screens' worth of
   information per invocation (scaled down for complex changes). Every file the
   agent may see or change MUST be @-mentioned in that same invocation;
   invocations are stateless, so each one must be fully self-contained.
7. The lint will flag any files missing in the import hierarchy. Review the lint
   output, adjusting the provided file list until you are satisfied with the
   results. Then, run a final `edit` call to apply the change to Mowgli.
7. Inspect the result. If the response carries no diff, the edit made no change: read the
   agent's message, correct the instruction or context, and rerun the whole request as a
   fresh invocation. Repeat steps 5-6 for the remaining batches.
8. Confirm SPEC.md and frontend.xml now reflect every screen and state that changed. The
   edit agent is expected to keep them in sync, but verify, since they are load-bearing
   for the rest of the system.
9. Write version metadata on the resulting version: a brief note of the changes applied,
   the codebase commit shorthash this version corresponds to, and any intentional
   divergence or omitted features for future reference.

---

## 5. How to author a good `edit` (read this before every push)

The `edit` agent is **agent-backed but exact-minded**: optimized for faithful translation,
not ideation. It already knows the mechanics - the canonical `.tsx` format, resolving
semantic Tailwind tokens to exact hex from theme files you provide, keeping SPEC.md and
frontend.xml in sync, writing a `<screen>` entry for a new screen. Don't spend your
instruction reciting those. Spend it on the two things the agent CANNOT get from inside
Mowgli.

### 1. The real source, via `-f`

The screen and every dependency that draws visible UI: leaf components, theme/token files
(`tailwind.config`, the CSS that defines the tokens), shared widgets. If you omit the
source for a piece of UI, the agent has nothing to anchor on and will invent it; if your
prose guesses or hand-waves, it will faithfully render your guess. This is where almost
every bad push comes from - see the import-tree and fidelity rules below. Provide the
theme files rather than hand-transcribing a hex table (the agent resolves the tokens
itself); only spell a mapping out when it can't be derived from the files you pass. State
which theme (light/dark) to render, since the source may support both and the agent won't
guess. Passing a file with `-f` does NOT mean you have to read it first - the agent reads
what you forward, so forward leaves liberally by name and save your own reading for
behavior (see the lint-first note below).

### 2. Project understanding - the real alpha

This lives in your head and across the wider codebase, not in any single file, so it's the
part only you can supply. Convey it concisely (the agent expands, so don't over-write it -
but don't gloss it either):

- what each screen IS and its role in the product;
- how the screens connect and how a user reaches them - entry points, navigation, the
  flows they sit in (read the routing and the backend, not just the screen file);
- which user journeys and which data-model entities/fields the screen touches;
- the full state list per screen (first state = default) - the states a screen can render
  are something only you can enumerate from the code (see state discipline below).

### Still required, but cheap

- **@-mention every Mowgli-side file the agent may read or change** (`@SPEC.md`,
  `@frontend.xml`, `@<Screen>.tsx`). File visibility is strict: an un-mentioned file the
  agent needs makes it refuse the whole request.
- **When adding a screen to a non-empty project, @-mention at least one EXISTING screen**
  (`@<ExistingScreen>.tsx`) - not just `@SPEC.md` and `@frontend.xml`. The endpoint
  requires it and refuses otherwise ("include at least one existing screen in the
  context"). This is load-bearing, not a formality: the existing screen is how the agent
  picks up the shared chrome (header, nav, footer), the palette, and the layout
  conventions, so the new screen matches instead of subtly drifting. Mark it do-not-touch
  (next bullet) so it is reference only.
- **Name the "do not touch" scope** when adding to an existing project: the existing
  states/screens that must stay byte-for-byte unchanged.

### Lint FIRST to map the tree, then lint again before EVERY edit (do not skip this)

The single most common cause of a bad push is sending a parent component without the
children that actually draw its UI, then describing those children from memory. The agent
faithfully renders your guess and the screen comes out wrong.

So lint EARLY, before you start reading: point it at just the root files (the page/screen
and maybe its top-level view) to cheaply enumerate the whole import tree without opening a
single leaf.

```bash
mowgli edit --lint -f src/pages/DashboardPage.tsx
```

The tree it prints is your map of what draws UI. Use it to decide what to send - and note
that **most leaf components can simply be added with `-f` by name, without you reading
them in full**. The agent reads the source you pass; you do not need to pre-read a button,
a card, a banner, or a theme file just to forward it. Reading is expensive, so spend it
where it pays off: discovering PRODUCT BEHAVIOR - what the screen is for, the user flows
and entry points, the data model, and the full state list (section 5.2). Pre-read a leaf
only when you must describe something the source alone won't tell you (e.g. which of its
internal states to render). Forwarding files cheaply + reading deeply for behavior is far
more token-efficient than opening every file in the tree.

Then, once you have settled the file set, run lint a final time on **every file you plan
to send** and **resolve every `[missing]` node before you call `edit`**:

```bash
mowgli edit --lint -f <every file you plan to send>
```

For each `[missing]`, make a deliberate choice: either add the file with `-f`, or
consciously decide it renders no visible UI (a hook, a type, a context, a pure-logic util)
and note why. It is not acceptable to run `edit` with `[missing]` entries you have not
accounted for - that unexamined leaf is precisely the one that comes back wrong.

The check is basename-only and ignores bare npm packages and assets, so it flags
candidates, not certainties: it will not catch a child pulled in via a path alias it
cannot resolve, and a `[ok]` only means a file with that basename was provided. So treat a
clean run as the floor, not the ceiling - you still owe the screen a real read of its
render path down to the leaves.

### Enumerating states (do not skimp here)

A "state" is any visually distinct rendering. Find them by:
- Tracing every overlay-gating boolean (`useState(false)` for modals, confirm dialogs,
  inline forms). Each open/closed is a state.
- Treating a **complex component as a state machine**, not one state. A single modal can
  have internal steps - a plan picker can branch into charge-preview, pricing-reference,
  create-team, and cancel-confirm sub-views. Enumerate each branch as its own state.
- Covering empty / loading / error / success variants, not just the happy path.

### Batch sizing

- **New screens:** roughly 2-5 per `edit` invocation. Each new screen `.tsx` MUST get a
  matching `<screen>` entry in `frontend.xml` in the SAME edit, or the edit is rejected
  with "has no spec". Tell the agent this explicitly when creating screens.
- **Complex single screens** (many states / heavy layout): one screen per invocation.
  Scale down by complexity, not just count.
- Do NOT go one-state-at-a-time for incremental additions; that is needlessly granular.
  Add multiple states to an existing screen in a single edit.

### Verify on signal, not by reflex

The edit response already returns a stat summary (files touched, `+/-` per file, version
bump). Read THAT. In the common case where it matches what you expected - the right files
changed, a new screen added the lines you'd expect, nothing you marked "do not touch" was
modified - you're done; move on. Do NOT reflexively re-`read` the `.tsx` and `frontend.xml`
after every edit: it burns tokens and time for a result that almost always just worked.

Pull the full patch with `diff --from N --to M`, or re-`read` a file, only when the summary
gives you a reason to: a no-diff (the edit changed nothing - fix and rerun), an unexpected
file set, a suspicious line count (a "small" change that rewrote a file, or an "additive"
one that removed lines), a touched do-not-touch file, or when the user reports something
wrong.

---

## 6. Hard-won lessons (fidelity failure modes to avoid)

These come from real pushes that came out wrong. Every one traced to the instruction or
the context selection, not the agent.

- **Walk the import tree down to the LEAF component. Do not stop at the view/page.**
  Providing a parent like `AppHeader.tsx`, `DesignView.tsx`, or `ChatPanel.tsx` is not
  enough when the real UI lives in children it imports (`HeaderActions`, `CanvasControls`,
  `ChatBubble`, the question-card wrapper). If you only give the parent and describe the
  child from memory, the child comes out wrong. Open each imported component that
  contributes visible UI and either `-f` it or describe it from its actual source.
  `mowgli edit --lint` exists to force exactly this check: run it before every push and
  clear every `[missing]` it reports (see section 5's gate). It is a forcing function to
  stop you sending on autopilot, not a substitute for reading the render path yourself.
- **Never hand-wave with "use plausible icons" or "a few representative buttons".** That
  is an instruction to fabricate. A header that really has seven controls (credits,
  feedback, share, presence, connect-MCP, export, theme) will render with two if you only
  name two.
- **Do not flatten genuinely different states.** Saying "state B: same as A but with
  different text" makes the agent produce a near-identical screen even when you provided
  source showing a different layout. Describe what is actually different, structurally.
- **Sample data is fine, but flag it.** Prices, names, dates, and copy you invent are not
  from the live API. Record in the version metadata that values are representative
  samples so a future sync does not treat them as real.
- **The edit is stateless.** You cannot add to a previous edit conversation. If an edit is
  refused or comes out wrong, rerun the FULL corrected instruction plus any newly required
  files as a brand-new invocation.
- **Concurrency:** an action fails if another action is already running on the project.
  Run edits sequentially; each `edit` builds on the current HEAD.
- **Shell quoting (zsh):** unquoted variables do not word-split in zsh, so building flags
  in a variable (`B="--base-url x"; mowgli $B ...`) sends them as one argument. Inline the
  flags or quote/array them.

---

## 7. First-pass checklist

When the user asks to push a codebase into Mowgli:

1. `mowgli projects` -> identify the project id (ask if ambiguous).
2. `mowgli -p <id> info` -> read project metadata; reconcile contradictions with the user;
   write/update metadata if missing or stale (codebase, stack, component libs, path map,
   theme token->hex table, current commit).
3. `mowgli -p <id> versions` -> read recent version `userdata` for the last synced commit;
   that is your diff baseline.
4. Map changed source files -> affected screens + SPEC.md + frontend.xml.
5. For each screen: `mowgli edit --lint -f <page file>` first to map the import tree
   cheaply. Read the page (and routing/backend) to understand the screen's role, how a
   user reaches it, the journeys/data it touches, and the full state list - reading is for
   BEHAVIOR, not for transcribing leaves. Forward the page + its UI leaves + theme via
   `-f` (leaves can go by name, unread); when adding to a project that already has
   screens, @-mention one existing screen as do-not-touch reference. Write a precise
   instruction centered on role/connections/journeys/states; name any do-not-touch scope.
   Re-run `mowgli edit --lint -f <all files>` and resolve every `[missing]` (add it, or
   confirm it draws no UI) - do NOT edit with unaccounted missing leaves. Only then
   `edit`. Batch 2-5 simple screens, or one complex screen, per invocation.
6. Read the edit's stat summary; only `diff`/re-`read` if it looks wrong (no-diff,
   unexpected file set or line counts, a touched do-not-touch file). Rerun fresh if no-diff
   or wrong.
7. `mowgli -p <id> version <n> set-meta` -> record changes, the commit shorthash, and any
   intentional divergence/omissions.
