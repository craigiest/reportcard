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
- Admin section title
- User menu (profile, logout)

**Left Sidebar (Main Menu)**
- Users
- Terms & Settings
- Comment Banks
- Student Prompts
- Rosters & Import
- Report Cards
- Analytics
- Data Management

### Core Admin Functions

#### 1. User Management

**View & Search Users**
- Search by name, email, role
- Filter by role (Teacher, Advisor, Admin, Student)
- View user account status

**Create User**
- Input name, email, role
- Generate temporary password or send Google login instructions
- Assign role

**Manage User**
- Change role
- Reset password
- Spoof user (see interface as that user)
  - Use case: Add student reflection on behalf of student (for comments submitted outside system)
  - Use case: Mark student reflection as complete for missing submissions
  - Use case: Complete a report card section if teacher is unable

**Delete User**
- Archive user account (do not permanently delete due to audit trail)

#### 2. Term Management

**View Terms**
- List all school years
- For each year, show all configured terms with:
  - Term name
  - Order
  - Start/end dates
  - Locked status
  - Report card types (midterm, final, both)

**Create New Term**
- Input school year
- Input term name
- Input order in sequence
- Select reporting periods to include (Midterm and/or Final); both share the same roster
- Set student reflection freeze dates (if applicable)
- Validation rules are inherited automatically from the global Midterm or Final configuration; no per-term rule entry needed

**Edit Term**
- Modify term name, order, dates, structure
- Cannot edit past terms (if locked for data integrity)

**Validation Configuration**

Admin maintains two global validation configurations — one for Midterm reporting periods and one for Final reporting periods. These are set once and inherited by all terms automatically.

**Teacher comment requirements:**
- Minimum number of skill areas that must have at least one comment
- Minimum total comments required per report card
- Maximum comments allowed per skill
- Maximum character limit for custom comments
- Required skills (must have at least one comment)
- Rules set to zero are not enforced and not shown to teachers

**Student reflection requirements:**
- Minimum character count required before submission is enabled
- Maximum character count allowed for a reflection
- Rules set to zero are not enforced

**Lock Term**
- Makes term read-only for teachers and advisors
- Admin can still see and export
- Used before distributing final report cards

#### 3. Comment Bank Management (Standard Comment Banks)

**View Comment Banks**
- List all standard comment bank sets
- For each bank, show:
  - Bank name
  - Associated department/course level
  - Number of subskills included
  - Last modified date
  - Applied to: which departments/courses use this bank

**Create/Upload Comment Bank**
- Upload CSV file with columns: Transferable Skill, Skill Area, Subskill, Strength, Growth
- Or input comments manually through form
- System organizes comments into the skill hierarchy automatically

**Set School-wide Default Comment Bank**
- Select the primary comment bank that all courses will use by default
- This applies unless overridden at department or course level

**Customize Comment Banks by Department/Course**
- Override the school default for specific departments (e.g., "World Languages" uses a different bank)
- Override department selection for specific subdepartments or courses (e.g., "Spanish for Native Speakers" within World Languages)
- Override for individual courses if needed
- Each course uses exactly one standard comment bank
- More specific selections override broader defaults

**Manage Standard Comments**
- Edit comment text within a bank
- Add new subskill with strength/growth comments
- Delete subskills or comments
- View which courses are using each bank

**Note on Custom Comments**
- Individual teachers' custom comments are not managed here
- Teachers export/import their own custom comments for sharing with colleagues
- Custom comments are stored per teacher and not shared by default

#### 4. Student Prompts Configuration

**View Prompts**
- List all configured prompts by term/course/department
- Show prompt text and customization level

**Create Prompt**
- Select customization level: Universal, Department, or Course
- Input prompt text
- Assign to term(s)
- Set minimum and maximum character requirements for student submission

**Edit Prompt**
- Update prompt text
- Change customization level
- Update term assignment

**Delete Prompt**
- Remove prompt (affects future submissions only)

