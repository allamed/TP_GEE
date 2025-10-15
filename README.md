# TP – GEE (Maroc) : Étapes à copier‑coller (avec explications)

> **Consigne générale** : exécutez **une étape à la fois**. À chaque étape :
>
> 1. **Copiez‑collez**  le bloc de code indiqué dans l’éditeur GEE ; 2) cliquez **Run** ; 3) vérifiez **ce que vous devez voir** sur la **carte** et/ou dans la **Console**.
>     Utilisez le panneau **Layers** pour (dé)cocher les couches et l’outil **Inspector** pour lire les valeurs.

---

## Étape 1 — Définir l’AOI (Maroc) et l’afficher

**Idée** : on charge les frontières, on sélectionne *Morocco*, on fusionne (union) puis on centre la carte et on dessine le contour.

**Code à copier‑coller :**

```js
// Frontières (USDOS)
var countries = ee.FeatureCollection('USDOS/LSIB_SIMPLE/2017');

// 1) Sélectionner Maroc
var mar = countries.filter(ee.Filter.inList('country_na', ['Morocco', 'Western Sahara']));

// 2) Dissoudre en une seule géométrie (union)
var marGeom = mar.geometry(); // union implicite de toutes les features

// 3) Affichage
Map.centerObject(marGeom, 5);
Map.addLayer(ee.Image().paint(marGeom, 1, 2), {palette: ['#FF6B00']}, 'Maroc');
```

**Vous devez voir** : la carte se centre sur le Maroc (zoom ≈ 5) et un **contour orange** apparaît.

---

## Étape 2 — Altitude (SRTM) + relief

**Idée** : on charge le **MNT SRTM** (≈30 m), on le **découpe** à l’AOI et on l’affiche en couleur ; puis on calcule le **relief ombré** (hillshade) et on l’ajoute.

**Code à copier‑coller :**

```js
// Modèle numérique de terrain (≈30 m)
var srtm = ee.Image('USGS/SRTMGL1_003').clip(marGeom);

var visElev = {min:0, max:3500, palette:['#003366','#66A3FF','#E5FFCC','#996633','#FFFFFF']};
Map.addLayer(srtm, visElev, 'Altitude (SRTM)');

var terrain = ee.Algorithms.Terrain(srtm);
Map.addLayer(terrain.select('hillshade'), {min:0, max:255}, 'Hillshade (relief)');
```

**Vous devez voir** : une **carte d’altitude** colorée couvrant le Maroc, et une couche **Hillshade** grisée (activez/désactivez pour comparer).

---

## Étape 3 — Définir la période d’étude pour Sentinel‑2

**Idée** : on indique la **fenêtre temporelle** des images (été 2024 ici).

**Code à copier‑coller :**

```js
// 1) Fenêtre temporelle
var start = '2024-06-01';
var end = '2024-09-30';
```

**Vous devez voir** : rien de nouveau sur la carte (c’est un réglage).

---

## Étape 4 — Créer la collection Sentinel‑2 filtrée et compter les images

**Idée** : on charge **Sentinel‑2 L2A harmonisé**, on **filtre** par zone (Maroc), dates et **nuages < 30%**, puis on affiche le **nombre d’images** trouvées.

**Code à copier‑coller :**

```js
// 2) Collection Sentinel‑2 SR (harmonisée)
var s2 = ee.ImageCollection('COPERNICUS/S2_SR_HARMONIZED')
.filterBounds(mar.geometry())
.filterDate(start, end)
.filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 30));

print('S2 (compte d’images) = ', s2.size());
```

**Vous devez voir** : dans la **Console**, une ligne `S2 (compte d’images) = <nombre>`.

---

## Étape 5 — Composite **médiane** et affichage **True Color**

**Idée** : on crée une **médiane temporelle** (réduit les nuages résiduels) et on affiche en **couleurs naturelles** (B4‑B3‑B2).

**Code à copier‑coller :**

```js
// 3) Composite médiane (réduction dans le temps)
var s2Median = s2.median();


// 4) Visualisation RGB (B4=R, B3=V, B2=B)
var visRGB = {bands:['B4','B3','B2'], min:0, max:3000, gamma:1.2};
Map.addLayer(s2Median, visRGB, 'Sentinel‑2 — True Color (médiane)');
```

**Vous devez voir** : une couche **Sentinel‑2 — True Color (médiane)** au‑dessus du Maroc.

---

## Étape 6 — Masquer les nuages (QA60) et recomposer

**Idée** : on définit une fonction qui **masque** les pixels nuageux (bande **QA60**) ; on l’applique à toute la collection puis on refait une **médiane** propre.

**Code à copier‑coller :**

```js
// Fonction de masque nuages (QA60)
function maskS2clouds(img) {
var qa = img.select('QA60');
var cloudBitMask = 1 << 10; // clouds
var cirrusBitMask = 1 << 11; // cirrus
var mask = qa.bitwiseAnd(cloudBitMask).eq(0)
.and(qa.bitwiseAnd(cirrusBitMask).eq(0));
return img.updateMask(mask);
}


// Appliquer le masque et recomposer
var s2Clean = s2.map(maskS2clouds).median();
Map.addLayer(s2Clean, visRGB, 'S2 — RGB (nuages masqués)');
```

**Vous devez voir** : une couche **S2 — RGB (nuages masqués)**, généralement plus **propre** que la médiane brute.

---

## Étape 7 — NDVI (végétation)

**Idée** : on calcule l’**indice NDVI** (B8, B4) et on l’affiche avec une palette verte.

**Code à copier‑coller :**

