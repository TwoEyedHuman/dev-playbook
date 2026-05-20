# Dev Playbook

Skeleton READMEs and lessons learned across projects. Stack-agnostic where possible.

## Structure

```
dev-playbook/
├── README.md               ← this file
├── lessons-learned.md      ← append after every project/epic
└── skeletons/
    └── web-app/
        └── README.template.md
```

## How to Use

**Starting a new project:**
1. Copy the relevant skeleton: `cp skeletons/web-app/README.template.md ../my-new-project/README.md`
2. Find/replace `[Project Name]`, `[slug]`, `[service]`, etc.
3. Paste the README into a Claude chat session: *"Use this as the implementation plan. Start with Story 1.1."*

**After finishing an epic or project:**
1. Append lessons to `lessons-learned.md` (date + project + lesson + pattern)
2. Commit

**Rasterizing (every 2-3 projects):**
1. Open Claude chat
2. Paste `lessons-learned.md` + the current skeleton
3. Say: *"Regenerate the skeleton incorporating these lessons. Keep the same structure but improve the AC, assumptions, and integration gates."*
4. Review, commit as the new skeleton
