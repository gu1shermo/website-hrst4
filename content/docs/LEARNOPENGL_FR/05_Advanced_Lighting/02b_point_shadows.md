# Ombres ponctuelles

Dans le dernier chapitre, nous avons appris à créer des ombres dynamiques avec le shadow mapping. Cette méthode fonctionne très bien, mais elle est surtout adaptée aux lumières directionnelles (ou ponctuelles), car les ombres ne sont générées que dans la direction de la source lumineuse. C'est pourquoi elle est également connue sous le nom de mapping directionnel des ombres, car la map de profondeur (ou d'ombres) est générée uniquement dans la direction vers laquelle la lumière regarde.

Ce chapitre se concentre sur la génération d'ombres dynamiques dans toutes les directions environnantes. La technique que nous utilisons est parfaite pour les lumières ponctuelles, car une vraie lumière ponctuelle projetterait des ombres dans toutes les directions. Cette technique est connue sous le nom d'ombres ponctuelles (lumière) ou plus anciennement sous le nom de maps d'ombres omnidirectionnelles.

>Ce chapitre s'appuie sur le chapitre précédent consacré au mapping des ombres. Par conséquent, si vous n'êtes pas familiarisé avec le mapping des ombres traditionnel, il est conseillé de lire d'abord le chapitre consacré au [[02a_shadow_mapping|mapping des ombres]].

La technique est essentiellement similaire au mapping directionnel des ombres : nous générons une map de profondeur à partir de la ou des perspectives de la lumière, nous échantillonnons la map de profondeur en fonction de la position actuelle du fragment et nous comparons chaque fragment à la valeur de profondeur stockée pour voir s'il se trouve dans l'ombre. La principale différence entre le mappage directionnel des ombres et le mappage omnidirectionnel des ombres est la carte de profondeur utilisée.

La carte de profondeur dont nous avons besoin nécessite le rendu d'une scène à partir de toutes les directions environnantes d'une lumière ponctuelle et, de ce fait, une map de profondeur 2D normale ne fonctionnera pas ; et si nous utilisions une cubemap à la place ? Comme une cubemap peut stocker des données d'environnement complètes avec seulement 6 faces, il est possible de rendre la scène entière sur chacune des faces d'une cubemap et de les échantillonner comme les valeurs de profondeur environnantes de la lumière ponctuelle.
![[02b_point_shadows-20230830-cubemap.png]]
Le cubemap de profondeur généré est ensuite transmis au shader de fragment d'éclairage qui échantillonne le cubemap avec un vecteur de direction pour obtenir la profondeur la plus proche (du point de vue de la lumière) de ce fragment. La plupart des choses compliquées ont déjà été abordées dans le chapitre sur le [[02a_shadow_mapping|shadow mapping]]. Ce qui rend cette technique un peu plus difficile est la génération du cubemap de profondeur.

## Générer la cubemap de profondeur
Pour créer une cubemap des valeurs de profondeur environnantes d'une lumière, nous devons effectuer le rendu de la scène 6 fois : une fois pour chaque face. Une façon (assez évidente) de le faire est de rendre la scène 6 fois avec 6 matrices de vue différentes, en attachant à chaque fois une face cubemap différente à l'objet framebuffer. Cela ressemblerait à quelque chose comme ceci :
```cpp
for(unsigned int i = 0; i < 6; i++)
{
    GLenum face = GL_TEXTURE_CUBE_MAP_POSITIVE_X + i;
    glFramebufferTexture2D(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, face, depthCubemap, 0);
    BindViewMatrix(lightViewMatrices[i]);
    RenderScene();  
}
```
Cependant, cela peut s'avérer assez coûteux car de nombreux appels de rendu sont nécessaires pour cette seule map de profondeur. Dans ce chapitre, nous allons utiliser une approche alternative (plus organisée) en utilisant une petite astuce dans le shader géométrique qui nous permet de construire le cubemap de profondeur avec une seule passe de rendu.

Tout d'abord, nous devons créer une cubemap :

