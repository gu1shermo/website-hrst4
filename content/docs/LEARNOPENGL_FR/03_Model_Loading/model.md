# Modèle
Il est maintenant temps de mettre la main à la pâte avec Assimp et de commencer à créer le code de chargement et de traduction. L'objectif de ce chapitre est de créer une autre classe qui représente un modèle dans son intégralité, c'est-à-dire un modèle qui contient plusieurs meshes, éventuellement avec plusieurs textures. **Une maison qui contient un balcon en bois, une tour et peut-être une piscine peut être chargée comme un seul modèle**. Nous chargerons le modèle via Assimp et le traduirons en plusieurs objets Mesh que nous avons créés dans le chapitre précédent.

Sans plus attendre, je vous présente la structure de la classe Model :
```cpp
class Mode
{
    public:
        Model(char *path)
        {
            loadModel(path);
        }
        void Draw(Shader &shader);	
    private:
        // model data
        vector<Mesh> meshes;
        string directory;

        void loadModel(string path);
        void processNode(aiNode *node, const aiScene *scene);
        Mesh processMesh(aiMesh *mesh, const aiScene *scene);
        vector<Texture> loadMaterialTextures(aiMaterial *mat, aiTextureType type, 
                                             string typeName);
};
```
La classe Model contient un vecteur d'objets Mesh et nécessite que nous lui indiquions l'emplacement d'un fichier dans son constructeur. Elle charge ensuite le fichier immédiatement via la fonction `loadModel` qui est appelée dans le constructeur. Les fonctions privées sont toutes conçues pour traiter une partie de la routine d'importation d'Assimp et nous les couvrirons bientôt. Nous stockons également le répertoire du chemin d'accès au fichier dont nous aurons besoin plus tard pour charger les textures.
La fonction Draw n'a rien de spécial et passe en boucle sur chacun des meshes pour appeler leur fonction Draw respective :
```cpp
void Draw(Shader &shader)
{
    for(unsigned int i = 0; i < meshes.size(); i++)
        meshes[i].Draw(shader);
}  
```
## Importer un modèle 3D dans OpenGL
Pour importer un modèle et le traduire dans notre propre structure, nous devons d'abord inclure les en-têtes appropriés d'Assimp :
```cpp
#include <assimp/Importer.hpp>
#include <assimp/scene.h>
#include <assimp/postprocess.h>
```
La première fonction que nous appelons est `loadModel`, qui est directement appelée depuis le constructeur. Dans `loadModel`, nous utilisons Assimp pour charger le modèle dans une structure de données d'Assimp appelée objet scène (scene object). Vous vous souvenez peut-être du premier chapitre de la série sur le chargement de modèles, qui expliquait qu'il s'agissait de l'objet racine de l'interface de données d'Assimp. Une fois que nous avons l'objet scène, nous pouvons accéder à toutes les données dont nous avons besoin à partir du modèle chargé.

L'avantage d'Assimp est qu'il fait abstraction de tous les détails techniques liés au chargement des différents formats de fichiers et qu'il fait tout cela en une seule ligne :
```cpp
Assimp::Importer importer;
const aiScene *scene = importer.ReadFile(path, aiProcess_Triangulate | aiProcess_FlipUVs);
```
Nous commençons par déclarer un objet `Importer` dans l'espace de noms d'Assimp, puis nous appelons sa fonction `ReadFile`. La fonction attend un chemin d'accès au fichier et plusieurs options de post-traitement comme second argument. Assimp nous permet de spécifier plusieurs options qui obligent Assimp à effectuer des calculs/opérations supplémentaires sur les données importées. En définissant `aiProcess_Triangulate`, nous indiquons à Assimp que si le modèle n'est pas (entièrement) constitué de triangles, il doit d'abord transformer toutes les formes primitives du modèle en triangles. L'option `aiProcess_FlipUVs` inverse les coordonnées de la texture sur l'axe y lorsque cela est nécessaire pendant le traitement (vous vous souvenez peut-être du chapitre sur les textures que la plupart des images en OpenGL étaient inversées autour de l'axe y ; cette petite option de post-traitement corrige cela pour nous). Quelques autres options utiles sont:
- `aiProcess_GenNormals` : crée des vecteurs normaux pour chaque sommet si le modèle ne contient pas de vecteurs normaux.
- `aiProcess_SplitLargeMeshes` : divise le grands meshes en sous-meshes plus petits, ce qui est utile si votre rendu a un nombre maximum de sommets autorisés et ne peut traiter que des meshes plus petits.
- `aiProcess_OptimizeMeshes` : fait l'inverse en essayant de joindre plusieurs meshes en un mesh plus grand, réduisant les appels au dessin pour l'optimisation.

