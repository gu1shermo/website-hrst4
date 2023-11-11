# Bloom
Les sources lumineuses vives et les régions très éclairées sont souvent difficiles à faire comprendre à l'observateur, car la plage d'intensité d'un moniteur est limitée. L'un des moyens de distinguer les sources lumineuses vives sur un moniteur consiste à les faire briller ; la lumière s'étend alors autour de la source lumineuse. Cela donne au spectateur l'illusion que ces sources lumineuses ou ces zones lumineuses sont intensément lumineuses.

Cet effet de brillance/lueur (bleeding) est obtenu grâce à un effet de post-traitement appelé **Bloom**. Le bloom donne à toutes les zones fortement éclairées d'une scène un effet de brillance. Vous trouverez ci-dessous un exemple de scène avec et sans effet de lueur (avec l'aimable autorisation d'Epic Games) :
![06_bloom-20230903-bloom1.png](06_bloom-20230903-bloom1.png)
Le Bloom donne des indications visuelles notables sur la luminosité des objets. Lorsqu'il est utilisé de manière subtile (ce que certains jeux ne font absolument pas), le Bloom augmente de manière significative l'éclairage de votre scène et permet une large gamme d'effets dramatiques.

**Le Bloom fonctionne mieux en combinaison avec le rendu HDR**. On croit souvent à tort que le HDR est la même chose que Bloom, car de nombreuses personnes utilisent ces termes de manière interchangeable. Il s'agit pourtant de techniques complètement différentes utilisées à des fins différentes. Il est possible de mettre en œuvre le Bloom avec des framebuffers par défaut d'une précision de 8 bits, tout comme il est possible d'utiliser HDR sans l'effet Bloom. **C'est simplement que le HDR rend la mise en œuvre de Bloom plus efficace (comme nous le verrons plus tard).**

Pour mettre en œuvre l'effet Bloom, nous effectuons le rendu d'une scène éclairée comme d'habitude et nous extrayons à la fois le tampon de couleurs HDR de la scène et une image de la scène dont seules les zones lumineuses sont visibles. Cette image de luminosité extraite est ensuite floutée et le résultat est ajouté à l'image originale de la scène HDR.

Illustrons ce processus étape par étape. Nous rendons une scène remplie de 4 sources lumineuses, visualisées sous forme de cubes colorés. Les cubes lumineux colorés ont des valeurs de luminosité comprises entre $1.5$ et $15.0$. Si nous effectuons le rendu dans un tampon de couleurs HDR, la scène se présente comme suit :
![06_bloom-20230903-bloom2.png](06_bloom-20230903-bloom2.png)
Nous prenons cette texture tampon couleur HDR et extrayons tous les fragments qui dépassent une certaine luminosité. Nous obtenons ainsi une image qui ne montre que les régions de couleur vive dont l'intensité des fragments dépasse un certain seuil :
![06_bloom-20230903-bloom3.png](06_bloom-20230903-bloom3.png)
Nous prenons ensuite cette texture de luminosité seuillée et nous rendons le résultat flou. L'intensité de l'effet d'efflorescence (?) est largement déterminée par la portée et l'intensité du filtre de flou utilisé.
![06_bloom-20230903-bloom4.png](06_bloom-20230903-bloom4.png)
La texture floue qui en résulte est utilisée pour obtenir l'effet de lueur ou de brillance de la lumière. **Cette texture floue est ajoutée à la texture originale de la scène HDR**. **Comme les régions lumineuses sont étendues en largeur et en hauteur grâce au filtre flou, les régions lumineuses de la scène paraissent rayonnantes ou saignent (?) de lumière.**

![06_bloom-20230903-bloom5.png](06_bloom-20230903-bloom5.png)
Le Bloom n'est pas une technique compliquée en soi, mais il est difficile de l'utiliser à bon escient. La qualité visuelle est en grande partie déterminée par la qualité et le type de filtre de flou utilisé pour estomper les zones de luminosité extraites. Le simple fait de modifier le filtre de flou peut changer radicalement la qualité de l'effet Bloom.

En suivant ces étapes, nous obtenons l'effet de post-traitement Bloom. L'image suivante résume brièvement les étapes nécessaires à la mise en œuvre de l'effet Bloom :
![06_bloom-20230903-bloom6.png](06_bloom-20230903-bloom6.png)
La première étape consiste à extraire toutes les couleurs vives d'une scène en fonction d'un certain seuil. Voyons d'abord ce qu'il en est.

