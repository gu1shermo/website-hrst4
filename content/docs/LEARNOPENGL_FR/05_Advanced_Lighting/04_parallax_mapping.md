# Le mapping parallaxe
Le mapping parallaxe est une technique similaire au normal mapping, mais basé sur des principes différents. **Tout comme le normal mapping, il s'agit d'une technique qui augmente considérablement les détails d'une surface texturée et lui donne une impression de profondeur**. Bien qu'il s'agisse également d'une illusion, le parallaxe mapping est beaucoup plus efficace pour donner une impression de profondeur et, combinée au normal mapping, il donne des résultats incroyablement réalistes. Bien que le parallaxe mapping ne soit pas nécessairement une technique directement liée à l'éclairage (avancé), j'en parlerai tout de même ici, car cette technique est une suite logique du normal mapping. Notez qu'il est fortement conseillé de se familiariser avec le normal mapping, en particulier l'espace tangent, avant d'apprendre le parallax mapping.

Le parallaxe mapping est étroitement liée à la famille des techniques de mapping de déplacement qui déplacent ou décalent les sommets sur la base d'informations géométriques stockées dans une texture. Une façon de procéder consiste à prendre un plan d'environ 1000 sommets et à déplacer chacun de ces sommets sur la base d'une valeur contenue dans une texture qui nous indique la hauteur du plan à cet endroit précis. Une telle texture qui contient des valeurs de hauteur par texel s'appelle une map de hauteur (height map). Un exemple de carte de hauteur dérivée des propriétés géométriques d'une simple surface de brique ressemble un peu à ceci :
![[04_parallax_mapping-20230903-parallax1.png]]
Lorsqu'il est étendu sur un plan, chaque sommet est déplacé en fonction de la valeur de hauteur échantillonnée dans la map de hauteur, transformant un plan plat en une surface rugueuse et bosselée basée sur les propriétés géométriques d'un matériau. Par exemple, un plan plat déplacé à l'aide de la map de hauteur ci-dessus donne l'image suivante :
![[04_parallax_mapping-20230903-parallax2.png]]
**Le problème du déplacement des sommets de cette manière est qu'un plan doit contenir un très grand nombre de triangles pour obtenir un déplacement réaliste, sinon le déplacement semble trop grossier. **Comme chaque surface plane peut alors nécessiter plus de 10000 vertices, cela devient rapidement infaisable sur le plan informatique. Et si nous pouvions obtenir un réalisme similaire sans avoir recours à des sommets supplémentaires ? En fait, si je vous disais que la surface déplacée montrée précédemment est en fait rendue avec seulement 2 triangles. Cette surface de briques est rendue avec le mapping de parallaxe, une technique de mapping de déplacement qui ne nécessite pas de données de vertex supplémentaires pour transmettre la profondeur, mais qui (comme le normal mapping) utilise une technique astucieuse pour tromper l'utilisateur.

L'idée derrière le parallax mapping est de modifier les coordonnées de la texture de manière à ce que la surface d'un fragment semble plus haute ou plus basse qu'elle ne l'est en réalité, tout cela en fonction de la direction de la vue et d'une map de hauteur. Pour comprendre comment cela fonctionne, regardez l'image suivante de notre surface de briques :
![[04_parallax_mapping-20230903-parallax3.png]]
Ici, la ligne rouge grossière représente les valeurs de la map des hauteurs en tant que représentation géométrique de la surface de la brique et le vecteur $\vec{V}$ représente la direction de la surface par rapport à la vue (`viewDir`). Si le plan avait un déplacement réel, l'observateur verrait la surface au point $B$. Cependant, comme notre plan n'a pas de déplacement réel, la direction de vue est calculée à partir du point $A$, comme on peut s'y attendre. Le mapping parallaxe vise à décaler les coordonnées de texture à la position $A$ du fragment de manière à obtenir des coordonnées de texture au point B. Nous utilisons ensuite les coordonnées de texture au point $B$ pour tous les échantillons de texture suivants, ce qui donne l'impression que l'observateur regarde réellement le point $B$.

