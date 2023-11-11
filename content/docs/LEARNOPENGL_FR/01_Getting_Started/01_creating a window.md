---
tags: [cg, opengl, computer graphics, cpp]
dg-publish: true
---
# Créer une fenêtre
La première chose à faire avant de commencer à créer des graphiques étonnants est de créer un contexte OpenGL et une fenêtre d'application pour dessiner. Cependant, ces opérations sont spécifiques à chaque système d'exploitation et OpenGL essaie délibérément de s'abstraire de ces opérations. Cela signifie que nous devons créer une fenêtre, définir un contexte et gérer l'entrée de l'utilisateur par nous-mêmes.  
  
Heureusement, il existe de nombreuses bibliothèques qui fournissent les fonctionnalités que nous recherchons, dont certaines sont spécifiquement destinées à OpenGL. Ces bibliothèques nous épargnent tout le travail spécifique au système d'exploitation et nous donnent une fenêtre et un contexte OpenGL pour effectuer le rendu. Certaines des bibliothèques les plus populaires sont **GLUT**, **SDL**, **SFML** et **GLFW**. Sur LearnOpenGL, nous utiliserons GLFW. N'hésitez pas à utiliser n'importe quelle autre bibliothèque, la configuration de la plupart d'entre elles est similaire à celle de GLFW.

## GLFW
GLFW est une bibliothèque, écrite en C, spécifiquement destinée à OpenGL. GLFW nous fournit le strict nécessaire pour le rendu de goodies à l'écran. **Il nous permet de créer un contexte OpenGL, de définir les paramètres de la fenêtre, et de gérer les entrées utilisateur, ce qui est largement suffisant pour nos besoins.**  
  
L'objectif de ce chapitre et du suivant est de faire fonctionner GLFW, en s'assurant qu'il crée correctement un contexte OpenGL et qu'il affiche une fenêtre simple dans laquelle nous pourrons nous amuser. Ce chapitre adopte une approche étape par étape pour récupérer, construire et lier la bibliothèque GLFW. Nous utiliserons l'IDE Microsoft Visual Studio 2019 à partir de cet écrit (notez que le processus est le même sur les versions plus récentes de Visual Studio). Si vous n'utilisez pas Visual Studio (ou une version plus ancienne), ne vous inquiétez pas, le processus sera similaire sur la plupart des autres IDE.

