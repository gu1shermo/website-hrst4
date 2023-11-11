# GLSL avancé
Ce chapitre ne vous montrera pas vraiment de nouvelles fonctionnalités super avancées qui donneront un énorme coup de pouce à la qualité visuelle de votre scène. Ce chapitre aborde plus ou moins des aspects intéressants de GLSL et des astuces qui peuvent vous aider dans vos projets futurs. En gros, quelques bonnes choses à savoir et des fonctionnalités qui peuvent vous faciliter la vie lorsque vous créez des applications OpenGL en combinaison avec GLSL.

Nous discuterons de quelques variables intégrées intéressantes, de nouvelles façons d'organiser l'entrée et la sortie des shaders, et d'un outil très utile appelé **objets tampons uniformes.**

## Les variables intégrées de GLSL (built-in variables)
Les shaders sont très complexes, et si nous avons besoin de données provenant d'une autre source que le shader en cours, nous devrons les faire circuler. Nous avons appris à le faire via les attributs de vertex, les uniformes et les samplers. Il existe cependant quelques variables supplémentaires définies par GLSL et préfixées par `gl_` qui nous donnent un moyen supplémentaire de collecter et/ou d'écrire des données. Nous en avons déjà vu deux dans les chapitres précédents : `gl_Position` qui est le vecteur de sortie du vertex shader, et `gl_FragCoord` du fragment shader.

Nous allons discuter de quelques variables d'entrée et de sortie intégrées intéressantes qui sont intégrées dans GLSL et expliquer comment elles peuvent nous être utiles. Notez que nous ne parlerons pas de toutes les variables intégrées qui existent dans GLSL, donc si vous voulez voir toutes les variables intégrées, vous pouvez consulter le [wiki](https://www.khronos.org/opengl/wiki/Built-in_Variable_(GLSL)) d'OpenGL.

## Variables du vertex shader
Nous avons déjà vu `gl_Position` qui est le vecteur de position de sortie de l'espace-clip du vertex shader. Définir `gl_Position` dans le vertex shader est une exigence stricte si vous voulez rendre quoi que ce soit à l'écran. Rien que nous n'ayons déjà vu auparavant.

#### `gl_PointSize`
L'une des primitives de rendu que nous pouvons choisir est `GL_POINTS`. Dans ce cas, chaque vertex est une primitive et est rendu comme un point. Il est possible de définir la taille des points rendus via la fonction `glPointSize` d'OpenGL, mais nous pouvons également influencer cette valeur dans le shader de vertex.

Une variable de sortie définie par GLSL est appelée `gl_PointSize`. **Il s'agit d'une variable flottante dans laquelle vous pouvez définir la largeur et la hauteur du point en pixels.** En définissant la taille du point dans le shader de sommets, nous obtenons un contrôle par sommet sur les dimensions de ce point.

L'influence sur la taille des points dans le vertex shader est désactivée par défaut, mais si vous souhaitez l'activer, vous devez activer la fonction `GL_PROGRAM_POINT_SIZE` d'OpenGL :
```cpp
glEnable(GL_PROGRAM_POINT_SIZE);  
```
Un exemple simple d'influence sur la taille des points consiste à définir une taille de point égale à la valeur z de la position de l'espace-clip, qui est égale à la distance du sommet par rapport à l'observateur. La taille du point devrait alors augmenter au fur et à mesure que l'on s'éloigne des sommets de l'observateur.

```cpp
void main()
{
    gl_Position = projection * view * model * vec4(aPos, 1.0);    
    gl_PointSize = gl_Position.z;    
}  
```
Le résultat est que les points que nous avons dessinés sont rendus plus grands au fur et à mesure que nous nous en éloignons :
![[advanced_glsl_pointsize.png]]
Vous pouvez imaginer que la variation de la taille des points par vertex est intéressante pour des techniques telles que la génération de particules.

#### `gl_VertexID`
Les variables `gl_Position` et `gl_PointSize` sont des variables de sortie puisque leur valeur est lue par le vertex shader ; nous pouvons influencer le résultat en y écrivant. Le vertex shader nous fournit également une variable d'entrée intéressante, que nous ne pouvons que lire, appelée `gl_VertexID`.

