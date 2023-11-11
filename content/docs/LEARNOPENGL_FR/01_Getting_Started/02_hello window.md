---
tags: [cg, opengl, computer graphics, cpp]
dg-publish: true
---
# Hello Window
Voyons si nous pouvons faire fonctionner GLFW. Tout d'abord, créez un fichier .cpp et ajoutez les inclusions suivantes au début de votre fichier nouvellement créé. 
```cpp
#include <glad/glad.h>
#include <GLFW/glfw3.h>
```
>Assurez-vous d'inclure GLAD avant GLFW. Le fichier include de GLAD inclut les en-têtes OpenGL nécessaires dans les coulisses (comme `GL/gl.h`). Il faut donc s'assurer d'inclure GLAD avant les autres fichiers d'en-tête qui nécessitent OpenGL (comme GLFW). 

 Ensuite, nous créons la fonction principale dans laquelle nous allons instancier la fenêtre GLFW:
 ```cpp
int main()
{
    glfwInit();
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
    //glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
  
    return 0;
}
```
Dans la fonction principale, nous commençons par initialiser GLFW avec `glfwInit`, après quoi nous pouvons configurer GLFW en utilisant `glfwWindowHint`.
Le premier argument de `glfwWindowHint` nous indique l'option que nous voulons configurer, où nous pouvons sélectionner l'option à partir d'une grande liste d'options possibles préfixées par `GLFW_`.
Le deuxième argument est un entier qui définit la valeur de notre option. Une liste de toutes les options possibles et de leurs valeurs correspondantes peut être trouvée dans la documentation sur la gestion des fenêtres de GLFW.
Si vous essayez d'exécuter l'application maintenant et qu'elle produit un grand nombre d'erreurs de références non définies, cela signifie que vous n'avez pas réussi à lier la bibliothèque GLFW.

Puisque ce livre se concentre sur la version 3.3 d'OpenGL, nous aimerions dire à GLFW que la version 3.3 est la version d'OpenGL que nous voulons utiliser. De cette manière, GLFW peut prendre les dispositions nécessaires lors de la création du contexte OpenGL. Cela permet de s'assurer que lorsqu'un utilisateur n'a pas la bonne version d'OpenGL, GLFW ne s'exécute pas. Nous fixons la version majeure et mineure à 3. Nous indiquons également à GLFW que nous voulons utiliser explicitement le core-profile. Dire à GLFW que nous voulons utiliser le core-profile signifie que nous aurons accès à un sous-ensemble plus petit de fonctionnalités OpenGL sans fonctionnalités rétrocompatibles dont nous n'avons plus besoin.
**Notez que sous Mac OS X**, vous devez ajouter `glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE)` ; à votre code d'initialisation pour que cela fonctionne.

>Assurez-vous que les versions 3.3 ou plus d'OpenGL sont installées sur votre système/matériel, sinon l'application plantera ou affichera un comportement indéfini. Pour connaître la version d'OpenGL sur votre machine, appelez `glxinfo` sur les machines Linux ou utilisez un utilitaire comme *OpenGL Extension Viewer* pour Windows. Si la version supportée est inférieure, essayez de vérifier si votre carte vidéo supporte OpenGL 3.3+ (sinon elle est vraiment vieille) et/ou mettez à jour vos pilotes.

Ensuite, nous devons créer un objet fenêtre. Cet objet contient toutes les données de fenêtrage et est nécessaire à la plupart des autres fonctions de GLFW. 

```cpp
GLFWwindow* window = glfwCreateWindow(800, 600, "LearnOpenGL", NULL, NULL);
if (window == NULL)
{
    std::cout << "Failed to create GLFW window" << std::endl;
    glfwTerminate();
    return -1;
}
glfwMakeContextCurrent(window);
```
La fonction `glfwCreateWindow` requiert la largeur et la hauteur de la fenêtre comme ses deux premiers arguments respectivement.
Le troisième argument nous permet de créer un nom pour la fenêtre ; pour l'instant nous l'appelons "LearnOpenGL" mais vous êtes autorisés à lui donner le nom que vous voulez.
Nous pouvons ignorer les deux derniers paramètres. La fonction renvoie un objet `GLFWwindow` dont nous aurons besoin plus tard pour d'autres opérations de GLFW. Après cela, nous demandons à GLFW de faire du contexte de notre fenêtre le contexte principal du thread courant.

