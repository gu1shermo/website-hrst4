---
title: "glossary"
date: 2023-11-11
author: hrst4
tags: ['cg','opengl','graphics','cpp']
draft: false
---
# Glossaire
- `OpenGL` : une spécification formelle d'une API graphique qui définit la disposition et la sortie de chaque fonction.  
- `GLAD` : une bibliothèque de chargement d'extension qui charge et définit tous les pointeurs de fonction d'OpenGL pour nous afin que nous puissions utiliser toutes les fonctions (modernes) d'OpenGL.  
- `Viewport` : la région de la fenêtre 2D dans laquelle nous effectuons le rendu.  
- `Graphics Pipeline` : le processus complet par lequel les vertices doivent passer avant de se retrouver sous la forme d'un ou plusieurs pixels sur l'écran.  
- `Shader` : un petit programme qui tourne sur la carte graphique. Plusieurs étapes du pipeline graphique peuvent utiliser des shaders créés par l'utilisateur pour remplacer les fonctionnalités existantes.  
- `Vertex` : une collection de données qui représentent un point unique.  
- `Normalized Device Coordinates (NDC)` : le système de coordonnées dans lequel vos sommets se retrouvent après que la division de la perspective ait été effectuée sur les coordonnées du clip. Toutes les positions de vertex dans le NDC entre `-1.0` et `1.0` ne seront pas rejetées ou coupées et finiront par être visibles.  
- `Vertex Buffer Object` (VBO) : un objet tampon qui alloue de la mémoire sur le GPU et y stocke toutes les données de vertex pour que la carte graphique puisse les utiliser.  
- `Vertex Array Object` (VAO) : stocke le tampon et les informations sur l'état des attributs des vertex.  
- `Element Buffer Object` (EBO): un objet tampon qui stocke les index sur le GPU pour le dessin indexé.  
- `Uniform` : un type spécial de variable GLSL qui est globale (chaque shader dans un programme de shader peut accéder à cette variable uniforme) et qui ne doit être définie qu'une seule fois.  
- `Texture` : un type spécial d'image utilisé dans les shaders et généralement enroulé autour des objets, donnant l'illusion qu'un objet est extrêmement détaillé.  
- `Texture Wrapping` : définit le mode qui spécifie comment OpenGL doit échantillonner les textures lorsque les coordonnées de la texture sont en dehors de l'intervalle : (`0`, `1`).  
- `Texture Filtering` : définit le mode qui spécifie comment OpenGL doit échantillonner la texture lorsqu'il y a plusieurs texels (pixels de texture) à choisir. Cela se produit généralement lorsqu'une texture est agrandie.  
- `Mipmaps` : stocke des versions plus petites d'une texture où la version de taille appropriée est choisie en fonction de la distance à l'observateur.  
- `stb_image` : bibliothèque de chargement d'images.  
- `Texture Units` : permet d'avoir plusieurs textures sur un seul programme shader en liant plusieurs textures, chacune à une unité de texture différente.  
- `Vecteur` : entité mathématique qui définit des directions et/ou des positions dans n'importe quelle dimension.  
- `Matrice` : un tableau rectangulaire d'expressions mathématiques avec des propriétés de transformation utiles.  
- `GLM` : une bibliothèque mathématique adaptée à OpenGL.  
- `Local Space` : l'espace dans lequel un objet est initialisé. Toutes les coordonnées relatives à l'origine d'un objet.  
- `World Space` : toutes les coordonnées relatives à une origine globale.  
- `View space` : toutes les coordonnées vues depuis la perspective d'une caméra.  
- `Clip Space` : toutes les coordonnées vues du point de vue de la caméra mais avec la projection appliquée. C'est l'espace dans lequel les coordonnées des vertex devraient se retrouver, en tant que sortie du vertex shader. OpenGL fait le reste (clipping et division de la perspective).  
- `Screen Space` : toutes les coordonnées vues de l'écran. Les coordonnées vont de `0` à la largeur/hauteur de l'écran.  
- `LookAt` : un type spécial de matrice de vue qui crée un système de coordonnées où toutes les coordonnées sont pivotées et translatées de manière à ce que l'utilisateur regarde une cible donnée à partir d'une position donnée.  
- `Angles d'Euler` : définis comme `yaw`, `pitch` et `roll` qui nous permettent de former n'importe quel vecteur de direction 3D à partir de ces 3 valeurs.