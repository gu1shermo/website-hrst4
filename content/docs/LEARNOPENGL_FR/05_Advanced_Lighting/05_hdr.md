# HDR
Les valeurs de luminosité et de couleur sont, par défaut, comprises entre $0.0$ et $1.0$ lorsqu'elles sont stockées dans un framebuffer. Cette déclaration, à première vue innocente, nous a poussés à toujours spécifier les valeurs de luminosité et de couleur quelque part dans cette fourchette, en essayant de les faire correspondre à la scène. Cela fonctionne bien et donne des résultats corrects, mais que se passe-t-il si nous marchons dans une zone très lumineuse avec plusieurs sources lumineuses dont la somme totale dépasse $1.0$ ? La réponse est que tous les fragments dont la somme de la luminosité ou de la couleur est supérieure à $1.0$ sont bloqués à $1.0$, ce qui n'est pas très joli à voir :
![05_hdr-20230903-hdr1.png](05_hdr-20230903-hdr1.png)
En raison du fait que les valeurs de couleur d'un grand nombre de fragments sont fixées à $1.0$, tous les fragments lumineux ont exactement la même valeur de couleur blanche dans de vastes régions, ce qui entraîne une perte importante de détails et donne un aspect faux.

Une solution à ce problème consisterait à réduire la puissance des sources lumineuses et à s'assurer qu'aucune zone de fragments de votre scène ne soit plus lumineuse que $1.0$ ; ce n'est pas une bonne solution car cela vous oblige à utiliser des paramètres d'éclairage irréalistes. **Une meilleure approche consiste à autoriser les valeurs de couleur à dépasser temporairement $1.0$ et à les ramener à la plage d'origine de $0.0$ et $1.0$ lors de la dernière étape, mais sans perdre de détails.**

**Les moniteurs (non HDR) sont limités à des couleurs comprises entre $0.0$ et $1.0$**, mais il n'y a pas de limitation de ce type dans les équations d'éclairage. En permettant aux couleurs des fragments de dépasser $1.0$, nous disposons d'une gamme de valeurs de couleurs beaucoup plus étendue, connue sous le nom de **gamme dynamique élevée (HDR). (High Dynamic Range)** Avec une gamme dynamique élevée, les choses claires peuvent être vraiment claires, les choses sombres peuvent être vraiment sombres, et les détails peuvent être vus dans les deux cas.

À l'origine, la plage dynamique élevée n'était utilisée que pour la photographie. Le photographe prenait plusieurs photos de la même scène avec différents niveaux d'exposition, capturant ainsi une large gamme de valeurs de couleur. En les combinant, on obtient une image HDR dans laquelle une large gamme de détails est visible en fonction des niveaux d'exposition combinés ou d'une exposition spécifique. 

Par exemple, l'image suivante (attribuée à Colin Smith) montre beaucoup de détails dans les zones très éclairées avec une faible exposition (regardez la fenêtre), mais ces détails disparaissent avec une forte exposition. Cependant, une exposition élevée révèle maintenant une grande quantité de détails dans les zones plus sombres qui n'étaient pas visibles auparavant.

![05_hdr-20230903-hdr2.png](05_hdr-20230903-hdr2.png)

Ce principe est également très proche du fonctionnement de l'œil humain et constitue la base du rendu de la gamme dynamique élevée. **Lorsqu'il y a peu de lumière, l'œil humain s'adapte pour que les parties sombres deviennent plus visibles, et il en va de même pour les zones lumineuses**. **C'est comme si l'œil humain disposait d'un curseur d'exposition automatique basé sur la luminosité de la scène.**

**Le rendu à plage dynamique élevée fonctionne un peu de la même manière.** Nous autorisons une gamme beaucoup plus large de valeurs de couleur pour le rendu, en collectant une large gamme de détails sombres et lumineux d'une scène, et à la fin nous transformons toutes les valeurs HDR en une gamme dynamique basse (LDR) de $[0,0, 1,0]$. **Ce processus de conversion des valeurs HDR en valeurs LDR est appelé "tone mapping"**. Il existe un grand nombre d'algorithmes de "tone mapping" qui visent à préserver la plupart des détails HDR au cours du processus de conversion. Ces algorithmes impliquent souvent un paramètre d'exposition qui favorise sélectivement les zones sombres ou lumineuses.

