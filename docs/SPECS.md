# Lightroom People — Spécifications fonctionnelles et techniques

> Version 0.1 (brouillon) — 2026-10-31
> Cible : plugin **Lightroom Classic** (SDK Lua), catalogue d'environ 80 000 photos.

Les points marqués **⚠ à valider** sont des hypothèses sur le SDK ou les API que
l'on vérifie par un prototype (voir §10, « Spikes ») avant d'écrire le code
définitif.

---

## 0. Résumé

Le plugin exploite des mots-clés « personne » et un **registre des personnes**
(dates de naissance et de décès) pour :

1. créer une **collection** (fixe) avec toutes les photos d'une personne ;
2. créer des **collections dynamiques** (smart collections) par personne et par
   tranche d'âge ;
3. ouvrir une fenêtre **« Parcourir par âge »** : on choisit une personne et on
   voit ses miniatures groupées par âge ;
4. gérer les **personnes décédées** et les **photos numérisées**, dont la date
   réelle diffère de la date du fichier ;
5. **(fonction principale)** générer un **album** pour une personne selon un
   rythme (une photo par heure le premier jour, une par jour la première
   semaine, une par mois ensuite…). Dans chaque créneau, le choix se fait par
   des heuristiques locales puis par **Gemini** (variété, netteté, importance du
   moment, personne au centre de la photo).

### Principes

- **Non destructif** : le plugin ne modifie ni les pixels, ni la date de prise
  de vue, ni les métadonnées standard. Il écrit seulement des collections et
  ses propres champs de métadonnées, qui restent dans le catalogue et ne
  partent pas à l'export.
- **Pensé pour 80 000 photos** : lectures groupées (`batchGetRawMetadata`),
  écritures par lots, tâches de fond annulables avec barre de progression.
- **L'IA ne fait que proposer** : la sélection de l'album passe toujours par
  une revue humaine, et chaque choix est accompagné de sa justification.
- **Données privées réduites au minimum** : vers Gemini, on n'envoie que des
  miniatures sans EXIF ni GPS et aucun nom (voir §7.6).

### Hors périmètre (v1)

- Lightroom « cloud » (non Classic) : son API ne permet pas ce plugin.
- Mise en page d'un livre photo. Le plugin produit une sélection ordonnée et un
  export de fichiers ; la mise en page se fait dans le module Livre ou chez un
  imprimeur.
- Reconnaissance faciale « maison » : c'est une option de la v2 (§6.4).

---

## 1. Contexte et contraintes techniques

| Sujet | Contrainte / choix |
|---|---|
| Plateforme | Lightroom Classic, SDK Lua (`Info.lua`, `LrCatalog`, `LrView`, `LrHttp`, `LrTasks`…). Mac et Windows. |
| Langage | Lua 5.1, sans module natif. JSON en Lua pur (ex. `dkjson` ou `rxi/json.lua`, intégré au plugin). |
| Accès au catalogue | Lecture : `catalog:findPhotos`, `keyword:getPhotos()`, `catalog:batchGetRawMetadata`. Écriture : uniquement dans `catalog:withWriteAccessDo` (collections, champs du plugin), par lots de 500 à 1000 photos. |
| Images vers Gemini | `photo:requestJpegThumbnail(w, h, cb)` : JPEG tiré des aperçus, sans export disque. Il faut des aperçus standard. Sinon, repli sur `LrExportSession` vers un dossier temporaire. |
| HTTP | `LrHttp.post` (HTTPS), dans des `LrTasks` parallèles (concurrence réglable, 4 par défaut). |
| Secrets | Clé API Gemini stockée avec `LrPasswords` (trousseau du système), jamais en clair. |
| Données du plugin | Dossier `<nom du catalogue> People Data/` à côté du `.lrcat` : registre, cache d'analyses, manifestes d'albums. |
| Format des dates | Le SDK compte en secondes depuis le 2001-01-01 (`LrDate`). Les heures de prise de vue sont en heure locale « naïve », sans fuseau, comme la naissance. |

---

## 2. Modèle de données

### 2.1 Personnes : mot-clé + registre

**Mot-clé (ce que voit Lightroom)** — convention recommandée :

```
Personnes
├── Famille Thomas
│   ├── Irène Thomas
│   └── Paul Thomas
└── Amis
    └── …
```

- Toute personne est un mot-clé placé sous une **racine configurable**
  (`Personnes` par défaut).
- Si c'est possible, ce sont des mots-clés de **type « Personne »**, compatibles
  avec la vue *Personnes* de Lightroom. **⚠ à valider** : le SDK ne semble pas
  donner accès à ce drapeau ; on se fie donc à la racine et au registre.
- **On ne met pas la date de naissance dans le nom du mot-clé.** Les noms de
  mots-clés partent dans les fichiers exportés (fuite de donnée personnelle),
  et une correction de date obligerait à renommer le mot-clé.
- En cas d'homonymes (une grand-mère et une petite-fille toutes deux
  « Irène Thomas »), on distingue les mots-clés, par exemple
  `Irène Thomas (1932)`, et le registre leur donne des identifiants différents.

**Registre (source de vérité des données biographiques)** —
`<catalogue> People Data/people.json`. On le modifie dans le plugin (§3.5) ou à
la main :

```json
{
  "version": 1,
  "keywordRoot": "Personnes",
  "people": [
    {
      "id": "irene",
      "keywordPath": ["Personnes", "Famille Thomas", "Irène Thomas"],
      "displayName": "Irène Thomas",
      "aliases": ["Irène", "Nene"],
      "sex": "F",
      "birth": { "value": "2014-06-23T09:32", "precision": "minute" },
      "death": null,
      "references": { "pinned": [], "auto": true },
      "notes": ""
    },
    {
      "id": "jeanne",
      "keywordPath": ["Personnes", "Famille Thomas", "Jeanne Thomas"],
      "displayName": "Jeanne Thomas",
      "sex": "F",
      "birth": { "value": "1932", "precision": "year" },
      "death": { "value": "2019-03-14", "precision": "day" }
    }
  ]
}
```