L'astuce consiste à trouver comment obtenir les coordonnées de texture au point $B$
à partir du point $A$. Le mapping parallaxe tente de résoudre ce problème en mettant à l'échelle le vecteur de direction fragment-vue $\vec{V}$ par la hauteur du fragment $A$. Nous mettons donc à l'échelle la longueur de $\vec{V}$ pour qu'elle soit égale à une valeur échantillonnée à partir de la carte de hauteur $H(A)$ à la position du fragment $A$. L'image ci-dessous montre ce vecteur mis à l'échelle $\vec{P}$ :
![[04_parallax_mapping-20230903-parallax4.png]]


Nous prenons ensuite ce vecteur $\vec{P}$ et ses coordonnées vectorielles qui s'alignent sur le plan comme décalage des coordonnées de texture. Cela fonctionne parce que le vecteur $\vec{P}$ est calculé à l'aide d'une valeur de hauteur provenant de la map des hauteurs. Ainsi, plus la hauteur d'un fragment est élevée, plus il est déplacé.

Cette petite astuce donne de bons résultats la plupart du temps, mais il s'agit toujours d'une approximation très grossière pour atteindre le point $B$. Lorsque les hauteurs changent rapidement sur une surface, les résultats ont tendance à paraître irréalistes, car le vecteur $\vec{P}$ ne se retrouvera pas près de $B$, comme vous pouvez le voir ci-dessous :
![[04_parallax_mapping-20230903-parallax5.png]]

Un autre problème lié au mapping de parallaxe est qu'il est difficile de déterminer les coordonnées à extraire de $\vec{P}$ lorsque la surface subit une rotation arbitraire. Nous préférons procéder dans un espace de coordonnées différent où les composantes x et y du vecteur $\vec{P}$ s'alignent toujours sur la surface de la texture. Si vous avez suivi le chapitre sur le [[03_normal_mapping|normal mapping]], vous avez probablement deviné comment nous pouvons y parvenir. Et oui, nous aimerions faire de le mapping parallaxe dans l'espace tangent.

En transformant le vecteur de direction fragment-vue $\vec{V}$ dans l'espace tangent, le vecteur $\vec{P}$ transformé aura ses composantes x et y alignées sur les vecteurs tangent et bitangent de la surface. Comme les vecteurs tangents et bitangents pointent dans la même direction que les coordonnées de texture de la surface, nous pouvons considérer les composantes x et y de $\vec{P}$ comme le décalage des coordonnées de texture, quelle que soit l'orientation de la surface.

Mais assez de théorie, mettons-nous à l'œuvre et commençons à implémenter le mapping parallaxe réel.

