# Mesh
Avec Assimp, nous pouvons charger de nombreux modèles différents dans l'application, mais une fois chargés, ils sont tous stockés dans les structures de données d'Assimp. **Ce que nous voulons finalement, c'est transformer ces données dans un format qu'OpenGL comprend afin que nous puissions effectuer le rendu des objets**. Nous avons appris dans le chapitre précédent qu'un mesh représente une seule entité dessinable, alors commençons par définir notre propre classe de mesh.

Revoyons un peu ce que nous avons appris jusqu'à présent pour réfléchir à ce qu'un mesh devrait minimalement avoir comme données. **Un mesh doit au moins avoir un ensemble de sommets, où chaque sommet contient un vecteur de position, un vecteur de normalité et un vecteur de coordonnées de texture**. **Un maillage doit également contenir des indices pour le dessin indexé et des données matérielles sous la forme de textures (maps diffuses/spéculaires).**

Maintenant que nous avons défini les exigences minimales pour une classe de mesh, nous pouvons définir un sommet dans OpenGL :
```cpp
struct Vertex {
    glm::vec3 Position;
    glm::vec3 Normal;
    glm::vec2 TexCoords;
};
```
Nous stockons chacun des attributs de vertex requis dans une structure appelée `Vertex`. En plus de la structure `Vertex`, nous voulons également organiser les données de texture dans une structure `Texture` :
```cpp
struct Texture {
    unsigned int id;
    string type;
}; 
```
Nous stockons l'identifiant de la texture et son type, par exemple une texture diffuse ou spéculaire.

Connaissant la représentation réelle d'un sommet et d'une texture, nous pouvons commencer à définir la structure de la classe de Mesh :
```cpp
class Mesh {
    public:
        // mesh data
        vector<Vertex>       vertices;
        vector<unsigned int> indices;
        vector<Texture>      textures;

        Mesh(vector<Vertex> vertices, vector<unsigned int> indices, vector<Texture> textures);
        void Draw(Shader &shader);
    private:
        //  render data
        unsigned int VAO, VBO, EBO;

        void setupMesh();
};  
```
Comme vous pouvez le voir, la classe n'est pas trop compliquée. Dans le constructeur, nous donnons au mesh toutes les données nécessaires, nous initialisons les buffers dans la fonction `setupMesh`, et enfin nous dessinons le mesh via la fonction `Draw`. Notez que nous donnons un shader à la fonction `Draw` ; en passant le shader au mesh, nous pouvons définir plusieurs uniformes avant de dessiner (comme lier les samplers aux unités de texture).

Le contenu de la fonction du constructeur est assez simple. Nous définissons simplement les variables publiques de la classe avec les variables d'argument correspondantes du constructeur. Nous appelons également la fonction `setupMesh` dans le constructeur :
```cpp
Mesh(vector<Vertex> vertices, vector<unsigned int> indices, vector<Texture> textures)
{
    this->vertices = vertices;
    this->indices = indices;
    this->textures = textures;

    setupMesh();
}
```
Il ne se passe rien de spécial ici. Plongeons maintenant dans la fonction `setupMesh`.

## Initialisation
Grâce au constructeur, nous disposons maintenant de grandes listes de données de mesh que nous pouvons utiliser pour le rendu. Nous devons configurer les buffers appropriés et spécifier la disposition du vertex shader via les pointeurs d'attributs de vertex. Vous ne devriez plus avoir de problème avec ces concepts, mais nous les avons un peu pimentés avec l'introduction des données de vertex dans les structures :
```cpp
void setupMesh()
{
    glGenVertexArrays(1, &VAO);
    glGenBuffers(1, &VBO);
    glGenBuffers(1, &EBO);
  
    glBindVertexArray(VAO);
    glBindBuffer(GL_ARRAY_BUFFER, VBO);

    glBufferData(GL_ARRAY_BUFFER, vertices.size() * sizeof(Vertex), &vertices[0], GL_STATIC_DRAW);  

    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, indices.size() * sizeof(unsigned int), 
                 &indices[0], GL_STATIC_DRAW);

    // vertex positions
    glEnableVertexAttribArray(0);	
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex), (void*)0);
    // vertex normals
    glEnableVertexAttribArray(1);	
    glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex), (void*)offsetof(Vertex, Normal));
    // vertex texture coords
    glEnableVertexAttribArray(2);	
    glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, sizeof(Vertex), (void*)offsetof(Vertex, TexCoords));

    glBindVertexArray(0);
}  
```
Le code n'est pas très différent de ce à quoi on pourrait s'attendre, mais quelques petites astuces ont été utilisées avec l'aide de la structure `Vertex`.

**Les structures ont une grande propriété en C++, à savoir que leur disposition en mémoire est séquentielle**. En d'autres termes, si nous devions représenter une structure comme un tableau de données, elle ne contiendrait que les variables de la structure dans un ordre séquentiel, **ce qui se traduit directement par un tableau de flottants (en fait d'octets) que nous voulons pour un buffer de tableau**. Par exemple, si nous avons une structure Vertex remplie, sa disposition en mémoire serait la suivante :
```cpp
Vertex vertex;
vertex.Position  = glm::vec3(0.2f, 0.4f, 0.6f);
vertex.Normal    = glm::vec3(0.0f, 1.0f, 0.0f);
vertex.TexCoords = glm::vec2(1.0f, 0.0f);
// = [0.2f, 0.4f, 0.6f, 0.0f, 1.0f, 0.0f, 1.0f, 0.0f];
```
Grâce à cette propriété utile, nous pouvons directement passer un pointeur sur une grande liste de structures `Vertex` en tant que données du buffer et elles correspondent parfaitement à ce que `glBufferData` attend en tant qu'argument :
```cpp
glBufferData(GL_ARRAY_BUFFER, vertices.size() * sizeof(Vertex), vertices[0], GL_STATIC_DRAW);    
```
Naturellement, l'opérateur `sizeof` peut également être utilisé sur la structure pour obtenir la taille appropriée en octets. Celle-ci devrait être de 32 octets (8 flottants * 4 octets chacun).

