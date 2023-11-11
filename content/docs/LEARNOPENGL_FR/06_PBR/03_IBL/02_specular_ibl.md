 * *# IBL Spéculaire
Dans le chapitre précédent, nous avons mis en place le PBR en combinaison avec l'éclairage basé sur l'image en pré-calculant une map d'irradiance comme partie diffuse indirecte de l'éclairage. Dans ce chapitre, nous allons nous concentrer sur la partie spéculaire de l'équation de réflectance :
$$
L_o(p,w_o)
=
\int_{\Omega}
(
k_d
{
c
\over
\pi
}
+
k_s
{
DFG
\over
4(w_o*n)(w_i*n)
}
L_i(p,w_i)n*w_idw_i
)
$$
Vous remarquerez que la partie spéculaire de Cook-Torrance (multipliée par $k_S$) n'est pas constante sur l'intégrale et dépend de la direction de la lumière entrante, mais aussi de la direction de la vue entrante. Essayer de résoudre l'intégrale pour toutes les directions de lumière entrante, y compris toutes les directions de vue possibles, est une surcharge combinatoire et beaucoup trop coûteuse à calculer en temps réel. Epic Games a proposé une solution permettant de pré-convoluer la partie spéculaire en temps réel, moyennant quelques compromis, connue sous le nom d'approximation de la somme divisée (*split sum approximation*).

Cette approximation divise la partie spéculaire de l'équation de réflectance en deux parties distinctes que nous pouvons convoluer individuellement et combiner plus tard dans le shader PBR pour un éclairage spéculaire indirect basé sur l'image. De la même manière que nous avons pré-convolué la map d'irradiance, l'approximation de la somme divisée nécessite une map d'environnement HDR comme entrée de convolution. Pour comprendre l'approximation de la somme fractionnée, nous allons à nouveau examiner l'équation de réflectance, mais en nous concentrant cette fois sur la partie spéculaire :
$$
L_o(p,w_o)
=
\int_{\Omega}
(
k_s
{
DFG
\over
4(w_o*n)(w_i*n)
}
L_i(p,w_i)n*w_idw_i
)
=
\int_\Omega
f_r(p,w_i,w_o)L_i(p,w_i)n*w_idw_i
$$
Pour les mêmes raisons (de performance) que pour la convolution de l'irradiance, nous ne pouvons pas résoudre la partie spéculaire de l'intégrale en temps réel et espérer une performance raisonnable. Il serait donc préférable de pré-calculer cette intégrale pour obtenir quelque chose comme une map IBL spéculaire, d'échantillonner cette map avec la normale du fragment et d'en finir. Cependant, c'est là que les choses se compliquent. Nous avons pu pré-calculer la carte d'irradiance car l'intégrale ne dépendait que de $\omega_i$ et nous avons pu déplacer les termes d'albédo diffus constants hors de l'intégrale. Cette fois, l'intégrale ne dépend pas seulement de $\omega_i$, comme le montre la BRDF :
$$
f_r(p,w_i,w_o)
=
{
DFG
\over
4(w_o*n)(w_i*n)
}
$$
L'intégrale dépend également de $w_o$, et nous ne pouvons pas vraiment échantillonner un cubemap pré-calculé avec deux vecteurs de direction. La position $p$ n'est pas pertinente ici, comme décrit dans le chapitre précédent. Le calcul préalable de cette intégrale pour toutes les combinaisons possibles de $ω_i$ et $ω_o$ n'est pas pratique dans un environnement en temps réel.

L'approximation de la somme divisée d'Epic Games résout le problème en divisant le pré-calcul en deux parties individuelles que nous pouvons ensuite combiner pour obtenir le résultat pré-calculé que nous recherchons. L'approximation de la somme divisée divise l'intégrale spéculaire en deux intégrales distinctes :
$$
L_o(p,w_o)
=
\int_\Omega
L_i(p,w_i)dw_i
*
\int_\Omega
f_r(p,w_i,w_o)n * w_idw_i
$$
La première partie (lorsqu'elle est convoluée) est connue sous le nom de **map d'environnement pré-filtrée**. Il s'agit (comme pour la map d'irradiance) d'une carte de convolution d'environnement pré-calculée, mais qui prend cette fois en compte la rugosité. Pour des niveaux de rugosité croissants, la map d'environnement est convoluée avec davantage de vecteurs d'échantillons dispersés, ce qui crée des réflexions plus floues. Pour chaque niveau de rugosité que nous convoluons, nous stockons les résultats séquentiellement plus flous dans les niveaux mipmap de la map pré-filtrée. Par exemple, une map d'environnement pré-filtrée stockant le résultat pré-convolué de 5 valeurs de rugosité différentes dans ses 5 niveaux mipmap se présente comme suit :
![[02_specular_ibl-20230910-pbribl1.png]]
Nous générons les vecteurs d'échantillonnage et leur quantité de diffusion en utilisant la fonction de distribution normale (NDF) de la BRDF de Cook-Torrance qui prend en entrée la normale et la direction de la vue. Comme nous ne connaissons pas à l'avance la direction de la vue lors de la convolution de la map de l'environnement, Epic Games fait une approximation supplémentaire en supposant que la direction de la vue (et donc la direction de la réflexion spéculaire) est égale à la direction de l'échantillon de sortie $ω_o$. Cela se traduit par le code suivant :
```cpp
vec3 N = normalize(w_o);
vec3 R = N;
vec3 V = R;
```
De cette façon, la convolution pré-filtrée de l'environnement n'a pas besoin d'être consciente de la direction de la vue. Cela signifie que nous n'obtenons pas de belles réflexions spéculaires rasantes lorsque nous regardons des réflexions de surface spéculaire depuis un angle, comme le montre l'image ci-dessous (avec l'autorisation de l'article Moving Frostbite to PBR) ; ceci est cependant généralement considéré comme un compromis acceptable :
![[02_specular_ibl-20230910-pbribl2.png]]
La deuxième partie de l'équation de la somme fractionnée est égale à la partie BRDF de l'intégrale spéculaire. Si nous supposons que la radiance entrante est complètement blanche pour chaque direction (donc $L(p,x)=1.0$), nous pouvons pré-calculer la réponse de la BRDF en fonction d'une rugosité d'entrée et d'un angle d'entrée entre la normale $\vec{n}$ et la direction de la lumière $ω_i$, ou $n⋅ω_i$. Epic Games stocke la réponse précalculée de la BRDF à chaque combinaison de normale et de direction de la lumière sur des valeurs de rugosité variables dans une texture de consultation 2D (LUT) connue sous le nom de map d'intégration de la BRDF. La texture de consultation 2D fournit une échelle (rouge) et une valeur de biais (verte) à la réponse de Fresnel de la surface, ce qui nous donne la deuxième partie de l'intégrale spéculaire divisée :
![[02_specular_ibl-20230910-pbribl3.png]]
Nous générons la texture de recherche en traitant la coordonnée de texture horizontale (comprise entre $0.0$ et $1.0$) d'un plan comme l'entrée $n⋅ω_i$ de la BRDF, et sa coordonnée de texture verticale comme la valeur de rugosité d'entrée. Avec cette map d'intégration de la BRDF et la map d'environnement pré-filtrée, nous pouvons combiner les deux pour obtenir le résultat de l'intégrale spéculaire :
```cpp
float lod             = getMipLevelFromRoughness(roughness);
vec3 prefilteredColor = textureCubeLod(PrefilteredEnvMap, refVec, lod);
vec2 envBRDF          = texture2D(BRDFIntegrationMap, vec2(NdotV, roughness)).xy;
vec3 indirectSpecular = prefilteredColor * (F * envBRDF.x + envBRDF.y) 
```
Cela devrait vous donner un aperçu de la façon dont l'approximation de la somme fractionnée d'Epic Games aborde la partie spéculaire indirecte de l'équation de réflectance. Essayons maintenant de construire nous-mêmes les parties pré-convoluées.

