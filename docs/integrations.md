# Intégrations IA

Cette page vise à documenter les outils et projets qui déclarent utiliser la checklist ou la skill `web-best-practices`.

## Outils compatibles

Les instructions d’installation décrivent les emplacements supportés par les agents suivants :

- OpenCode
- Claude Code
- Codex
- Cursor, GitHub Copilot (instructions de génération)
- les outils qui supportent le format Agent Skills (dossier contenant `SKILL.md`)

> **Note :** les chemins ci-dessous reflètent les conventions documentées par chaque outil à la date de rédaction. Vérifiez toujours la documentation officielle en cas de changement.

## Installation par outil

### OpenCode

OpenCode recherche les skills au niveau du projet (`.opencode/skills/`) puis dans les répertoires globaux de l’utilisateur (`~/.config/opencode/skills/`, où `~` est le dossier utilisateur, par ex. `C:\Users\votre-nom` sous Windows).

Installation dans le projet :

```bash
mkdir -p .opencode/skills/web-best-practices
cp -r skills/web-best-practices/* .opencode/skills/web-best-practices/
```

Installation globale (au niveau de l’utilisateur) :

```bash
mkdir -p ~/.config/opencode/skills/web-best-practices
cp -r skills/web-best-practices/* ~/.config/opencode/skills/web-best-practices/
```

OpenCode accepte aussi `AGENTS.md` pour les règles de projet et `opencode.json` avec un champ `instructions` pour charger des fichiers locaux ou distants.

Voir : https://opencode.ai/docs/skills/

### Claude Code

Installation dans le projet :

```bash
mkdir -p .claude/skills/web-best-practices
cp -r skills/web-best-practices/* .claude/skills/web-best-practices/
```

Installation globale (au niveau de l’utilisateur) :

```bash
mkdir -p ~/.claude/skills/web-best-practices
cp -r skills/web-best-practices/* ~/.claude/skills/web-best-practices/
```

### Codex

Installation dans le projet :

```bash
mkdir -p .agents/skills/web-best-practices
cp -r skills/web-best-practices/* .agents/skills/web-best-practices/
```

### Cursor

Cursor supporte les règles projet dans `.cursor/rules/`. Générer un fichier de règle à partir de la skill :

```bash
mkdir -p .cursor/rules
cat > .cursor/rules/web-best-practices.mdc << 'EOF'
---
description: Revue de projet web (sécurité, accessibilité, performance, RGPD, déploiement)
globs:
alwaysApply: false
---
Consulte skills/web-best-practices/SKILL.md et suis sa procédure d'audit
(quand tu revues du code web, prépares une mise en production ou audites
du code généré par IA). Cite les preuves (fichier:ligne), classe les risques
P0–P3, et demande une validation humaine pour les sujets juridiques,
financiers ou de sécurité.
EOF
```

### GitHub Copilot

Copilot n’a pas de dossier de skills natif ; utilisez un fichier d’instructions de projet **`.github/copilot-instructions.md`** :

```bash
mkdir -p .github
cat > .github/copilot-instructions.md << 'EOF'
# Instructions projet
Pour toute revue d'un projet web, suis la procédure de
`skills/web-best-practices/SKILL.md` : identifie le type de projet,
examine les fichiers pertinents, distingue les faits des suppositions,
classe les risques P0–P3 avec citations fichier:ligne, propose des
corrections minimales, et demande une validation humaine pour les sujets
juridiques, financiers ou de sécurité. N'inclus jamais de secrets dans
les sorties.
EOF
```

### Format portable Agent Skills

La référence officielle est fournie par Agent Skills :
https://agentskills.io/specification

Un index communautaire de skills réutilisables est disponible ici :
https://github.com/addyosmani/agent-skills et https://github.com/VoltAgent/awesome-agent-skills

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
