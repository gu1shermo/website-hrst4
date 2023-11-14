---
title: "opengl"
date: 2023-11-11
author: hrst4
tags: ['cg','opengl','graphics','cpp']
draft: false
---
# OpenGL
Avant de commencer notre voyage, nous devons d'abord définir ce qu'est OpenGL. OpenGL est principalement considéré comme une **API (une interface de programmation d'applications) qui nous fournit un large ensemble de fonctions que nous pouvons utiliser pour manipuler les graphiques et les images**. **Cependant, OpenGL en lui-même n'est pas une API, mais simplement une spécification, développée et maintenue par le groupe Khronos.**

La spécification OpenGL précise exactement le résultat/la sortie de chaque fonction et la manière dont elle doit fonctionner. Il appartient ensuite aux développeurs qui mettent en œuvre cette spécification de trouver une solution à la manière dont cette fonction doit fonctionner. Puisque la spécification OpenGL ne donne pas de détails sur l'implémentation, les versions développées d'OpenGL sont autorisées à avoir des implémentations différentes, tant que leurs résultats sont conformes à la spécification (et donc identiques pour l'utilisateur).

**Les personnes qui développent les bibliothèques OpenGL sont généralement les fabricants de cartes graphiques**. Chaque carte graphique que vous achetez prend en charge des versions spécifiques d'OpenGL qui sont les versions d'OpenGL développées spécifiquement pour cette carte (série). Lorsque vous utilisez un système Apple, la bibliothèque OpenGL est maintenue par Apple lui-même et sous Linux, il existe une combinaison de versions de fournisseurs graphiques et d'adaptations de ces bibliothèques par des amateurs. Cela signifie également que chaque fois qu'OpenGL montre un comportement bizarre qu'il ne devrait pas avoir, c'est très probablement la faute des fabricants de cartes graphiques (ou de ceux qui ont développé/maintenu la bibliothèque).

>Comme la plupart des implémentations sont conçues par les fabricants de cartes graphiques, chaque fois qu'il y a un bogue dans l'implémentation, il est généralement résolu par la mise à jour des pilotes de votre carte vidéo ; ces pilotes incluent les versions les plus récentes d'OpenGL que votre carte supporte. C'est l'une des raisons pour lesquelles il est toujours conseillé de mettre à jour de temps en temps les pilotes de votre carte graphique.

Khronos héberge publiquement tous les documents de spécification pour toutes les versions d'OpenGL. Le lecteur intéressé peut trouver la spécification OpenGL de la version 3.3 (qui est celle que nous utiliserons) ici. C'est une bonne lecture si vous voulez plonger dans les détails d'OpenGL (notez qu'ils décrivent principalement les résultats et non les implémentations). Les spécifications fournissent également une excellente référence pour trouver le fonctionnement exact de ses fonctions.

## Core-profile vs Immediate mode (Profil de base vs mode immédiat)

Autrefois, utiliser OpenGL signifiait développer en mode **immédiat** (souvent appelé pipeline de fonctions fixes (fixed function pipeline)), ce qui était une méthode facile à utiliser pour dessiner des graphiques. La plupart des fonctionnalités d'OpenGL étaient cachées dans la bibliothèque et les développeurs n'avaient pas beaucoup de contrôle sur la manière dont OpenGL effectuait ses calculs. Les développeurs ont fini par avoir envie de plus de flexibilité et, avec le temps, les spécifications sont devenues plus flexibles, ce qui a permis aux développeurs d'avoir un meilleur contrôle sur leurs graphiques. **Le mode immédiat est vraiment facile à utiliser et à comprendre, mais il est aussi extrêmement inefficace**. **C'est pourquoi la spécification a commencé à déprécier les fonctionnalités du mode immédiat à partir de la version 3.2 et a commencé à motiver les développeurs à développer en mode core-profile d'OpenGL, qui est une division de la spécification d'OpenGL qui a supprimé toutes les anciennes fonctionnalités dépréciées.**

En utilisant le profil de base d'OpenGL (core-profile), OpenGL nous force à utiliser des pratiques modernes. Chaque fois que nous essayons d'utiliser l'une des fonctions obsolètes d'OpenGL, OpenGL lève une erreur et arrête de dessiner. **L'avantage d'apprendre l'approche moderne est qu'elle est très flexible et efficace**. **Cependant, elle est aussi plus difficile à apprendre**. Le mode immédiat faisait abstraction de beaucoup d'opérations réelles effectuées par OpenGL et bien qu'il soit facile à apprendre, il était difficile de comprendre comment OpenGL fonctionnait réellement. L'approche moderne exige du développeur qu'il comprenne vraiment OpenGL et la programmation graphique et, bien que cela soit un peu difficile, cela permet une plus grande flexibilité, une plus grande efficacité et, plus important encore, une bien meilleure compréhension de la programmation graphique.

C'est également la raison pour laquelle ce livre est axé sur la version 3.3 du profil central d'OpenGL (core profile). Bien qu'elle soit plus difficile, l'effort en vaut la peine.  
  
A partir d'aujourd'hui, des versions supérieures d'OpenGL sont disponibles (au moment où j'écris ces lignes, la 4.6), ce qui peut vous amener à vous demander : **pourquoi vouloir apprendre OpenGL 3.3 alors qu'OpenGL 4.6 est déjà disponible ? La réponse à cette question est relativement simple. Toutes les versions futures d'OpenGL à partir de la 3.3 ajoutent des fonctionnalités supplémentaires utiles à OpenGL sans changer les mécanismes de base d'OpenGL ; les versions plus récentes introduisent simplement des moyens légèrement plus efficaces ou plus utiles pour accomplir les mêmes tâches. Le résultat est que tous les concepts et techniques restent les mêmes dans les versions modernes d'OpenGL, il est donc parfaitement valable d'apprendre OpenGL 3.3.** Lorsque vous serez prêt et/ou plus expérimenté, vous pourrez facilement utiliser les fonctionnalités spécifiques des versions plus récentes d'OpenGL.

