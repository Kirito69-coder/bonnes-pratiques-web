# Résumé exécutif

> **Avertissement :** cette checklist fournit des recommandations générales et des exemples techniques. Elle ne constitue pas un avis juridique, de sécurité ou de conformité. Les obligations applicables dépendent du pays, de l’activité, de l’architecture et des données traitées.

Ce document est une **checklist détaillée et structurée** de bonnes pratiques de développement web, destinée à garantir la fiabilité, la performance, la sécurité, l’accessibilité et la conformité légale de votre site ou application. Il couvre plusieurs types de projets (application PHP/MySQL classique, marketplace, site vitrine statique, SaaS avec paiements récurrents, API publique) et présente pour chaque point une *description concise, son importance*, des *actions concrètes*, des *exemples de code ou commandes* adaptés, ainsi que des *critères d’acceptation et des indicateurs de vérification*. Des logiques conditionnelles (“si / sinon si / sinon”) indiquent comment adapter chaque recommandation selon le type de projet. Le document inclut enfin un **workflow de déploiement** et une **timeline** en diagrammes Mermaid, ainsi que des tableaux comparatifs des tâches par priorité et par type de projet. Une checklist finale prête à copier-coller est fournie en commentaire.

# 1. Performance

- **Limiter les requêtes par visiteur (rate limiting)**.  
  *Description :* Éviter qu’un même utilisateur ou bot inonde le serveur de requêtes. *Pourquoi :* Prévenir les abus, protéger les API externes (limites de quotas) et les ressources serveurs. *Actions :* Implémenter un mécanisme de throttle (en PHP, suivre les adresses IP ou ID utilisateur dans un cache ou base Redis). Par exemple :  
  ```php
  // Exemple simple de limitation dans PHP
  $ip = $_SERVER['REMOTE_ADDR'];
  $count = $cache->get("req_count_$ip") ?? 0;
  if ($count > 100) {
      http_response_code(429); echo "Trop de requêtes (Rate limit atteint)."; exit;
  }
  $cache->set("req_count_$ip", $count+1, 60); // expire en 60s
  ```  
  *Critères :* Code HTTP 429 après le seuil, journalisation des rejets. *Métriques :* nombre de requêtes bloquées par minute, temps de réponse moyen.

  - *Si site vitrine statique :* moins critique (pas d’API dynamique à protéger), mais limiter login/admin si existant.  
  - *Sinon si API publique :* appliquer strictement (API key, OAuth scopes, quotas par utilisateur).  

- **Mettre en cache les réponses statiques et données peu volatiles**.  
  *Description :* Stocker en mémoire ou via HTTP des résultats (pages HTML, API JSON, fragments) fréquemment demandés. *Pourquoi :* Réduire la charge serveurs, améliorer la rapidité. *Actions :* Utiliser HTTP `Cache-Control` (ex. `max-age=3600`) pour le navigateur/CDN. Cacher en PHP (APCu, Redis) les requêtes SQL coûteuses. Exemple HTTP :  
  ```
  Cache-Control: public, max-age=300
  ```  
  *Critères :* Réponses avec en-tête `Cache-Control`, tests via `curl -I` montrent le bon `max-age`. *Métriques :* taux de cache hit (CDN/browser), nombre de requêtes/s après mise en cache.

  - *Si site statique :* déployer sur CDN ou service de pages statiques (Netlify, etc.) avec invalidation par version. *Métriques :* latence moyenne pour pages statiques.  
  - *Si SaaS/API :* configurer caches HTTP ou reverse-proxy (Varnish).

- **Afficher un indicateur de chargement**.  
  *Description :* Afficher un spinner ou message (« Chargement… ») pendant les requêtes longues (API ou navigation). *Pourquoi :* Améliorer l’UX en évitant un écran blanc ou suspens total. *Actions :* En front-end, désactiver le bouton Envoyer et montrer un spinner (Bootstrap `spinner-border`, CSS). Exemple Bootstrap :  
  ```html
  <button id="submitBtn">Envoyer</button>
  <div id="loader" class="spinner-border d-none" role="status"></div>
  <script>
    document.getElementById('submitBtn').addEventListener('click', function(){
      this.disabled = true;
      document.getElementById('loader').classList.remove('d-none');
      // procéder à la requête, etc.
    });
  </script>
  ```  
  *Critères :* Lors de chaque formulaire soumis, on voit un chargement et le bouton est désactivé. *Métriques :* Temps total de requête moyen, taux de clics multiples bloqués.

- **Gérer les cas de résultats vides (état « pas de données »)**.  
  *Description :* Prévoir et afficher un message clair quand une liste ou requête ne renvoie aucun élément. *Pourquoi :* Éviter confusion utilisateur (pas juste affichage vide). *Actions :* Afficher un texte comme « Aucun résultat trouvé » ou une icône, avec un code HTTP 204 ou 200. Exemple PHP/MySQL :  
  ```php
  $stmt = $pdo->query("SELECT * FROM articles WHERE categorie = ?");
  $data = $stmt->fetchAll();
  if (empty($data)) {
    echo "<p class='text-muted'>Aucun article à afficher.</p>";
  }
  ```  
  *Critères :* Pour toute requête, une page ou composant d’état vide existe (ex. liste avec 0 items) et n’affiche pas d’erreur. *Métriques :* Nombre de vues de page « vide », satisfaction user (feedback).

- **Paginer les longues listes**.  
  *Description :* Diviser les grandes tables/résultats en pages (limit+offset). *Pourquoi :* Charger de grandes quantités de données ralentit la page et augmente la charge DB. *Actions :* En PHP/MySQL, utiliser `LIMIT 50 OFFSET 0/50/100...`. Exemple :  
  ```php
  $page = $_GET['page'] ?? 1;
  $limit = 20; $offset = ($page-1)*$limit;
  $stmt = $pdo->prepare("SELECT * FROM posts ORDER BY date DESC LIMIT ? OFFSET ?");
  $stmt->execute([$limit, $offset]);
  ```  
  *Critères :* Les listes n’affichent jamais plus que N éléments (par ex. 20). Boutons/précédent/suivant fonctionnels. *Métriques :* Nombre de requêtes/page, temps de chargement de liste.

- **Ajouter des index aux colonnes fréquemment requêtées**.  
  *Description :* Créer des index SQL sur les champs utilisés en filtres (`WHERE`) et tris (`ORDER BY`). *Pourquoi :* Accélère considérablement les SELECT sur tables volumineuses. *Actions :* MySQL : `ALTER TABLE users ADD INDEX (email);` ou `CREATE INDEX idx_users_email ON users(email);`. *Critères :* Vérifier via `EXPLAIN` que les requêtes critiques utilisent l’index. *Métriques :* Temps des requêtes avant/après, nombre de lignes scannées (`EXPLAIN`).

- **Limiter le poids des fichiers uploadés et compresser les ressources**.  
  *Description :* Définir une taille max pour les uploads et compresser les images/JS/CSS en production. *Pourquoi :* Éviter surcharge réseau et stockage, améliorer temps de chargement. *Actions :* En PHP (`php.ini` ou code) par ex. `$_FILES` `upload_max_filesize=5M`. En front, compress/gzip les assets (config Apache/Nginx). *Critères :* Le serveur refuse au-delà (ex. message d’erreur « Fichier trop volumineux »). *Métriques :* Bande passante utilisé, temps de transfert moyen.

- **Prendre en compte plusieurs utilisateurs simultanés (tests de charge)**.  
  *Description :* Vérifier que le site tient sous la charge attendue. *Pourquoi :* S’assurer de la robustesse en production. *Actions :* Utiliser `ab`, `siege` ou JMeter. Ex. `ab -n 1000 -c 50 http://votresite.test/`. *Critères :* Les temps de réponse restent acceptables, pas d’erreur 5xx sous charge attendue. *Métriques :* Requêtes par seconde supportées, taux d’erreur sous charge.

- **Ne charger que les données nécessaires**.  
  *Description :* Éviter les requêtes SQL `SELECT *` ou appels API qui ramènent plus que nécessaire. *Pourquoi :* Réduire la bande passante et la mémoire consommée. *Actions :* Utiliser `SELECT colonne1, colonne2` précis. Exemple PDO :  
  ```php
  $stmt = $pdo->prepare("SELECT nom, prix FROM products WHERE categorie = ?");
  ```  
  *Critères :* Les requêtes SQL ne récupèrent que les champs utilisés. *Métriques :* Utilisation mémoire par requête, payload JSON moyen.

- **Mise en cache des données peu volatiles**.  
  *Description :* Garder en mémoire (Redis, Memcached, fichiers) ce qui ne change pas souvent (métadonnées, filtres). *Pourquoi :* Diminuer hits DB. *Actions :* Par ex. stocker le schéma d’un formulaire en cache. Utiliser l’en-tête HTTP `ETag` ou `Last-Modified` pour que le navigateur ne re-télécharge pas si inchangé. *Critères :* Présence de cache, tests reflétant changement (ex. modifier données -> nouvelle version servie). *Métriques :* Taux de cache hit, nombre de requêtes DB par minute.

- **Surveiller la disponibilité et les performances du site**.  
  *Description :* Utiliser un outil de monitoring (Pingdom, UptimeRobot, Datadog) pour détecter les pannes ou lenteurs. *Pourquoi :* Réagir rapidement en cas d’incident. *Actions :* Configurer alertes (e-mail/SMS) sur indisponibilité (>5 minutes) ou dépassement de latence. *Critères :* Alertes fonctionnelles, logs de monitoring accessibles. *Métriques :* Taux de disponibilité (uptime), temps de réponse moyen.

- **Mesurer et respecter les Core Web Vitals**.  
  *Description :* Vérifier les trois métriques d’expérience utilisateur de Google sur les pages critiques : **LCP** (chargement) ≤ 2,5 s, **INP** (interactivité) ≤ 200 ms, **CLS** (stabilité visuelle) < 0,1 — seuils « bons » sur mobile. *Pourquoi :* Ce sont les métriques mesurées par le CrUX de Google et corrélées à la perception de qualité (et au référencement). *Actions :* Mesurer avec Lighthouse / PageSpeed Insights / WebPageTest sur mobile ; corriger le plus gros élément visible (LCP : image hero prioritaire, serveur lent), les tâches JS longues (INP), les images/ads sans dimensions réservées (CLS). *Critères :* Les 3 métriques dans le vert sur les pages critiques. *Métriques :* Valeurs LCP/INP/CLS mesurées, évolutions avant/après chaque optimisation.

  - *Si site vitrine :* mesurer la page d’accueil et la page contact/rendez-vous.  
  - *Sinon si SaaS/application :* mesurer les écrans de connexion et le tableau de bord.  
  - *Si API publique :* non applicable aux réponses JSON, mais le TTFB des endpoints servis par une page web reste pertinent.

- **Définir un budget de performance**.  
  *Description :* Fixer un poids maximal de page (par ex. < 500 Ko chargées pour une vitrine, < 1,5 Mo pour une application riche) et le tenir. *Pourquoi :* Sans budget, chaque nouvelle librairie ou image « un peu lourde » finit par dégrader le site de façon invisible. *Actions :* Mesurer le poids actuel (DevTools → Network), le comparer au budget, et ajouter un contrôle en CI (par ex. `bundlesize`, Lighthouse CI) qui échoue si le budget est dépassé. *Critères :* Budget écrit et respecté à chaque release. *Métriques :* Poids total par page, évolution dans le temps, nombre de dépassements bloqués en CI.

