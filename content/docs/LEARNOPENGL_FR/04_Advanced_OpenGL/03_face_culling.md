
# Face culling
Essayez de visualiser mentalement un cube en 3D et comptez le nombre maximum de faces que vous pourrez voir dans n'importe quelle direction. Si votre imagination n'est pas trop créative, vous avez probablement abouti à un nombre maximal de 3. Vous pouvez voir un cube dans n'importe quelle position et/ou direction, mais vous ne pourrez jamais voir plus de 3 faces. Alors pourquoi gaspiller l'effort de dessiner ces 3 autres faces que nous ne pouvons même pas voir. Si nous pouvions les éliminer d'une manière ou d'une autre, nous économiserions plus de 50 % du nombre total d'exécutions du fragment shader de ce cube !

> Nous disons plus de 50 % au lieu de 50 %, car sous certains angles, seuls 2 ou même 1 face peuvent être visibles. Dans ce cas, nous économisons plus de 50 %. 


C'est une très bonne idée, mais il y a un problème à résoudre : comment savoir si une face d'un objet n'est pas visible du point de vue de l'observateur ? Si nous imaginons une forme fermée, chacune de ses faces a deux côtés. Chaque face fait face à l'utilisateur ou lui tourne le dos. Et si nous ne pouvions rendre que les faces qui font face à l'utilisateur ?  
  
**C'est exactement ce que fait le face culling. OpenGL vérifie toutes les faces qui font face à l'observateur et les rend tout en rejetant toutes les faces qui font face à l'arrière, ce qui nous permet d'économiser beaucoup d'appels au fragment shader.** Nous devons indiquer à OpenGL lesquelles des faces que nous utilisons sont les faces avant et lesquelles sont les faces arrière. OpenGL utilise une astuce intelligente pour cela en analysant l'ordre d'enroulement (winding order) des données de vertex.

## Winding order
**Lorsque nous définissons un ensemble de sommets de triangle, nous les définissons dans un certain ordre d'enroulement, soit dans le sens des aiguilles d'une montre, soit dans le sens inverse.** Chaque triangle est composé de 3 sommets et nous spécifions ces 3 sommets dans un ordre d'enroulement vu du centre du triangle. 
![[Pasted image 20230816161422.png]]
Comme vous pouvez le voir sur l'image, nous définissons d'abord le sommet 1 et, à partir de là, nous pouvons choisir si le sommet suivant est 2 ou 3. Ce choix définit l'ordre d'enroulement de ce triangle. Le code suivant l'illustre : 

```cpp
float vertices[] = {
    // clockwise
    vertices[0], // vertex 1
    vertices[1], // vertex 2
    vertices[2], // vertex 3
    // counter-clockwise
    vertices[0], // vertex 1
    vertices[2], // vertex 3
    vertices[1]  // vertex 2  
};
```
Chaque ensemble de 3 sommets qui forment un triangle primitif contient donc un ordre d'enroulement. OpenGL utilise cette information lors du rendu de vos primitives pour déterminer si un triangle est orienté vers l'avant ou vers l'arrière. **Par défaut, les triangles définis avec des sommets dans le sens inverse des aiguilles d'une montre sont traités comme des triangles orientés vers l'avant.**  
  
Lorsque vous définissez l'ordre des sommets, vous visualisez le triangle correspondant comme s'il vous faisait face, de sorte que chaque triangle que vous spécifiez doit être traité dans le sens inverse des aiguilles d'une montre, comme si vous lui faisiez directement face. L'avantage de spécifier tous les sommets de cette manière est que l'ordre d'enroulement réel est calculé lors de l'étape de rastérisation, c'est-à-dire lorsque le shader de sommets a déjà été exécuté. Les sommets sont alors vus du point de vue de l'observateur.  
  
Tous les sommets des triangles auxquels le spectateur fait face sont effectivement dans l'ordre d'enroulement correct tel que nous l'avons spécifié, mais les sommets des triangles situés de l'autre côté du cube sont maintenant rendus de telle sorte que leur ordre d'enroulement est inversé. Le résultat est que les triangles auxquels nous faisons face sont vus comme des triangles orientés vers l'avant et les triangles à l'arrière sont vus comme des triangles orientés vers l'arrière. L'image suivante illustre cet effet :
![[03_face_culling-20230816.png]]
Dans les données relatives aux sommets, nous avons défini les deux triangles dans le sens inverse des aiguilles d'une montre (les triangles avant et arrière étant 1, 2, 3). Cependant, du point de vue de l'observateur, le triangle arrière est rendu dans le sens des aiguilles d'une montre si nous le dessinons dans l'ordre 1, 2 et 3 du point de vue actuel de l'observateur. Même si nous avons spécifié le triangle arrière dans le sens inverse des aiguilles d'une montre, il est maintenant rendu dans le sens des aiguilles d'une montre. C'est exactement ce que nous voulons pour éliminer les faces non visibles !

