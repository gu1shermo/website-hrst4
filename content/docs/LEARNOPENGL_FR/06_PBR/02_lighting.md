# Lighting
Dans le chapitre précédent, nous avons jeté les bases d'un moteur de rendu réaliste basé sur la physique. Dans ce chapitre, nous allons nous concentrer sur la traduction de la théorie discutée précédemment en un moteur de rendu réel qui utilise des sources de lumière directes (ou analytiques) : pensez aux lumières ponctuelles, aux lumières directionnelles et/ou aux projecteurs.

Commençons par revoir l'équation de réflectance finale du chapitre précédent :
$$
L_0(p,w_0)
=
\int_{\Omega}
(
k_d {c\over \pi}
+
{
DFG
\over
4(w_0 *n)(w_i * n)
}
L_i(p,w_i)n
*
w_idw_i
)
$$

Nous savons maintenant à peu près ce qui se passe, mais il reste une grande inconnue : comment représenter exactement l'irradiation, la radiance totale $L$, de la scène.
Nous savons que la radiance $L$ (telle qu'elle est interprétée dans le domaine de l'infographie) mesure le flux radiant $\phi$ ou l'énergie lumineuse d'une source de lumière sur un angle solide donné $\omega$. Dans notre cas, nous avons supposé que l'angle solide $\omega$ est infiniment petit, auquel cas la radiance mesure le flux d'une source lumineuse sur un seul rayon lumineux ou vecteur de direction.

Compte tenu de ces connaissances, comment les transposer dans les connaissances sur l'éclairage que nous avons accumulées dans les chapitres précédents ? Imaginons que nous ayons une lumière ponctuelle (une source lumineuse qui brille de la même manière dans toutes les directions) avec un flux radiant de $(23.47, 21.31, 20.79)$ traduit en un triplet RVB. L'intensité rayonnante de cette source lumineuse est égale à son flux rayonnant pour tous les rayons de direction sortants. Cependant, lors de l'ombrage d'un point spécifique $p$ sur une surface, parmi toutes les directions de lumière entrantes possibles sur son hémisphère $\Omega$, seul un vecteur de direction entrant $w_i$ provient directement de la source lumineuse ponctuelle. Comme nous n'avons qu'une seule source lumineuse dans notre scène, supposée être un point unique dans l'espace, toutes les autres directions possibles de la lumière entrante ont une radiance nulle observée sur le point $p$ de la surface :
![02_lighting-20230909-pbrlight1.png](02_lighting-20230909-pbrlight1.png)
Si, dans un premier temps, nous supposons que l'atténuation de la lumière (diminution de la lumière sur la distance) n'affecte pas la source lumineuse ponctuelle, la radiance du rayon lumineux entrant est la même quel que soit l'endroit où nous positionnons la lumière (à l'exception de la mise à l'échelle de la radiance en fonction de l'angle d'incidence $cos\theta$). En effet, la lumière ponctuelle a la même intensité radiante quel que soit l'angle sous lequel on la regarde, ce qui permet de modéliser son intensité radiante comme son flux radiant : un vecteur constant $(23.47, 21.31, 20.79)$.

Cependant, la radiance prend également une position $p$ et comme toute source lumineuse ponctuelle réaliste prend en compte l'atténuation de la lumière, l'intensité rayonnante de la source lumineuse ponctuelle est mise à l'échelle par une mesure de la distance entre le point $p$ et la source lumineuse. Ensuite, comme l'indique l'équation originale de la radiance, le résultat est mis à l'échelle par le produit de points entre la normale à la surface $n$ et la direction de la lumière entrante $w_i$.