> Lorsque vous utilisez les fonctionnalités de la version la plus récente d'OpenGL, seules les cartes graphiques les plus modernes pourront faire fonctionner votre application. C'est souvent la raison pour laquelle la plupart des développeurs ciblent généralement les versions inférieures d'OpenGL et activent éventuellement les fonctionnalités des versions supérieures. 

 Dans certains chapitres, vous trouverez des caractéristiques plus modernes qui sont notées comme telles. 

## Extensions
L'une des grandes caractéristiques d'OpenGL est son support des extensions. Chaque fois qu'une société graphique propose une nouvelle technique ou une nouvelle optimisation importante pour le rendu, elle le fait souvent dans une extension implémentée dans les pilotes. Si le matériel sur lequel tourne une application supporte une telle extension, le développeur peut utiliser la fonctionnalité fournie par l'extension pour des graphiques plus avancés ou plus efficaces. De cette manière, un développeur graphique peut toujours utiliser ces nouvelles techniques de rendu sans avoir à attendre qu'OpenGL inclue la fonctionnalité dans ses futures versions, en vérifiant simplement si l'extension est supportée par la carte graphique. Souvent, lorsqu'une extension est populaire ou très utile, elle finit par être intégrée dans les futures versions d'OpenGL.  
  
Le développeur doit demander si l'une de ces extensions est disponible avant de l'utiliser (ou utiliser une bibliothèque d'extensions OpenGL). Cela permet au développeur de faire les choses mieux ou plus efficacement, en fonction de la disponibilité de l'extension :

```cpp
if(GL_ARB_extension_name)
{
    // Do cool new and modern stuff supported by hardware
}
else
{
    // Extension not supported: do it the old way
}
```
Avec OpenGL version 3.3, nous avons rarement besoin d'une extension pour la plupart des techniques, mais lorsque cela est nécessaire, des instructions appropriées sont fournies. 

## State machine (machine à états)
OpenGL est en soi une grande machine à états : une collection de variables qui définissent comment OpenGL doit fonctionner. **L'état d'OpenGL est communément appelé le contexte OpenGL**. Lorsque nous utilisons OpenGL, nous modifions souvent son état en réglant certaines options, en manipulant certains tampons, puis en effectuant le rendu en utilisant le contexte actuel.  
  
Chaque fois que nous disons à OpenGL que nous voulons dessiner des lignes au lieu de triangles par exemple, nous changeons l'état d'OpenGL en changeant une variable de contexte qui définit comment OpenGL doit dessiner. Dès que nous changeons le contexte en disant à OpenGL qu'il doit dessiner des lignes, les prochaines commandes de dessin dessineront des lignes au lieu de triangles.  
  
En travaillant avec OpenGL, nous rencontrerons plusieurs fonctions de changement d'état qui modifient le contexte et plusieurs fonctions d'utilisation d'état qui effectuent des opérations basées sur l'état actuel d'OpenGL. Tant que vous gardez à l'esprit qu'OpenGL est fondamentalement une grande machine à états, la plupart de ses fonctionnalités auront plus de sens.

## Objets
Les bibliothèques OpenGL sont écrites en C et permettent de nombreuses dérivations dans d'autres langages, mais elles restent essentiellement des bibliothèques en C. Comme beaucoup de constructions du langage C ne se traduisent pas très bien dans d'autres langages de plus haut niveau, OpenGL a été développé avec plusieurs abstractions à l'esprit. L'une de ces abstractions est l'objet dans OpenGL.  
  
Un objet OpenGL est une collection d'options qui représente un sous-ensemble de l'état d'OpenGL. Par exemple, nous pourrions avoir un objet qui représente les paramètres de la fenêtre de dessin ; nous pourrions alors définir sa taille, le nombre de couleurs qu'elle supporte et ainsi de suite. On peut visualiser un objet comme une structure de type C :
```cpp
struct object_name {
    float  option1;
    int    option2;
    char[] name;
};
```
 Lorsque nous voulons utiliser des objets, cela ressemble généralement à quelque chose comme ceci (avec le contexte d'OpenGL visualisé comme une grande structure) : 
 ```cpp
// The State of OpenGL
struct OpenGL_Context {
  	...
  	object_name* object_Window_Target;
  	...  	
};
```
```cpp
// create object
unsigned int objectId = 0;
glGenObject(1, &objectId);
// bind/assign object to context
glBindObject(GL_WINDOW_TARGET, objectId);
// set options of object currently bound to GL_WINDOW_TARGET
glSetObjectOption(GL_WINDOW_TARGET, GL_OPTION_WINDOW_WIDTH,  800);
glSetObjectOption(GL_WINDOW_TARGET, GL_OPTION_WINDOW_HEIGHT, 600);
// set context target back to default
glBindObject(GL_WINDOW_TARGET, 0);
```
Ce petit morceau de code est un flux de travail que vous verrez fréquemment lorsque vous travaillez avec OpenGL.
- Nous créons d'abord un objet et stockons une référence à cet objet sous la forme d'un identifiant (les données de l'objet réel sont stockées en coulisses).
- Ensuite, nous lions l'objet (en utilisant son identifiant) à l'emplacement cible du contexte (l'emplacement de l'objet cible de la fenêtre de l'exemple est défini comme GL_WINDOW_TARGET).
- Les options définies sont stockées dans l'objet référencé par objectId et restaurées dès que l'objet est à nouveau lié à GL_WINDOW_TARGET.

> Les exemples de code fournis jusqu'à présent ne sont que des approximations du fonctionnement d'OpenGL ; tout au long du livre, vous rencontrerez suffisamment d'exemples réels.

L'avantage d'utiliser ces objets est que nous pouvons définir plus d'un objet dans notre application, définir leurs options et chaque fois que nous démarrons une opération qui utilise l'état d'OpenGL, nous lions l'objet avec nos paramètres préférés. Il existe par exemple des objets qui agissent comme des objets conteneurs pour les données d'un modèle 3D (une maison ou un personnage) et chaque fois que nous voulons dessiner l'un d'entre eux, nous lions l'objet contenant les données du modèle que nous voulons dessiner (nous avons d'abord créé et défini les options de ces objets). Le fait d'avoir plusieurs objets nous permet de spécifier de nombreux modèles et lorsque nous voulons dessiner un modèle spécifique, nous lions simplement l'objet correspondant avant de dessiner sans avoir à définir à nouveau toutes leurs options.

## Commençons
Vous avez maintenant appris un peu sur OpenGL en tant que spécification et bibliothèque, comment OpenGL fonctionne approximativement sous le capot et quelques astuces personnalisées qu'OpenGL utilise. Ne vous inquiétez pas si vous n'avez pas tout compris ; tout au long du livre, nous passerons en revue chaque étape et vous verrez suffisamment d'exemples pour vraiment comprendre OpenGL. 

## Ressources
- [opengl.org](https://www.opengl.org/): site officiel d'OpenGL.
- [OpenGL registry](https://www.opengl.org/registry/): héberge les spécifications et les extensions OpenGL pour toutes les versions d'OpenGL.