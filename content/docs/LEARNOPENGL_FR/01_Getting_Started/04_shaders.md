---
tags: [cg, opengl, computer graphics, cpp]
dg-publish: true
---
# Shaders
Comme mentionné dans le chapitre [[03_hello triangle]], les shaders sont de petits programmes qui reposent sur le GPU. Ces programmes sont exécutés pour chaque section spécifique du pipeline graphique. **En gros, les shaders ne sont rien d'autre que des programmes qui transforment les entrées en sorties. Les shaders sont également des programmes très isolés dans la mesure où ils ne sont pas autorisés à communiquer entre eux ; la seule communication qu'ils ont se fait par le biais de leurs entrées et sorties.**

Dans le chapitre précédent, nous avons brièvement abordé la question des shaders et la manière de les utiliser correctement. Nous allons maintenant expliquer les shaders, et plus particulièrement le langage shader d'OpenGL, d'une manière plus générale.

## GLSL
Les shaders sont écrits dans le langage de type C GLSL. **GLSL est conçu pour être utilisé avec des graphiques et contient des fonctionnalités utiles spécifiquement destinées à la manipulation de vecteurs et de matrices.**

Les shaders commencent toujours par une déclaration de version, suivie d'une liste de variables d'entrée et de sortie, d'uniformes et de leur fonction principale. Le point d'entrée de chaque shader se situe au niveau de sa fonction principale, où nous traitons toutes les variables d'entrée et affichons les résultats dans les variables de sortie. Ne vous inquiétez pas si vous ne savez pas ce que sont les uniformes, nous y reviendrons bientôt.

Un shader a typiquement la structure suivante :
```cpp
#version version_number
in type in_variable_name;
in type in_variable_name;

out type out_variable_name;
  
uniform type uniform_name;
  
void main()
{
  // process input(s) and do some weird graphics stuff
  ...
  // output processed stuff to output variable
  out_variable_name = weird_stuff_we_processed;
}
```
Lorsque nous parlons spécifiquement du vertex shader, chaque variable d'entrée est également connue sous le nom d'attribut de vertex. Le nombre maximum d'attributs de vertex que nous sommes autorisés à déclarer est limité par le matériel. OpenGL garantit qu'il y a toujours au moins 16 attributs de vertex à 4 composantes disponibles, mais certains matériels peuvent en autoriser plus, ce que vous pouvez récupérer en interrogeant `GL_MAX_VERTEX_ATTRIBS` :
```cpp
int nrAttributes;
glGetIntegerv(GL_MAX_VERTEX_ATTRIBS, &nrAttributes);
std::cout << "Maximum nr of vertex attributes supported: " << nrAttributes << std::endl;
```
Cette méthode permet souvent d'obtenir un minimum de 16, ce qui est largement suffisant pour la plupart des cas.

## Types
Comme tout autre langage de programmation, GLSL dispose de types de données permettant de spécifier le type de variable avec lequel on souhaite travailler. GLSL possède la plupart des types de base par défaut que nous connaissons dans des langages comme le C : int, float, double, uint et bool. GLSL propose également deux types de conteneurs que nous utiliserons beaucoup, à savoir les **vecteurs** et les **matrices**. Nous aborderons les matrices dans un chapitre ultérieur.

### Vecteurs
Un vecteur en GLSL est un conteneur à 2, 3 ou 4 composants pour n'importe lequel des types de base mentionnés ci-dessus. Ils peuvent prendre la forme suivante (`n` représente le nombre de composants) :

- `vecn` : le vecteur par défaut de `n` flottants.
- `bvecn` : un vecteur de `n` booléens.
- `ivecn` : un vecteur de `n` entiers.
- `uvecn` : un vecteur de `n` entiers non signés.
- `dvecn` : un vecteur de `n` composantes doubles.

La plupart du temps, nous utiliserons le `vecn` de base, car les flottants suffisent pour la plupart de nos besoins.

Les composantes d'un vecteur sont accessibles via `vec.x` où x est la première composante du vecteur. Vous pouvez utiliser `.x`,` .y`, `.z` et `.w` pour accéder à leur première, deuxième, troisième et quatrième composante respectivement. GLSL vous permet également d'utiliser `rgba` pour les couleurs ou `stpq` pour les coordonnées de texture, en accédant aux mêmes composantes.