En ce qui concerne le rendu en temps réel, la plage dynamique élevée nous permet non seulement de dépasser la plage LDR de $[0,0, 1,0]$ et de préserver davantage de détails, mais aussi de spécifier l'intensité d'une source lumineuse en fonction de son intensité réelle. Par exemple, le soleil a une intensité beaucoup plus élevée qu'une lampe de poche, alors pourquoi ne pas configurer le soleil comme tel (par exemple, une luminosité diffuse de 100,0). Cela nous permet de configurer plus correctement l'éclairage d'une scène avec des paramètres d'éclairage plus réalistes, ce qui ne serait pas possible avec le rendu LDR car ils seraient alors directement clampés à $1.0$.

Comme les moniteurs (non HDR) n'affichent les couleurs que dans la plage comprise entre $0,0$ et $1,0$, nous devons retransformer la plage dynamique élevée des valeurs de couleur en fonction de la plage du moniteur. La simple retransformation des couleurs à l'aide d'une simple moyenne n'est pas très utile, car les zones plus claires deviennent alors beaucoup plus dominantes. **Ce que nous pouvons faire, c'est utiliser différentes équations et/ou courbes pour retransformer les valeurs HDR en LDR**, ce qui nous donne un contrôle total sur la luminosité de la scène. Il s'agit du processus appelé précédemment "tone mapping" et de l'étape finale du rendu HDR.

## Framebuffers à virgule flottante

Pour implémenter un rendu à haute gamme dynamique, nous avons besoin d'un moyen d'empêcher les valeurs de couleur d'être bloquées après chaque exécution du shader de fragment. Lorsque les framebuffers utilisent un format de couleur normalisé en virgule fixe (comme `GL_RGB`) comme format interne de leur tampon de couleur, OpenGL fixe automatiquement les valeurs entre $0.0$ et $1.0$ avant de les stocker dans le framebuffer. Cette opération est valable pour la plupart des formats de framebuffer, à l'exception des formats à virgule flottante.

Lorsque le format interne du tampon couleur d'un framebuffer est spécifié comme `GL_RGB16F`, `GL_RGBA16F`, `GL_RGB32F` ou `GL_RGBA32F`, le framebuffer est connu comme un framebuffer à virgule flottante qui peut stocker des valeurs à virgule flottante en dehors de la plage par défaut de $0.0$ et $1.0$. C'est parfait pour le rendu dans une plage dynamique élevée !

Pour créer un framebuffer à virgule flottante, la seule chose à modifier est le paramètre de format interne de son tampon de couleur :
```cpp
glBindTexture(GL_TEXTURE_2D, colorBuffer);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA16F, SCR_WIDTH, SCR_HEIGHT, 0, GL_RGBA, GL_FLOAT, NULL);
```
Le framebuffer par défaut d'OpenGL (par défaut) n'utilise que 8 bits par composante de couleur. Avec un framebuffer en virgule flottante de 32 bits par composante de couleur (en utilisant `GL_RGB32F` ou `GL_RGBA32F`), nous utilisons 4 fois plus de mémoire pour stocker les valeurs de couleur. Comme les 32 bits ne sont pas vraiment nécessaires (à moins que vous n'ayez besoin d'un haut niveau de précision), l'utilisation de `GL_RGBA16F` suffira.

Avec un tampon de couleurs en virgule flottante attaché à un framebuffer, nous pouvons maintenant effectuer le rendu de la scène dans ce framebuffer en sachant que les valeurs de couleurs ne seront pas bloquées entre $0.0$ et $1.0$. Dans l'exemple de démonstration de ce chapitre, nous rendons d'abord une scène éclairée dans le framebuffer à virgule flottante, puis nous affichons le tampon de couleurs du framebuffer sur un quad qui remplit l'écran ; cela ressemblera un peu à ceci :

```cpp
glBindFramebuffer(GL_FRAMEBUFFER, hdrFBO);
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);  
    // [...] render (lit) scene 
glBindFramebuffer(GL_FRAMEBUFFER, 0);

// now render hdr color buffer to 2D screen-filling quad with tone mapping shader
hdrShader.use();
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, hdrColorBufferTexture);
RenderQuad();
```
Ici, les valeurs de couleur d'une scène sont remplies dans un tampon de couleur à virgule flottante qui peut contenir n'importe quelle valeur de couleur arbitraire, éventuellement supérieure à $1.0$. Pour ce chapitre, une scène de démonstration simple a été créée avec un grand cube étiré agissant comme un tunnel avec quatre lumières ponctuelles, l'une d'entre elles étant extrêmement lumineuse et placée à l'extrémité du tunnel :

