---
title: "IBL diffuse irradiance"
date: 2023-11-11
author: hrst4
tags: ['cg','opengl','graphics','cpp']
draft: false
---

# Rayonnement diffus
L'**IBL**, ou éclairage basé sur l'image, est un ensemble de techniques permettant d'éclairer des objets, non pas par des lumières analytiques directes comme dans le chapitre précédent, mais en traitant l'environnement comme une grande source de lumière. Ceci est généralement réalisé en manipulant une map d'environnement cubemap (prise dans le monde réel ou générée à partir d'une scène 3D) de telle sorte que nous puissions l'utiliser directement dans nos équations d'éclairage : en traitant chaque texel cubemap comme un émetteur de lumière. De cette manière, nous pouvons capturer efficacement l'éclairage global et l'ambiance générale d'un environnement, ce qui donne aux objets un meilleur sentiment d'appartenance à leur environnement.

Comme les algorithmes d'éclairage basés sur l'image capturent l'éclairage d'un certain environnement (global), leur entrée est considérée comme une forme plus précise d'éclairage ambiant, voire une approximation grossière de l'illumination globale. C'est ce qui rend l'IBL intéressant pour la PBR, car les objets ont l'air beaucoup plus précis physiquement lorsque nous prenons en compte l'éclairage de l'environnement.

Pour commencer à introduire l'IBL dans notre système PBR, jetons un coup d'œil rapide à l'équation de la réflectance :

$$
L\_o(p,w\_o)=\int_\Omega(k\_d{c\over\pi}+k\_s{
    DFG \over 4(w\_o*n)(w\_i*n)}
    L\_i(p,w\_i)n*w\_idw\_i)
$$

Comme décrit précédemment, notre objectif principal est de résoudre l'intégrale de toutes les directions lumineuses entrantes $w_i$ sur l'hémisphère $\Omega$ . La résolution de l'intégrale dans le chapitre précédent était facile car nous connaissions à l'avance les quelques directions lumineuses $w_i$ exactes qui contribuaient à l'intégrale. Cette fois, cependant, **chaque** direction lumineuse $w_i$ du milieu environnant peut potentiellement avoir une certaine radiance, ce qui rend la résolution de l'intégrale moins triviale. Il en résulte deux exigences principales pour la résolution de l'intégrale :

- Nous avons besoin d'un moyen de récupérer la radiance de la scène pour n'importe quel vecteur de direction $\vec{w_i}$
- La résolution de l'intégrale doit être rapide et en temps réel.

La première condition est relativement facile à remplir. Nous y avons déjà fait allusion, mais l'une des façons de représenter l'irradiance d'un environnement ou d'une scène est de le faire sous la forme d'un cubemap d'environnement (traité). Avec une telle cubemap, nous pouvons visualiser chaque texel de la cubemap comme une seule source lumineuse émettrice. En échantillonnant ce cubemap avec n'importe quel vecteur de direction $\vec{w_i}$ nous récupérons la radiance de la scène à partir de cette direction.

Obtenir la radiance de la scène à partir d'un vecteur de direction $\vec{w_i}$ est alors aussi simple que :
```cpp
vec3 radiance = texture(_cubemapEnvironment, w_i).rgb;  
```
Cependant, la résolution de l'intégrale nous oblige à échantillonner la map de l'environnement non pas dans une seule direction, mais dans toutes les directions possibles de l'hémisphère $\Omega$, ce qui est beaucoup trop coûteux pour chaque invocation du fragment shader. Pour résoudre l'intégrale de manière plus efficace, nous devrons pré traiter ou pré calculer la plupart des calculs. Pour ce faire, nous devons approfondir l'équation de la réflectance :

$$
L_o(p,w_o)=
\int_\Omega
(
k_d
{
c
\over
\pi
}
+
k_s
{
DFG
\over
4(w_o*n)(w_i*n)
}
L_i(p,w_i)n*
w_idw_i
)
$$

En examinant attentivement l'équation de réflectance, nous constatons que les termes diffus $k_d$ et spéculaire $k_s$ de la BRDF sont indépendants l'un de l'autre et que nous pouvons diviser l'intégrale en deux :

$$
L_o(p,w_o)=
\int_\Omega
(
k_d
{
c
\over
\pi
}
)
L_i(p,w_i)n*w_idw_i+
\int_\Omega
(
k_s
{
DFG
\over
4(w_o*n)(w_i*n)
}
)
L_i(p,w_i)n*w_idw_i
$$

En divisant l'intégrale en deux parties, nous pouvons nous concentrer sur les termes diffus et spéculaires individuellement ; ce chapitre se concentre sur l'intégrale diffuse.

En examinant de plus près l'intégrale diffuse, nous constatons que le terme de Lambert diffus est un terme constant (la couleur $c$, le rapport de réfraction $k_d$ et $\pi$ sont constants dans l'intégrale) et ne dépend d'aucune des variables de l'intégrale. Nous pouvons donc retirer le terme constant de l'intégrale diffuse :

$$
L_o(p,w_o)=
k_d
{
c
\over
\pi
}
\int_\Omega
L_i(p,w_i)n*w_idw_i
$$