## Extraire les couleurs vives
La première étape consiste à extraire deux images d'une scène rendue. Nous pourrions rendre la scène deux fois, en effectuant le rendu dans un framebuffer différent avec des shaders différents, mais nous pouvons également utiliser une petite astuce appelée **Multiple Render Targets (MRT)** qui nous permet de spécifier plus d'une sortie de shader de fragment ; **cela nous donne la possibilité d'extraire les deux premières images en une seule passe de rendu**. En spécifiant un spécificateur d'emplacement de mise en page avant la sortie d'un shader de fragment, nous pouvons contrôler le tampon de couleur dans lequel le shader de fragment écrit :
```cpp
layout (location = 0) out vec4 FragColor;
layout (location = 1) out vec4 BrightColor; 
```
Cela ne fonctionne que si nous disposons de plusieurs tampons sur lesquels écrire. Pour pouvoir utiliser plusieurs sorties de fragment shader, nous avons besoin de plusieurs tampons de couleur attachés à l'objet framebuffer actuellement lié. Vous vous souvenez peut-être du chapitre sur les framebuffers qui nous permet de spécifier un numéro d'attachement de couleur lors de la liaison d'une texture en tant que tampon de couleur d'un framebuffer. Jusqu'à présent, nous avons toujours utilisé `GL_COLOR_ATTACHMENT0`, mais en utilisant également `GL_COLOR_ATTACHMENT1`, nous pouvons avoir deux tampons de couleur attachés à un objet framebuffer :
```cpp
// set up floating point framebuffer to render scene to
unsigned int hdrFBO;
glGenFramebuffers(1, &hdrFBO);
glBindFramebuffer(GL_FRAMEBUFFER, hdrFBO);
unsigned int colorBuffers[2];
glGenTextures(2, colorBuffers);
for (unsigned int i = 0; i < 2; i++)
{
    glBindTexture(GL_TEXTURE_2D, colorBuffers[i]);
    glTexImage2D(
        GL_TEXTURE_2D, 0, GL_RGBA16F, SCR_WIDTH, SCR_HEIGHT, 0, GL_RGBA, GL_FLOAT, NULL
    );
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
    // attach texture to framebuffer
    glFramebufferTexture2D(
        GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0 + i, GL_TEXTURE_2D, colorBuffers[i], 0
    );
}
```
Nous devons explicitement indiquer à OpenGL que nous effectuons le rendu vers plusieurs tampons de couleurs via `glDrawBuffers`. OpenGL, par défaut, n'effectue le rendu que sur la première couleur du framebuffer, ignorant toutes les autres. Nous pouvons le faire en passant un tableau d'enums d'attachement de couleur que nous aimerions rendre dans les opérations suivantes :
```cpp
unsigned int attachments[2] = { GL_COLOR_ATTACHMENT0, GL_COLOR_ATTACHMENT1 };
glDrawBuffers(2, attachments);  
```
Lors du rendu dans ce framebuffer, chaque fois qu'un fragment shader utilise le layout location specifier, le tampon de couleur correspondant est utilisé pour le rendu du fragment. C'est une bonne chose, car cela nous évite une passe de rendu supplémentaire pour extraire les régions lumineuses, puisque nous pouvons maintenant les extraire directement du fragment à rendre :
```cpp
#version 330 core
layout (location = 0) out vec4 FragColor;
layout (location = 1) out vec4 BrightColor;

[...]

void main()
{            
    [...] // first do normal lighting calculations and output results
    FragColor = vec4(lighting, 1.0);
    // check whether fragment output is higher than threshold, if so output as brightness color
    float brightness = dot(FragColor.rgb, vec3(0.2126, 0.7152, 0.0722));
    if(brightness > 1.0)
        BrightColor = vec4(FragColor.rgb, 1.0);
    else
        BrightColor = vec4(0.0, 0.0, 0.0, 1.0);
}
```
Ici, nous calculons d'abord l'éclairage comme d'habitude et le transmettons à la variable de sortie `FragColor` du premier shader de fragment. Ensuite, nous utilisons ce qui est actuellement stocké dans `FragColor` pour déterminer si sa luminosité dépasse un certain seuil. Nous calculons la luminosité d'un fragment en le transformant correctement en niveaux de gris (en prenant le produit scalaire des deux vecteurs, nous multiplions chaque composant individuel des deux vecteurs et ajoutons les résultats ensemble). Si la luminosité dépasse un certain seuil, nous envoyons la couleur dans le second tampon de couleur. Nous procédons de la même manière pour les cubes de lumière.

