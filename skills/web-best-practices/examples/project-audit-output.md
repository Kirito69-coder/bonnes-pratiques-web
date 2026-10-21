# Exemple de rapport d’audit

Cet exemple illustre le type de sortie attendu d’une revue `web-best-practices`. Il est **fictif et illustratif**, pas un vrai résultat d’audit : les fichiers et lignes sont supposés.

---

# Rapport d'audit — Boutique PHP/MySQL (exemple fictif) — 2026-10-21

## Contexte

- Type de projet : application PHP/MySQL avec API publique et paiements (Stripe).
- Périmètre : configuration, authentification, API, pages de paiement, déploiement.
- Méthode : skill web-best-practices v0.2.0.

## Synthèse

| Disposition      | P0 | P1 | P2 | P3 |
|------------------|----|----|----|----|
| Conforme         | 1  | 1  | 1  | 0  |
| Non conforme     | 2  | 1  | 1  | 0  |
| Bloqué           | 1  | 0  | 0  | 0  |
| Non applicable   | 0  | 0  | 0  | 0  |

**Décision recommandée : NO-GO** — deux P0 non conformes et un P0 bloqué.

## Constats

### P0

- **Clé secrète Stripe exposée côté client** — disposition : `Non conforme`
  - Type : fait
  - Preuve : `public/js/stripe-example.js:3` (`sk_live_...` en dur) *(supposé)*
  - Risque : un tiers peut déclencher des paiements ou consulter les transactions.
  - Correction minimale : retirer la clé secrète du code client ; passer toute opération par le serveur.
  - Validation humaine requise : **oui — financier**.

- **CORS en wildcard sur l’API** — disposition : `Non conforme`
  - Type : fait
  - Preuve : `config/cors.php:12` (`Access-Control-Allow-Origin: *`) *(supposé)*
  - Risque : n’importe quel site peut consommer l’API au nom des utilisateurs connectés.
  - Correction minimale : restreindre aux origines connues.
  - Validation humaine requise : **oui — sécurité**.

- **Hachage des mots de passe** — disposition : `Bloqué`
  - Type : preuve manquante
  - Preuve requise : la méthode d’inscription (`app/Auth/Register.php`) n’a pas pu être inspectée (fichier absent du périmètre).
  - Risque si non conforme : vol massif de credentials en cas de fuite de base.
  - Action : inspecter l’usage de `password_hash()` / `password_verify()` avant toute décision.

- **HTTPS et redirections** — disposition : `Conforme`
  - Type : fait
  - Preuve : `.htaccess:4-6` (redirection 301 HTTP→HTTPS) *(supposé)*
  - Risque résiduel : vérifier l’en-tête HSTS (voir P2).

### P1

- **Restauration de sauvegarde non testée** — disposition : `Non conforme`
  - Type : fait
  - Preuve : `scripts/backup.sh` présent, mais aucun rapport de test de restauration dans le dépôt *(supposé)*
  - Risque : la sauvegarde peut s’avérer inutilisable en incident.
  - Correction minimale : planifier et documenter un test de restauration.
  - Validation humaine requise : non.

- **Paiements sandbox testés** — disposition : `Conforme`
  - Type : fait
  - Preuve : `tests/PaymentTest.php:20-45` (scénarios refus + succès) *(supposé)*

### P2

- **Stack traces en production** — disposition : `Non conforme`
  - Type : fait *(config, à confirmer sur l’environnement cible)*
  - Preuve : `app/config/debug.php:4` (`display_errors = true`) *(supposé)*
  - Risque : fuite d’informations internes (chemins, versions).
  - Correction minimale : désactiver l’affichage des erreurs en production, journaliser à la place.
  - Validation humaine requise : non.

- **Images sans dimensions déclarées** — disposition : `Non conforme`
  - Type : fait
  - Preuve : `templates/header.php:8` (img sans `width`/`height`) *(supposé)*
  - Risque : décalage de mise en page (CLS) au chargement.
  - Correction minimale : ajouter les attributs de dimensions.

- **En-têtes de sécurité HTTP** — disposition : `Conforme`
  - Type : fait
  - Preuve : `nginx/security-headers.conf:1-5` (nosniff, X-Frame-Options, Referrer-Policy) *(supposé)*
  - Manque : CSP et HSTS absents → suivi P2.

## Points bloquants (preuves manquantes)

- Hachage des mots de passe — preuve requise : inspection de `app/Auth/Register.php`.

## Validation humaine requise

- Clé secrète Stripe exposée (**financier**) : décision de rotation des clés et vérification des transactions passées.
- Configuration CORS (**sécurité**) : liste des origines autorisées à valider par l’équipe.

## Limites du rapport

- Revue statique du code uniquement : aucune exécution, aucun test d’environnement.
- Les configurations de production (variables d’environnement, reverse-proxy) n’ont pas été inspectées ; plusieurs constats sont à revalider sur l’environnement cible.

---

## Notes

- Cet exemple est un rapport fictif : les fichiers et lignes marqués *(supposé)* sont illustratifs.
- Un vrai rapport ne peut pas marquer `Conforme` un point dont la preuve n’a pas été lue.
- Pour la structure officielle, voir le modèle dans `skills/web-best-practices/SKILL.md` et `docs/modele-audit.md`.
