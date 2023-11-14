---
title: "ssao"
date: 2023-11-11
author: hrst4
tags: ['cg','opengl','graphics','cpp']
draft: false
---

# SSAO
Nous avons brièvement abordé le sujet dans le chapitre sur l'éclairage de base : l'éclairage ambiant. **L'éclairage ambiant est une constante lumineuse fixe que nous ajoutons à l'éclairage global d'une scène pour simuler la diffusion de la lumière**. En réalité, la lumière se disperse dans toutes sortes de directions avec des intensités variables, de sorte que les parties indirectement éclairées d'une scène doivent également présenter des intensités variables. **L'un des types d'approximation de l'éclairage indirect est appelé occlusion ambiante**. Il tente d'approximer l'éclairage indirect en assombrissant les plis, les trous et les surfaces proches les unes des autres. Ces zones sont largement occultées par la géométrie environnante et les rayons lumineux ont donc moins d'endroits où s'échapper, d'où l'aspect plus sombre de ces zones. Jetez un coup d'œil aux coins et aux plis de votre pièce pour constater que la lumière y semble un peu plus sombre.

Vous trouverez ci-dessous un exemple d'image d'une scène avec et sans occlusion ambiante. Remarquez que la lumière (ambiante) est plus occultée, en particulier entre les plis :
![08_ssao-20230904-ssao1.png](08_ssao-20230904-ssao1.png)
Bien qu'il ne s'agisse pas d'un effet incroyablement évident, l'image avec l'occlusion ambiante activée semble beaucoup plus réaliste grâce à ces petits détails ressemblant à des occlusions, ce qui donne à l'ensemble de la scène une plus grande impression de profondeur.

Les techniques d'occlusion ambiante sont coûteuses car elles doivent prendre en compte la géométrie environnante. On pourrait tirer un grand nombre de rayons pour chaque point de l'espace afin de déterminer son degré d'occlusion, mais cela devient rapidement impossible à calculer pour des solutions en temps réel. En 2007, Crytek a publié une technique appelée "**screen-space ambient occlusion**" (**SSAO**) pour son titre Crysis. Cette technique utilise le tampon de profondeur d'une scène dans l'espace-écran pour déterminer la quantité d'occlusion au lieu de données géométriques réelles. Cette approche est incroyablement rapide par rapport à l'occlusion ambiante réelle et donne des résultats plausibles, ce qui en fait la norme de facto pour l'approximation de l'occlusion ambiante en temps réel.

Les principes de base de l'occlusion ambiante dans l'espace-écran sont simples : pour chaque fragment d'un quad qui remplit l'écran, nous calculons un facteur d'occlusion basé sur les valeurs de profondeur environnantes du fragment. Le facteur d'occlusion est ensuite utilisé pour réduire ou annuler la composante d'éclairage ambiant du fragment. Le facteur d'occlusion est obtenu en prenant plusieurs échantillons de profondeur dans un kernel d'échantillons de sphère entourant la position du fragment et en comparant chacun des échantillons avec la valeur de profondeur du fragment actuel. Le nombre d'échantillons dont la valeur de profondeur est supérieure à celle du fragment représente le facteur d'occlusion.
![08_ssao-20230904-ssao2.png](08_ssao-20230904-ssao2.png)
Chacun des échantillons de profondeur de gris situés à l'intérieur de la géométrie contribue au facteur d'occlusion total ; plus nous trouvons d'échantillons à l'intérieur de la géométrie, moins le fragment devrait recevoir d'éclairage ambiant.

Il est clair que la qualité et la précision de l'effet sont directement liées au nombre d'échantillons environnants que nous prenons. Si le nombre d'échantillons est trop faible, la précision diminue considérablement et nous obtenons un artefact appelé "**banding**" ; s'il est trop élevé, nous perdons en performance. Nous pouvons réduire le nombre d'échantillons à tester en introduisant un peu de hasard dans le kernel de l'échantillon. En faisant tourner au hasard le kernel de l'échantillon à chaque fragment, nous pouvons obtenir des résultats de haute qualité avec un nombre d'échantillons beaucoup plus réduit. **Cela a un prix, car le caractère aléatoire introduit un bruit perceptible que nous devrons corriger en rendant les résultats flous**. L'image ci-dessous (avec l'aimable autorisation de John Chapman) illustre l'effet de bande et l'effet du hasard sur les résultats :
![08_ssao-20230904-ssao3.png](08_ssao-20230904-ssao3.png)
Comme vous pouvez le constater, même si les résultats SSAO présentent des bandes visibles en raison du faible nombre d'échantillons, l'introduction d'un élément aléatoire permet de supprimer complètement les effets de bande.

