# Shadow mapping
Les ombres résultent de l'absence de lumière due à une occlusion. Lorsque les rayons lumineux d'une source de lumière n'atteignent pas un objet parce qu'il est occulté par un autre objet, l'objet est dans l'ombre. Les ombres ajoutent beaucoup de réalisme à une scène éclairée et permettent au spectateur d'observer plus facilement les relations spatiales entre les objets. Elles donnent une plus grande impression de profondeur à la scène et aux objets. Par exemple, regardez l'image suivante d'une scène avec et sans ombres :
![[shadow_mapping_with_without.png]]

Vous pouvez constater qu'avec les ombres, la relation entre les objets devient beaucoup plus évidente. Par exemple, le fait que l'un des cubes flotte au-dessus des autres n'est vraiment perceptible que lorsqu'il y a des ombres.

Les ombres sont cependant un peu difficiles à mettre en œuvre, notamment parce que la recherche actuelle sur le temps réel (graphiques rastérisés) n'a pas encore mis au point un algorithme d'ombres parfait. Il existe plusieurs bonnes techniques d'approximation des ombres, mais elles ont toutes leurs particularités et leurs inconvénients que nous devons prendre en compte.

L'une des techniques utilisées par la plupart des jeux vidéo, qui donne des résultats satisfaisants et qui est relativement facile à mettre en œuvre, est le **shadow mapping**. Le shadow mapping n'est pas trop difficile à comprendre, ne coûte pas trop cher en performance et s'étend facilement à des algorithmes plus avancés (comme les **Omnidirectional Shadow Maps** et les **Cascaded Shadow Maps**).

## Shadow mapping
L'idée derrière le shadow mapping est assez simple : nous rendons la scène du point de vue de la lumière et tout ce que nous voyons du point de vue de la lumière est éclairé et tout ce que nous ne pouvons pas voir doit être dans l'ombre. Imaginez une section de plancher avec une grande caisse entre elle et une source de lumière. Étant donné que la source lumineuse verra cette boîte et non la section du sol lorsqu'elle regardera dans sa direction, cette section spécifique du sol devrait être dans l'ombre.
![[shadow_mapping_theory.png]]

Ici, toutes les lignes bleues représentent les fragments que la source lumineuse peut voir. Les fragments occultés sont représentés par des lignes noires : ils sont rendus comme étant dans l'ombre. Si nous traçons une ligne ou un rayon de la source lumineuse vers un fragment de la boîte la plus à droite, nous pouvons voir que le rayon touche d'abord le conteneur flottant avant de toucher le conteneur le plus à droite. Par conséquent, le fragment du conteneur flottant est éclairé et le fragment du conteneur le plus à droite n'est pas éclairé et se trouve donc dans l'ombre.

Nous voulons déterminer le point du rayon où il a touché un objet pour la première fois et comparer ce point le plus proche à d'autres points du rayon. Nous effectuons ensuite un test de base pour voir si la position du rayon d'un point de test est plus éloignée que le point le plus proche et si c'est le cas, le point de test doit être dans l'ombre. L'itération à travers des milliers de rayons lumineux provenant d'une telle source de lumière est une approche extrêmement inefficace et ne se prête pas très bien au rendu en temps réel. Nous pouvons faire quelque chose de similaire, mais sans lancer de rayons lumineux. À la place, nous utilisons quelque chose que nous connaissons bien : le tampon de profondeur.

Vous vous souvenez peut-être du chapitre sur les tests de profondeur, selon lequel une valeur dans le tampon de profondeur correspond à la profondeur d'un fragment fixé à $[0,1]$ du point de vue de la caméra. Et si nous rendions la scène du point de vue de la lumière et stockions les valeurs de profondeur résultantes dans une texture ? De cette manière, nous pouvons échantillonner les valeurs de profondeur les plus proches du point de vue de la lumière. Après tout, les valeurs de profondeur montrent le premier fragment visible du point de vue de la lumière. Nous stockons toutes ces valeurs de profondeur dans une texture que nous appelons map de profondeur ou shadow map.

![[02a_shadow_mapping-20230829.png]]
L'image de gauche montre une source lumineuse directionnelle (tous les rayons lumineux sont parallèles) qui projette une ombre sur la surface située sous le cube. En utilisant les valeurs de profondeur stockées dans la map de profondeur, nous trouvons le point le plus proche et l'utilisons pour déterminer si les fragments sont dans l'ombre. Nous créons la map de profondeur en effectuant le rendu de la scène (du point de vue de la lumière) à l'aide d'une vue et d'une matrice de projection spécifiques à cette source lumineuse. Cette projection et cette matrice de vue forment ensemble une transformation $T$ qui transforme toute position 3D en espace de coordonnées de la lumière (visible).

>Une lumière directionnelle n'a pas de position puisqu'elle est modélisée pour être infiniment éloignée. Cependant, pour les besoins de la cartographie des ombres, nous devons rendre la scène du point de vue d'une lumière et donc rendre la scène à partir d'une position située quelque part dans la direction de la lumière.


