---
tags: [cg, opengl, computer graphics, cpp]
dg-publish: true
---
# Transformations
Nous savons maintenant comment créer des objets, les colorer et/ou leur donner une apparence détaillée à l'aide de textures, mais ils ne sont toujours pas très intéressants puisqu'il s'agit d'objets statiques. Nous pourrions essayer de les faire bouger en changeant leurs sommets et en reconfigurant leurs buffers à chaque image, **mais c'est lourd et coûteux en puissance de traitement**. **Il existe de bien meilleures façons de transformer un objet, et c'est en utilisant des objets matriciels** (multiples). Cela ne signifie pas que nous allons parler de Kung Fu et d'un vaste monde artificiel numérique.  (jeu de mot avec matrix)
  
Les matrices sont des constructions mathématiques très puissantes qui semblent effrayantes au début, mais qui se révèlent extrêmement utiles une fois que l'on s'y est habitué. Lorsque nous parlerons des matrices, nous devrons faire une petite plongée dans les mathématiques et pour les lecteurs les plus enclins aux mathématiques, je publierai des ressources supplémentaires pour une lecture plus approfondie.  
  
Toutefois, pour bien comprendre les transformations, nous devons d'abord approfondir la question des vecteurs avant d'aborder celle des matrices. L'objectif de ce chapitre est de vous donner un bagage mathématique de base pour les sujets dont nous aurons besoin plus tard. Si les sujets sont difficiles, essayez de les comprendre autant que possible et revenez à ce chapitre plus tard pour revoir les concepts lorsque vous en aurez besoin.

## Vecteurs
Dans leur définition la plus élémentaire, **les vecteurs sont des directions et rien de plus**. **Un vecteur a une direction et une magnitude** (également appelée force ou longueur). On peut comparer les vecteurs à des indications sur une carte au trésor : "aller à gauche sur 10 pas, puis aller au nord sur 3 pas et à droite sur 5 pas".
Ici, "gauche" est la direction et "10 pas" est la magnitude du vecteur. Les directions de la carte au trésor contiennent donc 3 vecteurs. **Les vecteurs peuvent avoir n'importe quelle dimension, mais nous travaillons généralement avec des dimensions de 2 à 4**. Si un vecteur a 2 dimensions, il représente une direction sur un plan (pensez aux graphiques en 2D) et s'il a 3 dimensions, il peut représenter n'importe quelle direction dans un monde en 3D.  
  
Ci-dessous, vous verrez 3 vecteurs où chaque vecteur est représenté par des flèches (x,y) dans un graphique en 2D. Comme il est plus intuitif d'afficher les vecteurs en 2D (plutôt qu'en 3D), vous pouvez considérer les vecteurs 2D comme des vecteurs 3D avec une coordonnée z de 0. Comme les vecteurs représentent des directions, l'origine du vecteur ne change pas sa valeur. Dans le graphique ci-dessous, nous pouvons voir que les vecteurs $\vec{v}$ et $\vec{w}$ sont égaux bien que leur origine soit différente :
![[img/transfo1.png]]
Pour décrire les vecteurs, les mathématiciens préfèrent généralement utiliser des symboles de caractères avec une petite barre au-dessus de leur tête, comme v¯. De même, lorsque les vecteurs sont affichés dans des formules, ils sont généralement présentés comme suit :

$$\vec{v} = \begin{pmatrix}
x\\
y\\
z
\end{pmatrix}$$
Comme les vecteurs sont spécifiés en tant que directions, il est parfois difficile de les visualiser en tant que positions. Si nous voulons visualiser les vecteurs comme des positions, nous pouvons imaginer que l'origine du vecteur de direction est (0,0,0) et qu'il pointe ensuite vers une certaine direction qui spécifie le point, ce qui en fait un vecteur de position (nous pourrions également spécifier une origine différente et dire : "ce vecteur pointe vers ce point dans l'espace à partir de cette origine"). Le vecteur position (3,5) pointerait alors vers (3,5) sur le graphique dont l'origine est (0,0). **Les vecteurs permettent donc de décrire des directions et des positions dans l'espace en 2D et en 3D.**  
  
Tout comme pour les nombres normaux, nous pouvons également définir plusieurs opérations sur les vecteurs (dont certaines ont déjà été vues).
### Opérations scalaires sur les vecteurs
Un scalaire est un chiffre unique. Lors de l'addition/soustraction/multiplication ou de la division d'un vecteur avec un scalaire, il suffit d'ajouter/soustraire/multiplier ou diviser chaque élément du vecteur par le scalaire. Dans le cas d'une addition, cela donnerait ceci : 
$$\begin{pmatrix}
1\\
2\\
3
\end{pmatrix} + x
= \begin{pmatrix}
1\\
2\\
3
\end{pmatrix} +
\begin{pmatrix}
x\\
x\\
x
\end{pmatrix} =
\begin{pmatrix}
1 + x\\
2 + x\\
3 + x
\end{pmatrix}
$$
Où $+$ peut être $+$, $-$, $⋅$ ou $÷$ où $⋅$ est l'opérateur de multiplication. 

### Négation du vecteur
**La négation d'un vecteur entraîne l'apparition d'un vecteur dans la direction inverse**. Un vecteur pointant vers le nord-est pointera vers le sud-ouest après la négation. Pour inverser un vecteur, nous ajoutons un signe moins à chaque composante (vous pouvez également le représenter comme une multiplication scalaire vectorielle avec une valeur scalaire de -1) : 
$$
-\vec{v}
= -\begin{pmatrix}
v_x\\
v_y\\
v_z
\end{pmatrix} =
\begin{pmatrix}
-v_x\\
-v_y\\
-v_z
\end{pmatrix}
$$
### Addition et soustraction
L'addition de deux vecteurs est définie comme une addition par composante, c'est-à-dire que chaque composante d'un vecteur est ajoutée à la même composante de l'autre vecteur de la manière suivante : 

