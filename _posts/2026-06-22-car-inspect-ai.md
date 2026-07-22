---
layout: project
title: "Car Inspect AI : apprendre à YOLO à lire une voiture comme le ferait un inspecteur"
date: 2026-06-22 14:00:00 +0000
lang: fr
tags: [Computer Vision, Machine Learning, Python, MLOps]
type: project
github_repo: "https://github.com/sanayasfp/car-inspect-ai"
---

Quiconque a déjà loué une voiture ou déposé un dossier d'assurance connaît le rituel : quelqu'un fait le tour du véhicule, prend quelques photos, et une personne décide plus tard si telle marque sur le pare-chocs était déjà là. C'est lent, c'est subjectif, et les litiges entre locataires, propriétaires et assureurs sur les "dommages préexistants" sont fréquents, précisément parce que tout repose sur la parole de quelqu'un face à une poignée de photos. **Car Inspect AI** est ma tentative de m'attaquer à ce problème : un système de vision par ordinateur qui détecte et identifie automatiquement les différentes parties d'un véhicule à partir d'une photo, comme première brique vers une inspection plus objective et automatisée.

Le code est ici : [github.com/sanayasfp/car-inspect-ai](https://github.com/sanayasfp/car-inspect-ai).

## Partir de la vraie contrainte : les données

Avant de toucher au moindre modèle, la vraie contrainte d'un projet comme celui-ci, ce sont les données. J'ai entraîné le modèle sur le jeu de données public **Car Parts Segmentation** (Kitsuchart Pasupa et al.), 500 images annotées de berlines, pickups et SUV au format COCO, couvrant 18 parties distinctes du véhicule — pare-chocs, portes, feux, rétroviseurs, capot, coffre, roues, etc. — photographiées de face, de dos et sous des angles inclinés, avec plaques et visages floutés pour la confidentialité. 500 images, ce n'est pas énorme à l'échelle du deep learning, et cette contrainte a façonné presque toutes les décisions qui ont suivi.

## Pourquoi YOLO11n précisément

J'ai choisi **YOLO11n** — la variante "nano" — de manière délibérée, pas simplement parce que "YOLO, c'est ce qu'on utilise pour la détection d'objets." Deux raisons :

- **C'est assez léger pour tourner sur du matériel modeste.** Un outil pensé pour des garages, des petites agences de location ou des inspecteurs indépendants ne sert à rien s'il exige un GPU puissant pour faire de l'inférence. YOLO11n sacrifie un peu de précision brute contre une empreinte qui tourne confortablement sur CPU ou sur un GPU d'entrée de gamme.
- **Avec seulement 500 images, la capacité du modèle est un handicap, pas un atout.** Un modèle plus large a plus de marge pour sur-apprendre un petit jeu de données. Une architecture légère, combinée à une augmentation de données agressive, était le choix le plus honnête compte tenu de ce que j'avais réellement pour entraîner.

Pour tirer le maximum de ce petit jeu de données, le prétraitement a inclus un redimensionnement aux dimensions d'entrée attendues par YOLO, ainsi qu'une augmentation par rotation (pour simuler différents angles de prise de vue), un flip horizontal, et l'ajout de bruit gaussien (pour rendre le modèle moins sensible aux variations d'éclairage — un vrai problème quand les photos viennent d'un téléphone au hasard sur un parking, pas d'un studio).

L'entraînement s'est fait sur 50 époques avec un split 80/20 entraînement/validation, une taille de batch de 16, et un arrêt anticipé si la performance sur le jeu de validation stagnait pendant 10 époques consécutives. Résultat : **87% de mAP** sur le jeu de validation, avec les meilleures performances sur les parties géométriquement bien définies comme les roues et les portes — exactement là où on s'attend à ce qu'un détecteur soit le plus fiable, et exactement le genre de résultat qui indique où concentrer les efforts ensuite (les petites parties ambiguës comme les rétroviseurs restent les cas les plus difficiles).

## Coder mon propre petit ORM plutôt que d'aller chercher SQLAlchemy