## Construction (build) de GLFW
GLFW peut être obtenu à partir de la page de [téléchargement](http://www.glfw.org/download.html) de leur site web. GLFW dispose déjà de binaires précompilés et de fichiers d'en-tête pour Visual Studio 2012 jusqu'à 2019, mais par souci d'exhaustivité, nous allons compiler GLFW nous-mêmes à partir du code source. Ceci afin de vous donner une idée du processus de compilation des bibliothèques open-source, car toutes les bibliothèques ne disposent pas de binaires précompilés. Téléchargeons donc le paquet source.

> Nous compilerons toutes les bibliothèques sous forme de binaires 64 bits. Assurez-vous donc d'obtenir les binaires 64 bits si vous utilisez leurs binaires précompilés. 

 Une fois que vous avez téléchargé le paquet source, extrayez-le et ouvrez son contenu. Seuls quelques éléments nous intéressent :

- La bibliothèque résultant de la compilation.
- Le dossier include.

La compilation de la bibliothèque à partir du code source garantit que la bibliothèque résultante est parfaitement adaptée à votre CPU/OS, un luxe que les binaires précompilés n'offrent pas toujours (parfois, les binaires précompilés ne sont pas disponibles pour votre système). Le problème avec la mise à disposition du code source dans le monde ouvert est que tout le monde n'utilise pas le même IDE ou le même système de construction pour développer son application, ce qui signifie que les fichiers de projet/solution fournis peuvent ne pas être compatibles avec la configuration d'autres personnes. Les utilisateurs doivent alors configurer leur propre projet/solution avec les fichiers .c/.cpp et .h/.hpp fournis, ce qui est fastidieux. C'est précisément pour ces raisons qu'il existe un outil appelé **CMake**.

## CMake
CMake est un outil qui peut générer des fichiers de projet/solution au choix de l'utilisateur (par exemple Visual Studio, Code::Blocks, Eclipse) à partir d'une collection de fichiers de code source en utilisant des scripts CMake prédéfinis. Cela nous permet de générer un fichier de projet Visual Studio 2019 à partir du paquetage source de GLFW que nous pouvons utiliser pour compiler la bibliothèque. Tout d'abord, nous devons télécharger et installer CMake qui peut être téléchargé sur leur page de [téléchargement](http://www.cmake.org/cmake/resources/software.html).  
  
Une fois CMake installé, vous pouvez choisir d'exécuter CMake à partir de la ligne de commande ou via leur interface graphique. Comme nous n'essayons pas de compliquer les choses, nous allons utiliser l'interface graphique. CMake nécessite un dossier de code source et un dossier de destination pour les binaires. Pour le dossier du code source, nous allons choisir le dossier racine du paquetage source GLFW téléchargé et pour le dossier de build, nous allons créer un nouveau répertoire build et sélectionner ce répertoire.
![[img/window1.png]]
Une fois les dossiers source et destination définis, cliquez sur le bouton Configure pour que CMake puisse lire les paramètres requis et le code source. Nous devons ensuite choisir le générateur du projet et comme nous utilisons Visual Studio 2019, nous choisirons l'option Visual Studio 16 (**Visual Studio 2019 est également connu sous le nom de Visual Studio 16**). CMake affichera alors les options de construction possibles pour configurer la bibliothèque résultante. Nous pouvons les laisser à leurs valeurs par défaut et cliquer à nouveau sur Configure pour enregistrer les paramètres. Une fois les paramètres définis, nous cliquons sur Generate et les fichiers de projet résultants seront générés dans votre dossier de construction.

## Compilation
Dans le dossier de build, un fichier nommé `GLFW.sln` peut maintenant être trouvé et nous l'ouvrons avec Visual Studio 2019. Puisque CMake a généré un fichier de projet qui contient déjà les paramètres de configuration appropriés, nous n'avons plus qu'à construire la solution. CMake devrait avoir automatiquement configuré la solution pour qu'elle se compile en une bibliothèque 64 bits ; maintenant, appuyez sur build solution. Cela nous donnera un fichier de bibliothèque compilé qui peut être trouvé dans build/src/Debug nommé `glfw3.lib`.  
  
Une fois la bibliothèque générée, nous devons nous assurer que l'IDE sait où trouver la bibliothèque et les fichiers d'inclusion pour notre programme OpenGL. Il y a deux approches communes pour faire cela :

1. Nous trouvons les dossiers `/lib` et `/include` de l'IDE/compilateur et ajoutons le contenu du dossier `include` de GLFW au dossier `/include` de l'IDE et ajoutons de la même manière `glfw3.lib` au dossier `/lib` de l'IDE. Cela fonctionne, mais ce n'est pas l'approche recommandée. Il est difficile de garder une trace de votre bibliothèque et des fichiers inclus et une nouvelle installation de votre IDE/compilateur vous oblige à recommencer ce processus.  
2. Une autre approche (recommandée) consiste à créer un nouvel ensemble de répertoires à l'emplacement de votre choix, contenant tous les fichiers d'en-tête/librairies de tiers auxquels vous pouvez vous référer à partir de votre IDE/compilateur. Vous pouvez, par exemple, créer un seul dossier qui contient un dossier `Libs` et `Include` où nous stockons toutes nos bibliothèques et fichiers d'en-tête respectivement pour les projets OpenGL. Maintenant, toutes les bibliothèques tierces sont organisées en un seul endroit (qui peut être partagé entre plusieurs ordinateurs). Cependant, à chaque fois que nous créons un nouveau projet, nous devons indiquer à l'IDE où trouver ces répertoires.

 Une fois que les fichiers nécessaires sont stockés à l'emplacement de votre choix, nous pouvons commencer à créer notre premier projet OpenGL GLFW. 

## Notre premier projet
Tout d'abord, ouvrons Visual Studio et créons un nouveau projet. Choisissez C++ si plusieurs options sont proposées et prenez le projet vide (n'oubliez pas de donner un nom approprié à votre projet). Comme nous allons tout faire en 64 bits et que le projet est par défaut en 32 bits, nous devrons changer la liste déroulante en haut à côté de Debug de x86 à x64 :
![[img/window2.png]]
Une fois cela fait, nous avons maintenant un espace de travail pour créer notre toute première application OpenGL ! 

