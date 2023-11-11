# Stencil testing
Une fois que le shader de fragment a traité le fragment, un test de stencil est exécuté qui, tout comme le test de profondeur, a la possibilité d'éliminer des fragments. Après cela, les fragments restants sont transmis au test de profondeur où OpenGL peut éventuellement rejeter encore plus de fragments. Le test de **stencil** (pochoir en français?) est basé sur le contenu d'un autre tampon appelé tampon de stencil que nous sommes autorisés à mettre à jour pendant le rendu pour obtenir des effets intéressants.  
  
Un tampon de stencil contient (généralement) 8 bits par valeur de stencil, ce qui représente un total de 256 valeurs de stencil différentes par pixel. Nous pouvons définir ces valeurs de stencil à notre convenance et nous pouvons écarter ou conserver des fragments lorsqu'un fragment particulier a une certaine valeur de stencil.

> Chaque bibliothèque de fenêtrage doit mettre en place un tampon de stencil pour vous. GLFW le fait automatiquement, nous n'avons donc pas besoin de lui dire d'en créer un, mais d'autres bibliothèques de fenêtrage peuvent ne pas créer de tampon de stencil par défaut, alors vérifiez la documentation de votre bibliothèque. 

 Un exemple simple de tampon de stencil est illustré ci-dessous (les pixels ne sont pas à l'échelle) : 
 ![[stencil_buffer 1.png]]
Le stencil buffer est d'abord vidé avec des zéros, puis un rectangle ouvert de `1` est stocké dans le stencil buffer. Les fragments de la scène ne sont alors rendus (les autres sont rejetés) que là où la valeur du stencil de ce fragment contient un `1`.  
  
Les opérations sur le tampon de stencil nous permettent de fixer le tampon de stencil à des valeurs spécifiques partout où nous rendons des fragments. En changeant le contenu du tampon de stencil pendant le rendu, nous _écrivons_ dans le tampon de stencil. Dans la même image (ou les suivantes), nous pouvons _lire_ ces valeurs pour écarter ou passer certains fragments. Lors de l'utilisation des tampons de stencil, vous pouvez être aussi fou que vous le souhaitez, mais les grandes lignes sont généralement les suivantes :  
  
- Activer l'écriture dans le tampon du stencil.  
- Rendre les objets, en mettant à jour le contenu de la mémoire tampon du stencil.  
- Désactiver l'écriture dans le tampon du stencil.  
- Rendu des (autres) objets, cette fois en écartant certains fragments sur la base du contenu de la mémoire tampon du stencil.  
  
L'utilisation de la mémoire tampon du stencil permet d'écarter certains fragments en fonction des fragments d'autres objets dessinés dans la scène.

Vous pouvez activer le test du stencil en activant `GL_STENCIL_TEST`. À partir de ce moment, tous les appels de rendu influenceront le tampon du stencil d'une manière ou d'une autre. 
```cpp
glEnable(GL_STENCIL_TEST);    
```
Notez que vous devez également effacer le tampon du stencil à chaque itération, tout comme les tampons de couleur et de profondeur : 
```cpp
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT | GL_STENCIL_BUFFER_BIT); 
```
De même, tout comme la fonction `glDepthMask` du test de profondeur, il existe une fonction équivalente pour le tampon du stencil. La fonction `glStencilMask` nous permet de définir un masque de bits qui est *ANDed* avec la valeur du stencil sur le point d'être écrite dans le tampon. Par défaut, ce masque est constitué de tous les 1 qui n'affectent pas la sortie, mais si nous le fixons à `0x00`, toutes les valeurs de stencil écrites dans le tampon se retrouveront sous forme de 0. C'est l'équivalent du masque de profondeur `glDepthMask(GL_FALSE)` du test de profondeur :
```cpp
glStencilMask(0xFF); // each bit is written to the stencil buffer as is
glStencilMask(0x00); // each bit ends up as 0 in the stencil buffer (disabling writes)
```
Dans la plupart des cas, vous n'utiliserez que `0x00` ou `0xFF` comme masque de stencil, mais il est bon de savoir qu'il existe des options permettant de définir des masques de bits personnalisés. 

## Fonctions Stencil

Comme pour les tests de profondeur, nous avons un certain contrôle sur le moment où un test de stencil doit réussir ou échouer et sur la manière dont il doit affecter le tampon de stencil. Il existe au total deux fonctions que nous pouvons utiliser pour configurer les tests de stencil : `glStencilFunc` et `glStencilOp`.  
  
La fonction `glStencilFunc(GLenum func, GLint ref, GLuint mask)` a trois paramètres :  
  
- `func` : définit la fonction de test du stencil qui détermine si un fragment est accepté ou rejeté. Cette fonction de test est appliquée à la valeur de stencil stockée et à la valeur `ref` de `glStencilFunc`. Les options possibles sont : `GL_NEVER, GL_LESS, GL_LEQUAL, GL_GREATER, GL_GEQUAL, GL_EQUAL, GL_NOTEQUAL, GL_ALWAYS`. Leur signification sémantique est similaire à celle des fonctions du tampon de profondeur.  
- `ref` : spécifie la valeur de référence pour le test du stencil. Le contenu de la mémoire tampon du stencil est comparé à cette valeur.  
- `mask` : spécifie un masque qui est `ANDed` avec la valeur de référence et la valeur de stencil stockée avant que le test ne les compare. Initialement, toutes les valeurs sont à `1`.  
  
Ainsi, dans le cas de l'exemple simple du stencil que nous avons montré au début, la fonction serait réglée sur :
```cpp
glStencilFunc(GL_EQUAL, 1, 0xFF)
```
Cela indique à OpenGL qu'à chaque fois que la valeur du stencil d'un fragment est égale (`GL_EQUAL`) à la valeur de référence 1, le fragment passe le test et est dessiné, sinon il est rejeté.  
  
Mais `glStencilFunc` décrit seulement si OpenGL doit accepter ou rejeter les fragments en se basant sur le contenu du tampon de stencil, mais pas comment nous pouvons réellement mettre à jour le tampon. C'est là que `glStencilOp` entre en jeu.  
  
La commande `glStencilOp(GLenum sfail, GLenum dpfail, GLenum dppass)` contient trois options parmi lesquelles nous pouvons spécifier pour chaque option l'action à entreprendre :  
  
`sfail` : action à entreprendre si le test du stencil échoue.  
`dpfail` : action à entreprendre si le test du stencil réussit, mais que le test de profondeur échoue.  
`dppass` : action à entreprendre si le test du stencil et le test de profondeur sont tous deux réussis.  
  
Ensuite, pour chacune des options, vous pouvez entreprendre l'une des actions suivantes :

|ACTION|DESCRIPTION|
|-|-|
|`GL_KEEP`|La valeur du stencil actuellement enregistrée est conservée.|
|`GL_ZERO`|La valeur du stencil est fixée à 0.|
								|`GL_REPLACE`|La valeur du stencil est remplacée par la valeur de référence définie avec glStencilFunc.|
|`GL_INCR`|La valeur du stencil est augmentée de 1 si elle est inférieure à la valeur maximale. |
|`GL_INCR_WRAP`|Identique à GL_INCR, mais le ramène à 0 dès que la valeur maximale est dépassée.|
|`GL_DECR`|La valeur du stencil est diminuée de 1 si elle est supérieure à la valeur minimale.|
|`GL_DECR_WRAP`|Identique à GL_DECR, mais l'enveloppe à la valeur maximale si elle est inférieure à 0.|
|`GL_INVERT`|Inverse dans le sens des bits la valeur actuelle de la mémoire tampon du stencil.|

Par défaut, la fonction `glStencilOp` est réglée sur (`GL_KEEP, GL_KEEP, GL_KEEP`), **de sorte que, quel que soit le résultat des tests, le tampon du stencil conserve ses valeurs**. Le comportement par défaut ne met pas à jour le tampon du stencil, donc si vous voulez écrire dans le tampon du stencil, vous devez spécifier au moins une action différente pour chacune des options.  
  
Ainsi, en utilisant `glStencilFunc` et `glStencilOp`, nous pouvons spécifier précisément quand et comment nous voulons mettre à jour le stencil buffer et quand passer ou rejeter des fragments en fonction de son contenu.

## Object outlining (contour d'objet)
Il est peu probable que vous ayez parfaitement compris le fonctionnement du test du stencil à partir des seules sections précédentes. Nous allons donc vous présenter une fonction particulièrement utile qui peut être mise en œuvre avec le seul test du stencil et qui s'appelle le contour d'objet ou **object outlining**. 
![[stencil_object_outlining.png]]

Le contournement d'objet fait exactement ce qu'il dit faire. Pour chaque objet (ou un seul), nous créons une petite bordure colorée autour des objets (combinés). Cet effet est particulièrement utile lorsque vous souhaitez sélectionner des unités dans un jeu de stratégie, par exemple, et que vous devez montrer à l'utilisateur quelles unités ont été sélectionnées. La routine pour délimiter vos objets est la suivante :  
  
1. Activer l'écriture au stencil.  
2. Mettre l'option stencil à `GL_ALWAYS` avant de dessiner les objets (à dessiner), en mettant à jour le tampon stencil avec des `1` chaque fois que les fragments des objets sont rendus.  
3. Effectuer le rendu des objets.  
4. Désactiver l'écriture du stencil et le test de profondeur.  
5. Modifiez légèrement l'échelle de chacun des objets.  
6. Utiliser un shader de fragment différent qui produit une seule couleur (de bordure).  
7. Dessiner à nouveau les objets, mais seulement si les valeurs de stencil de leurs fragments ne sont pas égales à `1`.  
8. Activer à nouveau le test de profondeur et restaurer la fonction stencil à `GL_KEEP`.  
  
Ce processus définit le contenu du tampon de stencil à `1` pour chacun des fragments de l'objet et lorsqu'il est temps de dessiner les bordures, nous dessinons des versions agrandies des objets seulement là où le test de stencil est réussi. Nous rejetons effectivement tous les fragments des versions agrandies qui font partie des fragments des objets originaux en utilisant le tampon du stencil.  
  
Nous allons donc commencer par créer un fragment shader très basique qui produit une couleur de bordure. Nous définissons simplement une valeur de couleur codée en dur et appelons le shader `shaderSingleColor` :

```cpp
void main()
{
    FragColor = vec4(0.04, 0.28, 0.26, 1.0);
}
```

En utilisant la scène du chapitre précédent, nous allons ajouter un contour d'objet aux deux conteneurs, nous laisserons donc le sol en dehors de cela. Nous voulons d'abord dessiner le sol, puis les deux conteneurs (tout en écrivant dans le tampon du stencil), et enfin dessiner les conteneurs agrandis (tout en jetant les fragments qui écrivent sur les fragments de conteneurs précédemment dessinés).  
  
Nous devons d'abord activer le test des stencil :
```cpp
glEnable(GL_STENCIL_TEST);
```
 Ensuite, pour chaque image, nous voulons spécifier l'action à entreprendre lorsque l'un des tests du stencil réussit ou échoue : 
```cpp
glStencilOp(GL_KEEP, GL_KEEP, GL_REPLACE);  
```
Si l'un des tests échoue, nous ne faisons rien ; nous conservons simplement la valeur actuellement stockée dans le tampon du stencil. Cependant, si le test du stencil et le test de profondeur réussissent, nous voulons remplacer la valeur du stencil stockée par la valeur de référence définie par `glStencilFunc`, que nous fixons ensuite à 1.  
  
Au début de l'image, la mémoire tampon du stencil est vidée et, pour les conteneurs, la mémoire tampon du stencil est mise à jour à 1 pour chaque fragment dessiné :

```cpp
glStencilOp(GL_KEEP, GL_KEEP, GL_REPLACE);  
glStencilFunc(GL_ALWAYS, 1, 0xFF); // all fragments should pass the stencil test
glStencilMask(0xFF); // enable writing to the stencil buffer
normalShader.use();
DrawTwoContainers();
```
En utilisant `GL_REPLACE` comme `op_function` du stencil, nous nous assurons que chacun des fragments des conteneurs met à jour le tampon du stencil avec une valeur de stencil de 1. Comme les fragments passent toujours le test du stencil, le tampon du stencil est mis à jour avec la valeur de référence partout où nous les avons dessinés.  
  
Maintenant que le stencil buffer est mis à jour avec des 1 là où les conteneurs ont été dessinés, nous allons dessiner les conteneurs mis à l'échelle, mais cette fois avec la fonction de test appropriée et en désactivant les écritures dans le stencil buffer :

```cpp
glStencilFunc(GL_NOTEQUAL, 1, 0xFF);
glStencilMask(0x00); // disable writing to the stencil buffer
glDisable(GL_DEPTH_TEST);
shaderSingleColor.use(); 
DrawTwoScaledUpContainers();
```
Nous définissons la fonction de stencil à `GL_NOTEQUAL` pour nous assurer que nous ne dessinons que les parties des conteneurs qui ne sont pas égales à 1. De cette façon, nous ne dessinons que la partie des conteneurs qui se trouve à l'extérieur des conteneurs précédemment dessinés. Notez que nous désactivons également le test de profondeur afin que les conteneurs mis à l'échelle (par exemple les bordures) ne soient pas écrasés par le sol. Veillez à réactiver le tampon de profondeur une fois que vous avez terminé.  
  
La routine de contour d'objets pour notre scène ressemble à ceci :
```cpp
glEnable(GL_DEPTH_TEST);
glStencilOp(GL_KEEP, GL_KEEP, GL_REPLACE);  
  
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT | GL_STENCIL_BUFFER_BIT); 

glStencilMask(0x00); // make sure we don't update the stencil buffer while drawing the floor
normalShader.use();
DrawFloor()  
  
glStencilFunc(GL_ALWAYS, 1, 0xFF); 
glStencilMask(0xFF); 
DrawTwoContainers();
  
glStencilFunc(GL_NOTEQUAL, 1, 0xFF);
glStencilMask(0x00); 
glDisable(GL_DEPTH_TEST);
shaderSingleColor.use(); 
DrawTwoScaledUpContainers();
glStencilMask(0xFF);
glStencilFunc(GL_ALWAYS, 1, 0xFF);   
glEnable(GL_DEPTH_TEST);  
```
Tant que vous comprenez l'idée générale des tests de stencils, cela ne devrait pas être trop difficile à comprendre. Sinon, relisez attentivement les sections précédentes et essayez de comprendre complètement ce que fait chacune des fonctions, maintenant que vous avez vu un exemple de leur utilisation.

Le résultat de l'algorithme de contour ressemble alors à ceci : 

![[stencil_scene_outlined 1.png]]
Consultez le code source [ici](https://learnopengl.com/code_viewer_gh.php?code=src/4.advanced_opengl/2.stencil_testing/stencil_testing.cpp) pour voir le code complet de l'algorithme de contour d'objet. 

Vous pouvez voir que les contours se chevauchent entre les deux conteneurs, ce qui est généralement l'effet recherché (pensez aux jeux de stratégie dans lesquels nous voulons sélectionner 10 unités ; la fusion des contours est généralement préférée). Si vous voulez un contour complet par objet, vous devez effacer le tampon de stencil par objet et faire preuve d'un peu de créativité avec le tampon de profondeur.
