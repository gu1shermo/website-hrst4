---
title: "advanced data"
date: 2023-11-11
author: hrst4
tags: ['cg','opengl','graphics','cpp']
draft: false
---

# Data avancée
Dans la plupart des chapitres, nous avons largement utilisé les tampons dans OpenGL pour stocker des données sur le GPU. **Dans ce chapitre, nous discuterons brièvement de quelques approches alternatives pour gérer les tampons.**

**Un tampon dans OpenGL est, à la base, un objet qui gère une partie de la mémoire du GPU et rien de plus. Nous donnons un sens à un tampon en le liant à une cible spécifique. Un tampon n'est qu'un tampon de tableau de vertex lorsque nous le lions à `GL_ARRAY_BUFFER`, mais nous pourrions tout aussi bien le lier à `GL_ELEMENT_ARRAY_BUFFER`. OpenGL stocke en interne une référence au tampon par cible et, en fonction de la cible, traite le tampon différemment.**

Jusqu'à présent, nous avons rempli la mémoire du tampon en appelant `glBufferData`, **qui alloue une partie de la mémoire du GPU et ajoute des données dans cette mémoire**. Si nous devions passer NULL comme argument de données, la fonction ne ferait qu'allouer de la mémoire et ne la remplirait pas. Ceci est utile si nous voulons d'abord réserver une quantité spécifique de mémoire et revenir plus tard à ce tampon.

Au lieu de remplir la totalité du tampon en un seul appel de fonction, nous pouvons également remplir des régions spécifiques du tampon en appelant `glBufferSubData`. **Cette fonction attend comme arguments une cible de tampon, un décalage, la taille des données et les données réelles.** Ce qui est nouveau avec cette fonction, c'est que nous pouvons maintenant donner un décalage qui spécifie l'endroit à partir duquel nous voulons remplir la mémoire tampon. Cela nous permet de n'insérer/mettre à jour que certaines parties de la mémoire tampon. Notez que le tampon doit avoir suffisamment de mémoire allouée, de sorte qu'un appel à `glBufferData` est nécessaire avant d'appeler `glBufferSubData` sur le tampon.
```cpp
glBufferSubData(GL_ARRAY_BUFFER, 24, sizeof(data), &data); // Range: [24, 24 + sizeof(data)]
```
Une autre méthode pour obtenir des données dans un tampon est de demander un pointeur vers la mémoire du tampon et de copier directement les données en mémoire. En appelant `glMapBuffer`, OpenGL renvoie un pointeur vers la mémoire du tampon actuellement lié pour que nous puissions l'utiliser :
```cpp
float data[] = {
  0.5f, 1.0f, -0.35f
  [...]
};
glBindBuffer(GL_ARRAY_BUFFER, buffer);
// get pointer
void *ptr = glMapBuffer(GL_ARRAY_BUFFER, GL_WRITE_ONLY);
// now copy data into memory
memcpy(ptr, data, sizeof(data));
// make sure to tell OpenGL we're done with the pointer
glUnmapBuffer(GL_ARRAY_BUFFER);
```

En indiquant à OpenGL que nous en avons fini avec les opérations de pointeur via `glUnmapBuffer`, OpenGL sait que vous avez terminé. En annulant le mappage, le pointeur devient invalide et la fonction renvoie `GL_TRUE` si OpenGL a été capable de mapper vos données avec succès dans le tampon.

L'utilisation de `glMapBuffer` est utile pour mapper directement des données dans un tampon, sans les stocker d'abord dans une mémoire temporaire. Pensez à lire directement les données d'un fichier et à les copier dans la mémoire du tampon.

## Batching vertex attributes
L'utilisation de `glVertexAttribPointer` nous a permis de spécifier la disposition des attributs du contenu de la mémoire tampon du tableau de vertex. Dans la mémoire tampon du tableau de vertex, nous avons entrelacé les attributs, c'est-à-dire que nous avons placé les coordonnées de position, de normale et/ou de texture les unes à côté des autres dans la mémoire pour chaque vertex. Maintenant que nous en savons un peu plus sur les tampons, nous pouvons adopter une approche différente.

Nous pourrions également regrouper toutes les données vectorielles en gros morceaux par type d'attribut au lieu de les entrelacer. **Au lieu d'une disposition entrelacée 123123123123, nous adoptons une approche par lots 111122223333.**

Lorsque vous chargez des données de vertex à partir d'un fichier, vous récupérez généralement un tableau de positions, un tableau de normales et/ou un tableau de coordonnées de texture. La combinaison de ces tableaux en un seul grand tableau de données entrelacées peut s'avérer coûteuse. L'approche par lots est alors une solution plus simple que nous pouvons facilement mettre en œuvre à l'aide de `glBufferSubData` :
```cpp
float positions[] = { ... };
float normals[] = { ... };
float tex[] = { ... };
// fill buffer
glBufferSubData(GL_ARRAY_BUFFER, 0, sizeof(positions), &positions);
glBufferSubData(GL_ARRAY_BUFFER, sizeof(positions), sizeof(normals), &normals);
glBufferSubData(GL_ARRAY_BUFFER, sizeof(positions) + sizeof(normals), sizeof(tex), &tex);
```