```cpp
std::vector<glm::vec3> lightColors;
lightColors.push_back(glm::vec3(200.0f, 200.0f, 200.0f));
lightColors.push_back(glm::vec3(0.1f, 0.0f, 0.0f));
lightColors.push_back(glm::vec3(0.0f, 0.0f, 0.2f));
lightColors.push_back(glm::vec3(0.0f, 0.1f, 0.0f));  
```
Le rendu dans un framebuffer à virgule flottante est exactement le même que le rendu normal dans un framebuffer. Ce qui est nouveau, c'est le fragment shader de `hdrShader` qui effectue le rendu du quad 2D final avec la texture du tampon de couleur en virgule flottante attachée. Définissons d'abord un simple fragment shader pass-through :
```cpp
#version 330 core
out vec4 FragColor;
  
in vec2 TexCoords;

uniform sampler2D hdrBuffer;

void main()
{             
    vec3 hdrColor = texture(hdrBuffer, TexCoords).rgb;
    FragColor = vec4(hdrColor, 1.0);
}
```
Ici, nous échantillonnons directement le tampon de couleur en virgule flottante et utilisons sa valeur de couleur comme sortie du shader de fragment. Cependant, comme la sortie du quad 2D est directement rendue dans le framebuffer par défaut, toutes les valeurs de sortie du shader de fragment resteront bloquées entre $0.0$ et $1.0$, même si plusieurs valeurs de la texture de couleur à virgule flottante dépassent $1.0$.
![05_hdr-20230903-hdr3.png](05_hdr-20230903-hdr3.png)
Il apparaît clairement que les valeurs de lumière intense au bout du tunnel sont bloquées à $1.0$ car une grande partie est complètement blanche, ce qui entraîne la perte de tous les détails de l'éclairage. Comme nous écrivons directement les valeurs HDR dans un tampon de sortie LDR, c'est comme si nous n'avions pas activé le HDR en premier lieu. Ce que nous devons faire, c'est transformer toutes les valeurs de couleur à virgule flottante dans la plage $0.0$ - $1.0$ sans perdre aucun détail. Nous devons appliquer un processus appelé "tone mapping".

## Tone mapping
La cartographie des tons est le processus de transformation des valeurs de couleur en virgule flottante vers la plage attendue $[0,0, 1,0]$ connue sous le nom de plage dynamique basse sans perdre trop de détails, souvent accompagnée d'une balance des couleurs stylistique spécifique.

L'un des algorithmes de tone mapping les plus simples est celui de **Reinhard**, qui consiste à diviser l'ensemble des valeurs de couleur HDR en valeurs de couleur LDR. L'algorithme de mapping des tons de Reinhard équilibre uniformément toutes les valeurs de luminosité sur LDR. Nous incluons le Reinhard tone mapping dans le fragment shader précédent et ajoutons également un filtre de correction gamma pour faire bonne mesure (y compris l'utilisation de textures sRGB) :

```cpp
void main()
{             
    const float gamma = 2.2;
    vec3 hdrColor = texture(hdrBuffer, TexCoords).rgb;
  
    // reinhard tone mapping
    vec3 mapped = hdrColor / (hdrColor + vec3(1.0));
    // gamma correction 
    mapped = pow(mapped, vec3(1.0 / gamma));
  
    FragColor = vec4(mapped, 1.0);
}
```
Avec l'application du mapping des tons de Reinhard, nous ne perdons plus aucun détail dans les zones lumineuses de notre scène. Elle a toutefois tendance à favoriser légèrement les zones lumineuses, ce qui fait que les zones plus sombres semblent moins détaillées et moins distinctes :

![05_hdr-20230903-hdr4.png](05_hdr-20230903-hdr4.png)
Ici, vous pouvez à nouveau voir les détails au bout du tunnel, car le motif de la texture du bois redevient visible. Grâce à cet algorithme relativement simple de mappage des tons, nous pouvons voir correctement toute la gamme des valeurs HDR stockées dans le tampon d'image en virgule flottante, ce qui nous permet de contrôler avec précision l'éclairage de la scène sans perdre de détails.

> Notez que nous pourrions également effectuer un tone map directement à la fin de notre shader d'éclairage, sans avoir besoin d'un framebuffer en virgule flottante ! Cependant, à mesure que les scènes deviennent plus complexes, vous aurez souvent besoin de stocker des résultats HDR intermédiaires dans des tampons à virgule flottante, c'est donc un bon exercice.

Une autre utilisation intéressante du tone mapping est de permettre l'utilisation d'un paramètre d'exposition. Vous vous souvenez probablement dans l'introduction que les images HDR contiennent de nombreux détails visibles à différents niveaux d'exposition. Si nous avons une scène qui présente un cycle jour/nuit, il est logique d'utiliser une exposition plus faible à la lumière du jour et une exposition plus élevée la nuit, de la même manière que l'œil humain s'adapte. Un tel paramètre d'exposition nous permet de configurer des paramètres d'éclairage qui fonctionnent aussi bien le jour que la nuit dans des conditions d'éclairage différentes, puisqu'il suffit de modifier le paramètre d'exposition.

