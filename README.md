# Atelier Power BI – Rapport Géopolitique et Météorologique (Version Complète)

Ce document constitue la version détaillée et enrichie du projet présenté sur LinkedIn. Il inclut toutes les étapes de préparation, transformation, modélisation, visualisation et création des mesures dans Power BI, ainsi que les bonnes pratiques et la compatibilité avec les nouvelles contraintes de l’API RestCountries.

---

# A) Préparation des données dans Power Query

## 1) Importer l’API RestCountries

### ⚠️ IMPORTANT — Changement dans l’API (2024–2025)

L’API *RestCountries* a évolué :

* Il faut désormais préciser **explicitement les champs à récupérer**.
* **Maximum 10 champs** par requête.

 Donc, pour récupérer plus de 10 champs, vous devrez **faire plusieurs appels** puis **fusionner les tables**.

### Exemple de requête (10 champs maximum)


### Champs recommandés (répartis sur plusieurs requêtes)

**Requête 1 : Identité + géographie**

* name
* cca2
* cca3
* ccn3
* capital
* region
* continents
* latlng
* area
* population

**Requête 2 : Langues + monnaies + autres métadonnées**

* languages
* currencies
* unMember
* maps
* flag

Après chaque import :

* Convertir en **Table**
* Supprimer les étapes automatiques
* Développer uniquement les champs nécessaires

---

## 2) Extraction des valeurs non exploitables

### Name

* Extraire : name → official

### Capital

* Extraire la valeur
* Remplacer les erreurs par null

### Latlng

* Extraire la liste → convertir en texte → remplacer virgule → fractionner → renommer en *Latitude* et *Longitude*

### Maps

* Extraire l’URL Google Maps uniquement

### Continents / Langages / Monnaies

Ces colonnes contiennent plusieurs valeurs → elles seront traitées dans des tables séparées.

---

## 3) Renommage et typage

* Renommer toutes les colonnes avec des noms simples et cohérents
* Vérifier type : texte, décimal, entier, etc.

---

## 4) Création des tables de référence (Langues, Monnaies, Continents)

Méthode :

1. Dupliquer la table principale **par référence**.
2. Conserver la clé importante (cca2) + colonne à dépivoter.
3. Développer la liste.
4. Dépivoter les colonnes.
5. Renommer (Langue, Devise, Continent…).

Ces tables permettront :

* un modèle propre
* des filtres avancés
* des relations 1→*

---

## 5) Gestion de la météo (API Open-Meteo)

### Méthodologie globale

1. Vérifier les coordonnées (latitude/longitude)
2. Tester une requête manuelle pour un pays
3. Transformer la réponse en table
4. Créer une fonction personnalisée
5. Appliquer la fonction à chaque ligne du dataset principal

### Exemple d’appel API

```
https://api.open-meteo.com/v1/forecast?latitude=12.11&longitude=-61.66&current_weather=true
```

### Champs à extraire

* temperature
* windspeed
* winddirection
* weathercode

### Weather Code

Une table externe est fournie dans `weather_table.txt` → importer une requête vide et coller le code.

---

## 6) URL des drapeaux — API Flagpedia

Format d’URL :

```
https://flagcdn.com/160x120/{cca2}.png
```

Créer une colonne :

```
"https://flagcdn.com/160x120/" & [CCA2] & ".png"
```

---

## 7) Finalisation de la table principale

* Dupliquer par référence
* Supprimer les colonnes techniques (languages, continents, url météo…)
* Désactiver le chargement de la table brute

---

# B) Modèle de données

Relations recommandées :

* *Main Country* 1 → *Langues*
* *Main Country* 1 → *Continents*
* *Main Country* 1 → *Monnaies*
* *Weather Codes* 1 → *Main Country*
* Toutes les relations en filtrage simple

---

# C) Création de la Carte Principale

## 1) Format de l’onglet

* Taille : 1920 × 1280
* Couleur fond : #121526

## 2) Carte choroplèthe

* Champ emplacement : **CCA2**
* Couleurs dynamiques selon la population (premier niveau)
* Zoom activé

### Échelle de couleurs

Dégradé bleu → rouge basé sur selected metric.

---

## 3) Zone de filtres

Filtres recommandés :

* Continent
* Langue
* Température
* Vitesse du vent
* Monnaie

Couleurs :

* Bloc : #161C32
* Fond filtres : #262D45

---

# D) Sélecteur dynamique de mesures (Metrics Switch)

L’objectif : permettre à l’utilisateur de changer la mesure colorant la carte.

## 1) Créer la table `MetricsTable`

```DAX
MetricsTable =
DATATABLE(
    "MetricName", STRING,
    "MetricValue", STRING,
    {
        {"Superficie (km²)", "Area"},
        {"Température", "Temperature"},
        {"Vitesse du vent", "WindSpeed"},
        {"Population", "Population"}
    }
)
```

## 2) Ajouter un segment basé sur MetricName

* Sélection unique

## 3) Mesure DAX : SelectedMetricValue

```DAX
SelectedMetricValue =
VAR SelectedMetric = SELECTEDVALUE(MetricsTable[MetricValue])
RETURN
    SWITCH(
        SelectedMetric,
        "Area", SUM('table principale'[superficie (km2)]),
        "Temperature", SUM('table principale'[Temperature]),
        "WindSpeed", SUM('table principale'[vitesse du vent]),
        "Population", SUM('table principale'[Population]),
        BLANK()
    )
```

## 4) Appliquer cette mesure à la carte
-

# E) Création d’une infobulle personnalisée

## 1) Configuration

* Taille de la page : 500 × 700
* Activer “Utiliser comme info-bulle”

## 2) Informations à afficher

### Bloc socio-démographique

* Capital
* Continent
* Population
* Superficie
* Langues (concaténées)
* Monnaies (concaténées)

Mesure exemple :

```DAX
ListeLangues = CONCATENATEX(Langues, Langues[Langue], ", ")
```

### Bloc météo

* Température
* Vent (km/h)
* Orientation
* Weather code (avec table externe)

## 3) Images dynamiques

* Drapeau via URL Flagpedia
* Icône météo (depuis table dédiée)
---

# F) Conclusion

Vous disposez maintenant d’un rapport complet combinant :

* données géopolitiques
* données météo en temps réel
* filtres avancés
* mesures dynamiques
* carte interactive
* infobulle intelligente

Ce projet met en pratique :

* Power Query avancé
* Appels API dynamiques
* Gestion de tables relationnelles
* DAX (mesures, concaténation, switch, selectors)
* Visualisations avancées dans Power BI