En termes plus pratiques, dans le cas d'une lumière ponctuelle directe, la fonction de radiance $L$ mesure la couleur de la lumière, atténuée sur sa distance à $p$ et mise à l'échelle par $n*w_i$, mais uniquement sur le seul rayon lumineux $w_i$ qui atteint $p$ et qui est égal au vecteur de direction de la lumière à partir de $p$. En code, cela se traduit par :
```cpp
vec3  lightColor  = vec3(23.47, 21.31, 20.79);
vec3  wi          = normalize(lightPos - fragPos);
float cosTheta    = max(dot(N, Wi), 0.0);
float attenuation = calculateAttenuation(fragPos, lightPos);
vec3  radiance    = lightColor * attenuation * cosTheta;
```
Hormis la terminologie différente, ce morceau de code devrait vous être très familier : c'est exactement la façon dont nous avons fait de l'éclairage diffus jusqu'à présent. En ce qui concerne l'éclairage direct, la radiance est calculée de la même manière que l'éclairage précédent, car un seul vecteur de direction de la lumière contribue à la radiance de la surface.

> Il convient de noter que cette hypothèse est valable dans la mesure où les lumières ponctuelles sont infiniment petites et ne représentent qu'un seul point dans l'espace. Si nous devions modéliser une lumière ayant une surface ou un volume, sa radiance serait non nulle dans plus d'une direction de lumière entrante.

Pour d'autres types de sources lumineuses provenant d'un seul point, nous calculons la radiance de la même manière. Par exemple, une source lumineuse directionnelle a un $w_i$ constant sans facteur d'atténuation. Et un projecteur n'aurait pas une intensité radiante constante, mais une intensité mise à l'échelle par le vecteur de direction vers l'avant du projecteur.

Cela nous ramène également à l'intégrale $\int$ sur l'hémisphère $\Omega$ de la surface. Comme nous connaissons à l'avance les emplacements uniques de toutes les sources lumineuses contribuant à l'ombrage d'un seul point de la surface, il n'est pas nécessaire d'essayer de résoudre l'intégrale. Nous pouvons directement prendre le nombre (connu) de sources lumineuses et calculer leur irradiance totale, étant donné que chaque source lumineuse n'a qu'une seule direction lumineuse qui influence l'irradiance de la surface. Cela rend le PBR sur les sources de lumière directe relativement simple puisque nous n'avons en fait qu'à boucler sur les sources de lumière qui y contribuent. Lorsque nous prenons ensuite en compte l'éclairage de l'environnement dans les chapitres [IBL](IBL) (todo: link), nous devons tenir compte de l'intégrale, car la lumière peut provenir de n'importe quelle direction.

## Un modèle de surface PBR
Commençons par écrire un shader de fragment qui met en œuvre les modèles PBR décrits précédemment. Tout d'abord, nous devons prendre les entrées PBR nécessaires à l'ombrage de la surface :

```cpp
#version 330 core
out vec4 FragColor;
in vec2 TexCoords;
in vec3 WorldPos;
in vec3 Normal;
  
uniform vec3 camPos;
  
uniform vec3  albedo;
uniform float metallic;
uniform float roughness;
uniform float ao;
```
Nous prenons les entrées standard calculées par un vertex shader générique et un ensemble de propriétés matérielles constantes sur la surface de l'objet.

Ensuite, au début du shader de fragment, nous effectuons les calculs habituels requis pour tout algorithme d'éclairage :
```cpp
void main()
{
    vec3 N = normalize(Normal); 
    vec3 V = normalize(camPos - WorldPos);
    [...]
}
```
### Lumière directe
Dans l'exemple de démonstration de ce chapitre, nous avons un total de 4 lumières ponctuelles qui, ensemble, représentent l'irradiance de la scène. Pour satisfaire l'équation de réflectance, nous bouclons sur chaque source lumineuse, calculons sa radiance individuelle et additionnons sa contribution mise à l'échelle par la BRDF et l'angle d'incidence de la lumière. Nous pouvons considérer que la boucle résout l'intégrale $\int$ sur $\Omega$ pour les sources lumineuses directes. Tout d'abord, nous calculons les variables pertinentes par lumière :
```cpp
vec3 Lo = vec3(0.0);
for(int i = 0; i < 4; ++i) 
{
    vec3 L = normalize(lightPositions[i] - WorldPos);
    vec3 H = normalize(V + L);
  
    float distance    = length(lightPositions[i] - WorldPos);
    float attenuation = 1.0 / (distance * distance);
    vec3 radiance     = lightColors[i] * attenuation; 
    [...]  
```

