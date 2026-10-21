# Prompts d’exemples

Prompts prêts à l’emploi pour demander à un agent d’utiliser la skill `web-best-practices`. Adaptez-les à la stack et au niveau de risque du projet.

## 1. Audit complet

```text
Utilise la skill web-best-practices pour auditer ce projet web.

1. Identifie le type de projet et son architecture.
2. Examine les fichiers pertinents pour ce contexte.
3. Distingue les faits des suppositions, avec citations fichier:ligne.
4. Classe chaque constat en P0, P1, P2 ou P3.
5. Attribue une disposition à chaque point : Conforme, Non conforme,
   Bloqué (preuve manquante) ou Non applicable.
6. Propose des corrections minimales.
7. Signale les points juridiques, financiers ou de sécurité
   nécessitant une validation humaine.
8. Termine par une recommandation GO / NO-GO / GO avec réserves.

N'inclus jamais de secrets ou d'identifiants dans la sortie.
```

## 2. Revue ciblée avant release

```text
Utilise la skill web-best-practices pour une revue de mise en production
de ce projet, en m'appuyant sur docs/workflow-deploiement.md.

Concentre-toi sur :
- l'écran de conditions bloquantes (no-go) de references/deployment.md ;
- les différences entre l'état actuel et le dernier tag/déploiement ;
- les secrets, migrations et configuration d'environnement.

Pour chaque point, donne la preuve (fichier:ligne ou commande) et la
disposition (Conforme / Non conforme / Bloqué / Non applicable).
Termine par GO / NO-GO avec la liste des risques acceptés.
```

## 3. Challenge de preuve (remettre en doute les conclusions)

```text
Tu viens de produire un rapport d'audit. Exerce maintenant le rôle de
reviewer sceptique : reprends chaque constat marqué "Conforme".

Pour chacun :
- La preuve citée (fichier:ligne) démontre-t-elle réellement le point,
  ou seulement une partie ?
- Une contre-mesure dans le code pourrait-elle invalider ce constat ?
- Le point pourrait-il être conforme en dev mais pas en production
  (config, variables d'environnement, reverse-proxy) ?

Passe en "Bloqué" tout Conforme dont la preuve est insuffisante.
Ne modifie aucun fichier : rapporte uniquement les révisions.
```

## Notes

- Les prompts sont des exemples, pas des garanties de résultat. Le résultat dépend de l’agent, des fichiers du projet disponibles et de la validation par un humain.
- Pour la structure du rapport attendu, voir `examples/project-audit-output.md`.