Cela montre également pourquoi Bloom fonctionne incroyablement bien avec le rendu HDR. Étant donné que nous effectuons le rendu en plage dynamique élevée, les valeurs de couleur peuvent dépasser $1.0$, ce qui nous permet de spécifier un seuil de luminosité en dehors de la plage par défaut, ce qui nous donne beaucoup plus de contrôle sur ce qui est considéré comme lumineux. Sans HDR, nous devrions définir un seuil inférieur à $1.0$, ce qui est toujours possible, mais les régions sont beaucoup plus rapidement considérées comme lumineuses. Cela conduit parfois à ce que l'effet de lueur devienne trop dominant (pensez à la neige blanche qui brille, par exemple).

Avec ces deux tampons de couleur, nous avons une image de la scène normale et une image des régions lumineuses extraites, le tout généré en une seule passe de rendu.

![06_bloom-20230903-bloom7.png](06_bloom-20230903-bloom7.png)
Avec une image des régions lumineuses extraites, nous devons maintenant flouter l'image. Nous pouvons le faire avec un simple filtre en boîte (?) comme nous l'avons fait dans la section post-traitement du chapitre sur les framebuffers, mais nous préférons utiliser un filtre de flou plus avancé (et plus esthétique) appelé **flou gaussien**.

## Flou gaussien
Dans le chapitre sur le flou du post-traitement, nous avons pris la moyenne de tous les pixels environnants d'une image. Bien que cette méthode permette d'obtenir un flou facile, elle ne donne pas les meilleurs résultats. **Un flou gaussien est basé sur la courbe gaussienne qui est communément décrite comme une courbe en forme de cloche donnant des valeurs élevées près de son centre qui s'estompent progressivement avec la distance. La courbe de Gauss peut être représentée mathématiquement sous différentes formes, mais elle a généralement la forme suivante :**
![06_bloom-20230903-bloom8.png](06_bloom-20230903-bloom8.png)
Comme la courbe gaussienne présente une zone plus large près de son centre, l'utilisation de ses valeurs en tant que poids pour rendre une image floue donne des résultats plus naturels, car les échantillons proches ont une plus grande priorité. Si, par exemple, nous échantillonnons une boîte de $32x32$ autour d'un fragment, nous utilisons des poids progressivement plus petits à mesure que la distance au fragment augmente ; cela donne un flou meilleur et plus réaliste, connu sous le nom de flou gaussien.

Pour mettre en œuvre un filtre de flou gaussien, nous avons besoin d'une boîte bidimensionnelle de poids que nous pouvons obtenir à partir d'une équation de courbe gaussienne bidimensionnelle. Le problème de cette approche est qu'elle devient rapidement très gourmande en performances. Prenons par exemple un kernel de flou de 32 par 32, cela nous obligerait à échantillonner une texture un total de 1024 fois pour chaque fragment !

Heureusement pour nous, **l'équation de Gauss possède une propriété très intéressante qui nous permet de séparer l'équation bidimensionnelle en deux équations unidimensionnelles plus petites : l'une qui décrit les poids horizontaux et l'autre qui décrit les poids verticaux**. Nous commençons par effectuer un flou horizontal avec les poids horizontaux sur la texture de la scène, puis nous effectuons un flou vertical sur la texture résultante. **Grâce à cette propriété, les résultats sont exactement les mêmes, mais cette fois-ci, nous économisons une quantité incroyable de performances puisque nous n'avons plus qu'à faire 32 + 32 échantillons au lieu de 1024 ! C'est ce qu'on appelle le flou gaussien à deux passages.**


![06_bloom-20230903-bloom9.png](06_bloom-20230903-bloom9.png)
Cela signifie que nous devons flouter une image au moins deux fois et cela fonctionne mieux avec l'utilisation d'objets framebuffer. Pour le flou gaussien à deux passages, nous allons implémenter des **framebuffers ping-pong**. Il s'agit d'une paire de framebuffers où nous effectuons le rendu et échangeons, un nombre donné de fois, le tampon de couleur de l'autre framebuffer dans le tampon de couleur du framebuffer actuel avec un effet de shader alternatif. En fait, nous changeons continuellement le framebuffer dans lequel nous effectuons le rendu et la texture avec laquelle nous dessinons. Cela nous permet d'abord de flouter la texture de la scène dans le premier framebuffer, puis de flouter le tampon de couleur du premier framebuffer dans le second framebuffer, puis le tampon de couleur du second framebuffer dans le premier, et ainsi de suite.