Dans l'image de droite, nous voyons la même lumière directionnelle et le spectateur. Nous rendons un fragment au point $P$ pour lequel nous devons déterminer s'il est dans l'ombre. Pour ce faire, nous transformons d'abord le point $P$ dans l'espace de coordonnées de la lumière à l'aide de $T$. Étant donné que le point $P$ est maintenant vu du point de vue de la lumière, sa coordonnée $z$ correspond à sa profondeur qui, dans cet exemple, est de $0.9$. En utilisant le point $P$, nous pouvons également indexer la map des profondeurs/ombres pour obtenir la profondeur visible la plus proche du point de vue de la lumière, qui se trouve au point $C$ avec une profondeur échantillonnée de $0.4$. Étant donné que l'indexation de la carte de profondeur renvoie une profondeur inférieure à celle du point $P$, nous pouvons conclure que le point $P$ est occulté et donc dans l'ombre.

**La cartographie des ombres consiste donc en deux passages : dans un premier temps, nous effectuons le rendu de la map de profondeur et, dans un second temps, nous effectuons le rendu normal de la scène et utilisons la map de profondeur générée pour calculer si des fragments sont dans l'ombre.** Cela peut sembler un peu compliqué, mais dès que nous aurons expliqué la technique étape par étape, elle commencera à prendre tout son sens.

## La map de profondeur
La première étape consiste à générer une map de profondeur. La map de profondeur est la texture de profondeur telle qu'elle est rendue du point de vue de la lumière que nous utiliserons pour tester les ombres. Comme nous devons stocker le résultat du rendu d'une scène dans une texture, nous allons à nouveau avoir besoin de framebuffers.

Tout d'abord, nous allons créer un objet framebuffer pour le rendu de la carte de profondeur :
```cpp
unsigned int depthMapFBO;
glGenFramebuffers(1, &depthMapFBO);
```
Ensuite, nous créons une texture 2D que nous utiliserons comme tampon de profondeur du framebuffer :
```cpp
const unsigned int SHADOW_WIDTH = 1024, SHADOW_HEIGHT = 1024;

unsigned int depthMap;
glGenTextures(1, &depthMap);
glBindTexture(GL_TEXTURE_2D, depthMap);
glTexImage2D(GL_TEXTURE_2D, 0, GL_DEPTH_COMPONENT, 
             SHADOW_WIDTH, SHADOW_HEIGHT, 0, GL_DEPTH_COMPONENT, GL_FLOAT, NULL);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT); 
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT); 
```
La génération de la map de profondeur ne devrait pas être trop compliquée. Comme nous ne nous intéressons qu'aux valeurs de profondeur, nous spécifions que le format de la texture est `GL_DEPTH_COMPONENT`. Nous donnons également à la texture une largeur et une hauteur de 1024 : c'est la résolution de la map de profondeur.

Avec la texture de profondeur générée, nous pouvons l'attacher au tampon de profondeur du framebuffer :
```cpp
glBindFramebuffer(GL_FRAMEBUFFER, depthMapFBO);
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_TEXTURE_2D, depthMap, 0);
glDrawBuffer(GL_NONE);
glReadBuffer(GL_NONE);
glBindFramebuffer(GL_FRAMEBUFFER, 0);  
```
Nous n'avons besoin des informations de profondeur que lors du rendu de la scène du point de vue de la lumière, il n'est donc pas nécessaire d'avoir un tampon de couleur. Un objet framebuffer n'est cependant pas complet sans un tampon de couleur, nous devons donc explicitement dire à OpenGL que nous n'allons pas rendre de données de couleur. Nous le faisons en réglant les tampons de lecture et de dessin sur `GL_NONE` avec `glDrawBuffer` et `glReadbuffer`.

Avec un framebuffer correctement configuré qui rend les valeurs de profondeur vers une texture, nous pouvons commencer la première passe : générer la carte de profondeur. Combinée à la deuxième passe, l'étape de rendu complète ressemblera à ceci :
```cpp
// 1. first render to depth map
glViewport(0, 0, SHADOW_WIDTH, SHADOW_HEIGHT);
glBindFramebuffer(GL_FRAMEBUFFER, depthMapFBO);
    glClear(GL_DEPTH_BUFFER_BIT);
    ConfigureShaderAndMatrices();
    RenderScene();
glBindFramebuffer(GL_FRAMEBUFFER, 0);
// 2. then render scene as normal with shadow mapping (using depth map)
glViewport(0, 0, SCR_WIDTH, SCR_HEIGHT);
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
ConfigureShaderAndMatrices();
glBindTexture(GL_TEXTURE_2D, depthMap);
RenderScene();
```
Ce code a omis quelques détails, mais il vous donnera l'idée générale du shadow mapping. Ce qu'il est important de noter ici, ce sont les appels à `glViewport`. Comme les cartes d'ombres ont souvent une résolution différente de celle avec laquelle nous avons rendu la scène à l'origine (généralement la résolution de la fenêtre), nous devons modifier les paramètres de la fenêtre pour tenir compte de la taille de la map d'ombres. Si nous oublions de mettre à jour les paramètres de la fenêtre, la carte de profondeur résultante sera soit incomplète, soit trop petite.
## Light space transform

La fonction `ConfigureShaderAndMatrices` est inconnue dans l'extrait de code précédent. Dans la deuxième passe, c'est la routine : s'assurer que les matrices de projection et de vue appropriées sont définies, et définir les matrices de modèle pertinentes par objet. Cependant, dans la première passe, nous devons utiliser une projection et une matrice de vue différentes pour rendre la scène du point de vue de la lumière.

