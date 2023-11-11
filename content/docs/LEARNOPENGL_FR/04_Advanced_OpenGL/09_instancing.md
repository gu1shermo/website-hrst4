# Instanciation
Supposons que vous ayez une scène dans laquelle vous dessinez un grand nombre de modèles dont la plupart contiennent le même ensemble de données de vertex, mais avec des transformations du monde différentes. Imaginez une scène remplie de feuilles d'herbe : chaque feuille d'herbe est un petit modèle composé de quelques triangles seulement. Vous voudrez probablement en dessiner un certain nombre et votre scène pourra se retrouver avec des milliers, voire des dizaines de milliers de feuilles d'herbe que vous devrez rendre à chaque image. Comme chaque feuille ne comporte que quelques triangles, le rendu de la feuille est presque instantané. Cependant, les milliers d'appels de rendu que vous devrez effectuer réduiront considérablement les performances.

Si nous devions rendre une telle quantité d'objets, cela ressemblerait un peu à cela dans le code :
```cpp
for(unsigned int i = 0; i < amount_of_models_to_draw; i++)
{
    DoSomePreparations(); // bind VAO, bind textures, set uniforms etc.
    glDrawArrays(GL_TRIANGLES, 0, amount_of_vertices);
}
```
Lorsque vous dessinez de nombreuses instances de votre modèle comme cela, vous atteindrez rapidement un goulot d'étranglement au niveau des performances à cause des nombreux appels de dessin. Comparé au rendu des sommets, dire au GPU de rendre vos données de sommets avec des fonctions comme `glDrawArrays` ou `glDrawElements` consomme pas mal de performance puisque OpenGL doit faire les préparations nécessaires avant de pouvoir dessiner vos données de sommets (comme dire au GPU quel tampon lire les données, où trouver les attributs de sommets et tout cela sur le bus CPU-GPU relativement lent). Ainsi, même si le rendu de vos sommets est très rapide, donner à votre GPU les commandes pour les rendre ne l'est pas.

**Il serait beaucoup plus pratique de pouvoir envoyer des données au GPU une seule fois, puis de dire à OpenGL de dessiner plusieurs objets en utilisant ces données avec un seul appel de dessin. C'est là qu'intervient l'instanciation.**

**L'instanciation est une technique qui permet de dessiner plusieurs objets (à données de maillage égales) en une seule fois avec un seul appel de rendu, nous épargnant ainsi toutes les communications CPU -> GPU à chaque fois que nous avons besoin de rendre un objet**. **Pour effectuer le rendu en utilisant l'instanciation, il suffit de changer les appels de rendu `glDrawArrays` et `glDrawElements` en `glDrawArraysInstanced` et `glDrawElementsInstanced` respectivement**. Ces versions instanciées des fonctions de rendu classiques prennent un paramètre supplémentaire appelé le nombre d'instances qui définit le nombre d'instances que nous voulons rendre. Nous envoyons toutes les données requises au GPU une seule fois, puis nous lui indiquons comment dessiner toutes ces instances en un seul appel. Le GPU rend alors toutes ces instances sans avoir à communiquer continuellement avec le CPU.

En soi, cette fonction est un peu inutile. Rendre le même objet un millier de fois ne nous sert à rien puisque chacun des objets rendus est rendu exactement de la même manière et donc au même endroit ; nous ne verrions qu'un seul objet ! C'est pourquoi GLSL a ajouté une autre variable intégrée dans le vertex shader, appelée `gl_InstanceID`.

Lorsque l'on dessine avec l'un des appels de rendu instancié, `gl_InstanceID` est incrémenté pour chaque instance rendue en partant de 0. Si nous devions rendre la 43e instance par exemple, `gl_InstanceID` aurait la valeur 42 dans le vertex shader. Le fait d'avoir une valeur unique par instance signifie que nous pouvons maintenant, par exemple, indexer un grand tableau de valeurs de position pour positionner chaque instance à un endroit différent dans le monde.