- **Format des dates** : ISO 8601, `AAAA[-MM[-JJ[THH:MM]]]`. Le format proposé
  `2014-06-23T09-32` est accepté en saisie et converti en `2014-06-23T09:32`.
- Le champ `precision` (`year`, `month`, `day`, `minute`) est déduit de la
  saisie. Il conditionne les calculs : sans heure de naissance, le premier jour
  ne peut pas être découpé heure par heure (§5.2).
- Le champ `sex` est optionnel. Il sert à distinguer les personnes sur une même
  photo (§6) et n'est jamais affiché.
- **Option de secours** : le plugin peut recopier la date de naissance dans un
  **synonyme non exporté** du mot-clé (`°2014-06-23T09:32`). Le registre peut
  ainsi être reconstruit depuis le catalogue seul.

### 2.2 Date effective d'une photo

Chaque photo a une **date effective**, qui est un **intervalle**
`[dateMin, dateMax]`. Une photo numérique a un intervalle de largeur nulle ; une
photo « vers 1985 » a un intervalle d'un an ou plus. Le plugin la calcule ainsi
(la première règle qui s'applique l'emporte) :

1. **Date saisie dans le plugin** (`lpDateMin`, `lpDateMax`, `lpDateLabel`) :
   prioritaire, utilisée pour les numérisations (§2.3).
2. **Photo numérisée sans date saisie** : la date de prise de vue n'est pas
   fiable (c'est souvent la date du scan). La photo va dans le groupe
   « Non datée » et n'est pas placée sur la ligne de temps.
3. **Date de prise de vue** (`dateTimeOriginal`, la *Capture Time* de
   Lightroom).
4. Aucune date : groupe « Non datée ».

**Champs de métadonnées du plugin** (déclarés dans un `LrMetadataProvider`,
visibles dans le panneau Métadonnées, `searchable = true`) :

| Champ | Type | Contenu |
|---|---|---|
| `lpDateMin` / `lpDateMax` | texte ISO | intervalle de date réelle saisi |
| `lpDateLabel` | texte | libellé libre : « été 1985 », « vers 1978 » |
| `lpScanned` | énum (`auto`, `oui`, `non`) | forcer ou non le statut « numérisée » |
| `lpAgeTokens` | texte (calculé) | jetons d'âge pour les collections dynamiques (§3.2) |
| `lpStatus` | texte (calculé) | anomalies : `apres_deces`, `avant_naissance`, `non_datee`… |

### 2.3 Photos numérisées

**Détection** (règles configurables, il suffit qu'une seule s'applique) :

- mot-clé `Numérisé` (nom configurable) ;
- `lpScanned = oui` ;
- appareil d'un modèle listé comme scanner (`Make`/`Model` : Epson, Nikon
  Coolscan, Plustek, etc.) ;
- dossier source dans un chemin listé (ex. `…/Scans/`) ;
- `dateTimeOriginal` vide alors que `dateTimeDigitized` est renseigné.

**Datation par lot** : la commande *Dater les photos numérisées…* s'applique à
la sélection. On y saisit « 1985 », « 1985-07 », « été 1985 », « 1983–1986 » ou
« vers 1980 » (± 2 ans), et le plugin remplit `lpDateMin`, `lpDateMax` et
`lpDateLabel`. Les saisons sont des plages de mois configurables.

**Effets** :

- L'âge devient un intervalle, affiché avec « ≈ » (« ≈ 3 ans »). Pour le
  regroupement par âge, on prend le milieu de l'intervalle, avec un badge
  « date approximative ».
- Dans l'album (§5.3), une photo imprécise ne peut occuper que les créneaux
  compatibles avec son intervalle. Elle passe après les photos datées
  précisément, sauf si aucune n'est disponible.
- La date de prise de vue de Lightroom n'est **pas** modifiée. L'utilisateur
  peut toujours la corriger lui-même (*Métadonnées > Modifier la date de
  capture*), et dans ce cas la règle 3 s'applique.

### 2.4 Personnes décédées

- Le registre contient `death`, éventuellement imprécise.
- L'affichage devient « Jeanne Thomas (1932–2019) », avec « † » dans les listes.
- **Photos datées après le décès** et taguées avec la personne : ce sont souvent
  des erreurs de date, des scans mal datés, des photos d'un portrait ou des
  obsèques. Le plugin :
  - les met dans un groupe distinct **« Après le décès »**, sans âge calculé ;
  - leur donne `lpStatus = apres_deces` et les liste dans le rapport
    d'anomalies (§3.5) ;
  - les **exclut par défaut** des albums, avec une option pour les inclure.
- **Photos avant la naissance** (grossesse, échographie) : groupe
  **« Avant la naissance »**, exclu par défaut, avec une option.
- **Albums « mémoire »** : un rythme prédéfini « Vie entière » (§5.2) donne
  quelques photos par décennie, sur toute la vie.
- Gemini ne génère **aucun** texte sur la mort ou le deuil. Les légendes
  éventuelles sont factuelles (âge, date).

### 2.5 Calcul de l'âge

- Âge calendaire : années, mois et jours révolus entre la naissance et la date
  effective. Une personne née un 29 février fête son anniversaire le 28 février
  les années non bissextiles.
- Si l'une des deux dates est un intervalle, on calcule l'intervalle d'âge
  `[min, max]`.