C'est la partie du projet que la plupart des gens survoleraient, mais c'est celle où j'ai le plus appris. L'application doit suivre les runs d'entraînement — quel modèle, combien d'époques, si l'entraînement s'est terminé, où se trouve le checkpoint, et s'il reprend un run précédent. Plutôt que d'importer SQLAlchemy pour ce qui reste fondamentalement une poignée de tables, j'ai écrit moi-même une petite couche de modèles basée sur les dataclasses :

```python
@dataclasses.dataclass
class TrainLogsModel(BaseModel):
    _table_name = "train_logs"

    name: str
    epochs: int
    model: str
    path: str
    completed: bool = Field(type=bool, default=False).set()
    id: Optional[int] = Field(type=int, primary_key=True, autoincrement=True).set()
    created_at: Optional[float] = Field(type=int, default=lambda: dt.now().timestamp()).set()
    resumed_from: Optional[int] = Field(type=int, foreign_key="id", foreign_table=_table_name).set()
```

`BaseModel` lit les annotations de type et les métadonnées de la dataclass et les transforme en définitions de colonnes SQL (`INTEGER`, `TEXT`, `REAL`, avec clés primaires, autoincrément et clés étrangères gérées explicitement), et déduit les noms de table à partir des noms de classe en snake_case, camelCase ou PascalCase selon le besoin. C'est une fraction de ce que fait un vrai ORM — pas de query builder, pas de système de migrations — mais écrire cette fraction à la main m'a obligé à vraiment comprendre ce qu'un ORM automatise, plutôt que d'en importer un et de faire confiance à la magie. C'est un arbitrage que je referais : pour un projet de cette taille, écrire 100 lignes pour comprendre le mécanisme valait mieux que 10 lignes qui le cachent.

Ce modèle alimente une fonctionnalité réellement utile : la page d'entraînement permet de choisir soit un YOLO11n de base, soit n'importe quel checkpoint précédent, de lancer l'entraînement et de le journaliser — y compris une clé étrangère `resumed_from` pointant vers le run dont il repart, ce qui me donne une vraie généalogie d'expériences plutôt qu'un dossier plein de `best_v2_final_FINAL.pt`.

## L'interface : Streamlit, par choix

Le tout est enveloppé dans une petite application Streamlit multi-pages — une page d'accueil, une page "enregistrer une voiture" (upload des photos avant/arrière/gauche/droite plus couleur et numéro de plaque), la page d'entraînement décrite plus haut, et une page de brouillons pour les expérimentations en cours. Streamlit était le bon choix ici précisément parce que ce n'est pas l'objet du projet : je voulais passer mon temps sur le modèle de détection et sur le suivi des entraînements, pas sur un frontend fait main, et Streamlit s'efface pour ça.

## Où en est vraiment le projet aujourd'hui

Je préfère être précis plutôt que d'arrondir : le modèle de détection des parties et le pipeline d'entraînement/versioning fonctionnent et sont mesurés (ce chiffre de 87% de mAP est réel, issu d'un run effectif, pas d'une estimation). Le pipeline d'inspection complet — fusionner les quatre angles d'un véhicule en un seul rapport fiable, et passer de "voici les parties détectées" à un véritable verdict de dommage ou de fraude — reste un chantier actif, pas un produit terminé. Le README liste la notation automatique de la gravité des dommages et l'intégration à un historique véhicule comme axes futurs, et c'est exact : c'est la feuille de route, pas quelque chose que je prétends déjà fonctionnel de bout en bout.

Si je devais résumer ce qui est réellement acquis aujourd'hui : un détecteur de parties de véhicule léger, honnêtement benchmarké, entraîné de manière reproductible, avec sa propre couche minimale de suivi d'expériences construite à partir de zéro. C'est une ambition plus modeste que "système de détection de fraude automatisé", mais c'est la vraie — et c'est une meilleure fondation pour construire la suite.

Le dépôt est public si vous voulez voir le code d'entraînement, le mini-ORM, ou me convaincre que j'aurais dû simplement utiliser SQLAlchemy : [github.com/sanayasfp/car-inspect-ai](https://github.com/sanayasfp/car-inspect-ai).
