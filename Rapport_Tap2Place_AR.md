# Rapport de projet : Application Tap to Place en Realite Augmentee

## 1. Presentation du projet

L'objectif de ce projet est de realiser une application mobile Android de realite augmentee (AR) de type **Tap to Place** : l'utilisateur pointe la camera de son telephone vers une surface reelle (sol, table...), l'application detecte cette surface, puis un tap sur l'ecran place un objet 3D virtuel a l'endroit touche.

---

## 2. Demarche suivie

### 2.1 Choix technologiques

| Element                | Choix retenu                                                  |
|------------------------|---------------------------------------------------------------|
| Moteur                 | **Unity 6** (version 6000.3.1f1)                              |
| Framework AR           | **AR Foundation 6.3.1**                                       |
| Plugin plateforme      | **Google ARCore XR Plugin 6.3.1**                             |
| Pipeline de rendu      | **Universal Render Pipeline (URP) 17.3.0**                    |
| Systeme d'input        | **Unity Input System 1.17.0**                                 |
| Plateforme cible       | Android (API 24+, ARM64, IL2CPP)                              |

**Pourquoi AR Foundation ?** C'est la couche d'abstraction officielle de Unity pour la realite augmentee. Elle fournit une API unifiee qui fonctionne avec ARCore (Android) et ARKit (iOS) sans modifier le code applicatif. Sous le capot, c'est ARCore de Google qui effectue la detection sur Android.

### 2.2 Point de depart : le depot officiel AR Foundation Samples

Plutot que de partir d'un projet vide, j'ai clone le depot officiel de Unity :

```
git clone https://github.com/Unity-Technologies/arfoundation-samples.git
```

Ce depot (branche `main`, compatible Unity 6) contient plusieurs scenes de demonstration. J'ai utilise la scene **SimpleAR** (`Assets/Scenes/SimpleAR/SimpleAR.unity`) qui implemente exactement le comportement Tap to Place recherche.

**Justification :** partir des samples officiels permet d'avoir un projet deja configure correctement (packages, render pipeline, prefabs) et de se concentrer sur la comprehension du fonctionnement plutot que sur la configuration initiale.

### 2.3 Etapes de mise en oeuvre

1. **Clonage du depot** et ouverture dans Unity 6
2. **Chargement de la scene** `SimpleAR` dans l'editeur
3. **Configuration Android** :
   - `File > Build Settings > Android > Switch Platform`
   - `Project Settings > XR Plug-in Management > Android > ARCore` coche
   - `Player Settings > Android` : API minimum 24, IL2CPP, ARM64
4. **Configuration de la scene de build** : seule la scene SimpleAR dans la liste `Scenes In Build`
5. **Preparation du telephone** : activation du mode developpeur et du debogage USB
6. **Build and Run** : compilation et deploiement direct sur le telephone Android

---

## 3. Fonctionnement de la scene SimpleAR

### 3.1 Architecture globale de la scene

La scene SimpleAR contient trois objets racine :

```
Scene SimpleAR
|
|-- Directional Light              -> Eclairage de la scene
|-- Canvas Template (prefab)       -> Interface utilisateur (boutons Reset, Pause, titre)
|-- XR Origin (prefab)             -> Coeur du systeme AR
    |-- Camera Offset
    |   |-- Main Camera            -> Camera AR (image reelle + objets virtuels)
    |-- [Composants sur le XR Origin] :
        |-- ARPlaneManager         -> Detecte les surfaces planes
        |-- ARPointCloudManager    -> Detecte les points d'interet (feature points)
        |-- ARRaycastManager       -> Lance des raycasts vers les surfaces detectees
        |-- RaycastEventController -> Ecoute les inputs et declenche les raycasts
        |-- ARPlaceObject          -> Place un objet 3D au point d'impact
        |-- ARSession              -> Gere le cycle de vie de la session AR
```

### 3.2 Detection des surfaces : le pipeline complet

La detection fonctionne en trois couches superposees :

#### Couche basse : ARCore (natif)

Le SDK natif ARCore de Google fait le travail de vision par ordinateur :