#### 5. Roster Management

**View Rosters**
- List all imported rosters by term
- Show classes, students, teacher assignments

**Import Rosters**
- CSV import:
  - Upload CSV file with format: `teacher,course,student`
  - Preview import showing number of rows/classes/students
  - Confirm and apply (with option to replace or merge)
  - Example:
    ```
    teacher,course,student
    "John Smith","Biology 101","Alice Johnson"
    "John Smith","Biology 101","Bob Davis"
    "Jane Doe","English 10","Charlie Brown"
    ```
- Blackbaud/Canvas integration (TBD: specifics for database manager)
  - Authenticate with external system
  - Select classes/students to sync
  - Schedule periodic sync or one-time import
  - View sync history

**Manage Rosters**
- Add/edit/delete students manually
- Add/edit/delete classes
- Reassign students to classes
- Reassign teachers to classes

**Export Rosters**
- Export class roster as CSV
- Export student list as CSV

#### 6. Report Card Status & Management

**View All Report Cards**
- Filter by term, class, teacher, status
- Status options: Draft, Finalized, Reviewed
- Show:
  - Student name
  - Class
  - Teacher
  - Current status
  - Whether advisor has reviewed
  - Export status (exported, not exported, newly reviewed)

**Flag Unreviewed Cards**
- Automatic flagging of report cards where advisor review is pending
- Option to exclude from bulk export
- Track which cards have been exported

**Resolve Issues**
- View report cards with validation errors
- Manually unfinalize if needed
- Add admin notes

#### 7. Report Card Export

**Bulk Export**
- Select export criteria:
  - By term
  - By class
  - By teacher
  - By status (include only reviewed, or all)
- Options for organization and file naming (TBD: specifics to be defined)
- System tracks what has been exported
- Option to export only newly-reviewed cards since last export
- Files generate as individual PDFs in a zip or as organized folder structure (TBD)

**Local Export**
- Teachers/advisors can export individual or small batch PDFs locally (same as now)

**Customize PDF**
- Upload/change school logo
- Edit header (school name, district name, etc.)
- Edit footer (address, contact, etc.)
- Edit boilerplate text (e.g., grading scale, competency descriptions)
- Preview PDF layout with customizations

#### 8. Analytics

Analytics is the default landing view for admin. It shows whether reporting is proceeding normally at a glance.

**Summary KPIs**
- Report Cards Completed (count and %, finalized or reviewed)
- Report Cards Reviewed (count and %)
- Student Reflections Submitted (count and %)

**Completion Breakdown**
- Summary card showing overall completion rate and any classes flagged below a threshold
- "View Breakdown" link opens a sortable table: Class, Teacher, Students, Completed, Reviewed, % Done
- Rows are expandable to show individual students with their report card status and whether a reflection was submitted
- Clicking a student name opens the impersonate (spoof) dialog for quick admin assistance

**Reflection Submission Breakdown**
- Summary card showing overall submission rate and any classes flagged below threshold
- "View Breakdown" link opens a sortable table: Class, Teacher, Students, Submitted, %
- Rows are expandable to individual students; clicking a name opens the spoof dialog

**Reports**
- Comment usage report (CSV): which standard comments were used, by how many teachers and classes
- Completion status report (CSV): report card and reflection status for all students in the term
- Teacher custom comment banks (CSV): export a specific teacher's personal comment bank for sharing
- Custom report: choose any combination of available data fields and generate a CSV export

#### 9. Data Management

**Export Old Data**
- Select data older than 5 years
- Generate export file (CSV or archives)
- Manual confirmation required before deletion
- Archive exported data (for records/compliance)

**View Data Integrity**
- Check for orphaned records
- Verify data consistency
- Generate audit report (if audit trail is implemented)

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
    - Selected standard comments (Array of comment references — subskill name + strength/growth type — resolved to text at display time from the bank, or snapshotted at finalization)
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
