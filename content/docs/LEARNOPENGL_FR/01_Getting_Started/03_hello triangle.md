---
tags: [cg, opengl, computer graphics, cpp]
dg-publish: true
---
# Hello 

Dans OpenGL, tout est dans l'espace 3D, mais l'écran ou la fenêtre est un tableau de pixels 2D. **Une grande partie du travail d'OpenGL consiste donc à transformer toutes les coordonnées 3D en pixels 2D qui tiennent sur l'écran**. Le processus de transformation des coordonnées 3D en pixels 2D est géré par le pipeline graphique d'OpenGL. **Le pipeline graphique peut être divisé en deux grandes parties : la première transforme vos coordonnées 3D en coordonnées 2D et la seconde transforme les coordonnées 2D en pixels colorés.** Dans ce chapitre, nous discuterons brièvement du pipeline graphique et de la manière dont nous pouvons l'utiliser à notre avantage pour créer des pixels fantaisistes.  
  
Le pipeline graphique prend en entrée un ensemble de coordonnées 3D et les transforme en pixels 2D colorés sur votre écran. **Le pipeline graphique peut être divisé en plusieurs étapes, chaque étape nécessitant en entrée la sortie de l'étape précédente.** **Toutes ces étapes sont hautement spécialisées (elles ont une fonction spécifique) et peuvent facilement être exécutées en parallèle**. En raison de leur nature parallèle, les cartes graphiques d'aujourd'hui possèdent des milliers de petits cœurs de traitement pour traiter rapidement vos données dans le pipeline graphique. Les cœurs de traitement exécutent de petits programmes sur le GPU pour chaque étape du pipeline. Ces petits programmes sont appelés **shaders**.  
  
Certains de ces shaders sont configurables par le développeur, ce qui nous permet d'écrire nos propres shaders pour remplacer les shaders par défaut existants. Cela nous donne un contrôle beaucoup plus fin sur des parties spécifiques du pipeline et, comme ils s'exécutent sur le GPU, ils peuvent aussi nous faire gagner un temps précieux sur le CPU. Les shaders sont écrits dans le langage OpenGL Shading Language (GLSL) et nous y reviendrons dans le prochain chapitre.

--- 

  
Vous trouverez ci-dessous une représentation abstraite de toutes les étapes du pipeline graphique. Notez que les sections bleues représentent les sections où nous pouvons injecter nos propres shaders. 
![[img/triangle1.png]]
Comme vous pouvez le constater, le pipeline graphique contient un grand nombre de sections qui gèrent chacune une partie spécifique de la **conversion de vos données de vertex en un pixel entièrement rendu**. Nous allons expliquer brièvement chaque partie du pipeline de manière simplifiée afin de vous donner une bonne vue d'ensemble de son fonctionnement.  
  
**En entrée du pipeline graphique, nous transmettons une liste de trois coordonnées 3D devant former un triangle dans un tableau appelé "données de sommet"** (vertex data) ; ces données de sommet sont une collection de sommets. **Un sommet est une collection de données par coordonnée 3D**. L**es données de ce sommet sont représentées à l'aide d'attributs de sommet qui peuvent contenir toutes les données souhaitées, mais pour des raisons de simplicité, supposons que chaque sommet se compose uniquement d'une position 3D et d'une valeur de couleur.**

>Pour qu'OpenGL sache quoi faire de votre collection de coordonnées et de valeurs de couleurs, il vous faut indiquer quel type de rendu vous souhaitez obtenir avec ces données. Voulons-nous que les données soient rendues sous la forme d'une collection de points, d'une collection de triangles ou peut-être simplement d'une longue ligne ? Ces indications sont appelées **primitives** et sont données à OpenGL lors de l'appel de n'importe quelle commande de dessin. Certaines de ces indications sont `GL_POINTS`, `GL_TRIANGLES` et `GL_LINE_STRIP`.

**La première partie du pipeline est le vertex shader qui prend en entrée un seul vertex**. **L'objectif principal du vertex shader est de transformer les coordonnées 3D en différentes coordonnées 3D** (nous y reviendrons plus tard) et le vertex shader nous permet d'effectuer quelques traitements de base sur les attributs du vertex.  
  
