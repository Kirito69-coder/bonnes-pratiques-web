# Checklist courte avant mise en production

## Sécurité

- [ ] Aucun secret, token ou mot de passe n’est présent dans le dépôt.
- [ ] Les secrets sont fournis par l’environnement ou un gestionnaire dédié.
- [ ] Les mots de passe sont hachés avec un algorithme adapté.
- [ ] Les entrées utilisateur sont validées côté serveur.
- [ ] Les requêtes SQL utilisent des paramètres liés.
- [ ] Les contrôles d’accès sont appliqués côté serveur.
- [ ] Les sessions, cookies et formulaires sensibles sont protégés.
- [ ] HTTPS est activé partout.
- [ ] Les en-têtes de sécurité HTTP sont configurés (CSP, HSTS, nosniff, X-Frame-Options).
- [ ] Aucun compte, mot de passe ou réglage par défaut n’est actif en production.
- [ ] Les dépendances sont à jour et vérifiées.

## Données et conformité

- [ ] Les données collectées, les finalités et les durées de conservation sont documentées.
- [ ] La politique de confidentialité et les mentions nécessaires sont accessibles.
- [ ] Le consentement est demandé lorsque cela est requis.
- [ ] Le droit à l’effacement est effectivement implémenté et testé.
- [ ] Une procédure de notification de violation de données sous 72 heures est écrite et à jour.
- [ ] Les paiements, remboursements et abonnements ont été testés.
- [ ] Les obligations locales ont été vérifiées avec une source officielle ou un professionnel.

## Qualité et exploitation

- [ ] Les états de chargement, d’erreur et de liste vide sont prévus.
- [ ] Les pages sont utilisables sur mobile et au clavier.
- [ ] Les images ont un texte alternatif pertinent lorsque nécessaire.
- [ ] Les contrastes et le focus visible ont été vérifiés.
- [ ] Les tests automatisés et les tests de régression passent.
- [ ] Les parcours critiques ont un test automatisé de bout en bout (connexion, paiement, formulaire).
- [ ] Les sauvegardes sont automatisées et une restauration a été testée.
- [ ] La supervision et les alertes sont configurées.
- [ ] Un plan de retour arrière est documenté.