Pour vous familiariser avec le dessin instancié, nous allons vous présenter un exemple simple qui rend une centaine de quads 2D en coordonnées normalisées avec un seul appel de rendu. Pour ce faire, nous positionnons chaque carré instancié de manière unique en indexant un tableau uniforme de 100 vecteurs de décalage. Le résultat est une grille bien organisée de quads qui remplissent toute la fenêtre :
![[instancing_quads 1.png]]
Chaque quad est composé de 2 triangles avec un total de 6 sommets. Chaque sommet contient un vecteur de position NDC 2D et un vecteur de couleur. Voici les données de vertex utilisées pour cet exemple - les triangles sont suffisamment petits pour s'adapter à l'écran lorsqu'il y en a une centaine :

```cpp
float quadVertices[] = {
    // positions     // colors
    -0.05f,  0.05f,  1.0f, 0.0f, 0.0f,
     0.05f, -0.05f,  0.0f, 1.0f, 0.0f,
    -0.05f, -0.05f,  0.0f, 0.0f, 1.0f,

    -0.05f,  0.05f,  1.0f, 0.0f, 0.0f,
     0.05f, -0.05f,  0.0f, 1.0f, 0.0f,   
     0.05f,  0.05f,  0.0f, 1.0f, 1.0f		    		
}; 
```
Les quads sont colorés dans le shader de fragment qui reçoit un vecteur de couleur du shader de sommet et le définit comme sa sortie :
```cpp
#version 330 core
out vec4 FragColor;
  
in vec3 fColor;

void main()
{
    FragColor = vec4(fColor, 1.0);
}
```

Rien de nouveau jusqu'à présent, mais au niveau du vertex shader, cela commence à devenir intéressant :
```cpp
#version 330 core
layout (location = 0) in vec2 aPos;
layout (location = 1) in vec3 aColor;

out vec3 fColor;

uniform vec2 offsets[100];

void main()
{
    vec2 offset = offsets[gl_InstanceID];
    gl_Position = vec4(aPos + offset, 0.0, 1.0);
    fColor = aColor;
}  
```
Ici, nous avons défini un tableau uniforme appelé `offsets` qui contient un total de 100 vecteurs de décalage. Dans le vertex shader, nous récupérons un vecteur de décalage pour chaque instance en indexant le tableau offsets à l'aide de gl_InstanceID. Si nous devions maintenant dessiner 100 quads avec le dessin instancié, nous obtiendrions 100 quads situés à des positions différentes.

Nous devons définir les positions de décalage que nous calculons dans une boucle for imbriquée avant d'entrer dans la boucle de rendu :

```cpp
glm::vec2 translations[100];
int index = 0;
float offset = 0.1f;
for(int y = -10; y < 10; y += 2)
{
    for(int x = -10; x < 10; x += 2)
    {
        glm::vec2 translation;
        translation.x = (float)x / 10.0f + offset;
        translation.y = (float)y / 10.0f + offset;
        translations[index++] = translation;
    }
}  
```
Ici, nous créons un ensemble de 100 vecteurs de translation qui contient un vecteur de décalage pour toutes les positions dans une grille de 10x10. En plus de générer le tableau de translations, nous devons également transférer les données vers le tableau d'uniformes du vertex shader :
```cpp
shader.use();
for(unsigned int i = 0; i < 100; i++)
{
    shader.setVec2(("offsets[" + std::to_string(i) + "]")), translations[i]);
}  
```
Dans cet extrait de code, nous transformons le compteur i de la boucle for en une chaîne de caractères afin de créer dynamiquement une chaîne d'emplacement pour interroger l'emplacement uniforme. Pour chaque élément du tableau uniforme offsets, nous définissons le vecteur de translation correspondant.

Maintenant que toutes les préparations sont terminées, nous pouvons commencer le rendu des quads. Pour dessiner via un rendu instancié, nous appelons `glDrawArraysInstanced` ou `glDrawElementsInstanced`. Comme nous n'utilisons pas de tampon d'index d'élément, nous allons appeler la version `glDrawArrays` :
```cpp
glBindVertexArray(quadVAO);
glDrawArraysInstanced(GL_TRIANGLES, 0, 6, 100);  
```
Les paramètres de `glDrawArraysInstanced` sont exactement les mêmes que ceux de `glDrawArrays`, à l'exception du dernier paramètre qui définit le nombre d'instances à dessiner. Comme nous voulons afficher 100 quads dans une grille de 10x10, nous le fixons à 100. L'exécution du code devrait maintenant vous donner l'image familière de 100 quads colorés.