La sortie de l'étape du vertex shader est optionnellement transmise au geometry shader. **Le shader géométrique prend en entrée une collection de sommets qui forment une primitive et a la capacité de générer d'autres formes en émettant de nouveaux sommets pour former une nouvelle (ou d'autres) primitive(s).** Dans cet exemple, il génère un deuxième triangle à partir de la forme donnée.  
  
L'étape d'assemblage des primitives prend en entrée tous les sommets (ou vertex si `GL_POINTS` est choisi) du vertex shader (ou du geometry shader) qui forment une ou plusieurs primitives et assemble tous les points dans la forme primitive donnée ; dans ce cas, un triangle.  
  
La sortie du geometry shader est ensuite transmise à l'étape de rasterization, où elle fait correspondre la ou les primitives résultantes aux pixels correspondants sur l'écran final, ce qui donne des fragments à utiliser par le fragment shader. Avant l'exécution des fragment shaders, un clipping est effectué. Le clipping élimine tous les fragments qui se trouvent en dehors de votre vue, ce qui améliore les performances.

> En OpenGL, un **fragment** est l'ensemble des données nécessaires au rendu d'un seul pixel. 

**L'objectif principal du fragment shader est de calculer la couleur finale d'un pixel et c'est généralement à ce stade que se produisent tous les effets OpenGL avancés.** En général, le fragment shader contient des données sur la scène 3D qu'il peut utiliser pour calculer la couleur finale du pixel (comme les lumières, les ombres, la couleur de la lumière, etc.)  
  
Une fois que toutes les valeurs de couleur correspondantes ont été déterminées, l'objet final passe par une étape supplémentaire que nous appelons l'étape de test alpha et de mélange (**alpha stage et blending stage**). Cette étape vérifie la valeur de depth (et de stencil) correspondante (nous y reviendrons plus tard) du fragment et l'utilise pour vérifier si le fragment résultant se trouve devant ou derrière d'autres objets et s'il doit être écarté en conséquence. L'étape vérifie également les valeurs alpha (les valeurs alpha définissent l'opacité d'un objet) et mélange les objets en conséquence. Ainsi, même si la couleur de sortie d'un pixel est calculée dans le fragment shader, la couleur finale du pixel peut être totalement différente lors du rendu de plusieurs triangles.  
  
Comme vous pouvez le constater, le pipeline graphique est un ensemble assez complexe qui contient de nombreuses parties configurables. Cependant, dans la plupart des cas, nous n'avons à travailler qu'avec les vertex shaders et les fragment shaders. Le shader de géométrie est facultatif et généralement laissé à son shader par défaut. Il y a aussi l'étape de tessellation et la boucle de rétroaction de la transformation que nous n'avons pas décrite ici, mais ce sera pour plus tard.  
  
**Dans l'OpenGL moderne, nous devons définir au moins un vertex et un fragment shader propre (il n'y a pas de vertex/fragment shaders par défaut sur le GPU)**. Pour cette raison, il est souvent difficile de commencer à apprendre l'OpenGL moderne, car il faut beaucoup de connaissances avant de pouvoir effectuer le rendu de son premier triangle. Une fois que vous aurez rendu votre triangle à la fin de ce chapitre, vous en saurez beaucoup plus sur la programmation graphique.

## Vertex input
Pour commencer à dessiner quelque chose, nous devons d'abord fournir à OpenGL des données de vertex en entrée. OpenGL est une bibliothèque graphique 3D, donc toutes les coordonnées que nous spécifions dans OpenGL sont en 3D (coordonnées x, y et z). **OpenGL ne transforme pas simplement toutes vos coordonnées 3D en pixels 2D sur votre écran ; OpenGL ne traite les coordonnées 3D que lorsqu'elles se trouvent dans une plage spécifique entre -1,0 et 1,0 sur les 3 axes (x, y et z). Toutes les coordonnées situées dans cette plage de coordonnées normalisées seront visibles sur votre écran (et toutes les coordonnées situées en dehors de cette zone ne le seront pas).**  
  
Étant donné que nous voulons effectuer le rendu d'un seul triangle, nous voulons spécifier un total de trois sommets, chaque sommet ayant une position 3D. Nous les définissons en coordonnées normalisées (la région visible d'OpenGL) dans un tableau de flottants :
```cpp
float vertices[] = {
    -0.5f, -0.5f, 0.0f,
     0.5f, -0.5f, 0.0f,
     0.0f,  0.5f, 0.0f
};  
```
Comme OpenGL travaille dans l'espace 3D, nous rendons un triangle 2D dont chaque sommet a une coordonnée z de 0,0. De cette manière, la profondeur du triangle reste la même, ce qui donne l'impression qu'il s'agit d'un triangle en 2D. 

>Coordonnées normalisées (NDC: normalized device coordinates)  
  Une fois que les coordonnées de vos vertex ont été traitées dans le vertex shader, elles doivent être en coordonnées normalisées, c'est-à-dire dans un petit espace où les valeurs x, y et z varient de -1,0 à 1,0. Toutes les coordonnées qui se situent en dehors de cette plage seront rejetées/coupées et ne seront pas visibles à l'écran. Ci-dessous, vous pouvez voir le triangle que nous avons spécifié dans les coordonnées normalisées de l'appareil (en ignorant l'axe z):
  ![[img/triangle2.png]]
  Contrairement aux coordonnées d'écran habituelles, **l'axe y positif pointe vers le haut et les coordonnées (0,0) sont au centre du graphique**, au lieu d'être en haut à gauche. En fin de compte, vous souhaitez que toutes les coordonnées (transformées) se retrouvent dans cet espace de coordonnées, sinon elles ne seront pas visibles.  
  Vos coordonnées NDC seront ensuite transformées en coordonnées d'espace-écran (screen space) via la transformation de la fenêtre de visualisation en utilisant les données que vous avez fournies avec `glViewport`. Les coordonnées d'espace-écran résultantes sont ensuite transformées en fragments qui serviront d'entrées à votre fragment shader.

Une fois les données des sommets définies, nous souhaitons les envoyer en entrée au premier processus du pipeline graphique : le vertex shader. **Pour ce faire, nous créons une mémoire sur le GPU où nous stockons les données des sommets, nous configurons la manière dont OpenGL doit interpréter la mémoire et nous spécifions la manière d'envoyer les données à la carte graphique.** Le vertex shader traite ensuite autant de sommets que nous le lui demandons à partir de sa mémoire.  
  
Nous gérons cette mémoire par l'intermédiaire d'objets tampons (vertex buffer objects, VBO) qui peuvent stocker un grand nombre de sommets dans la mémoire du GPU. **L'avantage d'utiliser ces objets tampons est que nous pouvons envoyer de grandes quantités de données en une seule fois à la carte graphique, et les conserver s'il reste suffisamment de mémoire, sans avoir à envoyer les données un sommet à la fois**. L'envoi de données à la carte graphique depuis le processeur est relativement lent, c'est pourquoi nous essayons d'envoyer autant de données que possible en une seule fois. Une fois que les données sont dans la mémoire de la carte graphique, le vertex shader a un accès quasi instantané aux sommets, ce qui le rend extrêmement rapide.
  
