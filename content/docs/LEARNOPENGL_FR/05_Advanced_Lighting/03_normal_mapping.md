# Normal mapping
Toutes nos scènes sont remplies de meshes, chacun composé de centaines, voire de milliers de triangles. Nous avons renforcé le réalisme en appliquant des textures 2D sur ces triangles plats, cachant ainsi le fait que les polygones ne sont que de minuscules triangles plats. Les textures sont utiles, mais lorsque l'on examine de près les meshes, il est toujours facile de voir les surfaces planes sous-jacentes. La plupart des surfaces de la vie réelle ne sont cependant pas plates et présentent de nombreux détails (bosses).

Prenons l'exemple d'une surface en briques. La surface d'une brique est assez rugueuse et n'est évidemment pas complètement plate : elle contient des bandes de ciment enfoncées et un grand nombre de petits trous et fissures détaillés. Si nous devions voir une telle surface de briques dans une scène éclairée, l'immersion serait facilement rompue. Ci-dessous, nous pouvons voir une texture de brique appliquée à une surface plane éclairée par une lumière ponctuelle.
![[03_normal_mapping-20230902-normal1.png]]
L'éclairage ne tient compte d'aucune des petites fissures et des trous et ignore complètement les rayures profondes entre les briques ; la surface semble parfaitement plate. Nous pouvons partiellement corriger l'aspect plat en utilisant une carte spéculaire pour prétendre que certaines surfaces sont moins éclairées en raison de la profondeur ou d'autres détails, mais il s'agit plus d'une astuce que d'une véritable solution. Ce qu'il nous faut, c'est un moyen d'informer le système d'éclairage de tous les petits détails de la surface qui ressemblent à de la profondeur.

**Si nous y réfléchissons du point de vue de la lumière, comment se fait-il que la surface soit éclairée comme une surface complètement plate ? La réponse est le vecteur normal de la surface**. Du point de vue de la technique d'éclairage, la seule façon de déterminer la forme d'un objet est son vecteur normal perpendiculaire. La surface de la brique n'a qu'un seul vecteur normal et, par conséquent, la surface est uniformément éclairée en fonction de la direction de ce vecteur normal. Et si, au lieu d'une normale par surface qui est la même pour chaque fragment, nous utilisions une normale par fragment qui est différente pour chaque fragment ? De cette façon, nous pouvons légèrement dévier le vecteur normal en fonction des petits détails d'une surface ; cela donne l'illusion que la surface est beaucoup plus complexe :
![[03_normal_mapping-20230902-normal2.png]]
L'utilisation de normales par fragment permet de faire croire à l'éclairage qu'une surface est constituée de minuscules plans (perpendiculaires aux vecteurs normaux), ce qui augmente considérablement le niveau de détail de la surface. Cette technique d'utilisation des normales par fragment par rapport aux normales par surface est appelée "**normal mapping**" ou "**bump mapping**". Appliquée au plan de la brique, elle ressemble un peu à ceci :
![[03_normal_mapping-20230902-normal3.png]]
Comme vous pouvez le constater, cette méthode permet d'augmenter considérablement le niveau de détail pour un coût relativement faible. Comme nous ne modifions que les vecteurs normaux par fragment, il n'est pas nécessaire de modifier l'équation d'éclairage. Nous transmettons maintenant à l'algorithme d'éclairage une normale par fragment, au lieu d'une normale de surface interpolée. L'éclairage fait le reste.

## Normal mapping
Pour que le normal mapping fonctionne, nous allons avoir besoin d'une normale par fragment. Comme nous l'avons fait avec les cartes diffuse et spéculaire, nous pouvons utiliser une texture 2D pour stocker les données normales par fragment. De cette manière, nous pouvons échantillonner une texture 2D pour obtenir un vecteur normal pour ce fragment spécifique.

