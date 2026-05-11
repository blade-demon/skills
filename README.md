# my-agent-skills

Install via npm:

```bash
npx skills add @blade-demon/skills
```

Install from GitHub:

```bash
npx skills add github:blade-demons/my-agent-skills
```

## Skills

- `skills/frontend-reviewer/SKILL.md`

## Publish

```bash
npm login
npm whoami
npm pack --dry-run
npm publish --access public
```

For each update, bump `version` in `package.json` before publishing again.
