# Ombrage différé
La méthode d'éclairage utilisée jusqu'à présent s'appelait "**forward rendering**" ou "**forward shading**". Il s'agit d'une approche simple qui consiste à effectuer le rendu d'un objet et à l'éclairer en fonction de toutes les sources lumineuses de la scène. Nous faisons cela pour chaque objet individuellement pour chaque objet de la scène. Bien qu'elle soit facile à comprendre et à mettre en œuvre, cette approche est également très coûteuse en termes de performances, car chaque objet rendu doit itérer sur chaque source de lumière pour chaque fragment rendu, ce qui est beaucoup ! Le forward rendering a également tendance à gaspiller beaucoup de shaders de fragments dans les scènes avec une grande complexité de profondeur (plusieurs objets couvrent le même pixel de l'écran) car les sorties des shaders de fragments sont remplacées par des sorties de shaders de fragments.

L'ombrage différé ou le rendu différé vise à surmonter ces problèmes en changeant radicalement la façon dont nous rendons les objets. Cela nous donne plusieurs nouvelles options pour optimiser de manière significative les scènes avec un grand nombre de lumières, nous permettant de rendre des centaines (ou même des milliers) de lumières avec un framerate acceptable. **L'image suivante est une scène avec 1847 lumières ponctuelles rendues avec un ombrage différé** (image fournie par Hannes Nevalainen) ; quelque chose qui ne serait pas possible avec un rendu direct.
![[07_deferred_shading-20230903-deferred1.png]]
L'ombrage différé est basé sur l'idée que nous différons ou reportons la majeure partie du rendu lourd (comme l'éclairage) à un stade ultérieur. L**'ombrage différé consiste en deux passes : dans la première passe, appelée passe géométrique, nous effectuons le rendu de la scène une fois et récupérons toutes sortes d'informations géométriques des objets que nous stockons dans une collection de textures appelée G-buffer ; pensez aux vecteurs de position, aux vecteurs de couleur, aux vecteurs de normalité et/ou aux valeurs spéculaires**. **Les informations géométriques d'une scène stockées dans le G-buffer sont ensuite utilisées pour des calculs d'éclairage (plus complexes)**. Voici le contenu du tampon G d'une seule image :
![[07_deferred_shading-20230903-deferred2.png]]
Nous utilisons les textures du tampon G dans une deuxième passe appelée **passe d'éclairage** où nous rendons un quad qui remplit l'écran et calculons l'éclairage de la scène pour chaque fragment en utilisant les informations géométriques stockées dans le tampon G ; pixel par pixel, nous itérons sur le tampon G. Au lieu de faire passer chaque objet du vertex shader au fragment shader, nous découplons les processus avancés de fragmentation à un stade ultérieur. Les calculs d'éclairage sont exactement les mêmes, mais cette fois-ci, nous prenons toutes les variables d'entrée nécessaires dans les textures correspondantes du G-buffer, au lieu du vertex shader (plus quelques variables uniformes).

L'image ci-dessous illustre bien le processus d'ombrage différé.

![[07_deferred_shading-20230903-deferred3.png]]

L'un des principaux avantages de cette approche est que, quel que soit le fragment qui se retrouve dans le tampon G, ce sont les informations relatives au fragment qui se retrouvent sous la forme d'un pixel à l'écran. Le test de profondeur a déjà conclu que ce fragment était le dernier et le plus élevé. Cela garantit que pour chaque pixel que nous traitons dans la passe d'éclairage, nous ne calculons l'éclairage qu'une seule fois. En outre, le rendu différé ouvre la voie à d'autres optimisations qui nous permettent de rendre un nombre beaucoup plus important de sources lumineuses par rapport au rendu direct.

Il présente également quelques inconvénients car le G-buffer nous oblige à stocker une quantité relativement importante de données de scène dans ses tampons de couleur de texture. Cela consomme de la mémoire, d'autant plus que les données de scène telles que les vecteurs de position nécessitent une grande précision. Un autre inconvénient est qu'il ne supporte pas le blending (car nous n'avons que des informations sur le fragment le plus haut) et que le MSAA ne fonctionne plus. Il existe plusieurs solutions de contournement que nous aborderons à la fin de ce chapitre.

Remplir le tampon G (dans la passe géométrique) n'est pas trop coûteux car nous stockons directement les informations de l'objet comme la position, la couleur ou les normales dans un framebuffer avec une quantité de traitement faible ou nulle. En utilisant des cibles de rendu multiples (MRT), nous pouvons même faire tout cela en une seule passe de rendu.

## Le G-Buffer
Le G-buffer est le terme collectif de toutes les textures utilisées pour stocker les données relatives à l'éclairage pour la passe d'éclairage finale. Prenons le temps de passer brièvement en revue toutes les données dont nous avons besoin pour éclairer un fragment avec le rendu direct :

- Un vecteur de **position** dans l'espace-monde 3D pour calculer la variable de position du fragment (interpolée) utilisée pour `lightDir` et `viewDir`.
- Un vecteur de **couleur** diffuse RVB également connu sous le nom d'**albédo**.
- Un vecteur **normal** 3D pour déterminer la pente d'une surface.
- Un vecteur **intensité spéculaire** flottant.
- Tous les vecteurs de position et de couleur des sources lumineuses.
- Le vecteur de position du joueur ou du spectateur.


Avec ces variables (par fragment) à notre disposition, nous sommes en mesure de calculer l'éclairage (Blinn-)Phong auquel nous sommes habitués. Les positions et les couleurs des sources de lumière, ainsi que la position de la vue du joueur, peuvent être configurées en utilisant des variables uniformes, mais les autres variables sont toutes spécifiques à un fragment. Si nous pouvons d'une manière ou d'une autre passer les mêmes données à la passe d'éclairage différée finale, nous pouvons calculer les mêmes effets d'éclairage, même si nous rendons des fragments d'un quad en 2D.

Il n'y a pas de limite dans OpenGL à ce que nous pouvons stocker dans une texture, il est donc logique de stocker toutes les données par fragment dans une ou plusieurs textures remplies à l'écran du G-buffer et de les utiliser plus tard dans la passe d'éclairage. Comme les textures du tampon G auront la même taille que le quad 2D de la passe d'éclairage, nous obtenons exactement les mêmes données de fragment que nous aurions eues dans le cadre d'un rendu direct, mais cette fois dans la passe d'éclairage ; il y a une correspondance un à un.

En pseudo-code, l'ensemble du processus ressemble à ceci :
```cpp
while(...) // render loop
{
    // 1. geometry pass: render all geometric/color data to g-buffer 
    glBindFramebuffer(GL_FRAMEBUFFER, gBuffer);
    glClearColor(0.0, 0.0, 0.0, 1.0); // keep it black so it doesn't leak into g-buffer
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
    gBufferShader.use();
    for(Object obj : Objects)
    {
        ConfigureShaderTransformsAndUniforms();
        obj.Draw();
    }  
    // 2. lighting pass: use g-buffer to calculate the scene's lighting
    glBindFramebuffer(GL_FRAMEBUFFER, 0);
    lightingPassShader.use();
    BindAllGBufferTextures();
    SetLightingUniforms();
    RenderQuad();
}
```
Les données que nous devrons stocker pour chaque fragment sont un vecteur de position, un vecteur de normale, un vecteur de couleur et une valeur d'intensité spéculaire. Dans la passe géométrique, nous devons effectuer le rendu de tous les objets de la scène et stocker ces composants de données dans le tampon G. Nous pouvons à nouveau utiliser des cibles de rendu multiples pour effectuer le rendu dans plusieurs tampons de couleur en une seule passe de rendu ; ceci a été brièvement discuté dans le chapitre sur le Bloom.

Pour la passe de géométrie, nous devons initialiser un objet framebuffer que nous appellerons `gBuffer`, auquel sont attachés plusieurs tampons de couleur et un seul objet `renderbuffer` de profondeur. Pour la position et la texture normale, nous utiliserons de préférence une texture de haute précision (16 ou 32 bits flottants par composant). Pour les valeurs d'albédo et de spéculaire, nous nous contenterons de la précision de texture par défaut (précision de 8 bits par composant). Notez que nous utilisons `GL_RGBA16F` plutôt que `GL_RGB16F` car les GPU préfèrent généralement les formats à 4 composantes aux formats à 3 composantes en raison de l'alignement des octets ; certains pilotes peuvent échouer à compléter le framebuffer dans le cas contraire.

```cpp
unsigned int gBuffer;
glGenFramebuffers(1, &gBuffer);
glBindFramebuffer(GL_FRAMEBUFFER, gBuffer);
unsigned int gPosition, gNormal, gColorSpec;
  
// - position color buffer
glGenTextures(1, &gPosition);
glBindTexture(GL_TEXTURE_2D, gPosition);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA16F, SCR_WIDTH, SCR_HEIGHT, 0, GL_RGBA, GL_FLOAT, NULL);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, gPosition, 0);
  
// - normal color buffer
glGenTextures(1, &gNormal);
glBindTexture(GL_TEXTURE_2D, gNormal);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA16F, SCR_WIDTH, SCR_HEIGHT, 0, GL_RGBA, GL_FLOAT, NULL);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT1, GL_TEXTURE_2D, gNormal, 0);
  
// - color + specular color buffer
glGenTextures(1, &gAlbedoSpec);
glBindTexture(GL_TEXTURE_2D, gAlbedoSpec);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, SCR_WIDTH, SCR_HEIGHT, 0, GL_RGBA, GL_UNSIGNED_BYTE, NULL);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT2, GL_TEXTURE_2D, gAlbedoSpec, 0);
  
// - tell OpenGL which color attachments we'll use (of this framebuffer) for rendering 
unsigned int attachments[3] = { GL_COLOR_ATTACHMENT0, GL_COLOR_ATTACHMENT1, GL_COLOR_ATTACHMENT2 };
glDrawBuffers(3, attachments);
  
// then also add render buffer object as depth buffer and check for completeness.
[...]
```

Puisque nous utilisons plusieurs cibles de rendu, nous devons explicitement indiquer à OpenGL lequel des tampons de couleur associés à `GBuffer` nous souhaitons utiliser pour le rendu avec `glDrawBuffers`. Il est également intéressant de noter que nous combinons les données de couleur et d'intensité spéculaire dans une seule texture RGBA ; cela nous évite d'avoir à déclarer une texture de tampon de couleur supplémentaire. Au fur et à mesure que votre pipeline d'ombrage différé devient plus complexe et nécessite plus de données, vous trouverez rapidement de nouvelles façons de combiner les données dans des textures individuelles.

Ensuite, nous devons effectuer le rendu dans le tampon G. En supposant que chaque objet possède une texture diffuse, normale et spéculaire, nous utiliserons quelque chose comme le fragment shader suivant pour effectuer le rendu dans le G-buffer :

```cpp
#version 330 core
layout (location = 0) out vec3 gPosition;
layout (location = 1) out vec3 gNormal;
layout (location = 2) out vec4 gAlbedoSpec;

in vec2 TexCoords;
in vec3 FragPos;
in vec3 Normal;

uniform sampler2D texture_diffuse1;
uniform sampler2D texture_specular1;

void main()
{    
    // store the fragment position vector in the first gbuffer texture
    gPosition = FragPos;
    // also store the per-fragment normals into the gbuffer
    gNormal = normalize(Normal);
    // and the diffuse per-fragment color
    gAlbedoSpec.rgb = texture(texture_diffuse1, TexCoords).rgb;
    // store specular intensity in gAlbedoSpec's alpha component
    gAlbedoSpec.a = texture(texture_specular1, TexCoords).r;
}
```
Comme nous utilisons plusieurs cibles de rendu, le spécificateur de disposition indique à OpenGL dans quel tampon de couleur du framebuffer actif nous effectuons le rendu. Notez que nous ne stockons pas l'intensité spéculaire dans une seule texture de tampon de couleur car nous pouvons stocker sa valeur flottante dans la composante alpha de l'une des autres textures de tampon de couleur.

> Gardez à l'esprit que pour les calculs d'éclairage, il est extrêmement important de conserver toutes les variables pertinentes dans le même espace de coordonnées. Dans ce cas, nous stockons (et calculons) toutes les variables dans l'espace monde.

Si nous devions maintenant rendre une grande collection d'objets sac à dos dans le framebuffer `gBuffer` et visualiser son contenu en projetant chaque tampon de couleur un par un sur un quad qui remplit l'écran, nous obtiendrions quelque chose comme ceci :
![[07_deferred_shading-20230904-deferred4.png]]
Essayez de visualiser que les vecteurs de position et de normalité de l'espace monde sont effectivement corrects. Par exemple, les vecteurs normaux pointant vers la droite seront plus alignés sur une couleur rouge, de même pour les vecteurs de position qui pointent de l'origine de la scène vers la droite. Dès que vous êtes satisfait du contenu du tampon G, il est temps de passer à l'étape suivante : la passe d'éclairage.

## La passe d'éclairage différée
Avec une large collection de données de fragments dans le G-Buffer à notre disposition, nous avons la possibilité de calculer complètement les couleurs éclairées finales de la scène. Pour ce faire, nous itérons sur chacune des textures du G-Buffer, pixel par pixel, et utilisons leur contenu comme entrée des algorithmes d'éclairage. Comme les valeurs des textures du tampon G représentent toutes les valeurs finales des fragments transformés, nous n'avons à effectuer les opérations d'éclairage coûteuses qu'une fois par pixel. Ceci est particulièrement utile dans les scènes complexes où nous pourrions facilement invoquer plusieurs appels coûteux de shaders de fragments par pixel dans un contexte de rendu direct.

Pour la passe d'éclairage, nous allons effectuer le rendu d'un quad 2D qui remplit l'écran (un peu comme un effet de post-traitement) et exécuter un shader de fragment d'éclairage coûteux sur chaque pixel :
```cpp
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, gPosition);
glActiveTexture(GL_TEXTURE1);
glBindTexture(GL_TEXTURE_2D, gNormal);
glActiveTexture(GL_TEXTURE2);
glBindTexture(GL_TEXTURE_2D, gAlbedoSpec);
// also send light relevant uniforms
shaderLightingPass.use();
SendAllLightUniformsToShader(shaderLightingPass);
shaderLightingPass.setVec3("viewPos", camera.Position);
RenderQuad();  
```
Nous lions toutes les textures pertinentes du tampon G avant le rendu et envoyons également les variables uniformes relatives à l'éclairage au shader.

Le fragment shader de la passe d'éclairage est largement similaire aux shaders de chapitre d'éclairage que nous avons utilisés jusqu'à présent. La nouveauté réside dans la méthode d'obtention des variables d'entrée de l'éclairage, que nous échantillonnons désormais directement à partir du G-buffer :
```cpp
#version 330 core
out vec4 FragColor;
  
in vec2 TexCoords;

uniform sampler2D gPosition;
uniform sampler2D gNormal;
uniform sampler2D gAlbedoSpec;

struct Light {
    vec3 Position;
    vec3 Color;
};
const int NR_LIGHTS = 32;
uniform Light lights[NR_LIGHTS];
uniform vec3 viewPos;

void main()
{             
    // retrieve data from G-buffer
    vec3 FragPos = texture(gPosition, TexCoords).rgb;
    vec3 Normal = texture(gNormal, TexCoords).rgb;
    vec3 Albedo = texture(gAlbedoSpec, TexCoords).rgb;
    float Specular = texture(gAlbedoSpec, TexCoords).a;
    
    // then calculate lighting as usual
    vec3 lighting = Albedo * 0.1; // hard-coded ambient component
    vec3 viewDir = normalize(viewPos - FragPos);
    for(int i = 0; i < NR_LIGHTS; ++i)
    {
        // diffuse
        vec3 lightDir = normalize(lights[i].Position - FragPos);
        vec3 diffuse = max(dot(Normal, lightDir), 0.0) * Albedo * lights[i].Color;
        lighting += diffuse;
    }
    
    FragColor = vec4(lighting, 1.0);
} 
```
Le shader de la passe d'éclairage accepte 3 textures uniformes qui représentent le tampon G et contiennent toutes les données que nous avons stockées dans la passe de géométrie. Si nous échantillonnions ces textures avec les coordonnées de la texture du fragment actuel, nous obtiendrions exactement les mêmes valeurs de fragment que si nous rendions la géométrie directement. Notez que nous récupérons à la fois la couleur albédo et l'intensité spéculaire à partir de la texture unique `gAlbedoSpec`.

Comme nous disposons maintenant des variables par fragment (et des variables uniformes correspondantes) nécessaires au calcul de l'éclairage Blinn-Phong, nous n'avons pas besoin de modifier le code d'éclairage. La seule chose que nous changeons ici dans l'ombrage différé est la méthode d'obtention des variables d'entrée de l'éclairage.

L'exécution d'une démo simple avec un total de 32 petites lumières ressemble un peu à ceci :
![[07_deferred_shading-20230904-deffered5.png]]
L'un des inconvénients de l'ombrage différé est qu'il n'est pas possible d'effectuer des blendings, car toutes les valeurs du tampon G proviennent de fragments uniques, et les mélanges opèrent sur la combinaison de fragments multiples. Un autre inconvénient est que l'ombrage différé vous oblige à utiliser le même algorithme d'éclairage pour la plupart des éclairages de votre scène ; vous pouvez pallier ce problème en incluant plus de données spécifiques aux matériaux dans le tampon G.

Pour surmonter ces inconvénients (en particulier le blending), nous divisons souvent le moteur de rendu en deux parties : une partie de rendu différé, et l'autre une partie de rendu direct spécifiquement destinée au mélange ou à des effets de shaders spéciaux qui ne conviennent pas à un pipeline de rendu différé. Pour illustrer comment cela fonctionne, nous rendrons les sources lumineuses sous forme de petits cubes en utilisant un moteur de rendu différé, car les cubes lumineux nécessitent un shader spécial (il suffit de produire une seule couleur de lumière).

## Combiner le rendu différé et le rendu anticipé
Supposons que nous voulions rendre chacune des sources de lumière sous la forme d'un cube 3D positionné à l'emplacement de la source de lumière et émettant la couleur de la lumière. La première idée qui nous vient à l'esprit est de simplement effectuer le rendu de toutes les sources de lumière sur le quad d'éclairage différé à la fin du pipeline d'ombrage différé. En fait, nous rendons les cubes comme nous le ferions normalement, mais seulement après avoir terminé les opérations de rendu différé. Dans le code, cela ressemblera un peu à ceci :
```cpp
// deferred lighting pass
[...]
RenderQuad();
  
// now render all light cubes with forward rendering as we'd normally do
shaderLightBox.use();
shaderLightBox.setMat4("projection", projection);
shaderLightBox.setMat4("view", view);
for (unsigned int i = 0; i < lightPositions.size(); i++)
{
    model = glm::mat4(1.0f);
    model = glm::translate(model, lightPositions[i]);
    model = glm::scale(model, glm::vec3(0.25f));
    shaderLightBox.setMat4("model", model);
    shaderLightBox.setVec3("lightColor", lightColors[i]);
    RenderCube();
}
```
Cependant, ces cubes rendus ne tiennent pas compte de la profondeur géométrique stockée dans le moteur de rendu différé et sont, par conséquent, toujours rendus au-dessus des objets rendus précédemment ; ce n'est pas le résultat que nous recherchions.
![[07_deferred_shading-20230904-deferred6.png]]
Ce que nous devons faire, c'est d'abord copier les informations de profondeur stockées dans la passe géométrique dans le tampon de profondeur du framebuffer par défaut et ensuite seulement rendre les cubes de lumière. De cette manière, les fragments des cubes de lumière ne sont rendus que lorsqu'ils se trouvent au-dessus de la géométrie rendue précédemment.

Nous pouvons copier le contenu d'un framebuffer dans le contenu d'un autre framebuffer à l'aide de `glBlitFramebuffer`, une fonction que nous avons également utilisée dans le chapitre sur l'anticrénelage pour résoudre les framebuffers multi-échantillonnés. La fonction `glBlitFramebuffer` nous permet de copier une région définie par l'utilisateur d'un framebuffer dans une région définie par l'utilisateur d'un autre framebuffer.

Nous avons stocké la profondeur de tous les objets rendus dans la passe de géométrie différée dans le FBO `gBuffer`. Si nous copions le contenu de son tampon de profondeur dans le tampon de profondeur du framebuffer par défaut, les cubes de lumière seraient alors rendus comme si toute la géométrie de la scène était rendue avec le rendu différé. Comme nous l'avons brièvement expliqué dans le chapitre sur l'[[anti aliasing]] (todo: link), nous devons spécifier un framebuffer en tant que framebuffer de lecture et spécifier de la même manière un framebuffer en tant que framebuffer d'écriture :

```cpp
glBindFramebuffer(GL_READ_FRAMEBUFFER, gBuffer);
glBindFramebuffer(GL_DRAW_FRAMEBUFFER, 0); // write to default framebuffer
glBlitFramebuffer(
  0, 0, SCR_WIDTH, SCR_HEIGHT, 0, 0, SCR_WIDTH, SCR_HEIGHT, GL_DEPTH_BUFFER_BIT, GL_NEAREST
);
glBindFramebuffer(GL_FRAMEBUFFER, 0);
// now render light cubes as before
[...]  
```
Ici, nous copions tout le contenu du tampon de profondeur du framebuffer lu dans le tampon de profondeur du framebuffer par défaut ; cela peut être fait de la même manière pour les tampons de couleur et les tampons de stencil. Si nous effectuons ensuite le rendu des cubes de lumière, les cubes s'affichent correctement sur la géométrie de la scène :

![[07_deferred_shading-20230904-deferred6-1.png]]
Vous pouvez trouver le code source complet de la démo [ici](https://learnopengl.com/code_viewer_gh.php?code=src/5.advanced_lighting/8.1.deferred_shading/deferred_shading.cpp).

Avec cette approche, nous pouvons facilement combiner l'ombrage différé avec l'ombrage direct. C'est une excellente chose car nous pouvons encore appliquer des mélanges et rendre des objets qui nécessitent des effets de shaders spéciaux, ce qui n'est pas possible dans un contexte de rendu différé pur.

## Un plus grand nombre de lumières
Ce pour quoi le rendu différé est souvent loué, c'est sa capacité à rendre une énorme quantité de sources lumineuses sans que cela ait un coût important en termes de performances. Le rendu différé en lui-même ne permet pas d'obtenir un très grand nombre de sources lumineuses, car nous devrions toujours calculer la composante d'éclairage de chaque fragment pour chacune des sources lumineuses de la scène. **Ce qui rend possible une grande quantité de sources lumineuses est une optimisation très soignée que nous pouvons appliquer au pipeline de rendu différé : celle des volumes de lumière.**

Normalement, lorsque nous effectuons le rendu d'un fragment dans une grande scène éclairée, nous calculons la contribution de chaque source lumineuse de la scène, quelle que soit sa distance par rapport au fragment. Une grande partie de ces sources lumineuses n'atteindra jamais le fragment, alors pourquoi gaspiller tous ces calculs d'éclairage ?

L'idée derrière les volumes de lumière est de calculer le rayon, ou le volume, d'une source de lumière, c'est-à-dire la zone où sa lumière est capable d'atteindre les fragments. Comme la plupart des sources lumineuses utilisent une certaine forme d'atténuation, nous pouvons l'utiliser pour calculer la distance maximale ou le rayon que leur lumière est capable d'atteindre. Nous n'effectuons alors les calculs d'éclairage coûteux que si un fragment se trouve à l'intérieur d'un ou de plusieurs de ces volumes de lumière. Cela peut nous permettre d'économiser une quantité considérable de calculs, car nous ne calculons plus l'éclairage que là où c'est nécessaire.

L'astuce de cette approche consiste principalement à déterminer la taille ou le rayon du volume de lumière d'une source lumineuse.

### Calculer le rayon ou le volume d'une lumière
Pour obtenir le rayon de volume d'une lumière, nous devons résoudre l'équation d'atténuation pour le moment où sa contribution lumineuse devient $0.0$. Pour la fonction d'atténuation, nous utiliserons la fonction introduite dans le chapitre sur les [[projecteurs de lumière]] (todo: link) :

$$
F_{light}
=
{
1
\over
{
K_c + K_l * d + K_q * d²
}
}
$$
Ce que nous voulons faire, c'est résoudre cette équation lorsque $F_{light}$ est égal à $0.0$. Cependant, cette équation n'atteindra jamais exactement la valeur $0.0$, et il n'y aura donc pas de solution. Ce que nous pouvons faire cependant, c'est ne pas résoudre l'équation pour $0.0$, mais la résoudre pour une valeur de luminosité qui est proche de $0.0$ mais qui est toujours perçue comme sombre. La valeur de luminosité de $5/256$ serait acceptable pour la scène de démonstration de ce chapitre ; divisée par 256 car le framebuffer 8 bits par défaut ne peut afficher que ce nombre d'intensités par composant.

>La fonction d'atténuation utilisée est principalement sombre dans sa plage visible. Si nous devions la limiter à une luminosité encore plus sombre que $5/256$, le volume de lumière deviendrait trop important et donc moins efficace. Tant qu'un utilisateur ne peut pas voir une coupure soudaine d'une source lumineuse à ses limites de volume, tout va bien. Bien sûr, cela dépend toujours du type de scène ; un seuil de luminosité plus élevé permet d'obtenir des volumes de lumière plus petits et donc une meilleure efficacité, mais peut produire des artefacts perceptibles lorsque l'éclairage semble se briser aux limites d'un volume.

L'équation d'atténuation que nous devons résoudre devient:
$$
{
5 \over 256
}
=
{
I_{max}
\over
Attenuation
}

$$

Ici, I_{max} est la composante de couleur la plus lumineuse de la source lumineuse. Nous utilisons la composante de couleur la plus brillante d'une source lumineuse pour résoudre l'équation de la valeur d'intensité la plus brillante d'une lumière qui reflète le mieux le rayon de volume lumineux idéal.

À partir de là, nous continuons à résoudre l'équation :

![[07_deferred_shading-20230904-deferred7.png]]

La dernière équation est une équation de la forme $ax^2+bx+c=0$, que nous pouvons résoudre à l'aide de l'équation quadratique :
![[07_deferred_shading-20230904-deferred8.png]]
Nous obtenons ainsi une équation générale qui nous permet de calculer x, c'est-à-dire le rayon du volume de lumière pour la source lumineuse, en fonction d'un paramètre constant, linéaire et quadratique :
```cpp
float constant  = 1.0; 
float linear    = 0.7;
float quadratic = 1.8;
float lightMax  = std::fmaxf(std::fmaxf(lightColor.r, lightColor.g), lightColor.b);
float radius    = 
  (-linear +  std::sqrtf(linear * linear - 4 * quadratic * (constant - (256.0 / 5.0) * lightMax))) 
  / (2 * quadratic);  
```
Nous calculons ce rayon pour chaque source lumineuse de la scène et l'utilisons pour calculer l'éclairage de cette source lumineuse uniquement si un fragment se trouve à l'intérieur du volume de la source lumineuse. Ci-dessous se trouve le fragment shader de la passe d'éclairage mis à jour qui prend en compte les volumes de lumière calculés. Notez que cette approche n'est utilisée qu'à des fins pédagogiques et n'est pas viable dans un contexte pratique, comme nous le verrons bientôt :
```cpp
struct Light {
    [...]
    float Radius;
}; 
  
void main()
{
    [...]
    for(int i = 0; i < NR_LIGHTS; ++i)
    {
        // calculate distance between light source and current fragment
        float distance = length(lights[i].Position - FragPos);
        if(distance < lights[i].Radius)
        {
            // do expensive lighting
            [...]
        }
    }   
}
```
Les résultats sont exactement les mêmes qu'auparavant, mais cette fois-ci, chaque lumière ne calcule l'éclairage que pour les sources lumineuses dans le volume où elle se trouve.

Vous pouvez trouver le code source final de la démo [ici](https://learnopengl.com/code_viewer_gh.php?code=src/5.advanced_lighting/8.2.deferred_shading_volumes/deferred_shading_volumes.cpp).

### Comment nous utilisons réellement les volumes de lumière

Le fragment shader présenté ci-dessus ne fonctionne pas vraiment en pratique et illustre seulement comment nous pouvons en quelque sorte utiliser le volume d'une lumière pour réduire les calculs d'éclairage. **La réalité est que votre GPU et GLSL sont assez mauvais pour optimiser les boucles et les branches. La raison en est que l'exécution des shaders sur le GPU est hautement parallèle et que la plupart des architectures ont l'obligation d'exécuter exactement le même code de shaders pour un grand nombre de threads afin d'être efficaces**. Cela signifie souvent qu'un shader qui exécute toutes les branches d'une instruction `if` pour s'assurer que l'exécution du shader est la même pour ce groupe de threads, ce qui rend notre précédente optimisation de vérification du rayon complètement inutile ; nous calculerions toujours l'éclairage pour toutes les sources de lumière !

L'approche appropriée pour utiliser les volumes de lumière consiste à rendre des sphères réelles, mises à l'échelle par le rayon du volume de lumière. Les centres de ces sphères sont positionnés à l'emplacement de la source de lumière, et lorsqu'elle est mise à l'échelle par le rayon du volume de lumière, la sphère englobe exactement le volume visible de la lumière. C'est là qu'intervient l'astuce : nous utilisons le shader d'éclairage différé pour le rendu des sphères. Comme une sphère rendue produit des invocations de shader de fragment qui correspondent exactement aux pixels affectés par la source de lumière, nous ne rendons que les pixels concernés et sautons tous les autres pixels. L'image ci-dessous l'illustre :
![[07_deferred_shading-20230904-deferred9.png]]
Cette opération est effectuée pour chaque source lumineuse de la scène, et les fragments résultants sont mélangés de manière additive. Le résultat est alors exactement la même scène que précédemment, mais cette fois-ci en ne rendant que les fragments pertinents par source lumineuse. Cela réduit effectivement les calculs de $nr_{objets} * nr_{lumières}$ à $nr_{objets} + nr_{lumières}$, ce qui le rend incroyablement efficace dans les scènes avec un grand nombre de lumières. C'est cette approche qui rend le rendu différé si adapté au rendu d'un grand nombre de lumières.

**Il y a encore un problème avec cette approche : le face culling doit être activé** (sinon nous rendrions l'effet d'une lumière deux fois) et lorsqu'il est activé, l'utilisateur peut entrer dans le volume d'une source de lumière après quoi le volume n'est plus rendu (à cause du back-face culling), supprimant l'influence de la source de lumière ; nous pouvons résoudre ce problème en ne rendant que les faces arrières des sphères.

Le rendu des volumes de lumière a un impact sur les performances, et bien qu'il soit généralement beaucoup plus rapide que l'ombrage différé normal pour le rendu d'un grand nombre de lumières, nous pouvons encore l'optimiser. Il existe deux autres extensions populaires (et plus efficaces) de l'ombrage différé : l'éclairage différé et l'ombrage différé basé sur les tuiles (?) (**deferred lighting** et **tile-based deferred shading**). Elles sont encore plus efficaces pour le rendu de grandes quantités de lumière et permettent également un MSAA relativement efficace.

## Rendu différé ou rendu direct
En soi (sans volumes de lumière), l'ombrage différé est une bonne optimisation car chaque pixel n'exécute qu'un seul shader de fragment, par rapport au rendu direct où le shader de fragment est souvent exécuté plusieurs fois par pixel. Le rendu différé présente cependant quelques inconvénients : une surcharge mémoire importante, pas de MSAA, et le mélange doit toujours être effectué avec le rendu direct.

Lorsque la scène est petite et qu'il n'y a pas trop de lumières, le rendu différé n'est pas nécessairement plus rapide, et parfois même plus lent, car la surcharge l'emporte alors sur les avantages du rendu différé. Dans les scènes plus complexes, le rendu différé devient rapidement une optimisation significative, en particulier avec les extensions d'optimisation les plus avancées. En outre, certains effets de rendu (en particulier les effets de post-traitement) deviennent moins coûteux avec un pipeline de rendu différé, car de nombreuses entrées de la scène sont déjà disponibles dans le tampon G.

Pour finir, j'aimerais mentionner que pratiquement tous les effets qui peuvent être réalisés avec le rendu direct peuvent également être implémentés dans un contexte de rendu différé ; cela ne nécessite souvent qu'une petite étape de traduction. Par exemple, si nous voulons utiliser le normal mapping dans un moteur de rendu différé, nous modifierons les shaders de la passe géométrique pour produire une normale à l'espace monde extraite d'une carte de normales (en utilisant une matrice TBN) au lieu de la normale à la surface ; les calculs d'éclairage dans la passe d'éclairage n'ont pas besoin d'être modifiés du tout. Et si vous voulez que le parallax mapping fonctionne, vous devez d'abord déplacer les coordonnées de texture dans la passe géométrique avant d'échantillonner les textures diffuses, spéculaires et normales d'un objet. Une fois que vous avez compris l'idée derrière le rendu différé, il n'est pas trop difficile de faire preuve de créativité.

# Ressources supplémentaires
- [Tutoriel 35 : Deferred Shading - Part 1](http://ogldev.atspace.co.uk/www/tutorial35/tutorial35.html) : un tutoriel en trois parties sur l'ombrage différé par OGLDev.
- [Deferred Rendering for Current and Future Rendering Pipelines](https://software.intel.com/sites/default/files/m/d/4/1/d/8/lauritzen_deferred_shading_siggraph_2010.pdf) : diapositives d'Andrew Lauritzen sur l'ombrage différé et l'éclairage différé de haut niveau basés sur les tuiles.

