# Todo Workspace

This context describes people organizing Todos and visual Canvases within shared Workspaces, together with the learning workflow used while building the product. The glossary names project concepts independently of their technical implementation.

## Product language

**User**:
A person with an application identity who may participate in many Workspaces.
_Avoid_: Member, account

**Workspace**:
The collaboration boundary containing Members, Todo Lists, and Canvases.
_Avoid_: Project, tenant

**Member**:
A User participating in a Workspace. Membership describes the User's relationship to that Workspace, not their global identity.
_Avoid_: User, account

**Role**:
A Member's authority within a Workspace.
_Avoid_: User type, access level

**Owner**:
A Member whose Role includes managing Members and the Workspace lifecycle. All Owners are peers, and an active Workspace always has at least one.
_Avoid_: Administrator, primary Owner, Workspace creator

**Invitation**:
A revocable, expiring offer from an Owner for the User who controls a specific verified email address to become a Member. An Invitation grants no Workspace access before acceptance.
_Avoid_: Access link, share link

**Todo List**:
A named collection of Todos belonging to one Workspace.
_Avoid_: Project, board

**Todo**:
An item on a Todo List that can be completed.
_Avoid_: Task

**Canvas**:
A visual diagram belonging to one Workspace.
_Avoid_: Exploration, board

## Learning language

**Learning Opportunity**:
A product decision or implementation change that introduces a new mental model, consequential boundary, costly decision, or unverified capability.
_Avoid_: Lesson candidate, learning trigger

**Learning Loop**:
A bounded learning cycle connecting retrieval, targeted explanation, authentic practice, feedback, and a recorded outcome to product work.
_Avoid_: Course unit, lesson

**Learning Evidence**:
An observable demonstration that a person can explain, evaluate, diagnose, or modify the concept being learned.
_Avoid_: Completion, attendance

**Learning Gate**:
A process constraint that pauses only the product work dependent on an unresolved Learning Opportunity until Learning Evidence exists.
_Avoid_: CI gate, course prerequisite

**Reference Note**:
Durable project-specific knowledge promoted from a Learning Loop for later reuse.
_Avoid_: Lesson transcript, generic documentation copy

**Personal Learning Record**:
A person's history of Learning Evidence, current proficiency, and scheduled retrieval for a topic.
_Avoid_: Private learning record, conversation transcript