Comme nous modélisons une source de lumière directionnelle, tous ses rayons lumineux sont parallèles. Pour cette raison, nous allons utiliser une matrice de projection orthographique pour la source lumineuse où il n'y a pas de déformation de la perspective :
```cpp
float near_plane = 1.0f, far_plane = 7.5f;
glm::mat4 lightProjection = glm::ortho(-10.0f, 10.0f, -10.0f, 10.0f, near_plane, far_plane);
```
Voici un exemple de matrice de projection orthographique utilisée dans la scène de démonstration de ce chapitre. Étant donné qu'une matrice de projection détermine indirectement l'étendue de ce qui est visible (c'est-à-dire ce qui n'est pas coupé), vous devez vous assurer que la taille du frustum de projection contient correctement les objets que vous souhaitez voir figurer dans la map de profondeur. Lorsque des objets ou des fragments ne figurent pas dans la map de profondeur, ils ne produisent pas d'ombres.

Pour créer une matrice de vue afin de transformer chaque objet pour qu'il soit visible du point de vue de la lumière, nous allons utiliser la fameuse fonction `glm::lookAt` ; cette fois-ci avec la position de la source de lumière regardant le centre de la scène.
```cpp
glm::mat4 lightView = glm::lookAt(glm::vec3(-2.0f, 4.0f, -1.0f), 
                                  glm::vec3( 0.0f, 0.0f,  0.0f), 
                                  glm::vec3( 0.0f, 1.0f,  0.0f));  
```
En combinant ces deux éléments, nous obtenons une matrice de transformation de l'espace lumineux qui transforme chaque vecteur de l'espace mondial en l'espace visible depuis la source lumineuse ; c'est exactement ce dont nous avons besoin pour calculer la map de profondeur.
```cpp
glm::mat4 lightSpaceMatrix = lightProjection * lightView; 
```
Cette matrice espace-lumière est la matrice de transformation que nous avons précédemment désignée par $T$. Avec cette matrice espace-lumière, nous pouvons effectuer le rendu de la scène comme d'habitude tant que nous donnons à chaque shader les équivalents espace-lumière des matrices de projection et de vue. Cependant, nous ne nous intéressons qu'aux valeurs de profondeur et non à tous les calculs coûteux des fragments (éclairage). Pour gagner en performance, nous allons utiliser un shader différent, mais beaucoup plus simple, pour le rendu de la map de profondeur.
## Rendu de la depth map

Lorsque nous rendons la scène du point de vue de la lumière, nous préférons utiliser un shader simple qui ne transforme que les sommets dans l'espace lumière et pas grand-chose de plus. Pour un tel shader simple appelé `simpleDepthShader`, nous utiliserons le vertex shader suivant :
```cpp
#version 330 core
layout (location = 0) in vec3 aPos;

uniform mat4 lightSpaceMatrix;
uniform mat4 model;

void main()
{
    gl_Position = lightSpaceMatrix * model * vec4(aPos, 1.0);
} 
```
Ce shader de vertex prend un modèle par objet, un vertex, et transforme tous les vertex dans l'espace lumière en utilisant `lightSpaceMatrix`.

Comme nous n'avons pas de tampon de couleur et que nous avons désactivé les tampons de dessin et de lecture, les fragments résultants ne nécessitent aucun traitement et nous pouvons donc simplement utiliser un shader de fragment vide:
```cpp
#version 330 core

void main()
{             
    // gl_FragDepth = gl_FragCoord.z;
}  
```
Ce fragment shader vide ne fait aucun traitement, et à la fin de son exécution, le tampon de profondeur est mis à jour. Nous pourrions explicitement définir la profondeur en décommentant sa seule ligne, mais c'est en fait ce qui se passe en coulisses de toute façon.

```cpp
simpleDepthShader.use();
glUniformMatrix4fv(lightSpaceMatrixLocation, 1, GL_FALSE, glm::value_ptr(lightSpaceMatrix));

glViewport(0, 0, SHADOW_WIDTH, SHADOW_HEIGHT);
glBindFramebuffer(GL_FRAMEBUFFER, depthMapFBO);
    glClear(GL_DEPTH_BUFFER_BIT);
    RenderScene(simpleDepthShader);
glBindFramebuffer(GL_FRAMEBUFFER, 0);  
```
Ici, la fonction `RenderScene` prend un programme de shader, appelle toutes les fonctions de dessin pertinentes et définit les matrices de modèle correspondantes si nécessaire.

Le résultat est un tampon de profondeur joliment rempli qui contient la profondeur la plus proche de chaque fragment visible du point de vue de la lumière. En rendant cette texture sur un quad 2D qui remplit l'écran (similaire à ce que nous avons fait dans la section de post-traitement à la fin du chapitre sur les framebuffers), nous obtenons quelque chose comme ceci :
![[shadow_mapping_depth_map.png]]
Pour le rendu de la map de profondeur sur un quad, nous avons utilisé le fragment shader suivant :
```cpp
#version 330 core
out vec4 FragColor;
  
in vec2 TexCoords;

uniform sampler2D depthMap;

void main()
{             
    float depthValue = texture(depthMap, TexCoords).r;
    FragColor = vec4(vec3(depthValue), 1.0);
}  
```
Notez qu'il y a quelques changements subtils lorsque l'on affiche la profondeur en utilisant une matrice de projection en perspective plutôt qu'une matrice de projection orthographique, car la profondeur n'est pas linéaire lorsque l'on utilise la projection en perspective. À la fin de ce chapitre, nous aborderons certaines de ces différences subtiles.

