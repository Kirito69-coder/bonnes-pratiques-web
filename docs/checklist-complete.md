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
  *Description :* Utiliser un outil de monitoring (Pingdom, UptimeRobot, Datadog) pour détecter les pannes ou lenteurs. *Pourquoi :* Réagir rapidement en cas d’incident. *Actions :* Configurer alertes (e-mail/SMS) sur indisponibilité (>5 minutes) ou dépassement de latence. *Critères :* Alertes fonctionnelles, logs de monitoring accessibles. *Métriques :* Taux de disponibilité (uptime), temps de réponse moyen.

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
  *Description :* Maintenir à jour librairies et frameworks (PHP, JS, etc.). *Pourquoi :* Bénéficier des correctifs de sécurité. *Actions :* Utiliser `composer update` et `npm update` régulièrement. Scanner les vulnérabilités (ex. `composer audit`, `npm audit`). *Critères :* Aucun package obsolète critique (selon `composer outdated`). *Métriques :* Rapport d’audit de sécurité (ex. Snyk).

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
  *Si France/EU (ou pays non précisé)* : se conformer RGPD/CNIL (pages requises mentionnées ci-dessus). *Si USA ou autre* : adapter aux lois locales (ex. CCPA pour Californie, COPPA si enfants). Toujours mentionner que le pays n’est pas spécifié dans la politique (ex. « site US en cours de vérification légale »). *Si marketplace ou SaaS aux USA* : inclure politique de remboursements et conditions de service (exigences PCI pour paiements). *Si API publique* : proposer (ou non) une politique d’API usage/limitations, disclaimer pour usage des données.

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
  *Description :* Structurer le contenu (JSON-LD pour données, sitemaps) pour que les moteurs/assistants l’interprètent. *Pourquoi :* Améliorer SEO et accès aux données publiques. *Actions :* Ajouter un sitemap.xml à la racine, structurer les données (schema.org) pour le produit, l’article, etc. *Critères :* Google Search Console valide le sitemap. *Métriques :* Présence des rich snippets dans les résultats de recherche.

# 6. Mise en production

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

# 7. Tests

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
  *Description :* Vérifier que les fichiers uploadés (si public) n’exécutent pas de code. *Pourquoi :* Sécurité. *Actions :* Restreindre dossier uploads en non exécutable (Apache `Options -ExecCGI`). *Critères :* Un fichier `.php` uploadé dans `uploads/` ne doit pas être exécuté. *Métriques :* Aucune exécution PHP depuis uploads (test manuel).

# 8. Surveillance (Monitoring)

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

# 9. Priorisation (P0–P3) par projet

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
| **P2**   | Performance / Accessibilité | Cache (APCu, HTTP), compression gzip, contraste WCAG, navigation clavier, alt text. | De même + optimisation filtres recherche, responsive. | Optimization SEO/temps de chargement, ARIA où nécessaire. | Performance élastique (autoscaling), tableau de bord usage (UI accessible). | Documentation API (Swagger), code example, CORS configuré. |
| **P3**   | Finitions / Lancement     | Favicon, nom de domaine, pages légales finales (CGU, CGV, cookies), tests multi-navigateurs, suppression de mentions “demo”. | Graphismes finaux, pages tuto (guide usage), UI/UX revue. | SEO (sitemap, robots.txt), design complet, alternatives de contenu (langues?). | Page d’accueil soignée, support client (FAQ, mail), branding complet. | Portail développeur, exemples de requêtes, support (GitHub issues). |

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
- [ ] Mettre en place une surveillance (uptime, alertes erreurs).
- [ ] Mettre en place l'envoi d'alertes en cas de crash ou d’erreurs 500 importantes.
- [ ] Prévoir la journalisation des erreurs utilisateur (pour analyser les incidents).
- [ ] Nettoyer tout élément de démo ("Demo", "Fake") avant mise en prod. -->