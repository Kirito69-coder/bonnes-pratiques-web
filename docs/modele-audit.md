# Modèle de rapport d’audit

Template vierge à copier pour chaque audit d’un projet web avec la skill `web-best-practices`. Copier ce fichier (par ex. `audits/audit-monprojet-AAAA-MM-JD.md`), remplir, et archiver avec le dépôt ou dans le ticket associé.

> Rappel des règles : un point n’est jamais `Conforme` sans preuve citée (`fichier:ligne` ou commande). Un P0 `Non conforme` ou `Bloqué` entraîne un **NO-GO**. Les sujets juridiques, financiers et de sécurité exigent une validation humaine.

---

# Rapport d'audit — [nom du projet] — [AAAA-MM-JJ]

## 1. Contexte

- **Nom / dépôt :** […]
- **Type de projet :** [PHP/MySQL | site vitrine | marketplace | SaaS | API publique | autre]
- **Architecture observée :** [stack, hébergement, BDD, services tiers — éléments vus, pas supposés]
- **Périmètre de l’audit :** [fichiers/dossiers/sections examinés]
- **Périmètre exclu :** [ce qui n’a pas été examiné et pourquoi]
- **Méthode :** skill web-best-practices v[version] ; revue statique uniquement / revue + exécution
- **Date :** […] — **Relecteur :** [humain ou agent]

## 2. Synthèse

| Disposition      | P0 | P1 | P2 | P3 |
|------------------|----|----|----|----|
| Conforme         |    |    |    |    |
| Non conforme     |    |    |    |    |
| Bloqué           |    |    |    |    |
| Non applicable   |    |    |    |    |

**Décision recommandée :** [GO / NO-GO / GO avec réserves]

Motif : [un P0 non conforme ou bloqué => NO-GO ; sinon, lister les réserves]

## 3. Constats

> Répéter le bloc suivant pour chaque constat, groupés par priorité décroissante.

### P[n]

- **[Titre du constat]** — disposition : `[Conforme / Non conforme / Bloqué / Non applicable]`
  - Type : [fait / supposition]
  - Preuve : `[fichier:ligne]` (ou commande exécutée)
  - Risque : [conséquence concrète si non traité]
  - Correction minimale : [action]
  - Validation humaine requise : [oui — juridique / financier / sécurité ; non]

## 4. Points bloquants (preuves manquantes)

- [point] — preuve requise : [fichier à inspecter, donnée à obtenir, accès à demander]

## 5. Validation humaine requise

- [constat] — domaine [juridique / financier / sécurité] — à valider par [qui] avant [quand]

## 6. Risques acceptés (optionnel — GO avec réserves)

| Risque | Priorité | Décideur | Échéance de traitement |
|--------|:--------:|----------|------------------------|
| […]   | P[n]    | […]     | […]                   |

## 7. Décision GO / NO-GO

| Champ | Valeur |
|-------|--------|
| Décision | [GO / NO-GO / GO avec réserves] |
| Décideur | [nom / rôle] |
| Date | […] |
| Prochaine réévaluation | [date ou événement] |

## 8. Limites du rapport

- [ce qui n’a pas pu être vérifié et pourquoi]
- [hypothèses de travail posées]

---

## Conseils de remplissage

- **Faits vs suppositions :** si vous n’avez pas lu la ligne, ce n’est pas un fait — écrivez `supposition` et, si le point est important, classez `Bloqué`.
- **Corrections minimales :** proposer le plus petit changement qui traite le risque, pas une refonte.
- **Réévaluation :** refaire l’audit à chaque évolution majeure (nouvelle intégration, nouveau type de données, nouveau prestataire).
- Un rapport d’audit ne remplace ni les tests automatisés, ni un audit de sécurité professionnel.