## Tableaux instanciés
Bien que l'implémentation précédente fonctionne bien pour ce cas d'utilisation spécifique, lorsque nous rendons beaucoup plus de 100 instances (ce qui est assez courant), nous finirons par atteindre une limite sur la quantité de données uniformes que nous pouvons envoyer aux shaders. Une option alternative est connue sous le nom de tableaux instanciés. Les tableaux instanciés sont définis comme un attribut de sommet (ce qui nous permet de stocker beaucoup plus de données) qui est mis à jour par instance au lieu d'être mis à jour par sommet.

Avec les attributs de sommet, au début de chaque exécution du shader de sommet, le GPU récupère le prochain ensemble d'attributs de sommet appartenant au sommet actuel. Cependant, lorsqu'un attribut de sommet est défini comme un tableau instancié, le shader de sommet ne met à jour le contenu de l'attribut de sommet que pour chaque instance. Cela nous permet d'utiliser les attributs de vertex standard pour les données par vertex et d'utiliser le tableau instancié pour stocker des données uniques par instance.

Pour vous donner un exemple de tableau instancié, nous allons reprendre l'exemple précédent et convertir le tableau uniforme offset en tableau instancié. Nous devrons mettre à jour le shader de vertex en ajoutant un autre attribut de vertex :
```cpp
#version 330 core
layout (location = 0) in vec2 aPos;
layout (location = 1) in vec3 aColor;
layout (location = 2) in vec2 aOffset;

out vec3 fColor;

void main()
{
    gl_Position = vec4(aPos + aOffset, 0.0, 1.0);
    fColor = aColor;
}  
```
Nous n'utilisons plus `gl_InstanceID` et pouvons utiliser directement l'attribut `offset` sans indexer au préalable un grand tableau uniforme.

Parce qu'un tableau instancié est un attribut de vertex, tout comme les variables de position et de couleur, nous devons stocker son contenu dans un objet tampon de vertex et configurer son pointeur d'attribut. Nous allons d'abord stocker le tableau de translations (de la section précédente) dans un nouvel objet tampon :
```cpp
unsigned int instanceVBO;
glGenBuffers(1, &instanceVBO);
glBindBuffer(GL_ARRAY_BUFFER, instanceVBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(glm::vec2) * 100, &translations[0], GL_STATIC_DRAW);
glBindBuffer(GL_ARRAY_BUFFER, 0); 
```

Ensuite, nous devons également définir son pointeur d'attribut de sommet et activer l'attribut de sommet :
```cpp
glEnableVertexAttribArray(2);
glBindBuffer(GL_ARRAY_BUFFER, instanceVBO);
glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, 2 * sizeof(float), (void*)0);
glBindBuffer(GL_ARRAY_BUFFER, 0);	
glVertexAttribDivisor(2, 1);  
```
Ce qui rend ce code intéressant est la dernière ligne où nous appelons `glVertexAttribDivisor`. **Cette fonction indique à OpenGL quand mettre à jour le contenu d'un attribut de vertex vers l'élément suivant**.
Son premier paramètre est l'attribut de sommet en question et le second le diviseur d'attribut. Par défaut, le diviseur d'attribut est 0, ce qui indique à OpenGL de mettre à jour le contenu de l'attribut de sommet à chaque itération du shader de sommet. En mettant cet attribut à 1, nous indiquons à OpenGL que nous voulons mettre à jour le contenu de l'attribut vertex lorsque nous commençons à effectuer le rendu d'une nouvelle instance. En lui donnant la valeur 2, nous mettrons à jour le contenu toutes les 2 instances, et ainsi de suite. En fixant le diviseur d'attribut à 1, nous indiquons effectivement à OpenGL que l'attribut de sommet à l'emplacement d'attribut 2 est un tableau instancié.

Si nous rendions à nouveau les quads avec `glDrawArraysInstanced`, nous obtiendrions la sortie suivante :
![[instancing_quads 1.png]]
C'est exactement la même chose que dans l'exemple précédent, mais maintenant avec des tableaux instanciés, ce qui nous permet de passer beaucoup plus de données (autant que la mémoire nous le permet) au vertex shader pour le dessin instancié.