## Linking (création de liens)
Pour que le projet puisse utiliser GLFW, nous devons lier la bibliothèque à notre projet. Cela peut être fait en spécifiant que nous voulons utiliser `glfw3.lib` dans les paramètres de l'éditeur de liens, mais notre projet ne sait pas encore où trouver `glfw3.lib` puisque nous stockons nos bibliothèques tierces dans un répertoire différent. Nous devons donc d'abord ajouter ce répertoire au projet.
  
Nous pouvons dire à l'IDE de prendre en compte ce répertoire lorsqu'il doit rechercher des fichiers de bibliothèque et d'inclusion. Cliquez avec le bouton droit de la souris sur le nom du projet dans l'explorateur de solutions et allez ensuite dans VC++ Directories comme le montre l'image ci-dessous :
![[img/window3.png]]
À partir de là, vous pouvez ajouter vos propres répertoires pour indiquer au projet où chercher. Vous pouvez le faire en l'insérant manuellement dans le texte ou en cliquant sur la chaîne de caractères appropriée et en sélectionnant l'option `<Editer..>`. Procédez de la même manière pour les répertoires `Library` et `Include` : 
![[img/window4.png]]
Ici, vous pouvez ajouter autant de répertoires supplémentaires que vous le souhaitez et à partir de ce moment, l'IDE recherchera également ces répertoires lors de la recherche de bibliothèques et de fichiers d'en-tête. Dès que votre dossier Include de GLFW est inclus, vous pourrez trouver tous les fichiers d'en-tête pour GLFW en incluant . Il en va de même pour les répertoires de bibliothèques.  
  