- **Affichage** :

| Âge | Format |
|---|---|
| < 24 h | `J0 · 3 h` (heures depuis la naissance si l'heure est connue) |
| 1 à 6 jours | `J3` |
| 7 à 30 jours | `2 sem.` |
| 1 à 23 mois | `5 mois` |
| 2 à 17 ans | `3 ans` |
| ≥ 18 ans | `35 ans` (ou tranches de 5 ans, réglable) |

### 2.6 Stockage hors catalogue

```
<catalogue> People Data/
├── people.json                 registre
├── settings.json               réglages (pondérations, rythmes, modèle Gemini…)
├── cache/
│   └── gemini/<photoUUID>.json analyses par photo (clé = uuid + version prompt + modèle)
├── albums/
│   └── <albumId>.json          manifeste d'album (créneaux, choix, alternatives, scores, raisons)
└── logs/
```

Le cache sert à ne jamais payer deux fois l'analyse d'une même photo. Il est
invalidé si la photo change (date de développement), si le prompt change de
version ou si le modèle change.

---

## 3. Fonctionnalités

Les commandes se trouvent dans *Bibliothèque > Modules externes > Lightroom
People*. Les plus utilisées sont aussi dans le menu contextuel des photos.

### 3.1 F1 — Collection « toutes les photos de X »

- **Commande** : *Créer la collection d'une personne…*. On choisit une personne
  (liste du registre avec recherche) et éventuellement l'option « inclure les
  sous-mots-clés ».
- **Résultat** : collection fixe `People / Irène Thomas / Toutes (fixe
  2026-10-31)`. C'est un **instantané** : utile pour un export ou un livre, à
  la différence de F2 qui reste à jour.
- **Implémentation** : `keyword:getPhotos()`, plus les photos des synonymes ou
  alias si l'option est cochée, puis `collection:addPhotos()` par lots.
- **Critère d'acceptation** : sur 10 000 photos, terminé en moins de 30 s avec
  une barre de progression.

### 3.2 F2 — Collections dynamiques

Arborescence créée ou mise à jour par *Créer les collections dynamiques de X* :

```
People (ensemble)
└── Irène Thomas (ensemble)
    ├── ★ Toutes les photos           (dynamique)
    ├── Par âge (ensemble)
    │   ├── Naissance (J0)            (dynamique)
    │   ├── 1re semaine               (dynamique)
    │   ├── 0–1 an … 17–18 ans        (dynamiques)
    │   └── Non datées                (dynamique)
    └── Albums (ensemble)             (rempli par F4)
```

**Contrainte** : une collection dynamique ne sait pas calculer un âge. Elle ne
filtre que sur des valeurs fixes. Deux modes sont prévus :

| Mode | Principe | + | − |
|---|---|---|---|
| **A. Plages de dates** | `mot-clé = X` ET `date de capture entre naissance + n ans et naissance + n+1 ans` | 100 % natif, sans maintenance | ignore les dates corrigées des numérisations |
| **B. Jetons d'âge** (recommandé) | Le plugin calcule pour chaque photo un champ `lpAgeTokens`, ex. `ireneA03 ireneM40`, et la collection filtre `lpAgeTokens contient le mot ireneA03` | tient compte des scans, des intervalles, du décès | à recalculer quand le registre ou une date change (automatique depuis le plugin, ou commande *Recalculer les âges*) |

Format des jetons : `<id><unité><valeur>`, en caractères alphanumériques
seulement. Exemples : `ireneJ0`, `ireneM05` (mois révolus), `ireneA03`
(années révolues), `ireneNODATE`, `ireneDEAD`. Une photo avec plusieurs
personnes reçoit les jetons de chacune.

Exemple de `searchDesc` pour le mode A (**⚠ syntaxe à valider** : on crée la
collection à la main, on l'exporte en `.lrsmcol` et on recopie le format) :

```lua
catalog:createSmartCollection("3–4 ans", {
  combine = "intersect",
  { criteria = "keywords",    operation = "words", value = "Irène Thomas" },
  { criteria = "captureTime", operation = "in",
    value = "2017-06-23", value2 = "2018-06-22" },
}, parentSet, true)
```

**⚠ à valider (S1)** : un champ de plugin `searchable` doit être disponible
comme critère de collection dynamique, et l'on doit connaître son nom de
critère dans `searchDesc`. Sinon, on garde le mode A, avec une collection
fixe en complément pour les scans, régénérée par le plugin.

### 3.3 F3 — Fenêtre « Parcourir par âge »

```
┌ Lightroom People — Parcourir par âge ───────────────────────────────┐
│ Personne : [ Irène Thomas (2014) ▼ ]  🔍      Photos : 9 412          │
│ Affichage : (•) par âge  ( ) par année civile   Taille : [M ▼]      │
│ [x] inclure les dates approximatives  [ ] avant naissance  [ ] après │
├─────────────────────────────────────────────────────────────────────┤
│ ▼ J0 — naissance (212)                                  [Ouvrir ▸]  │
│   [img][img][img][img][img][img][img][img] … +204                    │
│ ▼ 1re semaine (431)                                     [Ouvrir ▸]  │
│   [img][img][img][img][img][img][img][img] … +423                    │
│ ▶ 1 mois (120)                                                       │
│ ▶ 2 mois (98)                                                        │
│ …                                                                    │
│ ▶ ≈ 3 ans — dates approximatives (14)                       ≈        │
│ ▶ Non datées (37)                                    [Dater…]        │
├─────────────────────────────────────────────────────────────────────┤
│ [Créer collections dynamiques] [Créer un album…]          [Fermer]   │
└─────────────────────────────────────────────────────────────────────┘
```

- **Tranches par défaut** : J0 ; J1 à J6 ; semaines 2 à 4 ; mois 1 à 23 ; ans 2
  à 17 ; puis par an ou par 5 ans au choix. Il y a aussi les groupes spéciaux
  « Avant la naissance », « Après le décès », « Dates approximatives » et
  « Non datées ».
- **Miniatures** : vue `catalog_photo` de `LrView`. Chaque groupe affiche au
  plus N miniatures (8 par défaut) dans un `scrolled_view`, choisies par note
  puis réparties dans le temps. On ne charge jamais des milliers de vignettes
  dans un dialogue.
- **« Ouvrir ▸ »** : le plugin remplit une collection temporaire
  `People / _Navigation` avec les photos du groupe et l'active
  (`catalog:setActiveSources`). On profite ainsi de la grille native de
  Lightroom, rapide, avec tri, loupe et comparaison.
- **Clic sur une miniature** : sélectionne la photo dans la grille.
- **Performance** : l'index « photo → âge » d'une personne est calculé à
  l'ouverture (lecture groupée des dates), puis mis en cache pour la session.
  Objectif : moins de 3 s pour 10 000 photos.

### 3.4 F4 — Album automatique (fonction principale)

Voir le §5, qui le détaille.

### 3.5 F5 — Gestion du registre et rapport d'anomalies

- **Éditeur de personnes** : liste, ajout, modification (nom, mot-clé associé,
  naissance, décès, sexe, alias, photos de référence épinglées).
  - *Importer depuis les mots-clés* : crée une entrée pour chaque mot-clé sous
    la racine qui n'en a pas encore, dates à compléter.
- **Rapport d'anomalies**, avec un bouton par ligne pour afficher les photos
  concernées :
  - photos d'une personne datées **avant sa naissance** ou **après son décès** ;
  - personnes sans date de naissance ;
  - photos numérisées non datées ;
  - photos où la personne aurait plus de 110 ans ;
  - mots-clés « personne » orphelins (absents du registre).
- **Import Google Takeout (optionnel, v2)** : les fichiers JSON qui
  accompagnent chaque photo d'un export Google Photos contiennent souvent un
  tableau `people` avec les **noms** des visages nommés, sans les cadres. Le
  plugin peut rapprocher ces fichiers du catalogue (nom de fichier d'origine +
  date de prise de vue) et proposer d'ajouter les mots-clés « personne »
  correspondants. C'est utile pour taguer rapidement les 80 000 photos.
  **⚠ à valider (S7)** : présence de ce champ et taux de correspondance sur un
  vrai export.

---

## 4. Réglages généraux

| Réglage | Défaut |
|---|---|
| Racine des mots-clés personnes | `Personnes` |
| Règles de détection des numérisations | mot-clé `Numérisé` + liste de scanners |
| Mode des collections dynamiques | B (jetons d'âge) |
| Tranches d'âge adultes | par an |
| Modèle Gemini | modèle « Flash » le plus récent (champ texte libre) |
| Taille des images envoyées | 768 px sur le grand côté |
| Requêtes Gemini en parallèle | 4 |
| Budget maximal par album | 5 € (estimation affichée avant de lancer) |
| Inclure avant naissance / après décès | non / non |

---

## 5. F4 — Album automatique : spécification détaillée

### 5.1 Parcours utilisateur

1. *Créer un album…*, depuis le menu ou la fenêtre F3.
2. **Paramètres** : personne ; période (par défaut de la naissance à
   aujourd'hui ou au décès) ; **rythme** (prédéfini ou personnalisé) ;
   filtres ; pondérations ; « utiliser Gemini » oui ou non ; budget.
3. **Estimation** : nombre de créneaux, de candidats, d'images envoyées à
   Gemini, coût et durée estimés. L'utilisateur confirme.
4. **Calcul** en tâche de fond, annulable et reprenable grâce au cache.
5. **Revue** (§5.7) : pour chaque créneau, le choix retenu, 2 ou 3
   alternatives et la justification. On peut accepter, échanger, verrouiller
   un choix ou vider un créneau.
6. **Sortie** (§5.8) : collection, manifeste et export optionnel.

### 5.2 Rythmes (créneaux)

Un rythme est une liste de règles **relatives à la naissance**. Chaque règle
découpe une période d'âge en créneaux et fixe un quota par créneau.

```lua
-- Rythme « Naissance » (celui de la demande)
{
  { from = "0h",  to = "24h", step = "1h",  quota = 1, label = "J0 · %Hh" },
  { from = "1d",  to = "7d",  step = "1d",  quota = 1, label = "J%d" },
  { from = "7d",  to = "1M",  step = "1M",  quota = 1, label = "1er mois" },
  { from = "1M",  to = "end", step = "1M",  quota = 1, label = "%M mois / %A ans" },
}
```

Rythmes prédéfinis :

| Nom | Contenu | Nombre de créneaux (0 à 18 ans) |
|---|---|---|
| **Naissance** | 1/heure J0 · 1/jour J1-J6 · 1/mois ensuite | ~ 24 + 6 + 216 = **246** |
| **Naissance+** | idem + 1/semaine de la 2e à la 4e semaine, puis 1/trimestre après 6 ans | ~ 140 |
| **Enfance** | 1/mois jusqu'à 2 ans, puis 1/trimestre | ~ 88 |
| **Anniversaires** | 1 par an, dans les ± 3 jours de l'anniversaire, plus 1 « année » | 2 × années |
| **Vie entière** (mémoire) | 1/an jusqu'à 20 ans, puis 3 par décennie | ~ 40 |

Règles particulières :

- **Heure de naissance inconnue** : les créneaux horaires de J0 sont remplacés
  par un seul créneau « J0 » de quota 3.
- Les **mois** sont des mois calendaires d'âge : le créneau « 5 mois » va du
  23/11 au 22/12 pour une naissance un 23/06.
- Les **créneaux vides** restent vides : on n'invente pas de photo.
  - Option « compenser » : un créneau vide peut céder son quota à un créneau
    voisin de la même règle qui contient un moment fort.
- **Bonus « moments forts »** (option) : jusqu'à +N photos par an pour les
  candidats au score d'importance ≥ 0,9 (anniversaire, premiers pas, Noël…),
  même au-delà du quota de leur créneau.

### 5.3 Pipeline de sélection

```
Toutes les photos du mot-clé
  │ 1. Date effective + âge (§2.2), attribution aux créneaux
  │    (photos imprécises : créneaux compatibles seulement, priorité basse)
  ▼
  │ 2. Filtres stricts : rejetées (drapeau X) exclues ; copies virtuelles
  │    réduites à une par photo maître ; vidéos exclues (option) ;
  │    avant naissance / après décès exclues (option)
  ▼
  │ 3. Rafales : photos du même appareil à moins de 2 s d'écart
  │    → un seul représentant (meilleure note, sinon celle du milieu)
  ▼
  │ 4. Présélection locale par créneau : on garde au plus K candidats
  │    (K = 8 par défaut) selon le score local (§5.4), répartis dans le temps
  ▼
  │ 5. Analyse Gemini des candidats (§7) — mise en cache
  ▼
  │ 6. Sélection par créneau avec pénalité de ressemblance (§5.5)
  ▼
  │ 7. Revue humaine (§5.7) → 8. Sortie (§5.8)
```

Quand un créneau n'a qu'un candidat, celui-ci est retenu sans appel à Gemini,
**sauf** pour vérifier que la personne est bien le sujet (option, activée par
défaut).

### 5.4 Score local (sans IA)

Données disponibles dans Lightroom :

| Signal | Source | Effet |
|---|---|---|
| Note (étoiles) | `rating` | + |
| Choix (drapeau P) | `pickStatus` | + |
| Étiquette de couleur | `colorNameForLabel` | réglable |
| Nombre de personnes taguées | mots-clés sous la racine | **1 seule = la personne est très probablement le sujet** |
| Proximité de l'anniversaire de la personne | date effective | + importance |
| Fêtes (Noël, etc.) | calendrier configurable | + importance |
| Densité de photos dans la journée | nombre de photos le même jour | beaucoup de photos = événement probable |
| Date précise ou approximative | §2.2 | − pour les approximatives |

`scoreLocal = 0,35·note + 0,15·choix + 0,20·seulSurPhoto + 0,15·densitéJour + 0,15·calendrier`

Sans Gemini, la sélection repose sur ce seul score. Le plugin reste donc
utilisable sans clé API.

### 5.5 Score final et variété

Pour chaque candidat `c` d'un créneau :

```
qualité(c) = 0,30·focal + 0,20·netteté + 0,25·importance + 0,10·esthétique
           + 0,10·expression + 0,05·scoreLocal
```

(les pondérations sont réglables, et les valeurs proviennent de Gemini, §7.4)

- **Filtre** : si `focal < 0,3`, la personne n'est pas le sujet et le candidat
  est écarté. S'il ne reste aucun candidat, on garde le meilleur, signalé
  « sujet incertain » dans la revue.
- **Variété** (pénalité de ressemblance avec ce qui est déjà choisi) : on
  parcourt les créneaux dans l'ordre chronologique et on choisit
  `argmax qualité(c) − λ · max_{s ∈ choisis récents} ressemblance(c, s)`, avec
  λ = 0,3 et une fenêtre de 6 créneaux.
  - `ressemblance` combine les **étiquettes Gemini** (type de scène, lieu,
    activité, tenue, personnes présentes, cadrage), comparées par indice de
    Jaccard, et la proximité temporelle.
  - **Option v2** : des embeddings d'image (API d'embeddings multimodaux de
    Google) pour une mesure de ressemblance plus fine.
- **Contraintes de variété globales** (souples) : au plus 30 % de photos du
  même type de scène sur une année ; on évite trois créneaux consécutifs dans
  le même lieu.

### 5.6 Comparer à l'intérieur du créneau plutôt que noter dans l'absolu

Gemini note de façon plus cohérente quand il **compare** des photos entre
elles. On envoie donc **un appel par créneau**, avec tous ses candidats (≤ K).
Les notes absolues renvoyées servent aux pénalités de ressemblance et aux
bonus « moments forts », mais le classement est relatif au créneau.

### 5.7 Revue

```
┌ Album « Irène — Naissance » — revue ─────────────────────────────────┐
│ Créneau        Choix      Alternatives          Pourquoi             │
│ J0 · 0h        [img] 🔒   [img][img]            « peau à peau, net,  │
│                                                  1re photo »         │
│ J0 · 1h        —          (aucun candidat)                           │
│ J0 · 2h        [img]      [img][img][img]       « bain, regard … »   │
│ 3 mois         [img] ⚠    [img]                 sujet incertain      │
│ …                                                                     │
├──────────────────────────────────────────────────────────────────────┤
│ 183 / 246 créneaux remplis · coût réel 0,62 € · [Relancer non verrouillés] │
│ [Exporter…] [Créer la collection]                         [Fermer]    │
└──────────────────────────────────────────────────────────────────────┘
```

- On revoit **par pages** (une année par page), pour ne pas charger 246 lignes
  de vignettes d'un coup.
- **« Relancer non verrouillés »** recalcule la sélection avec d'autres
  pondérations ou un autre λ. Les analyses en cache sont réutilisées, sans
  nouvel appel payant.

### 5.8 Sortie

- **Collection fixe** `People / Irène Thomas / Albums / Naissance —
  2026-10-31`. L'ordre chronologique y est naturellement l'ordre des âges.
  **⚠** : le SDK ne permet pas de fixer un ordre personnalisé dans une
  collection.
- **Manifeste** `albums/<id>.json` : paramètres, créneaux, choix, alternatives,
  scores et raisons. Il permet de rouvrir la revue plus tard.
- **Export optionnel** (`LrExportSession`) vers un dossier, avec des noms
  ordonnés : `0001_J0-00h.jpg`, `0031_M05.jpg`, `0102_A03-M02.jpg`, et un fichier
  `legendes.csv` (fichier, âge, date, lieu) prêt pour un livre photo.
- **Option** : écrire l'âge (« 3 ans et 2 mois ») dans la légende IPTC des
  **copies exportées** seulement. Les originaux ne sont jamais touchés.

---

## 6. Le problème du « point focal » : savoir si la personne est le sujet

### 6.1 Ce que l'on sait déjà

- **Le mot-clé garantit (presque) la présence** de la personne. On n'a donc
  pas besoin de *reconnaître* qui elle est, seulement de savoir si elle est
  **le sujet principal**, nette et visible.
- **Le registre donne l'âge et le sexe de chaque personne taguée** à la date de
  la photo. Sur une photo taguée « Irène, Paul, Marie », Gemini peut chercher
  « un nourrisson de 2 mois », « un garçon de 5 ans » et « une femme de 38
  ans ». C'est une description, pas de la biométrie, et c'est le cas le plus
  courant dans une photothèque familiale.

### 6.2 Stratégie en cascade

| Niveau | Méthode | Quand | Coût |
|---|---|---|---|
| **N0** | Une seule personne taguée → elle est le sujet (sauf si Gemini ne voit personne) | toujours | 0 |
| **N1** | **Gemini sans référence**, guidé par la description (âge et sexe) de chaque personne taguée. Gemini détecte les personnes (cadres `box_2d`), les associe aux descriptions et note la mise en avant (taille, position, netteté, regard). | cas courant | inclus dans l'appel du créneau |
| **N2** | **Photos de référence par tranche d'âge**, quand deux personnes taguées ont un profil proche (jumeaux, cousins du même âge, sœurs adultes) | ambiguïté signalée par N1 | +2 à 4 images par appel |
| **N3** | **Vrais cadres de visage** (§6.4) | v2, ou si N2 ne suffit pas | variable |

### 6.3 Photos de référence (N2)

- **Un enfant change très vite** : une photo à 1 an n'aide pas à reconnaître le
  même enfant à 8 ans. Les références sont donc choisies **par tranche
  d'âge**, au plus près de l'âge sur la photo à analyser (± 3 mois avant 2 ans,
  ± 1 an avant 18 ans, ± 5 ans ensuite).
- **Sélection automatique** : parmi les photos où la personne est la **seule**
  taguée, on prend celles qui sont bien notées et où Gemini a déjà vu un seul
  visage net. Pas besoin de cadres de visage.
- **Épinglage manuel** dans l'éditeur de personnes : 3 à 10 photos couvrant les
  âges, pour les cas difficiles.
- **Limite** : Gemini peut refuser de **comparer des visages**, ou le faire de
  façon peu fiable. C'est une règle de sécurité des modèles sur
  l'identification de personnes. Le plugin ne dépend donc **jamais** de N2 :
  sans réponse exploitable, le créneau est marqué « sujet incertain » pour la
  revue humaine. Le spike S5 mesure le taux de réussite réel.

### 6.4 Obtenir de vrais cadres de visage (N3, v2)

| Source | Cadres | Identité | Remarques |
|---|---|---|---|
| **Vue *Personnes* de Lightroom Classic** (détection locale) | oui | oui (nom = mot-clé personne) | Le SDK ne donne pas accès aux cadres. Ils sont dans la base SQLite du `.lrcat`, lisible par un utilitaire externe, sur une **copie** du catalogue. **⚠ à valider (S6)** : schéma des tables selon la version. Indexer 80 000 photos prend du temps, mais c'est local et gratuit. |
| **Gemini** (détection d'objets) | oui (`box_2d`) | non (description seulement) | Déjà utilisé en N1. |
| **Google Cloud Vision — détection de visages** | oui | **non** (l'API ne fait volontairement pas de reconnaissance) | Renvoie aussi les probabilités de flou, de sourire et d'exposition, utiles pour le score de netteté. Coût faible. |
| **Reconnaissance locale** (InsightFace, dlib…) | oui | oui, par comparaison avec les références | Le plus fiable et le plus privé (aucun visage d'enfant ne quitte la machine). Il faut un utilitaire Python à côté du plugin. |

### 6.5 Google Photos

- **Les « personnes » de Google Photos ne sont pas accessibles par API.**
  L'API Library ne les a jamais exposées, et depuis le **31 mars 2025** elle
  ne donne plus accès qu'aux contenus créés par l'application elle-même. Les
  portées `photoslibrary.readonly` et équivalentes ont été retirées. L'API
  Picker ne permet que de sélectionner des photos à la main.
- **Google Takeout** fournit, à côté de chaque photo, un fichier JSON qui
  contient en général un tableau `people` avec les **noms** des visages
  nommés, **sans les cadres**. Il sert à **taguer** les photos (§3.5), pas à
  savoir qui est au centre.
- Google Cloud (Vision, Vertex AI) n'a pas accès à votre bibliothèque Google
  Photos.

---

## 7. Intégration Gemini

### 7.1 Appel

- Point d'accès : `POST https://generativelanguage.googleapis.com/v1beta/models/<modèle>:generateContent`,
  avec l'en-tête `x-goog-api-key`. Variante : Vertex AI, si l'on préfère un
  projet Google Cloud.
- **Sortie structurée** : `generationConfig.responseMimeType = "application/json"`
  avec un schéma de réponse (§7.4). Si le JSON est invalide, le plugin
  réessaie une fois, puis marque le créneau en erreur.
- **Images** : JPEG de 768 px sur le grand côté, obtenus par
  `requestJpegThumbnail`, encodés en base64, chacun précédé d'une étiquette
  texte (`Candidat C3`).
- **Concurrence** : 4 requêtes en parallèle. En cas d'erreur 429 ou 5xx, on
  réessaie avec un délai croissant (2, 4, 8, 16 s).
- **Mode lot (option)** : l'API Batch de Gemini, moins chère mais asynchrone
  (jusqu'à quelques heures), pour les très grands albums.

### 7.2 Coût (ordre de grandeur, **à recalculer avec la grille tarifaire en vigueur**)

Rythme « Naissance », de 0 à 18 ans, avec environ 180 créneaux non vides et en
moyenne 6 candidats par créneau, soit environ 1 100 images :

- quelques centaines de jetons par image à 768 px, selon le modèle et le
  réglage de résolution, plus le texte : **≈ 0,5 à 1 million de jetons en
  entrée** ;
- coût : **moins de 1 €** avec un modèle « Flash », **quelques euros** avec un
  modèle « Pro ».

L'estimation est affichée avant le lancement (§5.1). Grâce au cache, un nouveau
calcul ne coûte que les nouveaux candidats.

### 7.3 Prompt (esquisse)

```
Rôle : tu aides à choisir des photos de famille pour un album.

Contexte du créneau : « 5 mois » (photos du 23/11/2014 au 22/12/2014).
Personne SUJET : nourrisson, fille, environ 5 mois.
Autres personnes taguées sur certaines photos :
  - P2 : garçon, environ 5 ans
  - P3 : femme, environ 38 ans
Photos déjà retenues juste avant (pour la variété) :
  « bain, salle de bain », « parc, poussette », « bras de P3, intérieur ».

Pour chaque candidat Cn (les personnes taguées sur la photo sont indiquées) :
  1. Repère chaque personne visible (box_2d) et associe-la, si possible,
     à SUJET / P2 / P3 d'après l'âge et le sexe apparents.
  2. Note de 0 à 1 :
     - focal : SUJET est-il le sujet principal (taille, position, mise au point) ?
     - sharpness : netteté DU SUJET (pas de l'image entière)
     - expression : visage visible, yeux ouverts, expression
     - aesthetic : composition, lumière, exposition
     - importance : importance probable du moment (fête, premier bain, rencontre…)
  3. Donne 3 à 6 étiquettes (scène, lieu, activité, tenue, cadrage).
  4. Signale les quasi-doublons entre candidats.
  5. Classe les candidats et justifie le premier en 15 mots maximum.
N'identifie personne par son nom ; n'invente pas de contexte absent de l'image.
```

### 7.4 Schéma de réponse (extrait)

```json
{
  "candidates": [{
    "id": "C3",
    "subjectFound": true,
    "subjectBox": [120, 340, 610, 720],
    "ambiguous": false,
    "scores": { "focal": 0.9, "sharpness": 0.8, "expression": 0.7,
                "aesthetic": 0.6, "importance": 0.4 },
    "tags": ["intérieur", "salon", "tapis d'éveil", "body rayé", "plan moyen"],
    "nearDuplicateOf": null
  }],
  "ranking": ["C3", "C1", "C5"],
  "reason": "Sujet net au centre, sourire, scène nouvelle par rapport aux précédentes"
}
```

Si Gemini répond `ambiguous: true` (il ne sait pas qui est SUJET), le plugin
relance le créneau en mode N2, avec les photos de référence.

### 7.5 Versions et traçabilité

Chaque analyse en cache enregistre : modèle, version du prompt, date, jetons
consommés, coût estimé. Un rapport donne le coût cumulé.

### 7.6 Vie privée

- Il faut utiliser une **clé API rattachée à un compte payant** (ou Vertex AI).
  Selon les conditions de Google, les données de l'offre payante ne servent
  pas à entraîner les modèles, contrairement à l'offre gratuite. À vérifier au
  moment de la mise en service.
- On n'envoie que des **miniatures recompressées** : pas d'EXIF, pas de GPS,
  pas de nom de fichier.
- **Aucun nom de personne** n'est envoyé, seulement des rôles (SUJET, P2…) et
  des descriptions (âge, sexe).
- Le dialogue de lancement indique qu'on envoie N images à Google, avec un
  bouton « Ne plus demander ».

---

## 8. Architecture du plugin

```
LightroomPeople.lrplugin/
├── Info.lua                 menus, LrMetadataProvider, version SDK
├── PeopleMetadata.lua       champs lp* (§2.2)
├── lib/
│   ├── json.lua             JSON pur Lua
│   ├── dates.lua            parsing ISO / imprécis, intervalles, âge calendaire
│   ├── registry.lua         people.json (lecture, écriture, validation, migration)
│   ├── photoIndex.lua       date effective + âge par photo (lectures groupées)
│   ├── scanDetect.lua       règles « numérisée »
│   ├── collections.lua      F1 / F2, ensembles de collections, searchDesc
│   ├── ageTokens.lua        calcul et écriture de lpAgeTokens (par lots)
│   ├── rhythm.lua           définition et découpage des créneaux
│   ├── selector.lua         score local, rafales, sélection avec variété
│   ├── gemini.lua           client HTTP, prompt, schéma, nouvelles tentatives
│   ├── cache.lua            cache des analyses sur disque
│   └── album.lua            orchestration F4, manifeste, export
├── ui/
│   ├── PeoplePicker.lua     sélection d'une personne (recherche)
│   ├── AgeBrowser.lua       F3
│   ├── AlbumWizard.lua      paramètres + estimation
│   ├── AlbumReview.lua      revue
│   ├── RegistryEditor.lua   F5
│   └── Settings.lua         réglages, clé API
└── tests/                   tests de dates, rythmes et sélection
                             (exécutables hors Lightroom avec lua 5.1)
```

- Les modules de calcul (`dates`, `rhythm`, `selector`) **ne dépendent pas du
  SDK**. On peut donc les tester hors de Lightroom.
- Tous les traitements longs passent par `LrTasks.startAsyncTask` et
  `LrProgressScope`, avec annulation.
- Toutes les écritures dans le catalogue sont regroupées dans
  `withWriteAccessDo`, avec un nom d'annulation explicite (« Lightroom People :
  créer collections Irène »).

---

## 9. Cas limites

| Cas | Comportement |
|---|---|
| Personne sans date de naissance | F1 et F2 « Toutes » fonctionnent ; F2 par âge, F3 et F4 indisponibles, avec un message et un lien vers l'éditeur |
| Naissance imprécise (« 1932 ») | âges en intervalles ; tranches par an uniquement |
| Homonymes | identifiants distincts dans le registre ; le mot-clé fait foi |
| Photo taguée avec la personne mais où elle n'apparaît pas (photo de groupe coupée, objet lui appartenant) | Gemini renvoie `subjectFound=false` → exclue de l'album et listée dans la revue |
| Photo prise à l'étranger (décalage horaire) | ignoré, sauf le J0 heure par heure : le premier jour est découpé sur l'heure locale de l'appareil |
| Appareil mal réglé (date fausse) | visible dans le rapport d'anomalies quand il produit des dates avant la naissance ; correction par l'utilisateur dans Lightroom |
| Copies virtuelles | une seule par photo maître dans les albums |
| Vidéos | exclues de l'album par défaut ; option pour inclure une image fixe de la vidéo (v2) |
| Mot-clé renommé ou déplacé | le registre retrouve le mot-clé par son identifiant local mémorisé, puis par son chemin ; il propose de se mettre à jour |
| Catalogue déplacé | le dossier `People Data` suit le catalogue ; s'il manque, on peut en choisir un autre |

---

## 10. Spikes (à faire avant le développement)

| # | Question | Méthode | Critère de réussite |
|---|---|---|---|
| S1 | Un champ `searchable` du plugin est-il utilisable dans une collection dynamique ? Quel nom de critère dans `searchDesc` ? | créer la collection dans l'interface, puis l'exporter en `.lrsmcol` ou lire `collection:getSearchDescription()` | collection créée par code, résultats identiques |
| S2 | Syntaxe de `captureTime` (`in`, `value`, `value2`) | idem | idem |
| S3 | Vitesse et qualité de `requestJpegThumbnail` à 768 px, avec et sans aperçus standard | 200 photos | < 100 ms par photo avec aperçus |
| S4 | `LrHttp.post` avec 8 images (≈ 1 Mo en base64) et 4 tâches en parallèle | appel réel | aucun délai dépassé, Lightroom reste utilisable |
| S5 | Fiabilité de N1 et N2 (sujet trouvé, ambiguïtés, refus) | 200 photos annotées à la main, dont 50 avec plusieurs enfants | N1 ≥ 90 % correct sur les cas non ambigus ; mesure du taux de refus de N2 |
| S6 | Cadres de visage de Lightroom dans le `.lrcat` (tables, coordonnées) | lecture SQLite d'une copie | association cadre ↔ mot-clé retrouvée |
| S7 | Champ `people` dans les JSON de Google Takeout et taux de correspondance avec le catalogue | export d'un échantillon | ≥ 95 % de correspondance sur l'échantillon |

---

## 11. Jalons

1. **M1 — Base** : registre, dates, éditeur de personnes, rapport d'anomalies,
   F1. *Tests unitaires de `dates.lua`.*
2. **M2 — Navigation** : index photo → âge, F3, détection et datation des
   numérisations.
3. **M3 — Collections dynamiques** : F2 (mode A, puis B après S1).
4. **M4 — Album sans IA** : rythmes, score local, rafales, revue, sortie et
   export.
5. **M5 — Album + Gemini** : client, cache, score, variété, N0 à N2,
   estimation du coût.
6. **M6 (v2)** : import Takeout, vrais cadres de visage (Lightroom, Vision ou
   reconnaissance locale), embeddings, bonus « moments forts » appris.

---

## 12. Questions ouvertes

1. **Origine des tags personnes** : seront-ils posés avec la vue *Personnes*
   de Lightroom (on aurait alors des cadres de visage, §6.4), à la main, ou
   importés de Google Photos ?
2. Les 80 000 photos sont-elles **aussi dans Google Photos**, avec des visages
   nommés ? (Cela détermine l'intérêt de l'import Takeout.)
3. Comment les **numérisations** sont-elles datées aujourd'hui : date de prise
   de vue corrigée à la main, date du scan, ou rien ?
4. **Usage de l'album** : livre imprimé (combien de pages ou de photos au
   maximum ?), diaporama, dossier partagé ?
5. Acceptez-vous d'**envoyer des miniatures** de photos d'enfants à l'API
   Gemini (offre payante) ? Sinon, on donne la priorité à la reconnaissance
   locale (§6.4).
6. Version de Lightroom Classic et système (Mac ou Windows) ?
7. Hiérarchie de mots-clés existante à respecter ?