## Face culling
Au début de ce chapitre, nous avons dit qu'OpenGL est capable de rejeter les primitives triangulaires si elles sont rendues sous forme de triangles orientés vers l'arrière. Maintenant que nous savons comment définir l'ordre d'enroulement des sommets, nous pouvons commencer à utiliser l'option d'élimination des faces d'OpenGL qui est désactivée par défaut.  
  
Les données des sommets du cube que nous avons utilisées dans les chapitres précédents n'ont pas été définies en tenant compte de l'ordre d'enroulement dans le sens inverse des aiguilles d'une montre, j'ai donc mis à jour les données des sommets pour refléter un ordre d'enroulement dans le sens inverse des aiguilles d'une montre, que vous pouvez copier ici. C'est une bonne pratique d'essayer de visualiser que ces sommets sont effectivement tous définis dans l'ordre inverse des aiguilles d'une montre pour chaque triangle.  
  
Pour activer le face culling, il suffit d'activer l'option GL_CULL_FACE d'OpenGL :
```cpp
glEnable(GL_CULL_FACE);  
```
A partir de là, toutes les faces qui ne sont pas des faces avant sont éliminées (essayez de vous déplacer à l'intérieur du cube pour voir que toutes les faces intérieures sont effectivement éliminées). Actuellement, nous économisons plus de 50% de performance sur le rendu des fragments si OpenGL décide de rendre les faces arrière en premier (sinon le test de profondeur les aurait déjà éliminées). **Notez que cela ne fonctionne vraiment qu'avec des formes fermées comme un cube. Nous devons à nouveau désactiver le face culling lorsque nous dessinons les feuilles d'herbe du chapitre précédent, puisque leurs faces avant et arrière doivent être visibles.**  
  
OpenGL nous permet également de changer le type de face que nous voulons éliminer. Que faire si nous voulons éliminer les faces avant et non les faces arrière ? Nous pouvons définir ce comportement avec `glCullFace` :
```cpp
glCullFace(GL_FRONT);  
```

La fonction `glCullFace` a trois options possibles :  
  
- `GL_BACK` : Ne traite que les faces arrière.  
- `GL_FRONT` : ne traite que les faces avant.  
- `GL_FRONT_AND_BACK` : élimine les faces avant et arrière.  
  
La valeur initiale de `glCullFace` est GL_BACK. Nous pouvons également indiquer à OpenGL que nous préférons les faces avant dans le sens des aiguilles d'une montre plutôt que dans le sens inverse des aiguilles d'une montre via `glFrontFace` :
```cpp
glFrontFace(GL_CCW);  
```
La valeur par défaut est `GL_CCW` qui correspond à un ordre dans le sens inverse des aiguilles d'une montre, l'autre option étant `GL_CW` qui correspond (évidemment) à un ordre dans le sens des aiguilles d'une montre.  
  
Pour un simple test, nous pouvons inverser l'ordre d'enroulement en disant à OpenGL que les faces avant sont maintenant déterminées par un ordre dans le sens des aiguilles d'une montre au lieu d'un ordre dans le sens inverse :  
```cpp
glEnable(GL_CULL_FACE);
glCullFace(GL_BACK);
glFrontFace(GL_CW);
```
 Il en résulte que seules les faces arrière sont rendues : 
 ![[faceculling_reverse.png]]
 
Notez que vous pouvez créer le même effet en éliminant les faces avant avec l'ordre d'enroulement par défaut dans le sens inverse des aiguilles d'une montre : 
```cpp
glEnable(GL_CULL_FACE);
glCullFace(GL_FRONT);  
```
Comme vous pouvez le constater, l'élimination des faces est un excellent outil pour augmenter les performances de vos applications OpenGL avec un minimum d'effort, d'autant plus que toutes les applications 3D exportent des modèles avec des ordres d'enroulement cohérents (`CCW` par défaut). Vous devez garder une trace des objets qui bénéficieront réellement de l'élimination des faces et de ceux qui ne devraient pas être éliminés du tout.
