# Third-party skill porting into a Hermes profile

Use this when the user gives a GitHub repo for a skill that is not already a Hermes hub skill.

## Pattern

1. **Inspect before installing**
   - Clone to `/tmp/<repo-name>` or use a temporary directory.
   - Read `README.md`, manifest files (`skill.json`, plugin manifests), and any generated skill/template files.
   - Identify the real source of truth: many assistant-specific skills keep canonical content under `src/...` and generate Claude/Cursor/Codex folders from templates.

2. **Prefer profile-local install**
   - Install under the active profile only, e.g. `~/.hermes/profiles/keith/skills/<category>/<skill-name>/`.
   - Do not write to the default profile unless explicitly asked.
   - For non-Hermes skill repos, create a Hermes-compatible `SKILL.md` with YAML frontmatter and adapt paths to absolute profile-local paths.

3. **Copy functional assets, not just docs**
   - Include scripts under `scripts/`.
   - Include data/search indexes under `data/` or `references/` as appropriate.
   - Include upstream README/manifests under `references/` for provenance.
   - If the upstream repo has a CLI installer, only use it if it supports Hermes directly; otherwise manual packaging is safer.

4. **Path adaptation**
   - Replace upstream examples such as `python3 skills/<name>/scripts/search.py` with the actual installed path under the active profile.
   - Make scripts executable when relevant.
   - Avoid symlinks if the skill must survive cleanup of the temp clone.

5. **Verification**
   - Run the included script/tool directly with a representative query.
   - Confirm `hermes --profile <profile> skills list` shows the skill enabled.
   - Confirm `skill_view(name='<skill-name>')` loads the new skill.
   - Optionally run `hermes --profile <profile> chat -s <skill-name> -q ...` to verify a fresh Hermes process can load it.

## Example: UI UX Pro Max

Installed as profile-local skill:

```text
~/.hermes/profiles/keith/skills/creative/ui-ux-pro-max/
```

Copied:

- `SKILL.md` adapted from upstream templates
- `scripts/search.py`, `scripts/core.py`, `scripts/design_system.py`
- `data/*.csv` and `data/stacks/*.csv`
- upstream README/manifest into `references/`

Representative verification command:

```bash
python3 ~/.hermes/profiles/keith/skills/creative/ui-ux-pro-max/scripts/search.py \
  "direct booking marketplace premium tropical" --design-system -p "Cebu Direct Stays"
```
