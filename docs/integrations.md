# Intégrations IA

Cette page vise à documenter les outils et projets qui déclarent utiliser la checklist ou la skill `web-best-practices`.

## Outils compatibles

Les instructions d’installation décrivent les emplacements supportés par les agents suivants :

- OpenCode
- Claude Code
- Codex
- les outils qui supportent le format Agent Skills (dossier contenant `SKILL.md`)

## Installation par outil

### OpenCode

OpenCode recherche les skills au niveau du projet et dans les répertoires globaux. Installer la skill dans le projet :

```bash
mkdir -p .opencode/skills/web-best-practices
cp skills/web-best-practices/SKILL.md .opencode/skills/web-best-practices/SKILL.md
```

Installation globale (au niveau de l’utilisateur) :

```bash
mkdir -p ~/.opencode/skills/web-best-practices
cp skills/web-best-practices/SKILL.md ~/.opencode/skills/web-best-practices/SKILL.md
```

OpenCode accepte aussi `AGENTS.md` pour les règles de projet et `opencode.json` avec un champ `instructions` pour charger des fichiers locaux ou distants.

Voir : https://opencode.ai/docs/skills/

### Claude Code

Installer la skill dans le projet :

```bash
mkdir -p .claude/skills/web-best-practices
cp skills/web-best-practices/SKILL.md .claude/skills/web-best-practices/SKILL.md
```

### Codex

Installer la skill dans le projet :

```bash
mkdir -p .agents/skills/web-best-practices
cp skills/web-best-practices/SKILL.md .agents/skills/web-best-practices/SKILL.md
```

### Format portable Agent Skills

La référence officielle est fournie par Agent Skills :
https://agentskills.io/specification

La skill `web-best-practices` est découpée pour être chargée seuls :
- `skills/web-best-practices/SKILL.md`
- `skills/web-best-practices/references/` (security.md, accessibility.md, performance.md, privacy.md, deployment.md)
- `skills/web-best-practices/examples/` (review-prompt.md, project-audit-output.md)

### Versions vérifiées

À compléter dès qu’au moins un outil confirme le chargement et l’usage de la skill.

## Projet utilisant la skill

À compléter dès qu’un premier utilisateur ou projet confirme l’utilisation de la skill, avec un lien vers le dépôt ou la documentation de l’intégration.

## Limites

- Une skill peut influencer le comportement d’un agent, mais ne remplace pas la validation humaine.
- Ne pas utiliser la skill pour expés, exfiltration de fichiers ou exécution de commandes destructrices non vérifiées.
- Les résultats de l’agent restent la responsabilité de l’utilisateur qui les valide, les teste et les publie.

## Contribuer à cette page

Si votre projet ou outil utilise la skill, ouvrez une issue ou une pull request en ajoutant votre outil, version vérifiée et commandes d’installation dans cette page, ou déposez un lien dans l’issue de suivi des intégrations.