La méthode SSAO développée par Crytek avait un certain style visuel. Le kernel d'échantillonnage utilisé étant une sphère, les murs plats paraissent gris car la moitié des échantillons du kernel se retrouvent dans la géométrie environnante. Vous trouverez ci-dessous une image de l'occlusion ambiante de l'espace-écran de Crysis qui illustre clairement cette impression de gris :
![08_ssao-20230904-ssao4.png](08_ssao-20230904-ssao4.png)
C'est pourquoi nous n'utiliserons pas un kernel d'échantillonnage de sphère, mais plutôt un kernel d'échantillonnage d'hémisphère orienté le long du vecteur normal d'une surface.
![08_ssao-20230904-ssao5.png](08_ssao-20230904-ssao5.png)
En échantillonnant autour de cet hémisphère orienté vers la normale, nous ne considérons pas la géométrie sous-jacente du fragment comme une contribution au facteur d'occlusion. Cela supprime la sensation de gris de l'occlusion ambiante et produit généralement des résultats plus réalistes. La technique de ce chapitre est basée sur cette méthode de l'hémisphère orienté normal et sur une version légèrement modifiée du brillant tutoriel SSAO de John Chapman.

## Tampons pour les échantillons
SSAO nécessite des informations géométriques car nous avons besoin d'un moyen de déterminer le facteur d'occlusion d'un fragment. Pour chaque fragment, nous aurons besoin des données suivantes :

- Un vecteur **position** par fragment.
- Un vecteur **normal** par fragment.
- Une couleur **albedo** par fragment.
- Un kernel d'échantillon**.
- Un vecteur de **rotation aléatoire** par fragment, utilisé pour faire pivoter le kernel de l'échantillon.

En utilisant une position par fragment dans l'espace visuel, nous pouvons orienter un échantillon de kernel hémisphérique autour de la normale à la surface du fragment dans l'espace visuel et utiliser ce kernel pour échantillonner la texture du tampon de position à des décalages variables. Pour chaque échantillon de kernel par fragment, nous comparons sa profondeur avec sa profondeur dans le tampon de position pour déterminer le degré d'occlusion. Le facteur d'occlusion résultant est ensuite utilisé pour limiter la composante finale de l'éclairage ambiant. En incluant également un vecteur de rotation par fragment, nous pouvons réduire de manière significative le nombre d'échantillons que nous devrons prendre, comme nous le verrons bientôt.
![08_ssao-20230904-ssao6.png](08_ssao-20230904-ssao6.png)
Le SSAO étant une technique d'espace-écran, nous calculons son effet sur chaque fragment dans un quad 2D qui remplit l'écran. Cela signifie que nous ne disposons d'aucune information géométrique sur la scène. Ce que nous pourrions faire, c'est rendre les données géométriques par fragment dans des textures de l'espace-écran que nous envoyons ensuite au shader SSAO afin d'avoir accès aux données géométriques par fragment. Si vous avez suivi le chapitre précédent, vous vous rendrez compte que cela ressemble beaucoup à la configuration du tampon G d'un moteur de rendu différé. C'est pourquoi SSAO est parfaitement adapté à la combinaison avec le rendu différé, car nous disposons déjà des vecteurs de position et de normalité dans le G-buffer.

>Dans ce chapitre, nous allons implémenter SSAO au-dessus d'une version légèrement simplifiée du [moteur de rendu différé](moteur%20de%20rendu%20différé) (todo: link) du chapitre sur l'ombrage différé. Si vous n'êtes pas sûr de ce qu'est l'ombrage différé, lisez d'abord ce chapitre.