$$
\vec{v}
= \begin{pmatrix}
1\\
2\\
3
\end{pmatrix}
,
\vec{k}
= \begin{pmatrix}
4\\
5\\
6
\end{pmatrix}
=>
\vec{v} + \vec{k} =
\begin{pmatrix}
1 + 4\\
2 + 5\\
3 + 6
\end{pmatrix}
=
\begin{pmatrix}
5\\
7\\
9
\end{pmatrix}
$$
Visuellement, cela ressemble à ceci pour les vecteurs $\vec{v}=(4,2)$ et $\vec{k}=(1,2)$, où le deuxième vecteur est ajouté à l'extrémité du premier vecteur pour trouver le point final du vecteur résultant (méthode de la tête à la queue (head to tail method)) :
![[img/transfo2.png]]
Tout comme l'addition et la soustraction normales, la soustraction de vecteurs est la même chose que l'addition avec un second vecteur annulé : 
$$
\vec{v}
= \begin{pmatrix}
1\\
2\\
3
\end{pmatrix}
,
\vec{k}
= \begin{pmatrix}
4\\
5\\
6
\end{pmatrix}
=>
\vec{v} + -\vec{k} =
\begin{pmatrix}
1 + (-4)\\
2 + (-5)\\
3 + (-6)
\end{pmatrix}
=
\begin{pmatrix}
-3\\
-3\\
-3
\end{pmatrix}
$$
En soustrayant deux vecteurs l'un de l'autre, on obtient un vecteur qui est la différence des positions vers lesquelles pointent les deux vecteurs. Cela s'avère utile dans certains cas où nous devons récupérer un vecteur qui est la différence entre deux points. 
![[img/transfo3.png]]
### Longueur
Pour déterminer la longueur/magnitude d'un vecteur, nous utilisons le théorème de Pythagore dont vous vous souvenez peut-être de vos cours de mathématiques. Un vecteur forme un triangle lorsque vous visualisez ses composantes x et y individuelles comme les deux côtés d'un triangle :
![[img/transfo4.png]]
Puisque les longueurs des deux côtés (x, y) sont connues et que nous voulons connaître la longueur du côté incliné $\vec{v}$, nous pouvons la calculer en utilisant le théorème de Pythagore comme suit :
$$
||\vec{v}||
=
\sqrt{x^2 + y^2}
$$
Où $||\vec{v}||$  est noté comme la longueur du vecteur $\vec{v}$. Il est facile d'étendre ce principe à la 3D en ajoutant $z^2$ à l'équation.
Dans ce cas, la longueur du vecteur $(4, 2)$ est égale à :
$$
||\vec{v}||
=
\sqrt{4^2 + 2^2}
=
\sqrt{16 + 4}
=
\sqrt{20} = 4.47
$$
Ce qui donne 4.47. 

Il existe également un type spécial de vecteur que nous appelons **vecteur unitaire**. **Un vecteur unitaire possède une propriété supplémentaire : sa longueur est exactement égale à 1.** Nous pouvons calculer un vecteur unitaire $\hat{n}$ à partir de n'importe quel vecteur en divisant chacune des composantes du vecteur par sa longueur :
$$
\hat{n}
= { \vec{v} \over ||\vec{v}|| }
$$
 C'est ce que nous appelons la **normalisation** d'un vecteur. Les vecteurs unitaires sont affichés avec un petit toit (ou un chapeau) au-dessus de leur tête et sont généralement **plus faciles à utiliser, en particulier lorsque nous ne nous intéressons qu'à leur direction** (la direction ne change pas si nous modifions la longueur d'un vecteur). 
### Multiplication entre les vecteurs
La multiplication de deux vecteurs est un cas un peu particulier. La multiplication normale n'est pas vraiment définie sur les vecteurs puisqu'elle n'a pas de signification visuelle, mais nous avons deux cas spécifiques que nous pouvons choisir lors de la multiplication : l'un est le produit scalaire noté $\vec{v} \cdot \vec{k}$ (dot product) et l'autre est le produit vectoriel (cross product) noté $\vec{v} \times \vec{k}$. 