Pour le plaisir, nous pourrions lentement réduire l'échelle de chaque quadrant du haut à droite au bas à gauche en utilisant à nouveau `gl_InstanceID`, parce que pourquoi pas ?
```cpp
void main()
{
    vec2 pos = aPos * (gl_InstanceID / 100.0);
    gl_Position = vec4(pos + aOffset, 0.0, 1.0);
    fColor = aColor;
} 
```
Le résultat est que les premières instances des quads sont dessinées extrêmement petites et plus nous avançons dans le processus de dessin des instances, plus `gl_InstanceID` se rapproche de 100 et donc plus les quads retrouvent leur taille d'origine. Il est parfaitement légal d'utiliser des tableaux instanciés avec `gl_InstanceID` de cette manière.

Si vous n'êtes toujours pas sûr du fonctionnement du rendu instancié ou si vous voulez voir comment tout s'articule, vous pouvez trouver le code source complet de l'application [ici](https://learnopengl.com/code_viewer_gh.php?code=src/4.advanced_opengl/10.1.instancing_quads/instancing_quads.cpp).

Bien qu'amusants, ces exemples ne sont pas vraiment de bons exemples d'instanciation. Certes, ils vous donnent un aperçu facile du fonctionnement de l'instanciation, **mais l'instanciation tire le meilleur parti de sa puissance lorsqu'elle dessine une énorme quantité d'objets similaires**. C'est pourquoi nous allons nous aventurer dans l'espace.

# Un champ d'astéroïdes
Imaginez une scène où une grande planète se trouve au centre d'un grand anneau d'astéroïdes. Un tel anneau d'astéroïdes peut contenir des milliers ou des dizaines de milliers de formations rocheuses et devient rapidement impossible à rendre sur une carte graphique décente. Ce scénario s'avère particulièrement utile pour le rendu instancié, puisque tous les astéroïdes peuvent être représentés avec un seul modèle. Chaque astéroïde obtient alors sa variation à partir d'une matrice de transformation unique à chaque astéroïde.

