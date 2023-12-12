---
title: "Truchet TAoC"
date: 2023-12-12T09:19:38+01:00
tags: ["shader", "tiling", "truchet", "trad_fr"]
author: "hrst4"
draft: false
---
# Comment ça marche?

Tout d'abord on divise l'espace en une grille de carrés.

![truchet_taoc-20231212](./medias/truchet/truchet_taoc-20231212.png)

Dans chaque cellule, on dessine une forme très simple (par exemple une diagonale).

![truchet_taoc-20231212-1](./medias/truchet/truchet_taoc-20231212-1.png)

Pour toutes les cellules:

![truchet_taoc-20231212-2](./medias/truchet/truchet_taoc-20231212-2.png)

Ensuite, on *flip* les cellules de manière **aléatoire**.

![truchet_taoc-20231212-3](./medias/truchet/truchet_taoc-20231212-3.png)

# Dans Kodelife
Tout d'abord on divise l'espace en multipliant les uv puis en récupérant seulement la partie fractionnelle.

![truchet_taoc-20231212-4](./medias/truchet/truchet_taoc-20231212-4.png)

On décale d'un `vec2(.5)` pour centrer les coordonnées.

![truchet_taoc-20231212-5](./medias/truchet/truchet_taoc-20231212-5.png)

On peut aussi dessiner le contour des cellules à des fins de debug.

![truchet_taoc-20231212-6](./medias/truchet/truchet_taoc-20231212-6.png)

On dessine une diagonale pour chaque cellule.

![truchet_taoc-20231212-7](./medias/truchet/truchet_taoc-20231212-7.png)

Pour obtenir l'autre diagonale, il suffit de soustraire `gv.y` à `gv.x`.

![truchet_taoc-20231212-8](./medias/truchet/truchet_taoc-20231212-8.png)

Nous voulons choisir le sens de la diagonale aléatoirement.
Pour cela nous avons besoin d'identifier chaque cellule, on peut faire cela grâce à la fonction `floor`.

![truchet_taoc-20231212-9](./medias/truchet/truchet_taoc-20231212-9.png)

On utilise cet id unique pour générer un nombre aléatoire, qui déterminera le sens de la diagonale.
Pour cela on écrit la fonction `Hash21` (2 pour le nombre d'inputs et 1 pour le nombre d'outputs).

```cpp
float Hash21(vec2 p)
{
	p = fract(p*vec2(234.34,435.345));
	p += dot(p, p+34.23);
	return fract(p.x*p.y);
}
```

![truchet_taoc-20231212-10](./medias/truchet/truchet_taoc-20231212-10.png)

On peut utiliser ce nombre aléatoire pour décider du sens de la diagonale.

![truchet_taoc-20231212-11](./medias/truchet/truchet_taoc-20231212-11.png)

On remarque une anomalie aux coins des cellules:
![truchet_taoc-20231212-12](./medias/truchet/truchet_taoc-20231212-12.png)

Pour y remédier on trace 2 diagonales qui partent du centre des edges.

![truchet_taoc-20231212-13](./medias/truchet/truchet_taoc-20231212-13.png)

Maintenant au lieu de dessiner 2 lignes droites, on pourrait dessiner 2 arcs de cercle.

![truchet_taoc-20231212-14](./medias/truchet/truchet_taoc-20231212-14.png)

# Rajouter des textures
D'abord on récupère l'angle des points situés sur le cercle avec `atan`.
On affiche l'angle grâce à `sin` et un mouvement dans le temps.

![atan_truchet](./medias/truchet/atan_truchet.gif)

## Animer la texture
On aimerait que le mouvement ne soit pas inversé entre chaque cellule. Pour cela on utilise `mod`.
En l'appliquant à `id.x + id.y` on sélectionne bien les cellules pour lesquelles on veut inverser le mouvement. (une sur deux)

![truchet_taoc-20231212-15](./medias/truchet/truchet_taoc-20231212-15.png)

On décale l'intervalle des valeurs (0 ou +1) vers -1 ou +1 pour obtenir le résultat escompté: le mouvement a une direction constante.

![atan_truchet2 1](./medias/truchet/atan_truchet2 1.gif)