## GLAD
Dans le chapitre précédent, nous avons mentionné que GLAD gère les pointeurs de fonction pour OpenGL, nous voulons donc initialiser GLAD avant d'appeler une fonction OpenGL :
```cpp
if (!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress))
{
    std::cout << "Failed to initialize GLAD" << std::endl;
    return -1;
}   
```
Nous passons à GLAD la fonction pour charger l'adresse des pointeurs de fonction OpenGL qui est spécifique au système d'exploitation. GLFW nous fournit `glfwGetProcAddress` qui définit la fonction correcte en fonction du système d'exploitation pour lequel nous compilons. 
## Viewport
Avant de commencer le rendu, nous devons faire une dernière chose. Nous devons indiquer à OpenGL la taille de la fenêtre de rendu afin qu'OpenGL sache comment nous voulons afficher les données et les coordonnées par rapport à la fenêtre. Nous pouvons définir ces dimensions via la fonction `glViewport` :
```cpp
glViewport(0, 0, 800, 600);
```
Les deux premiers paramètres de `glViewport` définissent l'emplacement du coin inférieur gauche de la fenêtre. Les troisième et quatrième paramètres définissent la largeur et la hauteur de la fenêtre de rendu en pixels, que nous fixons à la taille de la fenêtre de GLFW.  
  
**Nous pourrions en fait fixer les dimensions de la fenêtre à des valeurs inférieures à celles de GLFW **; ainsi, tout le rendu OpenGL serait affiché dans une fenêtre plus petite et nous pourrions, par exemple, afficher d'autres éléments en dehors de la fenêtre OpenGL.

>En coulisses, OpenGL utilise les données spécifiées par `glViewport` pour transformer les coordonnées 2D qu'il a traitées en coordonnées sur votre écran. Par exemple, un point traité de la position (-0.5,0.5) serait (comme sa transformation finale) mappé à (200,450) en coordonnées d'écran. Notez que les coordonnées traitées dans OpenGL sont comprises entre -1 et 1, de sorte que nous mappons effectivement de l'intervalle (-1 à 1) à (0, 800) et (0, 600).

Cependant, lorsque l'utilisateur redimensionne la fenêtre, la fenêtre de visualisation doit également être ajustée. Nous pouvons enregistrer une fonction de rappel sur la fenêtre qui est appelée chaque fois que la fenêtre est redimensionnée. Cette fonction de rappel de redimensionnement a le prototype suivant :
```cpp
void framebuffer_size_callback(GLFWwindow* window, int width, int height);  
```
La fonction `framebuffer_size_callback` prend une fenêtre GLFW comme premier argument et deux entiers indiquant les nouvelles dimensions de la fenêtre. Chaque fois que la taille de la fenêtre change, GLFW appelle cette fonction et remplit les arguments appropriés pour que vous puissiez les traiter.
```cpp
void framebuffer_size_callback(GLFWwindow* window, int width, int height)
{
    glViewport(0, 0, width, height);
} 
```
Nous devons indiquer à GLFW que nous voulons appeler cette fonction à chaque redimensionnement de fenêtre en l'enregistrant : 
```cpp
glfwSetFramebufferSizeCallback(window, framebuffer_size_callback);  
```
Lorsque la fenêtre est affichée pour la première fois, `framebuffer_size_callback` est également appelé avec les dimensions résultantes de la fenêtre. Pour les écrans Rétina, la largeur et la hauteur seront significativement plus élevées que les valeurs d'origine.  
  
Il existe de nombreuses fonctions de rappel que nous pouvons définir pour enregistrer nos propres fonctions. Par exemple, nous pouvons créer une fonction de rappel pour traiter les changements d'entrée du joystick, traiter les messages d'erreur, etc. Nous enregistrons les fonctions de rappel après avoir créé la fenêtre et avant que la boucle de rendu ne soit lancée.

