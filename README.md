# Portfolio — Akasche Mahintan

Portfolio one-page **HTML / CSS / JavaScript vanilla** — aucun framework, aucun bundler, aucune dépendance à installer. Tout ce qui suit (scène 3D, scrollytelling, rendu PDF, thème, accessibilité) est écrit à la main dans `index.html`, avec quelques librairies chargées en CDN (Three.js, PDF.js) uniquement pour ce qu'elles font mieux qu'une réimplémentation maison.

## Sommaire

- [Pourquoi vanilla](#pourquoi-vanilla)
- [Système de design (CSS custom properties)](#système-de-design)
- [Scène 3D — Three.js](#scène-3d--threejs)
- [Scrollytelling du hero](#scrollytelling-du-hero)
- [Aperçu du CV — PDF.js + modal](#aperçu-du-cv--pdfjs--modal)
- [Traînée du curseur](#traînée-du-curseur)
- [Frise chronologique interactive](#frise-chronologique-interactive)
- [Responsive & menu mobile](#responsive--menu-mobile)
- [Accessibilité](#accessibilité)
- [Performance](#performance)
- [Structure du dépôt](#structure-du-dépôt)
- [Déploiement](#déploiement)

---

## Pourquoi vanilla

Aucun React/Vue, aucun Webpack/Vite. Choix assumé : un portfolio statique n'a pas besoin d'un framework, et l'absence de build montre que chaque effet (WebGL, scroll, canvas, thème) est compris et écrit directement, sans abstraction qui masquerait la mécanique.

Deux dépendances externes seulement, chargées via CDN et utilisées uniquement là où elles apportent une vraie valeur :
- **Three.js** (`three@0.160.0`, ES modules via `importmap`) → rendu WebGL de la scène 3D.
- **PDF.js** (`3.11.174`, UMD) → rendu du CV en `<canvas>` sans dépendre du plugin PDF natif du navigateur.

## Système de design

Toutes les couleurs passent par des **custom properties CSS** définies une seule fois (`:root`) :

```css
:root{
  --bg: #F7F6F3; --bg-panel: #FFFFFF;
  --ink: #17181B; --ink-soft: #54555B; --ink-faint: #7B7C81;
  --accent: #9C5E1E; --accent-dim: #7A4816;
  --btn-ghost-bg: rgba(23,24,27,0.035);
}
```

Chaque composant (cartes, boutons, timeline, CV) consomme ces variables plutôt que des couleurs en dur — un seul point de vérité pour toute évolution de palette. Le contraste de l'accent bronze a été recalculé pour rester lisible en texte sur fond clair (≈ 5:1), et non repris tel quel d'un thème sombre.

## Scène 3D — Three.js

Dans le hero : un réseau de nœuds (icosaèdre + arêtes), un cœur filaire central, un champ de particules, et une photo (BMW E30 M3) en arrière-plan.

Points techniques :
- **Renderer transparent** (`alpha: true`, `setClearColor(0x000000, 0)`) : le canvas WebGL n'a pas de fond opaque, la photo CSS en dessous reste visible à travers le réseau de nœuds.
- **Fog exponentiel** (`FogExp2`) calé sur la couleur de fond claire, pour que les particules/lignes lointaines se fondent dans le décor au lieu de s'arrêter net.
- **Un seul point de vérité géométrique** : les nœuds sont générés par déduplication des sommets d'un `IcosahedronGeometry` (`Set` sur les coordonnées arrondies), pas placés à la main.
- **Repli propre (`try/catch`)** : si WebGL échoue (navigateur, GPU bloqué, contexte perdu), la classe `no-webgl` est ajoutée au `<body>` et la photo CSS suffit seule — jamais d'écran cassé.
- **`prefers-reduced-motion`** : coupe les rotations et le drag caméra sans couper le rendu (la scène reste visible mais statique).

## Scrollytelling du hero

La demande initiale : *« la voiture reste fixe, les infos défilent en fondu au scroll »*. Implémentation :

1. Le hero est un wrapper de **hauteur = 3 × la hauteur d'écran** (`#hero-scroll`), contenant un panneau `position: sticky` (la photo + la scène 3D) qui reste épinglé pendant que le wrapper défile.
2. À chaque frame de scroll (throttlée via `requestAnimationFrame`), un **pourcentage de progression** (0 → 1) est calculé à partir de `getBoundingClientRect()` :
   ```js
   const scrollableHeight = wrapper.offsetHeight - pin.offsetHeight;
   const scrolled = headerHeight - wrapper.getBoundingClientRect().top;
   const progress = clamp(scrolled / scrollableHeight, 0, 1);
   ```
3. Cette progression est découpée en segments égaux (un par scène) ; chaque scène calcule son opacité et sa translation à partir d'un pourcentage de fondu d'entrée/sortie — pas de librairie de scroll-jacking, juste de l'interpolation linéaire.
4. La hauteur du wrapper est **mesurée en JS et recalculée au `resize`**, pas seulement fixée en CSS : évite tout décalage si la barre d'adresse mobile change de taille ou si le contenu recalcule sa hauteur.
5. **Repli sans JavaScript / `prefers-reduced-motion`** : si l'utilisateur préfère moins d'animation, le script n'ajoute jamais la classe `js-scenes` et les 3 scènes s'affichent normalement empilées, sans scroll piégé — dégradation progressive plutôt que tout ou rien.

## Aperçu du CV — PDF.js + modal

Plutôt qu'un `<iframe>` (barre d'outils du navigateur imposée, invisible sur mobile), le CV est **rendu en `<canvas>`** :

- Chaque page du PDF est rendue une fois en miniature (`page.render()` avec un `viewport` calculé au `devicePixelRatio`, capé à 2 pour ne pas surcharger le GPU sur les écrans très denses).
- Au clic sur une miniature, la **même page est re-rendue à une résolution plus élevée**, adaptée à la taille de l'écran (`Math.min(maxH/height, maxW/width)`) — pas un simple zoom CSS flou, un vrai second rendu net.
- Le modal se ferme au clic sur le fond, sur la croix, ou à `Échap`.
- **Repli automatique** : si PDF.js ne charge pas (CDN bloqué, `file://`), un `<iframe>` classique reprend le relais sans casser la page.

## Traînée du curseur

Canvas `position: fixed` par-dessus tout le site (`z-index: 9999`, `pointer-events: none` pour ne jamais bloquer les clics) :
- Chaque mouvement de souris **interpole des points intermédiaires** entre l'ancienne et la nouvelle position (au lieu d'un point par `mousemove`), pour un trait continu même en mouvement rapide.
- Chaque point vieillit et s'efface (`age / MAX_AGE`), dessiné avec un dégradé radial — pas de bibliothèque, ~50 lignes de canvas 2D.
- **Désactivée automatiquement** sur écran tactile (`matchMedia('(pointer: fine)')`) et si `prefers-reduced-motion` est actif.

## Frise chronologique interactive

- **Desktop/tablette** : une ligne avec des points ; au survol (ou au tap, géré en JS pour les écrans tactiles), une carte de détail apparaît en fondu au-dessus du point, positionnée à gauche/droite pour les points d'extrémité (évite qu'elle sorte de l'écran).
- **Mobile** : bascule automatique (media query) vers une liste verticale classique où tout le contenu est déjà visible — un survol qui doit « disparaître au retrait du curseur » n'a pas de sens au doigt, donc pas de fausse interactivité tactile.

## Responsive & menu mobile

- Menu hamburger avec dropdown animé (`max-height` transitionnée), fermeture au clic sur un lien ou au redimensionnement au-delà du breakpoint.
- Hauteur du hero en `100dvh` (viewport dynamique) plutôt que `100vh`, pour éviter le saut visuel causé par l'apparition/disparition de la barre d'adresse sur mobile.
- Tous les grids (compétences, projets, CV, contact) repassent en une colonne sous 760–800px ; padding resserré sous 420px.

## Accessibilité

- `prefers-reduced-motion` respecté à trois endroits indépendants : scène 3D, scrollytelling, traînée du curseur.
- Boutons de la frise et des pages du CV **focusables au clavier** (`aria-expanded`, `aria-hidden`, `aria-label` descriptifs), pas de `<div onclick>`.
- Contraste texte/fond recalculé pour le thème clair (accent ≈ 5:1 sur blanc).
- `:focus-visible` stylé explicitement (pas de `outline: none` sans remplacement).

## Performance

- Scroll listeners **throttlés en `requestAnimationFrame`** (pas de calcul à chaque `scroll` brut).
- `devicePixelRatio` toujours **capé à 2** (canvas WebGL, PDF.js, traînée) pour éviter de faire ramer les écrans 3×/4×.
- Nombre de particules 3D réduit sur petit écran (`isSmall` détecté une seule fois au chargement).
- Boucle `requestAnimationFrame` de la scène 3D **coupée sur `visibilitychange`** (onglet en arrière-plan = pas de rendu inutile).

## Structure du dépôt

```
index.html
README.md
assets/
  cv-akasche-mahintan.pdf
  e30-m3.webp
```

Un seul fichier HTML : CSS et JS sont inline, volontairement — pas de requêtes réseau supplémentaires à gérer pour un site de cette taille.

## Déploiement

Le site est servi par **GitHub Pages** :
`https://akshiigit.github.io/Portfolio-by-Akasche-MAHINTAN/`

1. Paramètres du dépôt → **Settings → Pages → Source → Deploy from a branch → `main` / `(root)`**.
2. Pousser `index.html` (et `assets/` si modifié) sur `main` :
   ```bash
   git add index.html assets
   git commit -m "Mise à jour du portfolio"
   git push
   ```

### Mettre à jour le CV plus tard

Remplacer `assets/cv-akasche-mahintan.pdf` en gardant exactement le même nom de fichier — aucune autre modification n'est nécessaire dans `index.html`.

### Tester en local avant de pousser

Ouvrir `index.html` directement en `file://` bloque certains scripts chargés en CDN (PDF.js notamment, par restriction CORS du navigateur). Préférer un petit serveur local :
```bash
python3 -m http.server
```
puis ouvrir `http://localhost:8000`.