Comme nous calculons l'éclairage dans l'espace linéaire (nous corrigerons le [gamma](../05_Advanced_Lighting/01_gamma_correction.md) à la fin du shader), nous atténuons les sources de lumière par la loi de l'inverse du carré, plus correcte physiquement.

> Bien qu'elle soit physiquement correcte, vous pouvez toujours utiliser l'équation d'atténuation linéaire-quadratique constante qui (bien qu'elle ne soit pas physiquement correcte) peut vous offrir un contrôle beaucoup plus important sur l'affaiblissement de l'énergie de la lumière.

Ensuite, pour chaque lumière, nous voulons calculer le terme de la BRDF spéculaire de Cook-Torrance :
$$
{
DFG
\over
4(w_0*n)(w_i*n)
}
$$
La première chose à faire est de calculer le rapport entre la réflexion spéculaire et la réflexion diffuse, c'est-à-dire la mesure dans laquelle la surface réfléchit la lumière par rapport à la mesure dans laquelle elle la réfracte. Le chapitre précédent nous a appris que l'équation de Fresnel permet justement de calculer ce rapport (notez le **clamp** ici pour éviter les taches noires) :
```cpp
vec3 fresnelSchlick(float cosTheta, vec3 F0)
{
    return F0 + (1.0 - F0) * pow(clamp(1.0 - cosTheta, 0.0, 1.0), 5.0);
} 
```
L'approximation de Fresnel-Schlick prévoit un paramètre $F_0$ qui est connu comme étant la réflexion de la surface à incidence nulle ou la quantité de réflexion de la surface si l'on regarde directement la surface. Le $F_0$ varie selon le matériau et est teinté sur les métaux, comme on le trouve dans les grandes bases de données de matériaux. Dans le flux de travail PBR pour les surfaces métalliques, nous faisons l'hypothèse simplificatrice que la plupart des surfaces diélectriques sont visuellement correctes avec un $F_0$ constant de $0.04$, alors que nous spécifions le $F_0$ pour les surfaces métalliques comme étant donné par la valeur de l'albédo. Cela se traduit par le code suivant :
```cpp
vec3 F0 = vec3(0.04); 
F0      = mix(F0, albedo, metallic);
vec3 F  = fresnelSchlick(max(dot(H, V), 0.0), F0);
```
Comme vous pouvez le constater, pour les surfaces non métalliques, $F_0$ est toujours de $0.04$. Pour les surfaces métalliques, nous faisons varier $F_0$ par interpolation linéaire entre la valeur originale de $F_0$ et la valeur de l'albédo correspondant à la propriété métallique.

Étant donné $F$ les termes restants à calculer sont la fonction de distribution normale $D$ et la fonction géométrique $G$.

Dans un shader d'éclairage PBR direct, leurs équivalents dans le code sont les suivants :
```cpp
float DistributionGGX(vec3 N, vec3 H, float roughness)
{
    float a      = roughness*roughness;
    float a2     = a*a;
    float NdotH  = max(dot(N, H), 0.0);
    float NdotH2 = NdotH*NdotH;
	
    float num   = a2;
    float denom = (NdotH2 * (a2 - 1.0) + 1.0);
    denom = PI * denom * denom;
	
    return num / denom;
}

float GeometrySchlickGGX(float NdotV, float roughness)
{
    float r = (roughness + 1.0);
    float k = (r*r) / 8.0;

    float num   = NdotV;
    float denom = NdotV * (1.0 - k) + k;
	
    return num / denom;
}
float GeometrySmith(vec3 N, vec3 V, vec3 L, float roughness)
{
    float NdotV = max(dot(N, V), 0.0);
    float NdotL = max(dot(N, L), 0.0);
    float ggx2  = GeometrySchlickGGX(NdotV, roughness);
    float ggx1  = GeometrySchlickGGX(NdotL, roughness);
	
    return ggx1 * ggx2;
}
```
Il est important de noter ici que, contrairement au chapitre sur la [théorie](01_theory.md), nous transmettons le paramètre de rugosité directement à ces fonctions ; de cette façon, nous pouvons apporter des modifications spécifiques au terme à la valeur de rugosité d'origine. D'après les observations de Disney, adoptées par Epic Games, l'éclairage semble plus correct si l'on élève la rugosité au carré dans la géométrie et la fonction de distribution normale.