Comme nous devrions disposer des données de position et de normale par fragment à partir des objets de la scène, le shader de fragment de l'étape de géométrie est assez simple :
```cpp
#version 330 core
layout (location = 0) out vec4 gPosition;
layout (location = 1) out vec3 gNormal;
layout (location = 2) out vec4 gAlbedoSpec;

in vec2 TexCoords;
in vec3 FragPos;
in vec3 Normal;

void main()
{    
    // store the fragment position vector in the first gbuffer texture
    gPosition = FragPos;
    // also store the per-fragment normals into the gbuffer
    gNormal = normalize(Normal);
    // and the diffuse per-fragment color, ignore specular
    gAlbedoSpec.rgb = vec3(0.95);
}
```

Puisque le SSAO est une technique d'espace-écran où l'occlusion est calculée à partir de la vue visible, il est logique d'implémenter l'algorithme dans l'espace-écran. Par conséquent, `FragPos` et `Normal`, tels qu'ils sont fournis par le vertex shader de l'étape géométrique, sont transformés en espace de vue (multipliés également par la matrice de vue).

> Il est possible de reconstruire les vecteurs de position à partir des seules valeurs de profondeur, en utilisant quelques astuces astucieuses décrites par Matt Pettineo sur son blog. Cela nécessite quelques calculs supplémentaires dans les shaders, mais nous évite d'avoir à stocker les données de position dans le G-buffer (qui coûte beaucoup de mémoire). Pour les besoins d'un exemple plus simple, nous laisserons ces optimisations en dehors du chapitre.

La texture du tampon de couleur `gPosition` est configurée comme suit :
```cpp
glGenTextures(1, &gPosition);
glBindTexture(GL_TEXTURE_2D, gPosition);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA16F, SCR_WIDTH, SCR_HEIGHT, 0, GL_RGBA, GL_FLOAT, NULL);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE); 
```
Cela nous donne une texture de position que nous pouvons utiliser pour obtenir des valeurs de profondeur pour chacun des échantillons du kernel. Notez que nous stockons les positions dans un format de données à virgule flottante ; de cette façon, les valeurs de position ne sont pas limitées à $[0.0,1.0]$ et nous avons besoin d'une plus grande précision. Notez également que la méthode d'enveloppement de la texture est `GL_CLAMP_TO_EDGE`. Cela permet de s'assurer que nous ne suréchantillonnons pas accidentellement les valeurs de position/profondeur dans l'espace-écran en dehors de la région de coordonnées par défaut de la texture.

Ensuite, nous avons besoin du kernel d'échantillonnage de l'hémisphère et d'une méthode pour le faire pivoter de manière aléatoire.

## Hémisphère orienté vers la normale
Nous devons générer un certain nombre d'échantillons orientés le long de la normale d'une surface. Comme nous l'avons brièvement évoqué au début de ce chapitre, nous voulons générer des échantillons qui forment un hémisphère. Comme il est difficile et peu plausible de générer un kernel d'échantillons pour chaque direction de la normale à la surface, nous allons générer un kernel d'échantillons dans l'espace tangent, avec le vecteur normal pointant dans la direction z positive.
![08_ssao-20230904-ssao7.png](08_ssao-20230904-ssao7.png)
En supposant que nous disposons d'un hémisphère unitaire, nous pouvons obtenir un kernel d'échantillonnage avec un maximum de 64 valeurs d'échantillonnage comme suit :
```cpp
std::uniform_real_distribution<float> randomFloats(0.0, 1.0); // random floats between [0.0, 1.0]
std::default_random_engine generator;
std::vector<glm::vec3> ssaoKernel;
for (unsigned int i = 0; i < 64; ++i)
{
    glm::vec3 sample(
        randomFloats(generator) * 2.0 - 1.0, 
        randomFloats(generator) * 2.0 - 1.0, 
        randomFloats(generator)
    );
    sample  = glm::normalize(sample);
    sample *= randomFloats(generator);
    ssaoKernel.push_back(sample);  
}
```
Nous faisons varier les directions x et y dans l'espace tangent entre $-1.0$ et $1.0$, et la direction z des échantillons entre $0.0$ et $1.0$ (si nous faisions varier la direction z entre $-1.0$ et $1.0$ également, nous aurions un kernel d'échantillon sphérique). Comme le kernel d'échantillons sera orienté le long de la normale à la surface, les vecteurs d'échantillons résultants se retrouveront tous dans l'hémisphère.