Bien que les vecteurs normaux soient des entités géométriques et que les textures ne soient généralement utilisées que pour les informations de couleur, le stockage des vecteurs normaux dans une texture n'est pas toujours évident. Si vous pensez aux vecteurs de couleur dans une texture, ils sont représentés comme un vecteur 3D avec une composante r, g et b. Nous pouvons de la même manière stocker un vecteur normal dans une texture. De la même manière, nous pouvons stocker les composantes x, y et z d'un vecteur normal dans les composantes de couleur correspondantes. Les vecteurs normaux sont compris entre -1 et 1, ils sont donc d'abord mappés sur $[0,1]$ :
```cpp
vec3 rgb_normal = normal * 0.5 + 0.5; // transforms from [-1,1] to [0,1]  
```
Avec des vecteurs normaux transformés en une composante de couleur RVB comme celle-ci, nous pouvons stocker une normale par fragment dérivée de la forme d'une surface sur une texture 2D. Un exemple de map des normales de la surface de briques présentée au début de ce chapitre est illustré ci-dessous :
![[03_normal_mapping-20230902-normal-rgb.png]]
Cette map (et presque toutes les maps de normales que vous trouverez en ligne) aura une teinte bleutée. En effet, les normales sont toutes étroitement orientées vers l'axe z positif (0,0,1) : une couleur bleutée. Les écarts de couleur représentent des vecteurs normaux légèrement décalés par rapport à l'axe z positif général, ce qui donne une impression de profondeur à la texture. Par exemple, vous pouvez voir qu'au sommet de chaque brique, la couleur a tendance à être plus verdâtre, ce qui est logique puisque la face supérieure d'une brique a des normales qui pointent davantage dans la direction positive y (0,1,0), ce qui correspond à la couleur verte !

Avec un simple plan, en regardant l'axe z positif, nous pouvons prendre [cette texture diffuse](https://learnopengl.com/img/textures/brickwall.jpg) et [cette texture de normale](https://learnopengl.com/img/textures/brickwall_normal.jpg) pour rendre l'image de la section précédente. Notez que la map des normales liée est différente de celle montrée ci-dessus. La raison en est qu'OpenGL lit les coordonnées de texture avec la coordonnée y (ou v) inversée par rapport à la façon dont les textures sont généralement créées. La carte des normales liées a donc sa composante y (ou verte) inversée (vous pouvez voir que les couleurs vertes pointent maintenant vers le bas) ; si vous ne prenez pas cela en compte, l'éclairage sera incorrect. Chargez les deux textures, liez-les aux unités de texture appropriées et effectuez le rendu d'un plan avec les modifications suivantes dans le shader de fragment d'éclairage :

```cpp
uniform sampler2D normalMap;  

void main()
{           
    // obtain normal from normal map in range [0,1]
    normal = texture(normalMap, fs_in.TexCoords).rgb;
    // transform normal vector to range [-1,1]
    normal = normalize(normal * 2.0 - 1.0);   
  
    [...]
    // proceed with lighting as normal
}  
```
Ici, nous inversons le processus de mappage des normales aux couleurs RVB en remappant la couleur normale échantillonnée de $[0,1]$ à $[-1,1]$, puis nous utilisons les vecteurs normaux échantillonnés pour les calculs d'éclairage à venir. Dans ce cas, nous avons utilisé un shader Blinn-Phong.

En déplaçant lentement la source lumineuse dans le temps, vous obtenez vraiment une sensation de profondeur en utilisant la map de normale. L'exécution de cet exemple de normal mapping donne les mêmes résultats que ceux présentés au début de ce chapitre :
![[03_normal_mapping-20230902-normal5.png]]
Il existe cependant un problème qui limite considérablement l'utilisation des maps de normales. La map des normales que nous avons utilisée comportait des vecteurs de normales qui pointaient tous dans la direction positive z. Cela fonctionnait parce que la normale de la surface du plan pointait également dans la direction positive z. Cependant, que se passerait-il si nous utilisions la même map des normales sur un plan posé sur le sol dont le vecteur normal de surface pointerait dans la direction positive y ?
![[03_normal_mapping-20230902-normal6.png]]
L'éclairage ne semble pas correct ! Cela est dû au fait que les normales échantillonnées de ce plan pointent toujours grosso modo dans la direction z positive, alors qu'elles devraient plutôt pointer dans la direction y positive. Par conséquent, l'éclairage pense que les normales de la surface sont les mêmes qu'auparavant, lorsque le plan pointait vers la direction z positive ; l'éclairage est incorrect. L'image ci-dessous montre à quoi ressemblent approximativement les normales échantillonnées sur cette surface :
![[03_normal_mapping-20230902-normal8.png]]
Vous pouvez voir que toutes les normales sont orientées vers la direction z positive alors qu'elles devraient être orientées vers la direction y positive. Une solution à ce problème consiste à définir une carte de normales pour chaque direction possible de la surface ; dans le cas d'un cube, nous aurions besoin de 6 cartes de normales. Cependant, avec des meshes avancés qui peuvent avoir plus de centaines de directions de surface possibles, cette approche devient irréalisable.