Une fois les deux fonctions définies, le calcul de la $NDF$ et du terme $G$ dans la boucle de réflectance est simple :
```cpp
float NDF = DistributionGGX(N, H, roughness);       
float G   = GeometrySmith(N, V, L, roughness);  
```
Cela nous permet de calculer la BRDF de Cook-Torrance :
```cpp
vec3 numerator    = NDF * G * F;
float denominator = 4.0 * max(dot(N, V), 0.0) * max(dot(N, L), 0.0)  + 0.0001;
vec3 specular     = numerator / denominator;  
```
Notez que nous ajoutons $0.0001$ au dénominateur pour éviter une division par zéro au cas où un produit de points aboutirait à $0.0$.

Nous pouvons maintenant calculer la contribution de chaque lumière à l'équation de réflectance. Comme la valeur de Fresnel correspond directement à $k_S$ nous pouvons utiliser $F$ pour désigner la contribution spéculaire de toute lumière qui frappe la surface. À partir de $k_S$, nous pouvons calculer le rapport de réfraction $k_D$ :

```cpp
vec3 kS = F;
vec3 kD = vec3(1.0) - kS;
  
kD *= 1.0 - metallic;	
```
Étant donné que $k_S$ représente l'énergie de la lumière qui est réfléchie, le rapport restant de l'énergie de la lumière est la lumière qui est réfractée, que nous stockons sous la forme de $k_D$. En outre, comme les surfaces métalliques ne réfractent pas la lumière et n'ont donc pas de réflexions diffuses, nous renforçons cette propriété en annulant $k_D$ si la surface est métallique. Nous obtenons ainsi les données finales dont nous avons besoin pour calculer la valeur de réflectance sortante de chaque lumière :

```cpp
    const float PI = 3.14159265359;
  
    float NdotL = max(dot(N, L), 0.0);        
    Lo += (kD * albedo / PI + specular) * radiance * NdotL;
}
```
La valeur $L_o$ résultante, ou radiance sortante, est en fait le résultat de l'intégrale de l'équation de réflectance $\int$ sur $\Omega$. Il n'est pas vraiment nécessaire d'essayer de résoudre l'intégrale pour toutes les directions possibles de la lumière entrante, car nous connaissons exactement les 4 directions de la lumière entrante qui peuvent influencer le fragment. De ce fait, nous pouvons directement boucler sur ces directions de lumière entrante, par exemple le nombre de lumières dans la scène.

Il ne reste plus qu'à ajouter un terme ambiant (improvisé) au résultat de l'éclairage direct $L_o$ et nous avons la couleur éclairée finale du fragment :
```cpp
vec3 ambient = vec3(0.03) * albedo * ao;
vec3 color   = ambient + Lo;  
```
## Rendu linéaire et HDR
Jusqu'à présent, nous avons supposé que tous nos calculs étaient effectués dans un espace colorimétrique linéaire et, pour en tenir compte, nous devons procéder à une correction gamma à la fin du shader. Le calcul de l'éclairage dans l'espace linéaire est extrêmement important car le PBR exige que toutes les entrées soient linéaires. Ne pas en tenir compte se traduira par un éclairage incorrect. En outre, nous voulons que les entrées de lumière soient proches de leurs équivalents physiques, de sorte que leurs valeurs de radiance ou de couleur puissent varier de façon sauvage sur un large spectre de valeurs. Par conséquent, $L_o$ peut rapidement devenir très élevé, puis être bloqué entre $0.0$ et $1.0$ en raison de la sortie à faible plage dynamique (LDR) par défaut. Nous corrigeons ce problème en prenant $L_o$ et en appliquant le ton ou l'exposition de la valeur HDR (High Dynamic Range) correctement à LDR avant la correction gamma :