Actuellement, tous les échantillons sont distribués de manière aléatoire dans le kernel d'échantillonnage, mais nous préférons accorder plus d'importance aux occlusions proches du fragment réel. Nous voulons distribuer plus d'échantillons du kernel près de l'origine. Nous pouvons le faire à l'aide d'une fonction d'interpolation accélérée :
```cpp
float scale = (float)i / 64.0; 
   scale   = lerp(0.1f, 1.0f, scale * scale);
   sample *= scale;
   ssaoKernel.push_back(sample);  
}
```

Où la fonction `lerp` est définie comme suit:
```cpp
float lerp(float a, float b, float f)
{
    return a + f * (b - a);
}
```
Nous obtenons ainsi une distribution du kernel qui place la plupart des échantillons plus près de son origine.
![08_ssao-20230904-ssao8.png](08_ssao-20230904-ssao8.png)
Chacun des échantillons du kernel sera utilisé pour décaler la position du fragment dans l'espace visuel afin d'échantillonner la géométrie environnante. Nous avons besoin d'un grand nombre d'échantillons dans l'espace visuel pour obtenir des résultats réalistes, ce qui peut s'avérer trop lourd en termes de performances. Cependant, si nous pouvons introduire une rotation/un bruit semi-aléatoire par fragment, nous pouvons réduire de manière significative le nombre d'échantillons requis.

## Rotations du kernel au hasard
En introduisant un peu d'aléatoire dans les kernels d'échantillonnage, nous réduisons considérablement le nombre d'échantillons nécessaires pour obtenir de bons résultats. Nous pourrions créer un vecteur de rotation aléatoire pour chaque fragment d'une scène, mais cela consomme rapidement de la mémoire. Il est plus judicieux de créer une petite texture de vecteurs de rotation aléatoire que nous répartissons sur l'écran.

Nous créons un tableau $4\times4$ de vecteurs de rotation aléatoires orientés autour de la normale à la surface de l'espace tangent :
```cpp
std::vector<glm::vec3> ssaoNoise;
for (unsigned int i = 0; i < 16; i++)
{
    glm::vec3 noise(
        randomFloats(generator) * 2.0 - 1.0, 
        randomFloats(generator) * 2.0 - 1.0, 
        0.0f); 
    ssaoNoise.push_back(noise);
}
```
Comme le kernel d'échantillonnage est orienté le long de la direction z positive dans l'espace tangent, nous laissons la composante z à $0.0$ afin d'effectuer une rotation autour de l'axe z.

Nous créons ensuite une texture $4\times 4$ qui contient les vecteurs de rotation aléatoires ; assurez-vous de régler sa méthode de wrapping sur `GL_REPEAT` pour qu'elle s'étende correctement sur l'écran.
```cpp
unsigned int noiseTexture; 
glGenTextures(1, &noiseTexture);
glBindTexture(GL_TEXTURE_2D, noiseTexture);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA16F, 4, 4, 0, GL_RGB, GL_FLOAT, &ssaoNoise[0]);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);  
```
Nous disposons à présent de toutes les données d'entrée nécessaires à la mise en œuvre du SSAO.
## Le shader SSAO
Le shader SSAO s'exécute sur un quad 2D qui remplit l'écran et calcule la valeur d'occlusion pour chacun de ses fragments. Comme nous devons stocker le résultat de l'étape SSAO (pour l'utiliser dans le shader d'éclairage final), nous créons encore un autre objet framebuffer :
```cpp
unsigned int ssaoFBO;
glGenFramebuffers(1, &ssaoFBO);  
glBindFramebuffer(GL_FRAMEBUFFER, ssaoFBO);
  
unsigned int ssaoColorBuffer;
glGenTextures(1, &ssaoColorBuffer);
glBindTexture(GL_TEXTURE_2D, ssaoColorBuffer);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RED, SCR_WIDTH, SCR_HEIGHT, 0, GL_RED, GL_FLOAT, NULL);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
  
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, ssaoColorBuffer, 0); 
```
Comme le résultat de l'occlusion ambiante est une valeur unique en niveaux de gris, nous n'aurons besoin que de la composante rouge d'une texture, c'est pourquoi nous définissons le format interne du tampon de couleur sur `GL_RED`.