- **Visual-Inertial Odometry (VIO)** : la camera capture des images en continu et les combine avec les capteurs inertiels (accelerometre, gyroscope) pour suivre le mouvement de l'appareil dans l'espace 3D.
- **Extraction de feature points** : des points d'interet visuels sont identifies dans chaque image (coins, contrastes, textures).
- **Triangulation** : en suivant les memes feature points entre images successives prises depuis des positions differentes, le systeme estime leur profondeur par triangulation.
- **Plane fitting** : un algorithme ajuste des surfaces planes sur les groupes de points 3D coplanaires. Les plans sont agrandis et affines au fur et a mesure que la camera observe davantage de donnees.

#### Couche intermediaire : XRPlaneSubsystem (abstraction Unity)

La classe `XRPlaneSubsystem` definit une interface abstraite (`Provider`) que chaque plugin plateforme (ARCore, ARKit) doit implementer. Ses responsabilites :

- Exposer les changements de plans via `GetChanges()`, qui retourne des listes de `BoundedPlane` **ajoutes**, **mis a jour** ou **supprimes**.
- Chaque `BoundedPlane` est une structure de donnees contenant :
  - `pose` : position et orientation du plan dans l'espace de session
  - `size` / `extents` : dimensions du plan en metres
  - `alignment` : horizontal (sol, table) ou vertical (mur)
  - `classifications` : type semantique (sol, mur, plafond, table, siege...)
  - `boundary` : sommets du contour polygonal du plan
  - `trackingState` : qualite du suivi (none, limited, tracking)

#### Couche haute : ARPlaneManager (Unity MonoBehaviour)

Le composant `ARPlaneManager` effectue un **polling** a chaque frame Unity dans sa methode `Update()` :

```
A chaque frame :
    changements = subsystem.GetChanges()

    Pour chaque plan AJOUTE :
        Instancier un nouveau GameObject (prefab "AR Plane Debug Visualizer")
        Y attacher un composant ARPlane
        Afficher le maillage de la surface detectee

    Pour chaque plan MIS A JOUR :
        Mettre a jour la position, rotation, dimensions et contour

    Pour chaque plan SUPPRIME :
        Detruire le GameObject correspondant

    Emettre l'evenement trackablesChanged
```

Dans la scene SimpleAR, le prefab de visualisation utilise est **AR Plane Debug Visualizer** : il dessine un maillage semi-transparent sur chaque surface detectee pour la rendre visible a l'utilisateur.

### 3.3 Placement d'objet : du tap au cube

Le placement d'un objet suite a un tap utilise une chaine de trois scripts qui communiquent via un **evenement Scriptable Object** (`ARRaycastHitEventAsset`) :

```
    [1] Input (tap ecran)
            |
            v
    [2] RaycastEventController
        - Ecoute le tap via le Input System
        - Lance un AR Raycast vers les plans detectes
        - Si un plan est touche : emet un evenement ARRaycastHitEvent
            |
            v
    [3] ARPlaceObject
        - S'abonne a l'evenement ARRaycastHitEvent
        - Instancie le prefab "AR Placed Cube" a la position du hit
        - Si l'objet existe deja, le deplace au nouveau point
```

#### Script 1 : RaycastEventController.cs

Ce script gere l'input et le raycast. Il s'abonne a trois types d'actions :
- **Screen tap** : tap tactile sur ecran de telephone
- **Right/Left trigger** : pression de gachette pour casques VR/MR

Lors d'un tap :
1. Il verifie que le tap ne touche pas un element d'UI (via `GraphicRaycaster`)
2. Il lance un raycast AR depuis la position du touch via `ARRaycastManager.Raycast()`
3. Le raycast cherche des intersections avec les plans detectes (type `PlaneWithinPolygon`)
4. Si un hit est trouve, il emet l'evenement `ARRaycastHitEventAsset.Raise(hit)`

```csharp
void ScreenTapped(InputAction.CallbackContext context)
{
    // Verifie que ce n'est pas un tap sur l'UI
    if (m_HasGraphicRaycaster) { /* ... filtre les taps UI ... */ }

    var tapPosition = pointer.position.ReadValue();

    // Lance le raycast AR
    if (m_RaycastManager.Raycast(tapPosition, s_Hits, m_TrackableType))
    {
        // Emet l'evenement avec le premier hit (le plus proche)
        m_ARRaycastHitEvent.Raise(s_Hits[0]);
    }
}
```

#### Script 2 : ARPlaceObject.cs