##  Pré-filtrage d'une carte d'environnement HDR
Le pré-filtrage d'une map d'environnement est assez similaire à la manière dont nous avons convolué une map d'irradiance. La différence est que nous tenons compte de la rugosité et que nous stockons séquentiellement les réflexions les plus rugueuses dans les niveaux mip de la map pré-filtrée.

Tout d'abord, nous devons générer une nouvelle cubemap pour contenir les données de la map d'environnement pré-filtrée. Pour être sûr d'allouer suffisamment de mémoire pour les niveaux mip, nous appelons `glGenerateMipmap` afin d'allouer facilement la quantité de mémoire nécessaire :
```cpp
unsigned int prefilterMap;
glGenTextures(1, &prefilterMap);
glBindTexture(GL_TEXTURE_CUBE_MAP, prefilterMap);
for (unsigned int i = 0; i < 6; ++i)
{
    glTexImage2D(GL_TEXTURE_CUBE_MAP_POSITIVE_X + i, 0, GL_RGB16F, 128, 128, 0, GL_RGB, GL_FLOAT, nullptr);
}
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_R, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR); 
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

glGenerateMipmap(GL_TEXTURE_CUBE_MAP);
```

Notez qu'étant donné que nous prévoyons d'échantillonner les mipmaps de `prefilterMap`, vous devrez vous assurer que son filtre de minification est réglé sur `GL_LINEAR_MIPMAP_LINEAR` afin d'activer le filtrage trilinéaire. Nous stockons les réflexions spéculaires pré-filtrées dans une résolution par face de 128 par 128 au niveau du mip de base. Cela devrait suffire pour la plupart des réflexions, mais si vous avez un grand nombre de matériaux lisses (pensez aux réflexions des voitures), vous voudrez peut-être augmenter la résolution.

Dans le chapitre précédent, nous avons convolué la carte de l'environnement en générant des vecteurs d'échantillonnage uniformément répartis sur l'hémisphère $\Omega$ en utilisant des coordonnées sphériques. Si cette méthode fonctionne parfaitement pour l'irradiation, elle est moins efficace pour les réflexions spéculaires. En ce qui concerne les réflexions spéculaires, en fonction de la rugosité d'une surface, la lumière se reflète de près ou de loin autour d'un vecteur de réflexion $\vec{r}$ sur une normale $\vec{n}$, mais (à moins que la surface ne soit extrêmement rugueuse) autour du vecteur de réflexion tout de même :
![[02_specular_ibl-20230910-pblibr4.png]]
La forme générale des réflexions possibles de la lumière sortante est connue sous le nom de **lobe spéculaire**. Plus la rugosité augmente, plus la taille du lobe spéculaire augmente ; et la forme du lobe spéculaire change en fonction de la direction de la lumière entrante. La forme du lobe spéculaire dépend donc fortement du matériau.

En ce qui concerne le modèle de microsurface, nous pouvons imaginer le lobe spéculaire comme l'orientation de la réflexion autour des vecteurs de la micro-facette en fonction de la direction de la lumière entrante. Étant donné que la plupart des rayons lumineux aboutissent à un lobe spéculaire réfléchi autour des vecteurs médians de la micro-facette, il est logique de générer les vecteurs d'échantillonnage d'une manière similaire, car la plupart d'entre eux seraient sinon gaspillés. Ce processus est connu sous le nom d'échantillonnage d'importance.

###  Intégration de Monte Carlo et échantillonnage d'importance
Pour bien comprendre l'importance de l'échantillonnage, il convient d'abord de se pencher sur la construction mathématique connue sous le nom d'**intégration de Monte Carlo**. L'intégration de Monte Carlo repose principalement sur une combinaison de statistiques et de théorie des probabilités. Monte Carlo nous aide à résoudre discrètement le problème de la détermination d'une statistique ou d'une valeur d'une population sans avoir à prendre en considération l'ensemble de la population.

Par exemple, supposons que vous souhaitiez calculer la taille moyenne de tous les citoyens d'un pays. Pour obtenir votre résultat, vous pourriez mesurer chaque citoyen et faire la moyenne de sa taille, ce qui vous donnerait la réponse exacte que vous recherchez. Cependant, étant donné que la plupart des pays ont une population considérable, cette approche n'est pas réaliste : elle nécessiterait trop d'efforts et de temps.

Une approche différente consiste à choisir un sous-ensemble beaucoup plus petit et totalement aléatoire (sans biais) de cette population, à mesurer leur taille et à faire la moyenne des résultats. Cette population peut se limiter à une centaine de personnes. Bien que la réponse ne soit pas aussi précise que la réponse exacte, vous obtiendrez une réponse relativement proche de la vérité de terrain. C'est ce qu'on appelle **la loi des grands nombres**. L'idée est que si vous mesurez un ensemble plus petit de taille $N$ d'échantillons réellement aléatoires de la population totale, le résultat sera relativement proche de la vraie réponse et se rapprochera au fur et à mesure que le nombre d'échantillons $N$ augmente.