Le processus complet de rendu du SSAO ressemble alors à ceci :
```cpp
// geometry pass: render stuff into G-buffer
glBindFramebuffer(GL_FRAMEBUFFER, gBuffer);
    [...]
glBindFramebuffer(GL_FRAMEBUFFER, 0);  
  
// use G-buffer to render SSAO texture
glBindFramebuffer(GL_FRAMEBUFFER, ssaoFBO);
    glClear(GL_COLOR_BUFFER_BIT);    
    glActiveTexture(GL_TEXTURE0);
    glBindTexture(GL_TEXTURE_2D, gPosition);
    glActiveTexture(GL_TEXTURE1);
    glBindTexture(GL_TEXTURE_2D, gNormal);
    glActiveTexture(GL_TEXTURE2);
    glBindTexture(GL_TEXTURE_2D, noiseTexture);
    shaderSSAO.use();
    SendKernelSamplesToShader();
    shaderSSAO.setMat4("projection", projection);
    RenderQuad();
glBindFramebuffer(GL_FRAMEBUFFER, 0);
  
// lighting pass: render scene lighting
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
shaderLightingPass.use();
[...]
glActiveTexture(GL_TEXTURE3);
glBindTexture(GL_TEXTURE_2D, ssaoColorBuffer);
[...]
RenderQuad();  
```
Le shader SSAO prend en entrée les textures du tampon G, la texture de bruit et les échantillons du kernel de l'hémisphère orienté vers la normale :

```cpp
#version 330 core
out float FragColor;
  
in vec2 TexCoords;

uniform sampler2D gPosition;
uniform sampler2D gNormal;
uniform sampler2D texNoise;

uniform vec3 samples[64];
uniform mat4 projection;

// tile noise texture over screen, based on screen dimensions divided by noise size
const vec2 noiseScale = vec2(800.0/4.0, 600.0/4.0); // screen = 800x600

void main()
{
    [...]
}
```
Il est intéressant de noter ici la variable `noiseScale`. Nous voulons placer la texture de bruit sur tout l'écran, mais comme les `TexCoords` varient entre $0.0$ et $1.0$, la texture `texNoise` ne sera pas placée sur l'ensemble de l'écran. Nous allons donc calculer la quantité nécessaire pour mettre à l'échelle les `TexCoords` en divisant les dimensions de l'écran par la taille de la texture de bruit.
```cpp
vec3 fragPos   = texture(gPosition, TexCoords).xyz;
vec3 normal    = texture(gNormal, TexCoords).rgb;
vec3 randomVec = texture(texNoise, TexCoords * noiseScale).xyz; 
```
Comme nous avons réglé les paramètres de tiling de `texNoise` sur `GL_REPEAT`, les valeurs aléatoires seront répétées sur tout l'écran. Avec le vecteur `fragPos` et le vecteur normal, nous disposons alors de suffisamment de données pour créer une matrice TBN qui transforme n'importe quel vecteur de l'espace tangent à l'espace visuel :
```cpp
vec3 tangent   = normalize(randomVec - normal * dot(randomVec, normal));
vec3 bitangent = cross(normal, tangent);
mat3 TBN       = mat3(tangent, bitangent, normal); 
```
En utilisant un processus appelé **processus de Gramm-Schmidt**, nous créons une base orthogonale, chaque fois légèrement inclinée en fonction de la valeur de `randomVec`. Notez que comme nous utilisons un vecteur aléatoire pour construire le vecteur tangent, il n'est pas nécessaire que la matrice TBN soit exactement alignée sur la surface de la géométrie, et il n'est donc pas nécessaire d'avoir des vecteurs tangents (et bitangents) par sommet.

Ensuite, nous itérons sur chacun des échantillons du kernel, transformons les échantillons de la tangente à l'espace visuel, les ajoutons à la position actuelle du fragment et comparons la profondeur de la position du fragment avec la profondeur de l'échantillon stockée dans le tampon de position de l'espace visuel. Voyons cela étape par étape :
```cpp
float occlusion = 0.0;
for(int i = 0; i < kernelSize; ++i)
{
    // get sample position
    vec3 samplePos = TBN * samples[i]; // from tangent to view-space
    samplePos = fragPos + samplePos * radius; 
    
    [...]
} 
```
Ici, `kernelSize` et radius sont des variables que nous pouvons utiliser pour ajuster l'effet ; dans ce cas, une valeur de $64$ et $0.5$ respectivement. Pour chaque itération, nous transformons d'abord l'échantillon respectif en espace-temps. Nous ajoutons ensuite l'échantillon de décalage du kernel de l'espace visuel à la position du fragment de l'espace visuel. Nous multiplions ensuite l'échantillon décalé par le rayon pour augmenter (ou diminuer) le rayon d'échantillonnage effectif du **SSAO**.