#### Produit scalaire (dot product)
Le produit scalaire de deux vecteurs est égal au produit scalaire de leurs longueurs par le cosinus de l'angle qui les sépare. Si cela vous semble confus, jetez un coup d'œil à sa formule :
$$
\vec{v} \cdot \vec{k} =
||\vec{v}|| \cdot ||\vec{k}|| \cdot \cos{\theta}
$$
L'angle qui les sépare est représenté par la valeur thêta ($\theta$). Pourquoi est-ce intéressant ? Imaginons que $\vec{v}$ et $\vec{k}$ soient des vecteurs unitaires, leur longueur serait alors égale à 1, ce qui réduirait la formule à :
$$
\hat{v} \cdot \hat{k} =
1 \cdot 1 \cdot \cos{\theta} = \cos{\theta}
$$
Le produit scalaire ne définit plus que l'angle entre les deux vecteurs. Vous vous souvenez peut-être que la fonction cosinus ou cos devient 0 lorsque l'angle est de 90 degrés ou 1 lorsque l'angle est de 0. **Cela nous permet de tester facilement si les deux vecteurs sont orthogonaux ou parallèles l'un à l'autre à l'aide du produit scalaire** (orthogonal signifie que les vecteurs sont à angle droit l'un par rapport à l'autre). Si vous souhaitez en savoir plus sur les fonctions sin et cos, je vous conseille les vidéos suivantes de la [Khan Academy](https://www.khanacademy.org/math/trigonometry/basic-trigonometry/basic_trig_ratios/v/basic-trigonometry) sur la trigonométrie de base.

Comment calculer le produit scalaire ? Le produit scalaire est une multiplication par composantes où l'on additionne les résultats. Voici à quoi cela ressemble avec deux vecteurs unitaires (vous pouvez vérifier que leurs longueurs respectives sont exactement égales à 1 :
$$
\begin{pmatrix}
0.6\\
-0.8\\
0
\end{pmatrix}
\cdot
\begin{pmatrix}
0\\
1\\
0
\end{pmatrix}
=
(0.6 * 0) + (-0.8 * 1) + (0 * 0) = -0.8
$$
Pour calculer le degré entre ces deux vecteurs unitaires, nous utilisons l'inverse de la fonction cosinus $\cos{}^{-1}$, ce qui donne 143,1 degrés. Nous avons donc calculé l'angle entre ces deux vecteurs. Le produit scalaire s'avère très utile pour les calculs d'éclairage ultérieurs. 

#### Produit vectoriel (cross product)
Le produit vectoriel n'est défini que dans l'espace 3D. Il prend deux vecteurs non parallèles en entrée et produit un troisième vecteur qui est orthogonal aux deux vecteurs d'entrée. Si les deux vecteurs d'entrée sont également orthogonaux l'un par rapport à l'autre, un produit vectoriel produira trois vecteurs orthogonaux, ce qui s'avérera utile dans les chapitres suivants. L'image suivante montre à quoi cela ressemble dans l'espace 3D :
![[img/transfo5.png]]
Contrairement aux autres opérations, le produit vectoriel n'est pas vraiment intuitif si l'on ne se plonge pas dans l'algèbre linéaire. Il est donc préférable de mémoriser la formule et de s'en sortir (ou de ne pas le faire, vous vous en sortirez probablement aussi). Vous trouverez ci-dessous le produit vectoriel de deux vecteurs orthogonaux A et B : 
$$
\begin{pmatrix}
A_x\\
A_y\\
A_z
\end{pmatrix}
\times
\begin{pmatrix}
B_x\\
B_y\\
B_z
\end{pmatrix}
=
\begin{pmatrix}
A_y \cdot B_z - A_z \cdot B_y \\
A_z \cdot B_x - A_x \cdot B_z \\
A_x \cdot B_y - A_y \cdot B_x
\end{pmatrix}
$$
Comme vous pouvez le constater, cela n'a pas vraiment de sens. Cependant, si vous suivez ces étapes, vous obtiendrez un autre vecteur orthogonal à vos vecteurs d'entrée. 

## Matrices
Maintenant que nous avons abordé la quasi-totalité des vecteurs, il est temps d'entrer dans la matrice ! Une matrice est un tableau rectangulaire de nombres, de symboles et/ou d'expressions mathématiques. Chaque élément d'une matrice est appelé élément de la matrice. Voici un exemple de matrice de 2 x 3 : 
$$
\begin{bmatrix}
1 & 2 & 3\\
4 & 5 & 6 \\
\end{bmatrix}
$$
Les matrices sont indexées par $(i,j)$ où $i$ est la ligne et  $j$ est la colonne, c'est pourquoi la matrice ci-dessus est appelée une matrice $2\times3$ (3 colonnes et 2 lignes, également connues comme les dimensions de la matrice). C'est l'inverse de ce à quoi vous êtes habitué lorsque vous indexez des graphiques 2D comme $(x,y)$. Pour récupérer la valeur 4, nous devrions l'indexer comme (2,1) (deuxième ligne, première colonne).  
  
Les matrices ne sont rien d'autre que des tableaux rectangulaires d'expressions mathématiques. Elles possèdent un ensemble de propriétés mathématiques très intéressantes et, tout comme les vecteurs, nous pouvons définir plusieurs opérations sur les matrices, à savoir: l'addition, la soustraction et la multiplication.

### Addition et soustraction
L'addition et la soustraction entre deux matrices s'effectuent par élément. Les règles générales applicables sont donc les mêmes que celles que nous connaissons pour les nombres normaux, mais elles s'appliquent aux éléments des deux matrices ayant le même indice. Cela signifie que l'addition et la soustraction ne sont définies que pour les matrices de même dimension. Une matrice 3x2 et une matrice 2x3 (ou une matrice 3x3 et une matrice 4x4) ne peuvent pas être additionnées ou soustraites ensemble. Voyons comment fonctionne l'addition de matrices sur deux matrices 2x2 :
$$
\begin{bmatrix}
1 & 2\\
3 & 4
\end{bmatrix}
+
\begin{bmatrix}
5 & 6\\
7 & 8
\end{bmatrix}
=
\begin{bmatrix}
1+5 & 2+6\\
3+7 & 4+8
\end{bmatrix}
=
\begin{bmatrix}
6 & 8\\
10 & 12
\end{bmatrix}
$$
Les mêmes règles s'appliquent à la soustraction de la matrice : 
$$
\begin{bmatrix}
4 & 2\\
1 & 6
\end{bmatrix}
-
\begin{bmatrix}
2 & 4\\
0 & 1
\end{bmatrix}
=
\begin{bmatrix}
4-2 & 2-4\\
1-0 & 6-1
\end{bmatrix}
=
\begin{bmatrix}
2 & -2\\
1 & 5
\end{bmatrix}
$$
### Produit scalaire de matrices
Un produit matrice-scalaire multiplie chaque élément de la matrice par un scalaire. L'exemple suivant illustre la multiplication : 

$$
2
\cdot
\begin{bmatrix}
1 & 2\\
3 & 4
\end{bmatrix}
=
\begin{bmatrix}
2\cdot1 & 2\cdot2\\
2\cdot3 & 2\cdot4
\end{bmatrix}
=
\begin{bmatrix}
2 & 4\\
6 & 8
\end{bmatrix}

$$
On comprend maintenant pourquoi ces nombres uniques sont appelés scalaires. **Un scalaire met à l'échelle tous les éléments de la matrice en fonction de sa valeur**. Dans l'exemple précédent, tous les éléments ont été mis à l'échelle par 2.

Jusqu'à présent, tout va bien, tous nos cas ne sont pas trop compliqués. Enfin, jusqu'à ce que nous nous attaquions à la multiplication matrice-matrice. 

### Multiplication entre matrices
La multiplication des matrices n'est pas nécessairement complexe, mais il est plutôt difficile de s'y habituer. La multiplication matricielle consiste essentiellement à suivre un ensemble de règles prédéfinies lors de la multiplication. Il existe cependant quelques restrictions :  
  
1. **Vous ne pouvez multiplier deux matrices que si le nombre de colonnes de la matrice de gauche est égal au nombre de lignes de la matrice de droite.**  
2. La multiplication des matrices n'est pas commutative, c'est-à-dire $A⋅B ≠ B⋅A$
  
Commençons par un exemple de multiplication de 2 matrices `2x2` :
$$
\begin{bmatrix}
1 & 2\\
3 & 4
\end{bmatrix}
\cdot
\begin{bmatrix}
5 & 6\\
7 & 8
\end{bmatrix}
=
\begin{bmatrix}
1\cdot5 + 2\cdot7 & 1\cdot6 + 2\cdot8\\
3\cdot5 + 4\cdot7 & 3\cdot6 + 4\cdot8
\end{bmatrix}
=
\begin{bmatrix}
5+14 & 6+16\\
15+28 & 18+32
\end{bmatrix}
=
\begin{bmatrix}
19 & 22\\
43 & 50
\end{bmatrix}
$$
À l'heure actuelle, vous essayez probablement de comprendre ce qui vient de se passer ? La multiplication matricielle est une combinaison de la multiplication normale et de l'addition utilisant les lignes de la matrice gauche avec les colonnes de la matrice droite. Essayons d'en discuter à l'aide de l'image suivante :
![[img/transfo6.png]]
Nous prenons d'abord la ligne supérieure de la matrice de gauche, puis une colonne de la matrice de droite. La ligne et la colonne que nous avons choisies déterminent la valeur de sortie de la matrice `2x2` résultante que nous allons calculer. Si nous prenons la première ligne de la matrice de gauche, la valeur résultante se retrouvera dans la première ligne de la matrice de résultat, puis nous choisissons une colonne et s'il s'agit de la première colonne, la valeur résultante se retrouvera dans la première colonne de la matrice de résultat. C'est exactement le cas du chemin rouge. Pour calculer le résultat en bas à droite, nous prenons la ligne inférieure de la première matrice et la colonne la plus à droite de la deuxième matrice.  
  
Pour calculer la valeur résultante, nous multiplions le premier élément de la ligne et de la colonne en utilisant la multiplication normale, nous faisons de même pour les deuxième éléments, troisième, quatrième, etc. Les résultats des multiplications individuelles sont ensuite additionnés et nous obtenons notre résultat. Il est logique que l'une des conditions requises soit que la taille des colonnes de la matrice de gauche et celle des lignes de la matrice de droite soient égales, sinon nous ne pouvons pas terminer les opérations !  
  
**Le résultat est donc une matrice dont les dimensions sont `(n,m)` où `n` est égal au nombre de lignes de la matrice de gauche et `m` est égal aux colonnes de la matrice de droite.**  
  
Ne vous inquiétez pas si vous avez du mal à imaginer les multiplications dans votre tête. Essayez simplement de faire les calculs à la main et revenez à cette page chaque fois que vous rencontrez des difficultés. Avec le temps, la multiplication matricielle deviendra pour vous une seconde nature.  
  
Terminons la discussion sur la multiplication matrice-matrice avec un exemple plus large. Essayez de visualiser le modèle à l'aide des couleurs. À titre d'exercice utile, voyez si vous pouvez trouver votre propre réponse à la multiplication, puis comparez-la avec la matrice résultante (une fois que vous aurez essayé de faire une multiplication matricielle à la main, vous comprendrez rapidement ce que c'est).
![[img/transfo7.png]]

Comme vous pouvez le constater, la multiplication matrice-matrice est un processus assez lourd et très sujet aux erreurs (c'est pourquoi nous laissons généralement les ordinateurs s'en charger) et cela devient rapidement problématique lorsque les matrices deviennent plus grandes. Si vous avez encore soif d'en savoir plus et que vous êtes curieux de découvrir d'autres propriétés mathématiques des matrices, je vous suggère vivement de jeter un coup d'œil à ces vidéos de la [Khan Academy](https://www.khanacademy.org/math/algebra-home/alg-matrices) sur les matrices.  
  
Quoi qu'il en soit, maintenant que nous savons comment multiplier des matrices entre elles, nous pouvons passer aux choses sérieuses.

### Multiplication Matrice-Vecteur
Jusqu'à présent, nous avons eu notre part de vecteurs. Nous les avons utilisés pour représenter des positions, des couleurs et même des coordonnées de texture. Allons un peu plus loin et disons qu'un vecteur est en fait une matrice $N\times1$ où $N$ est le nombre de composantes du vecteur (également connu sous le nom de vecteur à $N$ dimensions). Si vous y réfléchissez bien, c'est tout à fait logique. Les vecteurs sont comme les matrices, un tableau de nombres, mais avec une seule colonne. En quoi cette nouvelle information peut-elle nous être utile ? Eh bien, si nous avons une matrice $M\times N$, nous pouvons multiplier cette matrice avec notre vecteur $N\times 1$, puisque les colonnes de la matrice sont égales au nombre de lignes du vecteur, ce qui définit la multiplication matricielle.  
  
Mais pourquoi se soucier de savoir si l'on peut multiplier des matrices avec un vecteur ? Eh bien, il se trouve qu'il existe de nombreuses transformations 2D/3D intéressantes que l'on peut placer à l'intérieur d'une matrice, et le fait de multiplier cette matrice par un vecteur transforme alors ce vecteur. Au cas où vous seriez encore un peu confus, commençons par quelques exemples et vous comprendrez vite ce que nous voulons dire.
### Matrice identité
**Dans OpenGL, nous travaillons généralement avec des matrices de transformation $4\times 4$** pour plusieurs raisons, l'une d'entre elles étant que la plupart des vecteurs sont de taille 4. La matrice de transformation la plus simple à laquelle nous pouvons penser est la matrice identité. La matrice identité est une matrice $N\times N$ quine contient que des 0 sauf sur sa diagonale. Comme vous le verrez, cette matrice de transformation laisse un vecteur complètement intact :
$$
\begin{bmatrix}
1 & 0 & 0 & 0\\
0 & 1 & 0 & 0\\
0 & 0 & 1 & 0\\
0 & 0 & 0 & 1
\end{bmatrix}
\cdot
\begin{bmatrix}
1\\
2\\
3\\
4
\end{bmatrix}
=
\begin{bmatrix}
1\\
2\\
3\\
4
\end{bmatrix}
$$
Le vecteur est totalement intact. Les règles de multiplication le montrent clairement : le premier élément du résultat est chaque élément individuel de la première ligne de la matrice multiplié par chaque élément du vecteur. Étant donné que tous les éléments de la ligne sont égaux à 0, à l'exception du premier, nous obtenons : $1⋅1+0⋅2+0⋅3+0⋅4=1$ et il en va de même pour les 3 autres éléments du vecteur.

>	Vous vous demandez peut-être à quoi sert une matrice de transformation qui ne se transforme pas ? **La matrice identité est généralement un point de départ pour générer d'autres matrices de transformation et, si nous creusons encore plus loin dans l'algèbre linéaire, une matrice très utile pour prouver des théorèmes et résoudre des équations linéaires.**

### Mise à l'échelle (scaling)
Lorsque nous mettons un vecteur à l'échelle, nous augmentons la longueur de la flèche de la quantité que nous souhaitons mettre à l'échelle, tout en conservant la même direction. Comme nous travaillons en 2 ou 3 dimensions, nous pouvons définir la mise à l'échelle par un vecteur de 2 ou 3 variables de mise à l'échelle, chacune mettant à l'échelle un axe (x, y ou z).  
  
Essayons de mettre à l'échelle le vecteur $\vec{v}=(3,2)$. Nous allons mettre à l'échelle le vecteur le long de l'axe des $x$ par $0.5$, ce qui le rendra deux fois plus étroit ; et nous allons mettre à l'échelle le vecteur par 2 le long de l'axe des $y$, ce qui le rendra deux fois plus haut. Voyons ce que cela donne si nous mettons le vecteur à l'échelle $(0.5,2)$ (le résultat sera le vecteur $\vec{s}$) :
![[img/transfo8.png]]
Gardez à l'esprit qu'OpenGL fonctionne généralement dans l'espace 3D, donc pour ce cas 2D, nous pourrions mettre l'échelle de l'axe $z$ à 1, ce qui le laisserait intact. **L'opération de mise à l'échelle que nous venons d'effectuer est une mise à l'échelle non uniforme, car le facteur d'échelle n'est pas le même pour chaque axe. Si le facteur d'échelle était égal sur tous les axes, on parlerait d'une échelle uniforme.**  
  
Commençons par construire une matrice de transformation qui effectue la mise à l'échelle pour nous. Nous avons vu dans la matrice d'identité que chacun des éléments diagonaux était multiplié par l'élément vectoriel correspondant. Que se passerait-il si nous remplacions les 1 de la matrice d'identité par des 3 ? Dans ce cas, nous multiplierions chacun des éléments du vecteur par une valeur de 3, ce qui aurait pour effet d'augmenter uniformément le vecteur de 3. Si nous représentons les variables d'échelle par $(S1,S2,S3)$ nous pouvons définir une matrice d'échelle sur n'importe quel vecteur $(x,y,z)$ comme suit :
$$
\begin{bmatrix}
S1 & 0 & 0 & 0\\
0 & S2 & 0 & 0\\
0 & 0 & S3 & 0\\
0 & 0 & 0 & 1
\end{bmatrix}
\cdot
\begin{bmatrix}
x\\
y\\
z\\
1
\end{bmatrix}
=
\begin{bmatrix}
x \cdot S1\\
y \cdot S2\\
z \cdot S3\\
1
\end{bmatrix}
$$
**Notez que nous conservons la 4e valeur d'échelle 1. La composante $w$ est utilisée à d'autres fins, comme nous le verrons plus loin.**

### Translation
**La translation consiste à ajouter un autre vecteur au vecteur d'origine pour obtenir un nouveau vecteur avec une position différente, déplaçant ainsi le vecteur sur la base d'un vecteur de translation.** Nous avons déjà abordé la question de l'addition des vecteurs, ce qui ne devrait pas être trop nouveau.  
  
Tout comme la matrice de mise à l'échelle, il existe plusieurs emplacements sur une matrice 4 par 4 que nous pouvons utiliser pour effectuer certaines opérations et, p**our la translation, il s'agit des trois premières valeurs de la quatrième colonne.** Si nous représentons le vecteur de translation par $(T_x,T_y,T_z)$ nous pouvons définir la matrice de translation comme suit :
$$
\begin{bmatrix}
1 & 0 & 0 & T_x\\
0 & 1 & 0 & T_y\\
0 & 0 & 1 & T_z\\
0 & 0 & 0 & 1
\end{bmatrix}
\cdot
\begin{bmatrix}
x\\
y\\
z\\
1
\end{bmatrix}
=
\begin{bmatrix}
x + T_x\\
y + T_y\\
z + T_z\\
1
\end{bmatrix}
$$
Cela fonctionne parce que toutes les valeurs de translation sont multipliées par la colonne $w$ du vecteur et ajoutées aux valeurs originales du vecteur (rappelez-vous les règles de multiplication des matrices). Cela n'aurait pas été possible avec une matrice $3 \times 3$. 

>**Coordonnées homogènes**  
	**La composante $w$ d'un vecteur est également appelée coordonnée homogène**. **Pour obtenir le vecteur 3D à partir d'un vecteur homogène, nous divisons les coordonnées x, y et z par la coordonnée w**. Nous ne le remarquons généralement pas, car la composante $w$ est généralement égale à 1,0. **L'utilisation de coordonnées homogènes présente plusieurs avantages : elle nous permet d'effectuer des translations matricielles sur des vecteurs 3D (sans composante w, nous ne pouvons pas translater les vecteurs) et, dans le chapitre suivant, nous utiliserons la valeur w pour créer une perspective 3D.**  
  
>De plus, **lorsque la coordonnée homogène est égale à 0, le vecteur est spécifiquement appelé vecteur de direction**, car un vecteur dont la coordonnée w est égale à 0 ne peut pas être translaté.

Une matrice de translation permet de déplacer des objets dans n'importe laquelle des trois directions $(x, y, z)$, ce qui en fait une matrice de transformation très utile pour notre boîte à outils de transformation.

### Rotation
Les dernières transformations étaient relativement faciles à comprendre et à visualiser dans l'espace 2D ou 3D, mais **les rotations sont un peu plus délicates**. Si vous voulez savoir exactement comment ces matrices sont construites, je vous recommande de regarder les éléments de rotation des vidéos d'[algèbre linéaire](https://www.khanacademy.org/math/linear-algebra/matrix_transformations) de Khan Academy.  
  
Tout d'abord, définissons ce qu'est une rotation d'un vecteur. **Une rotation en 2D ou 3D est représentée par un angle. Un angle peut être exprimé en degrés ou en radians, un cercle entier ayant 360 degrés ou 2 PI radians. Je préfère expliquer les rotations en utilisant les degrés, car nous y sommes généralement plus habitués.**

>La plupart des fonctions de rotation nécessitent un angle en radians, mais heureusement, **les degrés sont facilement convertis en radians** :
	$angle en degrés = angle en radians * (180 / \pi)$
	$angle en radians = angle en degrés * (\pi / 180)$
	Où $\pi$ est égal (arrondi) à $3,14159265359$. 

La rotation d'un demi-cercle nous fait tourner de $360/2 = 180$ degrés et la rotation d'un cinquième vers la droite nous fait tourner de $360/5 = 72$ degrés vers la droite. Ceci est démontré pour un vecteur 2D de base où $\vec{v}$ est tourné de 72 degrés vers la droite, ou dans le sens des aiguilles d'une montre, à partir de $\vec{k}$ : 
![[img/transfo9.png]]
Les rotations en 3D sont spécifiées par un angle et un axe de rotation. L'angle spécifié fera tourner l'objet le long de l'axe de rotation indiqué. Essayez de visualiser cela en faisant tourner votre tête d'un certain degré tout en regardant continuellement vers le bas un seul axe de rotation. **Lorsque l'on fait tourner des vecteurs 2D dans un monde 3D, par exemple, on fixe l'axe de rotation sur l'axe z** (essayez de visualiser cela).  
  
En utilisant la trigonométrie, il est possible de transformer des vecteurs en vecteurs nouvellement tournés en fonction d'un angle. Cela se fait généralement par une combinaison intelligente des fonctions sinus et cosinus (communément abrégées en $\sin$ et $\cos$). Une discussion sur la manière dont les matrices de rotation sont générées n'entre pas dans le cadre de ce chapitre.  
  
Une matrice de rotation est définie pour chaque axe unitaire dans l'espace 3D où l'angle est représenté par le symbole thêta $\theta$.  
  
**Rotation autour de l'axe X :**
$$
\begin{bmatrix}
1 & 0 & 0 & 0\\
0 & \cos{\theta} & -\sin{\theta} & 0\\
0 & \sin{\theta} & \cos{\theta} & 0\\
0 & 0 & 0 & 1
\end{bmatrix}
\cdot
\begin{bmatrix}
x\\
y\\
z\\
1
\end{bmatrix}
=
\begin{bmatrix}
x\\
\cos{\theta} \cdot y - \sin{\theta} \cdot z\\
\sin{\theta} \cdot y + \sin{\theta} \cdot z\\\\
1
\end{bmatrix}
$$
**Rotation autour de l'axe Y :**
$$
\begin{bmatrix}
\cos{\theta} & 0 & \sin{\theta} & 0\\
0 & 1 & 0 & 0\\
-\sin{\theta} & 0 & \cos{\theta} & 0\\
0 & 0 & 0 & 1
\end{bmatrix}
\cdot
\begin{bmatrix}
x\\
y\\
z\\
1
\end{bmatrix}
=
\begin{bmatrix}

\cos{\theta} \cdot x + \sin{\theta} \cdot z\\
y\\
-\sin{\theta} \cdot x + \cos{\theta} \cdot z\\
1
\end{bmatrix}
$$
Rotation autour de l'axe Z :
$$
\begin{bmatrix}
\cos{\theta} & -\sin{\theta} & 0 & 0\\
\sin{\theta} &  \cos{\theta} & 0 & 0\\
0 & 0 & 1 & 0\\
0 & 0 & 0 & 1
\end{bmatrix}
\cdot
\begin{bmatrix}
x\\
y\\
z\\
1
\end{bmatrix}
=
\begin{bmatrix}
\cos{\theta} \cdot x - \sin{\theta} \cdot y\\
\sin{\theta} \cdot x + \cos{\theta} \cdot y\\
z\\
1
\end{bmatrix}
$$

En utilisant les matrices de rotation, nous pouvons transformer nos vecteurs de position autour de l'un des trois axes unitaires. **Pour tourner autour d'un axe 3D arbitraire, nous pouvons les combiner tous les trois en tournant d'abord autour de l'axe $X$, puis $Y$ et enfin $Z$, par exemple. Cependant, cela introduit rapidement un problème appelé "Gimbal lock".** 
Nous ne discuterons pas des détails, mais **une meilleure solution consiste à effectuer une rotation autour d'un axe unitaire arbitraire, par exemple $(0,662,0,2,0,722)$ (notez qu'il s'agit d'un vecteur unitaire), au lieu de combiner les matrices de rotation**. Une telle matrice (verbeuse) existe et est donnée ci-dessous avec $(R_x,R_y,R_z)$ comme axe de rotation arbitraire :

$$
\begin{bmatrix}
\cos{\theta} + {R_x}^{2}(1 - \cos{\theta}) &
R_xR_y(1 - \cos{\theta}) - R_z\sin{\theta} &
R_xR_z(1 - \cos{\theta}) + R_y\sin{\theta} &
0\\
R_yR_x(1 - \cos{\theta}) + R_z\sin{\theta} &
\cos{\theta} + {R_y}^{2}(1 - \cos{\theta}) &
R_yR_z(1 - \cos{\theta}) - R_x\sin{\theta} &
0\\
R_zR_x(1 - \cos{\theta}) - R_y\sin{\theta} &
R_zR_y(1 - \cos{\theta}) + R_x\sin{\theta} &
\cos{\theta} + {R_z}^{2}(1 - \cos{\theta}) &
0\\
0&0&0&1
\end{bmatrix}
$$
Une discussion mathématique sur la création d'une telle matrice n'entre pas dans le cadre de ce chapitre. Gardez à l'esprit que même cette matrice n'empêche pas complètement le gimbal lock (bien qu'il devienne beaucoup plus difficile à réaliser). Pour éviter réellement les gimbal locks (blocages de cadran), nous devons représenter les rotations à l'aide de **quaternions**, qui sont non seulement **plus sûrs, mais aussi plus faciles à calculer**. Cependant, une discussion sur les quaternions n'entre pas dans le cadre de ce chapitre.

### Combiner les matrices
**La véritable puissance de l'utilisation des matrices pour les transformations réside dans le fait que nous pouvons combiner plusieurs transformations dans une seule matrice grâce à la multiplication matrice-matrice.** Voyons si nous pouvons générer une matrice de transformation qui combine plusieurs transformations. Supposons que nous ayons un vecteur $(x,y,z)$ et que nous voulions le mettre à l'échelle par 2 puis le translater par $(1,2,3)$. Nous avons besoin d'une matrice de translation et d'une matrice de mise à l'échelle pour les étapes requises. La matrice de transformation résultante ressemblerait alors à:
$$
Trans \cdot Scale
=
\begin{bmatrix}
1 & 0 & 0 & 1\\
0 & 1 & 0 & 2\\
0 & 0 & 1 & 3\\
0 & 0 & 0 & 1
\end{bmatrix}
\cdot
\begin{bmatrix}
2 & 0 & 0 & 0\\
0 & 2 & 0 & 0\\
0 & 0 & 2 & 0\\
0 & 0 & 0 & 1
\end{bmatrix}
=
\begin{bmatrix}
2 & 0 & 0 & 1\\
0 & 2 & 0 & 2\\
0 & 0 & 2 & 3\\
0 & 0 & 0 & 1
\end{bmatrix}
$$
**Notez que nous effectuons d'abord une translation et ensuite une transformation d'échelle lorsque nous multiplions des matrices. La multiplication des matrices n'est pas commutative, ce qui signifie que leur ordre est important.** **Lors de la multiplication de matrices, la matrice la plus à droite est d'abord multipliée par le vecteur, vous devez donc lire les multiplications de droite à gauche.** Il est conseillé d'effectuer **d'abord les opérations de mise à l'échelle, puis les rotations et enfin les translations** lors de la combinaison de matrices, car elles peuvent avoir un effet (négatif) l'une sur l'autre. Par exemple, si vous effectuez d'abord une translation, puis une mise à l'échelle, le vecteur de translation sera également mis à l'échelle !
En appliquant la matrice de transformation finale à notre vecteur, on obtient le vecteur suivant :

$$
\begin{bmatrix}
2 & 0 & 0 & 1\\
0 & 2 & 0 & 2\\
0 & 0 & 2 & 3\\
0 & 0 & 0 & 1
\end{bmatrix}
\cdot
\begin{bmatrix}
x\\
y\\
z\\
1
\end{bmatrix}
=
\begin{bmatrix}
2x + 1\\
2y + 2\\
2z + 3\\
1
\end{bmatrix}
$$
Superbe ! Le vecteur est d'abord mis à l'échelle par deux, puis translaté par (1,2,3). 

## En pratique
Maintenant que nous avons expliqué toute la théorie derrière les transformations, il est temps de voir comment nous pouvons utiliser ces connaissances à notre avantage. OpenGL n'intègre aucune forme de connaissance des matrices ou des vecteurs, nous devons donc définir nos propres classes et fonctions mathématiques. Dans ce livre, nous préférons nous abstraire de tous les petits détails mathématiques et utiliser simplement des bibliothèques mathématiques pré-faites. Heureusement, il existe une bibliothèque mathématique facile à utiliser et adaptée à OpenGL, appelée **GLM**.

### GLM
GLM signifie OpenGL Mathematics et est une bibliothèque d'en-tête seulement, ce qui signifie qu'il suffit d'inclure les fichiers d'en-tête appropriés et le tour est joué ; aucune liaison ni compilation n'est nécessaire. GLM peut être téléchargé depuis leur [site web](https://glm.g-truc.net/0.9.8/index.html). Copiez le répertoire racine des fichiers d'en-tête dans votre dossier includes et c'est parti.  
  
La plupart des fonctionnalités de GLM dont nous avons besoin se trouvent dans 3 fichiers d'en-tête que nous inclurons comme suit :
```cpp
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>
#include <glm/gtc/type_ptr.hpp>
```
Voyons si nous pouvons mettre à profit nos connaissances en matière de transformation en translatant un vecteur $(1,0,0)$ par $(1,1,0)$ (notez que nous le définissons comme un `glm::vec4` dont la coordonnée homogène est fixée à $1.0$) : 
```cpp
glm::vec4 vec(1.0f, 0.0f, 0.0f, 1.0f);
glm::mat4 trans = glm::mat4(1.0f);
trans = glm::translate(trans, glm::vec3(1.0f, 1.0f, 0.0f));
vec = trans * vec;
std::cout << vec.x << vec.y << vec.z << std::endl;
```
Nous définissons d'abord un vecteur nommé `vec` en utilisant la classe vectorielle intégrée de GLM. Ensuite, nous définissons une `mat4` et l'initialisons explicitement à la matrice identité en initialisant les diagonales de la matrice à 1,0 ; **si nous ne l'initialisons pas à la matrice identité, la matrice sera une matrice nulle (tous les éléments sont à 0) et toutes les opérations matricielles ultérieures aboutiront également à une matrice nulle.**  
  
L'étape suivante consiste à créer une matrice de transformation en passant notre matrice identité à la fonction `glm::translate`, ainsi qu'un vecteur de translation (la matrice donnée est alors multipliée par une matrice de translation et la matrice résultante est renvoyée).  
Ensuite, nous multiplions notre vecteur par la matrice de transformation et nous renvoyons le résultat. Si nous nous souvenons encore de la manière dont fonctionne la translation des matrices, le vecteur résultant devrait être $(1+1,0+1,0+0)$, ce qui correspond à $(2,1,0)$. Cet extrait de code produit $210$, ce qui signifie que la matrice de transformation a fait son travail.  
  
Faisons quelque chose de plus intéressant et mettons à l'échelle et pivotons l'objet conteneur du chapitre précédent :
```cpp
glm::mat4 trans = glm::mat4(1.0f);
trans = glm::rotate(trans, glm::radians(90.0f), glm::vec3(0.0, 0.0, 1.0));
trans = glm::scale(trans, glm::vec3(0.5, 0.5, 0.5));  
```
Nous commençons par mettre à l'échelle le conteneur de $0.5$ sur chaque axe, puis nous le faisons pivoter de 90 degrés autour de l'axe $Z$. GLM attend ses angles en radians, nous convertissons donc les degrés en radians en utilisant `glm::radians`. Notez que le rectangle texturé est sur le plan $XY$ et que nous voulons donc le faire pivoter autour de l'axe $Z$. Gardez à l'esprit que l'axe autour duquel nous effectuons la rotation doit être un vecteur unitaire, donc assurez-vous de normaliser le vecteur d'abord si vous n'effectuez pas de rotation autour de l'axe $X$, $Y$ ou $Z$. Comme nous transmettons la matrice à chacune des fonctions de GLM, GLM multiplie automatiquement les matrices entre elles, ce qui donne une matrice de transformation qui combine toutes les transformations.  
  
La prochaine grande question est la suivante : **comment transmettre la matrice de transformation aux shaders ?** Nous avons déjà mentionné que GLSL possède également un type `mat4`. **Nous allons donc adapter le vertex shader pour qu'il accepte une variable uniforme mat4 et qu'il multiplie le vecteur de position par la matrice uniforme** :
```cpp
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec2 aTexCoord;

out vec2 TexCoord;
  
uniform mat4 transform;

void main()
{
    gl_Position = transform * vec4(aPos, 1.0f);
    TexCoord = vec2(aTexCoord.x, aTexCoord.y);
} 
```
>GLSL dispose également des types `mat2` et `mat3` qui permettent d'effectuer des opérations de type **swizzling**, tout comme les vecteurs. Toutes les opérations mathématiques mentionnées ci-dessus (comme la multiplication scalaire-matrice, la multiplication matrice-vecteur et la multiplication matrice-matrice) sont autorisées sur ces types de matrice. Lorsque des opérations matricielles spéciales seront utilisées, nous veillerons à expliquer ce qui se passe.

Nous avons ajouté l'uniforme et multiplié le vecteur de position avec la matrice de transformation avant de le passer à `gl_Position`. Notre conteneur devrait maintenant être deux fois plus petit et tourné de 90 degrés (incliné vers la gauche). Nous devons encore passer la matrice de transformation au shader :
```cpp
unsigned int transformLoc = glGetUniformLocation(ourShader.ID, "transform");
glUniformMatrix4fv(transformLoc, 1, GL_FALSE, glm::value_ptr(trans));
```
Nous demandons d'abord l'emplacement de la variable uniforme, puis nous envoyons les données matricielles aux shaders en utilisant `glUniform` avec `Matrix4fv` comme postfixe.
- Le premier argument devrait être familier maintenant, il s'agit de **l'emplacement de l'uniforme**
- Le deuxième argument indique à OpenGL **le nombre de matrices à envoyer**, qui est de 1.
- Le troisième argument nous demande si nous voulons transposer notre matrice, c'est-à-dire intervertir les colonnes et les lignes. Les développeurs d'OpenGL utilisent souvent une disposition interne de la matrice appelée ordre colonne-majeur (column-major) qui est la disposition par défaut de la matrice dans GLM, il n'y a donc pas besoin de transposer les matrices ; nous pouvons garder `GL_FALSE`.
- Le dernier paramètre est **la donnée réelle de la matrice**, mais GLM stocke les données de ses matrices d'une manière qui ne correspond pas toujours aux attentes d'OpenGL, **donc nous convertissons d'abord les données avec la fonction intégrée `value_ptr` de GLM**.  
  
Nous avons créé une matrice de transformation, déclaré un uniforme dans le vertex shader et envoyé la matrice aux shaders où nous transformons les coordonnées de nos vertex. Le résultat devrait ressembler à ceci :
![[img/transfo10.png]]
C'est parfait ! Notre conteneur est en effet incliné vers la gauche et deux fois plus petit, la transformation est donc réussie. Soyons un peu plus funky et voyons si nous pouvons faire pivoter le conteneur dans le temps, et pour le plaisir nous allons aussi repositionner le conteneur en bas à droite de la fenêtre. Pour faire pivoter le conteneur dans le temps, nous devons mettre à jour la matrice de transformation dans la boucle de rendu, car elle doit être mise à jour à chaque image. Nous utilisons la fonction temps de GLFW pour obtenir un angle dans le temps :
```cpp
glm::mat4 trans = glm::mat4(1.0f);
trans = glm::translate(trans, glm::vec3(0.5f, -0.5f, 0.0f));
trans = glm::rotate(trans, (float)glfwGetTime(), glm::vec3(0.0f, 0.0f, 1.0f));
```
Gardez à l'esprit que dans le cas précédent, nous pouvions déclarer la matrice de transformation n'importe où, mais que maintenant nous devons la créer à chaque itération pour mettre à jour la rotation en permanence. Cela signifie que nous devons recréer la matrice de transformation à chaque itération de la boucle de rendu. Habituellement, lors du rendu de scènes, nous avons plusieurs matrices de transformation qui sont recréées avec de nouvelles valeurs à chaque image.  
  
Ici, nous faisons d'abord pivoter le conteneur autour de l'origine $(0,0,0)$ et une fois qu'il a pivoté, nous traduisons sa version pivotée dans le coin inférieur droit de l'écran. **N'oubliez pas que l'ordre de transformation réel doit être lu à l'envers : même si, dans le code, nous effectuons d'abord une translation, puis une rotation, les transformations réelles appliquent d'abord une rotation, puis une translation.** Il est difficile de comprendre toutes ces combinaisons de transformations et la manière dont elles s'appliquent aux objets. Essayez d'expérimenter avec des transformations comme celles-ci et vous comprendrez rapidement.  
  
Si vous avez bien fait les choses, vous devriez obtenir le résultat suivant :
![[video/transfo_video1.mp4]]
Et voilà. Un conteneur translaté qui pivote au fil du temps, le tout grâce à une seule matrice de transformation ! Vous comprenez maintenant pourquoi les matrices sont si puissantes dans le domaine du graphisme. Nous pouvons définir un nombre infini de transformations et les combiner dans une seule matrice que nous pouvons réutiliser aussi souvent que nous le souhaitons. **L'utilisation de transformations comme celle-ci dans le vertex shader nous évite de redéfinir les données des vertex et nous fait également gagner du temps de traitement, puisque nous n'avons pas à renvoyer nos données en permanence (ce qui est assez lent) ; tout ce que nous avons à faire, c'est de mettre à jour l'uniforme de transformation.**  
  
Si vous n'avez pas obtenu le bon résultat ou si vous êtes bloqué à un autre endroit, jetez un coup d'œil au [code source](https://learnopengl.com/code_viewer_gh.php?code=src/1.getting_started/5.1.transformations/transformations.cpp) et à la [classe de shader](https://learnopengl.com/code_viewer_gh.php?code=includes/learnopengl/shader_m.h) mise à jour.  
  
Dans le prochain chapitre, nous verrons comment utiliser les matrices pour définir différents espaces de coordonnées pour nos sommets. Ce sera notre premier pas dans le graphisme 3D ! 

## Pour en savoir plus
- [Essence of Linear Algebra](https://www.youtube.com/playlist?list=PLZHQObOWTQDPD3MizzM2xVFitgF8hE_ab): une grande série de tutoriels vidéo par Grant Sanderson sur les mathématiques sous-jacentes des transformations et de l'algèbre linéaire.