Vous pouvez trouver le code source pour le rendu d'une scène vers une carte de profondeur [ici](https://learnopengl.com/code_viewer_gh.php?code=src/5.advanced_lighting/3.1.1.shadow_mapping_depth/shadow_mapping_depth.cpp).

## Rendu des ombres
Avec une map de profondeur correctement générée, nous pouvons commencer à rendre les ombres réelles. Le code permettant de vérifier si un fragment est dans l'ombre est (bien évidemment) exécuté dans le fragment shader, mais nous effectuons la transformation de l'espace-lumière dans le vertex shader :
```cpp
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aNormal;
layout (location = 2) in vec2 aTexCoords;

out VS_OUT {
    vec3 FragPos;
    vec3 Normal;
    vec2 TexCoords;
    vec4 FragPosLightSpace;
} vs_out;

uniform mat4 projection;
uniform mat4 view;
uniform mat4 model;
uniform mat4 lightSpaceMatrix;

void main()
{    
    vs_out.FragPos = vec3(model * vec4(aPos, 1.0));
    vs_out.Normal = transpose(inverse(mat3(model))) * aNormal;
    vs_out.TexCoords = aTexCoords;
    vs_out.FragPosLightSpace = lightSpaceMatrix * vec4(vs_out.FragPos, 1.0);
    gl_Position = projection * view * vec4(vs_out.FragPos, 1.0);
}

```
Ce qui est nouveau ici, c'est le vecteur de sortie supplémentaire `FragPosLightSpace`. Nous prenons la même matrice `lightSpaceMatrix` (utilisée pour transformer les sommets en espace lumière dans l'étape de la map de profondeur) et transformons la position des sommets dans l'espace monde en espace lumière pour l'utiliser dans le shader de fragments.

Le shader de fragments principal que nous utiliserons pour le rendu de la scène utilise le modèle d'éclairage Blinn-Phong. Dans le shader de fragment, nous calculons ensuite une valeur d'ombre qui est soit de $1.0$ lorsque le fragment est dans l'ombre, soit de $0.0$ lorsqu'il n'est pas dans l'ombre. Les composantes diffuse et spéculaire résultantes sont ensuite multipliées par cette composante d'ombre. Les ombres étant rarement complètement sombres (en raison de la diffusion de la lumière), nous ne tenons pas compte de la composante ambiante dans les multiplications de l'ombre.
```cpp
#version 330 core
out vec4 FragColor;

in VS_OUT {
    vec3 FragPos;
    vec3 Normal;
    vec2 TexCoords;
    vec4 FragPosLightSpace;
} fs_in;

uniform sampler2D diffuseTexture;
uniform sampler2D shadowMap;

uniform vec3 lightPos;
uniform vec3 viewPos;

float ShadowCalculation(vec4 fragPosLightSpace)
{
    [...]
}

void main()
{           
    vec3 color = texture(diffuseTexture, fs_in.TexCoords).rgb;
    vec3 normal = normalize(fs_in.Normal);
    vec3 lightColor = vec3(1.0);
    // ambient
    vec3 ambient = 0.15 * lightColor;
    // diffuse
    vec3 lightDir = normalize(lightPos - fs_in.FragPos);
    float diff = max(dot(lightDir, normal), 0.0);
    vec3 diffuse = diff * lightColor;
    // specular
    vec3 viewDir = normalize(viewPos - fs_in.FragPos);
    float spec = 0.0;
    vec3 halfwayDir = normalize(lightDir + viewDir);  
    spec = pow(max(dot(normal, halfwayDir), 0.0), 64.0);
    vec3 specular = spec * lightColor;    
    // calculate shadow
    float shadow = ShadowCalculation(fs_in.FragPosLightSpace);       
    vec3 lighting = (ambient + (1.0 - shadow) * (diffuse + specular)) * color;    
    
    FragColor = vec4(lighting, 1.0);
}
```
Le fragment shader est en grande partie une copie de ce que nous avons utilisé dans le chapitre sur l'éclairage avancé, mais avec un calcul d'ombre supplémentaire. Nous avons déclaré une fonction `ShadowCalculation` qui effectue la majeure partie du travail sur les ombres. À la fin du fragment shader, nous multiplions les contributions diffuses et spéculaires par l'inverse de la composante d'ombre, c'est-à-dire la proportion du fragment qui n'est pas dans l'ombre. Ce shader de fragment prend comme entrée supplémentaire la position du fragment dans l'espace-lumière et la map de profondeur générée lors de la première passe de rendu.