La variable entière `gl_VertexID` contient l'identifiant actuel du sommet que nous sommes en train de dessiner. Lors d'un rendu indexé (avec `glDrawElements`), cette variable contient l'index actuel du sommet que nous dessinons. Lors d'un dessin sans index (via `glDrawArrays`), cette variable contient le numéro du sommet en cours de traitement depuis le début de l'appel au rendu.

## Variables du fragment shader
Dans le fragment shader, nous avons également accès à quelques variables intéressantes. GLSL nous donne deux variables d'entrée intéressantes appelées `gl_FragCoord` et `gl_FrontFacing`.

### `gl_FragCoord`
Nous avons déjà vu le vecteur `gl_FragCoord` à plusieurs reprises au cours de la discussion sur les tests de profondeur, car la composante z du vecteur `gl_FragCoord` est égale à la valeur de profondeur de ce fragment particulier. Cependant, nous pouvons également utiliser les composantes x et y de ce vecteur pour obtenir des effets intéressants.

Les composantes x et y du vecteur `gl_FragCoord` sont les coordonnées du fragment dans l'espace de la fenêtre ou de l'écran, à partir de la partie inférieure gauche de la fenêtre. Nous avons spécifié une fenêtre de rendu de 800x600 avec `glViewport`, de sorte que les coordonnées de l'espace-écran du fragment auront des valeurs x comprises entre 0 et 800, et des valeurs y comprises entre 0 et 600.

En utilisant le shader de fragment, nous pouvons calculer une valeur de couleur différente en fonction des coordonnées d'écran du fragment. La variable `gl_FragCoord` est couramment utilisée pour comparer les résultats visuels de différents calculs de fragments, comme on le voit généralement dans les démonstrations techniques. Nous pourrions par exemple diviser l'écran en deux en rendant une sortie sur le côté gauche de la fenêtre et une autre sur le côté droit de la fenêtre. Un exemple de shader de fragment qui produit une couleur différente en fonction des coordonnées du fragment à l'écran est donné ci-dessous :
```cpp
void main()
{             
    if(gl_FragCoord.x < 400)
        FragColor = vec4(1.0, 0.0, 0.0, 1.0);
    else
        FragColor = vec4(0.0, 1.0, 0.0, 1.0);        
}  
```
Comme la largeur de la fenêtre est égale à 800, chaque fois que la coordonnée x d'un pixel est inférieure à 400, il doit se trouver à gauche de la fenêtre et nous donnerons à ce fragment une couleur différente.

![[advanced_glsl_fragcoord.png]]


Nous pouvons maintenant calculer deux résultats de fragment shader complètement différents et les afficher chacun d'un côté différent de la fenêtre. C'est idéal pour tester différentes techniques d'éclairage, par exemple.

### ``gl_FrontFacing`
Une autre variable d'entrée intéressante dans le fragment shader est la variable `gl_FrontFacing`. **Dans le chapitre sur l'élimination des faces, nous avons mentionné qu'OpenGL est capable de déterminer si une face est une face avant ou arrière en raison de l'ordre d'enroulement des sommets**. La variable `gl_FrontFacing` nous indique si le fragment courant fait partie d'une face avant ou arrière. Nous pourrions, par exemple, décider de produire des couleurs différentes pour toutes les faces arrière.

**La variable `gl_FrontFacing` est un bool qui vaut true si le fragment fait partie d'une face avant et false dans le cas contraire.** Nous pouvons créer un cube de cette manière avec une texture différente à l'intérieur et à l'extérieur :
```cpp
#version 330 core
out vec4 FragColor;
  
in vec2 TexCoords;

uniform sampler2D frontTexture;
uniform sampler2D backTexture;

void main()
{             
    if(gl_FrontFacing)
        FragColor = texture(frontTexture, TexCoords);
    else
        FragColor = texture(backTexture, TexCoords);
}  
```
Si nous jetons un coup d'œil à l'intérieur du conteneur, nous constatons qu'une texture différente est utilisée.
![[advanced_glsl_frontfacing.png]]
Notez que si vous avez activé l'élimination des faces, vous ne pourrez pas voir les faces à l'intérieur du conteneur et l'utilisation de `gl_FrontFacing` serait alors inutile.
### `gl_FragDepth`
La variable d'entrée `gl_FragCoord` est une variable d'entrée qui nous permet de lire les coordonnées de l'espace-écran et d'obtenir la valeur de la profondeur du fragment actuel, mais il s'agit d'une variable en lecture seule. Nous ne pouvons pas influencer les coordonnées de l'espace-écran du fragment, **mais il est possible de définir la valeur de la profondeur du fragment**. GLSL nous donne une variable de sortie appelée `gl_FragDepth` que nous pouvons utiliser pour définir manuellement la valeur de profondeur du fragment dans le shader.