## Préparez vos moteurs
Nous ne voulons pas que l'application dessine une seule image, puis qu'elle quitte immédiatement et ferme la fenêtre. Nous voulons que l'application continue à dessiner des images et à gérer les entrées de l'utilisateur jusqu'à ce que le programme reçoive l'ordre explicite de s'arrêter. **Pour cette raison, nous devons créer une boucle while,** que nous appelons maintenant **boucle de rendu**, qui continue à fonctionner jusqu'à ce que nous demandions à GLFW de s'arrêter. Le code suivant montre une boucle de rendu très simple :
```cpp
while(!glfwWindowShouldClose(window))
{
    glfwSwapBuffers(window);
    glfwPollEvents();    
}
```
La fonction `glfwWindowShouldClose` vérifie au début de chaque itération de la boucle si GLFW a reçu l'ordre de se fermer. Si c'est le cas, la fonction renvoie vrai et la boucle de rendu s'arrête, après quoi nous pouvons fermer l'application.

La fonction `glfwPollEvents` vérifie si des événements sont déclenchés (comme des entrées clavier ou des mouvements de souris), met à jour l'état de la fenêtre et appelle les fonctions correspondantes (que nous pouvons enregistrer via des méthodes de rappel). La fonction `glfwSwapBuffer` va permuter le color buffer (un grand tampon 2D qui contient les valeurs de couleur pour chaque pixel de la fenêtre de GLFW) qui est utilisé pour le rendu pendant cette itération de rendu et l'afficher en sortie à l'écran.

>Double buffer  
	Lorsqu'une application dessine dans une seule mémoire tampon, l'image résultante peut présenter des problèmes de scintillement. **En effet, l'image résultante n'est pas dessinée en un instant, mais pixel par pixel, généralement de gauche à droite et de haut en bas**. Comme cette image n'est pas affichée instantanément à l'utilisateur tout en étant rendue, le résultat peut contenir des artefacts. **Pour contourner ces problèmes, les applications de fenêtrage utilisent une double mémoire tampon pour le rendu**. La mémoire tampon avant (front buffer) contient l'image de sortie finale qui est affichée à l'écran, tandis que toutes les commandes de rendu dessinent dans la mémoire tampon arrière (back buffer). Dès que toutes les commandes de rendu sont terminées, le tampon arrière est remplacé par le tampon avant, de sorte que l'image peut être affichée sans être en train d'être rendue, ce qui supprime tous les artefacts susmentionnés.