La première chose à faire pour vérifier si un fragment est dans l'ombre est de transformer la position du fragment dans l'espace-lumière dans l'espace-clip en coordonnées normalisées de l'appareil (device). Lorsque nous transmettons une position de vertex dans l'espace-clip à `gl_Position` dans le vertex shader, OpenGL effectue automatiquement une division en perspective, par exemple en transformant les coordonnées de l'espace-clip dans l'intervalle $[-w,w]$ à $[-1,1]$ en divisant les composantes $x, y$ et $z$ par la composante $w$ du vecteur. Comme le clip-space `FragPosLightSpace` n'est pas transmis au fragment shader par l'intermédiaire de `gl_Position`, nous devons effectuer cette division de perspective nous-mêmes :
```cpp
float ShadowCalculation(vec4 fragPosLightSpace)
{
    // perform perspective divide
    vec3 projCoords = fragPosLightSpace.xyz / fragPosLightSpace.w;
    [...]
}
```
Elle renvoie la position du fragment dans l'espace lumineux dans l'intervalle $[-1,1]$.

>Lors de l'utilisation d'une matrice de projection orthographique, la composante w d'un sommet n'est pas modifiée et cette étape n'a donc aucun sens. Cependant, elle est nécessaire lors de l'utilisation d'une projection en perspective et le fait de conserver cette ligne permet de s'assurer qu'elle fonctionne avec les deux matrices de projection.