## Coordonnées UV
On veut fabriquer des coordonnées UV pour y placer des formes ultérieurement. On normalisera ces coordonnées pour plus de clarté.
- La coordonnée $x$ représente l'angle du point situé sur le cercle. (avec `atan`)
- La coordonnée $y$ représente la position du point sur l'axe perpendiculaire à l'arc de cercle (un côté du contour à $0$ et l'autre côté à $1$).
### la coordonnée x

![truchet_taoc-20231212-16](./medias/truchet/truchet_taoc-20231212-16.png)

### la coordonnée y

![truchet_taoc-20231212-17](./medias/truchet/truchet_taoc-20231212-17.png)

On remarque que la transition pour $y$ n'est pas fluide.
Pour y remédier, on change l'intervalle vers $[0.5, 0.0, 0.5]$ ...

![truchet_taoc-20231212-18](./medias/truchet/truchet_taoc-20231212-18.png)

... puis vers $[1.0, 0.0, 1.0]$ pour normaliser.

![truchet_taoc-20231212-19](./medias/truchet/truchet_taoc-20231212-19.png)

Voici l'affichage complet des uvs des arcs de cercle:

![truchet_taoc-20231212-21](./medias/truchet/truchet_taoc-20231212-21.png)

À partir de maintenant, on peut se servir des uvs pour intégrer une texture ou pourquoi pas un autre effet Truchet.

![truchet_taoc-20231212-22](./medias/truchet/truchet_taoc-20231212-22.png)

![atan_truchet3](./medias/truchet/atan_truchet3.gif)

On peut utiliser les uvs d'origine pour faire varier la taille en $y$.

![atan_truchet5](./medias/truchet/atan_truchet5.gif)

# Code GLSL

```cpp
#version 150
// src:
// https://youtu.be/2R7h76GoIJM

uniform float time;
uniform vec2 resolution;
uniform vec2 mouse;
uniform vec3 spectrum;

uniform sampler2D texture0;
uniform sampler2D texture1;
uniform sampler2D texture2;
uniform sampler2D texture3;
uniform sampler2D prevFrame;
uniform sampler2D prevPass;

in VertexData
{
    vec4 v_position;
    vec3 v_normal;
    vec2 v_texcoord;
} inData;

out vec4 fragColor;

float Hash21(vec2 p)
{
    p = fract(p*vec2(234.34,987.969));
    p += dot(p, p+34.23);
    return fract(p.x*p.y);
}

void main(void)
{
    vec2 uv = -1. + 2. * inData.v_texcoord;
    uv.x *= resolution.x / resolution.y;
    vec2 UV = inData.v_texcoord;
    uv += time*.2;
    vec3 col = vec3(0.);
    uv *= 1.5;
    vec2 id = floor(uv);

    
    
    vec2 gv = fract(uv)-.5;


    
    //if(gv.x > 0.48 || gv.y > 0.48) col = vec3(1.,0.,0.);
    
    
    float width = .2 * UV.y; ;
    
    float n = Hash21(id); // random number between 0 and 1
    float mask;
    
    // float d = abs(abs(gv.x - gv.y)-.5); straight lines
    if(n < .5) gv.x *= -1.;
    
    float d = abs(abs(gv.x+gv.y)-.5); // straight lines
    vec2 cUV = gv - sign(gv.x + gv.y+.001)*.5; // trick pour éviter le zéro
    d = length(cUV); // circle
    mask = smoothstep(0.02, 0.01, abs(d-.5)-width);
    float angle =atan(cUV.x, cUV.y);
    // méthode 2 avec le produit scalaire
    //mask = step(abs(dot(gv,vec2(-1.,1.))),.02);
    
    // col += mask;
    // col.rg = id*.2;
    float checker = mod(id.x+id.y,2.);
    checker = checker * 2. -1.;
    
    float flow = sin(checker*angle*10.+time);
    
    float x = angle; // 0 à PI/2
    x /= 1.57; // 0 à 1
    x = fract(checker*x+time*.2); // pour prendre en compte les valeurs négatives
    float y = d-(.5-width);
    y /= (2*width);
    y = abs(y-.5)*2.;
    
    vec2 tUV = vec2(x,y);
    //col.rg = tUV * mask;
    col += mask* texture(texture0, tUV*.5).rgb;
    col *= 1.-tUV.y;
    // col.rg = UV;
    fragColor = vec4(col,1.0);
    
}
```

# Vidéo originale
https://youtu.be/2R7h76GoIJM (the art of code)