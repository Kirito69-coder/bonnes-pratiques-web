# Checklist sécurité

Cette référence liste les points de sécurité à vérifier lors d’une revue. À adapter au contexte du projet. Sources de référence : [OWASP Developer Guide](https://owasp.org/www-project-developer-guide/) (cible actuelle), [OWASP Secure Coding Practices – Quick Reference Guide v2.1](https://owasp.org/www-project-secure-coding-practices-quick-reference-guide/) (projet archivé, PDF toujours utile), [OWASP Top 10](https://owasp.org/www-project-top-ten/).

## Secrets

- Aucun secret, token ou mot de passe dans le dépôt.
- Secrets fournis par l’environnement ou un gestionnaire dédié.
- `.env` et `.gitignore` configurés pour que les secrets ne soient pas commités.
- Les clés visibles dans le bundle JS côté client sont uniquement des clés **publiques** (ex. `pk_...`). Les clés secrètes (Stripe, AWS…) ne quittent jamais le serveur.

## Mots de passe

- Mots de passe hachés avec un algorithme adapté (par ex. `password_hash()` avec Argon2id en PHP le cas échéant).
- Aucun stockage ni comparaison en clair.

## Validation et injection

- Entrées utilisateur validées côté serveur.
- Requêtes SQL paramétrées ou liées.
- Échappement de sortie appliqué quand pertinent (par ex. `htmlspecialchars()` en PHP).
- Aucune exécution de code dynamique (`eval()`, `new Function()`) sur des données non fiables.

## Sessions et contrôle d’accès

- Cookies de session configurés de façon sûre (HTTPS, HttpOnly, SameSite).
- Contrôle d’accès appliqué côté serveur, pas seulement côté client.
- Stockage des tokens d’authentification dans des cookies HttpOnly, pas en `localStorage`/`sessionStorage` (vol possible via XSS).

## Headers de sécurité HTTP

À configurer au niveau serveur ou reverse-proxy (Apache/Nginx/CDN) :

- **CSP** (`Content-Security-Policy`) : limite les sources de contenu exécutables, principale défense contre les XSS.
- **HSTS** (`Strict-Transport-Security`) : force HTTPS, anti-downgrade.
- **X-Content-Type-Options: nosniff** : empêche le sniffing MIME.
- **X-Frame-Options** (ou `frame-ancestors` dans la CSP) : anti-clickjacking.
- **Referrer-Policy** : contrôle les informations de referer envoyées (fuite de paramètres d’URL).
- **Permissions-Policy** (ex-Feature-Policy) : restreint caméra, micro, géolocalisation.

Exemple minimal (Nginx) :

```nginx
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
# CSP : à durcir progressivement selon les ressources réellement utilisées
add_header Content-Security-Policy "default-src 'self'; img-src 'self' data:; script-src 'self'" always;
```

## API et intégrations

- CORS restreints aux origines autorisées (jamais `Access-Control-Allow-Origin: *` sur des endpoints sensibles).
- Limitation de débit (rate limiting) sur les endpoints publics et actions sensibles.
- Protection CSRF sur les formulaires sensibles quand applicable.
- Liens `target="_blank"` accompagnés de `rel="noopener noreferrer"`.

## Contenu mixte et HTTPS

- HTTPS activé partout, avec redirections depuis HTTP.
- Aucune ressource (image, script, CSS) chargée en HTTP depuis une page HTTPS (contenu mixte).
- Certificat valide et configuration TLS à jour.

## Dépendances et chaîne d’approvisionnement

- Dépendances raisonnablement à jour.
- Vulnérabilités vérifiées avec l’outillage disponible (par ex. `composer audit`, `npm audit`).
- Fichiers de lock (`composer.lock`, `package-lock.json`) versionnés.
- Pour les ressources CDN : intégrité vérifiée via Subresource Integrity (attribut `integrity`) quand pertinent.

## Sauvegardes et reprise

- Sauvegardes automatiques configurées.
- Procédure de restauration **testée** (une sauvegarde jamais restaurée n’est pas une sauvegarde).

## Journalisation et supervision

- Erreurs journalisées pour le débogage en production.
- Événements de sécurité pertinents enregistrés (échecs de connexion, accès refusés).
- Aucune stack trace ni chemin interne exposé dans les réponses d’erreur de production.

## Notes

- Ces points sont des aides à la vérification, pas une certification.
- Les décisions juridiques et de sécurité doivent être validées avec une source officielle ou un professionnel compétent.
- Un audit automatisé ne remplace pas la revue manuelle.