L'intégration de Monte Carlo s'appuie sur cette loi des grands nombres et adopte la même approche pour résoudre une intégrale. Plutôt que de résoudre une intégrale pour toutes les valeurs possibles (théoriquement infinies) de l'échantillon $x$ il suffit de générer $N$ valeurs d'échantillon choisies au hasard dans la population totale et d'en faire la moyenne. Au fur et à mesure que $N$ augmente, nous sommes assurés d'obtenir un résultat plus proche de la réponse exacte de l'intégrale :
$$
O=
\int_a^bf(x)dx=
{
1
\over
N
}
\sum_{i=0}^{N-1}
{
f(x)
\over
pdf(x)
}
$$
Pour résoudre l'intégrale, nous prélevons $N$ échantillons aléatoires sur la population $a$ à $b$, nous les additionnons et nous les divisons par le nombre total d'échantillons pour en faire la moyenne. La **pdf** est la fonction de densité de probabilité qui nous indique la probabilité qu'un échantillon spécifique se produise sur l'ensemble des échantillons. Par exemple, la **pdf** de la taille d'une population ressemblerait à ceci :
![[02_specular_ibl-20230910-pbribl7.png]]
Ce graphique montre que si l'on prend un échantillon aléatoire de la population, il y a plus de chances d'obtenir un échantillon d'une personne de taille $1.70$ que de chances d'obtenir un échantillon d'une personne de taille $1.50$.

En ce qui concerne l'intégration de Monte Carlo, certains échantillons peuvent avoir une probabilité plus élevée d'être générés que d'autres. C'est pourquoi, pour toute estimation Monte Carlo générale, nous divisons ou multiplions la valeur échantillonnée par la probabilité de l'échantillon, conformément à une $pdf$. Jusqu'à présent, dans chacun de nos cas d'estimation d'une intégrale, les échantillons que nous avons générés étaient uniformes, c'est-à-dire qu'ils avaient exactement la même probabilité d'être générés. Nos estimations jusqu'à présent étaient sans biais, ce qui signifie qu'avec un nombre toujours croissant d'échantillons, nous finirons par converger vers la solution exacte de l'intégrale.

**Cependant, certains estimateurs de Monte Carlo sont biaisés**, ce qui signifie que les échantillons générés ne sont pas complètement aléatoires, mais orientés vers une valeur ou une direction spécifique. Ces estimateurs de Monte Carlo biaisés ont un taux de convergence plus rapide, ce qui signifie qu'ils peuvent converger vers la solution exacte à un rythme beaucoup plus rapide, mais en raison de leur nature biaisée, il est probable qu'ils ne convergent jamais vers la solution exacte. Il s'agit généralement d'un compromis acceptable, en particulier dans le domaine de l'infographie, car la solution exacte n'est pas très importante tant que les résultats sont visuellement acceptables. Comme nous le verrons bientôt avec l'échantillonnage d'importance (qui utilise un estimateur biaisé), les échantillons générés sont biaisés vers des directions spécifiques, auquel cas nous en tenons compte en multipliant ou en divisant chaque échantillon par sa **pdf** correspondante.

L'intégration de Monte Carlo est très répandue dans l'infographie, car c'est un moyen assez intuitif d'approximer des intégrales continues d'une manière discrète et efficace : prenez une zone/un volume quelconque à échantillonner (comme l'hémisphère $\Omega$), générer un nombre $N$ d'échantillons aléatoires à l'intérieur de la zone/du volume, et additionner et pondérer la contribution de chaque échantillon au résultat final.

L'intégration de Monte Carlo est un sujet mathématique très vaste et je ne m'attarderai pas sur les détails, mais nous mentionnerons qu'il existe plusieurs façons de générer les échantillons aléatoires. Par défaut, chaque échantillon est complètement (pseudo)aléatoire, comme nous en avons l'habitude, mais en utilisant certaines propriétés des séquences semi-aléatoires, nous pouvons générer des vecteurs d'échantillons qui sont toujours aléatoires, mais qui ont des propriétés intéressantes. Par exemple, nous pouvons effectuer une intégration Monte Carlo sur ce que l'on appelle des **séquences à faible discrépance** (?), qui génèrent toujours des échantillons aléatoires, mais chaque échantillon est distribué plus uniformément (image fournie par James Heald) :
![[02_specular_ibl-20230910-pbribl8.png]]
Lorsque l'on utilise une séquence à faible discrépance pour générer les vecteurs d'échantillonnage de Monte Carlo, le processus est connu sous le nom d'**intégration Quasi-Monte Carlo**. **Les méthodes de Quasi-Monte Carlo ont un taux de convergence plus rapide, ce qui les rend intéressantes pour les applications gourmandes en performances.**

Compte tenu de notre nouvelle connaissance de Monte Carlo et de l'intégration Quasi-Monte Carlo, il existe une propriété intéressante que nous pouvons utiliser pour obtenir un taux de convergence encore plus rapide : **l'échantillonnage d'importance**. Nous l'avons déjà mentionné dans ce chapitre, mais lorsqu'il s'agit de réflexions spéculaires de la lumière, les vecteurs de lumière réfléchie sont contraints dans un lobe spéculaire dont la taille est déterminée par la rugosité de la surface. Étant donné que tout échantillon généré (quasi-)aléatoirement en dehors du lobe spéculaire n'est pas pertinent pour l'intégrale spéculaire, il est logique de concentrer la génération d'échantillons à l'intérieur du lobe spéculaire, au prix d'un biais de l'estimateur de Monte Carlo.

C'est essentiellement l'objet de l'échantillonnage d'importance : générer des vecteurs d'échantillonnage dans une région limitée par la rugosité orientée autour du vecteur médian de la micro-facette. En combinant l'échantillonnage Quasi-Monte Carlo avec une séquence à faible discrépance et en biaisant les vecteurs d'échantillonnage à l'aide de l'échantillonnage d'importance, nous obtenons un taux de convergence élevé. Comme nous atteignons la solution plus rapidement, nous aurons besoin de beaucoup moins d'échantillons pour obtenir une approximation suffisante.

###  Une séquence à faible discrépance
Dans ce chapitre, nous allons pré-calculer la partie spéculaire de l'équation de réflectance indirecte en utilisant l'échantillonnage d'importance à partir d'une séquence aléatoire à faible discrépance basée sur la méthode Quasi-Monte Carlo. La séquence que nous utiliserons est connue sous le nom de **séquence de Hammersley**, telle qu'elle a été soigneusement décrite par Holger Dammertz. La séquence de Hammersley est basée sur la séquence de Van Der Corput qui reflète une représentation binaire décimale autour de son point décimal.