Pour définir la valeur de la profondeur dans le shader, nous écrivons n'importe quelle valeur entre 0,0 et 1,0 dans la variable de sortie :
```cpp
gl_FragDepth = 0.0; // this fragment now has a depth value of 0.0
```
Si le shader n'écrit rien dans `gl_FragDepth`, la variable prendra automatiquement sa valeur dans `gl_FragCoord.z`.

**Définir manuellement la valeur de la profondeur présente cependant un inconvénient majeur**. En effet, OpenGL désactive le test de profondeur anticipé (tel que discuté dans le chapitre sur le test de profondeur) dès que nous écrivons dans `gl_FragDepth` dans le fragment shader. Il est désactivé parce qu'OpenGL ne peut pas savoir quelle valeur de profondeur aura le fragment avant de lancer le shader de fragment, puisque le shader de fragment peut en fait changer cette valeur.

En écrivant dans `gl_FragDepth`, vous devez prendre en compte cette pénalité de performance. A partir d'OpenGL 4.2 cependant, nous pouvons encore faire une sorte de médiation entre les deux côtés en redéclarant la variable `gl_FragDepth` au début du fragment shader avec une condition de profondeur :
```cpp
layout (depth_<condition>) out float gl_FragDepth;
```
Cette condition peut prendre les valeurs suivantes :

|Condition|Description|
|-|-|
|any|Valeur par défaut. Le test de profondeur précoce est désactivé.|
|greater|Vous pouvez seulement augmenter la valeur de la profondeur par rapport à `gl_FragCoord.z`.|
|less|Vous ne pouvez que réduire la valeur de la profondeur par rapport à `gl_FragCoord.z`.|
|unchanged|Si vous écrivez dans `gl_FragDepth`, vous écrirez exactement `gl_FragCoord.z`.|
En spécifiant supérieur ou inférieur comme condition de profondeur, OpenGL peut supposer que vous n'écrirez que des valeurs de profondeur plus grandes ou plus petites que la valeur de profondeur du fragment. De cette manière, OpenGL est toujours capable de tester la profondeur lorsque la valeur du tampon de profondeur fait partie de l'autre direction de `gl_FragCoord.z`.

Un exemple où nous augmentons la valeur de profondeur dans le fragment shader, mais où nous voulons encore préserver une partie du test de profondeur précoce est montré dans le fragment shader ci-dessous :
```cpp
#version 420 core // note the GLSL version!
out vec4 FragColor;
layout (depth_greater) out float gl_FragDepth;

void main()
{             
    FragColor = vec4(1.0);
    gl_FragDepth = gl_FragCoord.z + 0.1;
}  
```
Notez que cette fonctionnalité n'est disponible qu'à partir de la version 4.2 d'OpenGL.

## Blocs d'interface
Nous utilisons OpenGL depuis un certain temps maintenant et avons appris quelques trucs sympas, mais aussi quelques désagréments. Par exemple, lorsque nous utilisons plus d'un shader, nous devons continuellement définir des variables uniformes dont la plupart sont exactement les mêmes pour chaque shader.

OpenGL nous offre un outil appelé objets tampons uniformes (uniform buffer objects) qui nous permet de déclarer un ensemble de variables uniformes globales qui restent les mêmes quel que soit le nombre de programmes de shaders. **En utilisant les objets tampons uniformes, nous définissons les uniformes pertinents une seule fois dans la mémoire fixe du GPU**. Il reste cependant nécessaire de définir manuellement les uniformes qui sont uniques pour chaque shader. La création et la configuration d'un objet tampon uniforme nécessitent toutefois un peu de travail.