```cpp
unsigned int depthCubemap;
glGenTextures(1, &depthCubemap);
```
Et attribuer à chacune des faces du cubemap une image de texture 2D évaluée en fonction de la profondeur :
```cpp
const unsigned int SHADOW_WIDTH = 1024, SHADOW_HEIGHT = 1024;
glBindTexture(GL_TEXTURE_CUBE_MAP, depthCubemap);
for (unsigned int i = 0; i < 6; ++i)
        glTexImage2D(GL_TEXTURE_CUBE_MAP_POSITIVE_X + i, 0, GL_DEPTH_COMPONENT, 
                     SHADOW_WIDTH, SHADOW_HEIGHT, 0, GL_DEPTH_COMPONENT, GL_FLOAT, NULL);
```
N'oubliez pas de définir les paramètres de texture :
```cpp
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_R, GL_CLAMP_TO_EDGE);
```
Normalement, nous attacherions une seule face d'une texture cubemap à l'objet framebuffer et rendrions la scène 6 fois, en changeant à chaque fois la cible du tampon de profondeur du framebuffer pour une face cubemap différente. Puisque nous allons utiliser un shader géométrique, qui nous permet d'effectuer le rendu sur toutes les faces en un seul passage, nous pouvons directement attacher la texture cubemap à l'objet framebuffer avec `glFramebufferTexture` :
```cpp
glBindFramebuffer(GL_FRAMEBUFFER, depthMapFBO);
glFramebufferTexture(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, depthCubemap, 0);
glDrawBuffer(GL_NONE);
glReadBuffer(GL_NONE);
glBindFramebuffer(GL_FRAMEBUFFER, 0);  
```
Encore une fois, notez l'appel à `glDrawBuffer` et `glReadBuffer` : nous ne nous intéressons qu'aux valeurs de profondeur lors de la génération d'un cubemap de profondeur, nous devons donc explicitement indiquer à OpenGL que cet objet framebuffer n'effectue pas de rendu vers un tampon de couleur.

Avec les maps d'ombres omnidirectionnelles, nous avons deux passes de rendu : **premièrement, nous générons la cubemap de profondeur et deuxièmement, nous utilisons la cubemap de profondeur dans la passe de rendu normale pour ajouter des ombres à la scène**. Ce processus ressemble un peu à ceci :
```cpp
// 1. first render to depth cubemap
glViewport(0, 0, SHADOW_WIDTH, SHADOW_HEIGHT);
glBindFramebuffer(GL_FRAMEBUFFER, depthMapFBO);
    glClear(GL_DEPTH_BUFFER_BIT);
    ConfigureShaderAndMatrices();
    RenderScene();
glBindFramebuffer(GL_FRAMEBUFFER, 0);
// 2. then render scene as normal with shadow mapping (using depth cubemap)
glViewport(0, 0, SCR_WIDTH, SCR_HEIGHT);
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
ConfigureShaderAndMatrices();
glBindTexture(GL_TEXTURE_CUBE_MAP, depthCubemap);
RenderScene();
```
Le processus est exactement le même que pour le mappage d'ombres par défaut, bien que cette fois-ci nous effectuions le rendu et utilisions une texture de profondeur cubemap au lieu d'une texture de profondeur 2D.

### Light space transform
Avec le framebuffer et le cubemap définis, nous avons besoin d'un moyen de transformer toute la géométrie de la scène vers les espaces de lumière pertinents dans les 6 directions de la lumière. Tout comme pour le chapitre sur le shadow mapping, nous allons avoir besoin d'une matrice de transformation de l'espace de lumière $T$ mais cette fois-ci une pour chaque face.

Chaque matrice de transformation de l'espace lumière contient à la fois une matrice de projection et une matrice de vue. Pour la matrice de projection, nous allons utiliser une matrice de projection en perspective ; la source lumineuse représente un point dans l'espace et la projection en perspective est donc la plus logique. Chaque matrice de transformation de l'espace lumière utilise la même matrice de projection :

```cpp
float aspect = (float)SHADOW_WIDTH/(float)SHADOW_HEIGHT;
float near = 1.0f;
float far = 25.0f;
glm::mat4 shadowProj = glm::perspective(glm::radians(90.0f), aspect, near, far); 
```
Il est important de noter que le paramètre de champ de vision de `glm::perspective` est fixé à 90 degrés. En fixant ce paramètre à 90 degrés, nous nous assurons que le champ de vision est exactement assez grand pour remplir une seule face de la cubemap, de sorte que toutes les faces s'alignent correctement les unes sur les autres au niveau des bords.