Un objet tampon de vertex (VBO: vertex buffer object) est notre première occurrence d'un objet OpenGL, comme nous l'avons vu dans le chapitre sur OpenGL. Comme tout objet OpenGL, ce buffer a un identifiant unique correspondant à ce buffer, nous pouvons donc en générer un avec un identifiant de tampon en utilisant la fonction `glGenBuffers` :
```cpp
unsigned int VBO;
glGenBuffers(1, &VBO); 
```
 OpenGL possède de nombreux types d'objets tampons et **le type de tampon d'un objet tampon de sommet est `GL_ARRAY_BUFFER`**. OpenGL nous permet de nous lier à plusieurs tampons à la fois tant qu'ils ont un type de tampon différent. Nous pouvons lier le tampon nouvellement créé à la cible `GL_ARRAY_BUFFER` avec la fonction `glBindBuffer` :
```cpp
glBindBuffer(GL_ARRAY_BUFFER, VBO);  
``` 
A partir de là, tous les appels de tampon que nous faisons (sur la cible `GL_ARRAY_BUFFER`) seront utilisés pour configurer le tampon actuellement lié, qui est VBO. Nous pouvons ensuite appeler la fonction `glBufferData` pour copier les données de vertex précédemment définies dans la mémoire du tampon :
```cpp
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
```
`glBufferData` est une fonction spécifiquement destinée à copier des données définies par l'utilisateur dans le tampon actuellement lié.
Son premier argument est le type de tampon dans lequel nous voulons copier les données : l'objet tampon de sommet actuellement lié à la cible `GL_ARRAY_BUFFER`.
Le second argument spécifie la **taille des données (en octets)** que nous voulons passer au tampon ; une simple taille des données du sommet suffit.
Le troisième paramètre est la donnée que nous voulons envoyer.  
Le quatrième paramètre indique comment nous voulons que la carte graphique gère les données fournies. Il peut prendre 3 formes :
- `GL_STREAM_DRAW` : les données ne sont définies qu'une seule fois et utilisées par le GPU au maximum quelques fois.
- `GL_STATIC_DRAW` : les données ne sont définies qu'une seule fois et utilisées plusieurs fois.
- `GL_DYNAMIC_DRAW`: les données sont modifiées souvent et utilisées plusieurs fois.

Les données de position du triangle ne changent pas, sont très utilisées et restent les mêmes à chaque appel de rendu, de sorte que son type d'utilisation devrait être `GL_STATIC_DRAW`. Si, par exemple, un tampon contient des données susceptibles de changer fréquemment, le type d'utilisation `GL_DYNAMIC_DRAW` garantit que la carte graphique placera les données dans une mémoire permettant des écritures plus rapides.  
  
Jusqu'à présent, nous avons stocké les données des sommets dans la mémoire de la carte graphique, gérées par un objet tampon de sommets nommé VBO. Ensuite, nous voulons créer un vertex et un fragment shader qui traitent réellement ces données, alors commençons à les construire.

# Vertex shader
Le vertex shader est l'un des shaders programmables par des personnes comme nous. L'OpenGL moderne exige que nous configurions au moins un vertex et un fragment shader si nous voulons faire du rendu. Nous allons donc présenter brièvement les shaders et configurer deux shaders très simples pour dessiner notre premier triangle. Dans le prochain chapitre, nous aborderons les shaders plus en détail.  
  
La première chose à faire est d'écrire le vertex shader dans le langage GLSL (OpenGL Shading Language) et de compiler ce shader pour pouvoir l'utiliser dans notre application. Vous trouverez ci-dessous le code source d'un vertex shader très basique en GLSL :
```cpp
#version 330 core
layout (location = 0) in vec3 aPos;

void main()
{
    gl_Position = vec4(aPos.x, aPos.y, aPos.z, 1.0);
}
```
Comme vous pouvez le voir, GLSL ressemble au langage C. Chaque shader commence par une déclaration de sa version. Depuis OpenGL 3.3 et plus, les numéros de version de GLSL correspondent à la version d'OpenGL (la version 420 de GLSL correspond à la version 4.2 d'OpenGL par exemple). Nous mentionnons aussi explicitement que nous utilisons les fonctionnalités du profil de base (core).  
  
Ensuite, nous déclarons tous les attributs de vertex d'entrée dans le vertex shader avec le mot-clé `in`. Pour l'instant, nous ne nous intéressons qu'aux données de position, nous n'avons donc besoin que d'un seul attribut de sommet. GLSL dispose d'un type de données vectorielles qui contient de 1 à 4 nombres flottants en fonction de leur chiffre postfixe. Comme chaque sommet a une coordonnée 3D, nous créons une variable d'entrée `vec3` avec le nom `aPos`. Nous définissons également l'emplacement de la variable d'entrée via le `layout (location = 0)` et vous verrez plus tard pourquoi nous aurons besoin de cet emplacement.