Parce qu'un objet tampon uniforme est un tampon comme les autres, nous pouvons en créer un via `glGenBuffers`, le lier à la cible tampon `GL_UNIFORM_BUFFER` et stocker toutes les données uniformes pertinentes dans le tampon. Il existe certaines règles quant à la manière dont les données des objets tampons uniformes doivent être stockées, nous y reviendrons plus tard. Tout d'abord, nous allons prendre un simple vertex shader et stocker notre projection et notre matrice de vue dans ce que l'on appelle un bloc uniforme :
```cpp
#version 330 core
layout (location = 0) in vec3 aPos;

layout (std140) uniform Matrices
{
    mat4 projection;
    mat4 view;
};

uniform mat4 model;

void main()
{
    gl_Position = projection * view * model * vec4(aPos, 1.0);
}  
```
Dans la plupart de nos exemples, nous définissons une matrice uniforme de projection et de vue à chaque image pour chaque shader que nous utilisons. C'est un exemple parfait de l'utilité des objets tampons uniformes puisque nous n'avons plus qu'à stocker ces matrices une seule fois.

Ici, nous avons déclaré un bloc uniforme appelé Matrices qui stocke deux matrices 4x4. Les variables d'un bloc uniforme sont directement accessibles sans le nom du bloc comme préfixe. Ensuite, nous stockons ces valeurs de matrices dans un tampon quelque part dans le code OpenGL et chaque shader qui déclare ce bloc uniforme a accès aux matrices.

Vous vous demandez probablement ce que signifie l'énoncé `layout (std140)`. Cela signifie que le bloc uniforme actuellement défini utilise une disposition de mémoire spécifique pour son contenu ; cette déclaration définit la disposition du bloc uniforme.

## Disposition uniforme des blocs
Le contenu d'un bloc uniforme est stocké dans un objet tampon, qui n'est rien d'autre qu'un morceau réservé de la mémoire globale du GPU. Parce que ce morceau de mémoire ne contient aucune information sur le type de données qu'il contient, nous devons dire à OpenGL quelles parties de la mémoire correspondent à quelles variables uniformes dans le shader.

Imaginez le bloc uniforme suivant dans un shader :
```cpp
layout (std140) uniform ExampleBlock
{
    float value;
    vec3  vector;
    mat4  matrix;
    float values[3];
    bool  boolean;
    int   integer;
};  
```
Ce que nous voulons savoir, c'est la taille (en octets) et le décalage (depuis le début du bloc) de chacune de ces variables afin de pouvoir les placer dans le tampon dans leur ordre respectif. La taille de chacun des éléments est clairement indiquée dans OpenGL et correspond directement aux types de données C++ ; les vecteurs et les matrices étant de (grands) tableaux de flottants. **Ce qu'OpenGL n'indique pas clairement, c'est l'espacement entre les variables**. Cela permet au matériel de positionner ou de remplir les variables comme il l'entend. Le matériel est capable de placer un `vec3` à côté d'un `float` par exemple. Tous les matériels ne peuvent pas gérer cela et remplacent le `vec3` par un tableau de 4 flottants avant d'ajouter le flottant. C'est une fonctionnalité intéressante, mais peu pratique pour nous.

Par défaut, GLSL utilise une disposition uniforme de la mémoire appelée disposition partagée (shared layout)- partagée parce qu'une fois que les décalages sont définis par le matériel, ils sont partagés de manière cohérente entre plusieurs programmes. Avec une disposition partagée, GLSL est autorisé à repositionner les variables uniformes à des fins d'optimisation, tant que l'ordre des variables reste intact. Comme nous ne savons pas à quel décalage chaque variable uniforme se trouvera, nous ne savons pas comment remplir précisément notre tampon uniforme. Nous pouvons interroger cette information avec des fonctions comme `glGetUniformIndices`, mais ce n'est pas l'approche que nous allons adopter dans ce chapitre.

Bien qu'une disposition partagée nous permette d'économiser de l'espace, nous devrions interroger l'offset pour chaque variable uniforme, ce qui représente beaucoup de travail. La pratique générale consiste cependant à ne pas utiliser la disposition partagée, mais à utiliser la disposition `std140`. La disposition `std140` indique explicitement la disposition de la mémoire pour chaque type de variable en normalisant leurs décalages respectifs régis par un ensemble de règles. Puisque cette disposition est normalisée, nous pouvons déterminer manuellement les décalages pour chaque variable.