Comme la matrice de projection ne change pas en fonction de la direction, nous pouvons la réutiliser pour chacune des 6 matrices de transformation. Nous avons besoin d'une matrice de vue différente pour chaque direction. Avec `glm::lookAt`, nous créons 6 directions de vue, chacune regardant une direction de face du cubemap dans l'ordre suivant : droite, gauche, haut, bas, près et loin.

```cpp
std::vector<glm::mat4> shadowTransforms;
shadowTransforms.push_back(shadowProj * 
                 glm::lookAt(lightPos, lightPos + glm::vec3( 1.0, 0.0, 0.0), glm::vec3(0.0,-1.0, 0.0));
shadowTransforms.push_back(shadowProj * 
                 glm::lookAt(lightPos, lightPos + glm::vec3(-1.0, 0.0, 0.0), glm::vec3(0.0,-1.0, 0.0));
shadowTransforms.push_back(shadowProj * 
                 glm::lookAt(lightPos, lightPos + glm::vec3( 0.0, 1.0, 0.0), glm::vec3(0.0, 0.0, 1.0));
shadowTransforms.push_back(shadowProj * 
                 glm::lookAt(lightPos, lightPos + glm::vec3( 0.0,-1.0, 0.0), glm::vec3(0.0, 0.0,-1.0));
shadowTransforms.push_back(shadowProj * 
                 glm::lookAt(lightPos, lightPos + glm::vec3( 0.0, 0.0, 1.0), glm::vec3(0.0,-1.0, 0.0));
shadowTransforms.push_back(shadowProj * 
                 glm::lookAt(lightPos, lightPos + glm::vec3( 0.0, 0.0,-1.0), glm::vec3(0.0,-1.0, 0.0));
```
Ici, nous créons 6 matrices de vue et les multiplions avec la matrice de projection pour obtenir un total de 6 matrices de transformation de l'espace lumière différentes. Le paramètre cible de `glm::lookAt` regarde chacun dans la direction d'une seule face du cubemap.

Ces matrices de transformation sont envoyées aux shaders qui rendent la profondeur dans le cubemap.

### Shaders de profondeur (depth shaders)

Pour rendre les valeurs de profondeur dans un cubemap de profondeur, nous aurons besoin de trois shaders au total : un vertex et un fragment shader, et un geometry shader entre les deux.

Le shader de géométrie sera le shader responsable de la transformation de tous les sommets de l'espace monde (world space) en 6 espaces de lumière différents. Par conséquent, le shader de sommets transforme simplement les sommets dans l'espace-monde et les dirige vers le shader de géométrie :

```cpp
#version 330 core
layout (location = 0) in vec3 aPos;

uniform mat4 model;

void main()
{
    gl_Position = model * vec4(aPos, 1.0);
}  
```
Le shader géométrique prend en entrée 3 sommets de triangle et un tableau uniforme de matrices de transformation de l'espace lumineux. Le shader géométrique est responsable de la transformation des sommets en espaces de lumière ; c'est là que les choses deviennent intéressantes.

Le shader géométrique possède une variable intégrée appelée `gl_Layer` qui spécifie la face du cubemap vers laquelle émettre une primitive. Lorsqu'il est laissé à lui-même, le shader de géométrie envoie simplement ses primitives le long du pipeline comme d'habitude, mais lorsque nous mettons à jour cette variable, nous pouvons contrôler vers quelle face du cubemap nous effectuons le rendu pour chaque primitive. Bien sûr, cela ne fonctionne que lorsque nous avons une texture cubemap attachée au framebuffer actif.

```cpp
#version 330 core
layout (triangles) in;
layout (triangle_strip, max_vertices=18) out;

uniform mat4 shadowMatrices[6];

out vec4 FragPos; // FragPos from GS (output per emitvertex)

void main()
{
    for(int face = 0; face < 6; ++face)
    {
        gl_Layer = face; // built-in variable that specifies to which face we render.
        for(int i = 0; i < 3; ++i) // for each triangle vertex
        {
            FragPos = gl_in[i].gl_Position;
            gl_Position = shadowMatrices[face] * FragPos;
            EmitVertex();
        }    
        EndPrimitive();
    }
}  
```
Ce shader géométrique est relativement simple. Nous prenons en entrée un triangle, et sortons un total de 6 triangles (6 * 3 égale 18 sommets). Dans la fonction principale, nous itérons sur 6 faces de cubemap où nous spécifions chaque face comme face de sortie en stockant l'entier de la face dans `gl_Layer`. Nous générons ensuite les triangles de sortie en transformant chaque sommet d'entrée de l'espace-monde dans l'espace-lumière approprié en multipliant `FragPos` avec la matrice de transformation de l'espace-lumière de la face. Notez que nous avons également envoyé la variable `FragPos` résultante au shader de fragments dont nous aurons besoin pour calculer une valeur de profondeur.

