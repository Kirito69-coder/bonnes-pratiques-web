# Checklist déploiement

Cette référence liste les points à vérifier avant de passer en production ou après un déploiement.

## Écran de conditions bloquantes (no-go)

Avant toute décision de mise en ligne, vérifier qu’**aucun** de ces blocants n’est présent. Un seul suffit à stopper le déploiement :

- [ ] Secret, clé API ou mot de passe dans le dépôt ou exposé côté client.
- [ ] Mots de passe non hachés, ou contrôle d’accès absent côté serveur.
- [ ] HTTPS manquant, ou contenu mixte sur une page HTTPS.
- [ ] Injection SQL possible (requêtes concaténées sur des données utilisateur).
- [ ] Sauvegarde sans test de restauration récent, ou sans plan de retour arrière.
- [ ] Données personnelles traitées sans politique de confidentialité ni base légale.
- [ ] (E-commerce/SaaS) Flux de paiement non testé, ou clé secrète de paiement exposée.
- [ ] Erreurs détaillées (stack traces) affichées en production.

> Tout point **P0 non conforme** ou **P0 bloqué** = **NO-GO**.

## Avant le déploiement

- Changements et migrations revus.
- Secrets confirmés provenant de l’environnement cible (pas du dépôt).
- Tests, linters et contrôles de dépendances exécutés.
- Sauvegardes et procédure de restauration confirmées.
- Plan de retour arrière documenté (comment, en combien de temps, qui décide).
- Changements DNS, certificats et variables de configuration confirmés.

## Après le déploiement

- Vérifications fonctionnelles minimales sur les parcours critiques (connexion, paiement, envoi d’email le cas échéant).
- Logs, erreurs 4xx/5xx, latence et disponibilité revus.
- Paiements, emails et intégrations externes contrôlés le cas échéant.
- Supervision observée pendant la période post-déploiement.
- Critères de retour arrière documentés et appliqués si nécessaire.

## Hygiène de production

- Modes debug/verbeux désactivés en production.
- Pages d’erreur personnalisées pour les cas courants (404, 500).
- Marqueurs de démo, badges factices et contenu placeholder supprimés.
- Favicon et domaine personnalisé configurés le cas échéant.
- Headers de sécurité HTTP configurés (voir `references/security.md`).

## Références

- Workflow de déploiement du projet : `docs/workflow-deploiement.md`
- Modèle de rapport d’audit : `docs/modele-audit.md`

## Notes

- Cette checklist est une séquence générique : à adapter au niveau de risque et à l’architecture du projet.
- Documenter les décisions GO/NO-GO (qui décide, sur quelle preuve, avec quels risques acceptés).
