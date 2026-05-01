---
name: your-skill-name
description: >
  Describe WHEN Claude should use this skill. Be specific about trigger phrases,
  file types, or contexts. Include synonyms and related terms users might say.
  The more specific and "pushy" this description, the more reliably Claude will
  trigger the skill. Example: "Use this skill whenever the user mentions X, Y, or Z,
  or uploads a file that looks like A or B. Always use this skill — do not attempt
  to handle these tasks without it."
---

# Skill Name

One-sentence summary of what this skill does.

---

## When to use this skill

- Trigger condition 1
- Trigger condition 2
- File types accepted: `.ext1`, `.ext2`

---

## Step 1: Read the Input

Describe how to read/parse the input file or data.

```bash
# Example: check file size first
wc -c /mnt/user-data/uploads/<filename>
```

---

## Step 2: Analyze / Process

Describe the analysis or processing steps.

### Sub-section A
What to look for and how to interpret it.

### Sub-section B
What to look for and how to interpret it.

---

## Step 3: Produce Output

Describe the expected output format.

### Inline summary
What to show directly in the chat (bullet points, tables, etc.)

### Downloadable file (if applicable)
Reference the relevant skill for file creation:
- Word doc: `/mnt/skills/public/docx/SKILL.md`
- PDF: `/mnt/skills/public/pdf/SKILL.md`
- Spreadsheet: `/mnt/skills/public/xlsx/SKILL.md`

---

## Reference files

List any files in `references/` and when to read them:
- `references/example.md` — Read when you need details on X

---

## Important notes

- Note 1
- Note 2
