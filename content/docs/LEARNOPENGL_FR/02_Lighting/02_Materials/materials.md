---
tags: [cg, opengl, computer graphics, cpp]
dg-publish: true
---
# Materials
**Dans le monde réel, chaque objet a une réaction différente à la lumière**. Les objets en acier sont souvent plus brillants qu'un vase en argile, par exemple, et un récipient en bois ne réagit pas de la même manière à la lumière qu'un récipient en acier. Certains objets reflètent la lumière sans trop se disperser, ce qui donne de petits reflets spéculaires, tandis que d'autres se dispersent beaucoup, ce qui donne un plus grand rayon au reflet. Si nous voulons simuler plusieurs types d'objets dans OpenGL, nous devons définir des propriétés matérielles spécifiques à chaque surface.  
  
Dans le chapitre précédent, nous avons défini un objet et une couleur de lumière pour définir la sortie visuelle de l'objet, combinée à une composante d'intensité ambiante et spéculaire. Lors de la description d'une surface, nous pouvons définir une couleur de matériau pour chacune des trois composantes d'éclairage : l'éclairage ambiant, l'éclairage diffus et l'éclairage spéculaire. En spécifiant une couleur pour chacune des composantes, nous disposons d'un contrôle fin sur la couleur de sortie de la surface. Ajoutez maintenant un composant de brillance à ces trois couleurs et vous obtenez toutes les propriétés matérielles dont vous avez besoin : 
```cpp
#version 330 core
struct Material {
    vec3 ambient;
    vec3 diffuse;
    vec3 specular;
    float shininess;
}; 
  
uniform Material material;
```
Dans le fragment shader, nous créons une structure pour stocker les propriétés matérielles de la surface. Nous pouvons également les stocker sous forme de valeurs uniformes individuelles, mais le stockage dans une structure permet de mieux les organiser. Nous définissons d'abord l'agencement de la structure, puis nous déclarons simplement une variable uniforme avec la structure nouvellement créée comme type.  
  