Il existe une autre solution qui consiste à effectuer tout l'éclairage dans un espace de coordonnées différent : un espace de coordonnées dans lequel les vecteurs de la carte des normales pointent toujours vers la direction z positive ; tous les autres vecteurs d'éclairage sont alors transformés par rapport à cette direction z positive. De cette façon, nous pouvons toujours utiliser la même map de normales, quelle que soit l'orientation. Cet espace de coordonnées est appelé **espace tangent** (tangent space).

## L'espace tangent (tangent space)
Les vecteurs normaux d'une map de normales sont exprimés dans l'espace tangent, où les normales pointent toujours grosso modo dans la direction z positive. **L'espace tangent est un espace local à la surface d'un triangle** : les normales sont relatives au cadre de référence local des triangles individuels. Il s'agit de l'espace local des vecteurs de la map des normales ; ils sont tous définis comme pointant dans la direction z positive, quelle que soit la direction finale de la transformation. En utilisant une matrice spécifique, nous pouvons alors transformer les vecteurs normaux de cet espace tangent local en coordonnées du monde ou de la vue, en les orientant dans la direction de la surface cartographiée finale.

Supposons que nous ayons la surface incorrecte de la section précédente qui regarde dans la direction positive y. La carte des normales est définie dans l'espace tangent. La carte des normales étant définie dans l'espace tangent, une façon de résoudre le problème consiste à calculer une matrice pour transformer les normales de l'espace tangent en un espace différent de façon à ce qu'elles soient alignées sur la direction normale de la surface : les vecteurs normaux sont alors tous orientés grosso modo dans la direction y positive. L'avantage de l'espace tangent est que nous pouvons calculer cette matrice pour n'importe quel type de surface afin d'aligner correctement la direction z de l'espace tangent sur la direction normale de la surface.

Une telle matrice est appelée matrice **TBN**, où les lettres représentent les vecteurs Tangente, Bitangente et Normale. Ce sont les vecteurs dont nous avons besoin pour construire cette matrice. Pour construire une telle matrice de changement de base, qui transforme un vecteur de l'espace tangent en un espace de coordonnées différent, nous avons besoin de trois vecteurs perpendiculaires alignés le long de la surface d'une carte normale : un vecteur vers le haut, un vecteur vers la droite et un vecteur vers l'avant, comme nous l'avons fait dans le chapitre sur les cameras.

Nous connaissons déjà le vecteur haut, qui est le vecteur normal de la surface. Le vecteur droit et le vecteur avant sont respectivement le vecteur tangent et le vecteur bitangent. L'image suivante d'une surface montre les trois vecteurs sur une surface :
![[03_normal_mapping-20230902-normal9.png]]

Le calcul des vecteurs tangents et bitangents n'est pas aussi simple que celui du vecteur normal. Nous pouvons voir sur l'image que la direction des vecteurs tangent et bitangent de la carte des normales s'aligne sur la direction dans laquelle nous définissons les coordonnées de texture d'une surface. Nous utiliserons ce fait pour calculer les vecteurs tangents et bitangents pour chaque surface. Pour les récupérer, il faut faire un peu de mathématiques ; regardez l'image suivante :
![[03_normal_mapping-20230902-normal10.png]]
L'image montre que les différences de coordonnées de texture d'une arête $E2$ d'un triangle (notées $ΔU2$ et $ΔV2$) sont exprimées dans la même direction que le vecteur tangent $T$ et le vecteur bitangent $B$. Pour cette raison, nous pouvons écrire les deux arêtes affichées $E1$ et $E2$ du triangle comme une combinaison linéaire du vecteur tangent $T$ et du vecteur bitangent $B$ :
$$
E_1 = \Delta U_1T + \Delta V_1B
$$
$$
E_2 = \Delta U_2T + \Delta V_2B
$$
Que l'on peut aussi écrire:
$$
(E_{1x},E_{1y},E_{1z})=\Delta U_1(T_x,T_y,T_z)+\Delta V_1(B_x,B_y,B_z)
$$
$$
(E_{2x},E_{2y},E_{2z})=\Delta U_2(T_x,T_y,T_z)+\Delta V_2(B_x,B_y,B_z)
$$
Nous pouvons calculer $E$ comme le vecteur de différence entre les positions de deux triangles, et $ΔU$ et $ΔV$ comme leurs différences de coordonnées de texture. Nous nous retrouvons alors avec deux inconnues (tangente $T$ et bitangente $B$) et deux équations. Vous vous souvenez peut-être de vos cours d'algèbre, qui nous permettent de résoudre $T$ et $B$.

