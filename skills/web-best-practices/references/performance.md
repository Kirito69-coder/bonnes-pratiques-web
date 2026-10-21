# Checklist performance

Cette référence liste les points de performance à vérifier lors d’une revue.

## Core Web Vitals (seuils Google)

Objectifs « bons » sur mobile, mesurés sur les pages critiques (Lighthouse / PageSpeed Insights / WebPageTest) :

- **LCP** (Largest Contentful Paint) ≤ 2,5 s : le plus gros élément visible se charge vite.
- **INP** (Interaction to Next Paint) ≤ 200 ms : la page réagit vite aux interactions.
- **CLS** (Cumulative Layout Shift) < 0,1 : pas de sauts de mise en page pendant le chargement.
- **TTFB** (Time to First Byte) : réponse serveur rapide (idéalement < 800 ms).

## Budget de performance

- Un budget de poids de page est défini (par ex. < 1 500 Ko au chargement complet, idéalement < 500 Ko pour une page vitrine).
- Le budget est vérifié en CI quand c’est possible (échec du build si dépassé).

## Réseau et chargement

- Compression texte activée : **Brotli** ou Gzip (HTML, CSS, JS).
- HTTP/2 (ou HTTP/3) activé quand l’hébergement le permet.
- Ressources critiques : `preload` / `preconnect` pour les origines tierces essentielles.
- Scripts tiers chargés en asynchrone (`defer`/`async`) pour ne pas bloquer le rendu.
- Polices web : formats modernes (WOFF2), `font-display: swap`, nombre limité de variantes.

## Caching

- En-têtes HTTP utilisés quand pertinent (`Cache-Control`, `ETag`, `Last-Modified`).
- Cache applicatif (APCu, Redis) pour les données fréquemment lues et peu volatiles.

## Images

- Formats modernes : **WebP** ou **AVIF** avec repli (élément `<picture>`), au lieu de JPEG/PNG bruts.
- Images responsives : `srcset` et `sizes` pour servir la bonne résolution.
- Attributs `width`/`height` (ou `aspect-ratio`) posés sur les `<img>` : réserve l’espace et évite le CLS.
- `loading="lazy"` sur les images hors du premier écran (**pas** sur l’image hero, qui reste prioritaire).
- Images dimensionnées à leur taille d’affichage (servir une image de 2000 px dans un conteneur de 400 px gaspille la bande passante et le LCP).
- Poids par image raisonnable ; formats lourds (GIF animés) convertis en vidéo si volumineux.

## Payloads et requêtes

- JS, CSS et images compressés et raisonnables en production.
- Requêtes et réponses limitées aux données nécessaires (pas de `SELECT *` ni de payload JSON surdimensionné).
- Limites de taille d’upload configurées quand pertinent.
- Nombre de requêtes HTTP maîtrisé (assemblage, sprites/icônes SVG, HTTP/2).

## Rendu et responsive

- Pages utilisables sur mobile et desktop sans défilement horizontal.
- Animations et effets décoratifs contrôlés (pas excessifs).

## Base de données

- Index sur les colonnes fréquemment filtrées ou triées (vérifier avec `EXPLAIN`).
- Pagination pour les grands jeux de résultats.
- Requêtes inutiles et recalculs réduits.

## Charge et disponibilité

- Tests de charge envisagés au niveau de trafic attendu (`ab`, `siege`, JMeter).
- Supervision configurée pour détecter ralentissements et interruptions.

## Références

- Outils : Lighthouse, WebPageTest, PageSpeed Insights ; `ab`, `siege` pour la charge.
- CDN et hébergement statique pour le contenu statique quand pertinent.

## Notes

- Les objectifs de performance doivent refléter l’audience et la stack attendues.
- Mesurer avant **et** après chaque changement : une mesure unique ne prouve rien.
- Un bon score Lighthouse sur desktop ne dit rien du comportement sur mobile réel.