```js
// NDVI = (B8 - B4) / (B8 + B4)
var ndvi = s2Clean.normalizedDifference(['B8','B4']).rename('NDVI');
var visNDVI = {min:0, max:0.8, palette:['#FFFFFF','#CEFFCE','#66CC66','#006400']};
Map.addLayer(ndvi, visNDVI, 'NDVI (végétation)');
```

**Vous devez voir** : la couche **NDVI (végétation)** ; la végétation dense ressort en **vert foncé**.

---

## Étape 8 — NDWI (eau) et **masque eau**

**Idée** : on calcule l’**indice NDWI** (B3, B8) ; puis on crée un **masque binaire** de l’eau en appliquant un **seuil** (0.2).

**Code à copier‑coller :**

```js
// NDWI = (Green - NIR) / (Green + NIR) = (B3 - B8) / (B3 + B8)
var ndwi = s2Clean.normalizedDifference(['B3','B8']).rename('NDWI');
Map.addLayer(ndwi, {min:-0.5, max:0.7, palette:['#8a8a8a','#00FFFF']}, 'NDWI (indice)');


// Masque eau simple (seuil à ajuster)
var waterMask = ndwi.gte(0.2);
Map.addLayer(waterMask.updateMask(waterMask), {palette:['#00BFFF']}, 'Eau (NDWI≥0.2)');
```

**Vous devez voir** : l’**indice NDWI** (tons gris→cyan) et la couche **Eau (NDWI≥0.2)** en **bleu** sur lacs/barrages/côtes.

---

## Étape 9 — CHIRPS (précipitations quotidiennes 2024) filtré au Maroc

**Idée** : on charge la collection **CHIRPS DAILY** pour 2024 et on la restreint à l’AOI.

**Code à copier‑coller :**

```js
// CHIRPS quotidien (précipitations, mm/jour)
var chirps = ee.ImageCollection('UCSB-CHG/CHIRPS/DAILY')
.filterDate('2024-01-01', '2024-12-31')
.filterBounds(mar.geometry());
```

**Vous devez voir** : rien de nouveau sur la carte (collection préparée).

---

## Étape 10 — Agrégation **mensuelle** des pluies (somme par mois)

**Idée** : on crée une image par **mois** en sommant les jours, et on stocke l’attribut `month`.

**Code à copier‑coller :**

```js
// Regrouper par mois : somme mensuelle
var months = ee.List.sequence(1, 12);
var byMonth = ee.ImageCollection.fromImages(
months.map(function(m) {
var start = ee.Date.fromYMD(2024, m, 1);
var end = start.advance(1, 'month');
var sum = chirps.filterDate(start, end).sum().set({
'month': m,
'system:time_start': start.millis()
});
return sum;
})
);
```

**Vous devez voir** : rien de nouveau sur la carte (construction de la collection **byMonth**).

---

## Étape 11 — Graphique mensuel (moyenne spatiale) + carte de janvier

**Idée** : on calcule la **moyenne sur le Maroc** pour chaque mois (série), on trace un **graphique** dans la Console, et on affiche la **carte** de janvier.

**Code à copier‑coller :**

```js
// Statistique spatiale (moyenne sur le Maroc) pour chaque mois
var monthlyChart = ui.Chart.image.series({
imageCollection: byMonth,
region: mar.geometry(),
reducer: ee.Reducer.mean(),
scale: 5000
}).setOptions({
title: 'Précipitations mensuelles moyennes — Maroc (CHIRPS, 2024)',
hAxis: {title: 'Mois'},
vAxis: {title: 'mm/mois'}
});
print(monthlyChart);


// Afficher un mois particulier sur la carte (ex. janvier)
var jan = byMonth.filter(ee.Filter.eq('month', 1)).first().clip(marGeom);;
Map.addLayer(jan, {min:0, max:200, palette:['#FFFFCC','#41B6C4','#225EA8']}, 'Pluie — Janvier 2024 (mm)');
```

**Vous devez voir** :

* Dans la **Console** : le **graphique** « Précipitations mensuelles moyennes — Maroc (CHIRPS, 2024) ».
* Sur la **carte** : la couche **Pluie — Janvier 2024 (mm)** en dégradé jaune→bleu.

---

## Étape 12 — Statistique finale : altitude moyenne du Maroc

**Idée** : on calcule la **moyenne** d’altitude (SRTM) sur tout le Maroc.

**Code à copier‑coller :**

```js
var elevMean = srtm.reduceRegion({
reducer: ee.Reducer.mean(),
geometry: mar.geometry(),
scale: 1000,
maxPixels: 1e13
});
print('Altitude moyenne du Maroc (m)', elevMean.get('elevation'));
```

**Vous devez voir** : dans la **Console**, une ligne `Altitude moyenne du Maroc (m)  <valeur>`.

---

## Conseils de lecture & contrôle

* **Layers** : (dé)cochez *Altitude*, *Hillshade*, *Sentinel‑2 médiane*, *S2 — RGB (nuages masqués)*, *NDVI*, *NDWI*, *Eau*, *Pluie — Janvier 2024* pour comparer.
* **Inspector** : cliquez sur la carte pour lire NDVI/NDWI/Altitude/Pluie.
* **Console** : vérifiez le **compte d’images** S2 et le **graphique** CHIRPS.
* **Performance** : si c’est lent, zoomez sur une zone plus petite et réduisez le nombre de couches visibles.

---

## À retenir (1 minute)

* **AOI** : travailler sur une zone claire (clip) rend tout plus lisible et rapide.
* **Sentinel‑2** : filtrage par date/nuages + **médiane** → fond propre.
* **Indices** : **NDVI** (végétation) ; **NDWI** (eau) avec **seuil** ajustable.
* **Séries temporelles** : CHIRPS quotidien → **mensuel** + graphique.
* **Stat régionales** : `reduceRegion` pour un indicateur synthétique (altitude moyenne).