>**Vecteur**    
	En programmation graphique, nous utilisons assez souvent le concept mathématique de vecteur, car il représente proprement des positions/directions dans n'importe quel espace et possède des propriétés mathématiques utiles. Un vecteur en GLSL a une taille maximale de 4 et chacune de ses valeurs peut être récupérée via `vec.x`, `vec.y`, `vec.z` et `vec.w` respectivement où chacune d'entre elles représente une coordonnée dans l'espace. Notez que la composante `vec.w` n'est pas utilisée comme position dans l'espace (nous sommes en 3D, pas en 4D) mais est utilisée pour ce qu'on appelle la **division de la perspective**. Nous aborderons les vecteurs de manière plus approfondie dans un chapitre ultérieur.

Pour définir la sortie du vertex shader, nous devons assigner les données de position à la variable prédéfinie `gl_Position` qui est un `vec4` dans les coulisses. À la fin de la fonction principale, la valeur de la variable `gl_Position` sera utilisée comme sortie du vertex shader. Puisque notre entrée est un vecteur de taille 3, nous devons le convertir en un vecteur de taille 4. Nous pouvons le faire en insérant les valeurs `vec3` dans le constructeur de `vec4` et en fixant sa composante `w` à `1.0f` (nous expliquerons pourquoi dans un chapitre ultérieur).  
  
**Le vertex shader actuel est probablement le vertex shader le plus simple que l'on puisse imaginer** car nous n'avons effectué aucun traitement sur les données d'entrée et les avons simplement transmises à la sortie du vertex shader. **Dans les applications réelles, les données d'entrée ne sont généralement pas déjà en coordonnées normalisées, nous devons donc d'abord transformer les données d'entrée en coordonnées qui tombent dans la région visible d'OpenGL.**

## Compiler un shader
Nous prenons le code source du vertex shader et le stockons dans une const C-string en haut du fichier de code pour l'instant :
```cpp
const char *vertexShaderSource = "#version 330 core\n"
    "layout (location = 0) in vec3 aPos;\n"
    "void main()\n"
    "{\n"
    "   gl_Position = vec4(aPos.x, aPos.y, aPos.z, 1.0);\n"
    "}\0";
```
Pour qu'OpenGL puisse utiliser le shader, il doit le compiler dynamiquement au moment de l'exécution à partir de son code source. La première chose à faire est de créer un objet shader, encore une fois référencé par un ID. Nous stockons donc le vertex shader sous la forme d'un unsigned int et créons le shader avec `glCreateShader`:
```cpp
unsigned int vertexShader;
vertexShader = glCreateShader(GL_VERTEX_SHADER);
```
Nous fournissons le type de shader que nous voulons créer comme argument à `glCreateShader`. Puisque nous créons un vertex shader, nous passons `GL_VERTEX_SHADER`.

Ensuite, nous attachons le code source du shader à l'objet shader et compilons le shader :
```cpp
glShaderSource(vertexShader, 1, &vertexShaderSource, NULL);
glCompileShader(vertexShader);
```
La fonction `glShaderSource` prend comme premier argument l'objet shader à compiler. Le deuxième argument spécifie le nombre de chaînes de caractères que nous transmettons comme code source, c'est-à-dire une seule. Le troisième paramètre est le code source réel du vertex shader et nous pouvons laisser le quatrième paramètre à NULL.

>	Vous voudrez probablement vérifier si la compilation a réussi après l'appel à `glCompileShader` et, si ce n'est pas le cas, quelles erreurs ont été trouvées afin que vous puissiez les corriger. La vérification des erreurs de compilation s'effectue de la manière suivante :	
```cpp
		int  success;
		char infoLog[512];
		glGetShaderiv(vertexShader, GL_COMPILE_STATUS, &success);
```
>	Tout d'abord, nous définissons un entier pour indiquer le succès et un conteneur de stockage pour les messages d'erreur (le cas échéant). Ensuite, nous vérifions si la compilation a réussi avec `glGetShaderiv`. Si la compilation a échoué, nous devons récupérer le message d'erreur avec `glGetShaderInfoLog` et l'imprimer.
```cpp
if(!success)
{
    glGetShaderInfoLog(vertexShader, 512, NULL, infoLog);
    std::cout << "ERROR::SHADER::VERTEX::COMPILATION_FAILED\n" << infoLog << std::endl;
}
```
 Si aucune erreur n'a été détectée lors de la compilation du vertex shader, celui-ci est maintenant compilé. 

## Fragment shader
Le fragment shader est le deuxième et dernier shader que nous allons créer pour le rendu d'un triangle. **Le fragment shader calcule la couleur de sortie de vos pixels**. Pour simplifier les choses, le fragment shader produira toujours une couleur orangée.

>En infographie, les couleurs sont représentées sous la forme d'un tableau de 4 valeurs : les composantes **rouge**, **verte**, **bleue** et **alpha** (**opacité**), communément abrégées en **RGBA**. Lors de la définition d'une couleur dans OpenGL ou GLSL, nous fixons l'intensité de chaque composante à une valeur comprise entre 0,0 et 1,0. Si, par exemple, nous fixons le rouge à 1,0 et le vert à 1,0, nous obtenons un mélange des deux couleurs et la couleur jaune. Avec ces trois composantes de couleur, nous pouvons générer plus de 16 millions de couleurs différentes !

```cpp
#version 330 core
out vec4 FragColor;

void main()
{
    FragColor = vec4(1.0f, 0.5f, 0.2f, 1.0f);
} 
```
Le fragment shader ne nécessite qu'une seule variable de sortie, un vecteur de taille 4 qui définit la couleur finale que nous devons calculer nous-mêmes. Nous pouvons déclarer des valeurs de sortie avec le mot-clé `out`, que nous avons rapidement nommé `FragColor`. Ensuite, nous assignons simplement un `vec4` à la couleur de sortie comme une couleur orange avec une valeur alpha de `1,0` (`1,0` étant complètement opaque).  
  