Un algorithme relativement simple de mapping des tons d'exposition se présente comme suit :

```cpp
uniform float exposure;

void main()
{             
    const float gamma = 2.2;
    vec3 hdrColor = texture(hdrBuffer, TexCoords).rgb;
  
    // exposure tone mapping
    vec3 mapped = vec3(1.0) - exp(-hdrColor * exposure);
    // gamma correction 
    mapped = pow(mapped, vec3(1.0 / gamma));
  
    FragColor = vec4(mapped, 1.0);
}  
```
Ici, nous avons défini un uniforme d'exposition dont la valeur par défaut est $1.0$ et qui nous permet de spécifier plus précisément si nous souhaitons nous concentrer davantage sur les zones sombres ou claires des valeurs de couleur HDR. Par exemple, avec des valeurs d'exposition élevées, les zones sombres du tunnel sont nettement plus détaillées. À l'inverse, une exposition faible supprime en grande partie les détails des zones sombres, mais permet de voir plus de détails dans les zones lumineuses d'une scène. L'image ci-dessous montre le tunnel avec plusieurs niveaux d'exposition :
![05_hdr-20230903-hdr5.png](05_hdr-20230903-hdr5.png)
Cette image montre clairement les avantages d'un rendu à plage dynamique élevée. En changeant le niveau d'exposition, nous pouvons voir de nombreux détails de notre scène, qui auraient été perdus avec un rendu à faible plage dynamique. Prenons l'exemple de la fin du tunnel. Avec une exposition normale, la structure en bois est à peine visible, mais avec une faible exposition, les motifs en bois détaillés sont clairement visibles. Il en va de même pour les motifs en bois situés à proximité, qui sont plus visibles avec une exposition élevée.

Vous pouvez trouver le code source de la démo [ici](https://learnopengl.com/code_viewer_gh.php?code=src/5.advanced_lighting/6.hdr/hdr.cpp).

## Plus de HDR
**Les deux algorithmes de conversion des tons présentés ne sont que quelques-uns d'une vaste collection d'algorithmes de conversion des tons (plus avancés) dont chacun possède ses propres forces et faiblesses**. Certains algorithmes de conversion des tons favorisent certaines couleurs/intensités par rapport à d'autres et certains algorithmes affichent simultanément les couleurs à faible et à forte exposition afin de créer des images plus colorées et plus détaillées. **Il existe également un ensemble de techniques connues sous le nom d'ajustement automatique de l'exposition ou de techniques d'adaptation de l'œil qui déterminent la luminosité de la scène dans l'image précédente et adaptent (lentement) le paramètre d'exposition de sorte que la scène devienne plus lumineuse dans les zones sombres ou plus sombre dans les zones lumineuses, imitant ainsi l'œil humain.**

Le véritable avantage du rendu HDR apparaît vraiment dans les scènes vastes et complexes avec des algorithmes d'éclairage lourds. Comme il est difficile de créer une scène de démonstration aussi complexe à des fins d'enseignement tout en la gardant accessible, la scène de démonstration de ce chapitre est petite et manque de détails. Bien que relativement simple, elle montre certains des avantages du rendu HDR : aucun détail n'est perdu dans les zones hautes et sombres car elles peuvent être restaurées avec le tone mapping, l'ajout de plusieurs lumières n'entraîne pas de zones bloquées, et les valeurs de lumière peuvent être spécifiées par des valeurs de luminosité réelles sans être limitées par les valeurs LDR. **En outre, le rendu HDR rend plusieurs autres effets intéressants plus réalisables et plus réalistes ; l'un de ces effets est le bloom, dont nous parlerons dans le chapitre suivant.**

# Ressources supplémentaires
- [Le rendu HDR présente-t-il des avantages si le bloom n'est pas appliqué ?](http://gamedev.stackexchange.com/questions/62836/does-hdr-rendering-have-any-benefits-if-bloom-wont-be-applied) : une question de stackexchange qui comporte une longue réponse décrivant certains des avantages du rendu HDR.
- [Qu'est-ce que le tone mapping ? Quel est le rapport avec le HDR ?](http://photo.stackexchange.com/questions/7630/what-is-tone-mapping-how-does-it-relate-to-hdr) : une autre réponse intéressante avec de superbes images de référence pour expliquer le tone mapping.