Assimp propose un grand nombre d'options de post-traitement que vous pouvez trouver ici. Le chargement d'un modèle via Assimp est (comme vous pouvez le voir) étonnamment facile. Le plus dur est d'utiliser l'objet scène retourné pour traduire les données chargées en un tableau d'objets Mesh.
La fonction `loadModel` complète est présentée ici :
```cpp
void loadModel(string path)
{
    Assimp::Importer import;
    const aiScene *scene = import.ReadFile(path, aiProcess_Triangulate | aiProcess_FlipUVs);	
	
    if(!scene || scene->mFlags & AI_SCENE_FLAGS_INCOMPLETE || !scene->mRootNode) 
    {
        cout << "ERROR::ASSIMP::" << import.GetErrorString() << endl;
        return;
    }
    directory = path.substr(0, path.find_last_of('/'));

    processNode(scene->mRootNode, scene);
}  
```
Après avoir chargé le modèle, nous vérifions que la scène et le nœud racine de la scène ne sont pas `null` et nous vérifions l'un de ses flags pour voir si les données renvoyées sont incomplètes. Si l'une de ces conditions d'erreur est remplie, nous signalons l'erreur récupérée dans la fonction `GetErrorString` de l'importateur et nous faisons un return. Nous récupérons également le chemin d'accès au répertoire du fichier donné.  
  
Si rien ne s'est mal passé, nous voulons traiter tous les nœuds de la scène. Nous passons le premier nœud (nœud racine) à la fonction récursive `processNode`. Comme chaque nœud contient (éventuellement) un ensemble d'enfants, nous voulons d'abord traiter le nœud en question, puis continuer à traiter tous les enfants du nœud et ainsi de suite. Cela correspond à une structure récursive, nous allons donc définir une fonction récursive. **Une fonction récursive est une fonction qui effectue un traitement et appelle de manière récursive la même fonction avec différents paramètres jusqu'à ce qu'une certaine condition soit remplie**. Dans notre cas, la condition de sortie est remplie lorsque tous les nœuds ont été traités.  
  
Comme vous vous en souvenez peut-être dans la structure d'Assimp, chaque nœud contient un ensemble d'indices de meshes où chaque indice pointe vers un mesh spécifique situé dans l'objet de la scène. Nous voulons donc récupérer ces indices de mesh, récupérer chaque mesh, traiter chaque mesh, puis recommencer l'opération pour chacun des nœuds enfants du nœud. Le contenu de la fonction `processNode` est illustré ci-dessous :  
```cpp
void processNode(aiNode *node, const aiScene *scene)
{
    // process all the node's meshes (if any)
    for(unsigned int i = 0; i < node->mNumMeshes; i++)
    {
        aiMesh *mesh = scene->mMeshes[node->mMeshes[i]]; 
        meshes.push_back(processMesh(mesh, scene));			
    }
    // then do the same for each of its children
    for(unsigned int i = 0; i < node->mNumChildren; i++)
    {
        processNode(node->mChildren[i], scene);
    }
}  
```
Nous vérifions d'abord les indices de mesh de chaque nœud et récupérons le mesh correspondant en indexant le tableau `mMeshes` de la scène. Le mesh retourné est ensuite passé à la fonction `processMesh` qui retourne un objet **Mesh** que nous pouvons stocker dans la liste/le vecteur `meshes`.  
  
Une fois que tous les meshes ont été traités, nous parcourons tous les enfants du nœud et appelons la même fonction `processNode` pour chacun de ses enfants. Lorsqu'un nœud n'a plus d'enfants, la récursivité s'arrête. 

