# Checklist accessibilité

Cette référence liste les points d’accessibilité à vérifier lors d’une revue. Elle s’appuie sur les principes des **WCAG 2.2** (niveau AA visé) et, pour la France et le secteur public, sur le **RGAA 4.1**. En cas d’exigence de conformité, seules une source officielle ou un audit complet font foi.

## Images et alternatives

- Images informatives : `alt` descriptif et pertinent.
- Images décoratives : `alt=""` (ignorées par les lecteurs d’écran).
- Les images de texte sont évitées (sauf logos).

## Contraste et couleurs

- Contraste du texte : au moins 4,5:1 pour le texte normal, 3:1 pour le texte large (WCAG AA — critère RGAA 1.2).
- Indicateurs de focus et éléments interactifs discernables.
- L’information n’est jamais transmise **uniquement** par la couleur (texte ou icône en complément).

## Structure et sémantique

- Hiérarchie de titres logique (`h1` puis `h2`, sans saut de niveau), un `h1` par page.
- Landmarks et HTML sémantique utilisés (`main`, `nav`, `header`, `footer`).
- Un seul landmark `main` par page ; plusieurs `nav` étiquetés si présents.

## Formulaires

- Chaque champ interactif a un `<label>` associé.
- Erreurs identifiables visuellement **et** programmatiquement (`role="alert"`, `aria-describedby`).
- Un seul label par champ.

## Clavier et focus

- Fonctionnalités principales atteignables au clavier (tabulation logique).
- Focus visible et prévisible (ne jamais supprimer l’outline sans alternative).
- Gestion du focus lors des interactions dynamiques (modales : piège de focus, fermeture par Échap).
- Lien d’évitement (« Aller au contenu ») présent en tête de page.
- L’élément focalisé n’est jamais masqué par des en-tres/pieds de page ou bandeaux collants.

## A11y avancée (régions dynamiques et confort)

- Mises à jour de contenu dynamique annoncées via des régions ARIA (`aria-live`, `role="status"`, `role="alert"`).
- Respect de `prefers-reduced-motion` : animations réduites ou supprimées selon la préférence utilisateur.
- Aucun contenu qui clignote plus de 3 fois par seconde (risque de crise — critère sévère).
- Cibles tactiles d’au moins 44×44 px (WCAG 2.2 — critère 2.5.8).
- Reflow : contenu utilisable à 400 % de zoom sans défilement horizontal.
- Redimensionnement du texte jusqu’à 200 % sans perte de contenu ni de fonctionnalité.
- Aucun `user-scalable=no` dans le viewport (le zoom utilisateur doit rester possible).
- Autoplay de médias évité, ou contrôles de pause immédiatement disponibles.

## Médias

- Vidéos avec sous-tres synchronisés (préréenregistré avec audio) et descriptions audio si nécessaire.
- Audios avec transcription.

## Signification

- Instructions claires, pas uniquement sensorielles (« cliquez sur le bouton rond »).
- Langues : attribut `lang` (et `xml:lang`) valides et cohérents sur la page.

## Tests

- Outils automatisés : axe-core, WAVE, Asqatas — **en complément**, jamais comme preuve unique.
- Revue manuelle au clavier obligatoire.
- Test avec un lecteur d’écran réel (NVDA, VoiceOver, JAWS) sur les parcours critiques : c’est ce que les outils automatisés ne couvrent pas.
- Pour le RGAA : audit selon la méthode officielle (échantillon de pages, taux de conformité).

## Références

- WCAG 2.2 : https://www.w3.org/TR/WCAG22/
- RGAA 4.1 : https://accessibilite.numerique.gouv.fr/
- Comprendre le WCAG (W3C, traductions) : https://www.w3.org/WAI/WCAG22/Understanding/

## Notes

- L’accessibilité se vérifie avec une combinaison d’outils automatisés et de revue manuelle.
- Le niveau cible (AA ou autre) dépend des exigences du projet ; pour les organismes publics français, le RGAA s’applique.
- Cet audit n’établit pas une conformité : il identifie des écarts à corriger.