Grâce à quelques astuces, nous pouvons générer efficacement la séquence de Van Der Corput dans un programme d'ombrage que nous utiliserons pour obtenir un échantillon i de la séquence de Hammersley sur un total de $N$ échantillons :
```cpp
float RadicalInverse_VdC(uint bits) 
{
    bits = (bits << 16u) | (bits >> 16u);
    bits = ((bits & 0x55555555u) << 1u) | ((bits & 0xAAAAAAAAu) >> 1u);
    bits = ((bits & 0x33333333u) << 2u) | ((bits & 0xCCCCCCCCu) >> 2u);
    bits = ((bits & 0x0F0F0F0Fu) << 4u) | ((bits & 0xF0F0F0F0u) >> 4u);
    bits = ((bits & 0x00FF00FFu) << 8u) | ((bits & 0xFF00FF00u) >> 8u);
    return float(bits) * 2.3283064365386963e-10; // / 0x100000000
}
// ----------------------------------------------------------------------------
vec2 Hammersley(uint i, uint N)
{
    return vec2(float(i)/float(N), RadicalInverse_VdC(i));
```
 La fonction GLSL `Hammersley` nous donne l'échantillon $i$ à faible discrépance de l'ensemble de l'échantillon de taille $N$.

>Séquence de Hammersley sans support des opérateurs de bits

>Tous les pilotes OpenGL ne supportent pas les opérateurs de bits (WebGL et OpenGL ES 2.0 par exemple). Dans ce cas, vous pouvez utiliser une version alternative de la séquence de Van Der Corput qui ne s'appuie pas sur les opérateurs de bits :

```cpp
float VanDerCorput(uint n, uint base)
{
    float invBase = 1.0 / float(base);
    float denom   = 1.0;
    float result  = 0.0;

    for(uint i = 0u; i < 32u; ++i)
    {
        if(n > 0u)
        {
            denom   = mod(float(n), 2.0);
            result += denom * invBase;
            invBase = invBase / 2.0;
            n       = uint(float(n) / 2.0);
        }
    }

    return result;
}
// ----------------------------------------------------------------------------
vec2 HammersleyNoBitOps(uint i, uint N)
{
    return vec2(float(i)/float(N), VanDerCorput(i, 2u));
}
```
> Notez qu'en raison des restrictions de boucle GLSL dans le matériel plus ancien, la séquence boucle sur tous les 32 bits possibles. Cette version est moins performante, mais fonctionne sur tout le matériel si vous vous retrouvez sans opérateurs de bits.

###  GGX Échantillonnage de l'importance
Au lieu de générer uniformément ou aléatoirement (Monte Carlo) des vecteurs d'échantillonnage sur l'hémisphère $\Omega$ de l'intégrale, nous générerons des vecteurs d'échantillonnage biaisés en fonction de l'orientation générale de la réflexion du vecteur à mi-chemin de la microsurface, basé sur la rugosité de la surface. Le processus d'échantillonnage sera similaire à ce que nous avons vu auparavant : commencer une grande boucle, générer une valeur de séquence aléatoire (à faible discrépance), prendre la valeur de séquence pour générer un vecteur d'échantillon dans l'espace tangent, transformer en espace monde et échantillonner la radiance de la scène. Ce qui est différent, c'est que nous utilisons maintenant une valeur de séquence à faible discrépance comme entrée pour générer un vecteur d'échantillonnage :
```cpp
const uint SAMPLE_COUNT = 4096u;
for(uint i = 0u; i < SAMPLE_COUNT; ++i)
{
    vec2 Xi = Hammersley(i, SAMPLE_COUNT); 
```
De plus, pour construire un vecteur d'échantillonnage, nous avons besoin d'un moyen d'orienter et de biaiser le vecteur d'échantillonnage vers le lobe spéculaire d'une certaine rugosité de surface. Nous pouvons prendre le **NDF** tel que décrit dans le chapitre [[../01_theory|théorique]] et combiner le GGX NDF dans le processus de vecteur d'échantillon sphérique tel que décrit par Epic Games :
```cpp
vec3 ImportanceSampleGGX(vec2 Xi, vec3 N, float roughness)
{
    float a = roughness*roughness;
	
    float phi = 2.0 * PI * Xi.x;
    float cosTheta = sqrt((1.0 - Xi.y) / (1.0 + (a*a - 1.0) * Xi.y));
    float sinTheta = sqrt(1.0 - cosTheta*cosTheta);
	
    // from spherical coordinates to cartesian coordinates
    vec3 H;
    H.x = cos(phi) * sinTheta;
    H.y = sin(phi) * sinTheta;
    H.z = cosTheta;
	
    // from tangent-space vector to world-space sample vector
    vec3 up        = abs(N.z) < 0.999 ? vec3(0.0, 0.0, 1.0) : vec3(1.0, 0.0, 0.0);
    vec3 tangent   = normalize(cross(up, N));
    vec3 bitangent = cross(N, tangent);
	
    vec3 sampleVec = tangent * H.x + bitangent * H.y + N * H.z;
    return normalize(sampleVec);
}  
```
Cela nous donne un vecteur d'échantillonnage quelque peu orienté autour du vecteur de la microsurface attendu en fonction d'une certaine rugosité d'entrée et de la valeur de séquence $X_i$ à faible discrépance. Notez qu'Epic Games utilise la rugosité au carré pour obtenir de meilleurs résultats visuels, conformément à la recherche PBR originale de Disney.

Une fois la séquence d'Hammersley à faible écart et la génération d'échantillons définies, nous pouvons finaliser le shader de convolution pré-filtre :
```cpp
#version 330 core
out vec4 FragColor;
in vec3 localPos;

uniform samplerCube environmentMap;
uniform float roughness;

const float PI = 3.14159265359;

float RadicalInverse_VdC(uint bits);
vec2 Hammersley(uint i, uint N);
vec3 ImportanceSampleGGX(vec2 Xi, vec3 N, float roughness);
  
void main()
{		
    vec3 N = normalize(localPos);    
    vec3 R = N;
    vec3 V = R;

    const uint SAMPLE_COUNT = 1024u;
    float totalWeight = 0.0;   
    vec3 prefilteredColor = vec3(0.0);     
    for(uint i = 0u; i < SAMPLE_COUNT; ++i)
    {
        vec2 Xi = Hammersley(i, SAMPLE_COUNT);
        vec3 H  = ImportanceSampleGGX(Xi, N, roughness);
        vec3 L  = normalize(2.0 * dot(V, H) * H - V);

        float NdotL = max(dot(N, L), 0.0);
        if(NdotL > 0.0)
        {
            prefilteredColor += texture(environmentMap, L).rgb * NdotL;
            totalWeight      += NdotL;
        }
    }
    prefilteredColor = prefilteredColor / totalWeight;

    FragColor = vec4(prefilteredColor, 1.0);
}  
```