La dernière équation nous permet de l'écrire sous une forme différente : celle de la multiplication matricielle :
$$
\begin{bmatrix}
E_{1x} & E_{1y} & E_{1z} \\
E_{2x} & E_{2y} & E_{2z}
\end{bmatrix}
=
\begin{bmatrix}
\Delta U_1 & \Delta V_1  \\
\Delta U_2 & \Delta V_2
\end{bmatrix}
\begin{bmatrix}
T_x & T_y & T_z  \\
B_x & B_y & B_z
\end{bmatrix}
$$
Essayez de visualiser les multiplications de la matrice dans votre tête et confirmez qu'il s'agit bien de la même équation. La réécriture des équations sous forme de matrice présente l'avantage de rendre la résolution de $T$ et $B$ plus facile à comprendre. Si nous multiplions les deux côtés des équations par l'inverse de la matrice $ΔUΔV$, nous obtenons :

$$
\begin{bmatrix}
\Delta U_1 & \Delta V_1  \\
\Delta U_2 & \Delta V_2
\end{bmatrix}^{-1}
\begin{bmatrix}
E_{1x} & E_{1y} & E_{1z} \\
E_{2x} & E_{2y} & E_{2z}
\end{bmatrix}
=

\begin{bmatrix}
T_x & T_y & T_z  \\
B_x & B_y & B_z
\end{bmatrix}
$$
Cela nous permet de résoudre $T$ et $B$. Nous devons pour cela calculer l'inverse de la matrice des coordonnées de texture delta. Je n'entrerai pas dans les détails mathématiques du calcul de l'inverse d'une matrice, mais cela se traduit grosso modo par 1 sur le déterminant de la matrice, multiplié par sa matrice adjacente (?) :
$$
\begin{bmatrix}
T_x & T_y & T_z  \\
B_x & B_y & B_z
\end{bmatrix}

=
{
1
\over
\Delta U_1 \Delta V_2
-
\Delta U_2 \Delta V_1
}
\begin{bmatrix}
\Delta V_2 & -\Delta V_1 \\
-\Delta U_2 & \Delta U_1
\end{bmatrix}
\begin{bmatrix}
E_{1x} & E_{1y} & E_{1z} \\
E_{2x} & E_{2y} & E_{2z}
\end{bmatrix}
$$

Cette dernière équation nous donne une formule pour calculer le vecteur tangent $T$ et le vecteur bitangent $B$ à partir des deux arêtes d'un triangle et de ses coordonnées de texture.

Ne vous inquiétez pas si vous ne comprenez pas entièrement les mathématiques sous-jacentes. Tant que vous comprenez que nous pouvons calculer les tangentes et les bitangentes à partir des sommets d'un triangle et de ses coordonnées de texture (puisque les coordonnées de texture se trouvent dans le même espace que les vecteurs tangents), vous avez fait la moitié du chemin.

### Calcul manuel des tangentes et bitangentes

Dans la démonstration précédente, nous avions un simple plan normal orienté vers la direction z positive. Cette fois-ci, nous voulons implémenter le mapping de normales en utilisant l'espace tangent afin de pouvoir orienter ce plan comme nous le souhaitons et que le mapping de normales fonctionne toujours. En utilisant les mathématiques discutées précédemment, nous allons calculer manuellement les vecteurs tangents et bitangents de cette surface.