Le processus de compilation d'un fragment shader est similaire à celui du vertex shader, bien que cette fois nous utilisions la constante `GL_FRAGMENT_SHADER` comme type de shader :
```cpp
unsigned int fragmentShader;
fragmentShader = glCreateShader(GL_FRAGMENT_SHADER);
glShaderSource(fragmentShader, 1, &fragmentShaderSource, NULL);
glCompileShader(fragmentShader);
```
Les deux shaders sont maintenant compilés et la seule chose qui reste à faire est de lier les deux objets shaders dans un programme shader que nous pouvons utiliser pour le rendu. Assurez-vous de vérifier les erreurs de compilation ici aussi !

## Shader program
Un objet de programme de shader est la version finale liée de plusieurs shaders combinés. **Pour utiliser les shaders récemment compilés, nous devons les lier à un objet programme de shaders, puis activer ce programme de shaders lors du rendu des objets**. Les shaders du programme de shaders activé seront utilisés lors des appels de rendu.  
  
Lorsque l'on lie les shaders à un programme, les sorties de chaque shader sont liées aux entrées du shader suivant. **C'est également à ce stade que vous obtiendrez des erreurs de liaison si vos sorties et vos entrées ne correspondent pas**.  
  
La création d'un objet programme est facile :
```cpp
unsigned int shaderProgram;
shaderProgram = glCreateProgram();
```
La fonction `glCreateProgram` crée un programme et renvoie la référence ID à l'objet programme nouvellement créé. Nous devons maintenant attacher les shaders précédemment compilés à l'objet programme et les lier avec `glLinkProgram` :
```cpp
glAttachShader(shaderProgram, vertexShader);
glAttachShader(shaderProgram, fragmentShader);
glLinkProgram(shaderProgram);
```
Le code devrait être assez explicite, nous attachons les shaders au programme et les lions via `glLinkProgram`.

> 	Tout comme pour la compilation des shaders, nous pouvons également vérifier si la liaison d'un programme de shaders a échoué et récupérer le journal correspondant. Cependant, au lieu d'utiliser `glGetShaderiv` et `glGetShaderInfoLog`, nous utilisons maintenant :
```cpp
glGetProgramiv(shaderProgram, GL_LINK_STATUS, &success);
if(!success) {
    glGetProgramInfoLog(shaderProgram, 512, NULL, infoLog);
    ...
}
```
Le résultat est un objet programme que nous pouvons activer en appelant `glUseProgram` avec l'objet programme nouvellement créé comme argument :
```cpp
glUseProgram(shaderProgram);
```
Chaque appel de shader et de rendu après `glUseProgram` utilisera désormais cet objet programme (et donc les shaders).

Oh oui, et n'oubliez pas de supprimer les objets shaders une fois que nous les avons liés à l'objet programme ; nous n'en avons plus besoin : 
```cpp
glDeleteShader(vertexShader);
glDeleteShader(fragmentShader);  
```
Pour l'instant, nous avons envoyé les données de vertex d'entrée au GPU et indiqué au GPU comment traiter les données de vertex dans un vertex shader et un fragment shader. Nous y sommes presque, mais pas encore tout à fait. OpenGL ne sait pas encore comment il doit interpréter les données de vertex en mémoire et comment il doit connecter les données de vertex aux attributs du vertex shader. Nous allons être gentils et dire à OpenGL comment le faire.

## Lier les attributs des vertex (Linking vertex attributes)
**Le vertex shader nous permet de spécifier n'importe quelle entrée sous la forme d'attributs de vertex et bien que cela permette une grande flexibilité, cela signifie que nous devons spécifier manuellement quelle partie de nos données d'entrée va à quel attribut de vertex dans le vertex shader**. Cela signifie que **nous devons spécifier comment OpenGL doit interpréter les données de vertex avant le rendu.**  
  