Le type de données vectoriel permet une sélection intéressante et flexible des composantes, appelée "**swizzling**". Le swizzling nous permet d'utiliser une syntaxe comme celle-ci :
```cpp
vec2 someVec;
vec4 differentVec = someVec.xyxx;
vec3 anotherVec = differentVec.zyw;
vec4 otherVec = someVec.xxxx + anotherVec.yxzy;
```
Vous pouvez utiliser n'importe quelle combinaison de 4 lettres au maximum pour créer un nouveau vecteur (du même type) tant que le vecteur original possède ces composantes ; il n'est pas permis d'accéder à la composante `.z` d'un `vec2` par exemple. Nous pouvons également passer des vecteurs en tant qu'arguments à différents appels de constructeurs de vecteurs, ce qui réduit le nombre d'arguments requis :
```cpp
vec2 vect = vec2(0.5, 0.7);
vec4 result = vec4(vect, 0.0, 0.0);
vec4 otherResult = vec4(result.xyz, 1.0);
```
Les vecteurs sont donc un type de données flexible que nous pouvons utiliser pour toutes sortes d'entrées et de sorties. Tout au long du livre, vous trouverez de nombreux exemples de la manière dont nous pouvons gérer les vecteurs de manière créative.
## Ins and outs (entrées et sorties)
Les shaders sont de jolis petits programmes en soi, mais ils font partie d'un tout et c'est pour cette raison que nous voulons avoir des entrées et des sorties sur les shaders individuels afin de pouvoir déplacer des choses. GLSL a défini les mots-clés `in` et `out` spécifiquement dans ce but. Chaque shader peut spécifier des entrées et des sorties à l'aide de ces mots-clés et chaque fois qu'une variable de sortie correspond à une variable d'entrée de l'étape suivante du shader, elle est transmise. Les vertex shaders et les fragment shaders diffèrent quelque peu.

**Le vertex shader doit recevoir une certaine forme d'entrée, sinon il serait assez inefficace**. Le vertex shader diffère dans son entrée, en ce sens qu'il reçoit son entrée directement à partir des données du vertex.
Pour définir comment les données de sommets sont organisées, nous spécifions les variables d'entrée avec des métadonnées d'emplacement afin de pouvoir configurer les attributs de sommets sur l'unité centrale (CPU). Nous avons vu cela dans le chapitre précédent sous le nom de `layout (location = 0)`. Le vertex shader nécessite donc une spécification de disposition supplémentaire pour ses entrées afin que nous puissions le lier aux données de sommets.
>	Il est également possible d'omettre le spécificateur de disposition `(layout = 0)` et de demander l'emplacement des attributs dans votre code OpenGL via `glGetAttribLocation`, mais je préfère les définir dans le vertex shader. C'est plus facile à comprendre et cela vous épargne (ainsi qu'à OpenGL) un peu de travail.

L'autre exception est que le fragment shader nécessite une variable de sortie de couleur `vec4`, puisque le fragment shader a besoin de générer une couleur de sortie finale. Si vous ne spécifiez pas de couleur de sortie dans votre fragment shader, la sortie du buffer de couleur pour ces fragments sera indéfinie (ce qui signifie généralement qu'OpenGL les rendra en noir ou en blanc).