>Un lecteur attentif aura peut-être remarqué que nous pourrions oublier de traiter les nœuds et simplement parcourir en boucle tous les meshes de la scène directement, sans faire toutes ces choses compliquées avec les indices. La raison pour laquelle nous faisons cela est que l'idée initiale d'utiliser des nœuds comme celui-ci est qu'ils définissent une relation parent-enfant entre les meshes. En itérant récursivement à travers ces relations, nous pouvons définir certains meshes comme étant les parents d'autres meshes. 
>Un exemple d'utilisation d'un tel système est lorsque vous voulez traduire un mesh de voiture et vous assurer que tous ses enfants (comme un mesh de moteur, un mesh de volant, et ses meshes de pneus) se traduisent également ; un tel système est facilement créé en utilisant les relations parent-enfant.  
  
  >Pour l'instant, nous n'utilisons pas un tel système, mais il est généralement recommandé de conserver cette approche lorsque vous souhaitez exercer un contrôle supplémentaire sur les données de votre maillage. Ces relations de type nœud sont après tout définies par les artistes qui ont créé les modèles.  

 L'étape suivante consiste à traiter les données d'Assimp dans la classe **Mesh** du chapitre précédent.

## Assimp vers Mesh
Traduire un objet `aiMesh` en un objet mesh qui nous est propre n'est pas très difficile. Tout ce que nous avons à faire, c'est d'accéder à chacune des propriétés pertinentes du mesh et de les stocker dans notre propre objet. La structure générale de la fonction `processMesh` devient alors :

```cpp
Mesh processMesh(aiMesh *mesh, const aiScene *scene)
{
    vector<Vertex> vertices;
    vector<unsigned int> indices;
    vector<Texture> textures;

    for(unsigned int i = 0; i < mesh->mNumVertices; i++)
    {
        Vertex vertex;
        // process vertex positions, normals and texture coordinates
        [...]
        vertices.push_back(vertex);
    }
    // process indices
    [...]
    // process material
    if(mesh->mMaterialIndex >= 0)
    {
        [...]
    }

    return Mesh(vertices, indices, textures);
}  
```
**Le traitement d'un mesh est un processus en trois parties** : **récupération de toutes les données des sommets, récupération des indices du mesh et, enfin, récupération des données matérielles pertinentes. (matériau)** Les données traitées sont stockées dans l'un des trois vecteurs et, à partir de ceux-ci, un mesh est créé et renvoyé à l'appelant de la fonction.  
  
La récupération des données des sommets est assez simple : nous définissons une structure `Vertex` que nous ajoutons au tableau des sommets après chaque itération de la boucle. Nous bouclons pour autant de sommets qu'il existe dans le mesh  (récupérés via `mesh->mNumVertices`). Au cours de l'itération, nous voulons remplir cette structure avec toutes les données pertinentes. Pour les positions des sommets, cela se fait de la manière suivante :
```cpp
glm::vec3 vector; 
vector.x = mesh->mVertices[i].x;
vector.y = mesh->mVertices[i].y;
vector.z = mesh->mVertices[i].z; 
vertex.Position = vector;
```
Notez que nous définissons un `vec3` temporaire pour transférer les données d'Assimp. Ceci est nécessaire car Assimp maintient ses propres types de données pour les vecteurs, les matrices, les chaînes de caractères, etc. et ils ne se convertissent pas très bien aux types de données de ``glm``. 

> Assimp appelle son tableau de positions de vertex `mVertices`, ce qui n'est pas le nom le plus intuitif. 

 La procédure pour les normales n'est plus une surprise : 