```cpp
color = color / (color + vec3(1.0));
color = pow(color, vec3(1.0/2.2)); 
```
Ici, nous appliquons le tone mapping à la couleur HDR à l'aide de l'opérateur Reinhard, en préservant la plage dynamique élevée d'une irradiation qui peut varier fortement, après quoi nous corrigeons la couleur au niveau du gamma. Nous n'avons pas de framebuffer séparé ou d'étape de post-traitement, ce qui nous permet d'appliquer directement le mapping des tons et l'étape de correction gamma à la fin du fragment shader avant.

![02_lighting-20230909_pbrlight2.png](02_lighting-20230909_pbrlight2.png)

La prise en compte de l'espace colorimétrique linéaire et de la plage dynamique élevée est extrêmement importante dans un pipeline PBR. Sans cela, il est impossible de capturer correctement les détails élevés et faibles des différentes intensités lumineuses et vos calculs finissent par être incorrects et donc visuellement désagréables.

## Shader PBR à éclairage direct
Il ne reste plus qu'à transmettre les couleurs finales corrigées en fonction de la tonalité et du gamma au canal de sortie du fragment shader et nous avons un shader d'éclairage PBR direct. Par souci d'exhaustivité, la fonction principale complète est listée ci-dessous :
```cpp
#version 330 core
out vec4 FragColor;
in vec2 TexCoords;
in vec3 WorldPos;
in vec3 Normal;

// material parameters
uniform vec3  albedo;
uniform float metallic;
uniform float roughness;
uniform float ao;

// lights
uniform vec3 lightPositions[4];
uniform vec3 lightColors[4];

uniform vec3 camPos;

const float PI = 3.14159265359;
  
float DistributionGGX(vec3 N, vec3 H, float roughness);
float GeometrySchlickGGX(float NdotV, float roughness);
float GeometrySmith(vec3 N, vec3 V, vec3 L, float roughness);
vec3 fresnelSchlick(float cosTheta, vec3 F0);

void main()
{		
    vec3 N = normalize(Normal);
    vec3 V = normalize(camPos - WorldPos);

    vec3 F0 = vec3(0.04); 
    F0 = mix(F0, albedo, metallic);
	           
    // reflectance equation
    vec3 Lo = vec3(0.0);
    for(int i = 0; i < 4; ++i) 
    {
        // calculate per-light radiance
        vec3 L = normalize(lightPositions[i] - WorldPos);
        vec3 H = normalize(V + L);
        float distance    = length(lightPositions[i] - WorldPos);
        float attenuation = 1.0 / (distance * distance);
        vec3 radiance     = lightColors[i] * attenuation;        
        
        // cook-torrance brdf
        float NDF = DistributionGGX(N, H, roughness);        
        float G   = GeometrySmith(N, V, L, roughness);      
        vec3 F    = fresnelSchlick(max(dot(H, V), 0.0), F0);       
        
        vec3 kS = F;
        vec3 kD = vec3(1.0) - kS;
        kD *= 1.0 - metallic;	  
        
        vec3 numerator    = NDF * G * F;
        float denominator = 4.0 * max(dot(N, V), 0.0) * max(dot(N, L), 0.0) + 0.0001;
        vec3 specular     = numerator / denominator;  
            
        // add to outgoing radiance Lo
        float NdotL = max(dot(N, L), 0.0);                
        Lo += (kD * albedo / PI + specular) * radiance * NdotL; 
    }   
  
    vec3 ambient = vec3(0.03) * albedo * ao;
    vec3 color = ambient + Lo;
	
    color = color / (color + vec3(1.0));
    color = pow(color, vec3(1.0/2.2));  
   
    FragColor = vec4(color, 1.0);
}  
```
Avec la [théorie](01_theory.md) du chapitre précédent et la connaissance de l'équation de réflectance, ce shader ne devrait plus être aussi intimidant. Si nous prenons ce shader, 4 lumières ponctuelles, et quelques sphères dont nous faisons varier les valeurs métalliques et de rugosité sur les axes verticaux et horizontaux respectivement, nous obtiendrons quelque chose comme ceci :
![02_lighting-20230909-pbrlight3.png](02_lighting-20230909-pbrlight3.png)
De bas en haut, la valeur métallique varie de $0.0$ à $1.0$, la rugosité augmentant de gauche à droite de $0.0$ à $1.0$. Vous pouvez voir qu'en changeant seulement ces deux paramètres simples à comprendre, nous pouvons déjà afficher un large éventail de matériaux différents.