Nous pré-filtrons l'environnement, sur la base d'une certaine rugosité d'entrée qui varie pour chaque niveau de mipmap du cubemap pré-filtré (de $0.0$ à $1.0$), et nous stockons le résultat dans `prefilteredColor` (couleur pré-filtrée). La couleur pré filtrée résultante est divisée par le poids total de l'échantillon, les échantillons ayant moins d'influence sur le résultat final (pour les petits `NdotL`) contribuant moins au poids final.

## Capturer les niveaux de mipmap avant le filtre
Ce qu'il reste à faire est de laisser OpenGL pré-filtrer la carte de l'environnement avec différentes valeurs de rugosité sur plusieurs niveaux de la mipmap. C'est en fait assez facile à faire avec la configuration originale du chapitre sur l'[[01_diffuse_irradiance|irradiation]] :

```cpp
prefilterShader.use();
prefilterShader.setInt("environmentMap", 0);
prefilterShader.setMat4("projection", captureProjection);
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_CUBE_MAP, envCubemap);

glBindFramebuffer(GL_FRAMEBUFFER, captureFBO);
unsigned int maxMipLevels = 5;
for (unsigned int mip = 0; mip < maxMipLevels; ++mip)
{
    // reisze framebuffer according to mip-level size.
    unsigned int mipWidth  = 128 * std::pow(0.5, mip);
    unsigned int mipHeight = 128 * std::pow(0.5, mip);
    glBindRenderbuffer(GL_RENDERBUFFER, captureRBO);
    glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH_COMPONENT24, mipWidth, mipHeight);
    glViewport(0, 0, mipWidth, mipHeight);

    float roughness = (float)mip / (float)(maxMipLevels - 1);
    prefilterShader.setFloat("roughness", roughness);
    for (unsigned int i = 0; i < 6; ++i)
    {
        prefilterShader.setMat4("view", captureViews[i]);
        glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, 
                               GL_TEXTURE_CUBE_MAP_POSITIVE_X + i, prefilterMap, mip);

        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
        renderCube();
    }
}
glBindFramebuffer(GL_FRAMEBUFFER, 0);
```
Le processus est similaire à la convolution de la map d'irradiance, mais cette fois, nous mettons à l'échelle les dimensions du framebuffer à l'échelle appropriée de la mipmap, chaque niveau de mip réduisant les dimensions d'une échelle de 2. En outre, nous spécifions le niveau de mip dans lequel nous effectuons le rendu dans le dernier paramètre de `glFramebufferTexture2D` et nous passons la rugosité que nous pré-filtrons au shader de pré-filtrage.

Cela devrait nous donner une map d'environnement correctement pré-filtrée qui renvoie des réflexions plus floues au fur et à mesure que nous y accédons par un niveau de mip plus élevé. Si nous utilisons la cubemap d'environnement pré-filtrée dans le shader de la skybox et que nous forçons l'échantillonnage un peu plus haut que son premier niveau de mip comme ceci :
```cpp
vec3 envColor = textureLod(environmentMap, WorldPos, 1.2).rgb; 
```
Nous obtenons un résultat qui ressemble effectivement à une version plus floue de l'environnement original :
![[02_specular_ibl-20230917-ibl0.png]]
Si l'aspect est similaire, vous avez réussi à pré-filtrer la map de l'environnement HDR. Jouez avec différents niveaux de mipmap pour voir la map pré-filtrée passer progressivement de reflets nets à des reflets flous à mesure que les niveaux de mip augmentent.

## Artefacts de convolution du pré-filtre
Bien que la map pré-filtre actuelle fonctionne bien dans la plupart des cas, tôt ou tard vous rencontrerez plusieurs artefacts de rendu qui sont directement liés à la convolution du pré-filtre. Je vais énumérer ici les artefacts les plus courants et expliquer comment les corriger.

### Coutures Cubemap à forte rugosité
L'échantillonnage de la map de pré filtre sur les surfaces rugueuses signifie l'échantillonnage de la map de pré filtre sur certains de ses niveaux de mip inférieurs. Lors de l'échantillonnage de cubemaps, OpenGL par défaut n'interpole pas linéairement les faces du cubemap. Parce que les niveaux de mip inférieurs sont à la fois d'une résolution plus faible et que la map de pré-filtre est convoluée avec un lobe d'échantillonnage beaucoup plus grand, le manque de filtrage entre les faces du cubemap devient assez évident :
![[02_specular_ibl-20230917-ibl1.png]]
Heureusement pour nous, OpenGL nous donne la possibilité de filtrer correctement les faces du cubemap en activant `GL_TEXTURE_CUBE_MAP_SEAMLESS` :
```cpp
glEnable(GL_TEXTURE_CUBE_MAP_SEAMLESS);  
```
Il suffit d'activer cette propriété quelque part au début de votre application pour que les coutures disparaissent.

### Points lumineux dans la convolution du pré-filtre
En raison des détails à haute fréquence et des intensités lumineuses extrêmement variables dans les réflexions spéculaires, la convolution des réflexions spéculaires nécessite un grand nombre d'échantillons pour tenir compte de la nature extrêmement variable des réflexions environnementales HDR. Nous prenons déjà un très grand nombre d'échantillons, mais dans certains environnements, cela peut ne pas suffire pour les niveaux de mip les plus grossiers, auquel cas vous commencerez à voir apparaître des motifs en pointillés autour des zones lumineuses :
![[02_specular_ibl-20230917-ibr3.png]]
Une option consiste à augmenter le nombre d'échantillons, mais cela ne suffira pas pour tous les environnements. Comme l'a décrit Chetan Jags, nous pouvons réduire cet artefact (pendant la convolution du pré filtre) en n'échantillonnant pas directement la map de l'environnement, mais en échantillonnant un niveau de mip de la map de l'environnement basé sur le PDF de l'intégrale et la rugosité :
```cpp
float D   = DistributionGGX(NdotH, roughness);
float pdf = (D * NdotH / (4.0 * HdotV)) + 0.0001; 

float resolution = 512.0; // resolution of source cubemap (per face)
float saTexel  = 4.0 * PI / (6.0 * resolution * resolution);
float saSample = 1.0 / (float(SAMPLE_COUNT) * pdf + 0.0001);

float mipLevel = roughness == 0.0 ? 0.0 : 0.5 * log2(saSample / saTexel); 
```
N'oubliez pas d'activer le filtrage trilinéaire sur la map d'environnement dont vous voulez échantillonner les niveaux de mip :
```cpp
glBindTexture(GL_TEXTURE_CUBE_MAP, envCubemap);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR);
```
Et laisser OpenGL générer les mipmaps après que la texture de base de la cubemap ait été définie :
```cpp
// convert HDR equirectangular environment map to cubemap equivalent
[...]
// then generate mipmaps
glBindTexture(GL_TEXTURE_CUBE_MAP, envCubemap);
glGenerateMipmap(GL_TEXTURE_CUBE_MAP);
```
Cette méthode fonctionne étonnamment bien et devrait permettre d'éliminer la plupart, voire la totalité, des points de votre map de pré filtre sur les surfaces plus rugueuses.