- **Servir des images modernes et adaptatives**.  
  *Description :* Utiliser des formats efficaces (WebP ou AVIF avec repli via `<picture>`) et servir la bonne résolution selon l’écran. *Pourquoi :* L’image est souvent le premier poste de poids d’une page ; un JPEG non optimisé peut coûter plusieurs secondes de LCP. *Actions :* Générer des variantes et utiliser `srcset`/`sizes` ; poser `width`/`height` (ou `aspect-ratio`) sur chaque `<img>` pour éviter les décalages de mise en page (CLS) ; `loading="lazy"` sur les images hors premier écran (**pas** sur l’image hero) ; dimensionner l’image à sa taille d’affichage (servir 2000 px dans un conteneur de 400 px gaspille bande passante). Exemple :  
  ```html
  <img src="photo-400.avif" width="400" height="300" loading="lazy"
       srcset="photo-400.avif 400w, photo-800.avif 800w"
       sizes="(max-width: 600px) 100vw, 400px"
       alt="Description de la photo">
  ```  
  *Critères :* Aucune image sans dimensions ; images du premier écran en format moderne et dimensionnées justes. *Métriques :* Poids moyen des pages, LCP, CLS avant/après.

- **Activer la compression et les protocoles modernes**.  
  *Description :* Compresser les réponses textuelles (Brotli ou à défaut Gzip) et servir en HTTP/2 (ou HTTP/3) quand l’hébergement le permet. *Pourquoi :* HTML, CSS et JS compressés pèsent 3 à 5 fois moins ; HTTP/2 permet de multiplexer les requêtes sur une seule connexion. *Actions :* Activer la compression côté serveur (mod `mod_deflate` Apache, `gzip`/`brotli` Nginx, réglage CDN) et vérifier avec `curl -H "Accept-Encoding: br" -I https://votresite.fr` ; confirmer le protocole dans DevTools (onglet Network → colonne Protocol). *Critères :* Réponses textuelles compressées (en-tête `Content-Encoding`), site servi en h2/h3. *Métriques :* Taille transférée avant/après, temps de chargement.

- **Charger les scripts tiers sans bloquer et précharger les ressources critiques**.  
  *Description :* Empêcher les librairies externes (analytics, widgets, publicité) de bloquer l’affichage, et prévenir les allers-retours réseau vers les origines tierces. *Pourquoi :* Un script tiers synchronisé au `<head>` gèle le rendu jusqu’à son téléchargement (souvent le principal coupable d’un LCP dégradé). *Actions :* Utiliser `defer` (ou `async` si indépendant) sur les `<script>` ; `preconnect`/`dns-prefetch` vers les origines essentielles ; `preload` réservé aux ressources du chemin critique (image hero, CSS critique) — sans en abuser. Exemple :  
  ```html
  <link rel="preconnect" href="https://cdn.example.com">
  <script src="analytics.js" defer></script>
  ```  
  *Critères :* Aucun `<script>` bloquant non différé hors critical path ; pas de `preload` superflu (alertes DevTools). *Métriques :* TTFB du rendu, LCP, nombre de requêtes bloquantes.

- **Optimiser le chargement des polices web**.  
  *Description :* Limiter le nombre de polices et de graisses chargées, et éviter que leur téléchargement n’efface ou ne bloque le texte. *Pourquoi :* Les polices sont souvent le plus gros actif d’une page ; sans `font-display`, le navigateur masque le texte pendant le téléchargement (FOIT). *Actions :* Préférer WOFF2, limiter à 2 familles et aux graisses réellement utilisées, définir `font-display: swap` (ou `optional`), héberger les polices en local ou sur un domaine dédié avec `font-display` + `preload` de la variante principale. *Critères :* `font-display` présent sur chaque `@font-face` ; ≤ 2 familles chargées. *Métriques :* Poids des polices, CLS dû aux bascules de texte, LCP.

# 2. Design et Identité visuelle

- **Éviter les styles non professionnels**.  
  *Description :* Ne pas utiliser aveuglément les tendances (dégradés violets, boutons « pill » flashy, emojis dans les titres, etc.) simplement parce que c’est à la mode. *Pourquoi :* Un design générique ou surchargé peut nuire à la crédibilité du site. *Actions :* Choisir une palette de couleurs restreinte, des typographies lisibles (Arial, Roboto, sans-serif classiques), et un style cohérent avec la marque. Par ex. Bootstrap avec classes propres ( `.btn-primary`, pas de design personnalisé obscur). *Exemple de CSS :*  
  ```css
  .btn-custom { 
    background-color: #0055a5; color: #fff; border-radius: 4px;
  }
  h1 { color: #333; font-family: Arial, sans-serif; }
  ```  
  *Critères :* Design validé par revue UX/UI, tests utilisateurs. *Métriques :* Taux de conversion, retours utilisateurs.

- **Éviter le contenu factice**.  
  *Description :* Bannir faux témoignages, statistiques inventées, compteurs non démo. *Pourquoi :* Restez crédibles : des chiffres ou avis inventés peuvent être découverts. *Actions :* Utiliser de vrais témoignages (avec permission) ou exemples. Si nécessaire, indiquer « Exemple : ». *Critères :* Tous les retours/avis sont authentifiés. *Métriques :* Feedback clients, absence de retours négatifs sur contenu factice.

- **Animations discrètes et cohérentes**.  
  *Description :* Les effets (hover, transitions) doivent servir l’UX, pas simplement décorer. *Pourquoi :* Trop d’animations surcharge l’interface et gêne la lisibilité. *Actions :* Utiliser animations modérées (ex. fade-in en CSS, transitions 0.2s) et éviter les animations de scroll excessives. *Critères :* Tests clavier/souris : animations présentes et rapides, mais pas systématiques sur chaque élément. *Métriques :* Ergonomie (test utilisateur), score Lighthouse de performance (audits d’animations).

- **Utiliser des démonstrations ou captures réelles**.  
  *Description :* Préférer des captures d’écran ou vidéos vraies du produit plutôt que des maquettes abstraites. *Pourquoi :* L’utilisateur comprend mieux un véritable aperçu. *Actions :* Exemple : embed `<video>` ou `<img>` réels. Exemple HTML :  
  ```html
  <img src="screenshot-vraie-app.png" alt="Capture d'écran de l'application"> 
  ```  
  *Critères :* Présence d’au moins une vraie démo du produit/service. *Métriques :* Engagement utilisateur sur démonstrations (clics, vues).

- **Identité visuelle cohérente**.  
  *Description :* Logo, couleurs, typographie, etc. doivent être harmonisés et refléter la marque. *Pourquoi :* Renforce la reconnaissance et la confiance. *Actions :* Établir un style guide (couleurs HEX validées, police accessible comme Open Sans). *Critères :* Accord avec la charte graphique (exemple : palette de couleurs). *Métriques :* Cohérence visuelle selon audit manuel ou outils de palette.

- **Responsive design**.  
  *Description :* Assurer une bonne expérience sur mobile, tablette et desktop. *Pourquoi :* Les bonnes pratiques responsive et les référentiels d’accessibilité encouragent l’accès multi-plateforme. *Actions :* Utiliser Bootstrap grid, tester sur Chrome devtools (mobile view). *Critères :* Page adaptable (aucun scroll horizontal, menus hamburger visible sur mobile). *Métriques :* Scores de test responsive, trafic mobile vs taux de rebond.

- **Éviter éléments décoratifs superflus**.  
  *Description :* Pas d’éléments (animations, orbes flous, bruit de fond, etc.) qui n’apportent pas d’information. *Pourquoi :* Ces éléments distraient l’utilisateur et peuvent nuire à l’accessibilité. *Actions :* Utiliser images décoratives en background avec `aria-hidden="true"` et alt vide (`alt=""`). *Critères :* Audit d’accessibilité (les décorations ont `alt=""`). *Métriques :* Score d’accessibilité (voir section Accessibilité).

# 3. Sécurité et intégrité