```cpp
vector.x = mesh->mNormals[i].x;
vector.y = mesh->mNormals[i].y;
vector.z = mesh->mNormals[i].z;
vertex.Normal = vector;  
```
Les coordonnées de texture sont à peu près les mêmes, mais Assimp permet à un modèle d'avoir jusqu'à 8 coordonnées de texture différentes par vertex. Nous n'allons pas en utiliser 8, nous ne nous intéresserons qu'au premier jeu de coordonnées de texture. Nous voudrons également vérifier si le mesh contient effectivement des coordonnées de texture (ce qui n'est pas toujours le cas) :
```cpp
if(mesh->mTextureCoords[0]) // does the mesh contain texture coordinates?
{
    glm::vec2 vec;
    vec.x = mesh->mTextureCoords[0][i].x; 
    vec.y = mesh->mTextureCoords[0][i].y;
    vertex.TexCoords = vec;
}
else
    vertex.TexCoords = glm::vec2(0.0f, 0.0f);  
```
La structure vertex est maintenant complètement remplie avec les attributs de vertex requis et nous pouvons la pousser à l'arrière du vecteur `vertices` à la fin de l'itération. Ce processus est répété pour chaque sommet du mesh. 

## Indices
L'interface d'Assimp définit chaque mesh comme ayant un tableau de faces, où chaque face représente une seule primitive, qui dans notre cas (grâce à l'option `aiProcess_Triangulate`) sont toujours des triangles. Une face contient les indices des sommets que nous devons dessiner dans quel ordre pour sa primitive. Donc si nous itérons sur toutes les faces et stockons tous les indices des faces dans le vecteur `indices`, tout est prêt :
```cpp
for(unsigned int i = 0; i < mesh->mNumFaces; i++)
{
    aiFace face = mesh->mFaces[i];
    for(unsigned int j = 0; j < face.mNumIndices; j++)
        indices.push_back(face.mIndices[j]);
}  
```
Une fois la boucle externe terminée, nous disposons d'un ensemble complet de sommets et de données d'index pour dessiner le mesh via `glDrawElements`. Cependant, pour terminer la discussion et ajouter quelques détails au mesh, nous voulons également traiter le matériau du mesh. 

## Matériau
Comme les nœuds, un mesh ne contient qu'un index vers un objet matériau. Pour récupérer le matériau d'un mesh, nous devons indexer le tableau `mMaterials` de la scène. L'index du matériau du mesh est défini dans sa propriété `mMaterialIndex`, que nous pouvons également interroger pour vérifier si le mesh contient un matériau ou non :
```cpp
if(mesh->mMaterialIndex >= 0)
{
    aiMaterial *material = scene->mMaterials[mesh->mMaterialIndex];
    vector<Texture> diffuseMaps = loadMaterialTextures(material, 
                                        aiTextureType_DIFFUSE, "texture_diffuse");
    textures.insert(textures.end(), diffuseMaps.begin(), diffuseMaps.end());
    vector<Texture> specularMaps = loadMaterialTextures(material, 
                                        aiTextureType_SPECULAR, "texture_specular");
    textures.insert(textures.end(), specularMaps.begin(), specularMaps.end());
}  
```

Nous récupérons d'abord l'objet `aiMaterial` dans le tableau `mMaterials` de la scène. Ensuite, nous voulons charger les textures diffuses et/ou spéculaires du mesh. Un objet matériel stocke en interne un tableau d'emplacements de texture pour chaque type de texture. Les différents types de texture sont tous préfixés par `aiTextureType_`. Nous utilisons une fonction d'aide appelée `loadMaterialTextures` pour récupérer, charger et initialiser les textures du matériau. La fonction renvoie un vecteur de structures `Texture` que nous stockons à la fin du vecteur de textures du modèle.  
  
La fonction `loadMaterialTextures` itère sur tous les emplacements de texture du type de texture donné, récupère l'emplacement du fichier de texture, puis charge et génère la texture et stocke les informations dans une structure `Vertex`. Voici à quoi cela ressemble :
```cpp
vector<Texture> loadMaterialTextures(aiMaterial *mat, aiTextureType type, string typeName)
{
    vector<Texture> textures;
    for(unsigned int i = 0; i < mat->GetTextureCount(type); i++)
    {
        aiString str;
        mat->GetTexture(type, i, &str);
        Texture texture;
        texture.id = TextureFromFile(str.C_Str(), directory);
        texture.type = typeName;
        texture.path = str;
        textures.push_back(texture);
    }
    return textures;
}  
```
Nous vérifions d'abord la quantité de textures stockées dans le matériau via la fonction `GetTextureCount` qui attend un des types de texture que nous avons donnés. Nous récupérons l'emplacement de chaque fichier de texture via la fonction `GetTexture` qui stocke le résultat dans une chaîne de caractères `aiString`. Nous utilisons ensuite une autre fonction d'aide appelée `TextureFromFile` qui charge une texture (avec `stb_image.h`) pour nous et renvoie l'ID de la texture. Vous pouvez consulter la liste complète du code à la fin pour connaître son contenu si vous n'êtes pas sûr de la façon dont une telle fonction est écrite.

>Notez que nous supposons que les chemins des fichiers de texture dans les fichiers de modèle sont locaux à l'objet de modèle réel, c'est-à-dire dans le même répertoire que l'emplacement du modèle lui-même. Nous pouvons alors simplement concaténer la chaîne de l'emplacement de la texture et la chaîne du répertoire que nous avons récupérée plus tôt (dans la fonction `loadModel`) pour obtenir le chemin complet de la texture (c'est pourquoi la fonction `GetTexture` a également besoin de la chaîne du répertoire).  
  
>Certains modèles trouvés sur Internet utilisent des chemins absolus pour l'emplacement de leurs textures, ce qui ne fonctionnera pas sur toutes les machines. Dans ce cas, vous voudrez probablement éditer manuellement le fichier pour utiliser des chemins locaux pour les textures (si possible).

 Et c'est tout ce qu'il y a à faire pour importer un modèle avec Assimp. 
## Une optimisation
Nous n'avons pas encore tout à fait terminé, puisqu'il reste une optimisation importante (mais pas complètement nécessaire) à faire. La plupart des scènes réutilisent plusieurs de leurs textures sur plusieurs meshes ; pensez à nouveau à une maison dont les murs sont recouverts d'une texture de granit. Cette texture pourrait également être appliquée au sol, aux plafonds, à l'escalier, à une table et peut-être même à un petit puits situé à proximité. Le chargement des textures est une opération coûteuse et, dans notre implémentation actuelle, une nouvelle texture est chargée et générée pour chaque mesh, même si la même texture a pu être chargée plusieurs fois auparavant. Cela devient rapidement le goulot d'étranglement (bottleneck) de votre implémentation de chargement de modèle.  
  
Nous allons donc ajouter une petite modification au code du modèle en stockant globalement toutes les textures chargées. Chaque fois que nous voulons charger une texture, nous vérifions d'abord si elle n'a pas déjà été chargée. Si c'est le cas, nous prenons cette texture et sautons toute la routine de chargement, ce qui nous permet d'économiser beaucoup de puissance de traitement. Pour pouvoir comparer les textures, nous devons également stocker leur chemin d'accès :
```cpp
struct Texture {
    unsigned int id;
    string type;
    string path;  // we store the path of the texture to compare with other textures
};
```
Nous stockons ensuite toutes les textures chargées dans un autre vecteur déclaré en haut du fichier de classe du modèle en tant que variable privée :
```cpp
vector<Texture> textures_loaded; 
```
Dans la fonction `loadMaterialTextures`, nous voulons comparer le chemin de texture avec toutes les textures dans le vecteur `textures_loaded` pour voir si le chemin de texture actuel est égal à l'une d'entre elles. Si c'est le cas, nous sautons la partie chargement/génération de la texture et utilisons simplement la structure de texture trouvée comme texture du mesh. La fonction (mise à jour) est présentée ci-dessous :
```cpp
vector<Texture> loadMaterialTextures(aiMaterial *mat, aiTextureType type, string typeName)
{
    vector<Texture> textures;
    for(unsigned int i = 0; i < mat->GetTextureCount(type); i++)
    {
        aiString str;
        mat->GetTexture(type, i, &str);
        bool skip = false;
        for(unsigned int j = 0; j < textures_loaded.size(); j++)
        {
            if(std::strcmp(textures_loaded[j].path.data(), str.C_Str()) == 0)
            {
                textures.push_back(textures_loaded[j]);
                skip = true; 
                break;
            }
        }
        if(!skip)
        {   // if texture hasn't been loaded already, load it
            Texture texture;
            texture.id = TextureFromFile(str.C_Str(), directory);
            texture.type = typeName;
            texture.path = str.C_Str();
            textures.push_back(texture);
            textures_loaded.push_back(texture); // add to loaded textures
        }
    }
    return textures;
}  
```
>Certaines versions d'Assimp ont tendance à charger les modèles assez lentement lorsque vous utilisez la version debug et/ou le mode debug de votre IDE, alors assurez-vous de le tester avec les versions release si vous rencontrez des temps de chargement lents.

Vous pouvez trouver le code source complet de la classe `Model` [ici](https://learnopengl.com/code_viewer_gh.php?code=includes/learnopengl/model.h). 

## Plus de conteneurs!
Alors donnons un coup de pouce à notre implémentation en important un modèle créé par de véritables artistes, et non par le génie créatif que je suis. Parce que je ne veux pas me donner trop de crédit, je vais occasionnellement permettre à d'autres artistes de rejoindre les rangs et cette fois nous allons charger cet incroyable [Survival Guitar Backpack](https://sketchfab.com/3d-models/survival-guitar-backpack-low-poly-799f8c4511f84fab8c3f12887f7e6b36) de Berk Gedik. J'ai un peu modifié le matériau et les chemins pour qu'il fonctionne directement avec la façon dont nous avons configuré le chargement du modèle. Le modèle est exporté sous la forme d'un fichier `.obj` accompagné d'un fichier `.mtl` qui renvoie aux maps de diffuse, spéculaire et normale du modèle (nous y reviendrons plus tard). Vous pouvez télécharger le modèle ajusté pour ce chapitre [ici](https://learnopengl.com/data/models/backpack.zip). Notez qu'il y a quelques types de textures supplémentaires que nous n'utiliserons pas encore, et que toutes les textures et le(s) fichier(s) du modèle doivent être situés dans le même répertoire pour que les textures puissent être chargées.

> La version modifiée du sac à dos utilise des chemins de texture relatifs locaux et renomme les textures albédo et métallique en diffus et spéculaire respectivement. 

Déclarez maintenant un objet `Model` et indiquez l'emplacement du fichier du modèle. Le modèle devrait alors se charger automatiquement et (s'il n'y a pas d'erreurs) rendre l'objet dans la boucle de rendu en utilisant sa fonction `Draw` et c'est tout. Plus d'allocations de tampon, de pointeurs d'attributs et de commandes de rendu, juste une simple ligne de code. Si vous créez un ensemble simple de shaders où le shader de fragment ne produit que la texture diffuse de l'objet, le résultat ressemble un peu à ceci :
![[model_diffuse.png]]

Vous pouvez trouver le code source complet [ici](https://learnopengl.com/code_viewer_gh.php?code=src/3.model_loading/1.model_loading/model_loading.cpp). Notez que nous demandons à `stb_image.h` de retourner les textures verticalement, si vous ne l'avez pas déjà fait, avant de charger le modèle. Dans le cas contraire, les textures auront l'air tout chamboulé.  
  
Nous pouvons également être plus créatifs et introduire des lumières ponctuelles dans l'équation de rendu, comme nous l'avons appris dans les chapitres sur l'éclairage, et avec les maps de spéculaires, nous obtiendrons des résultats étonnants :
![[model_lighting.png]]
Même moi, je dois admettre que c'est peut-être un peu plus fantaisiste que les conteneurs que nous avons utilisés jusqu'à présent. Assimp permet de charger des tonnes de modèles trouvés sur Internet. Il existe un grand nombre de sites Internet qui proposent des modèles 3D gratuits à télécharger dans différents formats de fichiers. Notez que certains modèles ne se chargent pas correctement, ont des chemins de texture qui ne fonctionnent pas, ou sont simplement exportés dans un format qu'Assimp ne peut pas lire.

## Pour aller plus loin
- [How-To Texture Wavefront (.obj) Models for OpenGL](https://www.youtube.com/watch?v=4DQquG_o-Ac): excellent guide vidéo par Matthew Early sur la façon de configurer les modèles 3D dans Blender pour qu'ils fonctionnent directement avec le chargeur de modèles actuel (car la configuration des textures que nous avons choisie ne fonctionne pas toujours à l'extérieur de la boîte).