Supposons que le plan soit construit à partir des vecteurs suivants (avec 1, 2, 3 et 1, 3, 4 comme ses deux triangles) :
```cpp
// positions
glm::vec3 pos1(-1.0,  1.0, 0.0);
glm::vec3 pos2(-1.0, -1.0, 0.0);
glm::vec3 pos3( 1.0, -1.0, 0.0);
glm::vec3 pos4( 1.0,  1.0, 0.0);
// texture coordinates
glm::vec2 uv1(0.0, 1.0);
glm::vec2 uv2(0.0, 0.0);
glm::vec2 uv3(1.0, 0.0);
glm::vec2 uv4(1.0, 1.0);
// normal vector
glm::vec3 nm(0.0, 0.0, 1.0);  
```
Nous commençons par calculer les arêtes du premier triangle et les coordonnées delta UV :
```cpp
glm::vec3 edge1 = pos2 - pos1;
glm::vec3 edge2 = pos3 - pos1;
glm::vec2 deltaUV1 = uv2 - uv1;
glm::vec2 deltaUV2 = uv3 - uv1;  
```
Avec les données nécessaires au calcul des tangentes et des bitangentes, nous pouvons commencer à suivre l'équation de la section précédente :
```cpp

float f = 1.0f / (deltaUV1.x * deltaUV2.y - deltaUV2.x * deltaUV1.y);

tangent1.x = f * (deltaUV2.y * edge1.x - deltaUV1.y * edge2.x);
tangent1.y = f * (deltaUV2.y * edge1.y - deltaUV1.y * edge2.y);
tangent1.z = f * (deltaUV2.y * edge1.z - deltaUV1.y * edge2.z);

bitangent1.x = f * (-deltaUV2.x * edge1.x + deltaUV1.x * edge2.x);
bitangent1.y = f * (-deltaUV2.x * edge1.y + deltaUV1.x * edge2.y);
bitangent1.z = f * (-deltaUV2.x * edge1.z + deltaUV1.x * edge2.z);
  
[...] // similar procedure for calculating tangent/bitangent for plane's second triangle
```
Ici, nous pré-calculons d'abord la partie fractionnaire de l'équation sous la forme `f`, puis, pour chaque composante vectorielle, nous effectuons la multiplication matricielle correspondante multipliée par `f`. Si vous comparez ce code avec l'équation finale, vous pouvez voir qu'il s'agit d'une traduction directe. Comme un triangle est toujours une forme plate, nous n'avons besoin de calculer qu'une seule paire tangente/bitangente par triangle car elles seront les mêmes pour chacun des sommets du triangle.

Le vecteur tangent et bitangent résultant doit avoir une valeur de (1,0,0) et (0,1,0) respectivement qui, avec la normale (0,0,1), forme une matrice TBN orthogonale. Visualisés sur le plan, les vecteurs TBN se présentent comme suit :

![[03_normal_mapping-20230902-normal11.png]]
Avec les vecteurs tangents et bitangents définis pour chaque sommet, nous pouvons commencer à mettre en œuvre un mapping de normales approprié.

### Mapping des normales dans l'espace tangent

Pour que le mapping des normales fonctionne, nous devons d'abord créer une matrice TBN dans les shaders. Pour ce faire, nous transmettons les vecteurs de tangente et de bitangente calculés précédemment au shader de vertex en tant qu'attributs de vertex :

```cpp
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aNormal;
layout (location = 2) in vec2 aTexCoords;
layout (location = 3) in vec3 aTangent;
layout (location = 4) in vec3 aBitangent;  
```

Ensuite, dans la fonction principale du vertex shader, nous créons la matrice TBN :
```cpp
void main()
{
   [...]
   vec3 T = normalize(vec3(model * vec4(aTangent,   0.0)));
   vec3 B = normalize(vec3(model * vec4(aBitangent, 0.0)));
   vec3 N = normalize(vec3(model * vec4(aNormal,    0.0)));
   mat3 TBN = mat3(T, B, N);
}
```
Ici, nous transformons d'abord tous les vecteurs TBN dans le système de coordonnées dans lequel nous souhaitons travailler, qui dans ce cas est l'espace monde, car nous les multiplions avec la matrice du modèle. Ensuite, nous créons la matrice TBN réelle en fournissant directement au constructeur de `mat3` les vecteurs de colonnes appropriés. Notez que si nous voulons être vraiment précis, nous multiplierons les vecteurs TBN avec la matrice normale, car nous ne nous soucions que de l'orientation des vecteurs.

> Techniquement, la variable bitangente n'est pas nécessaire dans le vertex shader. Les trois vecteurs TBN sont perpendiculaires l'un à l'autre, nous pouvons donc calculer la bitangente nous-mêmes dans le vertex shader en prenant le produit vectoriel des vecteurs $T$ et $N$ : `vec3 B = cross(N, T) ;`

