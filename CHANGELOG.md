# Changelog

All notable changes to this repository will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- README: new “Positionnement” section (French-language pragmatic alternative, complementary to Front-End Checklist and Production Readiness Checklist, with comparison table).
- Four-state disposition model (`Conforme` / `Non conforme` / `Bloqué` / `Non applicable`) adopted across the skill and templates — no `Conforme` without a cited proof (`file:line` or command).
- `docs/modele-audit.md`: reusable audit report template (context, findings, blockers, human validation, GO/NO-GO decision).
- `docs/workflow-deploiement.md`: pre-deployment no-go conditions screen and GO/NO-GO decision record table.
- `AGENTS.md`: root instructions for AI assistants working on this repository.
- Markdown CI: `.markdownlint-cli2.jsonc` and `.github/workflows/docs-quality.yml` (markdownlint + broken-link check with lychee).
- Skill `web-best-practices` v0.2.0: bilingual front matter (English) / body (French), explicit 7-step procedure, standardized report structure.
- Skill examples: prompt library (`examples/review-prompt.md` with full audit, release review, and evidence-challenge prompts) and a 4-disposition example report.
- Security reference: HTTP security headers section (CSP, HSTS, nosniff, X-Frame-Options, Referrer-Policy, Permissions-Policy).
- Performance reference: Core Web Vitals targets (LCP, INP, CLS, TTFB), performance budget, modern image formats and responsive images.
- Accessibility reference: RGAA 4.1 alongside WCAG 2.2, advanced a11y items (ARIA live regions, reduced motion, touch targets, reflow, screen-reader testing).
- Privacy reference: data minimisation and retention, erasure request implementation, 72-hour breach notification, records of processing.

### Changed

- README: explicit references to the OWASP Developer Guide (current) alongside the archived OWASP Secure Coding Practices Quick Reference Guide v2.1, and to RGAA 4/WCAG 2.2.
- `docs/checklist-complete.md` §9: P0–P3 priorities now have operational definitions (trigger + examples + decision rule).
- `docs/checklist-complete.md` §3 and §5: OWASP and RGAA citations added.
- Skill references rewritten in French for consistency with the rest of the repository.
- `docs/integrations.md`: corrected OpenCode global skills path (`~/.config/opencode/skills/`), added Cursor and GitHub Copilot integration instructions, linked community skill indexes.

## [0.1.0] - 2026-10-21

### Added

- README.md rewritten with a clearer positioning and an “Utilisé par / Intégrations IA” section.
- LICENSE replaced with the full Creative Commons Attribution 4.0 International text.
- CODE_OF_CONDUCT.md based on Contributor Covenant v2.1.
- GitHub community health files:
  - `.github/ISSUE_TEMPLATE/bug-report.yml`
  - `.github/ISSUE_TEMPLATE/improvement.yml`
  - `.github/ISSUE_TEMPLATE/skill-request.yml`
  - `.github/PULL_REQUEST_TEMPLATE.md`
- Documentation:
  - `docs/integrations.md` with installation instructions for OpenCode, Claude Code, Codex and portable Agent Skills.
- Agent Skill skeleton:
  - `skills/web-best-practices/SKILL.md`
  - `skills/web-best-practices/references/security.md`
  - `skills/web-best-practices/references/accessibility.md`
  - `skills/web-best-practices/references/performance.md`
  - `skills/web-best-practices/references/privacy.md`
  - `skills/web-best-practices/references/deployment.md`
  - `skills/web-best-practices/examples/review-prompt.md`
  - `skills/web-best-practices/examples/project-audit-output.md`
- Front matter removed from README.md and docs/checklist-complete.md.
- First release `v0.1.0`.

### Changed

- README.md updated to present the project as a reusable, verifiable, French-language Web checklist usable by humans and IA assistants.
- LICENSE text standardized for better license detection.

### Known limitations

- Some issue templates and labels may still need to be refined from contributions.
- Integrations page lists installation paths but not verified tool versions yet.