Ensuite, nous voulons transformer l'échantillon en espace-écran afin de pouvoir échantillonner la valeur de position/profondeur de l'échantillon comme si nous rendions sa position directement à l'écran. Comme le vecteur se trouve actuellement dans l'espace de vue, nous allons d'abord le transformer en espace-clip à l'aide de la matrice de projection uniforme :

```cpp
vec4 offset = vec4(samplePos, 1.0);
offset      = projection * offset;    // from view to clip-space
offset.xyz /= offset.w;               // perspective divide
offset.xyz  = offset.xyz * 0.5 + 0.5; // transform to range 0.0 - 1.0  
```
Une fois la variable transformée en espace-clip, nous effectuons l'étape de division de la perspective en divisant ses composantes xyz avec sa composante w. Les coordonnées normalisées du dispositif qui en résultent sont ensuite transformées dans la plage $[0,0, 1,0]$ afin que nous puissions les utiliser pour échantillonner la texture de la position :

```cpp
float sampleDepth = texture(gPosition, offset.xy).z; 
```

Nous utilisons les composantes x et y du vecteur de décalage pour échantillonner la texture de la position afin de récupérer la profondeur (ou la valeur z) de la position de l'échantillon telle qu'elle est vue du point de vue de l'observateur (le premier fragment visible non occulté). Nous vérifions ensuite si la valeur de profondeur actuelle de l'échantillon est supérieure à la valeur de profondeur stockée et, si c'est le cas, nous l'ajoutons au facteur de contribution final :

```cpp
occlusion += (sampleDepth >= samplePos.z + bias ? 1.0 : 0.0);
```
Notez que nous ajoutons ici un petit biais à la valeur de profondeur du fragment d'origine (fixée à $0.025$ dans cet exemple). Un biais n'est pas toujours nécessaire, mais il permet d'ajuster visuellement l'effet SSAO et de résoudre les effets d'acné qui peuvent se produire en fonction de la complexité de la scène.