## Une dernière chose
Dès que nous sortons de la boucle de rendu, nous souhaitons nettoyer/supprimer correctement toutes les ressources de GLFW qui ont été allouées. Nous pouvons le faire via la fonction `glfwTerminate` que nous appelons à la fin de la fonction principale. 
```cpp
glfwTerminate();
return 0;
```
Ceci nettoiera toutes les ressources et quittera correctement l'application. Essayez maintenant de compiler votre application et si tout s'est bien passé, vous devriez voir la sortie suivante : 
![[img/window6.png]]
Si c'est une image noire très terne et ennuyeuse, c'est que vous avez bien fait les choses ! Si vous n'avez pas obtenu la bonne image ou si vous ne savez pas comment tout s'articule, consultez le code source complet [ici](https://learnopengl.com/code_viewer_gh.php?code=src/1.getting_started/1.1.hello_window/hello_window.cpp) (et si des couleurs différentes ont commencé à clignoter, continuez à lire).  
  
Si vous avez des problèmes pour compiler l'application, assurez-vous d'abord que toutes les options de votre linker sont correctement réglées et que vous avez inclus les bons répertoires dans votre IDE (comme expliqué dans le chapitre précédent). Assurez-vous également que votre code est correct ; vous pouvez le vérifier en le comparant avec le code source complet.

## Input (entrée)
Nous souhaitons également disposer d'une forme de contrôle des entrées dans GLFW et nous pouvons y parvenir grâce à plusieurs fonctions d'entrée de GLFW. Nous utiliserons la fonction `glfwGetKey` de GLFW qui prend la fenêtre en entrée ainsi qu'une touche. La fonction retourne si cette touche est actuellement pressée. Nous créons une fonction `processInput` pour organiser l'ensemble du code d'entrée :
```cpp
void processInput(GLFWwindow *window)
{
    if(glfwGetKey(window, GLFW_KEY_ESCAPE) == GLFW_PRESS)
        glfwSetWindowShouldClose(window, true);
}
```
Nous vérifions ici si l'utilisateur a appuyé sur la touche d'échappement (si ce n'est pas le cas, `glfwGetKey` renvoie `GLFW_RELEASE`). Si l'utilisateur a appuyé sur la touche d'échappement, nous fermons GLFW en définissant sa propriété `WindowShouldClose`à true à l'aide de `glfwSetwindowShouldClose`. La vérification de la condition suivante de la boucle while principale échoue alors et l'application se ferme.  
  
Nous appelons ensuite `processInput` à chaque itération de la boucle de rendu :
```cpp
while (!glfwWindowShouldClose(window))
{
    processInput(window);

    glfwSwapBuffers(window);
    glfwPollEvents();
}  
```
Nous disposons ainsi d'un moyen simple de vérifier la présence de touches spécifiques et de réagir en conséquence à chaque image. **Une itération de la boucle de rendu est plus communément appelée une image**.

## Rendering (le rendu)
Nous voulons placer toutes les commandes de rendu dans la boucle de rendu, puisque nous voulons exécuter toutes les commandes de rendu à chaque itération ou image de la boucle. Cela ressemblerait un peu à ceci :
```cpp
// render loop
while(!glfwWindowShouldClose(window))
{
    // input
    processInput(window);

    // rendering commands here
    ...

    // check and call events and swap the buffers
    glfwPollEvents();
    glfwSwapBuffers(window);
}
```
Juste pour tester si les choses fonctionnent réellement, nous voulons effacer l'écran avec une couleur de notre choix. **Au début de l'image, nous voulons effacer l'écran. Sinon, nous verrions toujours les résultats de l'image précédente (cela pourrait être l'effet recherché, mais en général ce n'est pas le cas)**. Nous pouvons effacer le tampon de couleur (color buffer) de l'écran en utilisant `glClear` où nous passons des bits de tampon (buffer bits) pour spécifier le tampon que nous voulons effacer. Les bits possibles sont `GL_COLOR_BUFFER_BIT`, `GL_DEPTH_BUFFER_BIT` et `GL_STENCIL_BUFFER_BIT`. Pour l'instant, nous ne nous intéressons qu'aux valeurs de couleur et nous n'effaçons donc que le tampon de couleur.
```cpp
glClearColor(0.2f, 0.3f, 0.3f, 1.0f);
glClear(GL_COLOR_BUFFER_BIT);
```
Notez que nous spécifions également la couleur à utiliser pour effacer l'écran à l'aide de `glClearColor`. Chaque fois que nous appelons `glClear` et que nous vidons le tampon de couleurs, celui-ci est entièrement rempli avec la couleur configurée par `glClearColor`. Il en résultera une couleur vert foncé-bleuâtre. 

>Comme vous vous en souvenez peut-être dans le chapitre sur OpenGL, la fonction `glClearColor` **est une fonction de définition d'état** et `glClear` **est une fonction d'utilisation d'état en ce sens qu'elle utilise l'état actuel pour récupérer la couleur d'effacement.** 

![[img/window7.png]]
Le code source complet de l'application est disponible [ici](https://learnopengl.com/code_viewer_gh.php?code=src/1.getting_started/1.2.hello_window_clear/hello_window_clear.cpp).

Nous avons maintenant tout ce qu'il faut pour remplir la boucle de rendu avec de nombreux appels de rendu, mais c'est pour le prochain chapitre. Je pense que nous avons assez divagué ici. 