Chaque variable a un alignement de base égal à l'espace qu'une variable prend (y compris le remplissage) dans un bloc uniforme en utilisant les règles de disposition `std140`. Pour chaque variable, nous calculons son décalage aligné : le décalage en octets d'une variable par rapport au début du bloc. Le décalage d'octet aligné d'une variable doit être égal à un multiple de son alignement de base. C'est un peu compliqué, mais nous verrons bientôt des exemples pour clarifier les choses.


Les règles exactes de mise en page peuvent être trouvées dans la spécification de tampon uniforme d'OpenGL [ici](http://www.opengl.org/registry/specs/ARB/uniform_buffer_object.txt), mais nous allons énumérer les règles les plus courantes ci-dessous. Chaque type de variable en GLSL, comme int, float et bool, est défini comme une quantité de quatre octets, chaque entité de 4 octets étant représentée par N.

|Type|Règle de Layout|
|-|-|
|Scalaire (int ou float)|Chaque scalaire a un alignement de base de N.|
|Vector|Soit 2N, soit 4N. Cela signifie qu'un vec3 a un alignement de base de 4N.|
|Tableau de scalaires ou de Vectors|Chaque élément a un alignement de base égal à celui d'un vec4.|
|Matrices|Stocké sous la forme d'un grand tableau de vecteurs de colonnes, où chacun de ces vecteurs a un alignement de base de vec4.|
|Struct|Égale à la taille calculée de ses éléments selon les règles précédentes, mais remplie d'un multiple de la taille d'un vec4.|
Comme la plupart des spécifications d'OpenGL, il est plus facile de comprendre avec un exemple. Nous prenons le bloc uniforme appelé `ExampleBlock` que nous avons présenté plus tôt et nous calculons le décalage aligné pour chacun de ses membres en utilisant la disposition `std140` :
```cpp
layout (std140) uniform ExampleBlock
{
                     // base alignment  // aligned offset
    float value;     // 4               // 0 
    vec3 vector;     // 16              // 16  (offset must be multiple of 16 so 4->16)
    mat4 matrix;     // 16              // 32  (column 0)
                     // 16              // 48  (column 1)
                     // 16              // 64  (column 2)
                     // 16              // 80  (column 3)
    float values[3]; // 16              // 96  (values[0])
                     // 16              // 112 (values[1])
                     // 16              // 128 (values[2])
    bool boolean;    // 4               // 144
    int integer;     // 4               // 148
}; 
```
À titre d'exercice, essayez de calculer vous-même les valeurs de décalage et comparez-les à ce tableau. Avec ces valeurs de décalage calculées, basées sur les règles de la disposition `std140`, nous pouvons remplir le tampon avec des données aux décalages appropriés en utilisant des fonctions comme `glBufferSubData`. Bien qu'elle ne soit pas la plus efficace, la disposition `std140` nous garantit que la disposition de la mémoire reste la même pour chaque programme qui a déclaré ce bloc uniforme.

En ajoutant la déclaration `layout (std140)` dans la définition du bloc uniforme, nous indiquons à OpenGL que ce bloc uniforme utilise la disposition `std140`. Il y a deux autres dispositions à choisir qui nous obligent à interroger chaque décalage avant de remplir les tampons. Nous avons déjà vu la disposition partagée, l'autre disposition restante étant emballée (packed). Lorsque l'on utilise le `layout packed`, il n'y a aucune garantie que le layout reste le même entre les programmes (non partagé) car il permet au compilateur d'optimiser les variables uniformes loin du bloc uniforme qui peut être différent pour chaque shader.

## Utilisation de tampons uniformes
Nous avons défini les blocs uniformes et spécifié leur disposition en mémoire, mais nous n'avons pas encore discuté de la manière de les utiliser.