Avant de nous pencher sur les framebuffers, examinons d'abord le fragment shader du flou gaussien :

```cpp
#version 330 core
out vec4 FragColor;
  
in vec2 TexCoords;

uniform sampler2D image;
  
uniform bool horizontal;
uniform float weight[5] = float[] (0.227027, 0.1945946, 0.1216216, 0.054054, 0.016216);

void main()
{             
    vec2 tex_offset = 1.0 / textureSize(image, 0); // gets size of single texel
    vec3 result = texture(image, TexCoords).rgb * weight[0]; // current fragment's contribution
    if(horizontal)
    {
        for(int i = 1; i < 5; ++i)
        {
            result += texture(image, TexCoords + vec2(tex_offset.x * i, 0.0)).rgb * weight[i];
            result += texture(image, TexCoords - vec2(tex_offset.x * i, 0.0)).rgb * weight[i];
        }
    }
    else
    {
        for(int i = 1; i < 5; ++i)
        {
            result += texture(image, TexCoords + vec2(0.0, tex_offset.y * i)).rgb * weight[i];
            result += texture(image, TexCoords - vec2(0.0, tex_offset.y * i)).rgb * weight[i];
        }
    }
    FragColor = vec4(result, 1.0);
}
```
Ici, nous prenons un échantillon relativement petit de poids gaussiens que nous utilisons chacun pour attribuer un poids spécifique aux échantillons horizontaux ou verticaux autour du fragment actuel. Vous pouvez voir que nous divisons le filtre de flou en une section horizontale et une section verticale en fonction de la valeur que nous avons fixée pour l'uniforme horizontal. Nous basons la distance de décalage sur la taille exacte d'un texel obtenue par la division de $1.0$ sur la taille de la texture (un `vec2` de `textureSize`).

Pour rendre une image floue, nous créons deux framebuffers de base, chacun avec seulement une texture de tampon de couleur :

```cpp
unsigned int pingpongFBO[2];
unsigned int pingpongBuffer[2];
glGenFramebuffers(2, pingpongFBO);
glGenTextures(2, pingpongBuffer);
for (unsigned int i = 0; i < 2; i++)
{
    glBindFramebuffer(GL_FRAMEBUFFER, pingpongFBO[i]);
    glBindTexture(GL_TEXTURE_2D, pingpongBuffer[i]);
    glTexImage2D(
        GL_TEXTURE_2D, 0, GL_RGBA16F, SCR_WIDTH, SCR_HEIGHT, 0, GL_RGBA, GL_FLOAT, NULL
    );
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
    glFramebufferTexture2D(
        GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, pingpongBuffer[i], 0
    );
}
```
Ensuite, après avoir obtenu une texture HDR et une texture de luminosité extraite, nous remplissons d'abord l'un des framebuffers ping-pong avec la texture de luminosité, puis nous floutons l'image 10 fois (5 fois horizontalement et 5 fois verticalement) :
```cpp
bool horizontal = true, first_iteration = true;
int amount = 10;
shaderBlur.use();
for (unsigned int i = 0; i < amount; i++)
{
    glBindFramebuffer(GL_FRAMEBUFFER, pingpongFBO[horizontal]); 
    shaderBlur.setInt("horizontal", horizontal);
    glBindTexture(
        GL_TEXTURE_2D, first_iteration ? colorBuffers[1] : pingpongBuffers[!horizontal]
    ); 
    RenderQuad();
    horizontal = !horizontal;
    if (first_iteration)
        first_iteration = false;
}
glBindFramebuffer(GL_FRAMEBUFFER, 0); 
```

À chaque itération, nous lions l'un des deux framebuffers selon que nous voulons flouter horizontalement ou verticalement, et nous lions le tampon de couleur de l'autre framebuffer en tant que texture à flouter. Lors de la première itération, nous lions spécifiquement la texture que nous souhaitons rendre floue (`brightnessTexture`), car les deux tampons de couleur seraient alors vides. En répétant ce processus 10 fois, l'image de luminosité se retrouve avec un flou gaussien complet qui a été répété 5 fois. Cette construction nous permet de flouter n'importe quelle image aussi souvent que nous le souhaitons ; plus le nombre d'itérations de flou gaussien est élevé, plus le flou est prononcé.