Dans le chapitre précédent, nous avons utilisé un fragment shader vide et laissé OpenGL calculer les valeurs de profondeur de la map de profondeur. Cette fois-ci, nous allons calculer notre propre profondeur (linéaire) comme la distance linéaire entre chaque position de fragment la plus proche et la position de la source lumineuse. Le calcul de nos propres valeurs de profondeur rend les calculs d'ombres ultérieurs un peu plus intuitifs.

```cpp
#version 330 core
in vec4 FragPos;

uniform vec3 lightPos;
uniform float far_plane;

void main()
{
    // get distance between fragment and light source
    float lightDistance = length(FragPos.xyz - lightPos);
    
    // map to [0;1] range by dividing by far_plane
    lightDistance = lightDistance / far_plane;
    
    // write this as modified depth
    gl_FragDepth = lightDistance;
} 
```
Le fragment shader prend en entrée le `FragPos` du geometry shader, le vecteur de position de la lumière, et la valeur du plan lointain du frustum. Ici, nous prenons la distance entre le fragment et la source lumineuse, nous l'inscrivons dans l'intervalle $[0,1]$ et nous l'écrivons en tant que valeur de profondeur du fragment.

Le rendu de la scène avec ces shaders et l'objet framebuffer attaché au cubemap actif devrait vous donner un cubemap de profondeur complètement rempli pour les calculs d'ombres de la deuxième passe.

## Map d'ombres omnidirectionnelles
Une fois que tout est en place, il est temps de rendre les ombres omnidirectionnelles. La procédure est similaire à celle du chapitre sur le mappage directionnel des ombres, bien que cette fois nous lions une texture cubemap au lieu d'une texture 2D et que nous transmettions également la variable du plan éloigné de la projection lumineuse aux shaders.

```cpp
glViewport(0, 0, SCR_WIDTH, SCR_HEIGHT);
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
shader.use();  
// ... send uniforms to shader (including light's far_plane value)
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_CUBE_MAP, depthCubemap);
// ... bind other textures
RenderScene();
```
Ici, la fonction `renderScene` rend quelques cubes dans une grande pièce cubique dispersés autour d'une source lumineuse au centre de la scène.

Les shaders de sommets et de fragments sont pour la plupart similaires aux shader de shadow mapping originaux : la différence étant que le shader de fragments n'a plus besoin d'une position de fragment dans l'espace lumière puisque nous pouvons maintenant échantillonner les valeurs de profondeur avec un vecteur de direction.

De ce fait, le vertex shader n'a pas besoin de transformer ses vecteurs de position dans l'espace lumière et nous pouvons donc supprimer la variable `FragPosLightSpace` :