Pour démontrer l'impact du rendu instancié, nous allons d'abord effectuer le rendu d'une scène d'astéroïdes en vol stationnaire autour d'une planète sans rendu instancié. La scène contiendra un grand modèle de planète qui peut être téléchargé [ici](https://learnopengl.com/data/models/planet.zip) et un grand ensemble d'astéroïdes que nous positionnerons correctement autour de la planète. Le modèle des astéroïdes peut être téléchargé [ici](https://learnopengl.com/data/models/rock.zip).

Dans les exemples de code, nous chargeons les modèles à l'aide du chargeur de modèle que nous avons défini précédemment dans les chapitres consacrés au chargement des modèles.

Pour obtenir l'effet recherché, nous allons générer une matrice de transformation du modèle pour chaque astéroïde. La matrice de transformation traduit d'abord le rocher quelque part dans l'anneau d'astéroïde - puis nous ajoutons une petite valeur de déplacement aléatoire au décalage pour que l'anneau ait l'air plus naturel. À partir de là, nous appliquons également une échelle et une rotation aléatoires. Le résultat est une matrice de transformation qui place chaque astéroïde quelque part autour de la planète tout en lui donnant un aspect plus naturel et unique par rapport aux autres astéroïdes.
```cpp
unsigned int amount = 1000;
glm::mat4 *modelMatrices;
modelMatrices = new glm::mat4[amount];
srand(glfwGetTime()); // initialize random seed	
float radius = 50.0;
float offset = 2.5f;
for(unsigned int i = 0; i < amount; i++)
{
    glm::mat4 model = glm::mat4(1.0f);
    // 1. translation: displace along circle with 'radius' in range [-offset, offset]
    float angle = (float)i / (float)amount * 360.0f;
    float displacement = (rand() % (int)(2 * offset * 100)) / 100.0f - offset;
    float x = sin(angle) * radius + displacement;
    displacement = (rand() % (int)(2 * offset * 100)) / 100.0f - offset;
    float y = displacement * 0.4f; // keep height of field smaller compared to width of x and z
    displacement = (rand() % (int)(2 * offset * 100)) / 100.0f - offset;
    float z = cos(angle) * radius + displacement;
    model = glm::translate(model, glm::vec3(x, y, z));

    // 2. scale: scale between 0.05 and 0.25f
    float scale = (rand() % 20) / 100.0f + 0.05;
    model = glm::scale(model, glm::vec3(scale));

    // 3. rotation: add random rotation around a (semi)randomly picked rotation axis vector
    float rotAngle = (rand() % 360);
    model = glm::rotate(model, rotAngle, glm::vec3(0.4f, 0.6f, 0.8f));

    // 4. now add to list of matrices
    modelMatrices[i] = model;
}  
```
Ce morceau de code peut sembler un peu intimidant, mais nous transformons essentiellement les positions x et z de l'astéroïde le long d'un cercle dont le rayon est défini par le rayon et nous déplaçons aléatoirement chaque astéroïde un peu autour du cercle par -offset et offset. Le déplacement en y a moins d'impact pour créer un anneau d'astéroïdes plus plat. Nous appliquons ensuite des transformations d'échelle et de rotation et stockons la matrice de transformation résultante dans `modelMatrices` qui est de la taille de la quantité. Ici, nous générons 1000 matrices de modèle, une par astéroïde.

Après avoir chargé les modèles de planètes et de roches et compilé un ensemble de shaders, le code de rendu ressemble à ceci :
```cpp
// draw planet
shader.use();
glm::mat4 model = glm::mat4(1.0f);
model = glm::translate(model, glm::vec3(0.0f, -3.0f, 0.0f));
model = glm::scale(model, glm::vec3(4.0f, 4.0f, 4.0f));
shader.setMat4("model", model);
planet.Draw(shader);
  
// draw meteorites
for(unsigned int i = 0; i < amount; i++)
{
    shader.setMat4("model", modelMatrices[i]);
    rock.Draw(shader);
}  
```
Nous dessinons d'abord le modèle de la planète, que nous traduisons et mettons à l'échelle pour l'adapter à la scène, puis nous dessinons un nombre de modèles de rochers égal à la quantité de transformations que nous avons générées précédemment. Cependant, avant de dessiner chaque roche, nous définissons d'abord la matrice de transformation du modèle correspondant dans le shader.

Le résultat est une scène spatiale où l'on peut voir un anneau d'astéroïdes d'apparence naturelle autour d'une planète :
![[instancing_asteroids.png]]
Cette scène contient un total de 1001 appels de rendu par image, dont 1000 pour le modèle de la roche. Vous pouvez trouver le code source de cette scène [ici](https://learnopengl.com/code_viewer_gh.php?code=src/4.advanced_opengl/10.2.asteroids/asteroids.cpp).

Dès que nous commençons à augmenter ce nombre, nous remarquons rapidement que la scène cesse de fonctionner de manière fluide et que le nombre d'images que nous sommes capables de rendre par seconde diminue de manière drastique. Dès que nous fixons la valeur à près de 2000, la scène devient si lente sur notre GPU qu'il devient difficile de se déplacer.

Essayons maintenant de rendre la même scène, mais cette fois avec un rendu instancié. Nous devons d'abord ajuster un peu le vertex shader :
```cpp
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 2) in vec2 aTexCoords;
layout (location = 3) in mat4 instanceMatrix;

out vec2 TexCoords;

uniform mat4 projection;
uniform mat4 view;

void main()
{
    gl_Position = projection * view * instanceMatrix * vec4(aPos, 1.0); 
    TexCoords = aTexCoords;
}
```
Nous n'utilisons plus de variable uniforme de modèle, mais nous déclarons un `mat4` comme attribut de sommet afin de pouvoir stocker un tableau instancié de matrices de transformation. Cependant, lorsque nous déclarons comme attribut de sommet un type de données supérieur à un `vec4`, les choses se passent un peu différemment. La quantité maximale de données autorisée pour un attribut de sommet est égale à un `vec4`. Étant donné qu'une `mat4` est essentiellement composée de 4 `vec4`, nous devons réserver 4 attributs de sommet pour cette matrice spécifique. Comme nous lui avons attribué un emplacement de 3, les colonnes de la matrice auront des emplacements d'attributs de sommet de 3, 4, 5 et 6.

Nous devons ensuite définir chacun des pointeurs d'attribut de ces 4 attributs de sommet et les configurer en tant que tableaux instanciés :
```cpp
// vertex buffer object
unsigned int buffer;
glGenBuffers(1, &buffer);
glBindBuffer(GL_ARRAY_BUFFER, buffer);
glBufferData(GL_ARRAY_BUFFER, amount * sizeof(glm::mat4), &modelMatrices[0], GL_STATIC_DRAW);
  
for(unsigned int i = 0; i < rock.meshes.size(); i++)
{
    unsigned int VAO = rock.meshes[i].VAO;
    glBindVertexArray(VAO);
    // vertex attributes
    std::size_t vec4Size = sizeof(glm::vec4);
    glEnableVertexAttribArray(3); 
    glVertexAttribPointer(3, 4, GL_FLOAT, GL_FALSE, 4 * vec4Size, (void*)0);
    glEnableVertexAttribArray(4); 
    glVertexAttribPointer(4, 4, GL_FLOAT, GL_FALSE, 4 * vec4Size, (void*)(1 * vec4Size));
    glEnableVertexAttribArray(5); 
    glVertexAttribPointer(5, 4, GL_FLOAT, GL_FALSE, 4 * vec4Size, (void*)(2 * vec4Size));
    glEnableVertexAttribArray(6); 
    glVertexAttribPointer(6, 4, GL_FLOAT, GL_FALSE, 4 * vec4Size, (void*)(3 * vec4Size));

    glVertexAttribDivisor(3, 1);
    glVertexAttribDivisor(4, 1);
    glVertexAttribDivisor(5, 1);
    glVertexAttribDivisor(6, 1);

    glBindVertexArray(0);
}  
```
Notez que nous avons un peu triché en déclarant la variable `VAO` du Mesh comme une variable publique au lieu d'une variable privée afin de pouvoir accéder à son objet vertex array. Ce n'est pas la solution la plus propre, mais il s'agit d'une simple modification pour convenir à cet exemple. Mis à part cette petite modification, ce code devrait être clair. Nous déclarons essentiellement comment OpenGL doit interpréter le tampon pour chaque attribut de sommet de la matrice et que chacun de ces attributs de sommet est un tableau instancié.

Ensuite, nous reprenons le VAO du (des) mesh(es) et cette fois nous dessinons en utilisant `glDrawElementsInstanced` :
```cpp
// draw meteorites
instanceShader.use();
for(unsigned int i = 0; i < rock.meshes.size(); i++)
{
    glBindVertexArray(rock.meshes[i].VAO);
    glDrawElementsInstanced(
        GL_TRIANGLES, rock.meshes[i].indices.size(), GL_UNSIGNED_INT, 0, amount
    );
}  
```
Ici, nous dessinons la même quantité d'astéroïdes que dans l'exemple précédent, mais cette fois avec un rendu instancié. Les résultats devraient être exactement les mêmes, mais une fois que nous aurons augmenté le nombre d'astéroïdes, vous commencerez vraiment à voir la puissance du rendu instancié. Sans le rendu instancié, nous avons pu obtenir un rendu fluide d'environ 1000 à 1500 astéroïdes. Avec le rendu instancié, nous pouvons maintenant fixer cette valeur à 100000. Ceci, avec le modèle de roche ayant 576 vertices, équivaudrait à environ 57 millions de vertices dessinés chaque image sans baisse significative de performance ; et seulement 2 appels de dessin !
![[instancing_asteroids_quantity.png]]
Cette image a été rendue avec 100000 astéroïdes avec un rayon de 150.0f et un décalage égal à 25.0f. Vous pouvez trouver le code source de la démo de rendu instancié [ici](https://learnopengl.com/code_viewer_gh.php?code=src/4.advanced_opengl/10.3.asteroids_instanced/asteroids_instanced.cpp).

>Sur différentes machines, un nombre d'astéroïdes de 100 000 peut être un peu trop élevé, alors essayez d'ajuster les valeurs jusqu'à ce que vous atteigniez un framerate acceptable.

Comme vous pouvez le constater, avec le bon type d'environnement, le rendu instancié peut faire une énorme différence dans les capacités de rendu de votre application. C'est pourquoi le rendu instancié est couramment utilisé pour l'herbe, la flore, les particules et les scènes de ce type - en fait, toute scène comportant de nombreuses formes répétitives peut bénéficier du rendu instancié.