Puisque VS peut maintenant trouver tous les fichiers nécessaires, nous pouvons enfin lier GLFW au projet en allant dans l'onglet `Linker` et `Input` :
![[img/window5.png]]
Pour lier à une bibliothèque, il faut spécifier le nom de la bibliothèque à l'éditeur de liens. Puisque le nom de la bibliothèque est `glfw3.lib`, nous l'ajoutons au champ Dépendances supplémentaires (soit manuellement, soit en utilisant l'option ) et à partir de là, GLFW sera lié lors de la compilation. En plus de GLFW, nous devrions également ajouter une entrée de lien vers la bibliothèque OpenGL, mais cela peut différer d'un système d'exploitation à l'autre :

### Bibliothèque OpenGL sous Windows
Si vous êtes sous Windows, la bibliothèque OpenGL `opengl32.lib` est fournie avec le SDK de Microsoft, qui est installé par défaut lorsque vous installez Visual Studio. Puisque ce chapitre utilise le compilateur VS et est sous Windows, nous ajoutons `opengl32.lib` aux paramètres de l'éditeur de liens. Notez que l'équivalent 64 bits de la bibliothèque OpenGL s'appelle `opengl32.lib`, tout comme l'équivalent 32 bits, ce qui est un nom un peu malheureux.

### Bibliothèque OpenGL sous Linux
Sur les systèmes Linux, vous devez vous lier à la bibliothèque `libGL.so` en ajoutant `-lGL` aux paramètres de votre éditeur de liens. Si vous ne trouvez pas cette bibliothèque, vous devez probablement installer l'un des paquets de développement Mesa, NVidia ou AMD.  
  
Ensuite, une fois que vous avez ajouté les bibliothèques GLFW et OpenGL aux paramètres de l'éditeur de liens, vous pouvez inclure les fichiers d'en-tête pour GLFW comme suit :
```cpp
#include <GLFW/glfw3.h>
```
>Pour les utilisateurs de Linux qui compilent avec GCC, les options de ligne de commande suivantes peuvent vous aider à compiler le projet : -lglfw3 -lGL -lX11 -lpthread -lXrandr -lXi -ldl. Le fait de ne pas lier correctement les bibliothèques correspondantes générera de nombreuses erreurs de référence non définie. 

 Ceci conclut l'installation et la configuration de GLFW. 

## GLAD
Nous ne sommes pas encore tout à fait au point, car il reste encore une chose à faire. Parce qu'OpenGL n'est qu'une norme/spécification, c'est au fabricant du pilote d'implémenter la spécification dans un pilote que la carte graphique supporte. Comme il existe de nombreuses versions différentes des pilotes OpenGL, l'emplacement de la plupart de ses fonctions n'est pas connu à la compilation et doit être interrogé à l'exécution. Il incombe alors au développeur de récupérer l'emplacement des fonctions dont il a besoin et de les stocker dans des pointeurs de fonction pour une utilisation ultérieure. La récupération de ces emplacements est spécifique au système d'exploitation. Sous Windows, cela ressemble à quelque chose comme ceci :
```cpp
// define the function's prototype
typedef void (*GL_GENBUFFERS) (GLsizei, GLuint*);
// find the function and assign it to a function pointer
GL_GENBUFFERS glGenBuffers  = (GL_GENBUFFERS)wglGetProcAddress("glGenBuffers");
// function can now be called as normal
unsigned int buffer;
glGenBuffers(1, &buffer);
```
Comme vous pouvez le constater, le code est complexe et il est fastidieux de le faire pour chaque fonction dont vous pourriez avoir besoin et qui n'est pas encore déclarée. Heureusement, il existe des bibliothèques à cet effet, dont GLAD, qui est une bibliothèque populaire et à jour. 
 
### Paramétrer GLAD
GLAD est une bibliothèque open source qui gère tout ce travail fastidieux dont nous avons parlé. GLAD a une configuration légèrement différente de la plupart des bibliothèques open source. GLAD utilise un service web qui permet d'indiquer à GLAD la version d'OpenGL que l'on souhaite définir et de charger toutes les fonctions OpenGL pertinentes en fonction de cette version.
  
Allez sur le [service web](http://glad.dav1d.de/) GLAD, assurez-vous que le langage est défini sur C++, et dans la section API sélectionnez une version d'OpenGL d'au moins 3.3 (c'est ce que nous utiliserons ; des versions plus élevées sont également acceptables). Assurez-vous également que le profil est défini sur Core et que l'option *Generate a loader* est cochée. Ignorez les extensions (pour l'instant) et cliquez sur *Generate* pour produire les fichiers de bibliothèque résultants.  
  
GLAD devrait maintenant vous avoir fourni un fichier zip contenant deux dossiers include, et un seul fichier `glad.c`. Copiez les deux dossiers `include` (`glad` et `KHR`) dans votre répertoire include(s) (ou ajoutez un élément supplémentaire pointant vers ces dossiers), et ajoutez le fichier `glad.c` à votre projet.  
  
Après les étapes précédentes, vous devriez être en mesure d'ajouter la directive include suivante au-dessus de votre fichier :
```cpp
#include <glad/glad.h> 
```
En appuyant sur le bouton de compilation, vous ne devriez pas avoir d'erreurs, et nous sommes prêts pour le prochain chapitre où nous verrons comment utiliser GLFW et GLAD pour configurer un contexte OpenGL et faire apparaître une fenêtre. Assurez-vous que tous vos répertoires include et library sont corrects et que les noms des librairies dans les paramètres de l'éditeur de liens correspondent aux librairies correspondantes.

## Ressources supplémentaires
- GLFW : [Window Guide](http://www.glfw.org/docs/latest/window_guide.html) : guide officiel de GLFW sur la mise en place et la configuration d'une fenêtre GLFW.  
- [Building applications](http://www.opengl-tutorial.org/miscellaneous/building-your-own-c-application/) : fournit d'excellentes informations sur le processus de compilation/liaison de votre application, ainsi qu'une longue liste d'erreurs possibles (avec leurs solutions).  
- [GLFW avec Code::Blocks](http://wiki.codeblocks.org/index.php?title=Using_GLFW_with_Code::Blocks) : construction de GLFW dans l'IDE Code::Blocks.  
- [Running CMake](http://www.cmake.org/runningcmake/) : bref aperçu de l'exécution de CMake sous Windows et Linux.  
- [Writing a build system under Linux](https://learnopengl.com/demo/autotools_tutorial.txt) : un tutoriel autotools par Wouter Verholst sur la façon d'écrire un système de compilation sous Linux.  
- [Polytonic/Glitter](https://github.com/Polytonic/Glitter) : un simple projet de base qui est préconfiguré avec toutes les bibliothèques pertinentes ; idéal si vous voulez un exemple de projet sans avoir à compiler toutes les bibliothèques vous-même.