De cette manière, nous pouvons directement transférer les tableaux d'attributs dans le tampon sans avoir à les traiter au préalable. Nous aurions également pu les combiner dans un grand tableau et remplir le tampon immédiatement à l'aide de `glBufferData`, mais l'utilisation de `glBufferSubData` se prête parfaitement à ce genre de tâches.

Nous devrons également mettre à jour les pointeurs d'attributs de vertex pour refléter ces changements :

```cpp
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), 0);  
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)(sizeof(positions)));  
glVertexAttribPointer(
  2, 2, GL_FLOAT, GL_FALSE, 2 * sizeof(float), (void*)(sizeof(positions) + sizeof(normals))); 
```
Notez que le paramètre `stride` est égal à la taille de l'attribut de sommet, puisque le vecteur d'attribut de sommet suivant peut être trouvé directement après ses 3 (ou 2) composants.

Cela nous donne encore une autre approche pour définir et spécifier les attributs de sommet. Il est possible d'utiliser l'une ou l'autre approche, il s'agit surtout d'une manière plus organisée de définir les attributs de sommet. **Cependant, l'approche entrelacée reste l'approche recommandée car les attributs de vertex pour chaque exécution du shader de vertex sont alors étroitement alignés en mémoire.**

## Copie des tampons
Une fois que vos tampons sont remplis de données, vous pouvez souhaiter partager ces données avec d'autres tampons ou copier le contenu du tampon dans un autre tampon. La fonction `glCopyBufferSubData` nous permet de copier les données d'un tampon vers un autre tampon avec une relative facilité. Le prototype de la fonction est le suivant :
```cpp
void glCopyBufferSubData(GLenum readtarget, GLenum writetarget, GLintptr readoffset,
                         GLintptr writeoffset, GLsizeiptr size);
```
Les paramètres `readtarget` et `writetarget` doivent indiquer les cibles de tampon à partir desquelles et vers lesquelles nous voulons effectuer la copie. Nous pouvons par exemple copier un tampon `VERTEX_ARRAY_BUFFER` vers un tampon `VERTEX_ELEMENT_ARRAY_BUFFER` en spécifiant ces cibles comme cibles de lecture et d'écriture respectivement. Les tampons actuellement liés à ces cibles seront alors affectés.

Mais que se passe-t-il si nous voulons lire et écrire des données dans deux tampons différents qui sont tous deux des tampons de tableaux de vertex ? Nous ne pouvons pas lier deux tampons en même temps à la même cible de tampon. C'est pour cette raison, et uniquement pour cette raison, qu'OpenGL nous donne deux cibles de tampon supplémentaires appelées `GL_COPY_READ_BUFFER` et `GL_COPY_WRITE_BUFFER`. Nous lions ensuite les tampons de notre choix à ces nouvelles cibles de tampons et définissons ces cibles en tant qu'argument `readtarget` et `writetarget`.

`glCopyBufferSubData` lit alors des données d'une taille donnée à partir d'un `readoffset` donné et les écrit dans le tampon `writetarget` à `writeoffset`. Un exemple de copie du contenu de deux tampons de tableau de vertex est illustré ci-dessous :

```cpp
glBindBuffer(GL_COPY_READ_BUFFER, vbo1);
glBindBuffer(GL_COPY_WRITE_BUFFER, vbo2);
glCopyBufferSubData(GL_COPY_READ_BUFFER, GL_COPY_WRITE_BUFFER, 0, 0, 8 * sizeof(float));
```
Nous aurions également pu le faire en liant uniquement le tampon de la cible d'écriture à l'un des nouveaux types de cibles de tampon :
```cpp
float vertexData[] = { ... };
glBindBuffer(GL_ARRAY_BUFFER, vbo1);
glBindBuffer(GL_COPY_WRITE_BUFFER, vbo2);
glCopyBufferSubData(GL_ARRAY_BUFFER, GL_COPY_WRITE_BUFFER, 0, 0, 8 * sizeof(float)); 
```
Avec quelques connaissances supplémentaires sur la façon de manipuler les tampons, nous pouvons déjà les utiliser de façon plus intéressante. Plus vous avancez dans OpenGL, plus ces nouvelles méthodes de tampons deviennent utiles. Dans le prochain chapitre, où nous discuterons des objets tampons uniformes, nous ferons bon usage de `glBufferSubData`.