Nous n'avons pas encore tout à fait terminé, car il reste un petit problème à prendre en compte. Lorsqu'un fragment est testé pour l'occlusion ambiante et qu'il est aligné près du bord d'une surface, il prendra également en compte les valeurs de profondeur des surfaces situées loin derrière la surface testée ; ces valeurs contribueront (à tort) au facteur d'occlusion. Nous pouvons résoudre ce problème en introduisant une vérification de la portée, comme l'illustre l'image suivante (avec l'aimable autorisation de John Chapman) :
![08_ssao-20230909-ssao9.png](08_ssao-20230909-ssao9.png)
Nous introduisons un contrôle de portée qui garantit qu'un fragment contribue au facteur d'occlusion si ses valeurs de profondeur sont comprises dans le rayon de l'échantillon. Nous remplaçons la dernière ligne par :

```cpp
float rangeCheck = smoothstep(0.0, 1.0, radius / abs(fragPos.z - sampleDepth));
occlusion       += (sampleDepth >= samplePos.z + bias ? 1.0 : 0.0) * rangeCheck;
```
Ici, nous avons utilisé la fonction `smoothstep` de GLSL qui interpole son troisième paramètre entre les plages du premier et du deuxième paramètre, en renvoyant $0.0$ si la valeur est inférieure ou égale à son premier paramètre et $1.0$ si elle est égale ou supérieure à son deuxième paramètre. Si la différence de profondeur se situe entre deux rayons, sa valeur est interpolée entre $0.0$ et $1.0$ à l'aide de la courbe suivante :
![08_ssao-20230909-ssao10.png](08_ssao-20230909-ssao10.png)
Si nous devions utiliser un contrôle de plage à coupure dure (?) (hard cut-off range) qui supprimerait brusquement les contributions d'occlusion si les valeurs de profondeur sont en dehors du rayon, nous verrions des frontières évidentes (peu attrayantes) à l'endroit où le contrôle de plage est appliqué.

La dernière étape consiste à normaliser la contribution de l'occlusion en fonction de la taille du kernel et à produire les résultats. Notez que nous soustrayons le facteur d'occlusion de $1.0$ afin de pouvoir utiliser directement le facteur d'occlusion pour mettre à l'échelle la composante d'éclairage ambiant.

```cpp
}
occlusion = 1.0 - (occlusion / kernelSize);
FragColor = occlusion;  
```
Si nous imaginons une scène où notre modèle de sac à dos préféré fait une petite sieste, le shader d'occlusion ambiante produit la texture suivante :
![08_ssao-20230909-ssao11.png](08_ssao-20230909-ssao11.png)
Comme nous pouvons le voir, l'occlusion ambiante donne une grande impression de profondeur. Avec la seule texture d'occlusion ambiante, nous pouvons déjà voir clairement que le modèle est effectivement posé sur le sol, au lieu de planer légèrement au-dessus.

Le résultat n'est pas encore parfait, car le motif répétitif de la texture de bruit est clairement visible. Pour créer un résultat d'occlusion ambiante lisse, nous devons flouter la texture d'occlusion ambiante.

## Flou d'occlusion ambiante

Entre la passe SSAO et la passe d'éclairage, nous voulons d'abord flouter la texture SSAO. Créons donc un autre objet framebuffer pour stocker le résultat du flou :
```cpp
unsigned int ssaoBlurFBO, ssaoColorBufferBlur;
glGenFramebuffers(1, &ssaoBlurFBO);
glBindFramebuffer(GL_FRAMEBUFFER, ssaoBlurFBO);
glGenTextures(1, &ssaoColorBufferBlur);
glBindTexture(GL_TEXTURE_2D, ssaoColorBufferBlur);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RED, SCR_WIDTH, SCR_HEIGHT, 0, GL_RED, GL_FLOAT, NULL);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, ssaoColorBufferBlur, 0);
```
Comme la texture vectorielle aléatoire en mosaïque nous donne un caractère aléatoire cohérent, nous pouvons utiliser cette propriété à notre avantage pour créer un simple shader de flou :

```cpp
#version 330 core
out float FragColor;
  
in vec2 TexCoords;
  
uniform sampler2D ssaoInput;

void main() {
    vec2 texelSize = 1.0 / vec2(textureSize(ssaoInput, 0));
    float result = 0.0;
    for (int x = -2; x < 2; ++x) 
    {
        for (int y = -2; y < 2; ++y) 
        {
            vec2 offset = vec2(float(x), float(y)) * texelSize;
            result += texture(ssaoInput, TexCoords + offset).r;
        }
    }
    FragColor = result / (4.0 * 4.0);
} 
```
Ici, nous parcourons les texels SSAO environnants entre $-2.0$ et $2.0$, en échantillonnant la texture SSAO d'une quantité identique aux dimensions de la texture de bruit. Nous décalons chaque coordonnée de texture de la taille exacte d'un seul texel en utilisant `textureSize` qui renvoie un `vec2` des dimensions de la texture donnée. Nous faisons la moyenne des résultats obtenus pour obtenir un flou simple mais efficace :
![08_ssao-20230909-ssao12.png](08_ssao-20230909-ssao12.png)
Et voilà, une texture avec des données d'occlusion ambiante par fragment, prête à être utilisée dans la passe d'éclairage.

## Application de l'occlusion ambiante
L'application des facteurs d'occlusion à l'équation d'éclairage est incroyablement facile : tout ce que nous avons à faire est de multiplier le facteur d'occlusion ambiante par fragment à la composante ambiante de l'éclairage et nous avons terminé. Si nous prenons le shader d'éclairage différé Blinn-Phong du chapitre précédent et que nous l'ajustons un peu, nous obtenons le shader de fragment suivant :
```cpp
#version 330 core
out vec4 FragColor;
  
in vec2 TexCoords;

uniform sampler2D gPosition;
uniform sampler2D gNormal;
uniform sampler2D gAlbedo;
uniform sampler2D ssao;

struct Light {
    vec3 Position;
    vec3 Color;
    
    float Linear;
    float Quadratic;
    float Radius;
};
uniform Light light;

void main()
{             
    // retrieve data from gbuffer
    vec3 FragPos = texture(gPosition, TexCoords).rgb;
    vec3 Normal = texture(gNormal, TexCoords).rgb;
    vec3 Diffuse = texture(gAlbedo, TexCoords).rgb;
    float AmbientOcclusion = texture(ssao, TexCoords).r;
    
    // blinn-phong (in view-space)
    vec3 ambient = vec3(0.3 * Diffuse * AmbientOcclusion); // here we add occlusion factor
    vec3 lighting  = ambient; 
    vec3 viewDir  = normalize(-FragPos); // viewpos is (0.0.0) in view-space
    // diffuse
    vec3 lightDir = normalize(light.Position - FragPos);
    vec3 diffuse = max(dot(Normal, lightDir), 0.0) * Diffuse * light.Color;
    // specular
    vec3 halfwayDir = normalize(lightDir + viewDir);  
    float spec = pow(max(dot(Normal, halfwayDir), 0.0), 8.0);
    vec3 specular = light.Color * spec;
    // attenuation
    float dist = length(light.Position - FragPos);
    float attenuation = 1.0 / (1.0 + light.Linear * dist + light.Quadratic * dist * dist);
    diffuse  *= attenuation;
    specular *= attenuation;
    lighting += diffuse + specular;

    FragColor = vec4(lighting, 1.0);
}
```
La seule chose (à part le changement d'espace de vue) que nous avons vraiment changée est la multiplication de la composante ambiante de la scène par `AmbientOcclusion`. Avec une seule lumière ponctuelle bleutée dans la scène, nous obtiendrions le résultat suivant :
![08_ssao-20230909-ssao12-1.png](08_ssao-20230909-ssao12-1.png)
Vous pouvez trouver le code source complet de la scène de démonstration [ici](https://learnopengl.com/code_viewer_gh.php?code=src/5.advanced_lighting/9.ssao/ssao.cpp).

L'occlusion ambiante dans l'espace-écran est un effet hautement personnalisable qui repose en grande partie sur l'ajustement de ses paramètres en fonction du type de scène. Il n'existe pas de combinaison parfaite de paramètres pour chaque type de scène. Certaines scènes ne fonctionnent qu'avec un petit rayon, tandis que d'autres scènes nécessitent un plus grand rayon et un plus grand nombre d'échantillons pour être réalistes. La démo actuelle utilise 64 échantillons, ce qui est un peu trop ; essayez d'obtenir de bons résultats avec une taille de kernel plus petite.

Certains paramètres peuvent être modifiés (en utilisant des uniformes par exemple) : la taille du kernel, le rayon, le biais et/ou la taille du kernel de bruit. Vous pouvez également augmenter la valeur finale de l'occlusion à une puissance définie par l'utilisateur afin d'accroître sa force :
```cpp
occlusion = 1.0 - (occlusion / kernelSize);       
FragColor = pow(occlusion, power);
```
Jouez avec différentes scènes et différents paramètres pour apprécier les possibilités de personnalisation du SSAO.

Même si le SSAO est un effet subtil qui n'est pas très visible, il ajoute beaucoup de réalisme aux scènes correctement éclairées et c'est sans aucun doute une technique que vous voudrez avoir dans votre boîte à outils.

# Ressources supplémentaires
- [SSAO Tutorial](http://john-chapman-graphics.blogspot.nl/2013/01/ssao-tutorial.html) : excellent tutoriel SSAO par John Chapman ; une grande partie du code et des techniques de ce chapitre sont basés sur son article.
- [Know your SSAO artifacts](https://mtnphil.wordpress.com/2013/06/26/know-your-ssao-artifacts/) : excellent article sur l'amélioration des artefacts spécifiques au SSAO.
- [SSAO With Depth Reconstruction](http://ogldev.atspace.co.uk/www/tutorial46/tutorial46.html) : tutoriel d'extension de SSAO par OGLDev sur la reconstruction des vecteurs de position à partir de la profondeur seule, ce qui nous évite de stocker les vecteurs de position coûteux dans le G-buffer.