Étant donné que la profondeur de la map de profondeur est comprise dans l'intervalle $[0,1]$ et que nous voulons également utiliser `projCoords` pour échantillonner à partir de la map de profondeur, nous transformons les coordonnées NDC dans l'intervalle $[0,1]$ :
```cpp
projCoords = projCoords * 0.5 + 0.5; 
```
Ces coordonnées projetées nous permettent d'échantillonner la map de profondeur, car les coordonnées $[0,1]$ résultant de `projCoords` correspondent directement aux coordonnées NDC transformées lors de la première passe de rendu. Cela nous donne la profondeur la plus proche du point de vue de la lumière :
```cpp
float closestDepth = texture(shadowMap, projCoords.xy).r;   
```
Pour obtenir la profondeur actuelle de ce fragment, il suffit de récupérer la coordonnée $z$ du vecteur projeté, qui est égale à la profondeur de ce fragment du point de vue de la lumière.
```cpp
float currentDepth = projCoords.z;  
```
La comparaison proprement dite consiste alors simplement à vérifier si `currentDepth` est supérieur à `closestDepth` et si c'est le cas, le fragment est dans l'ombre :
```cpp
float shadow = currentDepth > closestDepth  ? 1.0 : 0.0;  
```
La fonction complète de calcul des ombres devient alors :
```cpp
float ShadowCalculation(vec4 fragPosLightSpace)
{
    // perform perspective divide
    vec3 projCoords = fragPosLightSpace.xyz / fragPosLightSpace.w;
    // transform to [0,1] range
    projCoords = projCoords * 0.5 + 0.5;
    // get closest depth value from light's perspective (using [0,1] range fragPosLight as coords)
    float closestDepth = texture(shadowMap, projCoords.xy).r; 
    // get depth of current fragment from light's perspective
    float currentDepth = projCoords.z;
    // check whether current frag pos is in shadow
    float shadow = currentDepth > closestDepth  ? 1.0 : 0.0;

    return shadow;
}  
```
L'activation de ce shader, la liaison des textures appropriées et l'activation des matrices de projection et de vue par défaut lors de la deuxième passe de rendu devraient donner un résultat similaire à l'image ci-dessous :
![[shadow_mapping_shadows.png]]
Si vous avez bien fait les choses, vous devriez en effet voir (bien qu'avec quelques artefacts) des ombres sur le sol et les cubes. Vous pouvez trouver le code source de l'application de démonstration [ici](https://learnopengl.com/code_viewer_gh.php?code=src/5.advanced_lighting/3.1.2.shadow_mapping_base/shadow_mapping_base.cpp).

## Améliorer les shadow maps
Nous avons réussi à faire fonctionner les bases du shadow mapping, mais comme vous pouvez le constater, nous ne sommes pas encore au bout de nos peines à cause de plusieurs artefacts (clairement visibles) liés au shadow mapping que nous devons corriger. Nous nous concentrerons sur la correction de ces artefacts dans les prochaines sections.

### Shadow acne
L'image précédente montre clairement que quelque chose ne va pas. Un zoom plus rapproché montre un motif *Moiré* très évident :
![[shadow_mapping_acne.png]]
Nous pouvons voir qu'une grande partie de la surface du sol est rendue avec des lignes noires évidentes en alternance. Cet artefact de mappage des ombres est appelé acné des ombres et peut être expliqué par l'image suivante :
![[shadow_mapping_acne_diagram.png]]
Comme la map des ombres est limitée par la résolution, plusieurs fragments peuvent échantillonner la même valeur de la map de profondeur lorsqu'ils sont relativement éloignés de la source lumineuse. L'image montre le sol où chaque panneau jaune incliné représente un seul texel de la map de profondeur. Comme vous pouvez le voir, plusieurs fragments échantillonnent le même échantillon de profondeur.

Bien que cela soit généralement acceptable, cela devient un problème lorsque la source lumineuse regarde la surface sous un angle, car dans ce cas, la map de profondeur est également rendue sous un angle. Plusieurs fragments accèdent alors au même texel de profondeur incliné alors que certains sont au-dessus et d'autres au-dessous du sol ; nous obtenons une divergence d'ombre. De ce fait, certains fragments sont considérés comme étant dans l'ombre et d'autres non, ce qui donne le motif rayé de l'image.

Nous pouvons résoudre ce problème à l'aide d'une petite astuce appelée "biais d'ombre" (shadow bias), qui consiste simplement à décaler la profondeur de la surface (ou de la map d'ombre) d'une petite quantité de biais, de sorte que les fragments ne soient pas considérés à tort comme étant au-dessus de la surface.
![[shadow_mapping-20230830.png]]
Avec le biais appliqué, tous les échantillons ont une profondeur inférieure à la profondeur de la surface et la surface entière est donc correctement éclairée sans aucune ombre. Nous pouvons mettre en œuvre un tel biais de la manière suivante :
```cpp
float bias = 0.005;
float shadow = currentDepth - bias > closestDepth  ? 1.0 : 0.0;  
```
Un biais d'ombre de $0.005$ résout en grande partie les problèmes de notre scène, mais **vous pouvez imaginer que la valeur du biais dépend fortement de l'angle entre la source lumineuse et la surface**. Si la surface est très inclinée par rapport à la source lumineuse, les ombres peuvent encore présenter une acné. Une approche plus solide consisterait à modifier la valeur du biais en fonction de l'angle de la surface par rapport à la lumière : un problème que nous pouvons résoudre avec le produit scalaire :

```cpp
float bias = max(0.05 * (1.0 - dot(normal, lightDir)), 0.005);  
```
Nous avons ici un biais maximum de $0.05$ et un minimum de $0.005$ en fonction de la normale de la surface et de la direction de la lumière. Ainsi, les surfaces comme le sol qui sont presque perpendiculaires à la source lumineuse ont un petit biais, tandis que les surfaces comme les faces latérales du cube ont un biais beaucoup plus important. L'image suivante montre la même scène, mais avec un biais pour les ombres :
![[shadow_mapping_with_bias.png]]
Le choix de la (des) valeur(s) de biais correcte(s) nécessite quelques ajustements car elle(s) sera(ont) différente(s) pour chaque scène, mais la plupart du temps, il s'agit simplement d'incrémenter lentement le biais jusqu'à ce que toute l'acné soit supprimée.

### ### Peter panning
L'inconvénient de l'utilisation d'un biais d'ombre est que vous appliquez un décalage à la profondeur réelle des objets. Par conséquent, le biais peut devenir suffisamment important pour que les ombres soient visiblement décalées par rapport à l'emplacement réel des objets, comme vous pouvez le voir ci-dessous (avec une valeur de biais exagérée) :
![[shadow_mapping_peter_panning.png]]
Cet artefact d'ombre est appelé "**peter panning**", car les objets semblent légèrement détachés de leurs ombres. Nous pouvons utiliser une petite astuce pour résoudre la plupart des problèmes de peter panning en utilisant l'élimination des faces avant lors du rendu de la carte de profondeur. Vous vous souvenez peut-être du chapitre sur l'élimination des faces qu'OpenGL élimine par défaut les faces arrière. **En indiquant à OpenGL que nous voulons éliminer les faces avant pendant l'étape de la map des ombres, nous inversons cet ordre.**

Comme nous n'avons besoin que des valeurs de profondeur pour la map de profondeur, cela ne devrait pas avoir d'importance pour les objets solides que nous prenions la profondeur de leurs faces avant ou de leurs faces arrière. L'utilisation de la profondeur de la face arrière ne donne pas de mauvais résultats car l'existence d'ombres à l'intérieur des objets n'a pas d'importance ; de toute façon, nous ne pouvons pas voir à l'intérieur des objets.
![[Pasted image 20230830111353.png]]
Pour corriger le peter panning, nous éliminons toutes les faces avant lors de la génération de la map d'ombres. Notez que vous devez d'abord activer `GL_CULL_FACE`.
```cpp
glCullFace(GL_FRONT);
RenderSceneToDepthMap();
glCullFace(GL_BACK); // don't forget to reset original culling face
```
Cela résout effectivement les problèmes de peter panning, mais seulement pour les objets solides qui ont réellement un intérieur sans ouvertures. **Dans notre scène, par exemple, cela fonctionne parfaitement pour les cubes. Cependant, sur le sol, cela ne fonctionnera pas aussi bien car l'élimination de la face avant supprime complètement le sol de l'équation**. Le sol est un plan unique et serait donc complètement éliminé. Si l'on veut résoudre le problème du peter panning à l'aide de cette astuce, il faut veiller à n'éliminer que les faces avant des objets pour lesquels cela a un sens.

Il faut également tenir compte du fait que les objets proches du récepteur d'ombre (comme le cube éloigné) peuvent encore donner des résultats incorrects. Cependant, avec des valeurs de biais normales, il est généralement possible d'éviter le "peter panning".

### Over sampling
Une autre anomalie visuelle que vous pouvez apprécier ou non est que les régions situées en dehors du frustum visible de la lumière sont considérées comme étant dans l'ombre alors qu'elles ne le sont (généralement) pas. Cela est dû au fait que les coordonnées projetées en dehors du frustum de la lumière sont supérieures à $1.0$ et échantillonneront donc la texture de profondeur en dehors de sa plage par défaut de $[0.1]$. En se basant sur la méthode de wrapping de la texture, nous obtiendrons des résultats de profondeur incorrects qui ne sont pas basés sur les valeurs de profondeur réelles de la source lumineuse.
![[shadow_mapping_outside_frustum.png]]
Vous pouvez voir dans l'image qu'il y a une sorte de région imaginaire de lumière, et qu'une grande partie en dehors de cette zone est dans l'ombre ; cette zone représente la taille de la map de profondeur projetée sur le sol. Cette zone représente la taille de la map de profondeur projetée sur le sol. La raison pour laquelle cela se produit est que nous avons précédemment défini les options de wrapping de la map de profondeur sur `GL_REPEAT`.

Ce que nous préférons, c'est que toutes les coordonnées situées en dehors de la plage de la map de profondeur aient une profondeur de $1.0$, ce qui signifie que ces coordonnées ne seront jamais dans l'ombre (car aucun objet n'aura une profondeur supérieure à $1.0$). Nous pouvons le faire en configurant une couleur de bordure de texture et en réglant les options de wrapping de texture de la map de profondeur sur `GL_CLAMP_TO_BORDER` :
```cpp
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_BORDER);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_BORDER);
float borderColor[] = { 1.0f, 1.0f, 1.0f, 1.0f };
glTexParameterfv(GL_TEXTURE_2D, GL_TEXTURE_BORDER_COLOR, borderColor);  
```
Désormais, chaque fois que nous échantillonnons en dehors de la plage de coordonnées $[0,1]$ de la map de profondeur, la fonction de texture renvoie toujours une profondeur de $1.0$, ce qui produit une valeur d'ombre de $0.0$ Le résultat est désormais plus plausible :
![[shadow_mapping_clamp_edge.png]]
Il semble qu'il y ait encore une partie présentant une région sombre. Il s'agit des coordonnées situées à l'extérieur du plan éloigné du frustum orthographique de la lumière. Vous pouvez constater que cette région sombre se trouve toujours à l'extrémité du frustum de la source lumineuse en observant les directions des ombres.

La coordonnée d'un fragment projeté dans l'espace lumineux est plus éloignée que le plan éloigné de la lumière lorsque sa coordonnée $z$ est supérieure à $1.0$. Dans ce cas, la méthode de wrapping `GL_CLAMP_TO_BORDER` ne fonctionne plus car nous comparons la composante $z$ de la coordonnée avec les valeurs de la map de profondeur ; cette méthode renvoie toujours un résultat positif pour les valeurs $z$ supérieures à $1.0$.

La solution à ce problème est relativement simple : il suffit de forcer la valeur de l'ombre à $0.0$ chaque fois que la coordonnée $z$ du vecteur projeté est supérieure à $1.0$ :
```cpp
float ShadowCalculation(vec4 fragPosLightSpace)
{
    [...]
    if(projCoords.z > 1.0)
        shadow = 0.0;
    
    return shadow;
}
```
Le fait de vérifier le plan éloigné et de limiter la map de profondeur à une couleur de bordure spécifiée manuellement résout le problème du suréchantillonnage de la map de profondeur. Cela nous permet enfin d'obtenir le résultat que nous recherchons :
![[shadow_mapping_over_sampling_fixed.png]]
Le résultat de tout ceci signifie que nous n'avons des ombres que là où les coordonnées du fragment projeté se trouvent dans la zone de la map de profondeur, donc tout ce qui se trouve en dehors du frustum de lumière n'aura pas d'ombres visibles. Comme les jeux vidéo font généralement en sorte que cela ne se produise qu'au loin, c'est un effet beaucoup plus plausible que les régions noires évidentes que nous avions auparavant.

### PCF
Les ombres actuelles sont un ajout agréable au paysage, mais ce n'est pas encore exactement ce que nous voulons. Si l'on zoome sur les ombres, la dépendance de la résolution du mapping des ombres devient rapidement évidente.
![[shadow_mapping_zoom.png]]
Comme la map de profondeur a une résolution fixe, la profondeur s'étend souvent sur plus d'un fragment par texel. Par conséquent, plusieurs fragments échantillonnent la même valeur de profondeur à partir de la map de profondeur et parviennent aux mêmes conclusions d'ombre, ce qui produit ces bords irréguliers.

Vous pouvez réduire ces ombres en bloc en augmentant la résolution de la carte de profondeur ou en essayant d'ajuster le cône de lumière le plus près possible de la scène.

Une autre solution (partielle) à ces bords irréguliers est appelée **PCF** (**percentage-closer filtering**), un terme qui englobe de nombreuses fonctions de filtrage différentes qui produisent des ombres plus douces, en les faisant paraître moins bloquées ou dures. **L'idée est d'échantillonner plusieurs fois la map de profondeur, chaque fois avec des coordonnées de texture légèrement différentes**. Pour chaque échantillon individuel, nous vérifions s'il est dans l'ombre ou non. Tous les sous-résultats sont ensuite combinés et moyennés, ce qui permet d'obtenir une ombre douce et agréable à regarder.

Une implémentation simple du PCF consiste simplement à échantillonner les texels environnants de la map de profondeur et à faire la moyenne des résultats :
```cpp
float shadow = 0.0;
vec2 texelSize = 1.0 / textureSize(shadowMap, 0);
for(int x = -1; x <= 1; ++x)
{
    for(int y = -1; y <= 1; ++y)
    {
        float pcfDepth = texture(shadowMap, projCoords.xy + vec2(x, y) * texelSize).r; 
        shadow += currentDepth - bias > pcfDepth ? 1.0 : 0.0;        
    }    
}
shadow /= 9.0;
```
Ici, `textureSize` renvoie un `vec2` de la largeur et de la hauteur de la texture du sampler donnée au niveau 0 de la mipmap. 1 divisé par ce `vec2` **renvoie la taille d'un texel unique** que nous utilisons pour décaler les coordonnées de la texture, en nous assurant que chaque nouvel échantillon échantillonne une valeur de profondeur différente. Ici, nous échantillonnons 9 valeurs autour des valeurs $x$ et $y$ de la coordonnée projetée, nous testons l'occlusion des ombres et enfin nous faisons la moyenne des résultats en fonction du nombre total d'échantillons prélevés.

En utilisant plus d'échantillons et/ou en variant la variable `texelSize`, vous pouvez augmenter la qualité des ombres douces. Ci-dessous, vous pouvez voir les ombres avec un simple PCF appliqué :
![[shadow_mapping_soft_shadows.png]]
De loin, les ombres sont beaucoup plus belles et moins dures. Si vous zoomez, vous pouvez toujours voir les artefacts de résolution du shadow mapping, mais en général cela donne de bons résultats pour la plupart des applications.

Vous pouvez trouver le code source complet de l'exemple [ici](https://learnopengl.com/code_viewer_gh.php?code=src/5.advanced_lighting/3.1.3.shadow_mapping/shadow_mapping.cpp).

Il y a en fait beaucoup plus à faire avec le PCF et pas mal de techniques pour améliorer considérablement la qualité des ombres douces, mais pour des raisons de longueur de ce chapitre, nous laisserons cela pour une discussion ultérieure.

### Orthographique vs Perspective
Il existe une différence entre le rendu de la map de profondeur avec une matrice de projection orthographique ou perspective. Une matrice de projection orthographique ne déforme pas la scène avec la perspective, de sorte que tous les rayons de vue/lumière sont parallèles. Cela en fait une excellente matrice de projection pour les lumières directionnelles. En revanche, une matrice de projection en perspective déforme tous les sommets en fonction de la perspective, ce qui donne des résultats différents. L'image suivante montre les différentes zones d'ombre des deux méthodes de projection :
![[shadow_mapping-ortho-proj-0230830.png]]
Les projections en perspective sont plus utiles pour les sources lumineuses qui ont un emplacement réel, contrairement aux lumières directionnelles. **Les projections en perspective sont le plus souvent utilisées avec les projecteurs et les lumières ponctuelles, tandis que les projections orthographiques sont utilisées pour les lumières directionnelles.**

Une autre différence subtile avec l'utilisation d'une matrice de projection en perspective est que la visualisation du tampon de profondeur donne souvent un résultat presque entièrement blanc. Cela s'explique par le fait qu'avec la projection en perspective, la profondeur est transformée en valeurs de profondeur non linéaires dont la majeure partie de la plage visible se situe près du plan proche. Pour pouvoir visualiser correctement les valeurs de profondeur comme nous l'avons fait avec la projection orthographique, vous devez d'abord transformer les valeurs de profondeur non linéaires en valeurs linéaires, comme nous l'avons expliqué dans le chapitre sur les [[../04_Advanced_OpenGL/00_depth_testing|tests de profondeur]]:
```cpp
#version 330 core
out vec4 FragColor;
  
in vec2 TexCoords;

uniform sampler2D depthMap;
uniform float near_plane;
uniform float far_plane;

float LinearizeDepth(float depth)
{
    float z = depth * 2.0 - 1.0; // Back to NDC 
    return (2.0 * near_plane * far_plane) / (far_plane + near_plane - z * (far_plane - near_plane));
}

void main()
{             
    float depthValue = texture(depthMap, TexCoords).r;
    FragColor = vec4(vec3(LinearizeDepth(depthValue) / far_plane), 1.0); // perspective
    // FragColor = vec4(vec3(depthValue), 1.0); // orthographic
}  
```
Cela montre des valeurs de profondeur similaires à ce que nous avons vu avec la projection orthographique. Notez que cela n'est utile que pour le débogage ; les vérifications de profondeur restent les mêmes avec les matrices orthographiques ou de projection, car les profondeurs relatives ne changent pas.
## Ressources additionnelles
- [Tutoriel 16 : Shadow mapping](http://www.opengl-tutorial.org/intermediate-tutorials/tutorial-16-shadow-mapping/) : tutoriel similaire sur le shadow mapping par opengl-tutorial.org avec quelques notes supplémentaires.
- [Shadow Mapping - Part 1](http://ogldev.atspace.co.uk/www/tutorial23/tutorial23.html) : un autre tutoriel de shadow mapping par ogldev.
- [How Shadow Mapping Works](https://www.youtube.com/watch?v=EsccgeUpdsM) : un tutoriel YouTube en 3 parties par TheBennyBox sur le shadow mapping et son implémentation.
- [Common Techniques to Improve Shadow Depth Maps](https://msdn.microsoft.com/en-us/library/windows/desktop/ee416324%28v=vs.85%29.aspx) : un excellent article de Microsoft énumérant un grand nombre de techniques permettant d'améliorer la qualité des maps d'ombres.
- [How I Implemented Shadows in my Game Engine](https://www.youtube.com/watch?v=uueB2kVvbHo) : excellente vidéo de ThinMatrix sur ses méthodes d'amélioration des maps d'ombres.