Comme vous pouvez le voir, nous définissons un vecteur de couleur pour chacun des composants de l'éclairage Phong. Le vecteur de matériau ambiant définit la couleur que la surface reflète sous un éclairage ambiant ; il s'agit généralement de la même couleur que celle de la surface. Le vecteur de matériau diffus définit la couleur de la surface sous un éclairage diffus. La couleur diffuse est (comme pour l'éclairage ambiant) réglée sur la couleur de la surface souhaitée. Le vecteur de matériau spéculaire définit la couleur de l'accentuation spéculaire de la surface (ou peut même refléter une couleur spécifique à la surface). Enfin, la brillance influe sur la diffusion/le rayon de la lumière spéculaire.  
  
Avec ces 4 composants qui définissent le matériau d'un objet, nous pouvons simuler de nombreux matériaux du monde réel. Un tableau trouvé sur [devernay.free.fr](devernay.free.fr) montre une liste de propriétés de matériaux qui simulent des matériaux réels trouvés dans le monde extérieur. L'image suivante montre l'effet de plusieurs de ces valeurs de matériaux du monde réel sur notre cube :
![[material1.png]]
Comme vous pouvez le constater, en spécifiant correctement les propriétés matérielles d'une surface, il semble que la perception que nous avons de l'objet soit modifiée. Les effets sont clairement perceptibles, mais pour obtenir des résultats plus réalistes, nous devrons remplacer le cube par quelque chose de plus compliqué. Dans les chapitres consacrés au chargement des modèles, nous aborderons des formes plus complexes.  
  
Trouver les bons paramètres de matériau pour un objet est une tâche difficile qui requiert principalement de l'expérimentation et beaucoup d'expérience. Il n'est pas rare de détruire complètement la qualité visuelle d'un objet à cause d'un matériau mal placé.  
  
Essayons d'implémenter un tel système de matériaux dans les shaders.

# Implémentation des matériaux
Nous avons créé une structure de matériau uniforme dans le fragment shader. Nous allons donc modifier les calculs d'éclairage pour qu'ils soient conformes aux nouvelles propriétés du matériau. Comme toutes les variables matérielles sont stockées dans une structure, nous pouvons y accéder à partir du matériau uniforme :
```cpp
void main()
{    
    // ambient
    vec3 ambient = lightColor * material.ambient;
  	
    // diffuse 
    vec3 norm = normalize(Normal);
    vec3 lightDir = normalize(lightPos - FragPos);
    float diff = max(dot(norm, lightDir), 0.0);
    vec3 diffuse = lightColor * (diff * material.diffuse);
    
    // specular
    vec3 viewDir = normalize(viewPos - FragPos);
    vec3 reflectDir = reflect(-lightDir, norm);  
    float spec = pow(max(dot(viewDir, reflectDir), 0.0), material.shininess);
    vec3 specular = lightColor * (spec * material.specular);  
        
    vec3 result = ambient + diffuse + specular;
    FragColor = vec4(result, 1.0);
}
```
Comme vous pouvez le voir, nous accédons maintenant à toutes les propriétés de la structure matérielle là où nous en avons besoin et, cette fois, nous calculons la couleur de sortie résultante à l'aide des couleurs du matériau. Chaque attribut de matériau de l'objet est multiplié par ses composantes d'éclairage respectives.  
  
Nous pouvons définir le matériau de l'objet dans l'application en définissant les uniformes appropriés. En GLSL, une structure n'a rien de spécial en ce qui concerne la définition des uniformes ; une structure agit uniquement comme un espace de noms pour les variables d'uniformes. Si nous voulons remplir la structure, nous devrons définir les uniformes individuels, mais avec le préfixe du nom de la structure : 
```cpp
lightingShader.setVec3("material.ambient", 1.0f, 0.5f, 0.31f);
lightingShader.setVec3("material.diffuse", 1.0f, 0.5f, 0.31f);
lightingShader.setVec3("material.specular", 0.5f, 0.5f, 0.5f);
lightingShader.setFloat("material.shininess", 32.0f);
```
Nous réglons les composantes ambiante et diffuse sur la couleur que nous souhaitons donner à l'objet et nous réglons la composante spéculaire de l'objet sur une couleur moyennement lumineuse ; nous ne voulons pas que la composante spéculaire soit trop forte. Nous maintenons également la brillance à 32.  
  
Nous pouvons maintenant facilement influencer le matériau de l'objet à partir de l'application. L'exécution du programme donne quelque chose comme ceci :  
![[material2.png]]
Mais cela n'a pas l'air d'être le cas ?

## Propriétés de la lumiàre
**L'objet est beaucoup trop lumineux**. La raison pour laquelle l'objet est trop lumineux est que les couleurs ambiantes, diffuses et spéculaires sont reflétées avec toute leur force par n'importe quelle source de lumière. Les sources lumineuses ont également des intensités différentes pour leurs composantes ambiantes, diffuses et spéculaires respectivement. Dans le chapitre précédent, nous avons résolu ce problème en faisant varier les intensités ambiante et spéculaire à l'aide d'une valeur de force. Nous voulons faire quelque chose de similaire, mais cette fois en spécifiant des vecteurs d'intensité pour chacune des composantes de l'éclairage. Si nous visualisions `lightColor` comme `vec3(1.0)`, le code ressemblerait à ceci :
```cpp
vec3 ambient  = vec3(1.0) * material.ambient;
vec3 diffuse  = vec3(1.0) * (diff * material.diffuse);
vec3 specular = vec3(1.0) * (spec * material.specular); 
```
Ainsi, chaque propriété matérielle de l'objet est renvoyée avec une intensité complète pour chaque composante de la lumière. Ces valeurs `vec3(1.0)` peuvent être influencées individuellement pour chaque source de lumière et c'est généralement ce que nous voulons. En ce moment, la composante ambiante de l'objet influence complètement la couleur du cube. La composante ambiante ne devrait pas avoir un impact aussi important sur la couleur finale. Nous pouvons donc restreindre la couleur ambiante en réglant l'intensité ambiante de la lumière sur une valeur plus faible : 
```cpp
vec3 ambient = vec3(0.1) * material.ambient;  
```
Nous pouvons influencer l'intensité diffuse et spéculaire de la source lumineuse de la même manière. Ceci est très similaire à ce que nous avons fait dans le chapitre précédent ; on pourrait dire que nous avons déjà créé des propriétés de lumière pour influencer chaque composant d'éclairage individuellement. Nous voudrons créer quelque chose de similaire à la structure matérielle pour les propriétés de lumière :
```cpp
struct Light {
    vec3 position;
  
    vec3 ambient;
    vec3 diffuse;
    vec3 specular;
};

uniform Light light;  
```
Une source lumineuse a une intensité différente pour ses composantes ambiante, diffuse et spéculaire.

**La lumière ambiante est généralement réglée sur une faible intensité, car nous ne voulons pas que la couleur ambiante soit trop dominante. 
La composante diffuse d'une source lumineuse est généralement réglée sur la couleur exacte que l'on souhaite donner à la lumière, souvent un blanc éclatant. 
La composante spéculaire est généralement maintenue à `vec3(1.0)` et brille à pleine intensité.** Notez que nous avons également ajouté le vecteur de position de la lumière à la structure.  
  
Tout comme pour l'uniforme matériel, nous devons mettre à jour le fragment shader : 
```cpp
vec3 ambient  = light.ambient * material.ambient;
vec3 diffuse  = light.diffuse * (diff * material.diffuse);
vec3 specular = light.specular * (spec * material.specular);  
```
 Nous voulons ensuite définir les intensités lumineuses dans l'application : 
```cpp
lightingShader.setVec3("light.ambient",  0.2f, 0.2f, 0.2f);
lightingShader.setVec3("light.diffuse",  0.5f, 0.5f, 0.5f); // darken diffuse light a bit
lightingShader.setVec3("light.specular", 1.0f, 1.0f, 1.0f); 
```
Maintenant que nous avons modulé l'influence de la lumière sur le matériau de l'objet, nous obtenons un résultat visuel qui ressemble beaucoup à celui du chapitre précédent. Cette fois-ci, cependant, nous avons un contrôle total sur l'éclairage et le matériau de l'objet :
![[material3.png]]
Modifier l'aspect visuel des objets est relativement facile à l'heure actuelle. Mettons un peu de piment dans tout cela ! 
## Des couleurs de lumières différentes
Jusqu'à présent, nous avons utilisé les couleurs de lumière pour faire varier l'intensité de leurs composants individuels en choisissant des couleurs qui vont du blanc au gris en passant par le noir, sans affecter les couleurs réelles de l'objet (seulement son intensité). Puisque nous avons maintenant un accès facile aux propriétés de la lumière, nous pouvons changer leurs couleurs au fil du temps pour obtenir des effets vraiment intéressants. Comme tout est déjà configuré dans le fragment shader, changer les couleurs de la lumière est facile et crée immédiatement des effets amusants : 

TODO: intégrer le .gif

Comme vous pouvez le constater, une couleur de lumière différente influence grandement la couleur de sortie de l'objet. Comme la couleur de la lumière influence directement les couleurs que l'objet peut refléter (comme vous vous en souvenez peut-être dans le chapitre sur les couleurs), elle a un impact significatif sur la sortie visuelle.  
  
Nous pouvons facilement modifier les couleurs de la lumière au fil du temps en changeant les couleurs ambiantes et diffuses de la lumière via `sin` et `glfwGetTime` :
```cpp
glm::vec3 lightColor;
lightColor.x = sin(glfwGetTime() * 2.0f);
lightColor.y = sin(glfwGetTime() * 0.7f);
lightColor.z = sin(glfwGetTime() * 1.3f);
  
glm::vec3 diffuseColor = lightColor   * glm::vec3(0.5f); 
glm::vec3 ambientColor = diffuseColor * glm::vec3(0.2f); 
  
lightingShader.setVec3("light.ambient", ambientColor);
lightingShader.setVec3("light.diffuse", diffuseColor);
```
Essayez et expérimentez plusieurs valeurs d'éclairage et de matériaux et voyez comment elles affectent le résultat visuel. Vous pouvez trouver le code source de l'application [ici](https://learnopengl.com/code_viewer_gh.php?code=src/2.lighting/3.1.materials/materials.cpp).