## Précalcul de la BRDF
L'environnement pré-filtré étant opérationnel, nous pouvons nous concentrer sur la deuxième partie de l'approximation de la somme fractionnée : la BRDF. Revenons brièvement sur l'approximation de la somme fractionnée spéculaire :
$$
L_o(p,\omega_o)
=
\int_\Omega L_i(p,\omega_i)d\omega_i
*
\int_\Omega f_r(p,\omega_i,\omega_o)n \cdot \omega_id\omega_i
$$
Nous avons pré-calculé la partie gauche de l'approximation de la somme fractionnée dans la map de pré-filtrage pour différents niveaux de rugosité. La partie droite nous oblige à convoluer l'équation de la BRDF sur l'angle $n⋅\omega_o$, la rugosité de la surface et le $F_0$ de Fresnel. Ceci est similaire à l'intégration de la BRDF spéculaire avec un environnement blanc solide ou une radiance constante $L_i$ de $1.0$. Convoluer la BRDF sur 3 variables est un peu trop, mais nous pouvons essayer de sortir $F_0$ de l'équation de la BRDF spéculaire :
$$
\int_\Omega f_r(p,\omega_i,\omega_o)n \cdot \omega_id\omega_i
=
\int_\Omega f_r(p,\omega_i,\omega_o)
{
F(\omega_o,h)
\over
F(\omega_o,h)
}
n \cdot \omega_id\omega_i
$$
F étant l'équation de Fresnel. En déplaçant le dénominateur de Fresnel vers la BRDF, on obtient l'équation équivalente suivante :
$$
\int_\Omega
{
f_r(p,\omega_i,\omega_o)
\over
F(\omega_o,h)
}
F(\omega_o,h)n \cdot \omega_id\omega_i
$$
En remplaçant le F le plus à droite par l'approximation de Fresnel-Schlick, on obtient :
$$
\int_\Omega
{
f_r(p,\omega_i,\omega_o)
\over
F(\omega_o,h)
}
(F_0 + (1-F_0)(1-\omega_o \cdot h)⁵)n \cdot \omega_id\omega_i
$$
Remplaçons $(1-\omega_o \cdot h)^5$ par $\alpha$ pour faciliter la résolution de $F_0$.
$$
\int_\Omega
{
f_r(p,\omega_i,\omega_o)
\over
F(\omega_o,h)
}
(F_0 + (1-F_0)\alpha)n \cdot \omega_id\omega_i
$$
$$
\int_\Omega
{
f_r(p,\omega_i,\omega_o)
\over
F(\omega_o,h)
}
(F_0 + 1*\alpha - F_0*\alpha)n \cdot \omega_id\omega_i
$$
$$
\int_\Omega
{
f_r(p,\omega_i,\omega_o)
\over
F(\omega_o,h)
}
(F_0(1-\alpha) + \alpha)n \cdot \omega_id\omega_i
$$
Ensuite, nous divisons la fonction de Fresnel $F$ en deux intégrales :
$$
\int_\Omega
{
f_r(p,\omega_i,\omega_o)
\over
F(\omega_o,h)
}
(F_0(1-\alpha))n \cdot \omega_id\omega_i
+
\int_\Omega
{
f_r(p,\omega_i,\omega_o)
\over
F(\omega_o,h)
}
(\alpha)n \cdot \omega_id\omega_i
$$
De cette façon, $F_0$ est constant sur l'intégrale et nous pouvons retirer $F_0$ de l'intégrale. Ensuite, nous remplaçons $\alpha$ par sa forme originale, ce qui nous donne l'équation finale de la somme fractionnée de la BRDF :

$$
F_0
\int_\Omega

f_r(p,\omega_i,\omega_o)
(1-(1 -\omega_o \cdot h)⁵)n \cdot \omega_id\omega_i
+
\int_\Omega

f_r(p,\omega_i,\omega_o)
(1 -\omega_o \cdot h)⁵n \cdot \omega_id\omega_i

$$
Les deux intégrales résultantes représentent respectivement une échelle et un biais par rapport à $F_0$. Notez que comme $f_r(p,\omega_i,\omega_o)$ contient déjà un terme pour $F$, ils s'annulent tous les deux, supprimant $F$ de $f_r$.

De la même manière que pour les maps d'environnement convoluées précédentes, nous pouvons convoluer les équations de la BRDF sur leurs entrées : l'angle entre $n$ et $\omega_o$ et la rugosité. Nous stockons les résultats convolués dans une lookup texture 2D (LUT) connue sous le nom de map d'intégration BRDF que nous utilisons ensuite dans notre shader d'éclairage PBR pour obtenir le résultat spéculaire indirect convolué final.

Le shader de convolution BRDF opère sur un plan 2D, en utilisant ses coordonnées de texture 2D directement comme entrées pour la convolution BRDF (`NdotV` et rugosité). Le code de convolution est largement similaire à la convolution pré-filtre, sauf qu'il traite maintenant le vecteur d'échantillon selon la fonction géométrique de notre BRDF et l'approximation de Fresnel-Schlick :
```cpp
vec2 IntegrateBRDF(float NdotV, float roughness)
{
    vec3 V;
    V.x = sqrt(1.0 - NdotV*NdotV);
    V.y = 0.0;
    V.z = NdotV;

    float A = 0.0;
    float B = 0.0;

    vec3 N = vec3(0.0, 0.0, 1.0);

    const uint SAMPLE_COUNT = 1024u;
    for(uint i = 0u; i < SAMPLE_COUNT; ++i)
    {
        vec2 Xi = Hammersley(i, SAMPLE_COUNT);
        vec3 H  = ImportanceSampleGGX(Xi, N, roughness);
        vec3 L  = normalize(2.0 * dot(V, H) * H - V);

        float NdotL = max(L.z, 0.0);
        float NdotH = max(H.z, 0.0);
        float VdotH = max(dot(V, H), 0.0);

        if(NdotL > 0.0)
        {
            float G = GeometrySmith(N, V, L, roughness);
            float G_Vis = (G * VdotH) / (NdotH * NdotV);
            float Fc = pow(1.0 - VdotH, 5.0);

            A += (1.0 - Fc) * G_Vis;
            B += Fc * G_Vis;
        }
    }
    A /= float(SAMPLE_COUNT);
    B /= float(SAMPLE_COUNT);
    return vec2(A, B);
}
// ----------------------------------------------------------------------------
void main() 
{
    vec2 integratedBRDF = IntegrateBRDF(TexCoords.x, TexCoords.y);
    FragColor = integratedBRDF;
}
```
Comme vous pouvez le constater, la convolution de la BRDF est une traduction directe des mathématiques en code. Nous prenons l'angle $\theta$ et la rugosité en entrée, générons un vecteur d'échantillon avec un échantillonnage d'importance, le traitons sur la géométrie et le terme de Fresnel dérivé de la BRDF, et produisons à la fois une échelle et un biais pour $F_0$ pour chaque échantillon, en faisant la moyenne à la fin.