## Mapping parallaxe
Pour la mapping de parallaxe, nous allons utiliser un simple plan 2D dont nous avons calculé les vecteurs tangents et bitangents avant de l'envoyer au GPU, comme nous l'avons fait dans le chapitre sur la mapping de normales. Sur ce plan, nous allons attacher une [texture diffuse](https://learnopengl.com/img/textures/bricks2.jpg), une [map de normales](https://learnopengl.com/img/textures/bricks2_normal.jpg) et une [map de déplacement](https://learnopengl.com/img/textures/bricks2_disp.jpg) que vous pouvez télécharger à partir de leurs URL.
Pour cet exemple, nous allons utiliser le mapping de parallaxe en conjonction avec le mapping de normales. Comme le mapping de parallaxe donne l'illusion de déplacer une surface, l'illusion est rompue lorsque l'éclairage ne correspond pas. **Comme les maps de normales sont souvent générées à partir de maps de hauteurs**, l'utilisation d'une map de normales avec la map de hauteurs permet de s'assurer que l'éclairage est en place avec le déplacement.

Vous avez peut-être déjà remarqué que la map de déplacement liée ci-dessus est l'inverse de la carte de hauteur montrée au début de ce chapitre. Avec le mapping de parallaxe, il est plus logique d'utiliser l'inverse de la map de hauteur car il est plus facile de simuler la profondeur que la hauteur sur des surfaces planes. Cela modifie légèrement la façon dont nous percevons le mapping de parallaxe, comme le montre l'illustration ci-dessous :
![[04_parallax_mapping-20230903-parallax6.png]]
Nous avons à nouveau les points $A$ et $B$, mais cette fois nous obtenons le vecteur $\vec{P}$ en soustrayant le vecteur $\vec{V}$ des coordonnées de texture au point $A$. Nous pouvons obtenir des valeurs de profondeur au lieu de valeurs de hauteur en soustrayant les valeurs de la map de hauteur échantillonnée de $1.0$ dans les shaders, ou en inversant simplement ses valeurs de texture dans un logiciel d'édition d'images, comme nous l'avons fait avec la map de profondeur mentionnée ci-dessus.

Le mapping de parallaxe est implémenté dans le shader de fragment car l'effet de déplacement est différent sur toute la surface d'un triangle. Dans le shader de fragment, nous allons devoir calculer le vecteur de direction fragment-vue $\vec{V}$.
Nous avons donc besoin de la position de la vue et de la position du fragment dans l'espace tangent. Dans le chapitre sur le normal mapping, nous avions déjà un shader de sommets qui envoyait ces vecteurs dans l'espace tangent, nous pouvons donc prendre une copie exacte du shader de sommets de ce chapitre :
```cpp
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aNormal;
layout (location = 2) in vec2 aTexCoords;
layout (location = 3) in vec3 aTangent;
layout (location = 4) in vec3 aBitangent;

out VS_OUT {
    vec3 FragPos;
    vec2 TexCoords;
    vec3 TangentLightPos;
    vec3 TangentViewPos;
    vec3 TangentFragPos;
} vs_out;

uniform mat4 projection;
uniform mat4 view;
uniform mat4 model;

uniform vec3 lightPos;
uniform vec3 viewPos;

void main()
{
    gl_Position      = projection * view * model * vec4(aPos, 1.0);
    vs_out.FragPos   = vec3(model * vec4(aPos, 1.0));   
    vs_out.TexCoords = aTexCoords;    
    
    vec3 T   = normalize(mat3(model) * aTangent);
    vec3 B   = normalize(mat3(model) * aBitangent);
    vec3 N   = normalize(mat3(model) * aNormal);
    mat3 TBN = transpose(mat3(T, B, N));

    vs_out.TangentLightPos = TBN * lightPos;
    vs_out.TangentViewPos  = TBN * viewPos;
    vs_out.TangentFragPos  = TBN * vs_out.FragPos;
}
```
Dans le fragment shader, nous implémentons ensuite la logique de mapping parallaxe. Le fragment shader ressemble un peu à ceci :
```cpp
#version 330 core
out vec4 FragColor;

in VS_OUT {
    vec3 FragPos;
    vec2 TexCoords;
    vec3 TangentLightPos;
    vec3 TangentViewPos;
    vec3 TangentFragPos;
} fs_in;

uniform sampler2D diffuseMap;
uniform sampler2D normalMap;
uniform sampler2D depthMap;
  
uniform float height_scale;
  
vec2 ParallaxMapping(vec2 texCoords, vec3 viewDir);
  
void main()
{           
    // offset texture coordinates with Parallax Mapping
    vec3 viewDir   = normalize(fs_in.TangentViewPos - fs_in.TangentFragPos);
    vec2 texCoords = ParallaxMapping(fs_in.TexCoords,  viewDir);

    // then sample textures with new texture coords
    vec3 diffuse = texture(diffuseMap, texCoords);
    vec3 normal  = texture(normalMap, texCoords);
    normal = normalize(normal * 2.0 - 1.0);
    // proceed with lighting code
    [...]    
}
```
Nous avons défini une fonction appelée `ParallaxMapping` qui prend en entrée les coordonnées de texture du fragment et la direction fragment-vue $\vec{V}$ dans l'espace tangent. La fonction renvoie les coordonnées de texture déplacées. Nous utilisons ensuite ces coordonnées de texture déplacées comme coordonnées de texture pour l'échantillonnage de la map diffuse et normale. Par conséquent, le vecteur diffus et normal du fragment correspond correctement à la géométrie déplacée de la surface.

Jetons un coup d'œil à l'intérieur de la fonction `ParallaxMapping` :
```cpp
vec2 ParallaxMapping(vec2 texCoords, vec3 viewDir)
{ 
    float height =  texture(depthMap, texCoords).r;    
    vec2 p = viewDir.xy / viewDir.z * (height * height_scale);
    return texCoords - p;    
}
```
Cette fonction relativement simple est une traduction directe de ce que nous avons discuté jusqu'à présent. Nous prenons les coordonnées de texture originales `texCoords` et les utilisons pour échantillonner la hauteur (ou la profondeur) de la map des profondeurs au niveau du fragment courant $A$ sous la forme $H(A)$. Nous calculons ensuite $\vec{P}$ comme les composantes x et y du vecteur `viewDir` de l'espace tangent divisé par sa composante z et mis à l'échelle par $H(A)$. Nous avons également introduit un uniforme $height_scale$ pour un contrôle supplémentaire, car l'effet de parallaxe est généralement trop important sans paramètre d'échelle supplémentaire. Nous soustrayons ensuite ce vecteur $\vec{P}$ des coordonnées de la texture pour obtenir les coordonnées finales de la texture déplacée.

Il est intéressant de noter ici la division de `viewDir.xy` par `viewDir.z`.
Comme le vecteur `viewDir` est normalisé, `viewDir.z` sera compris entre $0.0$ et $1.0$. Lorsque `viewDir` est largement parallèle à la surface, sa composante z est proche de $0.0$ et la division renvoie un vecteur $\vec{P}$ beaucoup plus grand que lorsque `viewDir` est largement perpendiculaire à la surface. Nous ajustons la taille de $\vec{P}$ de manière à ce qu'il décale les coordonnées de la texture à une plus grande échelle lorsque l'on regarde une surface depuis un angle que lorsque l'on la regarde depuis le haut ; cela donne des résultats plus réalistes dans les angles.
Certains préfèrent ne pas tenir compte de la division par `viewDir.z`, car le mapping parallaxe par défaut pourrait produire des résultats indésirables dans les angles ; la technique est alors appelée mapping parallaxe avec limitation du décalage. Le choix de la technique est généralement une question de préférence personnelle.

Les coordonnées de la texture résultante sont ensuite utilisées pour échantillonner les autres textures (diffuse et normale), ce qui donne un effet de déplacement très net, comme vous pouvez le voir ci-dessous avec une échelle de hauteur d'environ $0.1$ :

![[04_parallax_mapping-20230903-parallax7.png]]
Ici, vous pouvez voir la différence entre le normal mapping et le parallax mapping combiné au normal mapping. Comme le mapping parallaxe tente de simuler la profondeur, il est possible de faire en sorte que des briques se superposent à d'autres briques en fonction de la direction dans laquelle vous les regardez.

Vous pouvez toujours voir quelques artefacts bizarres sur les bords du plan de parallaxe. Cela se produit parce qu'aux bords du plan, les coordonnées de texture déplacées peuvent être suréchantillonnées en dehors de l'intervalle $[0, 1]$. Cela donne des résultats irréalistes basés sur le(s) mode(s) de wrapping de la texture. Une astuce sympa pour résoudre ce problème est de jeter le fragment chaque fois qu'il échantillonne en dehors de la plage de coordonnées de texture par défaut :
```cpp
texCoords = ParallaxMapping(fs_in.TexCoords,  viewDir);
if(texCoords.x > 1.0 || texCoords.y > 1.0 || texCoords.x < 0.0 || texCoords.y < 0.0)
    discard;
```
Tous les fragments dont les coordonnées de texture (déplacées) se situent en dehors de la plage par défaut sont éliminés et le Parallax Mapping donne alors des résultats corrects sur les bords d'une surface. Notez que cette astuce ne fonctionne pas sur tous les types de surfaces, mais lorsqu'elle est appliquée à un plan, elle donne d'excellents résultats :
![[04_parallax_mapping-20230903-parallax8.png]]
Vous pouvez trouver le code source [ici](https://learnopengl.com/code_viewer_gh.php?code=src/5.advanced_lighting/5.1.parallax_mapping/parallax_mapping.cpp).

C'est très joli et assez rapide car nous n'avons besoin que d'un seul échantillon de texture supplémentaire pour que le parallax mapping fonctionne. Il y a cependant quelques problèmes, car il se décompose en quelque sorte lorsqu'on le regarde d'un angle (comme pour le normal mapping) et donne des résultats incorrects avec des changements de hauteur importants, comme vous pouvez le voir ci-dessous :
![[04_parallax_mapping-20230903-parallax9.png]]
La raison pour laquelle cela ne fonctionne pas toujours correctement est qu'il s'agit d'une approximation grossière du displacement mapping. Il existe cependant quelques astuces supplémentaires qui nous permettent d'obtenir des résultats presque parfaits avec des changements de hauteur importants, même en regardant sous un angle. Par exemple, que se passerait-il si, au lieu d'un échantillon, nous prenions plusieurs échantillons pour trouver le point le plus proche de $B$?

## Mapping de la parallaxe abrupte (Steep Parallax Mapping)
Le mapping parallaxe abrupte est une extension du mapping parallaxe en ce sens qu'il utilise les mêmes principes, mais au lieu d'un seul échantillon, il prend plusieurs échantillons pour mieux situer le vecteur $\vec{P}$ par rapport à $B$.
On obtient ainsi de bien meilleurs résultats, même en cas de fortes variations de hauteur, car la précision de la technique est améliorée par le nombre d'échantillons.

L'idée générale du mapping parallaxe à forte pente (abrupte?) est de diviser la plage de profondeur totale en plusieurs couches de même hauteur/profondeur. Pour chacune de ces couches, nous échantillonnons la map des profondeurs, en déplaçant les coordonnées de la texture le long de la direction de $\vec{P}$
jusqu'à ce que nous trouvions une valeur de profondeur échantillonnée inférieure à la valeur de profondeur de la couche actuelle. Regardez l'image suivante :
![[04_parallax_mapping-20230903-parallax10.png]]
Nous parcourons les couches de profondeur de haut en bas et, pour chaque couche, nous comparons sa valeur de profondeur à la valeur de profondeur stockée dans la map des profondeurs. Si la valeur de profondeur de la couche est inférieure à celle de la map des profondeurs, cela signifie que la partie du vecteur $\vec{P}$ n'est pas sous la surface. Nous poursuivons ce processus jusqu'à ce que la profondeur de la couche soit supérieure à la valeur stockée dans la map des profondeurs : ce point se trouve alors sous la surface géométrique (déplacée).

Dans cet exemple, nous pouvons voir que la valeur de la map des profondeurs à la deuxième couche ($D(2) = 0.73$) est inférieure à la valeur de la profondeur de la deuxième couche ($0.4$), nous continuons donc. À l'itération suivante, la valeur de profondeur de la couche $0.6$ est supérieure à la valeur de profondeur échantillonnée de la carte des profondeurs ($D(3) = 0.37$). Nous pouvons donc supposer que le vecteur $\vec{P}$ à la troisième couche comme étant la position la plus viable de la géométrie déplacée. Nous prenons ensuite le décalage de coordonnées de texture $T_3$ du vecteur $\vec{P}_3$ pour déplacer les coordonnées de texture du fragment. Vous pouvez constater que la précision augmente avec la profondeur des couches.

Pour mettre en œuvre cette technique, il suffit de modifier la fonction `ParallaxMapping`, car nous disposons déjà de toutes les variables nécessaires :
```cpp
vec2 ParallaxMapping(vec2 texCoords, vec3 viewDir)
{ 
    // number of depth layers
    const float numLayers = 10;
    // calculate the size of each layer
    float layerDepth = 1.0 / numLayers;
    // depth of current layer
    float currentLayerDepth = 0.0;
    // the amount to shift the texture coordinates per layer (from vector P)
    vec2 P = viewDir.xy * height_scale; 
    vec2 deltaTexCoords = P / numLayers;
  
    [...]     
}
```
Nous commençons par mettre les choses en place : nous spécifions le nombre de couches, nous calculons le décalage de profondeur de chaque couche, et enfin nous calculons le décalage des coordonnées de texture que nous devons déplacer le long de la direction de $\vec{P}$ par couche.

Nous itérons ensuite sur toutes les couches, en commençant par le haut, jusqu'à ce que nous trouvions une valeur de map de profondeur inférieure à la valeur de profondeur de la couche :
```cpp
// get initial values
vec2  currentTexCoords     = texCoords;
float currentDepthMapValue = texture(depthMap, currentTexCoords).r;
  
while(currentLayerDepth < currentDepthMapValue)
{
    // shift texture coordinates along direction of P
    currentTexCoords -= deltaTexCoords;
    // get depthmap value at current texture coordinates
    currentDepthMapValue = texture(depthMap, currentTexCoords).r;  
    // get depth of next layer
    currentLayerDepth += layerDepth;  
}

return currentTexCoords;
```
Ici, nous bouclons sur chaque couche de profondeur et nous nous arrêtons jusqu'à ce que nous trouvions le décalage des coordonnées de texture le long du vecteur $\vec{P}$ qui renvoie pour la première fois une profondeur inférieure à la surface (déplacée). Le décalage résultant est soustrait des coordonnées de texture du fragment pour obtenir un vecteur de coordonnées de texture déplacé final, cette fois avec beaucoup plus de précision par rapport au mapping parallaxe traditionnel.

Avec environ 10 échantillons, la surface de la brique semble déjà plus viable même lorsqu'on la regarde d'un angle, mais le mapping de parallaxe abrupte brille vraiment lorsqu'on a une surface complexe avec des changements de hauteur abrupts ; comme la surface du jouet en bois affichée plus tôt :
![[04_parallax_mapping-20230903-parallax11.png]]
Nous pouvons améliorer l'algorithme en exploitant l'une des propriétés du Parallax Mapping. Lorsque l'on regarde directement une surface, il n'y a pas beaucoup de déplacement de texture alors qu'il y a beaucoup de déplacement lorsque l'on regarde une surface depuis un angle (visualisez la direction de la vue dans les deux cas). En prenant moins d'échantillons lorsque l'on regarde directement une surface et plus d'échantillons lorsque l'on regarde d'un angle, nous n'échantillonnons que la quantité nécessaire :
```cpp
const float minLayers = 8.0;
const float maxLayers = 32.0;
float numLayers = mix(maxLayers, minLayers, max(dot(vec3(0.0, 0.0, 1.0), viewDir), 0.0));
```
Ici, nous prenons le produit scalaire de `viewDir` et de la direction z positive et utilisons le résultat pour aligner le nombre d'échantillons sur `minLayers` ou `maxLayers` en fonction de l'angle sous lequel nous regardons la surface (notez que la direction z positive est égale au vecteur normal de la surface dans l'espace tangent). Si nous regardions dans une direction parallèle à la surface, nous utiliserions un total de 32 couches.

Vous pouvez trouver le code source mis à jour [ici](https://learnopengl.com/code_viewer_gh.php?code=src/5.advanced_lighting/5.2.steep_parallax_mapping/steep_parallax_mapping.cpp). Vous pouvez également trouver la surface de la boîte à jouets en bois ici : [diffuse](https://learnopengl.com/img/textures/wood.png), [normale](https://learnopengl.com/img/textures/toy_box_normal.png) et [profondeur](https://learnopengl.com/img/textures/toy_box_disp.png).

Le **Steep Parallax Mapping** a aussi ses problèmes. Comme la technique est basée sur un nombre fini d'échantillons, nous obtenons des effets d'aliasing et les distinctions claires entre les couches peuvent être facilement repérées :
![[04_parallax_mapping-20230903-parallax12.png]]
Nous pouvons réduire ce problème en prenant un plus grand nombre d'échantillons, mais cela devient rapidement une charge trop lourde pour les performances. Plusieurs approches visent à résoudre ce problème en ne prenant pas la première position qui se trouve sous la surface (déplacée), mais en interpolant entre les deux couches de profondeur les plus proches de la position pour trouver une correspondance beaucoup plus étroite avec $B$.

Deux des approches les plus populaires sont appelées **Relief Parallax Mapping** et **Parallax Occlusion Mapping**.
Le Relief Parallax Mapping donne les résultats les plus précis, mais il est aussi plus gourmand en performances que la Parallax Occlusion Mapping.
Comme le mapping des occlusions parallèles donne presque les mêmes résultats que le mapping des occlusions en relief et qu'elle est également plus efficace, elle est souvent l'approche préférée.

## Mapping de l'occlusion parallaxe (Parallax Occlusion Mapping)
Le mapping de l'occlusion parallaxe est basée sur les mêmes principes que le mapping de la parallaxe abrupte, mais au lieu de prendre les coordonnées de texture de la première couche de profondeur après une collision, nous allons interpoler linéairement entre la couche de profondeur après et avant la collision. Nous basons le poids de l'interpolation linéaire sur la distance entre la hauteur de la surface et la valeur de la couche de profondeur des deux couches. Jetez un coup d'œil à l'image suivante pour comprendre comment cela fonctionne :
![[04_parallax_mapping-20230903-parallax13.png]]

Comme vous pouvez le voir, il s'agit d'une méthode largement similaire à la cartographie Steep Parallax, avec une étape supplémentaire, l'interpolation linéaire entre les coordonnées des textures des deux couches de profondeur entourant le point d'intersection. Il s'agit à nouveau d'une approximation, mais elle est nettement plus précise que le Steep Parallax Mapping.

Le code pour le Parallax Occlusion Mapping est une extension du Steep Parallax Mapping et n'est pas trop difficile :
```cpp
[...] // steep parallax mapping code here
  
// get texture coordinates before collision (reverse operations)
vec2 prevTexCoords = currentTexCoords + deltaTexCoords;

// get depth after and before collision for linear interpolation
float afterDepth  = currentDepthMapValue - currentLayerDepth;
float beforeDepth = texture(depthMap, prevTexCoords).r - currentLayerDepth + layerDepth;
 
// interpolation of texture coordinates
float weight = afterDepth / (afterDepth - beforeDepth);
vec2 finalTexCoords = prevTexCoords * weight + currentTexCoords * (1.0 - weight);

return finalTexCoords;  
```
Après avoir trouvé la couche de profondeur après avoir intersecté la géométrie de surface (déplacée), nous récupérons également les coordonnées de texture de la couche de profondeur avant l'intersection. Nous calculons ensuite la distance entre la profondeur de la géométrie (déplacée) et les couches de profondeur correspondantes, puis nous interpolons ces deux valeurs. L'interpolation linéaire est une interpolation de base entre les coordonnées de texture des deux couches. La fonction renvoie ensuite les coordonnées de texture interpolées finales.

Le Parallax Occlusion Mapping donne des résultats étonnamment bons et, bien que de légers artefacts et des problèmes de repliement soient encore visibles, il s'agit généralement d'un bon compromis qui n'est vraiment visible que lorsque l'on zoome fortement ou que l'on regarde sous des angles très prononcés.

![[04_parallax_mapping-20230903-parallax14.png]]
Vous pouvez trouver le code source [ici](https://learnopengl.com/code_viewer_gh.php?code=src/5.advanced_lighting/5.3.parallax_occlusion_mapping/parallax_occlusion_mapping.cpp).

Le mapping parallaxe est une excellente technique pour augmenter les détails de votre scène, mais elle s'accompagne de quelques artefacts que vous devez prendre en compte lorsque vous l'utilisez. Le plus souvent, le mapping parallaxe est utilisée sur des surfaces de type sol ou mur, où il n'est pas facile de déterminer le contour de la surface et où l'angle de vue est le plus souvent à peu près perpendiculaire à la surface. De cette manière, les artefacts du mapping parallaxe ne sont pas aussi visibles et en font une technique incroyablement intéressante pour améliorer les détails de vos objets.

## Ressources supplémentaires
- [Parallax Occlusion Mapping in GLSL](http://sunandblackcat.com/tipFullView.php?topicid=28) : excellent tutoriel de parallax mapping par sunandblackcat.com.
- [How Parallax Displacement Mapping Works](https://www.youtube.com/watch?v=xvOT62L-fQI) : un bon tutoriel vidéo sur le fonctionnement du mapping parallaxe par TheBennyBox.