Vous pouvez trouver le code source complet de la démo [ici](https://learnopengl.com/code_viewer_gh.php?code=src/6.pbr/1.1.lighting/lighting.cpp).

# PBR texturé
L'extension du système pour qu'il accepte désormais ses paramètres de surface en tant que textures au lieu de valeurs uniformes nous permet de contrôler les propriétés du matériau de surface par fragment :
```cpp
[...]
uniform sampler2D albedoMap;
uniform sampler2D normalMap;
uniform sampler2D metallicMap;
uniform sampler2D roughnessMap;
uniform sampler2D aoMap;
  
void main()
{
    vec3 albedo     = pow(texture(albedoMap, TexCoords).rgb, 2.2);
    vec3 normal     = getNormalFromNormalMap();
    float metallic  = texture(metallicMap, TexCoords).r;
    float roughness = texture(roughnessMap, TexCoords).r;
    float ao        = texture(aoMap, TexCoords).r;
    [...]
}
```
Notez que les textures d'albédo provenant des artistes sont généralement créées dans l'espace **sRGB**, c'est pourquoi nous les convertissons d'abord en espace linéaire avant d'utiliser l'albédo dans nos calculs d'éclairage. En fonction du système utilisé par les artistes pour générer les cartes d'occlusion ambiante, il se peut que vous deviez également les convertir de l'espace sRGB à l'espace linéaire. Les maps métalliques et de rugosité sont presque toujours créées dans l'espace linéaire.

Le remplacement des propriétés matérielles de l'ensemble précédent de sphères par des textures montre déjà une amélioration visuelle majeure par rapport aux algorithmes d'éclairage précédents que nous avons utilisés :
![02_lighting-20230909-pbrlight5.png](02_lighting-20230909-pbrlight5.png)
Vous pouvez trouver le code source complet de la démo texturée [ici](https://learnopengl.com/code_viewer_gh.php?code=src/6.pbr/1.2.lighting_textured/lighting_textured.cpp) et le jeu de textures utilisé [ici](http://freepbr.com/materials/rusted-iron-pbr-metal-material-alt/) (avec une map d'ao blanche). Gardez à l'esprit que les surfaces métalliques ont tendance à paraître trop sombres dans les environnements à éclairage direct car elles n'ont pas de réflectance diffuse. Elles sont plus correctes lorsque l'on prend en compte l'éclairage spéculaire de l'environnement, ce sur quoi nous nous concentrerons dans les prochains chapitres.

Bien qu'il ne soit pas aussi impressionnant visuellement que certaines démonstrations de rendu PBR, étant donné que nous n'avons pas encore intégré l'éclairage basé sur l'image, le système que nous avons maintenant est toujours un moteur de rendu basé sur la physique, et même sans IBL, vous verrez que votre éclairage sera beaucoup plus réaliste.