```cpp
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aNormal;
layout (location = 2) in vec2 aTexCoords;

out vec2 TexCoords;

out VS_OUT {
    vec3 FragPos;
    vec3 Normal;
    vec2 TexCoords;
} vs_out;

uniform mat4 projection;
uniform mat4 view;
uniform mat4 model;

void main()
{
    vs_out.FragPos = vec3(model * vec4(aPos, 1.0));
    vs_out.Normal = transpose(inverse(mat3(model))) * aNormal;
    vs_out.TexCoords = aTexCoords;
    gl_Position = projection * view * model * vec4(aPos, 1.0);
}
```
Le code d'éclairage Blinn-Phong du fragment shader est exactement le même que celui que nous avions auparavant, avec une multiplication de l'ombre à la fin :
```cpp
#version 330 core
out vec4 FragColor;

in VS_OUT {
    vec3 FragPos;
    vec3 Normal;
    vec2 TexCoords;
} fs_in;

uniform sampler2D diffuseTexture;
uniform samplerCube depthMap;

uniform vec3 lightPos;
uniform vec3 viewPos;

uniform float far_plane;

float ShadowCalculation(vec3 fragPos)
{
    [...]
}

void main()
{           
    vec3 color = texture(diffuseTexture, fs_in.TexCoords).rgb;
    vec3 normal = normalize(fs_in.Normal);
    vec3 lightColor = vec3(0.3);
    // ambient
    vec3 ambient = 0.3 * color;
    // diffuse
    vec3 lightDir = normalize(lightPos - fs_in.FragPos);
    float diff = max(dot(lightDir, normal), 0.0);
    vec3 diffuse = diff * lightColor;
    // specular
    vec3 viewDir = normalize(viewPos - fs_in.FragPos);
    vec3 reflectDir = reflect(-lightDir, normal);
    float spec = 0.0;
    vec3 halfwayDir = normalize(lightDir + viewDir);  
    spec = pow(max(dot(normal, halfwayDir), 0.0), 64.0);
    vec3 specular = spec * lightColor;    
    // calculate shadow
    float shadow = ShadowCalculation(fs_in.FragPos);                      
    vec3 lighting = (ambient + (1.0 - shadow) * (diffuse + specular)) * color;    
    
    FragColor = vec4(lighting, 1.0);
} 
```
Il y a quelques différences subtiles : le code d'éclairage est le même, mais nous avons maintenant un uniforme `samplerCube` et la fonction `ShadowCalculation` prend la position du fragment actuel comme argument au lieu de la position du fragment dans l'espace lumière. Nous incluons également la valeur `far_plane` du frustum de lumière dont nous aurons besoin plus tard.

La plus grande différence se trouve dans le contenu de la fonction `ShadowCalculation` qui échantillonne maintenant les valeurs de profondeur à partir d'un cubemap au lieu d'une texture 2D. Examinons son contenu étape par étape.

La première chose à faire est de récupérer la profondeur du cubemap. Vous vous souvenez peut-être que dans la section cubemap de ce chapitre, nous avons stocké la profondeur en tant que distance linéaire entre le fragment et la position de la lumière ; nous adoptons une approche similaire ici :
```cpp
float ShadowCalculation(vec3 fragPos)
{
    vec3 fragToLight = fragPos - lightPos; 
    float closestDepth = texture(depthMap, fragToLight).r;
}  
```
Ici, nous prenons le vecteur de différence entre la position du fragment et la position de la lumière et nous utilisons ce vecteur comme vecteur de direction pour échantillonner le cubemap. Le vecteur de direction n'a pas besoin d'être un vecteur unitaire pour échantillonner à partir d'un cubemap, il n'est donc pas nécessaire de le normaliser. La valeur `closeestDepth` résultante est la valeur de profondeur normalisée entre la source lumineuse et le fragment visible le plus proche.

La valeur `closestDepth` est actuellement comprise dans l'intervalle $[0,1]$, nous la transformons donc d'abord en $[0,far_plane]$ en la multipliant par `far_plane`.

```cpp
closestDepth *= far_plane;  
```
Ensuite, nous récupérons la valeur de profondeur entre le fragment actuel et la source lumineuse, que nous pouvons facilement obtenir en prenant la longueur de `fragToLight` en raison de la façon dont nous avons calculé les valeurs de profondeur dans le cubemap :
```cpp
float currentDepth = length(fragToLight);  
```
Cela renvoie une valeur de profondeur dans le même intervalle (ou plus grand) que `closestDepth`.

Nous pouvons maintenant comparer les deux valeurs de profondeur pour voir laquelle est la plus proche de l'autre et déterminer si le fragment actuel est dans l'ombre. Nous incluons également un biais d'ombre afin de ne pas obtenir d'acné d'ombre, comme nous l'avons vu dans le chapitre précédent.

```cpp
float bias = 0.05; 
float shadow = currentDepth -  bias > closestDepth ? 1.0 : 0.0; 
```
La fonction devient alors :