Maintenant que nous disposons d'une matrice TBN, comment allons-nous l'utiliser ? Il y a deux façons d'utiliser une matrice TBN pour le mapping de normales, et nous allons les présenter toutes les deux :

1. Nous prenons la matrice TBN qui transforme tout vecteur de l'espace tangent à l'espace monde, nous la donnons au shader de fragments et nous transformons la normale échantillonnée de l'espace tangent à l'espace monde en utilisant la matrice TBN ; la normale se trouve alors dans le même espace que les autres variables d'éclairage.
2. Nous prenons l'inverse de la matrice TBN qui transforme tout vecteur de l'espace monde à l'espace tangent, et nous utilisons cette matrice pour transformer non pas la normale, mais les autres variables d'éclairage pertinentes dans l'espace tangent ; la normale se trouve alors à nouveau dans le même espace que les autres variables d'éclairage.

Examinons le premier cas. Le vecteur normal que nous échantillonnons à partir de la map des normales est exprimé dans l'espace tangent alors que les autres vecteurs d'éclairage (lumière et direction de la vue) sont exprimés dans l'espace monde. En passant la matrice TBN au fragment shader, nous pouvons multiplier la normale échantillonnée dans l'espace tangent avec cette matrice TBN pour transformer le vecteur normal dans le même espace de référence que les autres vecteurs d'éclairage. De cette manière, tous les calculs d'éclairage (en particulier le produit scalaire) ont un sens.

Il est facile d'envoyer la matrice TBN au fragment shader :
```cpp
out VS_OUT {
    vec3 FragPos;
    vec2 TexCoords;
    mat3 TBN;
} vs_out;  
  
void main()
{
    [...]
    vs_out.TBN = mat3(T, B, N);
}
```
Dans le fragment shader, nous prenons également un `mat3` comme variable d'entrée :
```cpp
in VS_OUT {
    vec3 FragPos;
    vec2 TexCoords;
    mat3 TBN;
} fs_in;  
```
Avec cette matrice TBN, nous pouvons maintenant mettre à jour le code de mapping normale pour inclure la transformation de l'espace tangent à l'espace monde :
```cpp
normal = texture(normalMap, fs_in.TexCoords).rgb;
normal = normal * 2.0 - 1.0;   
normal = normalize(fs_in.TBN * normal); 
```
Comme la normale résultante est maintenant dans l'espace monde, il n'est pas nécessaire de modifier le code du fragment shader, car le code d'éclairage suppose que le vecteur normal est dans l'espace monde.

Examinons également le second cas, où nous prenons l'inverse de la matrice TBN pour transformer tous les vecteurs pertinents de l'espace mondial en l'espace dans lequel se trouvent les vecteurs normaux échantillonnés : l'espace tangent. La construction de la matrice TBN reste la même, mais nous inversons d'abord la matrice avant de l'envoyer au fragment shader :
```cpp
vs_out.TBN = transpose(mat3(T, B, N));   
```
Notez que nous utilisons ici la fonction transposée au lieu de la fonction inverse. **Une grande propriété des matrices orthogonales (chaque axe est un vecteur unitaire perpendiculaire) est que la transposée d'une matrice orthogonale est égale à son inverse.** C'est une excellente propriété car l'inverse est coûteux, ce qui n'est pas le cas de la transposée.

Dans le fragment shader, nous ne transformons pas le vecteur normal, mais nous transformons les autres vecteurs pertinents dans l'espace tangent, à savoir les vecteurs `lightDir` et `viewDir`. Ainsi, chaque vecteur se trouve dans le même espace de coordonnées : l'espace tangent.
```cpp
void main()
{           
    vec3 normal = texture(normalMap, fs_in.TexCoords).rgb;
    normal = normalize(normal * 2.0 - 1.0);   
   
    vec3 lightDir = fs_in.TBN * normalize(lightPos - fs_in.FragPos);
    vec3 viewDir  = fs_in.TBN * normalize(viewPos - fs_in.FragPos);    
    [...]
}  
```
La seconde approche semble demander plus de travail et nécessite également des multiplications de matrices dans le shader de fragment, alors pourquoi s'embêter avec la seconde approche ?

