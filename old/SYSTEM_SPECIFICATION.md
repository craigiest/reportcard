# Report Card System - Complete System Specification

**Version:** 1.0  
**Date:** April 20, 2026  
**Status:** Specification for Development  

---

## Table of Contents

1. [System Overview](#system-overview)
2. [User Roles & Permissions](#user-roles--permissions)
3. [Data Organization](#data-organization)
4. [Teacher Interface](#teacher-interface)
5. [Student Interface](#student-interface)
6. [Advisor Interface](#advisor-interface)
7. [Admin Interface](#admin-interface)
8. [Data Model & Schema](#data-model--schema)
9. [Backend Architecture & Data Sync](#backend-architecture--data-sync)
10. [Integration Requirements](#integration-requirements)
11. [Accessibility & Design](#accessibility--design)
12. [Flagged Questions for Database Manager](#flagged-questions-for-database-manager)

---

## System Overview

The Report Card System is a school-wide digital platform for creating, reviewing, and managing student report cards. It streamlines the process from teacher input through advisor review to final PDF export and admin distribution.

### Core Capabilities

- **Teachers** draft report cards by selecting from standard comment banks and adding custom comments
- **Students** submit self-reflections that populate into their report cards
- **Advisors** review and edit report cards for their advisees across all teachers
- **Admin** manages configuration, imports/exports, and system-wide oversight

### Key Principles

- **Local-first design**: All changes feel instantaneous to users; syncing happens in the background
- **Role-based access control**: Universal roles across the school, but changeable per user
- **Flexible term structure**: Terms can be named, ordered, and structured as needed (e.g., midterm/final)
- **Configurable validation**: Minimum/maximum comment requirements, character limits, and required skills are defined in global configurations — one for Midterm reporting periods and one for Final reporting periods. These are set once and apply automatically; admin does not need to reconfigure them for each new term.
- **Conflict prevention**: Finalization workflow prevents simultaneous editing between teachers and advisors

---

## User Roles & Permissions

All users have universal roles across the school. A person can transition between roles over time (e.g., was a student, now a teacher). Roles are not strictly separate user types — they are a function of what a person is assigned to. A teacher with no advisees has no advisory functionality; an advisor with no classes has no teacher functionality; a person can hold both simultaneously.

### Teacher

- View and select assigned classes for a given term
- View roster of students in each class
- View student reflections (read-only once submitted)
- Edit/add comments for each skill using:
  - Standard comment bank (department or course-specific)
  - Custom comments (personal, not shared)
- Write narrative comments
- Finalize report card (locks from own editing and enables advisor access)
- Export their own students'/advisees' report cards as PDF (local download)
- View unfinalized report cards for their advisees (as advisor)

### Advisor

An advisor is any adult staff member (teacher, counselor, or other staff) assigned long-term to a group of students. Advisors do not need to be teachers; a teacher is not necessarily an advisor either.

- View all assigned advisees' report cards across all classes in a term
- Edit comments on any advisee's report card (if teacher has finalized)
- Mark report card as "reviewed" (after reviewing, only available when teacher's finalize box is checked)
- If teacher unchecks finalize after advisor has made edits, advisor receives notification
- Navigate between advisees using prev/next arrows; pressing an arrow loads the next or previous advisee's first class (no attempt to match the current class by name)
- View historical report cards for context across terms
- Export their advisees' report cards as PDF (local download)

### Student

- View enrolled classes for current term
- Draft and submit self-reflection for each class
- Revise reflection while in draft status
- View submitted reflection (read-only after submission)
- Cannot edit reflection after submission (unless admin unsubmits)

### Admin

- User management (create, assign/change roles, reset passwords)
- Spoof any user (for adding external comments, marking reflections complete)
- Create and manage terms (name, order, structure with midterm/final options)
- Configure comment banks (by department or course)
- Import/export rosters (CSV or via Blackbaud/Canvas integration)
- Set student prompts (with customization by term, course, or department)
- Configure global validation rules for Midterm and Final reporting periods (min/max comments, character limits, required skills); these apply automatically to all terms of that type
- View system-wide report card status
- Export report cards in bulk with options for organization (TBD: specific organization and naming conventions)
- Customize PDF branding (logo, header, footer, boilerplate text)
- View analytics (standard comment usage, custom comment banks)
- Lock terms for read-only access
- Search and manage users
- Export and trim data 5+ years old (manual process, not automatic)

---

## Authentication

All users log in using their school-issued Google account (Google OAuth 2.0). This applies to teachers, advisors, students, and admin users. There is no separate local password system.

Users are authenticated via their existing school Google accounts, which should match their identities in other school systems (Blackbaud, Canvas, etc.). This allows the system to automatically match users to their roles and enrollments from imported roster data.

On first login, user accounts are created automatically with the email address from their school Google account. Roles can be assigned or changed by admin users.

Admin users have the ability to spoof other users by assuming their login temporarily (for purposes like adding external student comments or completing incomplete submissions).

---

## Data Organization

### School Year & Terms

Data is organized hierarchically:
- **School Year**: e.g., 2025-2026
- **Term**: Flexible naming and ordering (e.g., S1, I1, S2, I2 OR T1, T2, T3 OR other custom naming)
- **Reporting Period**: Midterm and/or Final within each term — both share the same class roster

A term (e.g., S1) has one roster of students and classes. Within that term, admin configures whether teachers complete a Midterm report card, a Final report card, or both. In the grading period dropdown, teachers see these as separate selectable periods (e.g., "25-26 S1 Midterm" and "25-26 S1 Final"), but the underlying class roster is the same for both.

Admin configures term structure, including:
- Term names and order
- Whether the term includes a Midterm reporting period, a Final reporting period, or both
- Start and end dates
- Freeze dates for student reflections (if applicable)
- Whether the term is locked (read-only to teachers/advisors)

### Comment Banks

There are two types of comment banks: **Standard** (created and managed by admin) and **Custom** (created by individual teachers).

#### Standard Comment Banks

Admin can create and upload multiple standard comment bank sets. Each set is a CSV file with five columns representing the skill hierarchy and comment types:

**Standard Comment Bank CSV Format:**
```
Transferable Skill,Skill Area,Subskill,Strength,Growth
Self-awareness and Growth Mindset,Motivation,Curiosity-fueled motivation,"You're motivated by curiosity, asking ""why?"" and digging for understanding, rather than just checking steps off for completion.","Work on asking ""why"" more consistently rather than just completing tasks—push yourself to dig into the reasoning behind what you're learning, not just the steps."
Self-awareness and Growth Mindset,Motivation,Awareness of personal motivations,"You understand and notice your intrinsic and external motivations.","Spend time identifying what actually drives you to engage with schoolwork. Notice when you're acting from external pressure versus genuine interest, and try to cultivate more of the latter."
Critical Thinking,Question Asking,Asking complex or nuanced questions,"You ask questions that...","Work on asking questions that..."
```

**By default, the entire school uses a single standard comment bank.** However, customization is possible at department or course level when needed:

- **School-wide (default)**: All courses use the default comment bank
- **Department customization**: A specific department can opt to use a different comment bank instead of the school-wide default (e.g., World Languages uses a different bank)
- **Subdepartment customization**: A subset of courses within a department can use yet another bank (e.g., "Spanish for Native Speakers" uses a specialized bank while other languages use the department bank)
- **Individual course customization**: A specific course can optionally use a different comment bank

**Important**: Each course uses exactly one standard comment bank. The hierarchy works top-down: school default → department override → subdepartment override → individual course override. More specific selections override broader defaults.

When teachers select comments in the interface, they see them organized by:
1. Transferable Skill (top level)
2. Skill Area (grouping)
3. Subskill (the comment selector)

For each subskill, two toggle buttons are shown: **S** (Strength) and **G** (Growth). Clicking S or G directly selects that type of comment; clicking the same button again deselects it; clicking the other button switches to that type. Clicking the subskill label text cycles through states: none → Strength → Growth → none.

Only one comment per subskill can be selected at a time (either Strength or Growth, not both).

#### Custom Comments

Teachers create and maintain their own personal custom comments. These are exported and can be shared with colleagues for their own customization.

**Custom Comment CSV Format:**
```
skill_area,comment
Curiosity-fueled motivation,"Needs to ask more why questions"
Collaboration,"Excellent group project leadership"
Organization of materials,"Keep physical and digital materials better organized"
```

Teachers export their personal custom comment banks as CSV and can share with colleagues, who import them into their own personal comment collections. The import process allows merging with existing comments or replacing them entirely.

### Skills & Rubric

Skills are derived from the comment bank structure. Each skill can have:
- Multiple standard comments available for selection
- Character limits and minimum/maximum requirements (per term)
- Required or optional status (per term)

---

## Teacher Interface

### Layout & Navigation

**Left Sidebar (Master Area)**
- Teacher name
- Grading Period selector (dropdown)
- Navigation arrows (▲/▼) to step to the previous or next student across all classes
- Class roster (collapsible list)
  - Classes collapse/expand to show students; clicking a class header toggles its expand/collapse state only
  - Clicking a student in the list activates that student in the main content area
- Status legend at the bottom of the sidebar:
  - Round dot: report card status (not started, in-progress, complete, finalized)
  - Square dot: student reflection submitted

**Main Content Area**
- Empty state: "← Select a student to begin" if no student is selected
- Student editing area (when student is selected):
  - Student metadata (name, course, teacher, grading period)
  - Student reflection (if submitted; read-only)
  - Skills section:
    - For each transferable skill:
      - Skill name
      - "Skill Selection" button — opens the comment panel for that skill
      - Selected comments displayed as text below the button
  - Narrative textarea (for general comments)
  - Save hint: "Changes saved automatically"
  - Checkbox: "Mark as finalized" — active for teacher; visible but inactive for advisor
  - Checkbox: "Reviewed" — active for advisor (only when finalized); visible but inactive for teacher
  - The finalize checkbox is disabled with an inline hint until minimum requirements are met
  - Preview Report Card button (opens a preview modal; Print / Save PDF available from the preview)

**Right Sidebar (Comment Panel)**
- Opens when clicking the "Skill Selection" button for a transferable skill; panel header displays the skill name
- Shows:
  - Standard comments organized by Skill Area, with each Subskill shown as a row containing two toggle buttons — **S** (Strength) and **G** (Growth)
  - A quick-add input field for one-time comments: text entered here is saved only for the current student, not added to the reusable comment bank
  - An "Edit comment bank…" link that opens the custom comment bank manager for adding or removing reusable comments
  - Saved custom comments (personal to this teacher, reusable across students) shown as a checklist — checked comments are applied to the current student

### Core Workflows

#### Selecting a Term & Class

1. Teacher clicks grading period dropdown in master area
2. Available grading periods load (filtered to currently-assigned terms)
3. Roster updates to show classes for the selected grading period
4. If a student was previously active, the system first looks for that student in the same class (by class name) in the new grading period; if found there, activates them in that class
5. If the student is not in the same class, the system searches the rest of the roster for the student by name; if found in another class, activates them there
6. If the student is not found anywhere in the new grading period, or if no student was previously selected, show empty state

#### Drafting a Report Card

1. Teacher clicks a student in the roster
2. Student editing area loads with:
   - Student reflection (if submitted)
   - All skills with no comments yet selected
3. Teacher clicks on each skill to open comment panel
4. Teacher selects comments from standard bank or adds custom
5. Changes are synced to server (on blur, or periodically every few seconds)
6. If internet connection is lost, changes queue locally and sync when back online
7. "Changes saved automatically" hint confirms sync

#### Finalizing & Advisor Review

1. Teacher checks "Mark as finalized"
   - Report card is locked for teacher editing and becomes available for advisor review
   - Advisor can now access and edit the report card
2. Advisor reviews and edits the report card
3. Advisor checks "Mark as reviewed"
   - Indicates advisor has completed their review
4. If teacher unchecks finalize after advisor has made edits, teacher sees notification
   - Callout box: "This report card has been edited by [Advisor Name]. Unfinalize again to edit, or reload to see their changes?"
   - Teacher can reload to see advisor edits or unfinalize to make their own edits

#### Exporting (Local)

1. Teacher clicks "Preview Report Card" button
2. Preview modal opens showing the current student's report card
3. Teacher clicks "Print / Save PDF" to download the PDF to their computer

### Validation & Constraints

Validation rules are configured globally by admin — one configuration for Midterm reporting periods and one for Final reporting periods. These apply automatically to all terms; admin does not need to reconfigure them per term. The two configurations will typically differ (e.g., finals may require more skill areas than midterms).

Each configuration includes:
- Minimum number of skill areas that must have at least one comment
- Minimum total comments required per report card
- Maximum comments allowed per skill
- Maximum character limit for custom comments
- Required skills (must have at least one comment)

Any rule set to zero is not enforced and is not surfaced in the teacher interface.

If a report card doesn't meet validation criteria, the "Mark as finalized" checkbox is disabled with an inline hint listing the specific unmet requirements (e.g., "selections in 2 more skill areas needed; overall comment needs 15 more characters").

---

## Student Interface

### Layout & Navigation

**Top Navigation**
- School name/logo
- "My Reflections" title
- User name
- Logout button

**Left Sidebar**
- List of enrolled classes for current term
- Each class shows course name and teacher name
- Clicking a class loads that reflection

**Main Content Area**
- Empty state: "← Select a class to begin" if no class selected
- Reflection editing area (when class is selected):
  - Class name and teacher name
  - Admin-determined prompt (varies by course/department/universal as configured)
  - Large textarea for student reflection
  - Submit button (prominent, clear state)
  - Status indicator (Draft / Submitted)
  - For submitted reflections: timestamp of submission and note "You cannot edit after submitting. Contact your teacher if you need to make changes."

### Core Workflows

#### Drafting a Reflection

1. Student logs in with Google account
2. Sees list of enrolled classes (auto-populated from imported roster)
3. Clicks a class
4. Sees the admin-configured prompt
5. Types reflection in textarea
6. As they type, changes are saved locally and synced periodically
7. Status shows "Draft"; Submit button is visible but disabled
8. Submit button becomes active once the reflection meets the minimum character count set by admin

#### Submitting a Reflection

1. Once reflection meets minimum requirements, "Submit" button becomes active/prominent
2. Student clicks "Submit"
3. Confirmation dialog: "Once submitted, you won't be able to edit this reflection. Are you sure?"
4. Upon confirmation, reflection is locked
5. Status changes to "Submitted [timestamp]"
6. Textarea becomes read-only
7. Message appears: "Your reflection has been submitted and is now visible to your teachers."

#### Revising a Draft

1. Student can freely edit a draft reflection before submitting
2. Changes sync in background
3. "Draft" status persists until submitted

### Accessibility

- Mobile-friendly (full functionality on phone)
- Responsive textarea (grows with content)
- Clear visual hierarchy for prompt and input

---

## Advisor Interface

### Layout & Navigation

**Identical to Teacher Interface** with key differences:

**Left Sidebar**
- Advisor name
- Grading Period selector (dropdown)
- Advisee list (instead of class roster)
  - Clicking an advisee shows their report cards organized by class
  - Selecting a specific class/report card activates it

**Main Content Area**
- Identical to teacher interface; both "Mark as finalized" and "Reviewed" checkboxes are visible, with active/inactive states reversed: advisor can check "Reviewed" (when finalized); "Mark as finalized" is visible but inactive for advisor
- Advisor can edit comments and narrative only on finalized report cards
- For unfinalized report cards, the content area is read-only with a note: "Awaiting teacher finalization"

**Right Sidebar**
- Comment panel (identical to teacher)

### Core Workflows

#### Viewing Advisees' Report Cards

1. Advisor selects term from dropdown
2. Advisee list loads showing assigned advisees
3. Clicking an advisee shows their classes (from all teachers)
4. Clicking a class shows that advisee's report card from that teacher
5. If no report card exists yet, shows "Not started"
6. If report card is not finalized, shows read-only view with note: "Awaiting teacher finalization"
7. If report card is finalized, advisor can view and edit

#### Editing a Report Card

1. Report card must have finalize checkbox checked (locked by teacher)
2. Advisor can freely edit:
   - Selected comments
   - Custom comments
   - Narrative field
3. Advisor CANNOT unfinalize a report card; the "Mark as finalized" checkbox is visible but inactive
4. Advisor can check "Reviewed" checkbox (only when finalized, to indicate they've completed their review); teacher sees this checkbox too but it is inactive for them
5. Changes sync with same local-first + periodic sync strategy as teacher

#### Conflict Handling

1. If teacher unchecks finalize after advisor has made edits:
   - Teacher sees callout box: "This report card has been edited by [Advisor Name] since you finalized it. Unfinalize again to edit, or reload to see their changes."
   - Page does NOT automatically reload; teacher must acknowledge and reload manually
   - Upon reload, teacher sees advisor edits and can decide to finalize again or make changes
2. If teacher unfinalizes while an advisor is actively viewing the report card:
   - The advisor's editing access is revoked on the next sync event (blur or save attempt)
   - Advisor sees a notification that the report card has been unfinalized by the teacher and is no longer editable

#### Jumping Between Advisees

1. Advisor presses next/previous arrow or clicks a different advisee in the list
2. System loads the selected advisee's report cards for the current grading period
3. Opens the first class in that advisee's list
4. When switching back to a previously viewed advisee, the system returns to the last class the advisor was viewing for that advisee

#### Historical Context

1. Advisor can switch to a different term while staying on the same advisee
2. For example: Viewing Advisee A in S1, switches to S2
3. System loads Advisee A's S2 report cards
4. Advisor can view previous term's report cards for context (read-only)
5. Current term's cards are editable if finalized; previous terms' are always read-only

---

## Admin Interface

### Navigation

**Top Navigation**
- School name/logo
- "Admin" badge (distinguishes from teacher/student interfaces)
- Current user name (top right)
- Log out button

**Left Sidebar (Main Menu)**
- Terms & Rosters
- Report Cards
- Analytics (default view)
- Users
- Comment Banks
- Student Reflection Prompts
- Assignments
- Settings
- Data Management

### Core Admin Functions

#### 1. User Management

**Search and filter**: Live search by name or email; role filter (All / Teacher / Advisor / Admin / Student); status filter (Active & Archived / Active Only / Archived Only).

Table columns: Name, Email, Roles (colored tags — multiple roles displayed simultaneously), Status (Active/Archived), Last Login, Actions.

**Actions per user row:**
- **Edit**: opens the User modal pre-filled with the user's current data
- **Spoof**: opens a confirmation modal explaining that the admin will temporarily assume the user's identity. Upon confirming ("Enter as User"), the admin's session switches to the target user's full interface. A persistent banner is shown at the top of the screen indicating that the admin is currently spoofing, with an option to exit and return to the admin view. The spoof session is recorded in the Audit Log.
- **Archive**: opens a confirmation modal. Archiving deactivates the user account — they can no longer log in — but all their data and history are preserved. Archived users can be reactivated at any time by editing their account. Archived users are not permanently deleted.

**New/Edit User modal fields:**
- Full Name
- School Email — must match the user's school-issued Google account (the one they use for OAuth login); a hint confirms this. The email address is the identity anchor and cannot be changed after account creation.
- Roles: checkboxes for Teacher, Advisor, Admin, Student. Multiple roles can be assigned simultaneously. Role changes take effect immediately on the user's next page load.

#### 2. Settings (Validation Rules)

Admin maintains two global validation configurations — one for Midterm reporting periods and one for Final reporting periods. These apply automatically to all terms of that type; no per-term configuration is needed. The two configurations are displayed as side-by-side cards (Midterm and Final), each with its own Save button. Rules are grouped to match the order of the report card.

Setting any numeric value to **0** disables that rule; disabled rules are not surfaced in the teacher interface. Leave the required skills field blank to skip that rule.

**Student Reflection**
- Minimum characters required before the submit button becomes active
- Maximum characters allowed (0 = no limit)

**Skill Comments**
- Minimum number of skill areas that must have at least one comment selected
- Required skills: a comma-separated list of skill names; each listed skill must have at least one comment before the teacher can finalize
- Minimum total skill comments per report card
- Maximum comments per skill area
- Maximum character limit for custom skill comments (free text a teacher writes in the skill panel — either a one-time note for that student or a saved personal comment — as opposed to a selection from the standard bank)

**Narrative**
- Minimum characters (default: 15)
- Maximum characters (0 = no limit)

#### 3. Terms & Rosters

This section has three sub-tabs: **Terms**, **Class Sections**, and **Import**.

**Terms sub-tab**

Displays all school years with their terms grouped by year. Each term row shows: term name, date range, reporting period tags (Midterm, Final, or both), locked status, and action buttons.

- Locked terms display a "Locked" badge; their Edit button is present but disabled (grayed out)
- Unlocked terms show Edit and Lock buttons

**Create Term** (modal):
- School year (text, e.g., "2025-2026")
- Term name (e.g., "S1", "T2", "Q3") and Order (integer — position within the school year)
- Start date and End date
- Reporting periods: Midterm and/or Final checkboxes (both enabled by default); both periods share the same class roster
- Student Reflection Freeze Date (optional): students cannot edit their reflections after this date
- Validation rules are inherited automatically from global Midterm/Final config; no per-term configuration is needed

**Edit Term** (same modal, pre-filled; locked terms cannot be edited):

**Lock Term**: makes the term read-only for all teachers and advisors. Admin can still view and export. Used before distributing final report cards. Locking is intended to be permanent once report cards are distributed — there is no UI to unlock a term.

**Class Sections sub-tab**

Displays class sections for the selected term, filterable by department. Table columns: Course, Section, Teacher, Department, Students (count).

- Clicking a course name opens a **Students modal** showing the enrolled student list with a search field and a "+ Add Student" button; admin can add or remove individual students from within this modal
- Each section row has Edit and Remove action buttons

**Add Class** modal (also accessible from this sub-tab):
- Term selector
- Course Name
- Department selector
- Teacher selector

**Import sub-tab**

Two panels displayed side by side:

*Import from CSV:*
- File drop zone; expected columns: `teacher, course, student`
- After file selection, an import preview appears showing counts (teachers, classes, students, total rows) and a sample of the first rows from the file
- Import mode: "Merge with existing roster" (adds to current data) or "Replace existing roster for this term" (overwrites)
- Cancel and Apply Import buttons

*Export Roster:*
- Term selector
- Format: Class roster (`teacher, course, student`) or Student list only
- Download CSV button

**Blackbaud/Canvas integration** (TBD: specifics for database manager):
- Authenticate with external system
- Select classes/students to sync
- Schedule periodic sync or one-time import
- View sync history

#### 4. Comment Bank Management (Standard Comment Banks)

This section manages the standard comment bank files. Assigning banks to specific departments and courses is done in the **Assignments** section.

**School-wide default bar** (above the bank list): a dropdown selects which uploaded bank is the school-wide default. A "Set Default" button saves the selection. A note explains that all departments and courses use this bank unless overridden in Assignments.

**Bank list table** columns: Bank Name (with a "School Default" badge on the current default), Subskills (count of subskill rows in the bank), Last Modified, Applied To (descriptive summary of where this bank is currently assigned).

**Actions per bank:**
- **Replace** (⬆): opens the Upload modal with the bank name pre-filled and read-only; uploading a new CSV replaces all comments in that bank entirely. The submit button reads "Upload & Replace".
- **⬇ CSV**: downloads the bank in the standard five-column CSV format (Transferable Skill, Skill Area, Subskill, Strength, Growth).
- **Delete**: removes the bank. A bank cannot be deleted if it is currently assigned to any department or course in any term; those assignments must be changed first. The school-wide default bank cannot be deleted.

**Upload New Bank** button opens a modal:
- Bank Name (required; must be unique)
- File drop zone: click to select or drag-and-drop a CSV file
- A format reference shows the expected column order and an example data row
- The submit button reads "Upload & Preview": the system parses the CSV and shows a preview of the parsed bank before it is created. Admin can cancel from the preview.

**Note on Custom Comments**
- Individual teachers' custom comments are not managed here
- Teachers export/import their own custom comments for sharing with colleagues
- Custom comments are stored per teacher and not shared by default

#### 5. Student Reflection Prompts Configuration

Prompts are authored here and set as defaults at the Universal or Department level. Assignments to specific departments and courses are managed in the **Assignments** section.

**View Prompts**

Prompts are displayed as cards, filterable by term and level (Universal / Department). Each card shows:
- Prompt name and scope tag (e.g., "Universal" or "Department: World Languages")
- Terms the prompt is active in
- A preview of the prompt text
- Min and max character requirements
- Edit and Delete action buttons

**Create / Edit Prompt** (modal):
- **Default For** (radio buttons):
  - "All classes (Universal)": this prompt is the school-wide default in the selected terms
  - "A specific department": a department dropdown appears; this prompt becomes the department-level default for the selected terms
  - Hint: "To assign a prompt to individual departments or courses, use the Assignments section."
- **Terms**: checkboxes for available terms, plus an "All future terms" option
- **Prompt Text**: the question or instruction shown to the student when writing their reflection
- **Min Characters** and **Max Characters** (0 = not enforced)

When saved, a Universal prompt becomes the active default for all classes in the selected terms that have no more specific assignment. A Department prompt becomes the default for that department in the selected terms. These defaults are reflected automatically in the Assignments section.

**Delete Prompt**: if the prompt is currently the active assignment for any class or department (as shown in Assignments), those assignments revert to the next broader default automatically. The admin should verify assignments after deleting a prompt.

#### 6. Assignments

The Assignments section provides a single, unified view for managing which comment bank and which reflection prompt applies to each department and course. All assignments are stored per term — changes to one term do not affect any other.

**Term selector** (top of section): changing the term reloads the assignment table for that term. When a new term is created, it automatically inherits all assignments from the most recent preceding term; the admin can adjust for the new term without affecting past terms.

**Assignment table structure**

Columns: Scope, Comment Bank, Student Prompt, and an action column.

Rows are organized in a three-level hierarchy:

**School Default row** (always first): dropdowns select the school-wide default bank and prompt. An "Apply to all depts & courses" button resets all overrides in the current term's table, reverting every department and course to inheriting from the school default.

**Department rows** (one per department, expandable): clicking a department row toggles its courses visible/hidden. Each department row shows its current bank and prompt assignment in one of two states:
- **Inherited**: the selector is grayed out and displays the currently inherited value; a small badge (e.g., "school") identifies the source; an "Override" link is shown. Clicking "Override" activates the selector for that department.
- **Overridden**: the selector is active with the chosen value; a "× Reset" link is shown. Clicking "× Reset" reverts to inheriting from the school default.
- An "Apply to all courses" button within each department row resets all course-level overrides for that department (courses revert to inheriting from the department).

**Course rows** (shown indented under the department when expanded): same inherited/overridden pattern, but inheriting from the department (not the school). The inherited badge shows "dept".

**Effective value resolution**: the value applied to a course is the most specific override that exists:
1. Course-level override (if set)
2. Department-level override (if set)
3. School default

Individual cell changes save immediately; there is no separate Save step for the table.

#### 7. Report Cards

The Report Cards section has two sub-tabs: **Status & Export** and **PDF Settings**.

**Status & Export sub-tab**

Filters (applied simultaneously): Term, Reporting Period (All / Midterm / Final), Teacher, Status (All / Draft / Finalized / Reviewed), Export Status (Any / Not Exported / Exported / Newly Reviewed).

A stat strip below the filters shows counts for the current filter: Total, Draft, Finalized, Reviewed, Exported.

Table columns: Student, Class, Teacher, Period, Status (color-coded pill badge), Advisor Reviewed (checkmark), Exported (checkmark), Actions.

Actions per row:
- **Unfinalize** (appears only on Finalized cards): reverts the report card to Draft, removing the teacher's finalize lock and restoring their editing access. Any advisor edits made after finalization are preserved in the draft; the teacher will see the standard conflict notification when they next open the card.
- **Note** (appears on all cards): opens a modal for the admin to write or edit an admin note attached to that report card. Notes are not visible to teachers, advisors, or students — they are internal admin records only.

**Bulk Export** button (top-right of the sub-panel) opens a modal:
- Term + Reporting Period selector (combined, e.g., "25–26 S1 — Final")
- Include: Reviewed cards only / Newly reviewed since last export / All finalized cards
- Optional filter by Teacher
- Optional filter by Class
- A count preview updates live to show how many cards match the current criteria (e.g., "41 reviewed cards match current criteria. Export will be tracked.")
- The Export button is labeled with the count (e.g., "Export 41 PDFs")
- After export, all included report cards are marked as Exported; this is reflected in the Exported column and the Export Status filter

**PDF Settings sub-tab**

*Branding & Logo:*
- Upload school logo (PNG or SVG; recommended 200×60px; max 1MB)
- School Name (appears in the PDF header)
- District / Subtitle (optional second header line)

*Footer:*
- Footer Text: appears at the bottom of every page of every exported PDF

*Boilerplate Text:*
- Introductory paragraph: shown at the top of each report card body, before skill comments
- Competency / grading scale note: explains the S/G designation system or any grading key

*Preview:*
- A thumbnail preview shows how the header and footer will appear; an "Open Preview" button opens a full-size layout preview

A single "Save PDF Settings" button persists all branding fields together.

#### 8. Analytics

Analytics is the default landing view when admin logs in.

**Term + Period selector** (top right): selects a specific term and reporting period (e.g., "25–26 S2 — Final") for all data shown on the page.

**KPI cards** (three, across the top):
- Report Cards Completed: count and % that are finalized or reviewed out of total report cards in that term/period
- Report Cards Reviewed: count and % that advisors have marked as reviewed
- Student Reflections Submitted: count and % of enrolled students who have submitted a reflection

**Summary cards** (two, side by side):

*Report Card Completion by Class:*
- A one-sentence summary of how many classes are on-track (e.g., "5 of 7 classes are ≥ 75% completed")
- A progress bar showing overall completion rate
- "View Breakdown →" button — opens the Completion Breakdown modal

*Student Reflection Submission by Class:*
- A one-sentence summary (e.g., "2 of 7 classes at 100% submitted")
- A progress bar
- "View Breakdown →" button — opens the Reflection Breakdown modal

**Completion Breakdown modal**: a sortable table — Class, Teacher, Students, Completed, Reviewed, % Done. Clicking a class row expands it to show individual students with their report card status and whether a reflection was submitted. Clicking a student name opens the Spoof confirmation dialog for immediate admin access. The modal footer includes an "⬇ Export Table" button that downloads the table as CSV.

**Reflection Breakdown modal**: same structure. Columns: Class, Teacher, Students, Submitted, %. Same expand/spoof/export behavior.

**Reports panel** (below summary cards):
- Comment usage report (CSV): which standard comments were used, by how many teachers and classes
- Completion status report (CSV): report card and reflection status for all students in the selected term
- Teacher custom comment banks (CSV): a teacher selector plus download button — exports that teacher's personal comment bank as CSV (for sharing between colleagues)
- "Custom Report…" button opens a modal with:
  - Term selector (specific term+period, or all terms)
  - Column checkboxes: Student name, Class, Teacher, Report card status, Reflection status, Advisor reviewed, Finalized date, Reviewed date, Last exported, Comment count, Narrative character count, Reflection character count
  - "Generate CSV" downloads the report with selected columns

#### 9. Data Management

**Export Data by Year**

- School year dropdown: select any available year
- Records in scope preview: shows counts for the selected year (terms, report cards, student reflections, export records)
- Format: CSV files (one file per data type) or ZIP archive with all files
- "⬇ Export Year Data" button opens a confirmation modal summarizing the record counts; confirming starts the download. The data remains in the live system after download — nothing is deleted automatically.

*Delete Year Records* (within the same panel, in a danger zone):
- Permanently removes all records for the selected year from the live system
- Admin must type "DELETE" in a confirmation input to enable the delete button
- This cannot be undone; admin should verify the exported archive is complete and stored securely before proceeding
- Intended for data hygiene once records are no longer needed in the live system (typically after 5+ years)

**Restore from Backup**

- Uploads a previously exported backup (ZIP or CSV) to restore data for a selected school year
- School year selector and file drop zone
- Admin must type "RESTORE" in a confirmation input to enable the restore button
- Restoring replaces all current records for the selected year with the contents of the backup file
- Intended for system recovery after data loss; not for routine use

**Data Integrity**

- "Run Check" button scans for orphaned records, broken foreign-key relationships, and data inconsistencies
- Results appear inline with pass / warn / fail indicators for each check category

**Audit Log**

- A persistent, automatically maintained log of significant admin actions
- Events captured: role changes, user archiving and reactivation, spoof sessions, term locks, bulk PDF exports, comment bank uploads and replacements, term creation, data exports, and data deletions
- Displayed as a scrollable table: Date, Admin, Action (descriptive text)
- "⬇ Export Full Audit Log" button downloads the complete log as CSV

---

## Data Model & Schema

### Entities

#### User
- ID
- Email
- Name
- Roles (Array: can be Teacher, Advisor, Admin, Student)
- Active/Archived status
- Password hash (or Google OAuth token)
- Created date
- Last login date

#### Term
- ID
- School year (e.g., "2025-2026")
- Term name (e.g., "S1", "T1")
- Order (sequence within year)
- Start date
- End date
- Locked (boolean)
- Reporting periods (Array: Midterm, Final — which are enabled for this term)
- Student reflection freeze date (if applicable)
- Validation rules: inherited from global Midterm or Final configuration; not stored per term

#### Class
- ID
- Term ID
- Teacher ID
- Course name
- Course code
- Student roster (Array of student IDs)
- Grade level
- Department

#### Student Enrollment
- ID
- Student ID
- Class ID
- Term ID

#### Comment Bank
- ID
- Name
- Level (School-wide default, Department, or Course)
- Parent bank ID (for department or course overrides)
- Skills (Array of skill objects, derived from the bank's CSV or manual entries):
  - Transferable Skill name
  - Skill Area name
  - Subskill name
  - Strength comment text
  - Growth comment text

The school-wide default comment bank is loaded from a separate default file. Additional banks are created by admin upload or manual entry. Each bank owns its full skill hierarchy — transferable skills, skill areas, and subskills are not shared entities across banks.

#### Student Prompt
- ID
- Term ID (or null for universal)
- Course ID (or null for universal)
- Department ID (or null for universal)
- Prompt text
- Min character requirement

#### Report Card
- ID
- Term ID
- Student ID
- Class ID (teacher's class)
- Reporting period (Midterm or Final)
- Status (Draft, Finalized)
- Teacher ID
- Advisor ID (if reviewed by advisor)
- Finalized by (user ID)
- Finalized timestamp
- Reviewed by advisor (boolean)
- Reviewed timestamp (advisor)
- Report card data:
  - Skills:
    - Skill name (stored as text snapshot, not a foreign key, so bank edits don't alter existing cards)
    - Selected standard comments (Array of text snapshots — comment text is copied from the bank at finalization; report cards are fully self-contained after finalization and do not depend on the comment bank)
    - One-time custom comments (Array of text strings added for this student only; stored directly as text in the database)
    - Saved custom comments applied (Array of text strings from the teacher's reusable bank, also stored as text snapshots)
  - Narrative field (text)
  - Student reflection ID (reference to submitted reflection)

#### Student Reflection
- ID
- Student ID
- Class ID
- Term ID
- Prompt ID (reference to which prompt was shown)
- Draft text
- Submitted text (null if still draft)
- Status (Draft or Submitted)
- Submitted timestamp
- Student name (snapshot at time of submission)
- Teacher ID (snapshot at time of creation)

#### Export Record
- ID
- Export date
- Export type (Bulk, Individual)
- Report cards included (Array of report card IDs)
- Export format (PDF, CSV)
- Admin user (who triggered export)
- Export file location (for tracking)

#### User Role History
- ID
- User ID
- Old role
- New role
- Change date
- Changed by (user ID)

### Data Relationships

```
User (1) → (many) Class (as teacher)
User (1) → (many) Report Card (as advisor)
User (1) → (many) Student Reflection (as student)

Term (1) → (many) Class
Term (1) → (many) Report Card
Term (1) → (many) Student Prompt

Class (1) → (many) Student Enrollment
Class (1) → (many) Report Card
Class (1) → (many) Student Reflection

Student (User) (1) → (many) Report Card
Student (User) (1) → (many) Student Reflection
Student (User) (1) → (many) Student Enrollment

Comment Bank (1) → (many) Skills (embedded within the bank)

Report Card (1) → (many) comment selections and text strings
Report Card (1) → (1) Student Reflection

Student Reflection (1) → (1) Student Prompt
```

---

## Backend Architecture & Data Sync

### Authentication

**[FLAG FOR DATABASE MANAGER]** Specify:
- Google OAuth 2.0 integration flow
- Token storage and refresh strategy
- Session management (duration, renewal)
- Local storage of auth tokens for offline capability

### Data Sync Strategy

#### Philosophy

- **Local-first**: All changes feel instantaneous; users don't wait for server
- **Eventual consistency**: Server updates happen in background; conflicts prevented by workflow
- **Offline resilience**: Changes survive page refresh and network interruptions

#### Sync Flow

1. **User makes change** (e.g., selects a comment)
   - Change is stored locally (localStorage or IndexedDB)
   - UI updates immediately
   - Change queued for sync

2. **Periodic sync** (every 3-5 seconds, or on blur)
   - Queued changes sent to server
   - Server validates and persists
   - Server returns confirmation with timestamp
   - Timestamp stored locally to track sync status

3. **Page refresh/offline**
   - On load, fetch latest from server (if online)
   - Merge server state with any unsynced local changes
   - Conflict resolution (see below)

4. **Coming back online**
   - Detect connection restoration
   - Sync any queued changes
   - Fetch latest server state to verify

#### Conflict Resolution

**Teacher & Advisor Editing Conflict:**

1. Teacher finalizes report card → Advisor can now edit
2. Advisor makes changes and syncs to server
3. Teacher unchecks finalize box → Wants to make changes
4. Before allowing teacher edits, check if advisor changes exist
5. If yes: Show callout box: "This report card has been edited by [Advisor Name] since you finalized it. Reload to see their changes?"
6. Teacher reloads → Sees advisor edits
7. Teacher decides: Keep as-is or unfinalize and make their own edits

**Student Reflection Conflict:**

Not applicable. Once submitted, students cannot edit.

#### Synced Data

On each sync:
- Report card changes (selected/deselected comments, narrative edits)
- Custom comments added to comment bank
- Any status changes (finalized, reviewed)
- Timestamps of changes

**[FLAG FOR DATABASE MANAGER]** Specify:
- Sync endpoint design (REST, GraphQL, etc.)
- Data format for sync payloads
- Error handling and retry logic
- Rate limiting and quota management

### Real-time Notifications

**Notification**: Teacher gets notified when they try to unfinalize a report card that the advisor has edited since finalization

**[FLAG FOR DATABASE MANAGER]** Specify:
- WebSocket vs polling vs long-polling for notifications
- Notification persistence (if user is offline)
- How notifications integrate with the page load conflict check

### Local Storage

**Data stored locally (survives refresh):**
- Current user info
- Current term/class/student selection
- Draft report card edits (unsynced)
- Draft student reflections (unsynced)
- Last sync timestamps
- Auth token (if applicable)

**Data NOT stored locally:**
- Sensitive user data (passwords)
- Full roster/comment bank (too large; fetch on demand)

### Performance & Scale

**[FLAG FOR DATABASE MANAGER]** Specify:
- Expected number of concurrent users
- Database indexing strategy for queries
- Caching strategy (server-side caching, CDN, etc.)
- Load testing requirements

---

## Integration Requirements

### Blackbaud Integration

**Purpose**: Import student/class roster data

**[FLAG FOR DATABASE MANAGER]** Specify:
- API endpoint and authentication
- Data extraction: student names, IDs, grade levels, class assignments, teacher names
- Frequency of sync (one-time import, periodic sync, real-time updates)
- Handling of data discrepancies (student moved, removed, etc.)
- Error handling for failed syncs

### Canvas Integration

**Purpose**: Supplement Blackbaud with LMS enrollment data

**[FLAG FOR DATABASE MANAGER]** Specify:
- API endpoint and authentication
- Data extraction: course enrollments, current active courses
- Sync frequency
- Conflict resolution (if student enrolled in Canvas but not Blackbaud)

### Google OAuth

**Purpose**: User authentication

**[FLAG FOR DATABASE MANAGER]** Specify:
- OAuth scope (email, profile, etc.)
- Consent flow UI
- User account creation on first login
- Account linking if user has existing local account

---

## Accessibility & Design

### Responsive Design

- **Primary target**: Laptop/desktop (1920x1080 and up, down to 1280x720)
- **Secondary target**: Tablet (iPad-sized, 768px and up)
- **Student interface**: Fully mobile-responsive (360px and up)

### Accessibility Standards

All interfaces must be:
- **Readable**: High contrast, legible fonts (minimum 14px body, larger for inputs)
- **Zoomable**: Support browser zoom up to 200% without layout breaking
- **Text-to-speech compatible**: Semantic HTML, proper ARIA labels
- **Keyboard navigable**: All functionality accessible via Tab, Enter, Space, Arrow keys
  - No keyboard traps
  - Logical tab order
  - Clear focus indicators (minimum 2px outline)
- **Screen reader tested**: NVDA, JAWS, VoiceOver compatibility

### Design Patterns

- **Modals**: Use for important confirmations (e.g., submit reflection, finalize report card)
- **Callout boxes**: Use for notifications and conflicts (e.g., advisor edits notification)
- **Tooltips**: Provide context for unfamiliar terms
- **Error messages**: Clear, specific, actionable
- **Disabled states**: Clear visual indication (grayed out, with explanation)

### Consistent UI Elements

- Buttons: Clear labeling, consistent sizing, hover/focus states
- Inputs: Clear labels, placeholder text, validation feedback
- Dropdowns: Clear options, no surprise defaults
- Checkboxes: Large hit targets (minimum 44px), clear labels
- Textareas: Auto-grow, character count if applicable

---

## Flagged Questions for Database Manager

### Backend Architecture

1. **Data synchronization endpoint design**: Should we use REST (PUT /report-card/:id) or GraphQL mutations? What is the optimal data format for sync payloads?

2. **Conflict detection & resolution**: How should we detect when an advisor has edited a report card that a teacher is trying to unfinalize? Should this be checked via a version number, timestamp, or change log?

3. **Real-time notifications**: Should we use WebSocket for real-time advisor edit notifications, or is polling acceptable? If polling, what interval?

4. **Authentication token management**: How should Google OAuth tokens be stored and refreshed? What is the session duration?

5. **Database schema for comment banks**: How should hierarchical comment banks (department → course → skill) be modeled to support efficient queries and edits?

6. **Audit trail** (optional): If audit trail is desired, what level of detail should be captured? (e.g., only finalization/review timestamps, or all edits?)

### Integration

7. **Blackbaud API**: What data do we need to extract, and what is the recommended sync frequency? Should this be scheduled or on-demand?

8. **Canvas API**: How should Canvas enrollment data supplement Blackbaud? What is the sync frequency?

9. **File storage**: Where should exported PDFs be stored (local server, cloud storage, etc.)? How long should exports be retained?

### Performance & Scalability

10. **Database indexing**: What queries will be most frequent, and how should we index the database to optimize performance?

11. **Caching strategy**: Should we implement server-side caching for comment banks or rosters? What about client-side caching?

12. **Expected load**: How many concurrent users, report cards, and comments should the system handle?

### Data Management

13. **Data retention & archiving**: How should we handle data exports and archiving for compliance? What is the retention policy beyond 5 years?

14. **Soft vs hard deletion**: Should deleted users/terms/classes be archived or permanently removed?

### Security

15. **Data encryption**: Should sensitive data (student names, teacher comments) be encrypted at rest or in transit?

16. **Permission validation**: Should permission checks happen on the client or server? (Server-side is more secure.)

---

## Summary of Changes from MVP (mockup-3)

### What Stays the Same

- Teacher workflow: select term → select class → select student → edit comments → finalize
- Comment bank structure (standard + custom comments)
- PDF export functionality
- UI layout (left sidebar, main content, right comment panel)

### What Changes

1. **No term/roster management by teachers**: Teachers select from pre-configured terms and rosters
2. **Standard comment banks read-only to teachers**: Maintained by admin instead
3. **Data syncing**: All data pushed to server (with local-first UX)
4. **Advisor review workflow**: New "Reviewed" checkbox; advisor can edit before review
5. **Student reflection input**: New student interface instead of pre-loaded data
6. **Admin dashboard**: New comprehensive admin interface for configuration and management

### New Capabilities

1. Student self-reflection submission
2. Advisor review workflow
3. Admin configuration and analytics
4. Blackbaud/Canvas integration
5. Bulk PDF export with tracking
6. User management and spoofing
7. Data archiving and export

---

## Appendix: Glossary

- **Finalize**: Teacher locks a report card, enabling advisor access and preventing further teacher edits unless unfinalizing
- **Reviewed**: Advisor marks a report card as reviewed (only available when teacher has finalized)
- **Spoof**: Admin temporarily assumes the identity of another user
- **Local-first**: Design pattern where all changes feel instantaneous locally; syncing happens in background
- **Comment Bank**: Collection of standard comments available for selection, organized by department/course
- **Custom Comments**: Teacher-created comments not in the standard bank
- **Term**: A reporting period (e.g., semester, trimester) within a school year
- **Report Card Type**: Midterm or Final report within a term
- **Prompt**: Admin-configured text guidance for student reflections
- **Advisor**: Any adult staff member (teacher, counselor, or other staff) assigned long-term to a group of students for the purpose of reviewing their report cards

---

**Document prepared for development**: Database manager and development team should address all flagged questions before implementation.