```cpp
float ShadowCalculation(vec3 fragPos)
{
    // get vector between fragment position and light position
    vec3 fragToLight = fragPos - lightPos;
    // use the light to fragment vector to sample from the depth map    
    float closestDepth = texture(depthMap, fragToLight).r;
    // it is currently in linear range between [0,1]. Re-transform back to original value
    closestDepth *= far_plane;
    // now get current linear depth as the length between the fragment and light position
    float currentDepth = length(fragToLight);
    // now test for shadows
    float bias = 0.05; 
    float shadow = currentDepth -  bias > closestDepth ? 1.0 : 0.0;

    return shadow;
}  
```
Avec ces shaders, nous obtenons déjà de bonnes ombres, et cette fois dans toutes les directions environnantes, à partir d'une lumière ponctuelle. Avec une lumière ponctuelle positionnée au centre d'une scène simple, cela ressemblera un peu à ceci :
![[02b_point_shadows-20230902_shadow01.png]]
Vous pouvez trouver le code source de cette démo [ici](https://learnopengl.com/code_viewer_gh.php?code=src/5.advanced_lighting/3.2.1.point_shadows/point_shadows.cpp).
## Visualisation du tampon de profondeur cubemap
Si vous êtes un peu comme moi, vous n'avez probablement pas réussi du premier coup, il est donc logique de faire un peu de débogage, l'une des vérifications évidentes étant de valider si la map de profondeur a été construite correctement. Une astuce simple pour visualiser le tampon de profondeur est de prendre la variable `closestDepth` dans la fonction `ShadowCalculation` et de l'afficher comme suit :
```cpp
FragColor = vec4(vec3(closestDepth / far_plane), 1.0);  
```
Le résultat est une scène grisée où chaque couleur représente les valeurs de profondeur linéaire de la scène :
![[02b_point_shadows-20230902_depth.png]]
Vous pouvez également voir les zones d'ombre à venir sur le mur extérieur. Si l'aspect est similaire, vous savez que le cubemap de profondeur a été correctement généré.

## PCF
Comme les maps d'ombres omnidirectionnelles sont basées sur les mêmes principes que les maps d'ombres traditionnelles, elles présentent les mêmes artefacts dépendant de la résolution. Si vous zoomez suffisamment près, vous pouvez à nouveau voir des bords irréguliers. Le filtrage par pourcentage de proximité (Percentage-closer filtering ou PCF) nous permet de lisser ces bords irréguliers en filtrant plusieurs échantillons autour de la position du fragment et en faisant la moyenne des résultats.

Si nous prenons le même filtre PCF simple du chapitre précédent et que nous y ajoutons une troisième dimension, nous obtenons :
```cpp
float shadow  = 0.0;
float bias    = 0.05; 
float samples = 4.0;
float offset  = 0.1;
for(float x = -offset; x < offset; x += offset / (samples * 0.5))
{
    for(float y = -offset; y < offset; y += offset / (samples * 0.5))
    {
        for(float z = -offset; z < offset; z += offset / (samples * 0.5))
        {
            float closestDepth = texture(depthMap, fragToLight + vec3(x, y, z)).r; 
            closestDepth *= far_plane;   // undo mapping [0;1]
            if(currentDepth - bias > closestDepth)
                shadow += 1.0;
        }
    }
}
shadow /= (samples * samples * samples);
```
Le code n'est pas très différent du code traditionnel de shadow mapping. Nous calculons et ajoutons des décalages de texture dynamiquement pour chaque axe sur la base d'un nombre fixe d'échantillons. Pour chaque échantillon, nous répétons le processus d'ombrage original sur la direction de l'échantillon décalé et nous faisons la moyenne des résultats à la fin.

Les ombres sont maintenant plus douces et lisses et donnent des résultats plus plausibles.
![[02b_point_shadows-20230902-PCF.png]]
Cependant, avec des échantillons réglés sur $4.0$, nous prélevons un total de 64 échantillons par fragment, ce qui est beaucoup !

Comme la plupart de ces échantillons sont redondants en ce sens qu'ils échantillonnent à proximité du vecteur de direction original, il pourrait être plus judicieux de n'échantillonner que dans les directions perpendiculaires au vecteur de direction de l'échantillon. Cependant, comme il n'y a pas de moyen (facile) de déterminer quelles sous-directions sont redondantes, cela devient difficile. Une astuce consiste à prendre un tableau de directions de décalage qui sont toutes à peu près séparables, c'est-à-dire que chacune d'entre elles pointe dans des directions complètement différentes. Cela permet de réduire considérablement le nombre de sous-directions proches les unes des autres. Nous présentons ci-dessous un tableau de 20 directions de décalage au maximum :
```cpp
vec3 sampleOffsetDirections[20] = vec3[]
(
   vec3( 1,  1,  1), vec3( 1, -1,  1), vec3(-1, -1,  1), vec3(-1,  1,  1), 
   vec3( 1,  1, -1), vec3( 1, -1, -1), vec3(-1, -1, -1), vec3(-1,  1, -1),
   vec3( 1,  1,  0), vec3( 1, -1,  0), vec3(-1, -1,  0), vec3(-1,  1,  0),
   vec3( 1,  0,  1), vec3(-1,  0,  1), vec3( 1,  0, -1), vec3(-1,  0, -1),
   vec3( 0,  1,  1), vec3( 0, -1,  1), vec3( 0, -1, -1), vec3( 0,  1, -1)
);   
```
À partir de là, nous pouvons adapter l'algorithme PCF pour qu'il prenne une quantité fixe d'échantillons à partir de `sampleOffsetDirections` et qu'il les utilise pour échantillonner le cubemap. L'avantage est que nous avons besoin de beaucoup moins d'échantillons pour obtenir des résultats visuellement similaires.
```cpp
float shadow = 0.0;
float bias   = 0.15;
int samples  = 20;
float viewDistance = length(viewPos - fragPos);
float diskRadius = 0.05;
for(int i = 0; i < samples; ++i)
{
    float closestDepth = texture(depthMap, fragToLight + sampleOffsetDirections[i] * diskRadius).r;
    closestDepth *= far_plane;   // undo mapping [0;1]
    if(currentDepth - bias > closestDepth)
        shadow += 1.0;
}
shadow /= float(samples); 
```
Ici, nous ajoutons plusieurs décalages, mis à l'échelle par un certain `diskRadius`, autour du vecteur de direction `fragToLight` original pour échantillonner à partir du cubemap.

Une autre astuce intéressante que nous pouvons appliquer ici est que nous pouvons changer le `diskRadius` en fonction de la distance de l'observateur au fragment, rendant les ombres plus douces lorsqu'elles sont éloignées et plus nettes lorsqu'elles sont proches.
```cpp
float diskRadius = (1.0 + (viewDistance / far_plane)) / 25.0;  
```
Les résultats de l'algorithme PCF mis à jour donnent des résultats tout aussi bons, voire meilleurs, pour les ombres douces :
![[02b_point_shadows-20230902-PCF2.png]]

Bien entendu, le biais que nous ajoutons à chaque échantillon dépend fortement du contexte et devra toujours être ajusté en fonction de la scène avec laquelle vous travaillez. Jouez avec toutes les valeurs et voyez comment elles affectent la scène.

Vous pouvez trouver le code final [ici](https://learnopengl.com/code_viewer_gh.php?code=src/5.advanced_lighting/3.2.2.point_shadows_soft/point_shadows_soft.cpp).

Je dois mentionner que l'utilisation de shaders géométriques pour générer une map de profondeur n'est pas nécessairement plus rapide que de rendre la scène 6 fois pour chaque face. L'utilisation d'un shader géométrique comme celui-ci a ses propres inconvénients en termes de performances qui peuvent l'emporter sur le gain de performance de l'utilisation d'un shader géométrique en premier lieu. Cela dépend bien sûr du type d'environnement, des pilotes spécifiques de la carte vidéo et de nombreux autres facteurs. Par conséquent, si vous souhaitez vraiment tirer le meilleur parti de votre système, assurez-vous d'établir le profil des deux méthodes et de sélectionner la plus efficace pour votre scène.

## Ressources additionnelles
- [Cartographie des ombres pour les sources lumineuses ponctuelles dans OpenGL](http://www.sunandblackcat.com/tipFullView.php?l=eng&topicid=36) : tutoriel de cartographie des ombres omnidirectionnelle par sunandblackcat.
- [Multipass Shadow Mapping With Point Lights](http://ogldev.atspace.co.uk/www/tutorial43/tutorial43.html) : tutoriel sur les ombres omnidirectionnelles par ogldev.
- [Ombres omnidirectionnelles](http://www.cg.tuwien.ac.at/~husky/RTR/OmnidirShadows-whyCaps.pdf) : une belle série de diapositives sur le mappage omnidirectionnel des ombres par Peter Houska.