En floutant 5 fois la texture de luminosité extraite, nous obtenons une image correctement floutée de toutes les régions lumineuses d'une scène.

![06_bloom-20230903-blur10.png](06_bloom-20230903-blur10.png)
La dernière étape pour compléter l'effet Bloom consiste à combiner cette texture de luminosité floue avec la texture HDR de la scène originale.

## Combiner (mélanger) les 2 textures
Avec la texture HDR de la scène et une texture de luminosité floue de la scène, il suffit de combiner les deux pour obtenir le fameux effet **Bloom** ou **glow**. Dans le fragment shader final (largement similaire à celui que nous avons utilisé dans le chapitre HDR), nous mélangeons les deux textures de manière additive :
```cpp
#version 330 core
out vec4 FragColor;
  
in vec2 TexCoords;

uniform sampler2D scene;
uniform sampler2D bloomBlur;
uniform float exposure;

void main()
{             
    const float gamma = 2.2;
    vec3 hdrColor = texture(scene, TexCoords).rgb;      
    vec3 bloomColor = texture(bloomBlur, TexCoords).rgb;
    hdrColor += bloomColor; // additive blending
    // tone mapping
    vec3 result = vec3(1.0) - exp(-hdrColor * exposure);
    // also gamma correct while we're at it       
    result = pow(result, vec3(1.0 / gamma));
    FragColor = vec4(result, 1.0);
}  
```
Il est intéressant de noter que nous ajoutons l'effet Bloom avant d'appliquer le tone mapping. De cette façon, la luminosité ajoutée par l'effet Bloom est également transformée en douceur en plage LDR, ce qui permet d'obtenir un meilleur éclairage relatif.

Avec les deux textures ajoutées ensemble, toutes les zones lumineuses de notre scène bénéficient maintenant d'un effet d'éclat approprié :
![06_bloom-20230903-bloom11.png](06_bloom-20230903-bloom11.png)
Les cubes colorés apparaissent maintenant beaucoup plus lumineux et donnent une meilleure illusion d'objets émettant de la lumière. Il s'agit d'une scène relativement simple et l'effet Bloom n'est donc pas très impressionnant, mais dans des scènes bien éclairées, il peut faire une différence significative lorsqu'il est correctement configuré. Vous pouvez trouver le code source de cette démo simple [ici](https://learnopengl.com/code_viewer_gh.php?code=src/5.advanced_lighting/7.bloom/bloom.cpp).

Pour ce chapitre, nous avons utilisé un filtre de flou gaussien relativement simple dans lequel nous ne prenons que 5 échantillons dans chaque direction. En prenant plus d'échantillons sur un plus grand rayon ou en répétant le filtre de flou un nombre supplémentaire de fois, nous pouvons améliorer l'effet de flou. La qualité du flou étant directement liée à la qualité de l'effet de Bloom, l'amélioration de l'étape de flou peut apporter une amélioration significative. Certaines de ces améliorations combinent des filtres de flou avec des noyaux de flou de taille variable ou utilisent plusieurs courbes gaussiennes pour combiner sélectivement les poids. Les ressources supplémentaires de Kalogirou et Epic Games expliquent comment améliorer de manière significative l'effet Bloom en améliorant le flou gaussien.

# Ressources supplémentaires
- [Efficient Gaussian Blur with linear sampling](http://rastergrid.com/blog/2010/09/efficient-gaussian-blur-with-linear-sampling/) : décrit très bien le flou gaussien et comment améliorer ses performances en utilisant l'échantillonnage de texture bilinéaire d'OpenGL.
- [Bloom Post Process Effect](https://udk-legacy.unrealengine.com/udk/Three/Bloom.html) : article d'Epic Games sur l'amélioration de l'effet Bloom en combinant plusieurs courbes gaussiennes pour ses poids.
- [How to do good Bloom for HDR rendering](http://kalogirou.net/2006/05/20/how-to-do-good-bloom-for-hdr-rendering/) : article de Kalogirou qui décrit comment améliorer l'effet Bloom en utilisant une meilleure méthode de flou gaussien.