Tout d'abord, nous devons créer un objet tampon uniforme, ce qui se fait via le familier `glGenBuffers`. Une fois que nous avons un objet tampon, nous le lions à la cible `GL_UNIFORM_BUFFER` et allouons suffisamment de mémoire en appelant `glBufferData`.
```cpp
unsigned int uboExampleBlock;
glGenBuffers(1, &uboExampleBlock);
glBindBuffer(GL_UNIFORM_BUFFER, uboExampleBlock);
glBufferData(GL_UNIFORM_BUFFER, 152, NULL, GL_STATIC_DRAW); // allocate 152 bytes of memory
glBindBuffer(GL_UNIFORM_BUFFER, 0);
```
Maintenant, chaque fois que nous voulons mettre à jour ou insérer des données dans le tampon, nous nous lions à `uboExampleBlock` et utilisons `glBufferSubData` pour mettre à jour sa mémoire. Nous n'avons à mettre à jour ce tampon uniforme qu'une seule fois, et tous les shaders qui utilisent ce tampon utilisent maintenant ses données mises à jour. Mais comment OpenGL sait-il quels tampons uniformes correspondent à quels blocs uniformes ?

Dans le contexte OpenGL, il y a un certain nombre de points de liaison définis auxquels nous pouvons lier un tampon uniforme. Une fois que nous avons créé un tampon uniforme, nous le lions à l'un de ces points de liaison et nous lions également le bloc uniforme dans le shader au même point de liaison, les liant ainsi ensemble. Le diagramme suivant illustre ce processus :
![[advanced_glsl_binding_points.png]]

Comme vous pouvez le voir, nous pouvons lier plusieurs tampons uniformes à différents points de liaison. Comme le shader A et le shader B ont tous deux un bloc uniforme lié au même point de liaison 0, leurs blocs uniformes partagent les mêmes données uniformes trouvées dans `uboMatrices` ; la condition étant que les deux shader définissent le même bloc uniforme Matrices.

Pour définir un bloc uniforme de shader à un point de liaison spécifique, nous appelons `glUniformBlockBinding` qui prend un objet de programme, un index de bloc uniforme et le point de liaison à lier. L'indice du bloc uniforme est un indice d'emplacement du bloc uniforme défini dans le shader. Il peut être récupéré via un appel à `glGetUniformBlockIndex` qui accepte un objet programme et le nom du bloc uniforme. Nous pouvons placer le bloc uniforme `Lights` du diagramme au point de liaison 2 de la manière suivante :
```cpp
unsigned int lights_index = glGetUniformBlockIndex(shaderA.ID, "Lights");   
glUniformBlockBinding(shaderA.ID, lights_index, 2);
```
Notez que nous devons répéter ce processus pour chaque shader.

>À partir de la version 4.2 d'OpenGL, il est également possible de stocker le point de liaison d'un bloc uniforme explicitement dans le shader en ajoutant un autre spécificateur de disposition, ce qui nous évite d'appeler `glGetUniformBlockIndex` et `glUniformBlockBinding`. Le code suivant définit explicitement le point de liaison du bloc uniforme Lights :
```cpp
layout(std140, binding = 2) uniform Lights { ... };
```
Nous devons ensuite lier l'objet tampon uniforme au même point de liaison, ce qui peut être réalisé avec `glBindBufferBase` ou `glBindBufferRange`.
```cpp
glBindBufferBase(GL_UNIFORM_BUFFER, 2, uboExampleBlock); 
// or
glBindBufferRange(GL_UNIFORM_BUFFER, 2, uboExampleBlock, 0, 152);
```
La fonction `glBindbufferBase` attend une cible, un index de point de liaison et un objet tampon uniforme. Cette fonction lie `uboExampleBlock` au point de liaison 2 ; à partir de ce point, les deux côtés du point de liaison sont liés. Vous pouvez également utiliser `glBindBufferRange` qui attend un paramètre supplémentaire de décalage et de taille - de cette façon, vous pouvez lier uniquement une plage spécifique du tampon uniforme à un point de liaison. En utilisant `glBindBufferRange`, vous pouvez avoir plusieurs blocs uniformes différents liés à un seul objet tampon uniforme.