Vous vous souvenez peut-être du chapitre sur la théorie que le terme géométrique de la BRDF est légèrement différent lorsqu'il est utilisé avec l'IBL, car sa variable $k$ a une interprétation légèrement différente :
$$
k_{direct}
=
{
(\alpha+1)²
\over
8
}
$$
$$
k_{IBL}
=
{
\alpha²
\over
2
}
$$
Comme la convolution de la BRDF fait partie de l'intégrale de l'IBL spéculaire, nous utiliserons $k_{IBL}$ pour la fonction géométrique Schlick-GGX :
```cpp
float GeometrySchlickGGX(float NdotV, float roughness)
{
    float a = roughness;
    float k = (a * a) / 2.0;

    float nom   = NdotV;
    float denom = NdotV * (1.0 - k) + k;

    return nom / denom;
}
// ----------------------------------------------------------------------------
float GeometrySmith(vec3 N, vec3 V, vec3 L, float roughness)
{
    float NdotV = max(dot(N, V), 0.0);
    float NdotL = max(dot(N, L), 0.0);
    float ggx2 = GeometrySchlickGGX(NdotV, roughness);
    float ggx1 = GeometrySchlickGGX(NdotL, roughness);

    return ggx1 * ggx2;
}  
```
Notez que si $k$ prend a comme paramètre, nous n'avons pas élevé la rugosité au carré comme $a$, comme nous l'avons fait à l'origine pour d'autres interprétations de $a$ ; il est probable que $a$ soit déjà élevé au carré ici. Je ne sais pas s'il s'agit d'une incohérence de la part d'Epic Games ou de l'article original de Disney, mais en traduisant directement la rugosité en $a$, on obtient une map d'intégration de la BRDF identique à la version d'Epic Games.

Enfin, pour stocker le résultat de la convolution BRDF, nous allons générer une texture 2D d'une résolution de $512\times512$ :
```cpp
unsigned int brdfLUTTexture;
glGenTextures(1, &brdfLUTTexture);

// pre-allocate enough memory for the LUT texture.
glBindTexture(GL_TEXTURE_2D, brdfLUTTexture);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RG16F, 512, 512, 0, GL_RG, GL_FLOAT, 0);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR); 
```
Notez que nous utilisons un format flottant d'une précision de 16 bits, comme le recommande Epic Games. Veillez à régler le mode d'enveloppement sur `GL_CLAMP_TO_EDGE` pour éviter les artefacts d'échantillonnage des bords.

Ensuite, nous réutilisons le même objet framebuffer et exécutons ce shader sur un quad d'espace d'écran NDC :
```cpp
glBindFramebuffer(GL_FRAMEBUFFER, captureFBO);
glBindRenderbuffer(GL_RENDERBUFFER, captureRBO);
glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH_COMPONENT24, 512, 512);
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, brdfLUTTexture, 0);

glViewport(0, 0, 512, 512);
brdfShader.use();
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
RenderQuad();

glBindFramebuffer(GL_FRAMEBUFFER, 0); 
```
La partie convoluée de la BRDF de l'intégrale de la somme divisée devrait donner le résultat suivant :
![[02_specular_ibl-20230917-ibl4.png]]
Avec la map d'environnement pré-filtrée et la LUT 2D BRDF, nous pouvons reconstruire l'intégrale spéculaire indirecte selon l'approximation de la somme divisée. Le résultat combiné sert alors de lumière spéculaire indirecte ou ambiante.

## Compléter la réflectance IBL
Pour obtenir la partie spéculaire indirecte de l'équation de réflectance, nous devons assembler les deux parties de l'approximation de la somme fractionnée. Commençons par ajouter les données d'éclairage pré calculées au sommet de notre shader PBR :
```cpp
uniform samplerCube prefilterMap;
uniform sampler2D   brdfLUT;  
```
Tout d'abord, nous obtenons les réflexions spéculaires indirectes de la surface en échantillonnant la map de l'environnement pré-filtrée à l'aide du vecteur de réflexion. Notez que nous échantillonnons le niveau de mip approprié en fonction de la rugosité de la surface, ce qui rend les réflexions spéculaires plus floues sur les surfaces plus rugueuses :
```cpp
void main()
{
    [...]
    vec3 R = reflect(-V, N);   

    const float MAX_REFLECTION_LOD = 4.0;
    vec3 prefilteredColor = textureLod(prefilterMap, R,  roughness * MAX_REFLECTION_LOD).rgb;    
    [...]
}
```
Dans l'étape de pré-filtrage, nous ne convoluons la map de l'environnement que jusqu'à un maximum de 5 niveaux de mip (0 à 4), que nous désignons ici par `MAX_REFLECTION_LOD` pour nous assurer que nous n'échantillonnons pas un niveau de mip où il n'y a pas de données (pertinentes).

Ensuite, nous échantillonnons la texture de recherche BRDF en fonction de la rugosité du matériau et de l'angle entre la normale et le vecteur de vue :
```cpp
vec3 F        = FresnelSchlickRoughness(max(dot(N, V), 0.0), F0, roughness);
vec2 envBRDF  = texture(brdfLUT, vec2(max(dot(N, V), 0.0), roughness)).rg;
vec3 specular = prefilteredColor * (F * envBRDF.x + envBRDF.y);
```
Compte tenu de l'échelle et du biais de $F_0$ (ici, nous utilisons directement le résultat indirect de Fresnel $F$) de la texture de recherche BRDF, nous les combinons avec la partie gauche du pré-filtre de l'équation de réflectance IBL et reconstruisons le résultat intégral approximatif en tant que spéculaire.

