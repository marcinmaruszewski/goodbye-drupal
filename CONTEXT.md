# Todo Workspace

This context describes people organizing Todos and visual Canvases within shared Workspaces. The glossary names product concepts independently of their technical implementation.

## Language

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

**Todo List**:
A named collection of Todos belonging to one Workspace.
_Avoid_: Project, board

**Todo**:
An item on a Todo List that can be completed.
_Avoid_: Task

**Canvas**:
A visual diagram belonging to one Workspace.
_Avoid_: Exploration, board