> **Références :** cette section s’appuie sur l’[OWASP Developer Guide](https://owasp.org/www-project-developer-guide/) (source actuelle recommandée) et le [Secure Coding Practices – Quick Reference Guide v2.1](https://owasp.org/www-project-secure-coding-practices-quick-reference-guide/) (projet archivé, toujours utile en PDF). Pour le classement des risques, voir l’[OWASP Top 10](https://owasp.org/www-project-top-ten/). Ces sources font foi en cas de divergence avec les exemples ci-dessous.

- **Clés et secrets dans `.env` et `.gitignore`.**  
  *Description :* Stocker les configurations sensibles (clés API, identifiants DB) hors du code source. *Pourquoi :* Empêcher fuite de mots de passe ou API keys publiques. *Actions :* Exemple `.env` :  
  ```dotenv
  DATABASE_URL=mysql://user:pass@localhost/mondb
  STRIPE_SECRET_KEY=sk_test_*************
  ```  
  Et dans `.gitignore` :  
  ```gitignore
  # Variables d'environnement et secrets
  .env
  /config/*.secret.php
  # Dossiers de dépendances
  /vendor/
  /node_modules/
  ```  
  *Critères :* Vérifier qu’aucune variable sensible n’est commitée (ex: `git grep` pour `key=`, `secret`). *Métriques :* Pas d’erreur de fuite au contrôle de code (ex. GitHub secret scanner).

- **Hachage des mots de passe (pas de clair)**.  
  *Description :* Ne jamais stocker de mots de passe en clair. *Pourquoi :* Risque de vol massif en cas de faille. *Actions :* En PHP, utiliser `password_hash()` (Argon2id de préférence) et `password_verify()`. Exemple :  
  ```php
  $hash = password_hash($pass, PASSWORD_ARGON2ID);
  // Vérification
  if (password_verify($pass, $hash)) { /* ok */ }
  ```  
  *Critères :* Tous les mots de passe dans DB sont hachés (pas de `password='xxx'`). *Métriques :* Passation de test de fuite (ex. Try to fetch DB dumps).

- **Validation et nettoyage côté serveur**.  
  *Description :* Contrôler toutes les données entrantes (formulaires, API). *Pourquoi :* Prévenir les injections SQL, XSS et autres attaques. *Actions :* En PHP, vérifier format (email, entiers), utiliser `htmlspecialchars()` sur l’affichage. Exemple XSS safe :  
  ```php
  echo htmlspecialchars($username, ENT_QUOTES, 'UTF-8');
  ```  
  *Critères :* Passer l’OWASP ZAP ou des tests XSS/SQLi (faux inputs `<script>alert()</script>`, `' OR '1'='1`). *Métriques :* Nombre de vulnérabilités détectées (dev/security scan).

- **Requêtes SQL préparées (PDO)**.  
  *Description :* Ne pas concaténer de données utilisateur dans les requêtes. *Pourquoi :* Empêche SQL Injection. *Actions :* Exemple PDO préparé :  
  ```php
  $stmt = $pdo->prepare("SELECT * FROM users WHERE email = :email");
  $stmt->execute([':email' => $email]);
  ```  
  *Critères :* `prepare/execute` utilisés partout. *Métriques :* Tests inject SQL échouent (aucune donnée en trop retournée).

- **Clés publiques vs secrètes**.  
  *Description :* Une clé publique (ex. `STRIPE_PUBLIC_KEY`) peut être exposée client, la clé secrète ne doit jamais quitter le serveur. *Pourquoi :* Sécurité. *Actions :* Exemple front-end Stripe : `<script src="https://js.stripe.com/v3/"></script>` ne contient pas la clé secrète. *Critères :* Revue de code confirme séparation. *Métriques :* Aucune clé secrète ne figure dans le code client.

- **Sessions sécurisées**.  
  *Description :* Protéger les cookies de session (HTTPS, HttpOnly, SameSite). *Pourquoi :* Éviter vol de session ou fixation. *Actions :* PHP.ini ou en code :  
  ```php
  session_set_cookie_params(['lifetime'=>3600, 'secure'=>true, 'httponly'=>true, 'samesite'=>'Lax']);
  session_start();
  ```  
  *Critères :* Les cookies `PHPSESSID` ont l’attribut Secure et HttpOnly (vérifiable dans navigateur). *Métriques :* Test d’outils (Firefox devtools > cookies).

- **Contrôles d’accès côté serveur**.  
  *Description :* Vérifier les autorisations en PHP, pas seulement en JS. *Pourquoi :* Le client peut être contourné. *Actions :* Exemple :  
  ```php
  if ($_SESSION['user_role'] !== 'admin') {
    http_response_code(403); exit("Accès interdit.");
  }
  ```  
  *Critères :* Les endpoints sensibles retournent 403 si rôle non admin. *Métriques :* Tentatives d’accès non autorisé (logs).

- **Limitation des tentatives de connexion (brute force)**.  
  *Description :* Compter les échecs de login par IP/utilisateur. *Pourquoi :* Empêcher devinettes de mot de passe. *Actions :* Après 5 essais échoués, bloquer temporairement l’IP. *Critères :* 6<sup>ème</sup> échec retourne message générique (“Identifiants incorrects”). *Métriques :* Nombre d’échecs de connexion bloqués.

- **Mécanismes CSRF**.  
  *Description :* Protéger les formulaires POST sensibles par un token CSRF. *Pourquoi :* Éviter que des requêtes malveillantes (e.g. sous forme de page HTML) fassent des actions au nom de l’utilisateur. *Actions :* Générer un `$_SESSION['csrf_token']`, l’injecter en champ caché dans le form. Exemple :  
  ```php
  echo "<input type='hidden' name='csrf_token' value='$_SESSION[csrf_token]'>";
  if ($_POST['csrf_token'] !== $_SESSION['csrf_token']) { exit("Requête invalide."); }
  ```  
  *Critères :* Les formulaires sensibles ont un token et refusent la soumission sans token valide. *Métriques :* Aucun incident CSRF (peut tester via script automatisé).

- **CORS**.  
  *Description :* Restreindre l’origine des requêtes AJAX (si API). *Pourquoi :* Les navigateurs Web limitent les requêtes inter-domaines selon le header `Access-Control-Allow-Origin`. *Actions :* En PHP ou config serveur, définir :  
  ```php
  header("Access-Control-Allow-Origin: https://mon-domaine.com");
  ```  
  *Critères :* Test avec un domaine non autorisé (`curl -H 'Origin: http://evil.com'` => pas d’Access-Control dans la réponse). *Métriques :* Aucun site tiers non autorisé ne peut consommer l’API.

- **HTTPS partout**.  
  *Description :* Forcer le chiffrement TLS sur toutes les pages (certificat valide). *Pourquoi :* Sécuriser les données en transit (cookies, formulaires, etc.) et répondre aux exigences RGPD/PCI. *Actions :* Configurer redirection HTTP->HTTPS. Exemple `.htaccess` :  
  ```htaccess
  RewriteEngine On
  RewriteCond %{HTTPS} !=on
  RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
  ```  
  *Critères :* Tous les liens internes et externes pointent en HTTPS, certificat valide. *Métriques :* Aucune requête en HTTP (test automatisé).

- **Sauvegardes automatiques et restaurations**.  
  *Description :* Programmer des dumps réguliers de la DB, et tester leur restauration. *Pourquoi :* Pouvoir récupérer rapidement d’une perte de données. *Actions :*  
  - Dump : `mysqldump -u root -p mondb > backup.sql`  
  - Restore : `mysql -u root -p mondb < backup.sql`  
  Effectuer un dump complet au moins chaque jour et le stocker hors-site. *Critères :* Restaurer un dump se fait sans erreur (test sur environnement test). *Métriques :* Dernière sauvegarde datée (< 24h), test de restauration réussi.

- **Configurations sécurisées (RLS, etc.)**.  
  *Description :* Si usage de Postgres/Supabase, configurer la Row-Level Security (RLS) pour filtrer les lignes en fonction de l’utilisateur connecté. *Pourquoi :* Empêcher qu’un utilisateur lise/écrive des données qui ne lui appartiennent pas. *Actions :* Par ex. en SQL :  
  ```sql
  ALTER TABLE commandes ENABLE ROW LEVEL SECURITY;
  CREATE POLICY user_can_view ON commandes FOR SELECT 
    USING (user_id = auth.uid());
  ```  
  *Critères :* Les requêtes SQL en BDD respectent la RLS (vérifiable via Supabase dev). *Métriques :* Audit des politiques RLS.  

  - *Si MySQL/PHP natif :* MySQL ne gère pas la RLS nativement. Implémenter des conditions `WHERE user_id = ?` dans chaque requête et vérifier `$userId` côté serveur à chaque accès (back-end).

- **Clé publique vs clé secrète (API externes)**.  
  *Description :* Ne jamais exposer de clé secrète (ex. AWS secret, Stripe secret) dans le JavaScript ou HTML. *Pourquoi :* Une clé secrète exposée permettrait l’usurpation d’identité. *Actions :* Utiliser seulement les clés publiques côté client. Toutes les opérations sensibles passent par votre serveur. *Critères :* Revoir le code front-end pour s’assurer que seules les *public keys* y figurent. *Métriques :* Aucun secret trouvé dans le code du navigateur (outil de dev ou grep).

- **Mises à jour des dépendances**.  
  *Description :* Maintenir à jour librairies et frameworks (PHP, JS, etc.). *Pourquoi :* Bénéficier des correctifs de sécurité. *Actions :* Utiliser `composer update` et `npm update` régulièrement. Scanner les vulnérabilités (ex. `composer audit`, `npm audit`). *Critères :* Aucun package obsolète critique (selon `composer outdated`). *Métriques :* Rapport d’audit de sécurité (ex. Snyk).

- **En-têtes de sécurité HTTP**.  
  *Description :* Configurer au niveau serveur/reverse-proxy les en-têtes qui restreignent ce que le navigateur autorise : **CSP** (`Content-Security-Policy`, principale défense contre les XSS), **HSTS** (`Strict-Transport-Security`, force HTTPS), **X-Content-Type-Options: nosniff**, **X-Frame-Options** ou `frame-ancestors` (anti-clickjacking), **Referrer-Policy** (évite la fuite de paramètres d’URL), **Permissions-Policy** (caméra, micro, géolocalisation). *Pourquoi :* Sans ces en-têtes, une injection ou un clic détourné n’est contenu par rien côté navigateur — et la CSP se révèle souvent le seul moyen de limiter une XSS existante. *Actions :* Exemple Nginx :  
  ```nginx
  add_header X-Content-Type-Options "nosniff" always;
  add_header X-Frame-Options "SAMEORIGIN" always;
  add_header Referrer-Policy "strict-origin-when-cross-origin" always;
  add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
  add_header Content-Security-Policy "default-src 'self'; img-src 'self' data:; script-src 'self'" always;
  ```  
  Tester avec l’onglet Security de DevTools ou https://securityheaders.com ; durcir la CSP progressivement en mode `Content-Security-Report-Only` avant de la rendre bloquante. *Critères :* Les 6 en-têtes présents et cohérents sur toutes les pages ; CSP sans `unsafe-inline` pour les scripts (ou justification). *Métriques :* En-têtes vérifiés en CI (`curl -I`), violations CSP remontées dans les rapports.

  - *Si hébergement mutualisé sans accès config :* privilégier les en-têtes ajoutables via `.htaccess` (mod_headers) ou le CDN/pare-feu applicatif (WAF).  
  - *Si application JavaScript derrière CDN :* la CSP doit lister explicitement les origines des scripts et APIs tierces réellement utilisées.

- **Tokens d’authentification côté client : cookie HttpOnly, jamais `localStorage`**.  
  *Description :* Stocker les jetons de session/connexion dans un cookie `HttpOnly; Secure; SameSite` plutôt que dans `localStorage`/`sessionStorage`. *Pourquoi :* tout ce qui est en `localStorage` est lisible par le moindre script exécuté sur la page : en cas de XSS, le jeton est volé silencieusement ; un cookie `HttpOnly` ne l’est pas. *Actions :* Vérifier que le backend renvoie les tokens en cookie (`Set-Cookie` avec `HttpOnly`) et que le front ne fait pas `localStorage.setItem('token', …)`. *Critères :* Aucun token d’auth dans le stockage local/session ; cookie marqué HttpOnly + Secure + SameSite. *Métriques :* Inspection du stockage navigateur après connexion, balayage grep sur `localStorage`.

- **Uploads de fichiers sécurisés**.  
  *Description :* Contrôler type, extension, taille et destination de tout fichier téléversé par un utilisateur. *Pourquoi :* un upload non contrôlé permet l’exécution de code (`.php` déposé dans la racine web), le déni de service (fichier géant) ou le stockage de contenus illicites. *Actions :* Vérifier le type côté serveur via le contenu réel (`finfo` en PHP), pas seulement l’extension ; renommer le fichier à l’acceptation ; le stocker **hors de la racine web**, servi par un script qui contrôle les droits ; limiter la taille (`upload_max_filesize`) et rejeter les extensions exécutables. Exemple :  
  ```php
  $finfo = new finfo(FILEINFO_MIME_TYPE);
  $mime = $finfo->file($_FILES['fichier']['tmp_name']);
  $autorises = ['image/jpeg' => 'jpg', 'image/png' => 'png', 'application/pdf' => 'pdf'];
  if (!isset($autorises[$mime])) { exit("Type de fichier non autorisé."); }
  ```  
  *Critères :* Aucun fichier utilisateur exécutable accessible publiquement ; type vérifié sur le contenu. *Métriques :* Nombre d’uploads rejetés, scan antivirus/contenu si pertinent.

- **Supprimer les comptes, mots de passe et réglages par défaut**.  
  *Description :* Éliminer avant mise en ligne tout compte d’administration par défaut, mot de passe standard, utilisateur de démonstration ou réglage de configuration « usine » connu publiquement. *Pourquoi :* les paires admin/admin et les comptes de démo sont les premières tentées lors d’un scan automatisé ; un réglage par défaut (répertoire d’installation résiduel, interface d’admin exposée) se trouve en quelques secondes. *Actions :* Lister les comptes existants (`SELECT username FROM users;`), désactiver/supprimer les comptes démo, changer tous les mots de passe d’installation, supprimer les fichiers/dossiers d’installation résiduels (`install/`, `phpmyadmin/`), restreindre l’accès à l’admin (IP, 2FA). *Critères :* Aucun compte inconnu ni mot de passe par défaut ; interface d’administration non joignable depuis l’extérieur sans authentification. *Métriques :* Écarts relevés lors de la vérification pré-mise en ligne, comptes actifs listés.

# 4. Obligations légales et confidentialité

- **Politique de confidentialité**.  
  *Description :* Documenter quelles données vous collectez (nom, email, IP, cookies, etc.), pourquoi et comment vous les stockez. *Pourquoi :* RGPD/CNIL (pour EU) exigent transparence. *Actions :* Rédiger une page “Politique de confidentialité” décrivant traitements et droits (RGPD). Exemple :  
  > « Nous collectons votre e-mail pour la gestion de votre compte utilisateur. Vos données sont conservées 3 ans. Conformément au RGPD, vous avez un droit d’accès et de rectification. Contact : contact@monsite.com ».  
  *Critères :* Présence d’une page détaillée, accessible depuis le footer. *Métriques :* Conformité RGPD (audit CNIL, quiz interne).

- **CGU / Conditions Générales d’Utilisation**.  
  *Description :* Règles contractuelles d’utilisation (propriété du contenu, responsabilité, modération). *Pourquoi :* Encadrer légalement l’usage du site et dégager votre responsabilité. *Actions :* Rédiger des CGU simplifiées. Par ex. : interdiction d’utiliser le site pour spam, modalité de suspension de compte, etc. *Critères :* Les CGU sont publiées et acceptées par l’utilisateur (case à cocher). *Métriques :* Acceptation des CGU lors d’une inscription.

- **Conditions Générales de Vente (CGV)**.  
  *Description :* Si vous vendez en ligne (produits, abonnements), des CGV décrivant prix, paiement, livraison, rétractation (14 jours UE) sont obligatoires. *Pourquoi :* Code de la Consommation (ex. en France, loi Hamon) impose l’information précontractuelle. *Actions :* Mentionner le prix TTC, frais de port, politique de remboursement, droit de rétractation, délai de livraison. *Critères :* CGV conforme (par ex. décompte du prix total, délais de rétractation). *Métriques :* Validation légale (juridique interne ou aide externe).  

- **Mentions légales (identité)**.  
  *Description :* Fournir l’identité du site/éditeur (entreprise ou personne). *Pourquoi :* Obligations commerciales (France: loi LCEN). *Actions :* En page dédiée (footer), indiquer :  
  > *Exemple :* « [Nom de la société], SARL au capital de 10 000€, RCS Paris B 123 456 789, TVA intracom FR123456789. Siège : 10 rue du Web, Paris. Directeur de la publication : M. Dupont. Hébergeur : OVH. ».  
  *Critères :* Tous les champs demandés par la loi sont remplis (voir un guide CNIL/bpifrance). *Métriques :* Contrôle CNIL ou DPO (aucune sanction).

- **Cookies et consentement**.  
  *Description :* Avertir l’utilisateur de l’usage des cookies et recueillir son consentement pour les cookies non essentiels. *Pourquoi :* Directive ePrivacy / RGPD. *Actions :* Bannière cookie offrant acceptation/rejet égal. Conserver ce choix (cookie fonctionnel). Mentionner dans la politique de confidentialité : quels cookies (nécessaires vs publicité vs analytics). Exemple CNIL : « Les cookies de statistiques (ex. Google Analytics) nécessitent votre consentement. ».  
  *Critères :* Les cookies non-essentiels ne sont pas déposés avant consentement. Bouton Refuser est aussi simple qu’Accepter. *Métriques :* Rapport CNIL de conformité (ils exigent [12†L146-L155] que le refus soit aussi simple que l’acceptation).

- **Base légale (RGPD)**.  
  *Description :* Expliquer la raison légale du traitement (ex. consentement, contrat). *Pourquoi :* RGPD exige de préciser l’article 6 applicable. *Actions :* Par ex. si newsletter, vous pouvez écrire « base légale : consentement ». *Critères :* Mention d’une base légale (Article 6) dans la politique. *Métriques :* Vérification RGPD (ex. CNIL checklist).

- **Destinataires des données**.  
  *Description :* Indiquer qui aura accès aux données (éditeur, hébergeur, sous-traitants). *Pourquoi :* Transparence RGPD. *Actions :* Dans la politique, lister : “Vos données sont accessibles par le personnel de notre société et l’hébergeur X.” Ajouter si transfert hors UE (ex. AWS US). *Critères :* Clarité sur receveurs des données. *Métriques :* Aucune plainte sur transfert non-mentionné.

- **Droits des utilisateurs**.  
  *Description :* Informer sur les droits d’accès, rectification, suppression (CNIL RGPD) et comment les exercer (mail/poste). *Pourquoi :* Obligatoire (RGPD). *Actions :* Exemple mention : « Vous pouvez écrire à rgpd@monsite.com pour toute demande d’accès, rectification ou suppression de vos données personnelles. » *Critères :* Page RGPD avec contacts. *Métriques :* Capacité à répondre à une demande (test interne).

- **Accessibilité IA**.  
  *Description :* Rendre les contenus lisibles par des robots ou IA (ex. assistant vocal). *Pourquoi :* Améliorer le SEO et l’accessibilité générale. *Actions :* Documenter l’API, fournir un sitemap XML, garder le HTML sémantique (titres `<h1>`, listes `<ul>`, `<table>` avec `<th>`). *Critères :* Un sitemap existe, le HTML est propre. *Métriques :* Score de référencement, indexation par Google.

- **Exemptions selon pays / projet**.  
  *Si France/EU (ou pays non précisé)* : se conformer RGPD/CNIL (pages requises mentionnées ci-dessus). *Si USA ou autre* : adapter aux lois locales (ex. CCPA pour Californie, COPPA si enfants). Toujours mentionner que le pays n’est pas spécifié dans la politique (ex. « site US en cours de vérification légale »). *Si marketplace ou SaaS aux USA* : inclure politique de remboursements et conditions de service (exigences PCI pour paiements). *Si API publique* : proposer (ou non) une politique d’API usage/limitations, disclaimer pour usage des données.

- **Minimisation et durées de conservation**.  
  *Description :* Ne collecter que les données strictement nécessaires à la finalité déclarée (art. 5-1-c du RGPD) et fixer une durée de conservation par catégorie de données, appliquée en pratique. *Pourquoi :* La durée indéfinie est la première criticité relevée lors d’un contrôle CNIL ; conserver des données sans besoin augmente le coût d’une éventuelle violation. *Actions :* Lister les données collectées et leurs finalités ; pour chaque catégorie, écrire la durée (ex. « compte inactif supprimé après 3 ans », « commandes conservées 10 ans pour obligation comptable ») ; implémenter expiration ou anonymisation (cron, tâche planifiée). *Critères :* Chaque donnée collectée a une finalité et une durée documentées ; un mécanisme technique applique ces durées. *Métriques :* Nombre de catégories de données sans durée définie (cible : 0), volume de données purgées.

  - *Si base légale = consentement :* le consentement a une durée de vie limitée et un mécanisme de renouvellement.  
  - *Si SaaS/marketplace :* distinguer les durées contractuelles (facturation), marketing (consentement) et légales (comptabilité).

- **Droit à l’effacement réellement implémenté**.  
  *Description :* S’assurer qu’une demande de suppression de compte ou de données se traduit bien par la suppression (ou l’anonymisation) effective des données concernées — y compris dans les copies secondaires (bases de test, sauvegardes, exports). *Pourquoi :* documenter un droit dans la politique sans le faire fonctionner est une faute ; l’article 17 impose une réponse « sans retard injustifié » (1 mois au maximum, art. 12-3). *Actions :* Traçer la demande (formulaire ou email dédié), définir ce qui est supprimé, ce qui reste par obligation légale (factures) et comment, indiquer les sauvegardes ; tester le parcours complet : demander → vérifier que le compte ne se reconnecte plus → vérifier les tables. *Critères :* Un parcours de suppression documenté, testé, avec délai de réponse maîtrisé. *Métriques :* Délai moyen de traitement des demandes, nombre de demandes traitées dans le délai.

- **Notification de violation de données sous 72 heures**.  
  *Description :* Disposer d’une procédure écrite de notification des violations de données personnelles à la CNIL (et, si risque élevé, aux personnes concernées) dans les 72 heures après prise de connaissance (art. 33 RGPD). *Pourquoi :* ce délai est légal et court ; improviser en pleine incident conduit à le dépasser, ce qui est sanctionné indépendamment de la gravité réelle de la violation. *Actions :* Écrire la procédure : qui détecte, qui décide de la notifiableté, qui remplit le formulaire de la CNIL, où sont les contacts (CNIL, DPO, hébergeur) ; préparer à l’avance la liste des mesures de sécurité (art. 33-1-d) ; faire un exercice de simulation annuel. *Critères :* Procédure écrite, contacts à jour, testée au moins une fois. *Métriques :* Temps réel entre détection et notification lors des simulations (cible : < 24 h pour laisser marge aux 72 h).

  - *Si responsable de traitement :* obligation pleine et entière.  
  - *Si sous-traitant (art. 28) :* notifier le responsable de traitement « sans délai indû » — le délai contractuel doit être écrit avec le client.

- **Registre des traitements et mesures de sécurité documentées**.  
  *Description :* Tenir le registre des traitements exigé par l’art. 30 RGPD (qui traite quoi, pour quelle finalité, quels destinataires, quels transferts, combien de temps) et documenter les mesures de sécurité de l’art. 32 (chiffrement, contrôle d’accès, pseudonymisation quand pertinent). *Pourquoi :* c’est la première pièce demandée lors d’un contrôle ; sans lui, impossible de répondre à une demande d’accès ou d’évaluer ses propres risques. *Actions :* Modèle de registre sur le site de la CNIL ; le tenir à jour à chaque nouveau traitement ; y rattacher la politique de confidentialité pour que les deux restent cohérents. *Critères :* Registre existant, à jour, cohérent avec la politique publiée. *Métriques :* Dernière date de mise à jour, nombre de traitements recensés vs recensés réels.

  - *Si petit site/vitrine sans données sensibles :* le registre est allégé mais recommandé.  
  - *Si données de santé, de mineurs ou de geolocalisation :* traitements à risque → analyser l’impact (DPIA, art. 35) avec un professionnel.

# 5. Accessibilité

- **Texte alternatif sur toutes les images informatives**.  
  *Description :* Fournir un `alt` descriptif pour chaque image portant de l’information. *Pourquoi :* Les lecteurs d’écran liront ce texte pour les malvoyants. *Actions :* Exemple HTML : `<img src="diagramme.png" alt="Schéma du processus de réservation">`. *Critères :* Aucun `<img>` utile sans `alt` (évaluez `document.querySelectorAll('img:not([alt])')`). *Métriques :* Test d’outil a11y (axe.dev, Wave).

- **Alt vide pour images décoratives**.  
  *Description :* Si l’image est purement décorative, utiliser `alt=""`. *Pourquoi :* Les lecteurs d’écran sauront l’ignorer (pas d’info inutile). *Actions :* Exemple : `<img src="ornement.svg" alt="">`. *Critères :* Les images de design n’ont pas de texte alternatif informatif. *Métriques :* Aucune image décorative n’est lue par assistive tech.

- **Contraste du texte élevé (WCAG 2.2 / RGAA 4.1.2)**.  
  *Description :* Respecter les ratios de contraste : au moins 4.5:1 pour texte normal (3:1 pour texte large). C’est une exigence WCAG AA **et** du RGAA (critère 1.2). *Pourquoi :* Assurer lisibilité pour malvoyants ; obligation légale pour les sites publics et services français. *Actions :* Vérifier chaque couleur de texte/fond avec un outil (ex. https://webaim.org/resources/contrastchecker/). *Critères :* Tous les textes critiques passent le test 4.5:1. *Métriques :* Scores d’accessibilité (audit automatisé et revue manuelle).

- **Structure sémantique (titres logiques)**.  
  *Description :* Utiliser `<h1>` puis `<h2>`, etc. *Pourquoi :* Aider les utilisateurs et moteurs à comprendre la hiérarchie. *Actions :* Contrôler l’ordre des headings. *Critères :* Chaque page a un `<h1>` unique, les titres suivent l’ordre logique. *Métriques :* Audit d’accessibilité (ex. WAVE).

- **Labels explicites pour formulaires**.  
  *Description :* Associer chaque champ `<input>` à un `<label>`. *Pourquoi :* Facilite la navigation clavier et l’usage par lecteurs d’écran. *Actions :* Exemple HTML :  
  ```html
  <label for="email">Email :</label>
  <input id="email" name="email" type="email">
  ```  
  *Critères :* Aucun champ interactif sans label. *Métriques :* Tests clavier : focus lisible, instructions vocalisées.

- **Navigation au clavier et focus visible**.  
  *Description :* Toutes les fonctionnalités utilisables via clavier (tab, enter) avec indicateur de focus clair (ex. outline). *Pourquoi :* Nécessaire pour déficients moteurs et conformité WCAG. *Actions :* S’assurer que les liens et boutons ont un `:focus` discernable (par défaut outline ou class CSS). *Critères :* On peut atteindre tous les boutons/inputs avec Tab. *Métriques :* Tests manuels (naviguer sans souris).

- **Information pas uniquement par la couleur**.  
  *Description :* Ne pas coder l’information uniquement par une couleur (ex. « champ en rouge » sans explication textuelle). *Pourquoi :* Les daltoniens ou lecteurs d’écran le manqueraient. *Actions :* Ajouter texte ou icône (« * » pour champ requis). *Critères :* Les tests fonctionnent en niveaux de gris. *Métriques :* Outils de simulation daltonisme (Toptal).

- **Messages d’erreur accessibles**.  
  *Description :* Afficher un message clair et programmatique (ARIA) lors d’erreur de saisie. *Pourquoi :* L’utilisateur comprend l’erreur et corrige. *Actions :* Ex : `<span id="emailError" role="alert">Email invalide</span>` associé au champ. *Critères :* Les erreurs ont un label audible et visible. *Métriques :* Test de formulaire (submit sans email).

- **WCAG 2.2 et RGAA 4 comme références**.  
  *Description :* Viser la conformité WCAG 2.2 niveau AA (international) et, pour les sites publics ou français, le **RGAA 4.1** (référentiel français, aligné sur les WCAG). Les critères RGAA applicables incluent : 1.2 (contraste), 3.2 (titres), 4.1 (alternatives aux images), 6.1 (liens), 8.3 (formulaires), 11.1 (scripts). *Pourquoi :* Normes d’accessibilité recommandées internationalement (WCAG) et exigées légalement en France pour le secteur public (RGAA). *Actions :* Contrôler la conformité via des outils (ex. axe, Asqatas) **et** un audit manuel — un audit automatisé ne prouve pas la conformité. *Critères :* 0 erreur critique d’accessibilité ; pour le RGAA : taux de conformité mesuré selon la méthode officielle (échantillon de pages). *Métriques :* Score WCAG (axe) et taux de conformité RGAA si applicable. *Limite :* seule une référence par un professionnel ou un audit complet permet d’affirmer une conformité.

- **Contenus intelligibles par IA**.  
  *Description :* Structurer le contenu (JSON-LD pour données, sitemaps) pour que les moteurs/assistants l’interprètent. *Pourquoi :* Améliorer SEO et accès aux données publiques. *Actions :* Ajouter un sitemap.xml à la racine, structurer les données (schema.org) pour le produit, l’article, etc. *Critères :* Google Search Console valide le sitemap. *Métriques :* Présence des rich snippets dans les résultats de recherche.

- **Piège de focus et gestion des fenêtres modales**.  
  *Description :* Dans une modale ou un menu déroulant, confiner le focus à l’intérieur, le restituer au bouton déclencheur à la fermeture, et permettre la fermeture avec la touche Échap. *Pourquoi :* sans cela, un utilisateur au clavier ou au lecteur d’écran poursuit sa tabulation « derrière » la fenêtre invisible et se retrouve bloqué. *Actions :* Écouter `keydown` (Échap) et `Tab` (boucle de focus) ; utiliser `inert` ou `aria-modal="true"` ; rendre le fond non focalisable. *Critères :* Test au clavier : on ne sort jamais d’une modale ouverte, et le focus revient au déclencheur. *Métriques :* Modales testées au clavier / modales totales.

- **Mises à jour dynamiques annoncées (régions ARIA)**.  
  *Description :* Annoncer aux lecteurs d’écran les changements de contenu qui n’impliquent pas de rechargement (ajout au panier, résultat de filtre, message de succès) via des régions `role="status"` / `aria-live="polite"` et `role="alert"` pour les erreurs. *Pourquoi :* un utilisateur de lecteur d’écran reste sur un écran qui a visiblement changé sans que rien ne soit dit — il croit la page cassée. *Actions :* Placer une région live persistante ; y injecter le message (« Article ajouté au panier »). Exemple :  
  ```html
  <div id="retour" role="status" aria-live="polite"></div>
  ```  
  *Critères :* Chaque action dynamique importante produit une annonce audible/annoncée. *Métriques :* Actions dynamiques sans annonce (cible : 0 sur les parcours critiques).

- **Respect des préférences utilisateur (mouvement réduit, cibles, zoom)**.  
  *Description :* Adapter l’interface aux réglages système : animations réduites avec `prefers-reduced-motion`, aucune animation clignotant plus de 3 fois par seconde (risque de crise), cibles tactiles ≥ 44×44 px (WCAG 2.5.8), zoom jusqu’à 200 % sans perte de contenu, jamais de `user-scalable=no` dans le viewport. *Pourquoi :* ces critères concernent les personnes sensibles au mouvement, les malvoyants zoomant fortement et les utilisateurs mobiles ; ils sont aussi des critères WCAG 2.2 AA et RGAA. *Actions :* Médias queries CSS :  
  ```css
  @media (prefers-reduced-motion: reduce) {
    * { animation-duration: 0.01ms !important; transition-duration: 0.01ms !important; }
  }
  ```  
  Contrôler `<meta name="viewport" content="width=device-width, initial-scale=1">` (sans `maximum-scale=1` ni `user-scalable=no`), la taille des boutons et l’absence de `overflow: hidden` global bloquant le zoom. *Critères :* 0 élément clignotant ; cibles ≥ 44 px ; zoom 200 % fonctionnel. *Métriques :* Audit axe/WCAG sur ces critères, taille moyenne des cibles.

- **Lien d’évitement et ordre de tabulation logique**.  
  *Description :* Fournir un premier lien « Aller au contenu principal » et s’assurer que l’ordre de tabulation suit l’ordre visuel, sans élément focalisable hors écran ni atteignable seulement par la souris. *Pourquoi :* sur les pages avec un en-tête riche (menu, recherche, réseaux sociaux), chaque visiteur clavier refait le parcours du menu à chaque page ; l’ordre de tabulation incohérent désoriente. *Actions :* Lier le lien d’évitement au `<main>` avec un `id` ; vérifier avec Tab seul ; utiliser `tabindex` **positif** uniquement si contraint (et le questionner), jamais sur des éléments normaux. *Critères :* Le lien d’évitement est le premier élément focalisable ; la tabulation suit la lecture visuelle. *Métriques :* Nombre de tabulations avant d’atteindre le contenu principal (< 2).

- **Sous-titres et transcriptions pour les médias**.  
  *Description :* Fournir des sous-titres synchronisés aux vidéos avec parole et une transcription aux contenus audio/vidéo. *Pourquoi :* sans transcription, le contenu sonore est inaccessible aux personnes sourdes/malentendantes (et souvent illisible en environnement bruyant ou sans son) ; c’est un critère RGAA (8.x) et WCAG. *Actions :* Sous-titres VTT alignés (`<track kind="captions">`) — génération automatique **à relire**, transcription dans la page ou en téléchargement. *Critères :* Toutes les vidéos parlées ont des sous-titres relus ; les audios ont une transcription. *Métriques :* Médias sans sous-titre (cible : 0).

# 6. SEO (référencement naturel)

- **Balises `title` et `meta description` uniques et pertinentes**.  
  *Description :* Chaque page indexable a un `<title>` unique (≈ 50–60 caractères) incluant le mot-clé principal et le nom du site, et une `meta description` unique (≈ 150–160 caractères) qui donne envie de cliquer. *Pourquoi :* ce sont les deux éléments les plus visibles dans les résultats de recherche ; dupliqués ou vides (« Page d’accueil »), ils sont remplacés automatiquement par Google au détriment du taux de clic. *Actions :* Vérifier l’unicité (liste des titles dans le crawl : `site:votresite.fr`), les rendre dynamiques côté template ; ne pas bourrer de mots-clés. *Critères :* 0 doublon de title/description ; chaque page stratégique a les siens. *Métriques :* Nombre de titles/descriptions dupliqués ou tronqués, taux de clic moyen dans la Search Console.

- **`robots.txt` et `sitemap.xml` cohérents**.  
  *Description :* Un `robots.txt` qui n’interdit pas par erreur les ressources nécessaires ni le sitemap, et un `sitemap.xml` à jour qui ne contient que des URL indexables (status 200, sans paramètres d’unicité, sans pages noindex). *Pourquoi :* un `Disallow: /` mal placé (ou un CDN mal configuré) efface le site des résultats ; un sitemap avec des URL mortes gaspille le budget de crawl. *Actions :*  
  ```text
  User-agent: *
  Allow: /
  Disallow: /admin/
  Sitemap: https://votresite.fr/sitemap.xml
  ```  
  Vérifier dans Google Search Console (outil robots.txt, soumission du sitemap). *Critères :* Robots.txt testé sans erreur ; sitemap soumis et sans URL en erreur. *Métriques :* Pages explorées/crawlées, erreurs d’indexation dans la Search Console.

  - *Si site vitrine / statique :* générer le sitemap à chaque build.  
  - *Si application dynamique :* le régénérer à chaque ajout/suppression de contenu.  
  - *Ne jamais* bloquer les fichiers CSS/JS dans robots.txt : Google doit pouvoir rendre la page.

- **Balises canoniques contre le contenu dupliqué**.  
  *Description :* Une balise `<link rel="canonical">` par page qui pointe vers l’URL d’indexation préférée, y compris sur les variantes (paramètres de tri/filtre, versions `www`/sans, `http`/`https`). *Pourquoi :* le même contenu servi sur plusieurs URL divise la popularité des liens et Google choisit parfois la mauvaise version à indexer. *Actions :* Poser la canonical côté serveur/template ; sur les paramètres non structurants, ajouter `rel="canonical"` ou l’exclusion dans la Search Console ; éviter les pages filtrées indexables sans valeur. *Critères :* Chaque page indexable a une canonical auto-référente (sauf cas de duplication volontaire type paginée). *Métriques :* URL « non canoniques » détectées dans la Search Console.

- **URLs lisibles, stables et redirections 301**.  
  *Description :* URL courtes, en minuscules, avec des tirets, sans paramètres d’identifiant inutiles ; toute URL changée ou supprimée redirigée en **301** vers l’URL équivalente la plus proche — jamais vers la page d’accueil en bloc. *Pourquoi :* des URLs stables évitent de perdre le référencement acquis ; une redirection 301 transfère l’essentiel de la valeur ; un cycle de redirections ou un maillage de 404 cassé dégrade l’expérience et le crawl. *Actions :* Changer d’URL = 301 (`.htaccess`, framework ou CDN) ; tenir un tableau URL ancienne → nouvelle ; surveiller les 404 dans la Search Console et corriger les liens internes pointant dessus. *Critères :* Aucune redirection en chaîne (> 1 hop) ni boucle ; 404 internes corrigés. *Métriques :* Nombre de 404, longueur moyenne des chaînes de redirection.

- **Maillage interne avec ancres descriptives**.  
  *Description :* Lier les pages entre elles avec des liens dont le texte d’ancre décrit réellement la page cible (« guide RGPD pour e-commerce », pas « cliquez ici »). *Pourquoi :* les liens internes transmettent la popularité et le contexte sémantique aux pages cibles ; un site en silo mal relié laisse ses meilleures pages sans poids. *Actions :* Lister les pages stratégiques et leurs liens entrants ; ajouter depuis les pages à fort trafic vers les pages à convertir ; vérifier qu’aucune page importante n’est à plus de 3 clics de l’accueil. *Critères :* Toutes les pages stratégiques ont des liens entrants internes ; ancres explicites. *Métriques :* Profondeur de clic, nombre de liens entrants internes par page clé.

- **Contenu accessible aux robots (sans JavaScript indispensable)**.  
  *Description :* S’assurer que le contenu critique (titres, texte, liens, prix de base) existe dans le HTML servi, pas seulement après exécution d’un JavaScript lourd. *Pourquoi :* Googlebot exécute le JS mais avec des délais et des limites ; les assistants IA et autres crawlers ne l’exécutent souvent pas du tout — un contenu rendu uniquement en client est invisible pour eux. *Actions :* Privilégier le rendu côté serveur (SSR) ou la génération statique (SSG) pour les pages de contenu ; en SPA, prévoir un repli (prérendu, HTML minimal) ; comparer le HTML brut (Ctrl+U) avec l’affiché. *Critères :* Le HTML brut contient titres, texte principal et liens de navigation. *Métriques :* Différences HTML brut vs rendu sur les pages stratégiques.

- **Suivi et diagnostic du référencement**.  
  *Description :* Connecter la Search Console (et Analytics si utilisé) dès la mise en ligne et suivre indexation, positions et erreurs. *Pourquoi :* sans mesure, aucun SEO n’est vérifiable ; une désindexation ou une chute de trafic doit être visible en jours, pas en mois. *Actions :* Vérifier la propriété (balise DNS/HTML), soumettre le sitemap, suivre le rapport « Couverture » et « Expériences pages » ; analyser périodiquement les requêtes qui amènent du trafic. *Critères :* Propriété vérifiée, sitemap soumis, alertes configurées. *Métriques :* Pages indexées, clics/positions dans la Search Console, Core Web Vitals du champ.

  - *Si site vitrine :* suivi mensuel suffit.  
  - *Si marketplace/SaaS avec contenu public :* suivi hebdomadaire + alerte sur les chutes d’indexation.

- **Open Graph et Twitter Card**.  
  *Description :* Balises `og:title`, `og:description`, `og:image` (et équivalents `twitter:*`) pour contrôler l’aperçu du lien quand il est partagé (réseaux sociaux, messageries). *Pourquoi :* sans ces balises, la carte de partage affiche une image quelconque ou rien, ce qui réduit fortement le taux de clic des partages. *Actions :* Poser les balises dans le `<head>` ; image aux dimensions recommandées (1200×630) ; tester avec le débogueur d’aperçus de chaque plateforme. *Critères :* Aperçu correct et identique à l’image voulue sur au moins deux plateformes testées. *Métriques :* Taux de clic des partages (si mesuré).

# 7. Internationalisation (i18n)

- **Encodage UTF-8 de bout en bout**.  
  *Description :* Tout le chaîne de traitement des caractères doit être en UTF-8 : `<meta charset="utf-8">` en premier dans le `<head>`, connexion et tables de la base en `utf8mb4` (MySQL), en-têtes HTTP (`Content-Type: text/html; charset=utf-8`), fichiers de code en UTF-8. *Pourquoi :* un encodage hétérogène produit les fameuses grues (« Ã© » au lieu de « é »), casse les caractères accentués, emojis et alphabets non latins, et peut corrompre des données à l’écriture en base. *Actions :* Vérifier les trois points ci-dessus ; en PHP : `PDO::MYSQL_ATTR_INIT_COMMAND => "SET NAMES utf8mb4"` ; convertir les anciennes tables si besoin. *Critères :* Aucun caractère cassé à l’écran ni en base (test avec émojis, « œ », « à ç é »). *Métriques :* Nombre de « mojibake » détectés (revue ou script de scan).

- **Attribut `lang` exact sur chaque page**.  
  *Description :* Déclarer la langue réelle du contenu dans `<html lang="fr">` (et `xml:lang` si XHTML), avec un code de langue BCP 47 valide (`fr`, `fr-FR`, `en`). *Pourquoi :* les lecteurs d’écran utilisent cette déclaration pour choisir leur synthèse vocale — un `lang` absent ou faux rend la page illisible au son ; les moteurs l’utilisent aussi pour éviter les erreurs d’indexation (critère RGAA 8.1 / WCAG 3.1.1). *Actions :* Poser `lang` sur le template racine ; si le contenu mélange les langues (ex. citation en anglais), marquer les passages par `lang` sur l’élément. *Critères :* `lang` présent, valide et cohérent avec le contenu visible. *Métriques :* Pages sans `lang` (cible : 0), résultats d’audit RGAA critère 8.1.

- **Textes externalisés (pas de chaînes en dur)**.  
  *Description :* Regrouper toutes les chaînes affichées dans des fichiers de traduction (ex. `fr.php`, `en.php`, JSON i18n) plutôt que de les éparpiller dans le code. *Pourquoi :* chaque texte en dur dans le code rend toute future traduction coûteuse ; c’est aussi la garantie de ne pas afficher un message d’erreur technique brut à l’utilisateur. *Actions :* Fonction de traduction (`__('bonjour')`) ; un fichier par langue ; lister les textes visibles avec grep pour repérer les chaînes en dur résiduelles (`grep -r "echo \"" app/`). *Critères :* Aucune phrase affichée directement en dur dans le code applicatif. *Métriques :* Nombre de chaînes traduites vs textes visibles relevés en revue.

  - *Si projet monolingue définitif :* la structure par fichier de langue se prépare quand même, le coût initial est faible.  
  - *Si marketplace/SaaS multilingue :* obligation de passer par des fichiers de traduction dès le départ.

- **Traductions complètes et relues**.  
  *Description :* Quand plusieurs langues sont proposées, chaque langue doit être intégralement traduite (interface, emails, pages légales, messages d’erreur) et relue par un locuteur natif — sans mélange de langues ni texte traduit automatiquement sans relecture. *Pourquoi :* une interface à moitié traduite (bouton en anglais, erreur en français) décrédibilise plus qu’une page en une seule langue ; les emails transactionnels mal traduits font fuir les clients. *Actions :* Tableau de couverture des clés par langue (script qui compare les clés présentes dans `fr` et `en`) ; relecture humaine ; marquer explicitement les chaînes non traduites plutôt que d’afficher des clés brutes. *Critères :* 100 % des clés présentes dans toutes les langues proposées ; relecture effectuée. *Métriques :* Taux de couverture des clés, signalements d’utilisateurs sur les textes.

- **Formats localisés (dates, nombres, devises)**.  
  *Description :* Afficher dates, nombres, tailles et montants selon les conventions de la langue/du pays : `31/12/2026` vs `12/31/2026`, `1 234,56 €` vs `€1,234.56`, devise correcte, fuseaux horaires explicites. *Pourquoi :* un montant ambigu (`12.50` = 12,50 ou 12 50 ?) ou une date inversée provoque des erreurs de paiement et de confiance ; les montants sont juridiquement sensibles. *Actions :* utiliser les fonctions de localisation plutôt que `date()` ou `number_format()` en dur : PHP `IntlFormatter`/`NumberFormatter`, JS `Intl.NumberFormat`/`Intl.DateTimeFormat`. Exemple JS :  
  ```js
  new Intl.NumberFormat('fr-FR', { style: 'currency', currency: 'EUR' }).format(1234.5);
  // → 1 234,50 €
  ```  
  *Critères :* Aucun format de date/nombre en dur dans l’affichage ; devise et séparateurs cohérents avec la langue. *Métriques :* Nombre de formats codés en dur détectés, erreurs support liées aux formats.

- **Sélecteur de langue et mémoire de choix**.  
  *Description :* Proposer un moyen explicite de changer de langue (si plusieurs), mémoriser le choix (cookie, préférence de compte) et s’appuyer sur la détection (`Accept-Language`) sans imposer la détection automatique sans retour possible. *Pourquoi :* une langue imposée et non modifiable ferme la porte aux visiteurs ; la persistance évite de rechoisir à chaque page. *Actions :* Lister les langues disponibles dans l’en-tête ; comparer avec `Accept-Language` pour le premier visit ; stocker le choix ; prévoir une page 404/erreur dans la langue de l’utilisateur. *Critères :* Le changement de langue est accessible depuis chaque page ; le choix persiste. *Métriques :* Taux d’utilisation du sélecteur, taux de rebond par langue.

- **`hreflang` pour les versions linguistiques**.  
  *Description :* Sur un site multilingue, déclarer les équivalents de chaque page dans les autres langues avec `<link rel="alternate" hreflang="…">` (ou sitemap hreflang), avec retour réciproque et une langue par défaut `x-default`. *Pourquoi :* sans hreflang, Google affiche parfois la mauvaise langue dans les résultats ; les internautes atterrissent sur une page qu’ils ne comprennent pas. *Actions :* Écrire les paires/réseau d’alternates (chaque page listant toutes les autres, y compris elle-même) ; valider dans la Search Console. *Critères :* Déclarations réciproques, aucun code de langue invalide, `x-default` présent. *Métriques :* Erreurs hreflang dans la Search Console (cible : 0).

  - *Si monolingue :* non applicable.  
  - *Si une seule langue mais marchés différents (ex. France/Québec) :* distincts par `fr` et `fr-CA` seulement si le contenu diffère réellement.

- **Prévoir l’extension à d’autres langues (textes longs, RTL, polices)**.  
  *Description :* Concevoir l’interface pour qu’elle supporte d’autres langues sans casse : textes plus longs (l’allemand ≈ +30 %), traductions plus courtes, écritures droite-à-gauche (arabe, hébreu), alphabets différents et polices qui les couvrent. *Pourquoi :* un design calibré sur le français en dur (boutons à largeur fixe, colonnes étroites) explose au premier ajout de langue — et l’incohérence visuelle décourage les relances. *Actions :* Éviter les largeurs fixes sur les textes, utiliser `min-width` + troncature maîtrisée ; pour le RTL, préférer `dir="auto"` et les propriétés CSS logiques (`margin-inline-start` plutôt que `margin-left`) ; vérifier le rendu avec `lang="de"` et `dir="rtl"` en devtools. *Critères :* Mise en page intacte avec un texte long et avec `dir="rtl"`. *Métriques :* Nombre d’éléments cassés lors des tests de langue longue/RTL.

# 8. Mise en production

- **Nom de domaine personnalisé**.  
  *Description :* Héberger le site sur votre domaine propre (ex. `www.monsite.com`). *Pourquoi :* Professionnalisme, SEO et cookies valides. *Actions :* Acheter domaine, configurer DNS A/AAAA vers serveur/IP. *Critères :* `curl -I http://votredomaine` retourne le site en front. *Métriques :* Temps de propagation DNS, Alexa rank.

- **HTTPS obligatoire**.  
  *Description :* (Déjà cité en sécurité) S’assurer d’un certificat SSL valide pour le domaine. *Pourquoi :* Conformité, confiance utilisateur (HTTPS lock). *Actions :* Obtenir un certificat (Let’s Encrypt, OVH, Cloudflare). *Critères :* HTTPS actif sur toutes les pages (scan Qualys SSL Labs = A+). *Métriques :* Pourcentage de trafic en HTTPS.

- **Favicon**.  
  *Description :* Petite icône de site visible dans l’onglet navigateur. *Pourquoi :* Aspect professionnel. *Actions :* Placer `favicon.ico` et balise `<link rel="icon" href="/favicon.ico">` dans `<head>`. *Critères :* L’onglet du navigateur affiche l’icône du site. *Métriques :* Feedback UX.

- **Retirer les mentions démo et badges**.  
  *Description :* En production, enlever tout logo « Made with AI », cartes factices, paliers fictifs, etc. *Pourquoi :* Éviter l’image de site inachevé. *Actions :* Ex : supprimer les classes ou sections de demo (HTML et CSS). *Critères :* Aucun badge ou texte « Démo » visible. *Métriques :* Revue finale.

- **Pages d’erreur appropriées**.  
  *Description :* Prévoir des pages 404, 500 avec un message clair et navigation de secours. *Pourquoi :* Améliorer l’expérience quand quelque chose échoue. *Actions :* En PHP, par exemple, `http_response_code(404); include '404.php';`. *Critères :* Accéder à une URL inexistante affiche votre 404 (testé). *Métriques :* Taux 404 (logs) et retours utilisateurs.

- **Tester les formulaires et accès**.  
  *Description :* Vérifier inscription/connexion, réinitialisation. *Pourquoi :* Fondamental pour un site avec comptes. *Actions :* Créer des utilisateurs test, essayer les actions (vérification email si nécessaire). *Critères :* Workflow complet sans erreur (inscription -> email -> login). *Métriques :* Nombre d’erreurs lors de tests.

- **Tester les rôles et permissions**.  
  *Description :* Si roles (admin, user, guest), s’assurer que chaque rôle ne peut faire que ce qui lui est permis. *Pourquoi :* Éviter escalades de privilèges. *Actions :* En PHP, se connecter en tant que chaque rôle et tester fonctions admin (création, suppression). *Critères :* Utilisateur non admin ne voit pas/admin actions (vérifié). *Métriques :* Tests d’accès (ex: attempt to access `/admin` en mode user -> 403).

- **Tester les échecs d’API**.  
  *Description :* Simuler l’indisponibilité d’une API externe (timeout ou 5xx). *Pourquoi :* Voir si l’application gère gracieusement (timeout, message). *Actions :* Par exemple, pointer temporairement vers un endpoint erroné. *Critères :* L’application affiche un message d’erreur convivial (“Service indisponible, réessayer plus tard”) et ne plante pas. *Métriques :* Logs d’erreur, absence de crash.

- **Tester les paiements**.  
  *Description :* Pour un site marchand ou SaaS, tester paiements réussis, échoués, doubles paiements. *Pourquoi :* Éviter des transactions en double ou erreurs de compta. *Actions :* Utiliser une sandbox Stripe (clé test) et simuler une carte refusée, puis une valide. Exemple Stripe en PHP :  
  ```php
  \Stripe\Stripe::setApiKey($env['STRIPE_SECRET_KEY']);
  $intent = \Stripe\PaymentIntent::create(['amount'=>1000, 'currency'=>'eur']);
  ```  
  *Critères :* Les montants sont corrects, les e-mails de confirmation envoyés, aucun double prélèvement. *Métriques :* Nombre d’erreurs de paiement (0 en test).

- **Tester les sauvegardes et restauration**.  
  *Description :* Vérifier qu’une sauvegarde antérieure de la base peut être restaurée sans perte de données. *Pourquoi :* S’assurer que les dumps fonctionnent. *Actions :* Importer un dump `.sql` dans une base temporaire (`mysql < backup.sql`) puis vérifier la cohérence. *Critères :* Les données test (ou squelette) sont intactes. *Métriques :* Temps de restauration, intégrité des données (tests SQL).

- **Contrôler l’espace de stockage**.  
  *Description :* Vérifier que vous disposez de suffisamment d’espace (log, uploads, DB). *Pourquoi :* Éviter panne par saturation (DB “disk full”). *Actions :* Mettre en place des alertes (ex. script cron `df -h` + alerte mail à 90%). *Critères :* Alerte avant saturation. *Métriques :* Espace restant sur disque (/var, /home).

- **Surveillance / logs**.  
  *Description :* Avoir une solution de logs (fichier ou centralisé). *Pourquoi :* Tracer les erreurs en production. *Actions :* Utiliser Monolog en PHP ou configurer syslog. *Critères :* Les erreurs 500 sont loguées (ex. `/var/log/apache2/error.log`). *Métriques :* Nombre d’erreurs (devrait idéalement tendre vers 0).

- **Supprimer le mode debug/verbeux**.  
  *Description :* En production, ne pas afficher les erreurs détaillées (stack traces). *Pourquoi :* Fuites d’informations sensibles. *Actions :* PHP `display_errors=Off`, Laravel: `APP_DEBUG=false`. *Critères :* En cas d’erreur, utilisateur voit message générique (“Une erreur est survenue”) et l’admin log complet. *Métriques :* Aucune info sensible dans les réponses 5xx.

# 9. Tests

- **Tests de charge (load testing)**.  
  *Description :* Évaluer la capacité de l’infrastructure sous haute charge (outsser). *Pourquoi :* Trouver le goulet d’étranglement (CPU, mémoire). *Actions :* Outils : ApacheBench, `siege`, JMeter. Exemple :  
  ```
  ab -n 500 -c 50 http://localhost/endpoint
  ```  
  (500 requêtes total, 50 en simultané). *Critères :* Le serveur répond sans timeout, le temps par requête reste stable. *Métriques :* Requêtes/s supportées, temps moyen, 95<sup>e</sup> percentile du temps de réponse.

- **Tests de montée en charge**.  
  *Description :* Simuler une augmentation progressive du trafic. *Pourquoi :* Voir à quel point le site continue à fonctionner. *Actions :* Par ex. avec Siege :  
  ```
  siege -c50 -t2M http://monsite.com
  ```  
  (50 utilisateurs virtuels pendant 2 minutes). *Critères :* Pas d’erreur 5xx majeure (<5%). *Métriques :* Pourcentage d’erreurs sous pic de charge, CPU/Memory peak.

- **Tests de restauration**.  
  *Description :* Comme mentionné, tester régulièrement la restauration de backup. *Pourquoi :* S’assurer que la procédure fonctionne (sinon, backups inutiles). *Actions :* Effectuer des tests sur un clone de production. *Critères :* Aucune donnée inattendue après restauration. *Métriques :* Succès automatique (peut s’automatiser avec script).

- **Tests multi-navigateurs / multi-plateformes**.  
  *Description :* Vérifier sur Chrome, Firefox, Edge, Safari mobile, etc. *Pourquoi :* Différences CSS/JS entre navigateurs. *Actions :* Outils Selenium ou manuellement. *Critères :* Fonctionnalités principales (formulaires, navigation) identiques partout. *Métriques :* Nombre de bugs cross-browser détectés.

- **Tests unitaires / fonctionnels**.  
  *Description :* Pour back-end critique (auth, calculs), implémenter tests unitaires. *Pourquoi :* Garantir le fonctionnement des fonctions clés après modifs. *Actions :* PHPUnit pour PHP, Lighthouse pour audits front. *Critères :* Couverture minimale du code critique (ex. 80%). *Métriques :* Pourcentage de tests passés après chaque build.

- **Test de débordement de formulaire**.  
  *Description :* Tester la gestion d’entrées trop longues (ex. nom à 1000 chars). *Pourquoi :* Prévenir injections ou crash (buffer overflow). *Actions :* Soumettre via devtools un champ énorme. *Critères :* Serveur renvoie erreur contrôlée (p. ex. code 400) et n’écrase pas la DB. *Métriques :* Logs d’erreurs, validation de schéma (UTF8 safe).

- **Test des permissions de fichiers**.  
  *Description :* Vérifier que les fichiers uploadés (si public) n’exécutent pas de code. *Pourquoi :* Sécurité. *Actions :* Restreindre dossier uploads en non exécutable (Apache `Options -ExecCGI`). *Critères :* Un fichier `.php` uploadé dans `uploads/` ne doit pas être exécuté. *Métriques :* Aucune exécution PHP depuis uploads (test manuel).

- **Tests E2E des parcours critiques**.  
  *Description :* Automatiser de bout en bout les parcours qui portent le revenu ou l’usage principal : inscription → connexion, ajout au panier → paiement, envoi de formulaire → réception de l’email, publication de contenu. *Pourquoi :* les tests unitaires testent des fonctions, pas des parcours ; c’est au niveau des parcours que se cachent les régressions les plus coûteuses (un bouton de paiement qui ne se déclenche plus, un formulaire qui ne valide plus). *Actions :* Un scénario par parcours critique avec Playwright/Cypress (ou Selenium), exécuté sur la version de staging à chaque release ; assertions sur le résultat visible (« Merci de votre commande »), pas seulement sur l’absence d’erreur JS. *Critères :* Chaque parcours critique a au moins un test E2E vert sur la release. *Métriques :* Couverture des parcours critiques, taux d’échec des E2E en CI.

- **Exécution automatique des contrôles en CI à chaque changement**.  
  *Description :* Faire tourner tests, linters, contrôles de dépendances et audit de sécurité automatiquement sur chaque *commit* ou *pull request*, avec un merge impossible tant que la CI est rouge. *Pourquoi :* un test qui n’est exécuté « de temps en temps » ne protège de rien ; c’est l’exécution systématique qui empêche la régression de rejoindre la branche principale. *Actions :* Pipeline (GitHub Actions, GitLab CI, etc.) : lint → tests unitaires → E2E → `npm audit`/`composer audit` → échec du build si budget de performance dépassé ; garder la CI rapide (< 10 min) pour qu’elle soit respectée. *Critères :* Aucun merge avec CI rouge ; couverture minimale tenue. *Métriques :* Pourcentage de builds avec CI exécutée, temps moyen de la CI, régressions bloquées.

- **Vérification sur mobile et réseaux lents**.  
  *Description :* Tester les parcours critiques dans une fenêtre de taille téléphone (≈ 375 px) et avec la simulation de réseau lent/3G des DevTools, pas uniquement sur un grand écran en fibre. *Pourquoi :* la majorité du trafic public vient du mobile ; un formulaire utilisable uniquement sur desktop, ou qui met 15 s en 4G dégradée, perd la majeure partie de sa conversion — les bugs de débordement horizontal et de modales hors écran n’apparaissent qu’à cette taille. *Actions :* Mode responsive des DevTools + throttling (« Slow 4G ») ; tester le scroll, les cibles tactiles et la saisie (clavier qui recouvre le champ) ; si possible, un passage sur un vrai appareil pour les écrans critiques. *Critères :* Parcours critiques réalisables au doigt, sans défilement horizontal, sur réseau lent simulé. *Métriques :* Taux de conversion mobile vs desktop (signal indirect), erreurs trouvées par session de test mobile.

# 10. Surveillance (Monitoring)

- **Détection d’interruption de service**.  
  *Description :* Être alerté en cas de downtime (ping, connexion DB, HTTP). *Pourquoi :* Réagir rapidement (SLA). *Actions :* Configurer Pingdom/UptimeRobot pour tester un URL toutes les 5 min. *Critères :* En cas de downtime, notification immédiate. *Métriques :* Temps de rétablissement, nombre de pannes (should be minimal).

- **Surveillance des performances**.  
  *Description :* Mesurer en continu temps de réponse (APM). *Pourquoi :* Détecter régressions/ressources limite. *Actions :* Utiliser New Relic, Datadog, ou Prometheus + Grafana. *Critères :* Alertes si latence > seuil (ex. 1s). *Métriques :* Taux d’erreur (%), temps de réponse moyen, 95<sup>e</sup> percent.

- **Suivi de la consommation API externe**.  
  *Description :* Si vous payez les API à l’usage (Stripe, Algolia, etc.), surveiller le quota/dépense. *Pourquoi :* Éviter surcoût surprise. *Actions :* Automatiser le monitoring via les dashboards des fournisseurs (webhooks usage). *Critères :* Alerte envoyée quand approche 90% du quota mensuel. *Métriques :* % de budget API consommé, coût mensuel actuel.

- **Monitoring sécurité**.  
  *Description :* Surveiller les logs pour tentatives d’attaque (fail2ban, audit logs). *Pourquoi :* Détecter brute force, injection suspecte. *Actions :* Installer un IDS/IPS (ex. fail2ban sur SSH/PHP). *Critères :* Capturer et bloquer IP abusives. *Métriques :* Nombre d’IP bannies, fréquence des tentatives.

- **Espace libre (logs, DB)**.  
  *Description :* Alertes si la base de données ou logs saturent le disque. *Pourquoi :* Éviter panne. *Actions :* Script `cron` avec `df -h`, alerte si >90%. *Critères :* Alerte envoyée en cas de seuil franchi. *Métriques :* Utilisation disque (%).

- **Journalisation des erreurs**.  
  *Description :* Toutes les erreurs critique remontent dans un log (ex. `error.log`). *Pourquoi :* Permet d’analyser post-mortem. *Actions :* Configurer `error_log("...")`. *Critères :* Chaque exception PHP non gérée crée une entrée log. *Métriques :* Nombre d’erreurs 500, leur texte (outil de log aggregation).

- **Alertes personnalisées**.  
  *Description :* Automatiser les alertes en cas d’anomalies (p. ex. trop d’erreurs 5xx sur un endpoint). *Pourquoi :* Action rapide avant crash total. *Actions :* Sur Datadog/NewRelic, définir une alerte sur le ratio d’erreurs. *Critères :* Recevoir un e-mail/Slack quand le seuil est dépassé. *Métriques :* Temps moyen entre l’erreur et l’alerte.

# 11. Priorisation (P0–P3) par projet

Nous classons chaque tâche en **priorité** P0–P3. Chaque niveau a une **sémantique opérationnelle** : il déclenche une action particulière, pas seulement un étiquetage.

| Niveau | Définition | Ce qu’il déclenche | Exemples |
|:------:|------------|--------------------|----------|
| **P0** — Critique | Sécurité, intégrité des données, ou conformité légale de base non négociable. Un P0 ouvert empêche la mise en production (**bloquant**). | Correction **immédiate** avant tout déploiement ; validation humaine obligatoire. | Auth, autorisations, hachage des mots de passe, protection des paiements, secrets, HTTPS, notification de violation RGPD (72 h). |
| **P1** — Fiabilité | Ce qui fait tenir le système dans la durée : erreurs, sauvegardes, performance de base, accessibilité minimale. | À traiter **avant la mise en production** sauf justification écrite et risque accepté. | Gestion des erreurs, backups/restauration testés, logs, rate limiting, index DB, Core Web Vitals, E2E des parcours critiques. |
| **P2** — Optimisation | Qualité et confort : cache, optimisations, accessibilité complète, SEO de base. | À planifier dans les itérations suivant le lancement. | Cache, compression, contraste WCAG/RGAA, navigation clavier, images optimisées, sitemap/canonical. |
| **P3** — Finitions | Image, branding, contenus et détails de polish. | Quand le reste est fait ; sans bloquer quoi que ce soit. | Favicon, branding, pages tuto, docs API, tests multi-navigateurs approfondis. |

> **Règle de décision :** un projet avec des P0 ouverts est en **NO-GO**. Les P1 ouverts doivent être explicitement acceptés (par qui, jusqu’à quand) ; tout le reste est du backlog.

L’ordre varie selon le projet :

| Priorité | Catégorie                 | Application PHP/MySQL | Marketplace (ex. Guide Tower)    | Site vitrine   | SaaS (abonnements) | API publique  |
|:--------:|:-------------------------:|:---------------------:|:--------------------------------:|:--------------:|:------------------:|:-------------:|
| **P0**   | Sécurité/Intégrité        | Authentification, autorisations, protection paiements, validation serveur. HTTPS, mots de passe hachés. | Même + vérification multi-acteurs (guide vs client), commission. Obligations KYC/Paiement. | SSL, aucune info perso collectée (si formulaire). | Auth renforcée (2FA?), conformité PCI, RGPD (données clients). | API keys OAuth/JWT, quotas, pas de CORS sauvage. |
| **P1**   | Fiabilité / Backend       | Gestion des erreurs, sauvegardes/restauration testées, logs, rate limit, index sur DB, pagination. | Comme ci-dessus + disponibilité 24/7 (marketplace clé). | Basic (pages statiques chargent vite). *Hébergement fiable*. | High-availability, backup DB, double paiement bloqué, fiabilité paiement. | Uptime API critique, mises à jour versionnées. |
| **P2**   | Performance / Accessibilité | Cache (APCu, HTTP), compression gzip, contraste WCAG, navigation clavier, alt text. | De même + optimisation filtres recherche, responsive. | Optimisation SEO (title, sitemap, canonical) et temps de chargement, contraste et navigation clavier. | Performance élastique (autoscaling), tableau de bord usage (UI accessible). | Documentation API (Swagger), code example, CORS configuré. |
| **P3**   | Finitions / Lancement     | Favicon, nom de domaine, pages légales finales (CGU, CGV, cookies), tests multi-navigateurs, suppression de mentions “demo”. | Graphismes finaux, pages tuto (guide usage), UI/UX revue. | Design complet, Open Graph, contenus multilingues optionnels. | Page d’accueil soignée, support client (FAQ, mail), branding complet. | Portail développeur, exemples de requêtes, support (GitHub issues). |

## Tableaux comparatifs

### Tâches clés par priorité (extrait)

| Tâche / Catégorie                      | Priorité | Justification clé                        |
|----------------------------------------|:--------:|:----------------------------------------|
| Authentification & sessions sécurisées | P0       | Garantie de l’identité et sécurité des données. |
| Contrôle permissions (serveur)         | P0       | Empêcher accès non autorisé.           |
| Paiements & CGV (e-commerce/SaaS)      | P0       | Obligations légales et financières.    |
| Validation & Protection données        | P0       | Prévenir injections et fuites.         |
| Backup DB & restauration testée        | P0       | Récupération après incident.           |
| HTTPS & headers sécurité               | P0       | Chiffrement et conformité.            |
| Configuration DB optimisée (index) | P1       | Réduction du temps de requête.         |
| Gestion erreurs utilisateur            | P1       | UX améliorée, éviter blocage.         |
| Limitations requêtes / taux API  | P1       | Prévenir surcharge et abus.           |
| Tests E2E des parcours critiques | P1       | Détecte les régressions de parcours (paiement, connexion) avant la release. |
| Internationalisation (UTF-8, `lang`, formats) | P1 | Intégrité des caractères, base de toute traduction et du SEO multilingue. |
| Exécution des tests en CI à chaque changement | P1 | Empêche la régression d’atteindre la branche principale. |
| SEO on-page (title, canonical, sitemap) | P2   | Visibilité et indexation correctes.   |
| Compression / cache                    | P2       | Optimisation performance (mise en cache). |
| Accessibilité (WCAG contrast)   | P2       | Conformité légale et confort.         |
| Mobile/responsive design               | P2       | Expérience utilisateur homogène.       |
| Documentation & démos réelles         | P3       | Crédibilité et formation utilisateur. |
| Favicon, branding final               | P3       | Image professionnelle.                |
| Tests multi-navigateurs               | P3       | Vérification compatibilité.            |

### Tâches clés par type de projet (extrait)

| Fonctionnalité / Exigence                | PHP/MySQL | Marketplace (Guide Tower) | Site vitrine statique       | SaaS récurrents            | API publique              |
|-----------------------------------------|:---------:|:-------------------------:|:---------------------------:|:-------------------------:|:--------------------------:|
| **Authentification**                    | Formulaire, mot de passe (P0) | Formulaire + rôles (clients, guides, admin) (P0) | Si compte admin (P0/P3) | Compte + 2FA (P0)           | Clé API / OAuth (P0)      |
| **RGPD/Cookies**                        | RGPD France (P0), cookies analytics (consent) | Même + consentement guides (P0) | Cookies analytics (consent) (P1) | RGPD & PSP compliance (P0) | Terme d’API usage (P1)    |
| **Paiements / CGV**                     | Simple paiement unique (CGV)(P0) | Paiements multi-roles, commissions (P0) | N/A (ou promotions statiques, P3) | Abonnements, rebilling, rétractation (P0) | N/A (voir contrats usage) |
| **Hébergement / Infra**                 | Hébergement mutualisé ou VPS (P1) | Haute dispo (cloud) (P1) | Hébergement statique (Netlify) (P3) | Auto-scaling (P1)          | Échelle cloud, CDN (P1)   |
| **UI/UX Design**                        | Standard Bootstrap (P2)     | Couleurs guides/tourisme (P2) | Branding corporate (P3)     | UI pro de SaaS (P2)        | Portail minimal (P3)      |
| **Accessibilité**                       | WCAG AA (P2)              | Même + recherche accessible (P2) | Normes ministérielles (P2)   | WCAG AA (P2)              | Documentation claire (P2) |

```mermaid
flowchart LR
  A[Code commit sur Git] --> B(CI/CD Pipeline)
  B --> C{Tests automatisés}
  C -- Tests passés --> D[Déploiement sur *Staging*]
  C -- Echec --> E[Notification aux devs]
  D --> F(Revue manuelle / recette)
  F --> |Approuvé| G[Déploiement sur *Production*]
  F --> |Rejeté| B
  G --> H(Surveillance en continu)
  H --> I[Alertes sur incidents/erreurs]
```

```mermaid
timeline
  title Lancement en production - Phases clés
  Phase 1 : Planification et spécifications
  Phase 2 : Développement et tests unitaires
  Phase 3 : Tests d'intégration et QA interne
  Phase 4 : Recette utilisateur (UAT)
  Phase 5 : Ajustements et validation finale
  Phase 6 : Préparation du déploiement (DNS, certificats, sauvegardes)
  Phase 7 : Mise en production
  Phase 8 : Surveillance et support post-déploiement
```

<!-- **Checklist finale (à copier-coller)**:
- [ ] Limiter le nombre de requêtes par utilisateur (rate limiting et vérification serveur).
- [ ] Mettre en place le caching (HTTP Cache-Control, données en mémoire).
- [ ] Afficher un indicateur de chargement (spinner) lors des requêtes longues.
- [ ] Prévoir un état 'aucune donnée' pour les listes vides (message utilisateur).
- [ ] Paginer les longues listes de résultats.
- [ ] Ajouter des index sur les colonnes fréquemment recherchées.
- [ ] Compresser les fichiers (images, assets, Gzip).
- [ ] Tester le site sous forte charge (ab, siege).
- [ ] Mettre à jour le domaine personnalisé et activer HTTPS.
- [ ] Ajouter un favicon et soigner l’identité de marque (couleurs, polices).
- [ ] Écrire et intégrer la Politique de Confidentialité (RGPD).
- [ ] Écrire et intégrer les CGU/CGV selon l’activité.
- [ ] Installer un CMP/cookie banner et demander le consentement (non-essentiels).
- [ ] Rédiger les mentions légales (identité, RCS, TVA) et les afficher.
- [ ] Hacher les mots de passe (password_hash, Argon2id).
- [ ] Ne jamais stocker de clés secrètes dans le code (utiliser .env et gitignore).
- [ ] Vérifier les autorisations coté serveur (RBAC).
- [ ] Utiliser des requêtes préparées (PDO) pour le SQL.
- [ ] Configurer les sessions (HTTPS, HttpOnly, expiration).
- [ ] Implémenter CSRF tokens sur les formulaires sensibles.
- [ ] Valider et filtrer toutes les données utilisateur (XSS/Injection).
- [ ] Maintenir les dépendances à jour (composer/npm audit).
- [ ] Tester les cas d’erreur (API indispo, upload trop grand, DB hors ligne).
- [ ] Automatiser les sauvegardes DB et tester la restauration.
- [ ] Vérifier que l'Upload de fichiers ne permet pas d'exécution de code.
- [ ] Réaliser les tests de régression multi-navigateurs et multi-plateformes.
- [ ] Tester le workflow de paiement (réussi, échoué, annulations récurrentes).
- [ ] Désactiver l’affichage des erreurs détaillées en production.
- [ ] Mettre en cache les données statiques et peu volatiles.
- [ ] Appliquer le contraste WCAG AA (4.5:1) pour tous les textes.
- [ ] Ajouter du texte alternatif pertinent à toutes les images.
- [ ] Vérifier la navigation clavier et le focus visible sur tous les éléments.
- [ ] Tester l'accessibilité avec un outil (axe, Wave).
- [ ] Planifier les tests de charge et de montée en charge.
- [ ] Configurer les en-têtes de sécurité HTTP (CSP, HSTS, nosniff, X-Frame-Options, Referrer-Policy).
- [ ] Stocker les tokens d’authentification en cookie HttpOnly (jamais en localStorage).
- [ ] Supprimer les comptes, mots de passe et réglages par défaut.
- [ ] Mettre en place la notification de violation de données sous 72 h (procédure écrite, contacts à jour).
- [ ] Exécuter tests et linters en CI à chaque commit/PR (merge bloqué si la CI est rouge).
- [ ] Automatiser un test E2E par parcours critique (connexion, paiement, formulaire).
- [ ] Vérifier l’UTF-8 de bout en bout et l’attribut `lang` de chaque page.
- [ ] Mesurer les Core Web Vitals sur les pages critiques (LCP ≤ 2,5 s, INP ≤ 200 ms, CLS < 0,1).
- [ ] Renseigner `title` et `meta description` uniques, `sitemap.xml` et `robots.txt` cohérents.
- [ ] Mettre en place une surveillance (uptime, alertes erreurs).
- [ ] Mettre en place l'envoi d'alertes en cas de crash ou d’erreurs 500 importantes.
- [ ] Prévoir la journalisation des erreurs utilisateur (pour analyser les incidents).
- [ ] Nettoyer tout élément de démo ("Demo", "Fake") avant mise en prod. -->