Nous obtenons ainsi la partie spéculaire indirecte de l'équation de réflectance. Combinez cette partie avec la partie diffuse IBL de l'équation de réflectance du dernier chapitre et vous obtiendrez le résultat PBR IBL complet :
```cpp
vec3 F = FresnelSchlickRoughness(max(dot(N, V), 0.0), F0, roughness);

vec3 kS = F;
vec3 kD = 1.0 - kS;
kD *= 1.0 - metallic;	  
  
vec3 irradiance = texture(irradianceMap, N).rgb;
vec3 diffuse    = irradiance * albedo;
  
const float MAX_REFLECTION_LOD = 4.0;
vec3 prefilteredColor = textureLod(prefilterMap, R,  roughness * MAX_REFLECTION_LOD).rgb;   
vec2 envBRDF  = texture(brdfLUT, vec2(max(dot(N, V), 0.0), roughness)).rg;
vec3 specular = prefilteredColor * (F * envBRDF.x + envBRDF.y);
  
vec3 ambient = (kD * diffuse + specular) * ao; 
```
Notez que nous ne multiplions pas le spéculaire par $k_S$ car nous avons déjà une multiplication de Fresnel.

Maintenant, en exécutant ce code exact sur la série de sphères qui diffèrent par leur rugosité et leurs propriétés métalliques, nous pouvons enfin voir leurs vraies couleurs dans le rendu PBR final :
![[02_specular_ibl-20230917-ibl5.png]]
Nous pourrions même aller plus loin et utiliser des matériaux PBR texturés :
![[02_specular_ibl-20230917-ibl6.png]]
Ou chargez ce [superbe modèle 3D PBR gratuit](http://artisaverb.info/PBT.html) d'Andrew Maximov :
![[02_specular_ibl-20230917-ibl7.png]]
Je suis sûr que nous sommes tous d'accord pour dire que notre éclairage est maintenant beaucoup plus convaincant. Ce qui est encore mieux, c'est que notre éclairage semble physiquement correct quelle que soit la map d'environnement que nous utilisons. Ci-dessous, vous verrez plusieurs cartes HDR pré-calculées différentes, qui changent complètement la dynamique de l'éclairage, mais qui restent physiquement correctes sans changer une seule variable d'éclairage !
![[02_specular_ibl-20230917-ibl8.png]]
Cette aventure PBR s'est avérée être un long voyage. Il y a beaucoup d'étapes et donc beaucoup de choses qui peuvent mal tourner, alors travaillez soigneusement sur les exemples de code de la scène sphérique ou de la scène texturée (y compris tous les shaders) si vous êtes bloqué, ou vérifiez et posez des questions dans les commentaires.

### Et ensuite ?
Nous espérons qu'à la fin de ce tutoriel, vous aurez une compréhension assez claire de ce qu'est le PBR, et que vous aurez même un moteur de rendu PBR opérationnel. Dans ces tutoriels, nous avons pré-calculé toutes les données d'éclairage PBR basées sur l'image au début de notre application, avant la boucle de rendu. C'était très bien à des fins éducatives, mais pas très bien pour une utilisation pratique du PBR. Tout d'abord, le pré-calcul ne doit être effectué qu'une seule fois, et non à chaque démarrage. Deuxièmement, dès que vous utilisez plusieurs maps d'environnement, vous devez pré-calculer chacune d'entre elles à chaque démarrage, ce qui a tendance à s'accumuler.

C'est pourquoi il est préférable de pré-calculer une map d'environnement en une map d'irradiance et de pré-filtre une seule fois, puis de la stocker sur disque (notez que la map d'intégration BRDF ne dépend pas d'une map d'environnement, vous n'avez donc besoin de la calculer ou de la charger qu'une seule fois). Cela signifie que vous devrez créer un format d'image personnalisé pour stocker les cubemaps HDR, y compris leurs niveaux de mip. Ou bien vous les stockerez (et les chargerez) dans l'un des formats disponibles (comme le format `.dds` qui permet de stocker les niveaux de mip).

De plus, nous avons décrit l'ensemble du processus dans ces tutoriels, y compris la génération d'images IBL pré-calculées pour nous aider à mieux comprendre le pipeline PBR. Mais vous pourrez tout aussi bien utiliser plusieurs excellents outils comme [cmftStudio](https://github.com/dariomanesku/cmftStudio) ou [IBLBaker](https://github.com/derkreature/IBLBaker) pour générer ces maps pré-calculées pour vous.

Un point que nous n'avons pas abordé est celui des cubemaps pré-calculés comme sondes de réflexion : interpolation des cubemaps et correction de la parallaxe. Il s'agit de placer plusieurs sondes de réflexion dans votre scène qui prennent un instantané cubemap de la scène à cet endroit spécifique, que nous pouvons ensuite convoluer en tant que données IBL pour cette partie de la scène. En interpolant entre plusieurs de ces sondes en fonction du voisinage de la caméra, nous pouvons obtenir un éclairage local très détaillé basé sur l'image qui est simplement limité par la quantité de sondes de réflexion que nous sommes prêts à placer. De cette manière, l'éclairage basé sur l'image peut être correctement mis à jour lorsque l'on passe d'une partie extérieure lumineuse d'une scène à une partie intérieure plus sombre, par exemple. J'écrirai un tutoriel sur les sondes de réflexion dans le futur, mais pour l'instant je vous recommande l'article de Chetan Jags ci-dessous pour vous donner une longueur d'avance.

## Ressources supplémentaires
- [Real Shading in Unreal Engine 4](http://blog.selfshadow.com/publications/s2013-shading-course/karis/s2013_pbs_epic_notes_v2.pdf) : explique l'approximation de la somme fractionnée d'Epic Games. C'est sur cet article qu'est basé le code IBL PBR.
- [Physically Based Shading and Image Based Lighting](http://www.trentreed.net/blog/physically-based-shading-and-image-based-lighting/) : excellent article de blog de Trent Reed sur l'intégration de l'IBL spéculaire dans un pipeline PBR en temps réel.
- [Image Based Lighting](https://chetanjags.wordpress.com/2015/08/26/image-based-lighting/) : article très complet de Chetan Jags sur l'éclairage spéculaire basé sur l'image et plusieurs de ses réserves, y compris l'interpolation de la sonde lumineuse.
- [Moving Frostbite to PBR](https://seblagarde.files.wordpress.com/2015/07/course_notes_moving_frostbite_to_pbr_v32.pdf) : aperçu bien écrit et approfondi de l'intégration de PBR dans un moteur de jeu AAA par Sébastien Lagarde et Charles de Rousiers.
- [Physically Based Rendering - Part Three](https://jmonkeyengine.github.io/wiki/jme3/advanced/pbr_part3.html) : vue d'ensemble de l'éclairage IBL et du PBR par l'équipe de JMonkeyEngine.
- [Notes d'implémentation : Runtime Environment Map Filtering for Image Based Lighting](https://placeholderart.wordpress.com/2015/07/28/implementation-notes-runtime-environment-map-filtering-for-image-based-lighting/) : article détaillé de Padraic Hennessy sur le pré-filtrage des maps d'environnement HDR et l'optimisation significative du processus d'échantillonnage.