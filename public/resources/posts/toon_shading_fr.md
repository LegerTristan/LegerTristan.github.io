# Réaliser un Toon Shader : Donner du volume avec la Spéculaire et la Rim Light

![Rendu final du Toon Shader avec Specular et Rim Light](resources/videos/ToonShading_Lights.gif)

Dans l'étape précédente de ma roadmap Technical Art, j'ai présenté comment transformer un dégradé de lumière classique en aplats de couleurs nets (le Cell Shading). Cependant, des couleurs plates ne suffisent pas à définir le volume d'un objet ou à le détacher de son environnement.

Aujourd'hui, je vais plus loin en ajoutant deux autres essentiels des modèles de lumière : la **Specular Light** (les reflets brillants) et la **Rim Light** (l'halo de contour).

---

## Les deux sources de lumières de la silhouette

Pour qu'un objet stylisé ait plus de profondeur, il doit réagir à la lumière de manière spécifique :
1.  **La Specular Light :** C'est le reflet direct de la source lumineuse. Dans un style NPR, au lieu d'être un dégradé flou, il devient généralement une tache nette qui souligne la courbure de l'objet.
2.  **La Rim Light :** C'est un halo lumineux qui parcourt les bords de la silhouette. Elle est cruciale pour détacher un personnage d'un fond sombre.

---

## Les différences entre Unity et Unreal Engine

Comme pour le précédent chaptire, les deux moteurs ont des approches radicalement différentes. 

* Pour Unity, j'ai gardé la même approche avec un accès direct aux données via `Main Light`. 
*   Pour Unreal, j'avais utilisé un post-Process global. C'est idéal pour aplatir l'image entière, mais c'est très limité pour la spéculaire et la Rim Light, car ces effets dépendent de la direction précise d'une lumière et de la position de la caméra.

J'ai donc décidé d'utiliser une **MPC (Material Parameter Collection)** cette fois-ci. Contrairement au Post-Process qui agit comme un filtre sur l'écran, le MPC permet d'envoyer des informations (comme la position du soleil) directement à tous les matériaux du jeu. Cela permet à chaque objet de calculer sa propre spéculaire de façon précise tout en restant synchronisé avec le reste du monde.

### Setup d'une MPC :

Voici brièvement comment mettre en place une **Material Parameter Collection (MPC)** :  

1.  Dans le Content Browser, je crée une MPC. À l'intérieur, il me faut un paramètre de type **Vector** que j'ai nommé `SunDirection`.
2.  Pour que la direction soit dynamique, il faut lier la rotation de la `Directional Light` au paramètre du MPC. Pour cela, dans le **Level Blueprint**, j'ai récupéré le Forward Vector de la lumière et utilisé le nœud `Set Vector Parameter Value` pour mettre à jour le MPC à chaque frame.
3.  Enfin, dans le Material Graph, j'appelle le nœud `CollectionParameter` et je sélectionne le MPC ainsi que le paramètre `SunDirection`. Ainsi, on obtient un vecteur utilisable pour les calculs de Dot Product, exactement comme dans Unity !

![Contenu du MPC dans Unreal](resources/visuels/posts/toonshading/ue_mpc.png)

![Configuration du Material Graph avec le MPC](resources/visuels/posts/toonshading/ue_nodes_mpc_setup.png)
---

## La Specular Light : Le modèle Blinn-Phong

Le calcul de la réflexion est relativement compliqué par défaut. Afin de le simplifier, aussi bien pour la compréhension que pour les performances du projet, j'utilise le modèle de **Blinn-Phong**. C'est une version optimisée du modèle de Phong classique, idéale pour le rendu en temps réel.

> **Note :** Il existe d'autres modèles plus complexes comme **Oren-Nayar** (pour les surfaces très rugueuses comme la peau) ou **Cook-Torrance** (utilisé dans le PBR pour un réalisme extrême). Mais pour du Toon Shading, Blinn-Phong est le plus adapté grâce à sa simplicité.

Voici comment mettre en place la specular light :

1. Contrairement à la lumière diffuse qui est la même peu importe d'où on regarde l'objet, la spéculaire dépend de la **position de la caméra**. Si vous bougez, le reflet bouge sur la surface. il faut donc utiliser le **Vecteur Caméra (V)**.
2. Dans le modèle de Phong original, il fallait  calculer le vecteur de réflexion exact de la lumière, ce qui est coûteux mathématiquement. Blinn-Phong simplifie cela avec le **Half-Vector**. Il s'agit de la moyenne normalisée entre la direction de la Lumière (L) et la direction de la Vue (V). Si la normale du mesh est parfaitement alignée avec ce "vecteur moyen", alors le reflet est au maximum de son intensité. 