Eh bien, la transformation des vecteurs de l'espace monde à l'espace tangent présente un avantage supplémentaire : nous pouvons transformer tous les vecteurs d'éclairage pertinents dans l'espace tangent dans le shader de sommets plutôt que dans le shader de fragments. Cela fonctionne parce que `lightPos` et `viewPos` ne sont pas mis à jour à chaque fragment, et pour `fs_in.FragPos` nous pouvons calculer sa position dans l'espace tangent dans le vertex shader et laisser l'interpolation des fragments faire son travail. Il n'est effectivement pas nécessaire de transformer un vecteur en espace tangent dans le shader de fragments, alors que c'est nécessaire avec la première approche car les vecteurs normaux échantillonnés sont spécifiques à chaque exécution du shader de fragments.

Ainsi, au lieu d'envoyer l'inverse de la matrice TBN au fragment shader, nous envoyons la position de la lumière dans l'espace tangent, la position de la vue et la position du sommet au fragment shader. Cela nous évite d'avoir à effectuer des multiplications de matrices dans le shader de fragment. Il s'agit d'une optimisation intéressante, car le shader de sommets s'exécute beaucoup moins souvent que le shader de fragments. C'est également la raison pour laquelle cette approche est souvent privilégiée.

```cpp
out VS_OUT {
    vec3 FragPos;
    vec2 TexCoords;
    vec3 TangentLightPos;
    vec3 TangentViewPos;
    vec3 TangentFragPos;
} vs_out;

uniform vec3 lightPos;
uniform vec3 viewPos;
 
[...]
  
void main()
{    
    [...]
    mat3 TBN = transpose(mat3(T, B, N));
    vs_out.TangentLightPos = TBN * lightPos;
    vs_out.TangentViewPos  = TBN * viewPos;
    vs_out.TangentFragPos  = TBN * vec3(model * vec4(aPos, 1.0));
}  
```
Dans le shader de fragment, nous utilisons ensuite ces nouvelles variables d'entrée pour calculer l'éclairage dans l'espace tangent. Comme le vecteur normal est déjà dans l'espace tangent, l'éclairage est logique.

