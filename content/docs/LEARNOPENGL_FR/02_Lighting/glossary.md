---
tags: [cg, opengl, computer graphics, cpp]
dg-publish: true
---
# Glossaire
- Vecteur de couleur : un vecteur représentant la plupart des couleurs du monde réel par une combinaison des composantes rouge, verte et bleue (abrégé en `RGB`). La couleur d'un objet est la composante de couleur réfléchie que l'objet n'a pas absorbée.
- Modèle d'éclairage Phong : modèle d'approximation de l'éclairage du monde réel par le calcul d'une composante ambiante, diffuse et spéculaire.
- Éclairage ambiant : approximation de l'éclairage global en donnant à chaque objet une petite luminosité afin que les objets ne soient pas complètement sombres s'ils ne sont pas directement éclairés.
- Éclairage diffus : l'éclairage devient plus fort au fur et à mesure qu'un sommet/fragment est aligné avec une source de lumière. Utilise les vecteurs normaux pour calculer les angles.
- Vecteur normal : vecteur unitaire perpendiculaire à une surface.
- Matrice normale : une matrice 3x3 qui est la matrice du modèle (ou de la vue du modèle) sans translation. Elle est également modifiée de telle manière (inverse-transposition) qu'elle maintient les vecteurs normaux orientés dans la bonne direction lors de l'application d'une mise à l'échelle non uniforme. Sinon, les vecteurs normaux sont déformés lors de l'utilisation d'une mise à l'échelle non uniforme.
- Éclairage spéculaire : définit un éclairage spéculaire plus l'observateur regarde la réflexion d'une source de lumière sur une surface. Il est basé sur la direction de l'observateur, la direction de la lumière et une valeur de brillance qui définit la quantité de diffusion de la lumière.
- Phong shading : le modèle d'éclairage Phong appliqué dans le fragment shader.
- Gouraud shading : le modèle d'éclairage Phong appliqué dans le vertex shader. Produit des artefacts perceptibles lors de l'utilisation d'un petit nombre de sommets. Gagne en efficacité pour une perte de qualité visuelle.
- GLSL struct : une structure de type C qui agit comme un conteneur pour les variables du shader. Principalement utilisée pour organiser les entrées, les sorties et les uniformes.
- Material : la couleur ambiante, diffuse et spéculaire qu'un objet reflète. Elles définissent les couleurs d'un objet.
- Light (properties) : l'intensité ambiante, diffuse et spéculaire d'une lumière.Ces propriétés peuvent prendre n'importe quelle valeur de couleur et définissent la couleur/l'intensité d'une source de lumière pour chaque composant Phong spécifique.
- Diffuse map : une image de texture qui définit la couleur diffuse par fragment.
- Specular map : une image de texture qui définit l'intensité/couleur spéculaire par fragment.Permet d'obtenir des reflets spéculaires uniquement sur certaines zones d'un objet.
- Directional light : une source de lumière avec seulement une direction. Elle est modélisée comme étant à une distance infinie, ce qui a pour effet que tous ses rayons lumineux semblent parallèles et que son vecteur de direction reste le même sur toute la scène.
- Lumière ponctuelle (point light): une source de lumière située à un endroit précis de la scène et dont la lumière s'estompe avec la distance.
- Atténuation : processus de réduction de l'intensité de la lumière sur la distance, utilisé pour les lumières ponctuelles et les projecteurs.
- Spotlight : source de lumière définie par un cône dans une direction spécifique.
- Flashlight : une spotlight placé du point de vue de l'observateur.
- GLSL uniform array : un tableau de valeurs uniformes. Il fonctionne comme un tableau C, sauf qu'il ne peut pas être alloué dynamiquement.