Ce script s'abonne a l'evenement de raycast et gere le placement :

```csharp
void PlaceObjectAt(object sender, ARRaycastHit hitPose)
{
    // Determine le parent (le plan touche)
    Transform objectParent = hitPose.trackable?.transform;

    // Premiere fois : instancier le prefab
    if (m_SpawnedObject == null)
    {
        m_SpawnedObject = Instantiate(m_PrefabToPlace, objectParent);
    }

    // Positionner l'objet au point d'impact (avec un leger offset)
    var offset = hitPose.pose.rotation * Vector3.up * 0.025f;
    m_SpawnedObject.transform.position = hitPose.pose.position + offset;
    m_SpawnedObject.transform.parent = objectParent;
}
```

Points notables :
- L'objet est **parente au plan** (`objectParent`), donc s'il est mis a jour par ARCore, l'objet suit.
- Un **offset** de 2.5 cm vers le haut est applique pour que l'objet ne soit pas enfonce dans le sol.
- Un seul objet est place a la fois : les taps suivants le **deplacent** au lieu d'en creer un nouveau.

### 3.4 Elements d'interface (UI)

La scene inclut un Canvas avec :
- Un **titre** "Simple AR" affiche en haut
- Un bouton **Reset** qui appelle `ARSession.Reset()` pour reinitialiser la detection
- Un bouton **Pause** qui desactive/reactive la session AR

### 3.5 Resume du flux complet

```
1. Lancement de l'application
   -> ARSession demarre, la camera AR s'active

2. L'utilisateur deplace le telephone
   -> ARCore analyse les images + capteurs inertiels
   -> Des feature points et des surfaces planes sont detectes
   -> ARPlaneManager cree des GameObjects avec un mesh semi-transparent
      pour chaque surface detectee
   -> ARPointCloudManager affiche les feature points sous forme de nuage de points

3. L'utilisateur tape sur une surface detectee
   -> RaycastEventController capte le tap via l'Input System
   -> Un AR Raycast est lance depuis la position du doigt vers les plans
   -> Le raycast touche un plan detecte
   -> L'evenement ARRaycastHitEvent est emis avec la position du hit

4. ARPlaceObject recoit l'evenement
   -> Instancie un cube 3D ("AR Placed Cube") a la position du hit
   -> Le cube est parente au plan detecte

5. Taps suivants
   -> Le cube existant est deplace a la nouvelle position

6. Bouton Reset
   -> Reinitialise la session AR, efface tous les plans et objets
```

---

## 4. Technologies et concepts cles

| Concept                    | Description                                                                                  |
|----------------------------|----------------------------------------------------------------------------------------------|
| AR Foundation              | Framework Unity multiplateforme pour la realite augmentee                                    |
| ARCore                     | SDK Google de RA pour Android (vision par ordinateur, detection de surfaces, suivi de pose)   |
| XR Origin                  | Point d'ancrage de la scene AR, transforme les coordonnees AR en coordonnees Unity            |
| AR Plane Manager           | Composant qui detecte et gere les surfaces planes dans l'environnement                       |
| AR Raycast Manager         | Composant qui permet de lancer des raycasts vers les surfaces AR detectees                   |
| AR Session                 | Gere le cycle de vie de la session AR (demarrage, pause, reinitialisation)                   |
| Scriptable Object Event    | Pattern de communication par evenements decouplant l'emission (raycast) de la reception (placement) |
| Input System               | Nouveau systeme d'input de Unity, gere le tactile et les controleurs XR de maniere unifiee   |
| URP                        | Universal Render Pipeline, pipeline de rendu optimise pour le mobile                         |

---

## 5. Conclusion

Ce projet a permis de mettre en place une application de realite augmentee fonctionnelle sur Android en utilisant le framework AR Foundation de Unity. En partant des samples officiels de Unity, j'ai pu me concentrer sur la comprehension de l'architecture et des mecanismes de detection de surfaces plutot que sur la configuration initiale du projet.

Le fonctionnement repose sur une architecture en couches : ARCore effectue la detection de surfaces par vision par ordinateur en natif, AR Foundation fournit une abstraction unifiee via le systeme de subsystems, et les scripts applicatifs (`RaycastEventController`, `ARPlaceObject`) gerent l'interaction utilisateur et le placement d'objets via un systeme d'evenements decouples.
