# Bonnes pratiques Web

Checklist francophone, vérifiable et réutilisable pour sécuriser et préparer des projets Web à la production — lisible par les humains et exploitable par les assistants IA.

> Cette documentation est une base de travail communautaire. Elle ne constitue pas un avis juridique, un audit de sécurité ni une garantie de conformité. Adaptez chaque recommandation à votre pays, votre activité, votre architecture et vos données.

## Ce que contient le dépôt

- [Checklist complète](docs/checklist-complete.md) : recommandations détaillées, exemples et critères de vérification.
- [Checklist courte](docs/checklist-courte.md) : contrôle rapide avant mise en production.
- [Workflow de déploiement](docs/workflow-deploiement.md) : séquence générique de validation et de mise en ligne.
- [Documentations d’intégration IA](docs/integrations.md) : instructions pour charger la skill dans OpenCode, Claude Code, Codex et outils compatibles SKILL.md.

Les sujets couverts incluent notamment la performance, l’UX, la sécurité applicative, les secrets, l’authentification, les paiements, le RGPD, l’accessibilité, le SEO, l’internationalisation, les sauvegardes, les tests et la supervision.

## Public visé

Ce dépôt s’adresse aux développeurs, équipes produit, freelances et petites équipes qui souhaitent disposer d’une checklist réutilisable pour des projets PHP/MySQL, sites vitrines, marketplaces, SaaS ou API publiques.

## Positionnement

Il existe déjà d’excellentes checklists web. Ce dépôt ne cherche pas à les remplacer : il se positionne comme **alternative francophone pragmatique et complémentaire**.

| Projet | Rapport à ce dépôt |
|--------|--------------------|
| [Front-End Checklist](https://github.com/thedaviddias/Front-End-Checklist) (thedaviddias) | Référence pour la qualité front-end (HTML, CSS, JS, a11y, SEO, perf) en anglais, avec site web et serveur MCP. **À utiliser en complément** pour le détail front-end. |
| [Production Readiness Checklist](https://github.com/MarinJursic/production-readiness-checklist) (MarinJursic) | Référence « go/no-go » industrielle (contrôles dits *evidence-driven*, scanner CLI). Beaucoup plus lourde ; ce dépôt en emprunte la philosophie des preuves et de la décision finale, à l’échelle d’une petite équipe. |
| [OWASP Developer Guide](https://owasp.org/www-project-developer-guide/) | Source de référence pour la section sécurité (successeur du *Secure Coding Practices Quick Reference Guide*, désormais archivé). |
| [RGAA 4](https://accessibilite.numerique.gouv.fr/) et WCAG 2.2 | Références d’accessibilité. Le RGAA est la norme de référence pour les sites publics français ; ce dépôt les cite sans s’y substituer. |

**Ce qui distingue ce dépôt :**

- français en priorité, approche pédagogique ;
- couverture PHP/MySQL, marketplace, SaaS, API et paiements ;
- [checklist courte](docs/checklist-courte.md) directement utilisable avant mise en production ;
- critères de vérification concrets ( Description / Pourquoi / Actions / Critères / Métriques ) ;
- compatibilité Agent Skills pour les assistants IA ;
- recommandations accompagnées de limites et d’avertissements (pas de certification).

## Utilisation

1. Parcourez la [checklist complète](docs/checklist-complete.md).
2. Sélectionnez les exigences pertinentes pour votre projet.
3. Transformez-les en tickets ou critères d’acceptation.
4. Notez les décisions, exceptions et risques propres à votre contexte.
5. Réévaluez la checklist à chaque évolution importante de l’application.

## Exemples d’usage

Ce projet supporte trois usages principaux :

- un développeur utilise la checklist pour auditer son projet ;
- une équipe transforme les points en critères d’acceptation ou tickets ;
- un assistant IA charge une compétence ciblée pour appliquer ces règles pendant une revue ou une implémentation.

Les sujets RGPD, paiements et sécurité sont présentés comme des aides à la vérification, pas comme une certification.

## Contribuer

Les corrections, précisions, traductions et nouveaux exemples sont bienvenus. Consultez [CONTRIBUTING.md](CONTRIBUTING.md) avant d’ouvrir une issue ou une pull request. Voir aussi les issues ouvertes avec les labels `good first issue`, `documentation`, `security`, `accessibility`, `french`, `agent-skill` et `help wanted`.

## Agent Skill

Ce dépôt contient une skill compatible avec le format Agent Skills pour OpenCode et les outils qui supportent `SKILL.md`.

Installation exemple :

```bash
mkdir -p skills/web-best-practices
cp docs/integrations.md skills/web-best-practices/ 2>/dev/null || true
# La SKILL.md principale est fournie dans ce dépôt.
```

Voir [docs/integrations.md](docs/integrations.md) pour les commandes d’installation propres à chaque outil.

## Licence

Le contenu documentaire est distribué sous licence [Creative Commons Attribution 4.0 International](LICENSE). Voir le fichier de licence pour les détails.

## Utilisé par / Intégrations IA

Cette section est mise à jour dès qu’un premier utilisateur ou projet confirme l’utilisation de la checklist ou de la skill.