Avec le mapping de normales appliqué dans l'espace tangent, nous devrions obtenir des résultats similaires à ceux que nous avons eus au début de ce chapitre. Cette fois, cependant, nous pouvons orienter notre plan comme nous le souhaitons et l'éclairage sera toujours correct :
```cpp
glm::mat4 model = glm::mat4(1.0f);
model = glm::rotate(model, (float)glfwGetTime() * -10.0f, glm::normalize(glm::vec3(1.0, 0.0, 1.0)));
shader.setMat4("model", model);
RenderQuad();
```
Ce qui ressemble en effet à un mapping de normales correct :
![[03_normal_mapping-20230902-normal13.png]]
Vous pouvez trouver le code source [ici](https://learnopengl.com/code_viewer_gh.php?code=src/5.advanced_lighting/4.normal_mapping/normal_mapping.cpp).

## Objets complexes
Nous avons démontré comment nous pouvons utiliser le normal mapping, ainsi que les transformations de l'espace tangent, en calculant manuellement les vecteurs tangents et bitangents. Heureusement pour nous, le calcul manuel de ces vecteurs tangents et bitangents n'est pas quelque chose que nous faisons très souvent. La plupart du temps, vous l'implémentez une fois dans un chargeur de modèle personnalisé, ou dans notre cas, vous utilisez un chargeur de modèle utilisant Assimp.

Assimp dispose d'un bit de configuration très utile que nous pouvons définir lors du chargement d'un modèle, appelé `aiProcess_CalcTangentSpace`. Lorsque le bit `aiProcess_CalcTangentSpace` est fourni à la fonction `ReadFile` d'Assimp, Assimp calcule des vecteurs tangents et bitangents lisses pour chacun des sommets chargés, de la même manière que nous l'avons fait dans ce chapitre.

```cpp
const aiScene *scene = importer.ReadFile(
    path, aiProcess_Triangulate | aiProcess_FlipUVs | aiProcess_CalcTangentSpace
);  
```
Dans Assimp, nous pouvons ensuite récupérer les tangentes calculées via :
```cpp
vector.x = mesh->mTangents[i].x;
vector.y = mesh->mTangents[i].y;
vector.z = mesh->mTangents[i].z;
vertex.Tangent = vector;  
```
Vous devrez alors mettre à jour le chargeur de modèle pour qu'il puisse également charger les maps de normales d'un modèle texturé. Le format d'objet wavefront (.obj) exporte les maps de normales légèrement différemment des conventions d'Assimp car `aiTextureType_NORMAL` ne charge pas les maps de normales, alors que `aiTextureType_HEIGHT` le fait :

```cpp
vector<Texture> normalMaps = loadMaterialTextures(material, aiTextureType_HEIGHT, "texture_normal");  
```
Bien sûr, cela est différent pour chaque type de modèle chargé et de format de fichier.

L'exécution de l'application sur un modèle avec des maps de spéculaires et de normales, en utilisant un chargeur de modèle mis à jour, donne le résultat suivant :
![[03_normal_mapping-20230903-normalmap1.png]]
Comme vous pouvez le constater, le mapping de normales permet d'augmenter considérablement le niveau de détail d'un objet sans trop de frais supplémentaires.

L'utilisation du mapping de normales est également un excellent moyen d'améliorer les performances. Avant le mapping de normales, il fallait utiliser un grand nombre de sommets pour obtenir un niveau de détail élevé sur un mesh. Avec le normal mapping, nous pouvons obtenir le même niveau de détail sur un mesh en utilisant beaucoup moins de sommets. L'image ci-dessous de Paolo Cignoni montre une belle comparaison des deux méthodes :
![[03_normal_mapping-20230903-normalmap2.png]]

Les détails du mesh high poly et du mesh low poly avec la conversion normale sont presque impossibles à distinguer. Le normal mapping n'est donc pas seulement esthétique, c'est aussi un excellent outil pour remplacer les meshes high poly par des meshes low poly sans perdre (trop) de détails.

## Une dernière chose
Il reste une dernière astuce qui permet d'améliorer légèrement la qualité sans trop de frais supplémentaires.

Lorsque les vecteurs tangents sont calculés sur des meshes plus importants qui partagent un nombre considérable de sommets, la moyenne des vecteurs tangents est généralement calculée afin d'obtenir des résultats agréables et lisses. Le problème de cette approche est que les trois vecteurs TBN peuvent se retrouver non perpendiculaires, ce qui signifie que la matrice TBN résultante n'est plus orthogonale. Le normal mapping ne serait que légèrement altéré par une matrice TBN non orthogonale, mais nous pouvons encore l'améliorer.

En utilisant une astuce mathématique appelée **processus de Gram-Schmidt**, nous pouvons réorthogonaliser les vecteurs TBN de sorte que chaque vecteur soit à nouveau perpendiculaire aux autres vecteurs. Dans le vertex shader, nous procédons de la manière suivante :
```cpp
vec3 T = normalize(vec3(model * vec4(aTangent, 0.0)));
vec3 N = normalize(vec3(model * vec4(aNormal, 0.0)));
// re-orthogonalize T with respect to N
T = normalize(T - dot(T, N) * N);
// then retrieve perpendicular vector B with the cross product of T and N
vec3 B = cross(N, T);

mat3 TBN = mat3(T, B, N) 
```

Cela améliore généralement les résultats du normal mapping, même si ce n'est que dans une faible mesure, pour un coût supplémentaire minime. Jetez un coup d'œil à la fin de la vidéo "Normal Mapping Mathematics" dans les ressources supplémentaires pour une explication détaillée du fonctionnement de ce processus.

## Ressources supplémentaires
- [Tutoriel 26 : Normal Mapping](http://ogldev.atspace.co.uk/www/tutorial26/tutorial26.html) : tutoriel sur le normal mapping par ogldev.
- [How Normal Mapping Works](https://www.youtube.com/watch?v=LIOPYmknj5Q) : un tutoriel vidéo intéressant sur le fonctionnement du normal mapping par TheBennyBox.
- [Normal Mapping Mathematics](https://www.youtube.com/watch?v=4FaWLgsctqY) : une vidéo similaire de TheBennyBox sur les mathématiques du normal mapping.
- [Tutoriel 13 : Normal Mapping](http://www.opengl-tutorial.org/intermediate-tutorials/tutorial-13-normal-mapping/) : tutoriel sur le normal mapping par opengl-tutorial.org.