Ainsi, **si nous voulons envoyer des données d'un shader à l'autre, nous devons déclarer une sortie dans le shader d'envoi et une entrée similaire dans le shader de réception**. Lorsque les types et les noms sont identiques des deux côtés, OpenGL liera ces variables ensemble et il sera alors possible d'envoyer des données entre les shaders (ce qui est fait lors de la liaison d'un objet de programme). Pour vous montrer comment cela fonctionne en pratique, nous allons modifier les shaders du chapitre précédent pour laisser le vertex shader décider de la couleur pour le fragment shader.

**Vertex shader**
```cpp
#version 330 core
layout (location = 0) in vec3 aPos; // the position variable has attribute position 0
  
out vec4 vertexColor; // specify a color output to the fragment shader

void main()
{
    gl_Position = vec4(aPos, 1.0); // see how we directly give a vec3 to vec4's constructor
    vertexColor = vec4(0.5, 0.0, 0.0, 1.0); // set the output variable to a dark-red color
}
```
**Fragment shader**
```cpp
#version 330 core
out vec4 FragColor;
  
in vec4 vertexColor; // the input variable from the vertex shader (same name and same type)  

void main()
{
    FragColor = vertexColor;
} 
```

Vous pouvez voir que nous avons déclaré une variable `vertexColor` en tant que sortie `vec4` que nous avons définie dans le vertex shader et que nous déclarons une entrée `vertexColor` similaire dans le fragment shader. Comme elles ont toutes deux le même type et le même nom, la variable `vertexColor` dans le fragment shader est liée à la variable `vertexColor` dans le vertex shader. Étant donné que nous avons défini une couleur rouge foncé dans le vertex shader, les fragments résultants devraient également être rouge foncé. L'image suivante montre le résultat :
![[img/shader1.png]]
Nous y voilà ! Nous venons de réussir à envoyer une valeur du vertex shader au fragment shader. Mettons un peu de piment et voyons si nous pouvons envoyer une couleur de notre application au fragment shader !
## Uniforms
Les uniformes sont un autre moyen de transmettre les données de notre application sur le processeur aux shaders sur le GPU. Les uniformes sont toutefois légèrement différents des attributs de vertex. Tout d'abord, les uniformes sont globaux. Globaux, ce qui signifie qu'une variable uniforme est unique par objet du programme de shaders, et qu'elle est accessible à partir de n'importe quel shader à n'importe quelle étape du programme de shaders. Deuxièmement, quelle que soit la valeur de l'uniforme, les uniformes conservent leur valeur jusqu'à ce qu'ils soient réinitialisés ou mis à jour.

Pour déclarer un uniforme en GLSL, il suffit d'ajouter le mot-clé `uniform` à un shader avec un type et un nom. A partir de là, nous pouvons utiliser l'uniforme nouvellement déclaré dans le shader. Voyons si cette fois-ci nous pouvons définir la couleur du triangle par le biais d'un uniforme :
```cpp
#version 330 core
out vec4 FragColor;
  
uniform vec4 ourColor; // we set this variable in the OpenGL code.

void main()
{
    FragColor = ourColor;
}   
```
Nous avons déclaré un uniforme `vec4 ourColor` dans le fragment shader et défini la couleur de sortie du fragment en fonction du contenu de cette valeur uniforme. Comme les uniformes sont des variables globales, nous pouvons les définir dans n'importe quelle étape du shader. Il n'est donc pas nécessaire de repasser par le vertex shader pour transmettre quelque chose au fragment shader. Nous n'utilisons pas cet uniforme dans le vertex shader, il n'est donc pas nécessaire de le définir ici.
>	Si vous déclarez un uniforme qui n'est utilisé nulle part dans votre code GLSL, le compilateur supprimera silencieusement la variable de la version compilée, ce qui est la cause de plusieurs erreurs frustrantes ; gardez cela à l'esprit !

L'uniforme est actuellement vide ; nous n'avons pas encore ajouté de données à l'uniforme, alors essayons de le faire. Nous devons d'abord trouver l'index/l'emplacement de l'attribut uniform dans notre shader. Une fois que nous avons l'index/l'emplacement de l'uniforme, nous pouvons mettre à jour ses valeurs. Au lieu de passer une seule couleur au fragment shader, nous allons pimenter les choses en changeant progressivement de couleur au fil du temps :
```cpp
float timeValue = glfwGetTime();
float greenValue = (sin(timeValue) / 2.0f) + 0.5f; // [0,1]
int vertexColorLocation = glGetUniformLocation(shaderProgram, "ourColor");
glUseProgram(shaderProgram);
glUniform4f(vertexColorLocation, 0.0f, greenValue, 0.0f, 1.0f);
```
Tout d'abord, nous récupérons le temps d'exécution en secondes via `glfwGetTime()`. Ensuite, nous faisons varier la couleur entre 0,0 et 1,0 à l'aide de la fonction `sin` et nous stockons le résultat dans `greenValue`.

Nous demandons ensuite l'emplacement de l'uniforme `ourColor` à l'aide de `glGetUniformLocation`. Nous fournissons le programme du shader et le nom de l'uniforme (dont nous voulons récupérer l'emplacement) à la fonction de requête. Si `glGetUniformLocation` renvoie -1, cela signifie qu'il n'a pas pu trouver l'emplacement. Enfin, nous pouvons définir la valeur de l'uniforme à l'aide de la fonction `glUniform4f`. Notez que la recherche de l'emplacement de l'uniforme n'exige pas que vous utilisiez d'abord le programme de shader, mais la mise à jour d'un uniforme exige que vous utilisiez d'abord le programme (en appelant `glUseProgram`), parce qu'il définit l'uniforme sur le programme de shader actuellement actif.

>Parce qu'OpenGL est à la base une bibliothèque C, il n'a pas de support natif pour la surcharge des fonctions, donc chaque fois qu'une fonction peut être appelée avec différents types, OpenGL définit de nouvelles fonctions pour chaque type requis ; `glUniform` est un parfait exemple de ceci. La fonction requiert un postfixe spécifique pour le type d'uniforme que vous souhaitez définir. Quelques-uns des postfixes possibles sont :
>- `f` : la fonction attend un `float` comme valeur.
>- `i` : la fonction attend un `int` comme valeur.
>- `ui` : la fonction attend un `unsigned int` comme valeur.
>- `3f` : la fonction attend 3 `float` comme valeur.
>- `fv` : la fonction attend un vecteur/rayon `float` comme valeur.
>Lorsque vous voulez configurer une option d'OpenGL, choisissez simplement la fonction surchargée qui correspond à votre type. Dans notre cas, nous voulons définir 4 flottants de l'uniforme individuellement, donc nous passons nos données via `glUniform4f` (notez que nous aurions aussi pu utiliser la version `fv`).

Maintenant que nous savons comment définir les valeurs des variables uniformes, nous pouvons les utiliser pour le rendu. Si nous voulons que la couleur change progressivement, nous devons mettre à jour cette variable uniforme à chaque image, sinon le triangle conserverait une seule couleur unie si nous ne la définissions qu'une seule fois. Nous calculons donc la valeur verte et mettons à jour l'uniforme à chaque itération de rendu :
```cpp
while(!glfwWindowShouldClose(window))
{
    // input
    processInput(window);

    // render
    // clear the colorbuffer
    glClearColor(0.2f, 0.3f, 0.3f, 1.0f);
    glClear(GL_COLOR_BUFFER_BIT);

    // be sure to activate the shader
    glUseProgram(shaderProgram);
  
    // update the uniform color
    float timeValue = glfwGetTime();
    float greenValue = sin(timeValue) / 2.0f + 0.5f;
    int vertexColorLocation = glGetUniformLocation(shaderProgram, "ourColor");
    glUniform4f(vertexColorLocation, 0.0f, greenValue, 0.0f, 1.0f);

    // now render the triangle
    glBindVertexArray(VAO);
    glDrawArrays(GL_TRIANGLES, 0, 3);
  
    // swap buffers and poll IO events
    glfwSwapBuffers(window);
    glfwPollEvents();
}
```
Le code est une adaptation relativement simple du code précédent. Cette fois, nous mettons à jour une valeur uniforme à chaque image avant de dessiner le triangle. Si vous mettez à jour l'uniforme correctement, vous devriez voir la couleur de votre triangle passer progressivement du vert au noir, puis au vert.

![[video/shaders.mp4]]
Consultez le code source [ici](https://learnopengl.com/code_viewer_gh.php?code=src/1.getting_started/3.1.shaders_uniform/shaders_uniform.cpp) si vous êtes bloqué.

Comme vous pouvez le voir, les uniformes sont un outil utile pour définir des attributs qui peuvent changer à chaque image, ou pour échanger des données entre votre application et vos shaders, mais que faire si nous voulons définir une couleur pour chaque vertex ? Dans ce cas, nous devrions déclarer autant d'uniformes que de sommets. Une meilleure solution serait d'inclure plus de données dans les attributs des vertex, ce que nous allons faire maintenant.

## Plus d'attributs!
Nous avons vu dans le chapitre précédent comment remplir un VBO, configurer des pointeurs d'attributs de vertex et stocker le tout dans un VAO. Cette fois, nous voulons également ajouter des données de couleur aux données de vertex. Nous allons ajouter des données de couleur sous la forme de 3 flottants au tableau des vertex. Nous attribuons une couleur rouge, verte et bleue à chacun des coins de notre triangle :
```cpp
float vertices[] = {
    // positions         // colors
     0.5f, -0.5f, 0.0f,  1.0f, 0.0f, 0.0f,   // bottom right
    -0.5f, -0.5f, 0.0f,  0.0f, 1.0f, 0.0f,   // bottom left
     0.0f,  0.5f, 0.0f,  0.0f, 0.0f, 1.0f    // top 
};    
```
Puisque nous avons maintenant plus de données à envoyer au vertex shader, il est nécessaire d'ajuster le vertex shader pour qu'il reçoive également notre valeur de couleur en tant qu'entrée d'attribut de vertex. Notez que nous avons fixé l'emplacement de l'attribut `aColor` à 1 à l'aide du spécificateur d'agencement :
```cpp
#version 330 core
layout (location = 0) in vec3 aPos;   // the position variable has attribute position 0
layout (location = 1) in vec3 aColor; // the color variable has attribute position 1
  
out vec3 ourColor; // output a color to the fragment shader

void main()
{
    gl_Position = vec4(aPos, 1.0);
    ourColor = aColor; // set ourColor to the input color we got from the vertex data
}  
```
Puisque nous n'utilisons plus d'uniforme pour la couleur du fragment, mais que nous utilisons maintenant la variable de sortie `ourColor`, nous devrons également modifier le shader du fragment :
```cpp
#version 330 core
out vec4 FragColor;  
in vec3 ourColor;
  
void main()
{
    FragColor = vec4(ourColor, 1.0);
}
```
Comme nous avons ajouté un autre attribut de sommet et mis à jour la mémoire du VBO, nous devons reconfigurer les pointeurs d'attributs de sommet. Les données mises à jour dans la mémoire du VBO ressemblent maintenant à ceci :
![[img/shader2.png]]
En connaissant la disposition actuelle, nous pouvons mettre à jour le format des vertex avec `glVertexAttribPointer` :
```cpp
// position attribute
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);
// color attribute
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float), (void*)(3* sizeof(float)));
glEnableVertexAttribArray(1);
```
Les premiers arguments de `glVertexAttribPointer` sont relativement simples. Cette fois, nous configurons l'attribut vertex sur l'emplacement d'attribut 1. Les valeurs de couleur ont une taille de 3 flottants et nous ne normalisons pas les valeurs.

**Comme nous avons maintenant deux attributs de vertex, nous devons recalculer la valeur de stride**. Pour obtenir la valeur d'attribut suivante (par exemple, la composante x suivante du vecteur de position) dans le tableau de données, nous devons déplacer 6 flottants vers la droite, trois pour les valeurs de position et trois pour les valeurs de couleur. Cela nous donne une valeur de stride de 6 fois la taille d'un flottant en octets (= 24 octets).
**Cette fois-ci, nous devons également spécifier un décalage (offset)**. Pour chaque sommet, l'attribut de sommet de position est le premier, nous déclarons donc un décalage de 0. L'attribut de couleur commence après les données de position, le décalage est donc de 3 * sizeof(float) en octets (= 12 octets).

L'exécution de l'application devrait donner l'image suivante :
![[img/shader3.png]]
Consultez le code source [ici](https://learnopengl.com/code_viewer_gh.php?code=src/1.getting_started/3.2.shaders_interpolation/shaders_interpolation.cpp) si vous êtes bloqué.

L'image n'est peut-être pas exactement ce à quoi vous vous attendez, car nous n'avons fourni que 3 couleurs, et non la vaste palette de couleurs que nous voyons actuellement. Tout ceci est le résultat de ce que l'on appelle **l'interpolation de fragments** dans le fragment shader. Lors du rendu d'un triangle, l'étape de **rastérisation** produit généralement beaucoup plus de fragments que de vertices spécifiés à l'origine. Le rasterizer détermine alors les positions de chacun de ces fragments en fonction de leur emplacement sur la forme du triangle.
Sur la base de ces positions, il interpole toutes les variables d'entrée du fragment shader. Disons par exemple que nous avons une ligne dont le point supérieur a une couleur verte et le point inférieur une couleur bleue. Si le fragment shader est exécuté sur un fragment situé à 70 % de la ligne, l'attribut d'entrée de couleur résultant sera une combinaison linéaire de vert et de bleu ; pour être plus précis : 30 % de bleu et 70 % de vert.

C'est exactement ce qui s'est passé pour le triangle. Nous avons 3 sommets et donc 3 couleurs, et à en juger par les pixels du triangle, il contient probablement environ 50000 fragments, où le fragment shader a interpolé les couleurs parmi ces pixels. Si vous regardez bien les couleurs, vous verrez que tout est logique : du rouge au bleu, on passe d'abord au violet, puis au bleu. **L'interpolation de fragments est appliquée à tous les attributs d'entrée du fragment shader**.

## Notre propre classe de shaders
L'écriture, la compilation et la gestion des shaders peuvent être assez lourdes. Pour terminer sur le sujet des shaders, nous allons nous simplifier la vie en construisant une classe de shaders qui lit les shaders sur le disque, les compile et les lie, vérifie les erreurs et est facile à utiliser. Cela vous donne également une idée de la manière dont nous pouvons encapsuler certaines des connaissances que nous avons acquises jusqu'à présent dans des objets abstraits utiles.

Nous allons créer la classe shader entièrement dans un fichier d'en-tête, principalement à des fins d'apprentissage et de portabilité. Commençons par ajouter les includes nécessaires et par définir la structure de la classe :
```cpp
#ifndef SHADER_H
#define SHADER_H

#include <glad/glad.h> // include glad to get all the required OpenGL headers
  
#include <string>
#include <fstream>
#include <sstream>
#include <iostream>
  

class Shader
{
public:
    // the program ID
    unsigned int ID;
  
    // constructor reads and builds the shader
    Shader(const char* vertexPath, const char* fragmentPath);
    // use/activate the shader
    void use();
    // utility uniform functions
    void setBool(const std::string &name, bool value) const;  
    void setInt(const std::string &name, int value) const;   
    void setFloat(const std::string &name, float value) const;
};
  
#endif
```
>	Nous avons utilisé plusieurs directives de préprocesseur en haut du fichier d'en-tête. L'utilisation de ces petites lignes de code informe votre compilateur de n'inclure et de ne compiler que ce fichier d'en-tête s'il n'a pas encore été inclus, même si plusieurs fichiers incluent l'en-tête du shader. Cela permet d'éviter les conflits d'édition de liens.

La classe shader contient l'ID du programme shader. Son constructeur requiert les chemins d'accès au code source du vertex et du fragment shader respectivement, que nous pouvons stocker sur le disque sous la forme de simples fichiers texte. Pour ajouter un petit plus, nous avons également ajouté plusieurs fonctions utilitaires pour nous faciliter un peu la vie : use active le programme shader, et tous les `set...` interrogent un emplacement uniforme et fixe sa valeur.
### Lire depuis un fichier
Nous utilisons des flux de fichiers C++ pour lire le contenu du fichier dans plusieurs objets de type chaîne de caractères :
```cpp
Shader(const char* vertexPath, const char* fragmentPath)
{
    // 1. retrieve the vertex/fragment source code from filePath
    std::string vertexCode;
    std::string fragmentCode;
    std::ifstream vShaderFile;
    std::ifstream fShaderFile;
    // ensure ifstream objects can throw exceptions:
    vShaderFile.exceptions (std::ifstream::failbit | std::ifstream::badbit);
    fShaderFile.exceptions (std::ifstream::failbit | std::ifstream::badbit);
    try 
    {
        // open files
        vShaderFile.open(vertexPath);
        fShaderFile.open(fragmentPath);
        std::stringstream vShaderStream, fShaderStream;
        // read file's buffer contents into streams
        vShaderStream << vShaderFile.rdbuf();
        fShaderStream << fShaderFile.rdbuf();		
        // close file handlers
        vShaderFile.close();
        fShaderFile.close();
        // convert stream into string
        vertexCode   = vShaderStream.str();
        fragmentCode = fShaderStream.str();		
    }
    catch(std::ifstream::failure e)
    {
        std::cout << "ERROR::SHADER::FILE_NOT_SUCCESFULLY_READ" << std::endl;
    }
    const char* vShaderCode = vertexCode.c_str();
    const char* fShaderCode = fragmentCode.c_str();
    [...]
```
Ensuite, nous devons compiler et lier les shaders. Notez que nous vérifions également si la compilation/liaison a échoué et si c'est le cas, nous affichons les erreurs de compilation. Ceci est extrêmement utile lors du débogage (vous aurez besoin de ces journaux d'erreurs un jour ou l'autre) :
```cpp
// 2. compile shaders
unsigned int vertex, fragment;
int success;
char infoLog[512];
   
// vertex Shader
vertex = glCreateShader(GL_VERTEX_SHADER);
glShaderSource(vertex, 1, &vShaderCode, NULL);
glCompileShader(vertex);
// print compile errors if any
glGetShaderiv(vertex, GL_COMPILE_STATUS, &success);
if(!success)
{
    glGetShaderInfoLog(vertex, 512, NULL, infoLog);
    std::cout << "ERROR::SHADER::VERTEX::COMPILATION_FAILED\n" << infoLog << std::endl;
};
  
// similiar for Fragment Shader
[...]
  
// shader Program
ID = glCreateProgram();
glAttachShader(ID, vertex);
glAttachShader(ID, fragment);
glLinkProgram(ID);
// print linking errors if any
glGetProgramiv(ID, GL_LINK_STATUS, &success);
if(!success)
{
    glGetProgramInfoLog(ID, 512, NULL, infoLog);
    std::cout << "ERROR::SHADER::PROGRAM::LINKING_FAILED\n" << infoLog << std::endl;
}
  
// delete the shaders as they're linked into our program now and no longer necessary
glDeleteShader(vertex);
glDeleteShader(fragment);
```
La fonction `use` est simple :
```cpp
void use() 
{ 
    glUseProgram(ID);
}  
```
De même pour les fonctions `set`:
```cpp
void setBool(const std::string &name, bool value) const
{         
    glUniform1i(glGetUniformLocation(ID, name.c_str()), (int)value); 
}
void setInt(const std::string &name, int value) const
{ 
    glUniform1i(glGetUniformLocation(ID, name.c_str()), value); 
}
void setFloat(const std::string &name, float value) const
{ 
    glUniform1f(glGetUniformLocation(ID, name.c_str()), value); 
} 
```
Et voilà, une [classe de shader] (https://learnopengl.com/code_viewer_gh.php?code=includes/learnopengl/shader_s.h) terminée. L'utilisation de la classe de shaders est assez simple ; nous créons un objet shader une fois et à partir de là, nous commençons simplement à l'utiliser :
```cpp
Shader ourShader("path/to/shaders/shader.vs", "path/to/shaders/shader.fs");
[...]
while(...)
{
    ourShader.use();
    ourShader.setFloat("someUniform", 1.0f);
    DrawStuff();
}
```
Ici, nous avons stocké le code source du vertex et du fragment shader dans deux fichiers appelés `shader.vs` et `shader.fs`. Vous êtes libre de nommer vos fichiers de shaders comme vous le souhaitez ; personnellement, je trouve les extensions `.vs` et `.fs` assez intuitives.

Vous pouvez trouver le code source [ici](https://learnopengl.com/code_viewer_gh.php?code=src/1.getting_started/3.3.shaders_class/shaders_class.cpp) en utilisant notre [classe de shaders](https://learnopengl.com/code_viewer_gh.php?code=includes/learnopengl/shader_s.h) nouvellement créée. Notez que vous pouvez cliquer sur les chemins des fichiers de shaders pour trouver le code source des shaders.