Maintenant que tout est en place, nous pouvons commencer à ajouter des données au tampon uniforme. Nous pouvons ajouter toutes les données sous la forme d'un tableau d'octets, ou mettre à jour des parties du tampon lorsque nous le souhaitons en utilisant `glBufferSubData`. Pour mettre à jour la variable uniforme booléenne, nous pouvons mettre à jour l'objet tampon uniforme comme suit :
```cpp
glBindBuffer(GL_UNIFORM_BUFFER, uboExampleBlock);
int b = true; // bools in GLSL are represented as 4 bytes, so we store it in an integer
glBufferSubData(GL_UNIFORM_BUFFER, 144, 4, &b); 
glBindBuffer(GL_UNIFORM_BUFFER, 0);
```
La même procédure s'applique à toutes les autres variables uniformes à l'intérieur du bloc uniforme, mais avec des arguments de portée différents.

## Un exemple simple
Prenons donc un exemple concret d'objets tampons uniformes. Si nous regardons tous les exemples de code précédents, nous avons continuellement utilisé trois matrices : la matrice de projection, la matrice de vue et la matrice de modèle. De toutes ces matrices, seule la matrice de modèle change fréquemment. Si nous avons plusieurs shaders qui utilisent ce même ensemble de matrices, nous ferions mieux d'utiliser des objets tampons uniformes.

Nous allons stocker les matrices de projection et de vue dans un bloc uniforme appelé `Matrices`. Nous n'allons pas y stocker la matrice du modèle, car celle-ci a tendance à changer fréquemment d'un shader à l'autre, de sorte que nous n'aurions pas vraiment intérêt à utiliser des objets tampons uniformes.

```cpp
#version 330 core
layout (location = 0) in vec3 aPos;

layout (std140) uniform Matrices
{
    mat4 projection;
    mat4 view;
};
uniform mat4 model;

void main()
{
    gl_Position = projection * view * model * vec4(aPos, 1.0);
}  
```
Il ne se passe pas grand-chose ici, si ce n'est que nous utilisons maintenant un bloc uniforme avec une disposition `std140`. Ce que nous allons faire dans notre exemple d'application est d'afficher 4 cubes où chaque cube est affiché avec un programme de shader différent. Chacun des 4 programmes de shaders utilise le même vertex shader, mais possède un fragment shader unique qui ne produit qu'une seule couleur différente pour chaque shader.

Tout d'abord, nous réglons le bloc uniforme des shaders de sommets sur le point de liaison 0. Notez que nous devons effectuer cette opération pour chaque shader :
```cpp
unsigned int uniformBlockIndexRed    = glGetUniformBlockIndex(shaderRed.ID, "Matrices");
unsigned int uniformBlockIndexGreen  = glGetUniformBlockIndex(shaderGreen.ID, "Matrices");
unsigned int uniformBlockIndexBlue   = glGetUniformBlockIndex(shaderBlue.ID, "Matrices");
unsigned int uniformBlockIndexYellow = glGetUniformBlockIndex(shaderYellow.ID, "Matrices");  
  
glUniformBlockBinding(shaderRed.ID,    uniformBlockIndexRed, 0);
glUniformBlockBinding(shaderGreen.ID,  uniformBlockIndexGreen, 0);
glUniformBlockBinding(shaderBlue.ID,   uniformBlockIndexBlue, 0);
glUniformBlockBinding(shaderYellow.ID, uniformBlockIndexYellow, 0);
```
Ensuite, nous créons l'objet tampon uniforme proprement dit et le lions au point de liaison 0 :
```cpp
unsigned int uboMatrices
glGenBuffers(1, &uboMatrices);
  
glBindBuffer(GL_UNIFORM_BUFFER, uboMatrices);
glBufferData(GL_UNIFORM_BUFFER, 2 * sizeof(glm::mat4), NULL, GL_STATIC_DRAW);
glBindBuffer(GL_UNIFORM_BUFFER, 0);
  
glBindBufferRange(GL_UNIFORM_BUFFER, 0, uboMatrices, 0, 2 * sizeof(glm::mat4));
```
Tout d'abord, nous allouons suffisamment de mémoire pour notre tampon qui est égal à 2 fois la taille de `glm::mat4`. La taille des matrices GLM correspond directement à `mat4` en GLSL. Ensuite, nous lions une plage spécifique du tampon, dans ce cas le tampon entier, au point de liaison 0.