Nos données de buffer de vertex (VBO) sont formatées comme suit :
![[img/triangle3.png]]
- Les données de position sont stockées sous forme de valeurs à virgule flottante de 32 bits (4 octets).
- Chaque position est composée de 3 de ces valeurs.
- Il n'y a pas d'espace (ou d'autres valeurs) entre chaque série de 3 valeurs. Les valeurs sont étroitement regroupées dans le tableau.
- La première valeur des données se trouve au début de la mémoire tampon.

 Avec cette connaissance, nous pouvons dire à OpenGL comment il doit interpréter les données des vertex (par attribut de vertex) en utilisant `glVertexAttribPointer` :
```cpp
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);
```
 La fonction `glVertexAttribPointer` a un certain nombre de paramètres, nous allons donc les examiner attentivement :
 - **Le premier paramètre spécifie l'attribut de vertex que nous voulons configurer**. Rappelez-vous que nous avons spécifié l'emplacement de l'attribut de vertex position dans le vertex shader avec `layout (location = 0)`. Ceci place l'emplacement de l'attribut de sommet à `0` et puisque nous voulons passer des données à cet attribut de sommet, nous passons `0`.  
- **L'argument suivant spécifie la taille de l'attribut vertex**. L'attribut vertex est un `vec3` donc il est composé de `3` valeurs.  
- **Le troisième argument spécifie le type des données** qui est GL_FLOAT (un `vec*` en GLSL consiste en des valeurs à virgule flottante).  
- **L'argument suivant spécifie si nous voulons que les données soient normalisées**. Si nous entrons des données de type integer (int, byte) et que nous avons mis GL_TRUE, les données integer sont normalisées à `0` (ou `-1` pour les données signées) et `1` lorsqu'elles sont converties en float. Ceci n'est pas pertinent pour nous, donc nous laisserons ce paramètre à GL_FALSE.  
- **Le cinquième argument est connu sous le nom de stride** **et nous indique l'espace entre les attributs de vertex consécutifs. Puisque le prochain jeu de données de position est situé exactement à 3 fois la taille d'un `float`, nous spécifions cette valeur comme étant le stride.** Notez que puisque nous savons que le tableau est très compact (il n'y a pas d'espace entre la valeur de l'attribut de sommet suivant), nous aurions pu spécifier la valeur `0` pour laisser OpenGL déterminer la distance (cela ne fonctionne que lorsque les valeurs sont très compactes). Dès que nous avons plus d'attributs de vertex, nous devons soigneusement définir l'espacement entre chaque attribut de vertex, mais nous verrons d'autres exemples plus tard.  
- Le dernier paramètre est de type `void*` et nécessite donc ce cast bizarre. C'est le décalage de l'endroit où les données de position commencent dans le buffer. Puisque les données de position sont au début du tableau de données, cette valeur est juste `0`. Nous explorerons ce paramètre plus en détail plus tard.

>Chaque attribut de sommet prend ses données dans la mémoire gérée par un VBO et le VBO dans lequel il prend ses données (vous pouvez avoir plusieurs VBO) est déterminé par le VBO actuellement lié à GL_ARRAY_BUFFER lors de l'appel à `glVertexAttribPointer`. Puisque le VBO précédemment défini est toujours lié avant d'appeler `glVertexAttribPointer`, l'attribut de sommet 0 est maintenant associé à ses données de sommet.

Maintenant que nous avons spécifié comment OpenGL doit interpréter les données de vertex, nous devons également activer l'attribut de vertex avec `glEnableVertexAttribArray` en donnant l'emplacement de l'attribut de vertex comme argument ; les attributs de vertex sont désactivés par défaut. A partir de là, tout est prêt : nous avons initialisé les données de vertex dans un tampon en utilisant un objet tampon de vertex (VBO), mis en place un vertex shader et un fragment shader et indiqué à OpenGL comment lier les données de vertex aux attributs de vertex du vertex shader. Dessiner un objet dans OpenGL ressemblerait maintenant à ceci :
```cpp
// 0. copy our vertices array in a buffer for OpenGL to use
glBindBuffer(GL_ARRAY_BUFFER, VBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
// 1. then set the vertex attributes pointers
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);  
// 2. use our shader program when we want to render an object
glUseProgram(shaderProgram);
// 3. now draw the object 
someOpenGLFunctionThatDrawsOurTriangle();  
```
Nous devons répéter ce processus chaque fois que nous voulons dessiner un objet. Cela n'a l'air de rien, mais imaginez que nous ayons plus de 5 attributs de vertex et peut-être des centaines d'objets différents (ce qui n'est pas rare). Lier les objets tampons appropriés et configurer tous les attributs de vertex pour chacun de ces objets devient rapidement un processus fastidieux. Et s'il existait un moyen de stocker toutes ces configurations d'état dans un objet et de lier simplement cet objet pour restaurer son état ?

## Vertex Array Object (VAO)
Un objet tableau de vertex (également connu sous le nom de VAO: vertex array object) peut être lié de la même manière qu'un objet VBO et tous les appels d'attributs de vertex ultérieurs seront stockés à l'intérieur du VAO. L'avantage est que lors de la configuration des pointeurs d'attributs de vertex, vous n'avez à faire ces appels qu'une seule fois et que chaque fois que nous voulons dessiner l'objet, nous pouvons simplement lier le VAO correspondant. Le passage d'une configuration de données et d'attributs de vertex à une autre est donc aussi simple que de lier un VAO différent. Tout l'état que nous venons de définir est stocké dans le VAO. 

> Le noyau d'OpenGL exige que nous utilisions un VAO afin qu'il sache quoi faire avec nos entrées de vertex. Si nous ne parvenons pas à lier un VAO, OpenGL refusera probablement de dessiner quoi que ce soit. 

Un VAO contient les éléments suivants :
- Appels à `glEnableVertexAttribArray` ou `glDisableVertexAttribArray`.
- Configurations des attributs de sommet via `glVertexAttribPointer`.
- VBO associés aux attributs de sommet par des appels à `glVertexAttribPointer`.
![[img/triangle4.png]]
 Le processus de création d'un VAO est similaire à celui d'un VBO :
```cpp
unsigned int VAO;
glGenVertexArrays(1, &VAO);  
```
Pour utiliser un VAO, il suffit de le lier à l'aide de `glBindVertexArray`. À partir de là, nous devons lier/configurer le(s) VBO(s) et le(s) pointeur(s) d'attribut(s) correspondant(s), puis délier le VAO en vue d'une utilisation ultérieure. Dès que nous voulons dessiner un objet, nous lions simplement le VAO avec les paramètres souhaités avant de dessiner l'objet, et c'est tout. En code, cela ressemblerait un peu à ceci :
```cpp
// ..:: Initialization code (done once (unless your object frequently changes)) :: ..
// 1. bind Vertex Array Object
glBindVertexArray(VAO);
// 2. copy our vertices array in a buffer for OpenGL to use
glBindBuffer(GL_ARRAY_BUFFER, VBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
// 3. then set our vertex attributes pointers
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);  

  
[...]

// ..:: Drawing code (in render loop) :: ..
// 4. draw the object
glUseProgram(shaderProgram);
glBindVertexArray(VAO);
someOpenGLFunctionThatDrawsOurTriangle();  
```
Et c'est tout ! Tout ce que nous avons fait au cours des quelques millions de pages précédentes a conduit à ce moment, **un VAO qui stocke notre configuration d'attributs de vertex et le VBO à utiliser.** **Habituellement, lorsque vous souhaitez dessiner plusieurs objets, vous commencez par générer/configurer tous les VAO (et donc les pointeurs de VBO et d'attributs requis) et vous les stockez en vue d'une utilisation ultérieure. Lorsque nous voulons dessiner l'un de nos objets, nous prenons le VAO correspondant, nous le lions, nous dessinons l'objet et nous délions à nouveau le VAO.**

## Le triangle que nous attendions tous
Pour dessiner les objets de notre choix, OpenGL nous fournit la fonction `glDrawArrays` qui dessine les primitives en utilisant le shader actif, la configuration des attributs de vertex définie précédemment et les données de vertex du VBO (indirectement liées via le VAO).

```cpp
glUseProgram(shaderProgram);
glBindVertexArray(VAO);
glDrawArrays(GL_TRIANGLES, 0, 3);
```
La fonction `glDrawArrays` prend comme premier argument le type de primitive OpenGL que nous souhaitons dessiner. Puisque j'ai dit au début que nous voulions dessiner un triangle, et que je n'aime pas vous mentir, nous passons `GL_TRIANGLES`.
Le deuxième argument spécifie l'indice de départ du tableau de sommets que nous souhaitons dessiner ; nous le laissons à 0.
Le dernier argument spécifie le nombre de sommets que nous souhaitons dessiner, soit 3 (nous ne rendons qu'un seul triangle à partir de nos données, qui font exactement 3 sommets).  
  
Essayez maintenant de compiler le code et revenez en arrière si des erreurs apparaissent. Dès que votre application se compile, vous devriez voir le résultat suivant :
![[img/triangle5.png]]
Le code source du programme complet se trouve [ici](https://learnopengl.com/code_viewer_gh.php?code=src/1.getting_started/2.1.hello_triangle/hello_triangle.cpp) .

Si votre résultat n'est pas le même, vous avez probablement fait une erreur en cours de route. Vérifiez donc le code source complet et voyez si vous avez manqué quelque chose.

## Element buffer objects
Il y a une dernière chose que nous aimerions aborder lors du rendu des sommets : les objets tampons d'éléments, abrégés en EBO (Element Buffer Object). Pour expliquer le fonctionnement des objets tampons d'éléments, il est préférable de donner un exemple : supposons que nous voulions dessiner un rectangle au lieu d'un triangle. Nous pouvons dessiner un rectangle en utilisant deux triangles (OpenGL fonctionne principalement avec des triangles). Cela générera l'ensemble de sommets suivant :
```cpp
float vertices[] = {
    // first triangle
     0.5f,  0.5f, 0.0f,  // top right
     0.5f, -0.5f, 0.0f,  // bottom right
    -0.5f,  0.5f, 0.0f,  // top left 
    // second triangle
     0.5f, -0.5f, 0.0f,  // bottom right
    -0.5f, -0.5f, 0.0f,  // bottom left
    -0.5f,  0.5f, 0.0f   // top left
}; 
```
**Comme vous pouvez le constater, les sommets spécifiés se chevauchent quelque peu**. Nous spécifions en bas à droite et en haut à gauche deux fois ! Il s'agit d'une surcharge de 50% puisque le même rectangle pourrait également être spécifié avec seulement 4 sommets, au lieu de 6. Cela ne fera qu'empirer dès que nous aurons des modèles plus complexes comportant plus de 1000 triangles où il y aura de grandes parties qui se chevaucheront. **La meilleure solution serait de ne stocker que les sommets uniques et de spécifier ensuite l'ordre dans lequel nous voulons dessiner ces sommets**. Dans ce cas, nous n'aurions qu'à stocker 4 sommets pour le rectangle, puis à spécifier l'ordre dans lequel nous souhaitons les dessiner. Ne serait-ce pas génial si OpenGL nous fournissait une telle fonctionnalité ?  
  
Heureusement, les objets tampons d'éléments fonctionnent exactement comme cela. Un EBO est un tampon, tout comme un objet tampon de sommet (VBO), **qui stocke des indices qu'OpenGL utilise pour décider quels sommets dessiner.** Ce qu'on appelle **le dessin indexé** est exactement la solution à notre problème. Pour commencer, nous devons d'abord spécifier les sommets (uniques) et les indices pour les dessiner sous forme de rectangle :
```cpp
float vertices[] = {
     0.5f,  0.5f, 0.0f,  // top right
     0.5f, -0.5f, 0.0f,  // bottom right
    -0.5f, -0.5f, 0.0f,  // bottom left
    -0.5f,  0.5f, 0.0f   // top left 
};
unsigned int indices[] = {  // note that we start from 0!
    0, 1, 3,   // first triangle
    1, 2, 3    // second triangle
};  
```
Vous pouvez constater qu'en utilisant des indices, nous n'avons besoin que de 4 sommets au lieu de 6. Ensuite, nous devons créer l'objet tampon de l'élément (EBO):
```cpp
unsigned int EBO;
glGenBuffers(1, &EBO);
```
Comme pour le VBO, nous lions l'EBO et copions les indices dans le tampon avec `glBufferData`. De même, comme pour le VBO, nous voulons placer ces appels entre un appel bind et un appel unbind, bien que cette fois-ci nous spécifions `GL_ELEMENT_ARRAY_BUFFER` comme type de tampon. 
```cpp
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW);
```
Notez que nous donnons maintenant `GL_ELEMENT_ARRAY_BUFFER` comme cible du tampon. La dernière chose qu'il nous reste à faire est de remplacer l'appel à `glDrawArrays` par `glDrawElements` pour indiquer que nous voulons rendre les triangles à partir d'un tampon d'index. Lors de l'utilisation de `glDrawElements`, nous allons dessiner en utilisant les indices fournis dans l'objet tampon d'élément actuellement lié :
```cpp
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);
```
Le premier argument spécifie le mode dans lequel nous voulons dessiner, comme pour `glDrawArrays`.
Le deuxième argument est le nombre d'éléments à dessiner. Nous avons spécifié 6 indices pour dessiner 6 sommets au total.
Le troisième argument est le type des indices qui est de type GL_UNSIGNED_INT.
Le dernier argument nous permet de spécifier un offset dans l'EBO (ou de passer un tableau d'index, mais c'est lorsque vous n'utilisez pas d'objets tampons d'éléments), mais nous allons simplement le laisser à 0.  
  
La fonction `glDrawElements` prend ses indices dans l'EBO actuellement lié à la cible `GL_ELEMENT_ARRAY_BUFFER`. Cela signifie que nous devons lier l'EBO correspondant à chaque fois que nous voulons rendre un objet avec des indices, ce qui est encore une fois un peu encombrant. Il se trouve qu'un objet de tableau de vertex garde également la trace des liaisons d'objets de tampon d'élément. Le dernier objet de tampon d'élément qui est lié pendant qu'un VAO est lié est stocké en tant qu'objet de tampon d'élément du VAO. Le fait de lier un VAO lie donc automatiquement cet EBO.
![[img/triangle6.png]]
> 	Un VAO stocke les appels `glBindBuffer` lorsque la cible est `GL_ELEMENT_ARRAY_BUFFER`. Cela signifie également qu'il stocke ses appels de déliaison. Assurez-vous donc de ne pas délier le tampon du tableau d'éléments avant de délier votre VAO, sinon il n'aura pas d'EBO configuré.

 Le code d'initialisation et de dessin qui en résulte ressemble maintenant à ceci : 
 ```cpp
// ..:: Initialization code :: ..
// 1. bind Vertex Array Object
glBindVertexArray(VAO);
// 2. copy our vertices array in a vertex buffer for OpenGL to use
glBindBuffer(GL_ARRAY_BUFFER, VBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
// 3. copy our index array in a element buffer for OpenGL to use
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW);
// 4. then set the vertex attributes pointers
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);  

[...]
  
// ..:: Drawing code (in render loop) :: ..
glUseProgram(shaderProgram);
glBindVertexArray(VAO);
glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);
glBindVertexArray(0);
```
L'exécution du programme devrait permettre d'obtenir l'image ci-dessous. L'image de gauche devrait vous sembler familière et l'image de droite est le rectangle dessiné en mode filaire. Le rectangle en mode filaire montre que le rectangle est en fait constitué de deux triangles.
![[img/triangle7.png]]
>**Mode filaire **
	Pour dessiner vos triangles en mode filaire, vous pouvez configurer la façon dont OpenGL dessine ses primitives via `glPolygonMode(GL_FRONT_AND_BACK, GL_LINE)`. Le premier argument indique que nous voulons l'appliquer à l'avant et à l'arrière de tous les triangles et la deuxième ligne nous indique de les dessiner comme des lignes. Tout appel de dessin ultérieur rendra les triangles en mode filaire jusqu'à ce que nous le ramenions à sa valeur par défaut en utilisant `glPolygonMode(GL_FRONT_AND_BACK, GL_FILL)`.

Si vous avez des erreurs, revenez en arrière et voyez si vous n'avez rien oublié. Vous pouvez trouver le code source complet [ici](https://learnopengl.com/code_viewer_gh.php?code=src/1.getting_started/2.2.hello_triangle_indexed/hello_triangle_indexed.cpp).  
  
Si vous avez réussi à dessiner un triangle ou un rectangle comme nous l'avons fait, alors félicitations, vous avez réussi à passer l'une des parties les plus difficiles de l'OpenGL moderne : dessiner votre premier triangle. C'est une partie difficile car il y a une grande quantité de connaissances requises avant d'être capable de dessiner votre premier triangle. Heureusement, nous avons franchi cette barrière et les chapitres à venir seront, je l'espère, beaucoup plus faciles à comprendre.

## Ressources additionnelles
- [antongerdelan.net/hellotriangle](http://antongerdelan.net/opengl/hellotriangle.html) : Le point de vue d'Anton Gerdelan sur le rendu du premier triangle.  
- [open.gl/drawing](https://open.gl/drawing) : Alexander Overvoorde sur le rendu du premier triangle.  
- [antongerdelan.net/vertexbuffers](http://antongerdelan.net/opengl/vertexbuffers.html) : quelques informations supplémentaires sur les objets tampons de vertex.  (VBO)
- learnopengl.com/In-Practice/Debugging](https://learnopengl.com/In-Practice/Debugging) : il y a beaucoup d'étapes dans ce chapitre ; si vous êtes bloqué, il peut être utile de lire un peu sur le débogage dans OpenGL (jusqu'à la section sur la sortie de débogage).