# Report Card System — Technical Specification v2

**Version:** 2.0  
**Date:** April 24, 2026  
**Status:** Draft  
**Reference:** mockup-4-teacher.html, mockup-4-student.html, mockup-4-admin.html

---

## Table of Contents

1. [System Overview & Principles](#1-system-overview--principles)
2. [Authentication & User Roles](#2-authentication--user-roles)
3. [Data Organization](#3-data-organization)
4. [Database Schema](#4-database-schema)
5. [Teacher Interface](#5-teacher-interface)
6. [Student Interface](#6-student-interface)
7. [Advisor Interface](#7-advisor-interface)
8. [Admin — Users & Settings](#8-admin--users--settings)
9. [Admin — Terms, Rosters & Comment Banks](#9-admin--terms-rosters--comment-banks)
10. [Admin — Prompts, Assignments & Report Cards](#10-admin--prompts-assignments--report-cards)
11. [Admin — Analytics & Data Management](#11-admin--analytics--data-management)
12. [Keyboard Navigation & Accessibility](#12-keyboard-navigation--accessibility)

---

## 1. System Overview & Principles

The Report Card System is a school-wide web application for drafting, reviewing, and distributing student report cards. It coordinates four user types — teachers, advisors, students, and admins — through a structured workflow that ends in bulk PDF export.

### Core workflow

1. Admin configures terms, rosters, comment banks, and validation rules before a reporting period opens.
2. Students submit self-reflections for each enrolled class.
3. Teachers select skill comments from a standard bank, add custom comments, write a narrative, and finalize each report card.
4. Advisors review finalized report cards for their assigned advisees, make edits if needed, and mark each as reviewed.
5. Admin bulk-exports reviewed report cards as PDFs for distribution.

### Key design principles

**Local-first.** All user-facing writes (comment selections, narrative edits, reflection text) are persisted locally and reflected in the UI immediately. Background sync to the server happens on blur or on a periodic timer (every few seconds). If the connection is lost, changes queue locally and sync when the connection is restored. The UI shows a "saved" indicator that reflects sync status.

**Configurable validation.** Minimum/maximum requirements (character counts, comment counts, required skills) are defined globally by admin — one configuration for Midterm reporting periods and one for Final reporting periods. These apply automatically to every term of that type. Admin does not need to reconfigure rules per term.

**Finalization as access control.** A teacher finalizing a report card is the mechanism that transfers edit access to the advisor. An unfinalized card is editable only by the teacher; a finalized card is editable only by the advisor. Neither can edit simultaneously.

**Immutable reflections.** Once a student submits a reflection, it becomes read-only. Only admin (via spoofing) can reset a submitted reflection.

**Text snapshots on finalization.** At finalization, all selected comment text is copied into the report card record. The stored report card does not reference the comment bank by foreign key — it stores the comment text directly. This ensures report cards remain stable even if the comment bank is later replaced or edited.

**Universal roles.** Roles are not distinct user types — they are attributes of a user record. A person can be a teacher, advisor, and admin simultaneously. Role changes take effect on the user's next page load.

---

## 2. Authentication & User Roles

### Authentication

All users authenticate via Google OAuth 2.0 using their school-issued Google account. There is no separate username/password system. On first login, a user record is created automatically from the OAuth profile (email address and display name). The email address is the identity anchor and cannot be changed after account creation.

Session management, token storage, and refresh strategy are implementation decisions for the backend engineer. The frontend assumes a session cookie or bearer token is available after login and handles 401 responses by redirecting to the login page.

### Spoofing

Admin users can assume the identity of any other user. During a spoof session:

- A persistent banner is displayed at the top of every page indicating the admin is spoofing, with an "Exit" button to return to the admin view.
- The spoofed user's full interface is shown with their data and permissions.
- The spoof session is recorded in the Audit Log (see Section 11).
- Spoof is intended for tasks like resetting a submitted reflection, adding a comment on behalf of a student, or debugging a user's view.

### User roles

Roles are stored as a set on the user record. A user can hold multiple roles simultaneously. The roles are:

| Role | Can do |
|------|--------|
| **Teacher** | Access classes they are assigned to teach; draft and finalize report cards for those students |
| **Advisor** | Access report cards for their assigned advisees across all classes; edit and mark as reviewed after teacher finalizes |
| **Student** | Submit reflections for enrolled classes |
| **Admin** | Full system access: user management, configuration, analytics, data management, spoofing |

**Role assignment rules:**
- A teacher with no advisory assignment sees only "My Classes" mode; the "My Advisory" tab is hidden.
- An advisor with no teaching assignment sees only "My Advisory" mode; the class roster is empty.
- A person with both roles sees both modes and switches between them via tabs in the left sidebar.

### Permissions summary

| Action | Teacher | Advisor | Student | Admin |
|--------|---------|---------|---------|-------|
| View own class roster | ✓ | — | — | ✓ |
| Edit report card (unfinalized) | ✓ | — | — | via spoof |
| Finalize report card | ✓ | — | — | — |
| Edit report card (finalized) | — | ✓ | — | via spoof |
| Mark report card as reviewed | — | ✓ | — | — |
| Unfinalize report card | — | — | — | ✓ |
| Submit reflection | — | — | ✓ | via spoof |
| View submitted reflection | ✓ (own class) | ✓ (advisees) | ✓ (own) | ✓ |
| Export PDFs (individual) | ✓ (own/advisees) | ✓ (advisees) | — | ✓ |
| Bulk export PDFs | — | — | — | ✓ |
| All admin functions | — | — | — | ✓ |

---

## 3. Data Organization

### Hierarchy

All data is organized hierarchically:

```
School Year  (e.g., "2025-2026")
└── Term  (e.g., "S1", "S2", "I1" — flexible naming and ordering)
    ├── Reporting Period: Midterm  (optional)
    └── Reporting Period: Final  (optional)
```

A **Term** has one roster of students and classes. Within that term, admin configures which reporting periods are active (Midterm, Final, or both). Both reporting periods within a term share the same roster — there is no separate enrollment per period.

In the grading period dropdown (seen by teachers and advisors), each active period appears as a distinct selectable entry, e.g., "25-26 S1 Midterm" and "25-26 S1 Final". A report card record is created per student per class per reporting period — so a student in a term with both Midterm and Final has two separate report card records per class.

### Terms

Admin creates and manages terms. Each term has:
- A school year label (text, e.g., "2025-2026")
- A term name (text, e.g., "S1")
- An order integer (sequence position within the school year)
- Start and end dates
- Which reporting periods are enabled (Midterm, Final, or both)
- An optional student reflection freeze date — after this date, students cannot edit or submit reflections for this term
- A locked flag — when locked, the term is read-only for all teachers and advisors (admin can still view and export)

Locking is a one-way operation from the admin interface — there is no unlock button in the UI. Locking is intended for use once report cards have been distributed.

### Comment bank assignment

By default, the entire school uses a single school-wide standard comment bank. Admin can override this at the department level or individual course level. The hierarchy is:

```
School default
└── Department override (if set)
    └── Course override (if set)
```

Each course resolves to exactly one comment bank — the most specific override in the chain. See Section 9 for bank management and Section 10 for assignment.

### Validation rules

Admin maintains two global validation rule sets — one for Midterm reporting periods and one for Final reporting periods. Each rule set specifies minimum/maximum requirements for student reflections, skill comments, and narratives. Rules with a value of 0 are disabled and not enforced. These rule sets apply automatically to every term of the matching type — no per-term configuration is needed.

### Comment bank CSV format

The standard comment bank is imported as a five-column CSV:

```
Transferable Skill, Skill Area, Subskill, Strength, Growth
```

Each row defines one subskill and its two comment options. Example:

```csv
Transferable Skill,Skill Area,Subskill,Strength,Growth
Ownership of Learning,Motivation,Curiosity-fueled motivation,"You're motivated by curiosity, asking ""why?"" and digging for understanding.","Work on asking ""why"" more consistently—push yourself to dig into the reasoning, not just the steps."
```

The four-level hierarchy (Transferable Skill → Skill Area → Subskill → Strength/Growth) is derived entirely from the CSV rows. There is no separate entity for each level — the hierarchy is implicit in the data.

### Custom comment CSV format

Teachers can export and import their personal custom comment banks as a two-column CSV:

```
skill_area, comment
```

`skill_area` corresponds to a Skill Area name from the standard bank. The custom comment appears in the comment panel for that skill area alongside standard comments.

---

## 4. Database Schema

All field types are suggestions; the backend engineer should adapt to the chosen database and ORM. Foreign keys are noted where relationships are critical to enforce.

### users

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| email | text UNIQUE NOT NULL | School Google account; identity anchor; immutable after creation |
| name | text NOT NULL | Display name from Google OAuth |
| roles | text[] | Array; values: `teacher`, `advisor`, `admin`, `student` |
| is_active | boolean DEFAULT true | False = archived; login denied but data preserved |
| created_at | timestamptz | |
| last_login_at | timestamptz | |

### terms

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| school_year | text NOT NULL | e.g., "2025-2026" |
| name | text NOT NULL | e.g., "S1", "T2" |
| sort_order | integer NOT NULL | Sequence within the school year |
| start_date | date | |
| end_date | date | |
| has_midterm | boolean DEFAULT false | Whether Midterm reporting period is active |
| has_final | boolean DEFAULT true | Whether Final reporting period is active |
| reflection_freeze_date | date NULLABLE | Students cannot edit reflections after this date |
| is_locked | boolean DEFAULT false | Read-only for teachers/advisors when true |
| created_at | timestamptz | |

### classes

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| term_id | uuid FK → terms.id NOT NULL | |
| teacher_id | uuid FK → users.id NOT NULL | |
| course_name | text NOT NULL | |
| course_code | text NULLABLE | |
| department | text NULLABLE | Used for comment bank and prompt assignment lookup |
| sort_order | integer | Display order within teacher's roster |

### enrollments

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| student_id | uuid FK → users.id NOT NULL | |
| class_id | uuid FK → classes.id NOT NULL | |
| UNIQUE | (student_id, class_id) | |

### advisories

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| advisor_id | uuid FK → users.id NOT NULL | |
| student_id | uuid FK → users.id NOT NULL | |
| term_id | uuid FK → terms.id NOT NULL | Advisory assignments can change per term |
| UNIQUE | (advisor_id, student_id, term_id) | |

### comment_banks

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| name | text NOT NULL UNIQUE | Display name |
| is_school_default | boolean DEFAULT false | Only one row should have this true at a time |
| created_at | timestamptz | |
| updated_at | timestamptz | |

### comment_bank_entries

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| bank_id | uuid FK → comment_banks.id NOT NULL | |
| transferable_skill | text NOT NULL | Top-level skill group label |
| skill_area | text NOT NULL | Second-level grouping |
| subskill | text NOT NULL | Third-level label shown in the comment panel |
| strength_text | text NOT NULL | Comment text shown when Strength is selected |
| growth_text | text NOT NULL | Comment text shown when Growth is selected |
| sort_order | integer | Display order within the bank |

Indexes: `(bank_id)`, `(bank_id, transferable_skill)`, `(bank_id, skill_area)`.

### bank_assignments

Stores comment bank overrides at department and course level. The school default is tracked on `comment_banks.is_school_default`.

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| term_id | uuid FK → terms.id NOT NULL | Assignments are per-term |
| bank_id | uuid FK → comment_banks.id NOT NULL | |
| scope | text NOT NULL | `department` or `course` |
| department | text NULLABLE | Required when scope = `department` |
| class_id | uuid FK → classes.id NULLABLE | Required when scope = `course` |

Resolution at query time: for a given class in a given term, find the most specific assignment (course → department → school default).

### teacher_custom_comments

Teachers' personal reusable comment banks. Not shared between teachers; exported as CSV for sharing.

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| teacher_id | uuid FK → users.id NOT NULL | |
| skill_area | text NOT NULL | Matches a Skill Area from the standard bank |
| comment_text | text NOT NULL | |
| sort_order | integer | |
| created_at | timestamptz | |

### reflection_prompts

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| name | text NOT NULL | Internal label for admin reference |
| prompt_text | text NOT NULL | Shown to students |
| min_chars | integer DEFAULT 0 | 0 = not enforced |
| max_chars | integer DEFAULT 0 | 0 = no limit |
| created_at | timestamptz | |

### prompt_assignments

Stores which prompt applies to a given scope in a given term. School-wide default is a row with scope = `universal`.

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| term_id | uuid FK → terms.id NOT NULL | |
| prompt_id | uuid FK → reflection_prompts.id NOT NULL | |
| scope | text NOT NULL | `universal`, `department`, or `course` |
| department | text NULLABLE | |
| class_id | uuid FK → classes.id NULLABLE | |

Resolution: course → department → universal, same as bank assignments.

### validation_configs

Two rows only — one for Midterm, one for Final. These are global and not per-term.

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| period_type | text NOT NULL | `midterm` or `final`; UNIQUE |
| reflection_min_chars | integer DEFAULT 0 | |
| reflection_max_chars | integer DEFAULT 0 | 0 = no limit |
| skill_min_areas | integer DEFAULT 0 | Minimum skill areas with ≥1 comment |
| skill_required_names | text[] | Skill names that must have ≥1 comment; empty = none required |
| skill_min_total_comments | integer DEFAULT 0 | |
| skill_max_per_area | integer DEFAULT 0 | 0 = no limit |
| skill_custom_max_chars | integer DEFAULT 0 | Max chars for teacher-typed custom comments |
| narrative_min_chars | integer DEFAULT 0 | |
| narrative_max_chars | integer DEFAULT 0 | |
| updated_at | timestamptz | |

### report_cards

One record per student per class per reporting period.

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| term_id | uuid FK → terms.id NOT NULL | |
| class_id | uuid FK → classes.id NOT NULL | |
| student_id | uuid FK → users.id NOT NULL | |
| teacher_id | uuid FK → users.id NOT NULL | Snapshot — denormalized for query convenience |
| reporting_period | text NOT NULL | `midterm` or `final` |
| status | text NOT NULL DEFAULT 'draft' | `draft` or `finalized` |
| narrative | text DEFAULT '' | |
| is_advisor_reviewed | boolean DEFAULT false | |
| finalized_by | uuid FK → users.id NULLABLE | |
| finalized_at | timestamptz NULLABLE | |
| reviewed_by | uuid FK → users.id NULLABLE | |
| reviewed_at | timestamptz NULLABLE | |
| last_exported_at | timestamptz NULLABLE | |
| admin_note | text NULLABLE | Internal admin annotation; not visible to teachers/advisors/students |
| created_at | timestamptz | |
| updated_at | timestamptz | |
| UNIQUE | (class_id, student_id, reporting_period) | |

### report_card_skill_comments

Stores comment selections for a report card. Each row is one selected comment (either a standard bank comment or a custom comment). Comment text is stored as a snapshot at the time of selection — not as a foreign key to the bank entry.

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| report_card_id | uuid FK → report_cards.id NOT NULL | |
| transferable_skill | text NOT NULL | Snapshot label |
| skill_area | text NOT NULL | Snapshot label |
| subskill | text NULLABLE | Null for custom comments |
| comment_type | text NULLABLE | `strength`, `growth`, or null for custom |
| comment_text | text NOT NULL | Snapshot of the text at time of selection |
| is_custom | boolean DEFAULT false | True for teacher-written custom or one-time comments |
| is_one_time | boolean DEFAULT false | True for comments added inline for this student only (not from teacher's saved bank) |
| sort_order | integer | |

### student_reflections

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| student_id | uuid FK → users.id NOT NULL | |
| class_id | uuid FK → classes.id NOT NULL | |
| term_id | uuid FK → terms.id NOT NULL | |
| reporting_period | text NOT NULL | `midterm` or `final` |
| prompt_id | uuid FK → reflection_prompts.id NULLABLE | Which prompt was shown; nullable if prompt deleted |
| prompt_text_snapshot | text NULLABLE | Snapshot of prompt text at time of submission |
| draft_text | text DEFAULT '' | Current draft; updated on each save |
| submitted_text | text NULLABLE | Locked copy at time of submission |
| status | text NOT NULL DEFAULT 'not_started' | `not_started`, `draft`, or `submitted` |
| submitted_at | timestamptz NULLABLE | |
| UNIQUE | (student_id, class_id, reporting_period) | |

### export_records

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| exported_at | timestamptz NOT NULL | |
| exported_by | uuid FK → users.id NOT NULL | Admin user |
| export_type | text NOT NULL | `bulk` or `individual` |
| report_card_ids | uuid[] | Array of included report card IDs |
| filter_description | text NULLABLE | Human-readable summary of filters applied |

### audit_log

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| occurred_at | timestamptz NOT NULL | |
| actor_id | uuid FK → users.id NOT NULL | Admin who performed the action |
| spoofed_as_id | uuid FK → users.id NULLABLE | If action was taken while spoofing |
| action | text NOT NULL | Short action code, e.g., `user.archive`, `term.lock`, `spoof.start` |
| description | text NOT NULL | Human-readable description |
| metadata | jsonb NULLABLE | Structured data relevant to the action |

### user_role_history

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| user_id | uuid FK → users.id NOT NULL | |
| changed_by | uuid FK → users.id NOT NULL | |
| old_roles | text[] | |
| new_roles | text[] | |
| changed_at | timestamptz | |

### Key relationships diagram

```
users ──< classes (as teacher)
users ──< enrollments >── classes
users ──< advisories >── users (advisor → student)
terms ──< classes ──< enrollments
terms ──< report_cards
classes ──< report_cards ──< report_card_skill_comments
classes ──< student_reflections
report_cards >── student_reflections (1:1 per period)
comment_banks ──< comment_bank_entries
comment_banks ──< bank_assignments
reflection_prompts ──< prompt_assignments
```

---

## 5. Teacher Interface

### Layout

Three regions:

- **Left sidebar** (~290px fixed width): grading period selector, mode tabs, class roster with students
- **Main content area** (flex-grow): student editing area or empty state
- **Right comment panel** (~455px fixed width, conditionally visible): skill comment selection UI

### Left sidebar

**Header area:**
- Teacher's display name
- Grading period dropdown — lists all active reporting periods for terms assigned to this teacher (e.g., "25-26 S1 Midterm", "25-26 S1 Final", "25-26 S2 Final"), ordered by term sort order and period type (Midterm before Final within a term)
- Prev/Next arrows (▲/▼) — step to the previous or next student in the entire visible roster, wrapping across class boundaries; arrows are disabled when the roster is empty

**Mode tabs** (shown only when the user has both teacher and advisor roles):
- "My Classes" — default; shows the teacher's own class roster
- "My Advisory" — switches to the advisor view (see Section 7)

**Class roster (My Classes mode):**
- One collapsible section per class assigned to the teacher for the selected period
- Clicking the class header toggles expand/collapse for that class only; it does not navigate to a student
- Each student row shows the student's name, a round status dot (report card status), and a square dot (reflection submitted)
- Clicking a student row makes that student active in the main content area

**Status legend** (bottom of sidebar):
- Round dot colors: gray = not started, orange = in progress, blue = complete (all requirements met but not finalized), green = finalized
- Square dot: filled = reflection submitted, empty = not submitted

**Grading period change behavior:**
When the teacher changes the grading period, the system tries to keep the same student active:
1. Look for the previously active student in the same class (by class name) in the new period — if found, activate them there.
2. If not found in the same class, search all classes in the new period for the student by name — if found, activate them.
3. If not found, show the empty state.

### Main content area — empty state

Shown when no student is selected: a centered message "← Select a student to begin".

### Main content area — student editing area

Shown when a student is active. Contains the following cards in order:

#### Student info card

Read-only display: student name, course name, teacher name, grading period. If the report card has been advisor-reviewed, shows a "Reviewed" badge.

If the student has submitted a reflection, the reflection text is shown below the metadata in a read-only box labeled "Student Reflection". If no reflection has been submitted, this box is omitted.

#### Skills section

One block per Transferable Skill in the comment bank (in bank order). Each block contains:

- The Transferable Skill name (heading)
- A "Skill Selection" button — clicking opens the right comment panel for this skill
- The selected comments for this skill displayed as a bulleted list of text snippets

If no comments are selected for a skill, the bullet list area is empty (no placeholder text).

#### Narrative card

- A textarea labeled "Overall Comment" (or equivalent)
- Minimum height ~110px; vertically resizable
- Auto-expands to fit content
- A save hint line below the textarea: shows "Changes saved automatically" in gray normally, turns green briefly after a successful sync, and shows a warning if sync fails

#### Finalize row

- Checkbox: "Mark as finalized"
- The checkbox is **disabled** until all applicable validation rules are met
- When disabled, an inline hint lists the specific unmet requirements, e.g.: "Need comments in 2 more skill areas · Narrative needs 20 more characters"
- When enabled and the teacher checks it, a confirmation modal appears: warn that finalizing will lock the card and open it to the advisor; require a second click to confirm
- Once finalized, the checkbox shows as checked and disabled (teacher cannot uncheck from the UI); to unfinalize, admin must use the Report Cards admin panel

#### Advisor review row

Shown **only** when the report card is finalized. Displays one of:
- "✓ Reviewed by [Advisor Name]" (green) — if the advisor has marked it reviewed
- "Pending advisor review" (gray) — if not yet reviewed

This row is read-only for the teacher; it is a status display only.

#### Preview button

At the bottom of the editing area: "Preview Report Card" button. Opens the preview modal (see below).

### Right comment panel

Opens when the teacher clicks "Skill Selection" for any Transferable Skill. Only one panel is open at a time; opening another closes the current one.

**Panel header:** Transferable Skill name + a close button (✕).

**Standard comments section:**

Comments are grouped by Skill Area (a subheading), then by Subskill (each subskill is a row). Each Subskill row contains:

- A **Subskill label** (clickable text)
- An **S button** (Strength)
- A **G button** (Growth)

**Selection behavior — this is non-obvious and must be implemented exactly:**

- Exactly zero or one comment can be selected per subskill (never both S and G simultaneously).
- Clicking the **S button** directly: if Strength is already selected, deselects it; otherwise selects Strength (and deselects Growth if it was selected).
- Clicking the **G button** directly: same logic for Growth.
- Clicking the **Subskill label text** cycles through states in order: **none → Strength → Growth → none**, repeating. This is the keyboard-friendly way to cycle without using the mouse on small buttons.
- The selected S or G button appears highlighted/filled; the unselected button appears as an outline.

**Hover behavior on S and G buttons:**

- Hovering over the S button shows the full Strength comment text as a tooltip/hint.
- Hovering over the G button shows the full Growth comment text as a tooltip/hint.
- This allows the teacher to read the full comment text before selecting it, without cluttering the UI when not hovering.
- The tooltip should appear near the cursor and be readable (not truncated). Use a `title` attribute or a custom tooltip component — whichever is used, the text must be the full comment string with no truncation.

**Custom comments section** (below a divider):

Three sub-areas:

1. **One-time comment input:** A text input labeled "Add a note for this student" (or similar) with an "Add" button. Text entered here is saved as a one-time comment for the current student only — it is NOT added to the teacher's reusable comment bank. One-time comments appear as deletable chips or list items below the input. The `is_one_time` flag on `report_card_skill_comments` distinguishes these.

2. **Edit comment bank link:** A small link "Edit comment bank…" that opens the Custom Comment Bank modal for this skill (see below).

3. **Saved custom comments checklist:** The teacher's personal reusable comments for this skill area, displayed as checkboxes. Checking a box adds that comment to the current student's report card; unchecking removes it. Changes apply immediately.

**Panel footer:** A "Done" button closes the panel.

### Custom Comment Bank modal

Opened via "Edit comment bank…" in the comment panel. Displays the teacher's saved custom comments for the current skill.

- A draggable list of existing custom comments, each with a delete button
- An "Add" input field + button to add a new comment to the reusable bank
- **Import/Export:**
  - "Export CSV" — downloads the teacher's full custom comment bank as a two-column CSV (`skill_area, comment`)
  - "Import CSV" — opens a file picker; after file selection, shows a preview with the count of comments found and two options: "Append to existing" or "Replace existing"; a Cancel and "Apply" button complete the import
- The modal has no Save button — changes to the list (add, delete, reorder) save immediately

### Preview modal

A full-size read-only preview of the report card for the active student. Contains:

- Header: student name, course, teacher, term, and reporting period
- Student Reflection section (if submitted)
- Transferable Skills section — shows only skills with at least one comment selected; each skill is a heading followed by the selected comments as bullet points
- Overall Comment section — the narrative text in a bordered box
- "Print / Save PDF" button — generates a PDF using the school branding settings and downloads it locally

The preview modal does not require the report card to be finalized. Teachers can preview at any time to check formatting.

### Sync behavior

- On every blur of an input (textarea, custom comment field), queue a sync to the server.
- Also sync on a periodic timer (~5 seconds) if there are unsynced changes.
- The save hint shows:
  - Default gray text: "Changes saved automatically"
  - Briefly turns green with a checkmark after a successful sync
  - Shows an amber warning if the last sync failed, with a retry indicator

---

## 6. Student Interface

### Layout

Two regions:

- **Left sidebar**: grading period selector, class list with status indicators
- **Main content area**: reflection editing area or empty state

On mobile (narrow viewport), the sidebar becomes a drawer. A "← Classes" back button appears in the main area when a class is active, allowing the student to return to the class list.

### Left sidebar

**Top:** School name / "My Reflections" heading.

**Grading period selector:** Dropdown listing all active reporting periods the student is enrolled in. Students see only periods for which they have at least one class enrollment. Period names follow the same format as the teacher view (e.g., "25-26 S1 Final").

**Class list:** One row per enrolled class for the selected period. Each row shows:
- Course name
- Teacher name
- A status indicator dot:
  - Empty / no fill = not started (no draft text saved)
  - Orange dot = draft in progress
  - Blue dot = submitted

Clicking a class row loads that class's reflection in the main content area and makes the row active (highlighted).

### Main content area — empty state

"← Select a class to begin" shown when no class is selected.

### Main content area — reflection editing area

Shown when a class is active. Contains:

#### Class info card

Read-only: course name, teacher name, grading period.

#### Prompt box

The admin-configured prompt text for this class/department/universal context, displayed in a visually distinct box (e.g., blue background). This is read-only.

#### Reflection textarea

- Large textarea for the student's reflection text
- Minimum height ~180px; auto-expands to fit content
- When reflection is submitted, the textarea is read-only

#### Character count indicator

Shown below the textarea. Displays current character count. Color-coded:
- Gray — textarea is empty
- Orange — below the configured minimum (characters remaining shown)
- Green — at or above the minimum

The minimum threshold comes from the reflection prompt's `min_chars` setting, which is drawn from the active prompt assignment for this class (resolved via the prompt_assignments hierarchy).

#### Save hint

"Changes saved automatically" — same sync indicator behavior as the teacher interface. Drafts sync periodically and on blur.

#### Status badge and submit area

- Status badge: "Not started" / "Draft" / "Submitted"
- Submit button: disabled until the character minimum is met; enabled once met
- When submitted: button is replaced by a notice showing the submission timestamp and "Your reflection has been submitted. Contact your teacher if you need to make changes."

#### Submission confirmation modal

When the student clicks Submit:
- Modal title: "Submit your reflection?"
- Body: explanation that the reflection cannot be edited after submission
- Buttons: "Cancel" and "Submit"
- On confirming: status transitions to Submitted, textarea becomes read-only, sync fires immediately

---

## 7. Advisor Interface

### Overview

The advisor interface is not a separate application — it is the "My Advisory" mode of the same teacher/advisor interface. A user with the advisor role sees a "My Advisory" tab in the left sidebar alongside (or instead of) "My Classes". Switching to the "My Advisory" tab changes the sidebar content and the behavior of the main content area as described below.

### Left sidebar — My Advisory mode

- Grading period dropdown (same behavior as teacher mode)
- Prev/Next arrows (▲/▼) — step through advisees; pressing an arrow loads the next or previous advisee's first class in the current period (no attempt to match the class name that was previously active)
- Advisee list: one row per assigned advisee for the current term. Each advisee row shows the student's name and a summary status (e.g., a dot indicating overall completion across their classes).
- Clicking an advisee expands or selects that advisee. The main content area loads the first class for that advisee in the selected period.
- Within an expanded advisee, individual class rows can be clicked to load a specific report card.

### Main content area — advisor view

Largely identical to the teacher editing area, with these differences:

**Student info card:** Same as teacher view, but with an advisory context label.

**Skills and narrative:** Editable only when the report card is finalized. If not yet finalized, all inputs are read-only and a callout reads: "Awaiting teacher finalization — this report card is not yet available for review."

**Finalize row:** The "Mark as finalized" checkbox is **visible but disabled** for advisors. Advisors cannot finalize or unfinalize.

**Advisor review row:** The "✓ Reviewed by Advisor" checkbox is **active** for the advisor (and visible but inactive for the teacher). The advisor can check or uncheck this box at any time while the report card is finalized. Checking it marks the card as reviewed; unchecking reverts it. The checkbox is only interactable when the report card is finalized.

**Unfinalized report cards in advisory mode:** The content area shows the skill selections and narrative in read-only state with the "Awaiting teacher finalization" callout. The advisor can read but not edit.

### Conflict handling

If a teacher unfinalizes a report card that the advisor has edited since finalization:

- The **next time the teacher opens that report card** (or on the next sync if they already have it open), they see a callout box:  
  *"This report card has been edited by [Advisor Name] since you finalized it. Unfinalize again to edit, or reload to see their changes."*
- The page does not automatically reload.
- If the teacher clicks "Reload," they see the advisor's edits in a read-only view and can decide to finalize again or make further changes.
- If the teacher unfinalizes again (from the reload view), they regain edit access; any advisor edits are preserved in the draft.
- If the advisor is actively viewing the report card when the teacher unfinalizes it: the advisor's edit access is revoked on their next sync event. They see a notification: "This report card has been unfinalized by the teacher and is no longer editable."

### Navigating between advisees

- Prev/Next arrows move through the advisee list in order.
- When the advisor switches to a different advisee, the system opens that advisee's first class in the current period.
- When the advisor returns to a previously viewed advisee, the system reopens the last class the advisor was viewing for that advisee (remembered in session state, not persisted across page loads).

### Historical context

The advisor can switch to any past term while staying on the same advisee:
- The grading period dropdown lists all periods across all terms (not just the current one).
- Switching to a past period loads that advisee's report cards for that period in read-only mode (even if finalized).
- Only the current term's finalized cards are editable.

---

## 8. Admin — Users & Settings

### Admin interface layout

The admin interface is separate from the teacher/student interface. It has:

- **Top nav:** School name/logo, "Admin" badge, current user name, Logout button
- **Left sidebar** (~200px fixed): navigation menu with links to each admin section
- **Main content area:** The active section's content

The left sidebar menu items, in order:
1. Terms & Rosters
2. Report Cards
3. Analytics ← **default landing view on login**
4. Users
5. Comment Banks
6. Student Reflection Prompts
7. Assignments
8. Settings
9. Data Management

### Users section

**Toolbar:** Search input (live filter by name or email) + Role filter dropdown (All / Teacher / Advisor / Admin / Student) + Status filter (Active Only / Active & Archived / Archived Only) + "New User" button.

**User table columns:** Name, Email, Roles (colored tag chips — multiple roles displayed simultaneously), Status (Active / Archived), Last Login, Actions.

**Actions per row:**
- **Edit** — opens the User modal pre-filled with that user's data
- **Spoof** — opens a Spoof Confirmation modal; on confirming, the admin's session switches to that user's interface with a persistent banner
- **Archive** — opens an Archive Confirmation modal; archiving sets `is_active = false`, preventing login while preserving all data. Archived users can be reactivated by editing their account and re-enabling active status.

**New/Edit User modal fields:**
- Full Name (text, required)
- School Email (text, required; must be the school Google account used for OAuth; hint explains this; email cannot be changed after account creation — the field is read-only in edit mode)
- Roles (checkboxes): Teacher, Advisor, Admin, Student — any combination allowed
- Active toggle (edit mode only)

Role changes take effect on the user's next page load. No notification is sent to the user.

### Settings section (Validation Rules)

Two side-by-side cards: **Midterm** and **Final**. Each card has its own Save button. Rules are grouped to match the order sections appear in the report card.

Setting any numeric field to **0** disables that rule; 0-valued rules are not enforced and are not surfaced in the teacher interface. Leave the Required Skills field blank to skip that rule.

**Student Reflection group:**
- Minimum characters (before Submit button activates)
- Maximum characters (0 = no limit)

**Skill Comments group:**
- Minimum skill areas with ≥1 comment selected
- Required skills: comma-separated list of Transferable Skill names; each listed skill must have at least one comment before the teacher can finalize
- Minimum total skill comments per report card
- Maximum comments per skill area
- Maximum characters for a custom skill comment (applies to both one-time and saved custom comments)

**Narrative group:**
- Minimum characters
- Maximum characters (0 = no limit)

Each card saves independently. A success indicator confirms when saved.

---

## 9. Admin — Terms, Rosters & Comment Banks

### Terms & Rosters section

Three sub-tabs: **Terms**, **Class Sections**, **Import**.

#### Terms sub-tab

Displays all school years with their terms grouped by year. Each term row shows: term name, date range, which reporting periods are enabled (Midterm / Final badges), locked status, and action buttons.

- Locked terms display a "Locked" badge; their Edit button is grayed out.
- Unlocked terms show Edit and Lock buttons.

**Create / Edit Term modal:**
- School Year (text, e.g., "2025-2026")
- Term Name (text, e.g., "S1")
- Order (integer — position within the school year; determines dropdown sort order in teacher view)
- Start Date, End Date
- Reporting Periods: checkboxes for Midterm and Final (both enabled by default)
- Student Reflection Freeze Date (optional date — students cannot submit or edit reflections after this date)

**Lock Term:** Sets `is_locked = true`. Once locked, there is no unlock button — this is intentional. Admin can still view and export from locked terms. Locking is intended for use after report cards are distributed.

#### Class Sections sub-tab

Displays class sections for a selected term, filterable by department. Table columns: Course Name, Section/Period, Teacher, Department, Students (enrollment count).

- Clicking a course name row opens a **Students modal** for that class: a searchable list of enrolled students with an "+ Add Student" button and a remove button per row.
- Each class row has Edit and Remove buttons.

**Add Class modal:**
- Term selector
- Course Name
- Department
- Teacher (dropdown of users with teacher role)
- After saving, the class appears in the roster with 0 students; students are added via the Students modal or via CSV import.

#### Import sub-tab

Two panels side by side:

**Import from CSV:**
- File drop zone (or click to browse)
- Expected CSV columns: `teacher_email, course_name, student_email` (exact column names TBD by backend engineer; document them in the interface as a format hint)
- After file selection, a preview appears: row counts (teachers found, classes, students, total rows) and the first few rows of the file
- Import mode radio: "Merge with existing roster" (additive) or "Replace existing roster for this term" (overwrites all class and enrollment data for the term)
- Cancel and "Apply Import" buttons

**Export Roster:**
- Term selector
- Format: "Class roster (teacher, course, student)" or "Student list only"
- "Download CSV" button

**Blackbaud / Canvas integration:** The Import sub-tab also surfaces integration options for syncing roster data from Blackbaud and/or Canvas. Specific API details and sync scheduling are backend implementation decisions; the UI should provide at minimum: connect/authenticate, trigger a sync, and view sync history.

### Comment Banks section

Manages standard comment bank files. Assignment of banks to departments and courses is in the Assignments section.

**School-wide default bar** (above the bank list): a dropdown to select which bank is the school-wide default, with a "Set Default" button. A note explains that all courses use this bank unless overridden in Assignments.

**Bank list table:** Bank Name (with "School Default" badge on the current default), Subskills (count of rows in the bank), Last Modified, Applied To (summary of department/course assignments).

**Actions per bank:**
- **Replace (⬆):** Opens Upload modal with bank name pre-filled and locked; uploading a new CSV replaces all entries in that bank entirely. Button reads "Upload & Replace".
- **Download (⬇ CSV):** Downloads the bank in the standard five-column format.
- **Delete:** Removes the bank. Cannot delete if the bank is currently assigned to any department or course in any term, or if it is the school-wide default. The UI shows which assignments must be changed first.

**Upload New Bank button:** Opens a modal:
- Bank Name (required, must be unique)
- File drop zone (CSV)
- A format reference showing the expected column headers and one example row
- Submit button reads "Upload & Preview" — after clicking, the system parses the CSV and shows a preview (row count, first few entries, any parse errors) before creating the bank. Admin can cancel from the preview or confirm to create.

**Note:** Individual teachers' custom comments are not managed here. Teachers manage their own via the comment panel.

---

## 10. Admin — Prompts, Assignments & Report Cards

### Student Reflection Prompts section

Prompts are authored here. Assignments to specific departments and courses are managed in the Assignments section.

**Prompt list:** Displayed as cards, filterable by term and scope (Universal / Department). Each card shows:
- Prompt name and scope tag
- Terms the prompt is active in
- A preview of the prompt text (truncated)
- Min and max character requirements
- Edit and Delete action buttons

**Create / Edit Prompt modal:**
- **Prompt Name** (internal label for admin reference)
- **Default For** (radio buttons):
  - "All classes (Universal)" — this prompt is the school-wide default for the selected terms
  - "A specific department" — a department dropdown appears; this prompt is the department-level default
  - *Note: Assigning to individual courses is done in Assignments, not here.*
- **Terms** (checkboxes for available terms; may include an "All future terms" option)
- **Prompt Text** (textarea — shown to students)
- **Min Characters** and **Max Characters** (0 = not enforced)

**Delete Prompt:** If the prompt is currently the active assignment for any class, department, or universal scope, those assignments revert to the next broader default automatically. Admin should verify assignments after deleting.

### Assignments section

A unified view for managing which comment bank and which reflection prompt applies to each department and course. All assignments are per-term; changing one term does not affect another.

**Term selector** (top): changing the term reloads the assignment table. When a new term is created, it inherits all assignments from the most recent preceding term automatically.

**Assignment table:** Three columns — Scope, Comment Bank, Student Prompt — plus an action column. Rows are organized in a three-level hierarchy:

**School Default row** (always first): dropdowns select the school-wide default bank and prompt. An "Apply to all depts & courses" button resets all department and course overrides for this term, reverting everything to inherit from the school default.

**Department rows** (one per department): each department row shows its current bank and prompt assignment in one of two states:
- **Inherited:** selector is grayed out with the inherited value shown and a badge identifying the source (e.g., "school"); an "Override" link activates the selector.
- **Overridden:** selector is active with the chosen value; a "× Reset" link reverts to inherited.

An "Apply to all courses" button within each department row clears course-level overrides for that department (courses revert to inheriting from the department).

Clicking a department row header toggles its course rows visible/hidden.

**Course rows** (indented under each department when expanded): same inherited/overridden pattern, inheriting from the department. The inherited badge shows "dept".

Individual cell changes save immediately (no separate Save step). The effective value resolution for any course is: course override → department override → school default.

### Report Cards section

Two sub-tabs: **Status & Export** and **PDF Settings**.

#### Status & Export sub-tab

**Filters:** Term, Reporting Period (All / Midterm / Final), Teacher, Status (All / Draft / Finalized / Reviewed), Export Status (Any / Not Exported / Exported / Newly Reviewed). All filters apply simultaneously.

**Stat strip** below filters: counts for current filter set — Total, Draft, Finalized, Reviewed, Exported.

**Table columns:** Student, Class, Teacher, Period, Status (color-coded pill badge), Advisor Reviewed (checkmark), Exported (checkmark), Actions.

**Actions per row:**
- **Unfinalize** (only on finalized cards): reverts the report card to Draft status. The teacher regains edit access. Any advisor edits made after finalization are preserved; the teacher will see the standard conflict notification when they next open the card.
- **Note**: Opens a modal for the admin to write or edit an internal admin note attached to that report card. Notes are not visible to teachers, advisors, or students.

**Bulk Export button** (top-right of panel): opens a modal:
- Term + Reporting Period selector (e.g., "25-26 S1 — Final")
- Include filter: Reviewed only / Newly reviewed since last export / All finalized
- Optional filter by Teacher
- Optional filter by Class
- A live count preview updates as filters change (e.g., "41 reviewed cards match current criteria")
- "Export N PDFs" button triggers bulk PDF generation and download (e.g., as a ZIP). After export, all included cards are marked with `last_exported_at`.

#### PDF Settings sub-tab

Controls the branding applied to all generated PDFs.

**Branding & Logo:**
- Upload school logo (PNG or SVG; recommended 200×60px; max 1MB)
- School Name (appears in PDF header)
- District / Subtitle (optional second header line)

**Footer:**
- Footer text (appears at the bottom of every page of every exported PDF)

**Boilerplate Text:**
- Introductory paragraph (shown at the top of each report card body, before skills)
- Competency / grading scale note (explains the Strength/Growth designation or grading key)

**Preview:**
- A thumbnail preview of the header/footer layout; an "Open Preview" button opens a full-size layout preview

A single "Save PDF Settings" button persists all fields.

---

## 11. Admin — Analytics & Data Management

### Analytics section

This is the **default landing view** when admin logs in.

**Term + Period selector** (top right of page): selects a specific term and reporting period (e.g., "25-26 S2 — Final") that governs all data on this page.

**KPI cards** (three across the top):
1. **Report Cards Completed:** count and percentage of report cards that are finalized or reviewed out of the total for this term/period
2. **Report Cards Reviewed:** count and percentage marked as advisor-reviewed
3. **Student Reflections Submitted:** count and percentage of enrolled students who have submitted a reflection

**Summary cards** (two side by side):

*Report Card Completion by Class:*
- A one-sentence summary (e.g., "5 of 7 classes are ≥75% completed")
- A progress bar showing the overall completion rate
- "View Breakdown →" button — opens the Completion Breakdown modal

*Student Reflection Submission by Class:*
- Same structure for reflections

**Completion Breakdown modal:** A sortable table — Class, Teacher, Students (enrolled count), Completed, Reviewed, % Done. Clicking a class row expands it to show individual students with their report card status and whether a reflection was submitted. Clicking a student name opens the Spoof Confirmation dialog for immediate admin access to that student's or teacher's view. Modal footer has an "⬇ Export Table" button (downloads CSV).

**Reflection Breakdown modal:** Same structure and behavior as the Completion Breakdown modal, for reflections.

**Reports panel** (below the summary cards):

- **Comment usage report (CSV):** Lists which standard comments were used, by how many teachers and in how many classes, for the selected term.
- **Completion status report (CSV):** Report card and reflection status for all students in the selected term.
- **Teacher custom comment banks:** A teacher selector dropdown + "Download CSV" button — exports that teacher's personal custom comment bank (for admin use, e.g., sharing between colleagues).
- **Custom Report button:** Opens a modal with:
  - Term selector (a specific term+period, or all terms)
  - Column checkboxes: Student name, Class, Teacher, Report card status, Reflection status, Advisor reviewed, Finalized date, Reviewed date, Last exported, Comment count, Narrative character count, Reflection character count
  - "Generate CSV" button

### Data Management section

**Export Data by Year:**
- School year dropdown
- Records in scope preview: shows counts for the selected year (terms, classes, report cards, student reflections, export records)
- Format selector: CSV files (one per data type) or ZIP archive of all
- "⬇ Export Year Data" button — opens a confirmation modal summarizing the record counts; confirming downloads the archive. Data remains in the live system after download.

*Delete Year Records* (danger zone within the same panel):
- Permanently removes all records for the selected year from the live system
- Admin must type "DELETE" in a confirmation input to enable the delete button
- Cannot be undone; admin should verify the exported archive before proceeding
- Intended for data hygiene once records are no longer needed in the live system (typically after 5+ years)

**Restore from Backup:**
- Uploads a previously exported archive (ZIP or CSV) to restore data for a selected school year
- School year selector and file drop zone
- Admin must type "RESTORE" in a confirmation input to enable the restore button
- Restoring replaces all current records for the selected year with the backup contents
- Intended for system recovery; not for routine use

**Data Integrity:**
- "Run Check" button scans for orphaned records, broken foreign-key relationships, and data inconsistencies
- Results appear inline with pass / warn / fail indicators per check category

**Audit Log:**
- A persistent, automatically maintained log of significant admin actions
- Events captured: user role changes, user archiving/reactivation, spoof sessions (start and end), term locks, bulk PDF exports, comment bank uploads/replacements, term creation, data exports, data deletions
- Displayed as a scrollable table: Date, Admin, Action (descriptive text)
- "⬇ Export Full Audit Log" button downloads the complete log as CSV

---

## 12. Keyboard Navigation & Accessibility

The system must be fully operable without a mouse. Every action available by clicking must also be reachable and triggerable by keyboard alone. The goal is efficient use for teachers who enter many report cards sequentially — mouse use should be optional, not required.

### General rules

- All interactive elements are focusable via Tab (forward) and Shift+Tab (backward).
- Focus order follows the visual reading order of the page.
- The focused element always has a visible focus indicator (minimum 2px outline, high-contrast color).
- No keyboard traps — Tab can always leave any component.
- Escape closes any open modal, panel, or dropdown and returns focus to the element that triggered it.
- Semantic HTML is used throughout (`<button>`, `<input>`, `<select>`, `<textarea>`, `<label for>`) so that browser and screen reader defaults work without custom ARIA where native semantics suffice.

### Teacher interface keyboard behavior

**Grading period dropdown:** Standard `<select>` — arrow keys change selection; Enter/Space opens; Escape closes.

**Class roster (left sidebar):**
- Tab moves focus through class headers and student rows.
- Enter or Space on a class header toggles its expand/collapse.
- Enter or Space on a student row activates that student (same as clicking).
- Arrow keys (↑/↓) can navigate between visible student rows without tabbing through class headers — treat the roster as a single composite widget with roving tabindex or `aria-activedescendant`.

**Prev/Next student arrows (▲/▼):**
- Tab-focusable buttons; Enter/Space activates.
- A suggested keyboard shortcut: `Alt+↑` / `Alt+↓` to trigger these from anywhere on the page without having to Tab to the sidebar.

**Mode tabs (My Classes / My Advisory):**
- Tab focuses the tab strip; arrow keys move between tabs; Enter/Space activates the focused tab.

**Skill Selection button:**
- Tab-focusable; Enter or Space opens the right comment panel for that skill.

**Right comment panel:**
- When the panel opens, focus moves to the panel's first focusable element (typically the first subskill label or the close button).
- Tab navigates through all interactive elements in the panel (close button, subskill labels, S buttons, G buttons, one-time input, Edit comment bank link, saved custom comment checkboxes, Done button).
- Escape closes the panel and returns focus to the "Skill Selection" button that opened it.

**Subskill label (cycling):**
- Enter or Space on the subskill label cycles through: none → Strength → Growth → none.
- This is the primary keyboard mechanism for comment selection, replacing the need to Tab to the small S/G buttons.

**S and G buttons:**
- Tab-focusable individually; Enter or Space selects/deselects that comment type.
- Hovering (`:hover`) and focusing (`:focus`) both trigger the tooltip showing the full comment text. The tooltip must also be accessible to screen readers — use `aria-describedby` pointing to a visually hidden element containing the comment text, or use a `<details>` pattern.

**Narrative textarea:** Standard textarea keyboard behavior; Tab exits the textarea (does not insert a tab character).

**Finalize checkbox:** Tab-focusable; Space toggles. When disabled, focus can still land on it but Space has no effect; the inline hint text is associated via `aria-describedby`.

**Preview modal:**
- When opened, focus moves to the modal.
- Tab cycles through modal content (Print/Save PDF button, close button).
- Escape closes the modal.
- Focus returns to the Preview button when closed.

**Custom Comment Bank modal:**
- Same modal focus trapping and Escape behavior.
- The draggable list items should be keyboard-reorderable (e.g., focus a handle button, then use arrow keys to move the item up/down).

### Student interface keyboard behavior

**Class list (left sidebar):**
- Tab/arrow navigation through class rows; Enter activates the selected class.

**Grading period dropdown:** Standard `<select>`.

**Reflection textarea:** Standard textarea.

**Submit button:** Tab-focusable; Enter or Space triggers submit (opens confirmation modal).

**Submission confirmation modal:** Focus trapped in modal; Tab moves between Cancel and Submit buttons; Escape = Cancel; Enter on Submit confirms.

### Admin interface keyboard behavior

**Left sidebar navigation:** Tab through menu items; Enter navigates to the section.

**Tables with action buttons:** Each row's action buttons are tab-focusable in order. For long tables, consider adding a "skip to content" link at the top.

**Modals:** All admin modals follow the same focus-trap pattern — Tab cycles within the modal, Escape closes it, focus returns to the triggering element.

**User table filter dropdowns:** Standard `<select>` elements. Search input is a standard text field.

**Confirmation inputs (DELETE / RESTORE):** Standard text inputs; the confirm button enables reactively as the user types the required string.

### Screen reader support

- All images and icons have `alt` text or `aria-label`.
- Status dots in the sidebar (report card status, reflection submitted) have `aria-label` values that describe their meaning (e.g., `aria-label="Report card: finalized; Reflection: submitted"`).
- Color is never the sole means of conveying information — status dots are also distinguishable by shape (round vs. square) and by label.
- Live regions (`aria-live="polite"`) announce sync status changes ("Changes saved") and validation hint updates without interrupting the user's flow.
- The right comment panel's open/close state is communicated via `aria-expanded` on the triggering button.

### Responsive design targets

- **Primary:** Laptop/desktop (1280px and up)
- **Secondary:** Tablet (768px and up) — teacher and admin interfaces
- **Full mobile support:** Student interface only (360px and up); sidebar becomes a drawer, back button appears in main area

Browser zoom up to 200% must not break layout or hide content.

---

*End of specification.*