Il ne reste plus qu'à remplir le tampon. Si nous maintenons constante la valeur du champ de vision de la matrice de projection (donc plus de zoom de la caméra), nous ne devons la mettre à jour qu'une seule fois dans notre application - ce qui signifie que nous ne devons l'insérer dans le tampon qu'une seule fois également. Comme nous avons déjà alloué suffisamment de mémoire à l'objet tampon, nous pouvons utiliser `glBufferSubData` pour stocker la matrice de projection avant d'entrer dans la boucle de rendu :
```cpp
glm::mat4 projection = glm::perspective(glm::radians(45.0f), (float)width/(float)height, 0.1f, 100.0f);
glBindBuffer(GL_UNIFORM_BUFFER, uboMatrices);
glBufferSubData(GL_UNIFORM_BUFFER, 0, sizeof(glm::mat4), glm::value_ptr(projection));
glBindBuffer(GL_UNIFORM_BUFFER, 0);
```
Ici, nous stockons la première moitié du tampon uniforme avec la matrice de projection. Ensuite, avant de rendre les objets à chaque image, nous mettons à jour la seconde moitié de la mémoire tampon avec la matrice de visualisation :
```cpp
glm::mat4 view = camera.GetViewMatrix();	       
glBindBuffer(GL_UNIFORM_BUFFER, uboMatrices);
glBufferSubData(GL_UNIFORM_BUFFER, sizeof(glm::mat4), sizeof(glm::mat4), glm::value_ptr(view));
glBindBuffer(GL_UNIFORM_BUFFER, 0);  
```
Et c'est tout pour les objets tampons uniformes. Chaque vertex shader qui contient un bloc uniforme Matrices contiendra désormais les données stockées dans `uboMatrices`. Ainsi, si nous dessinons 4 cubes en utilisant 4 shaders différents, leur projection et leur matrice de vue devraient être identiques :
```cpp
glBindVertexArray(cubeVAO);
shaderRed.use();
glm::mat4 model = glm::mat4(1.0f);
model = glm::translate(model, glm::vec3(-0.75f, 0.75f, 0.0f));	// move top-left
shaderRed.setMat4("model", model);
glDrawArrays(GL_TRIANGLES, 0, 36);        
// ... draw Green Cube
// ... draw Blue Cube
// ... draw Yellow Cube	
```
Le seul uniforme que nous devons encore définir est l'uniforme du modèle. L'utilisation d'objets tampons uniformes dans un scénario comme celui-ci nous permet d'éviter un certain nombre d'appels d'uniformes par shader. Le résultat ressemble à ceci :
![[advanced_glsl_uniform_buffer_objects.png]]
Chacun des cubes est déplacé d'un côté de la fenêtre par translation de la matrice du modèle et, grâce aux différents shaders de fragment, leurs couleurs diffèrent selon l'objet. Il s'agit d'un scénario relativement simple d'utilisation des objets tampons uniformes, mais toute grande application de rendu peut avoir des centaines de programmes de shaders actifs, et c'est là que les objets tampons uniformes commencent vraiment à briller.

Vous pouvez trouver le code source complet de l'exemple d'application uniforme [ici](https://learnopengl.com/code_viewer_gh.php?code=src/4.advanced_opengl/8.advanced_glsl_ubo/advanced_glsl_ubo.cpp).

Les objets tampons uniformes présentent plusieurs avantages par rapport aux uniformes individuels. Tout d'abord, il est plus rapide de définir un grand nombre d'uniformes à la fois que de définir plusieurs uniformes un par un. Deuxièmement, si vous voulez changer le même uniforme sur plusieurs shaders, il est beaucoup plus facile de changer un uniforme une fois dans un tampon d'uniformes. Un dernier avantage qui n'est pas immédiatement apparent est que vous pouvez utiliser beaucoup plus d'uniformes dans les shaders en utilisant des objets tampons uniformes. OpenGL a une limite à la quantité de données d'uniformes qu'il peut gérer, qui peut être interrogée avec `GL_MAX_VERTEX_UNIFORM_COMPONENTS`. Lorsque l'on utilise des objets tampons uniformes, cette limite est beaucoup plus élevée. Ainsi, lorsque vous atteignez un nombre maximum d'uniformes (lors d'une animation squelettique par exemple), il y a toujours des objets tampons uniformes.
