Project type list and description:
* Team : Multiple developer can modify this project at the same time via git.
* Solo : Only one developer have modification for this project

Branching policy by type:
* Team : Create a branch per unit of work; merge to the default branch only on explicit user request (fast-forward when possible).
* Solo : Commit directly to the default branch. Create a branch only when the user explicitly requests one. The agent may suggest a branch for a large or risky change, but never branches on its own.
* Default (unknown) : Team (branch-maintained).

Here are configuration
<project_type>
Solo
</project_type>