> Sans lui, le calcul de la réflexion est beaucoup plus lourd pour la carte graphique. C'est une opération coûteuse car elle implique des calculs de vecteurs symétriques complexes. Blinn-Phong simplifie cela avec le **Half-Vector (H)** : c'est la moyenne normalisée entre la direction de la Lumière (L) et la direction de la Vue (V).

3. Enfin, pour obtenir cet effet "tache nette", on passe le résultat dans un `step` ou un `smoothstep`.

```hlsl
// Calcul du Half-Vector
float3 halfVector = normalize(lightDir + viewDir);

// Produit scalaire entre la normale et le half-vector
float dotNH = saturate(dot(normal, halfVector));

// Transformation en tache nette (Toon)
float specIntensity = smoothstep(_SpecSize - _SpecSmooth, _SpecSize + _SpecSmooth, dotNH);
float3 specularColor = specIntensity * _SpecularTint.rgb * mainLight.color;
```
Pour Unreal, comme indiqué précédemment, on utilise une **MPC** pour récupérer la direction du soleil et effectuer le calcul. Le résultat est appliqué comme une couleur émissive l'émissif pour override le post-process et forcer la specular light (et par la suite la Rim light) a être dessiné par dessus le Cell Shading.

![Logic des nodes Unreal pour la spéculaire](resources/visuels/posts/toonshading/ue_nodes_specular_lighting.png)

![Rendu de la Specular Light](resources/visuels/posts/toonshading/specular_lighting.png)

---

## La Rim Light : L'Effet Fresnel

Pour rappel, la Rim Light (ou lumière de contour) simule la lumière qui vient de derrière l'objet et "bave" sur les bords. Par conséquent cela va dépendre de nouveau de la vue (et donc la position de la caméra) et bien évidemment de la normal du mesh.

Comme d'habitude, cela repose sur un produit scalaire, cette fois-ci celui entre la **Normale (N)** et le **Vecteur Vue (V)**. On interpète le résultat compris entre [-1;1] ainsi :

*   Au centre de l'objet, les vecteurs sont alignés (Dot = 1).
*   Sur les bords, les vecteurs sont perpendiculaires (Dot = 0).

Si on laisse ainsi, le produit scalaire va donner du blanc au centre. Pour obtenir un contour, il faut inverser le résultat. En faisant `1.0 - dot(N, V)`, on obtient une valeur de **1 sur les bords** (brillant) et **0 au centre** (noir). C'est ce qu'on appelle **l'effet Fresnel**.

Voici le code hlsl :

```hlsl
// Calcul du Fresnel inversé
float rimDot = 1.0 - saturate(dot(normal, viewDir));

// Seuil net pour l'effet Toon
float rimIntensity = smoothstep(_RimThreshold - _RimSmooth, _RimThreshold + _RimSmooth, rimDot);

// Masquage par la lumière (optionnel) pour que la Rim n'apparaisse que côté lumière
rimIntensity *= saturate(dot(normal, lightDir));

float3 rimColor = rimIntensity * _RimColor.rgb;
```
Dans Unreal, on peut utiliser le nœud **Fresnel** natif. Dans mon cas, j'ai utilisé la même solution que sur Unity avec simple `Dot` suivi d'un `OneMinus`.

![Logic des nodes Unreal pour la Rim Light](resources/visuels/posts/toonshading/ue_nodes_rim_lighting.png)

![Rendu de la Rim Light](resources/visuels/posts/toonshading/rim_lighting.png)

> **Note :** Pour un rendu de haute qualité, on peut mélanger souvent les normales du mesh (**Vertex Normals**) avec les normales de la texture (**Pixel/Normal Map**). Cela permet d'avoir une Rim Light soit parfaitement lisse (style manga propre), soit très détaillée et accidentée selon les besoins artistiques.

---

## Conclusion

En combinant la **Diffuse**, l'**Ambiante**, la **Spéculaire** et la **Rim Light**, on obtient un rendu lumineux complet qui définit bien les volumes tout en conservant un look NPR.

![Rendu final complet](resources/videos/ToonShading.gif)