Une autre utilisation intéressante des structures est une directive du préprocesseur appelée `offsetof(s,m)` qui prend comme premier argument une structure et comme second argument un nom de variable de la structure. La macro renvoie l'octet de décalage de cette variable à partir du début de la structure. Cette macro est parfaite pour définir le paramètre offset de la fonction `glVertexAttribPointer` :
```cpp
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex), (void*)offsetof(Vertex, Normal));  
```
Le décalage est maintenant défini à l'aide de la macro `offsetof` qui, dans ce cas, définit le décalage d'octet du vecteur normal égal au décalage d'octet de l'attribut normal dans la structure qui est de 3 flottants et donc de 12 octets.

**L'utilisation d'une telle structure ne nous permet pas seulement d'obtenir un code plus lisible, mais aussi d'étendre facilement la structure.** Si nous voulons un autre attribut de sommet, nous pouvons simplement l'ajouter à la structure et, en raison de sa nature flexible, le code de rendu ne sera pas interrompu.
## Rendu (Rendering)
La dernière fonction que nous devons définir pour que la classe `Mesh` soit complète est la fonction `Draw`. **Avant de rendre le mesh, nous voulons d'abord lier les textures appropriées avant d'appeler `glDrawElements`**. ****Cependant, ceci est quelque peu difficile puisque nous ne savons pas dès le départ combien de textures (s'il y en a) le mesh possède et quel type elles peuvent avoir. Comment définir les unités de texture et les échantillonneurs dans les shaders ?

**Pour résoudre ce problème, nous allons partir d'une certaine convention de nommage** : chaque texture diffuse est nommée `texture_diffuseN`, et chaque texture spéculaire doit être nommée `texture_specularN`, **où N est un nombre quelconque allant de 1 au nombre maximum d'échantillonneurs de textures autorisé**. Supposons que nous ayons 3 textures diffuses et 2 textures spéculaires pour un maillage particulier, leurs échantillonneurs de texture devraient alors être appelés :
```cpp
uniform sampler2D texture_diffuse1;
uniform sampler2D texture_diffuse2;
uniform sampler2D texture_diffuse3;
uniform sampler2D texture_specular1;
uniform sampler2D texture_specular2;
```
Grâce à cette convention, nous pouvons définir autant de samplers de textures que nous le souhaitons dans les shaders (jusqu'au maximum d'OpenGL) et si un maillage contient effectivement (autant) de textures, nous savons quels seront leurs noms. **Grâce à cette convention, nous pouvons traiter n'importe quel nombre de textures sur un seul maillage et le développeur de shaders est libre d'en utiliser autant qu'il le souhaite en définissant les samplers appropriés.**

>Il existe de nombreuses solutions à ce type de problème et si vous n'aimez pas cette solution particulière, c'est à vous de faire preuve de créativité et de trouver votre propre approche.

Le code de dessin résultant devient alors :
```cpp
void Draw(Shader &shader) 
{
    unsigned int diffuseNr = 1;
    unsigned int specularNr = 1;
    for(unsigned int i = 0; i < textures.size(); i++)
    {
        glActiveTexture(GL_TEXTURE0 + i); // activate proper texture unit before binding
        // retrieve texture number (the N in diffuse_textureN)
        string number;
        string name = textures[i].type;
        if(name == "texture_diffuse")
            number = std::to_string(diffuseNr++);
        else if(name == "texture_specular")
            number = std::to_string(specularNr++);

        shader.setInt(("material." + name + number).c_str(), i);
        glBindTexture(GL_TEXTURE_2D, textures[i].id);
    }
    glActiveTexture(GL_TEXTURE0);

    // draw mesh
    glBindVertexArray(VAO);
    glDrawElements(GL_TRIANGLES, indices.size(), GL_UNSIGNED_INT, 0);
    glBindVertexArray(0);
}

```
Nous calculons d'abord la composante **N** par type de texture et la concaténons à la chaîne de type de la texture pour obtenir le nom d'uniforme approprié. Nous localisons ensuite le sampler approprié, lui donnons la valeur d'emplacement correspondant à l'unité de texture actuellement active et lions la texture. **C'est également la raison pour laquelle nous avons besoin du shader dans la fonction Draw.**

Nous avons également ajouté "`material.`" au nom de l'uniforme résultant parce que nous stockons généralement les textures dans une structure matérielle (cela peut différer selon l'implémentation).

>Notez que nous incrémentons les compteurs diffus et spéculaires au moment où nous les convertissons en chaînes de caractères. En C++, l'appel à l'incrémentation : **variable++ renvoie la variable telle quelle, puis l'incrémente, tandis que ++variable incrémente d'abord la variable, puis la renvoie.** Dans notre cas, la valeur transmise à `std::string` est la valeur originale du compteur. Ensuite, la valeur est incrémentée pour le prochain tour.

Vous pouvez trouver le code source complet de la classe `Mesh` [ici](https://learnopengl.com/code_viewer_gh.php?code=includes/learnopengl/mesh.h).

La classe `Mesh` que nous venons de définir est une **abstraction** pour de nombreux sujets que nous avons abordés dans les premiers chapitres. Dans le prochain chapitre, nous créerons un modèle qui servira de conteneur pour plusieurs objets mesh et qui implémentera l'interface de chargement d'Assimp.