Cela nous donne une intégrale qui ne dépend que de $w_i$ (en supposant que $p$ est au centre de la map de l'environnement). Avec cette connaissance, nous pouvons calculer ou pré calculer une nouvelle cubemap qui stocke dans chaque direction d'échantillonnage (ou texel) $w_o$ le résultat de l'intégrale diffuse par convolution.

La convolution consiste à appliquer un calcul à chaque entrée d'un ensemble de données en tenant compte de toutes les autres entrées de l'ensemble de données, l'ensemble de données étant la radiance de la scène ou la map de l'environnement. Ainsi, pour chaque direction d'échantillonnage dans la cubemap, nous prenons en compte toutes les autres directions d'échantillonnage sur l'hémisphère $\Omega$ sont prises en compte.

Pour convoluer (?) une map d'environnement, nous résolvons l'intégrale pour chaque direction d'échantillonnage $w_o$ en échantillonnant discrètement un grand nombre de directions $w_i$ sur l'hémisphère $\Omega$ et en calculant la moyenne de leur radiance. L'hémisphère à partir duquel nous construisons les directions d'échantillonnage $w_i$ est orienté vers la direction d'échantillonnage $w_o$ de sortie que nous convoluons.
![01_diffuse_irradiance-20230909-pbrdiff1.png](01_diffuse_irradiance-20230909-pbrdiff1.png)
Cette cubemap précalculée, qui pour chaque direction d'échantillonnage $w_o$ stocke le résultat intégral, peut être considérée comme la somme précalculée de toute la lumière diffuse indirecte de la scène frappant une surface alignée le long de la direction $w_o$. Une telle cubemap est connue sous le nom de map d'irradiance, étant donné que la cubemap convoluée nous permet effectivement d'échantillonner directement l'irradiance (précalculée) de la scène à partir de n'importe quelle direction $\vec{w_o}$.

>L'équation de la radiance dépend également d'une position $p$, que nous avons supposée être au centre de la carte d'irradiation. Cela signifie que toute la lumière indirecte diffuse doit provenir d'une seule map d'environnement, ce qui peut briser l'illusion de la réalité (en particulier à l'intérieur). Les moteurs de rendu résolvent ce problème en plaçant des sondes (?) de réflexion dans toute la scène, chacune d'entre elles calculant sa propre map d'irradiance de son environnement. De cette manière, l'irradiance (et la radiance) à la position $p$ est l'irradiance interpolée entre les sondes de réflexion les plus proches. Pour l'instant, nous supposons que nous échantillonnons toujours la map de l'environnement à partir de son centre.

Voici un exemple de map d'environnement cubemap et de la carte d'irradiance qui en résulte (avec l'aimable autorisation de wave engine), en calculant la moyenne de l'irradiation de la scène pour chaque direction.
![01_diffuse_irradiance-20230909-pbrdiff2.png](01_diffuse_irradiance-20230909-pbrdiff2.png)
En stockant le résultat convolué dans chaque texel cubemap (dans la direction de $\vec{w_o}$), la map d'irradiance s'affiche un peu comme une couleur moyenne ou un affichage de l'éclairage de l'environnement. L'échantillonnage de n'importe quelle direction à partir de cette map d'environnement nous donnera l'irradiance de la scène dans cette direction particulière.

## PBR et HDR
Nous l'avons brièvement évoqué dans le chapitre [précédent](../02_lighting.md) : il est extrêmement important de prendre en compte la gamme dynamique élevée de l'éclairage de votre scène dans un pipeline PBR. Comme le PBR base la plupart de ses entrées sur des propriétés et des mesures physiques réelles, il est logique de faire correspondre les valeurs de lumière entrantes à leurs équivalents physiques. Que nous fassions des suppositions éclairées sur le flux radiant de chaque lumière ou que nous utilisions leur équivalent physique direct, la différence entre une simple ampoule ou le soleil est significative dans les deux cas. Sans travailler dans un environnement de rendu HDR, il est impossible de spécifier correctement l'intensité relative de chaque lumière.

PBR et HDR vont donc de pair, mais quel est le rapport avec l'éclairage basé sur l'image ? Nous avons vu dans le chapitre précédent qu'il est relativement facile de faire fonctionner le PBR en HDR. Cependant, étant donné que pour l'éclairage basé sur l'image, nous basons l'intensité de la lumière indirecte de l'environnement sur les valeurs de couleur d'une cubemap d'environnement, nous avons besoin d'un moyen de stocker la plage dynamique élevée de l'éclairage dans une map d'environnement.

Les maps d'environnement que nous avons utilisées jusqu'à présent comme cubemaps (utilisées comme skyboxes par exemple) sont à faible plage dynamique (LDR). Nous avons directement utilisé les valeurs de couleur des images individuelles des faces, comprises entre $0.0$ et $1.0$, et les avons traitées telles quelles. Bien que cela puisse fonctionner pour une sortie visuelle, cela ne fonctionnera pas pour des paramètres d'entrée physiques.

### Le format de fichier HDR radiance
Entrez dans le format de fichier radiance. Le format de fichier radiance (avec l'extension `.hdr`) stocke un cubemap complet avec les 6 faces sous forme de données à virgule flottante. Cela nous permet de spécifier des valeurs de couleur en dehors de la plage de $0.0$ à $1.0$ pour donner aux lumières les intensités de couleur correctes. Le format de fichier utilise également une astuce pour stocker chaque valeur en virgule flottante, non pas comme une valeur de 32 bits par canal, mais de 8 bits par canal en utilisant le canal alpha de la couleur comme exposant (ce qui implique une perte de précision). Cela fonctionne assez bien, mais nécessite que le programme d'analyse syntaxique reconvertisse chaque couleur en son équivalent en virgule flottante.

Il existe un certain nombre de maps d'environnement HDR de radiance disponibles gratuitement à partir de sources telles que l'[archive sIBL](http://www.hdrlabs.com/sibl/archive.html), dont vous pouvez voir un exemple ci-dessous :
![01_diffuse_irradiance-20230909-pbrdiff3.png](01_diffuse_irradiance-20230909-pbrdiff3.png)
Ce n'est peut-être pas exactement ce à quoi vous vous attendiez, car l'image semble déformée et ne montre aucune des 6 faces cubemap individuelles des cartes d'environnement que nous avons vues auparavant. Cette map d'environnement est projetée à partir d'une sphère sur un plan plat, ce qui nous permet de stocker plus facilement l'environnement dans une seule image, connue sous le nom de map équirectangulaire. Cette méthode est assortie d'une petite mise en garde : la majeure partie de la résolution visuelle est stockée dans la direction horizontale de la vue, tandis qu'une partie moindre est préservée dans les directions inférieure et supérieure. Dans la plupart des cas, il s'agit d'un bon compromis, car avec presque tous les moteurs de rendu, vous trouverez la plupart des éclairages et des environnements intéressants dans les directions de visualisation horizontales.

### HDR et `stb_image.h`
Le chargement direct des images HDR de radiance nécessite une certaine connaissance du format de fichier, ce qui n'est pas trop difficile, mais néanmoins encombrant. Heureusement pour nous, la populaire bibliothèque d'en-tête `stb_image.h` prend en charge le chargement direct des images HDR de radiance sous la forme d'un tableau de valeurs à virgule flottante, ce qui répond parfaitement à nos besoins. Avec `stb_image` ajouté à votre projet, le chargement d'une image HDR est maintenant aussi simple que suit :
```cpp
#include "stb_image.h"
[...]

stbi_set_flip_vertically_on_load(true);
int width, height, nrComponents;
float *data = stbi_loadf("newport_loft.hdr", &width, &height, &nrComponents, 0);
unsigned int hdrTexture;
if (data)
{
    glGenTextures(1, &hdrTexture);
    glBindTexture(GL_TEXTURE_2D, hdrTexture);
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB16F, width, height, 0, GL_RGB, GL_FLOAT, data); 

    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

    stbi_image_free(data);
}
else
{
    std::cout << "Failed to load HDR image." << std::endl;
}  
```
`stb_image.h` convertit automatiquement les valeurs HDR en une liste de valeurs à virgule flottante : 32 bits par canal et 3 canaux par couleur par défaut. C'est tout ce dont nous avons besoin pour stocker la map d'environnement HDR équirectangulaire dans une texture 2D à virgule flottante.

### D'équirectangulaire à Cubemap
Il est possible d'utiliser la carte équirectangulaire directement pour les recherches d'environnement, mais ces opérations peuvent être relativement coûteuses, auquel cas un échantillon cubemap direct est plus performant. Par conséquent, dans ce chapitre, nous allons d'abord convertir l'image équirectangulaire en cubemap pour un traitement ultérieur. Notez que nous montrons également comment échantillonner une carte équirectangulaire comme s'il s'agissait d'une map d'environnement 3D, auquel cas vous êtes libre de choisir la solution que vous préférez.

Pour convertir une image équirectangulaire en cubemap, nous devons calculer un cube (unitaire) et projeter la map équirectangulaire sur toutes les faces du cube depuis l'intérieur et prendre 6 images de chaque côté du cube comme une face du cubemap. Le vertex shader de ce cube rend simplement le cube tel quel et transmet sa position locale au fragment shader en tant que vecteur d'échantillon 3D :
```cpp
#version 330 core
layout (location = 0) in vec3 aPos;

out vec3 localPos;

uniform mat4 projection;
uniform mat4 view;

void main()
{
    localPos = aPos;  
    gl_Position =  projection * view * vec4(localPos, 1.0);
}
```
Pour le fragment shader, nous colorons chaque partie du cube comme si nous avions plié la map équirectangulaire sur chaque côté du cube. Pour ce faire, nous prenons la direction d'échantillonnage du fragment comme interpolée à partir de la position locale du cube, puis nous utilisons ce vecteur de direction et un peu de trigonométrie (de sphérique à cartésien) pour échantillonner la map équirectangulaire comme s'il s'agissait d'une map cubique. Nous stockons directement le résultat sur le fragment de la face du cube, ce qui devrait être tout ce que nous avons à faire :
```cpp
#version 330 core
out vec4 FragColor;
in vec3 localPos;

uniform sampler2D equirectangularMap;

const vec2 invAtan = vec2(0.1591, 0.3183);
vec2 SampleSphericalMap(vec3 v)
{
    vec2 uv = vec2(atan(v.z, v.x), asin(v.y));
    uv *= invAtan;
    uv += 0.5;
    return uv;
}

void main()
{		
    vec2 uv = SampleSphericalMap(normalize(localPos)); // make sure to normalize localPos
    vec3 color = texture(equirectangularMap, uv).rgb;
    
    FragColor = vec4(color, 1.0);
}
```
Si vous effectuez le rendu d'un cube au centre de la scène avec une carte HDR équirectangulaire, vous obtiendrez quelque chose qui ressemble à ceci :
![01_diffuse_irradiance-20230910-pbrdiff5.png](01_diffuse_irradiance-20230910-pbrdiff5.png)
Ceci démontre que nous avons effectivement mappé une image équirectangulaire sur une forme cubique, mais ne nous aide pas encore à convertir l'image HDR source en une texture cubemap. Pour ce faire, nous devons effectuer le rendu du même cube 6 fois, en regardant chaque face individuelle du cube, tout en enregistrant le résultat visuel à l'aide d'un objet framebuffer :
```cpp
unsigned int captureFBO, captureRBO;
glGenFramebuffers(1, &captureFBO);
glGenRenderbuffers(1, &captureRBO);

glBindFramebuffer(GL_FRAMEBUFFER, captureFBO);
glBindRenderbuffer(GL_RENDERBUFFER, captureRBO);
glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH_COMPONENT24, 512, 512);
glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_RENDERBUFFER, captureRBO);
```
Bien entendu, nous générons également les textures colorées cubemap correspondantes, en pré-allouant de la mémoire pour chacune de ses 6 faces :
```cpp
unsigned int envCubemap;
glGenTextures(1, &envCubemap);
glBindTexture(GL_TEXTURE_CUBE_MAP, envCubemap);
for (unsigned int i = 0; i < 6; ++i)
{
    // note that we store each face with 16 bit floating point values
    glTexImage2D(GL_TEXTURE_CUBE_MAP_POSITIVE_X + i, 0, GL_RGB16F, 
                 512, 512, 0, GL_RGB, GL_FLOAT, nullptr);
}
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_R, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
```
Il ne reste plus qu'à capturer la texture 2D équirectangulaire sur les faces du cubemap.

Je ne m'étendrai pas sur les détails car le code détaille des sujets déjà abordés dans les chapitres sur les [framebuffers](../../04_Advanced_OpenGL/04_framebuffers.md) et les [ombres ponctuelles](../../05_Advanced_Lighting/02b_point_shadows.md), mais cela se résume à mettre en place 6 matrices de vue différentes (face à chaque côté du cube), à mettre en place une matrice de projection avec un **fov** (field of view) de $90$ degrés pour capturer l'ensemble de la face, et à rendre un cube 6 fois en stockant les résultats dans un framebuffer à virgule flottante :
```cpp
glm::mat4 captureProjection = glm::perspective(glm::radians(90.0f), 1.0f, 0.1f, 10.0f);
glm::mat4 captureViews[] = 
{
   glm::lookAt(glm::vec3(0.0f, 0.0f, 0.0f), glm::vec3( 1.0f,  0.0f,  0.0f), glm::vec3(0.0f, -1.0f,  0.0f)),
   glm::lookAt(glm::vec3(0.0f, 0.0f, 0.0f), glm::vec3(-1.0f,  0.0f,  0.0f), glm::vec3(0.0f, -1.0f,  0.0f)),
   glm::lookAt(glm::vec3(0.0f, 0.0f, 0.0f), glm::vec3( 0.0f,  1.0f,  0.0f), glm::vec3(0.0f,  0.0f,  1.0f)),
   glm::lookAt(glm::vec3(0.0f, 0.0f, 0.0f), glm::vec3( 0.0f, -1.0f,  0.0f), glm::vec3(0.0f,  0.0f, -1.0f)),
   glm::lookAt(glm::vec3(0.0f, 0.0f, 0.0f), glm::vec3( 0.0f,  0.0f,  1.0f), glm::vec3(0.0f, -1.0f,  0.0f)),
   glm::lookAt(glm::vec3(0.0f, 0.0f, 0.0f), glm::vec3( 0.0f,  0.0f, -1.0f), glm::vec3(0.0f, -1.0f,  0.0f))
};

// convert HDR equirectangular environment map to cubemap equivalent
equirectangularToCubemapShader.use();
equirectangularToCubemapShader.setInt("equirectangularMap", 0);
equirectangularToCubemapShader.setMat4("projection", captureProjection);
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, hdrTexture);

glViewport(0, 0, 512, 512); // don't forget to configure the viewport to the capture dimensions.
glBindFramebuffer(GL_FRAMEBUFFER, captureFBO);
for (unsigned int i = 0; i < 6; ++i)
{
    equirectangularToCubemapShader.setMat4("view", captureViews[i]);
    glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, 
                           GL_TEXTURE_CUBE_MAP_POSITIVE_X + i, envCubemap, 0);
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

    renderCube(); // renders a 1x1 cube
}
glBindFramebuffer(GL_FRAMEBUFFER, 0);  
```
Nous prenons l'attachement de couleur du framebuffer et changeons sa cible de texture pour chaque face de la cubemap, en rendant directement la scène sur l'une des faces de la cubemap. Une fois cette routine terminée (ce que nous ne devons faire qu'une seule fois), la cubemap `envCubemap` devrait être la version de l'environnement cubemap de notre image HDR originale.

Testons le cubemap en écrivant un shader skybox très simple pour afficher le cubemap autour de nous :
```cpp
#version 330 core
layout (location = 0) in vec3 aPos;

uniform mat4 projection;
uniform mat4 view;

out vec3 localPos;

void main()
{
    localPos = aPos;

    mat4 rotView = mat4(mat3(view)); // remove translation from the view matrix
    vec4 clipPos = projection * rotView * vec4(localPos, 1.0);

    gl_Position = clipPos.xyww;
}
```
Notez l'astuce de `xyww` qui garantit que la valeur de profondeur des fragments de cube rendus est toujours égale à $1.0$, la valeur de profondeur maximale, comme décrit dans le chapitre sur les cubemap. Notez que nous devons changer la fonction de comparaison de profondeur en `GL_LEQUAL` :
```cpp
glDepthFunc(GL_LEQUAL);  
```
Le shader de fragment échantillonne ensuite directement la map d'environnement cubemap en utilisant la position locale du fragment du cube :
```cpp
#version 330 core
out vec4 FragColor;

in vec3 localPos;
  
uniform samplerCube environmentMap;
  
void main()
{
    vec3 envColor = texture(environmentMap, localPos).rgb;
    
    envColor = envColor / (envColor + vec3(1.0));
    envColor = pow(envColor, vec3(1.0/2.2)); 
  
    FragColor = vec4(envColor, 1.0);
}
```
Nous échantillonnons la map de l'environnement en utilisant les positions interpolées des vertex cube qui correspondent directement au vecteur de direction correct à échantillonner. Étant donné que les composantes de translation de la caméra sont ignorées, le rendu de ce shader sur un cube devrait vous donner la map de l'environnement comme un arrière-plan immobile. De plus, comme nous sortons directement les valeurs HDR de la map de l'environnement dans le framebuffer **LDR** par défaut, nous voulons que les valeurs de couleur soient correctement reproduites. De plus, presque toutes les cartes **HDR** sont dans un espace colorimétrique linéaire par défaut, nous devons donc appliquer une correction gamma avant d'écrire dans le framebuffer par défaut.

Le rendu de la map d'environnement échantillonnée sur les sphères précédemment rendues devrait ressembler à ceci :
![01_diffuse_irradiance-20230910-pbrdiff6.png](01_diffuse_irradiance-20230910-pbrdiff6.png)
Eh bien... il nous a fallu pas mal de configuration pour en arriver là, mais nous avons réussi à lire une carte d'environnement HDR, à la convertir de son mapping équirectangulaire en un cubemap, et à rendre le cubemap HDR dans la scène en tant que skybox. De plus, nous avons mis en place un petit système pour effectuer le rendu sur les 6 faces d'une cubemap, ce dont nous aurons à nouveau besoin lors de la convolution de la map d'environnement. Vous pouvez trouver le code source de l'ensemble du processus de conversion [ici](https://learnopengl.com/code_viewer_gh.php?code=src/6.pbr/2.1.1.ibl_irradiance_conversion/ibl_irradiance_conversion.cpp).

## Convolution de la cubemap
Comme décrit au début du chapitre, notre objectif principal est de résoudre l'intégrale de tous les éclairages indirects diffus étant donné l'irradiance de la scène sous la forme d'une carte d'environnement cubemap. Nous savons que nous pouvons obtenir la radiance de la scène $L(p,w_i)$ dans une direction particulière en échantillonnant une carte d'environnement HDR dans la direction $w_i$. Pour résoudre l'intégrale, nous devons échantillonner la radiance de la scène dans toutes les directions possibles de l'hémisphère $\Omega$ pour chaque fragment.

Il est cependant impossible, d'un point de vue informatique, d'échantillonner l'éclairage de l'environnement dans toutes les directions possibles de l'hémisphère $\Omega$ le nombre de directions possibles étant théoriquement infini. Nous pouvons cependant approximer le nombre de directions en prenant un nombre fini de directions ou d'échantillons, espacés uniformément ou prélevés au hasard dans l'hémisphère, pour obtenir une approximation assez précise de l'irradiance ; en fait, nous résolvons l'intégrale $\int$ de manière discrète

Il est cependant trop coûteux d'effectuer cette opération pour chaque fragment en temps réel, car le nombre d'échantillons doit être significativement élevé pour obtenir des résultats satisfaisants, c'est pourquoi nous voulons effectuer un calcul préalable. Étant donné que l'orientation de l'hémisphère détermine l'endroit où nous capturons l'irradiance, nous pouvons pré calculer l'irradiance pour chaque orientation possible de l'hémisphère orienté autour de toutes les directions sortantes $w_o$:
$$
L(p,w_o)
=
k_d
{
c
\over
\pi
}
\int_\Omega
L_i(p,w_i)n * w_idw_i
$$
Étant donné un vecteur de direction $w_i$ dans la passe d'éclairage, nous pouvons alors échantillonner la map d'irradiance précalculée pour récupérer l'irradiance diffuse totale à partir de la direction $w_i$. Pour déterminer la quantité de lumière diffuse indirecte (irradiante) à la surface d'un fragment, nous récupérons l'irradiance totale de l'hémisphère orienté autour de sa normale de surface. L'obtention de l'éclairement énergétique de la scène est alors aussi simple que :
```cpp
vec3 irradiance = texture(irradianceMap, N).rgb;
```
Maintenant, pour générer la map d'irradiance, nous devons convoluer l'éclairage de l'environnement tel qu'il a été converti en cubemap. Étant donné que pour chaque fragment, l'hémisphère de la surface est orienté le long du vecteur normal $\vec{N}$, la convolution d'une cubemap revient à calculer la radiance moyenne totale de chaque direction $w_i$ dans l'hémisphère $\Omega$ orienté le long de $\vec{N}$.
![01_diffuse_irradiance-20230910-pbrdiff7.png](01_diffuse_irradiance-20230910-pbrdiff7.png)
Heureusement, toute la configuration fastidieuse de ce chapitre n'est pas inutile puisque nous pouvons maintenant prendre directement la cubemap convertie, la convoluer dans un fragment shader, et capturer le résultat dans une nouvelle cubemap en utilisant un framebuffer qui rend dans les 6 directions de la face. Comme nous l'avons déjà fait pour convertir la map d'environnement équirectangulaire en cubemap, nous pouvons adopter exactement la même approche mais en utilisant un fragment shader différent :
```cpp
#version 330 core
out vec4 FragColor;
in vec3 localPos;

uniform samplerCube environmentMap;

const float PI = 3.14159265359;

void main()
{		
    // the sample direction equals the hemisphere's orientation 
    vec3 normal = normalize(localPos);
  
    vec3 irradiance = vec3(0.0);
  
    [...] // convolution code
  
    FragColor = vec4(irradiance, 1.0);
}
```
La map d'environnement étant la cubemap HDR convertie à partir de la carte d'environnement HDR équirectangulaire.

Il existe de nombreuses façons de convoluer la map d'environnement, mais pour ce chapitre, nous allons générer une quantité fixe de vecteurs d'échantillonnage pour chaque texel de la cubemap le long d'un hémisphère $\Omega$ orienté autour de la direction de l'échantillon et faire la moyenne des résultats. La quantité fixe de vecteurs d'échantillonnage sera uniformément répartie à l'intérieur de l'hémisphère. Notez qu'une intégrale est une fonction continue et que l'échantillonnage discret de sa fonction à partir d'un nombre fixe de vecteurs d'échantillonnage ne sera qu'une approximation. Plus le nombre de vecteurs d'échantillonnage est élevé, meilleure est l'approximation de l'intégrale.

L'intégrale $\int$ de l'équation de réflectance tourne autour de l'angle solide $dw$, ce qui est assez difficile à traiter. Au lieu d'intégrer sur l'angle solide $dw$, nous intégrerons sur ses coordonnées sphériques équivalentes $\theta$ et $\phi$.
![01_diffuse_irradiance-20230910-pbrdiff8.png](01_diffuse_irradiance-20230910-pbrdiff8.png)
Nous utilisons l'azimut polaire $\phi$ pour échantillonner autour de l'anneau de l'hémisphère entre $0$ et $2\pi$, et utilisons l'inclinaison zénithale $\theta$ entre $0$ et $12\pi$ pour échantillonner les anneaux croissants de l'hémisphère. Nous obtenons ainsi l'intégrale de réflectance mise à jour :

$$
L_o(p,\phi_o,\theta_o)=
k_d
{
c
\over
\pi
}
\int_{\phi=0}^{2\pi}
\int_{\theta=0}^{{1\over2}\pi}
L_i(p,\phi_i,\theta_i)\cos{(\theta)}\sin{(\theta)}d\phi d\theta
$$

La résolution de l'intégrale exige que nous prenions un nombre fixe d'échantillons discrets dans l'hémisphère $\Omega$ et que nous fassions la moyenne de leurs résultats. Cela traduit l'intégrale en la version discrète suivante, basée sur la somme de Riemann, étant donné $n1$ et $n2$ échantillons discrets sur chaque coordonnée sphérique respectivement :

$$
L_o(p,\phi_o,\theta_o)=
k_d
{
c\pi
\over
n1n2
}
\sum_{\phi=0}^{n1}
\sum_{\theta=0}^{n2}
L_i(p,\phi_i,\theta_i)
\cos{(\theta)}\sin{(\theta)}d\phi d\theta
$$

Comme nous échantillonnons les deux valeurs sphériques de manière discrète, chaque échantillon se rapproche d'une zone de l'hémisphère ou en fait la moyenne, comme le montre l'image ci-dessus. Notez que (en raison des propriétés générales d'une forme sphérique) la zone d'échantillonnage discrète de l'hémisphère se réduit à mesure que l'angle zénithal $\theta$ augmente et que les régions d'échantillonnage convergent vers le centre supérieur. Pour compenser les zones plus petites, nous pondérons leur contribution en mettant à l'échelle la zone par $\sin{(\theta)}$.

L'échantillonnage discret de l'hémisphère en fonction des coordonnées sphériques de l'intégrale se traduit par le fragment de code suivant :

```cpp
vec3 irradiance = vec3(0.0);  

vec3 up    = vec3(0.0, 1.0, 0.0);
vec3 right = normalize(cross(up, normal));
up         = normalize(cross(normal, right));

float sampleDelta = 0.025;
float nrSamples = 0.0; 
for(float phi = 0.0; phi < 2.0 * PI; phi += sampleDelta)
{
    for(float theta = 0.0; theta < 0.5 * PI; theta += sampleDelta)
    {
        // spherical to cartesian (in tangent space)
        vec3 tangentSample = vec3(sin(theta) * cos(phi),  sin(theta) * sin(phi), cos(theta));
        // tangent space to world
        vec3 sampleVec = tangentSample.x * right + tangentSample.y * up + tangentSample.z * N; 

        irradiance += texture(environmentMap, sampleVec).rgb * cos(theta) * sin(theta);
        nrSamples++;
    }
}
irradiance = PI * irradiance * (1.0 / float(nrSamples));
```
Nous spécifions une valeur delta `sampleDelta` fixe pour parcourir l'hémisphère ; la diminution ou l'augmentation du delta sample augmentera ou diminuera la précision respectivement.

Dans les deux boucles, nous prenons les deux coordonnées sphériques pour les convertir en un vecteur d'échantillonnage cartésien 3D, nous convertissons l'échantillon de la tangente à l'espace mondial orienté autour de la normale et nous utilisons ce vecteur d'échantillonnage pour échantillonner directement la map de l'environnement HDR. Nous ajoutons chaque résultat d'échantillonnage à l'irradiance, que nous divisons ensuite par le nombre total d'échantillons prélevés, ce qui nous donne l'irradiance moyenne échantillonnée. Notez que nous mettons à l'échelle la valeur de la couleur échantillonnée par $\cos{(\theta)}$ en raison de l'affaiblissement de la lumière à des angles plus grands et par $\sin{(\theta)}$ pour tenir compte des zones d'échantillonnage plus petites dans les zones de l'hémisphère supérieur.

Il ne reste plus qu'à configurer le code de rendu OpenGL de manière à ce que nous puissions convoluer le `envCubemap` capturé précédemment. Tout d'abord, nous créons la cubemap d'irradiance (encore une fois, nous n'avons à le faire qu'une seule fois avant la boucle de rendu) :
```cpp
unsigned int irradianceMap;
glGenTextures(1, &irradianceMap);
glBindTexture(GL_TEXTURE_CUBE_MAP, irradianceMap);
for (unsigned int i = 0; i < 6; ++i)
{
    glTexImage2D(GL_TEXTURE_CUBE_MAP_POSITIVE_X + i, 0, GL_RGB16F, 32, 32, 0, 
                 GL_RGB, GL_FLOAT, nullptr);
}
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_R, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
```
Comme la map d'irradiance fait la moyenne de toute la radiance environnante de manière uniforme, elle n'a pas beaucoup de détails à haute fréquence, donc nous pouvons stocker la map à une faible résolution ($32\times32$) et laisser le filtrage linéaire d'OpenGL faire le plus gros du travail. Ensuite, nous redimensionnons le framebuffer de capture à la nouvelle résolution :
```cpp
glBindFramebuffer(GL_FRAMEBUFFER, captureFBO);
glBindRenderbuffer(GL_RENDERBUFFER, captureRBO);
glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH_COMPONENT24, 32, 32);  
```
À l'aide du shader de convolution, nous rendons la map de l'environnement de la même manière que nous avons capturé la cubemap de l'environnement :
```cpp
irradianceShader.use();
irradianceShader.setInt("environmentMap", 0);
irradianceShader.setMat4("projection", captureProjection);
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_CUBE_MAP, envCubemap);

glViewport(0, 0, 32, 32); // don't forget to configure the viewport to the capture dimensions.
glBindFramebuffer(GL_FRAMEBUFFER, captureFBO);
for (unsigned int i = 0; i < 6; ++i)
{
    irradianceShader.setMat4("view", captureViews[i]);
    glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, 
                           GL_TEXTURE_CUBE_MAP_POSITIVE_X + i, irradianceMap, 0);
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

    renderCube();
}
glBindFramebuffer(GL_FRAMEBUFFER, 0);
```
Après cette routine, nous devrions avoir une map d'irradiance pré-calculée que nous pouvons directement utiliser pour notre éclairage basé sur l'image diffuse. Pour voir si nous avons réussi à convoluer la map d'environnement, nous allons substituer la map d'environnement à la map d'irradiance en tant qu'échantillonneur d'environnement de la skybox :
![01_diffuse_irradiance-20230910-pbrdiff8-1.png](01_diffuse_irradiance-20230910-pbrdiff8-1.png)
Si elle ressemble à une version très floue de la map de l'environnement, vous avez réussi à convoluer la map de l'environnement.

## PBR et éclairage par rayonnement indirect
La map d'irradiance représente la partie diffuse de l'intégrale de la réflectance, accumulée à partir de toute la lumière indirecte environnante. Étant donné que la lumière ne provient pas de sources lumineuses directes, mais de l'environnement, nous traitons l'éclairage indirect diffus et spéculaire comme l'éclairage ambiant, remplaçant ainsi le terme constant que nous avions défini précédemment.

Tout d'abord, assurez-vous d'ajouter la map d'irradiance précalculée en tant qu'échantillonneur de cube :
```cpp
uniform samplerCube irradianceMap;
```
Étant donné la map d'irradiance qui contient toute la lumière diffuse indirecte de la scène, la récupération de l'irradiance influençant le fragment est aussi simple qu'un simple échantillon de texture donné à la normale de la surface :
```cpp
// vec3 ambient = vec3(0.03);
vec3 ambient = texture(irradianceMap, N).rgb;
```
Cependant, comme l'éclairage indirect contient à la fois une partie diffuse et une partie spéculaire (comme nous l'avons vu dans la version fractionnée de l'équation de réflectance), nous devons pondérer la partie diffuse en conséquence. Comme dans le chapitre précédent, nous utilisons l'équation de Fresnel pour déterminer le rapport de réflectance indirecte de la surface, à partir duquel nous déduisons le rapport de réfraction (ou rapport diffus) :
```cpp
vec3 kS = fresnelSchlick(max(dot(N, V), 0.0), F0);
vec3 kD = 1.0 - kS;
vec3 irradiance = texture(irradianceMap, N).rgb;
vec3 diffuse    = irradiance * albedo;
vec3 ambient    = (kD * diffuse) * ao; 
```
Comme la lumière ambiante provient de toutes les directions à l'intérieur de l'hémisphère orienté autour de la normale $\vec{N}$, il n'y a pas de vecteur unique à mi-chemin pour déterminer la réponse de Fresnel. Pour continuer à simuler Fresnel, nous calculons le Fresnel à partir de l'angle entre la normale et le vecteur de vue. Cependant, nous avons utilisé précédemment le vecteur médian de la micro-surface, influencé par la rugosité de la surface, comme entrée de l'équation de Fresnel. Comme nous ne tenons actuellement pas compte de la rugosité, le taux de réflexion de la surface sera toujours relativement élevé. La lumière indirecte suit les mêmes propriétés que la lumière directe et nous nous attendons donc à ce que les surfaces plus rugueuses se reflètent moins fortement sur les bords de la surface. C'est pourquoi l'intensité de la réflexion indirecte de Fresnel semble faible sur les surfaces non métalliques rugueuses (légèrement exagérée à des fins de démonstration) :
![01_diffuse_irradiance-20230910-pbrdiff11.png](01_diffuse_irradiance-20230910-pbrdiff11.png)
 Nous pouvons résoudre ce problème en injectant un terme de rugosité dans l'équation de Fresnel-Schlick, comme l'a décrit [Sébastien Lagarde](https://seblagarde.wordpress.com/2011/08/17/hello-world/) :
```cpp
vec3 fresnelSchlickRoughness(float cosTheta, vec3 F0, float roughness)
{
    return F0 + (max(vec3(1.0 - roughness), F0) - F0) * pow(clamp(1.0 - cosTheta, 0.0, 1.0), 5.0);
} 
```
En tenant compte de la rugosité de la surface lors du calcul de la réponse de Fresnel, le code ambiant est le suivant :
```cpp
vec3 kS = fresnelSchlickRoughness(max(dot(N, V), 0.0), F0, roughness); 
vec3 kD = 1.0 - kS;
vec3 irradiance = texture(irradianceMap, N).rgb;
vec3 diffuse    = irradiance * albedo;
vec3 ambient    = (kD * diffuse) * ao;
```
Comme vous pouvez le voir, le calcul de l'éclairage basé sur l'image est assez simple et ne nécessite qu'une seule recherche de texture cubemap ; la majeure partie du travail consiste à pré-calculer ou à convoluer la map d'irradiance.

Si nous prenons la scène initiale du chapitre sur l'[éclairage PBR](../02_lighting.md), où chaque sphère a une valeur métallique croissante verticalement et une valeur de rugosité croissante horizontalement, et que nous ajoutons l'éclairage diffus basé sur l'image, cela ressemblera un peu à ceci :
![01_diffuse_irradiance-20230910-pbrdiff12.png](01_diffuse_irradiance-20230910-pbrdiff12.png)
C'est encore un peu bizarre car les sphères les plus métalliques ont besoin d'une certaine forme de réflexion pour commencer à ressembler à des surfaces métalliques (car les surfaces métalliques ne reflètent pas la lumière diffuse) qui, pour l'instant, ne proviennent que (à peine) des sources lumineuses ponctuelles. Néanmoins, on peut déjà dire que les sphères se sentent plus à leur place dans l'environnement (surtout si vous passez d'une map d'environnement à l'autre) car la réponse de la surface réagit en fonction de l'éclairage ambiant de l'environnement.

Vous pouvez trouver le code source complet des sujets abordés [ici](https://learnopengl.com/code_viewer_gh.php?code=src/6.pbr/2.1.2.ibl_irradiance/ibl_irradiance.cpp). Dans le [prochain](prochain) chapitre, nous ajouterons la partie spéculaire indirecte de l'intégrale de réflectance et nous verrons alors la puissance du PBR.

## Lectures supplémentaires
 - [Coding Labs : Physically based rendering](http://www.codinglabs.net/article_physically_based_rendering.aspx) : une introduction au PBR et comment et pourquoi générer une map d'irradiance.
- [The Mathematics of Shading](http://www.scratchapixel.com/lessons/mathematics-physics-for-computer-graphics/mathematics-of-shading) : une brève introduction par ScratchAPixel sur plusieurs des mathématiques décrites dans ce tutoriel, en particulier sur les coordonnées polaires et les intégrales.

