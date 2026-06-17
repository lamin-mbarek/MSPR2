# FutureKawa — Documentation API Backend

Documentation des API à destination de l'équipe Frontend.
Deux services indépendants, conteneurisés :

| Service | URL locale | Rôle |
|---------|-----------|------|
| **Backend Central (siège)** | `http://localhost:8001` | Point d'entrée du frontend. Consolide et route vers les pays. |
| **Backend Pays (Brésil)** | `http://localhost:8000` | Logique métier + base PostgreSQL + MQTT. |

> **Le frontend doit appeler le Backend Central (8001).** Il ne s'adresse jamais directement au backend pays.
> Architecture : `Frontend → Central (8001) → Backend pays (8000) → PostgreSQL`

- CORS est activé sur les deux backends (`allow_origins=["*"]`).
- Toutes les réponses sont en JSON.
- Documentation interactive auto-générée (Swagger) :
  - Central : `http://localhost:8001/docs`
  - Brésil : `http://localhost:8000/docs`

Dans cette doc, `{country}` correspond au code pays. Aujourd'hui une seule valeur disponible : **`brazil`**.

---

## 1. Backend Central (port 8001) — à utiliser par le frontend

### Infos générales

#### `GET /`
Statut du service.
```json
{ "message": "FutureKawa Central Backend", "pays_connectes": ["brazil"], "status": "running" }
```

#### `GET /countries`
Liste des pays connectés (pour alimenter un sélecteur de pays).
```json
[ { "code": "brazil", "nom": "Brésil" } ]
```

---

### Données d'un pays

#### `GET /countries/{country}/kpis`
Indicateurs consolidés du pays.
```json
{
  "total_lots": 5,
  "total_alerts": 3,
  "total_mesures": 42,
  "lots_conformes": 3,
  "lots_alerte": 0,
  "lots_expiration_proche": 1,
  "lots_expires": 1,
  "total_quantite_kg": 2400,
  "derniere_temperature": 29.5,
  "derniere_humidite": 55.5,
  "temperature_moyenne": 29.0,
  "humidite_moyenne": 55.0
}
```

#### `GET /countries/{country}/lots`
Liste de tous les lots du pays.
```json
[
  {
    "Id_lot": 1,
    "date_stockage": "2025-01-10",
    "statut": "conforme",
    "quantite": 800,
    "date_expiration": "2026-01-10",
    "Id_entrepot": 1
  }
]
```

#### `GET /countries/{country}/lots/fifo`
Mêmes données que `/lots`, mais **triées par date de stockage croissante** (les lots les plus anciens en premier — logique FIFO).

#### `POST /countries/{country}/lots`
Créer un lot. **Body :**
```json
{
  "date_stockage": "2026-06-01",
  "date_expiration": "2027-06-01",
  "quantite": 500,
  "statut": "conforme",
  "Id_entrepot": 1
}
```
Réponse : le lot créé (avec son `Id_lot`).

#### `DELETE /countries/{country}/lots/{id}`
Supprimer un lot.
```json
{ "message": "Lot 3 supprimé" }
```

#### `POST /countries/{country}/lots/check-expiration`
Lance la vérification de l'âge et de l'expiration de tous les lots.
Crée automatiquement les alertes nécessaires (lot > 365 j, expiré, proche expiration) et envoie les emails. Met aussi à jour le `statut` des lots.
```json
{ "message": "Vérification âge/expiration terminée" }
```

#### `GET /countries/{country}/mesures?limit=50`
Dernières mesures (les plus récentes d'abord). Paramètre optionnel `limit` (défaut 50).
```json
[
  {
    "Id_measure": 42,
    "temperature": 29.5,
    "humidite": 55.5,
    "date_mesure": "2026-06-15T10:30:00",
    "Id_entrepot": 1
  }
]
```

#### `POST /countries/{country}/mesures`
Ajouter une mesure manuellement. Déclenche la vérification d'alerte température/humidité.
**Body :**
```json
{ "temperature": 29.0, "humidite": 55.0, "Id_entrepot": 1 }
```

#### `GET /countries/{country}/alerts`
Toutes les alertes du pays (les plus récentes d'abord).
```json
[
  {
    "Id_alert": 7,
    "message": "Température anormale entrepôt #1 : 35.0°C (seuil Brazil : 26.0-32.0°C)",
    "type_alerte": "temperature",
    "niveau": "critical",
    "created_at": "2026-06-15T10:31:00",
    "Id_lot": 3
  }
]
```
- `type_alerte` : `temperature`, `humidite`, `expiration`, `expiration_proche`, `lot_ancien`
- `niveau` : `warning` ou `critical`

#### `GET /countries/{country}/alerts/critical`
Idem, mais uniquement les alertes de niveau `critical`.

#### `GET /countries/{country}/entrepots`
Liste des entrepôts du pays.
```json
[
  { "Id_entrepot": 1, "nom": "Entrepôt Principal SP", "localisation": "Zone Sud, São Paulo", "capacite_stockage": 10000 }
]
```

#### `GET /countries/{country}/dashboard`
Vue prête à l'emploi pour un tableau de bord pays (KPIs + 6 dernières alertes + 20 dernières mesures).
```json
{
  "country": "brazil",
  "kpis": { "...": "voir /kpis" },
  "latest_alerts": [ "..." ],
  "latest_mesures": [ "..." ]
}
```

---

### Données consolidées (tous les pays)

Chaque élément renvoyé contient un champ supplémentaire `_country` indiquant son pays d'origine.

| Méthode | Route | Description |
|---------|-------|-------------|
| `GET` | `/dashboard` | KPIs additionnés de tous les pays + détail par pays (`kpis_par_pays`) |
| `GET` | `/lots` | Tous les lots, tous pays confondus |
| `GET` | `/lots/fifo` | FIFO global (trié par date de stockage, tous pays) |
| `GET` | `/alerts` | Toutes les alertes, tous pays |
| `GET` | `/mesures?limit=50` | Toutes les mesures, tous pays |

Exemple `GET /dashboard` :
```json
{
  "pays_connectes": 1,
  "total_lots": 5,
  "total_alerts": 3,
  "total_mesures": 42,
  "lots_conformes": 3,
  "lots_expires": 1,
  "lots_expiration_proche": 1,
  "lots_alerte": 0,
  "total_quantite_kg": 2400,
  "kpis_par_pays": { "brazil": { "...": "..." } }
}
```

---

## 2. Backend Pays — Brésil (port 8000)

> Normalement, le frontend n'appelle pas directement ce backend. Ces routes sont documentées pour référence / debug. Le central les expose toutes via `/countries/{country}/...`.

| Méthode | Route | Description |
|---------|-------|-------------|
| `GET` | `/` | Statut |
| `GET` | `/pays` | Pays + seuils (température/humidité min/max) |
| `GET` | `/entrepots` | Liste des entrepôts |
| `POST` | `/entrepots` | Créer un entrepôt |
| `GET` | `/lots` | Liste des lots |
| `POST` | `/lots` | Créer un lot |
| `GET` | `/lots/fifo` | Lots triés FIFO |
| `GET` | `/lots/{id}` | Détail d'un lot |
| `PUT` | `/lots/{id}` | Modifier un lot |
| `DELETE` | `/lots/{id}` | Supprimer un lot |
| `POST` | `/lots/check-expiration` | Vérifier âge / expiration |
| `GET` | `/mesures?limit=50` | Dernières mesures |
| `GET` | `/mesures/latest` | Dernière mesure |
| `POST` | `/mesures` | Ajouter une mesure |
| `GET` | `/alerts` | Toutes les alertes |
| `GET` | `/alerts/critical` | Alertes critiques |
| `GET` | `/kpis` | KPIs |

### `GET /pays`
```json
[
  {
    "Id_pays": 1,
    "nom": "Brazil",
    "temp_min": 26.0, "temp_max": 32.0,
    "humidity_min": 53.0, "humidity_max": 57.0
  }
]
```

---

## 3. Modèles de données

### Lot
| Champ | Type | Description |
|-------|------|-------------|
| `Id_lot` | int | Identifiant |
| `date_stockage` | date (`YYYY-MM-DD`) | Entrée en entrepôt |
| `date_expiration` | date (`YYYY-MM-DD`) | Date de péremption |
| `quantite` | int | Quantité en kg |
| `statut` | string | `conforme` \| `alerte` \| `expiration_proche` \| `expire` |
| `Id_entrepot` | int | Entrepôt de rattachement |

### Mesure
| Champ | Type | Description |
|-------|------|-------------|
| `Id_measure` | int | Identifiant |
| `temperature` | number | °C |
| `humidite` | number | % |
| `date_mesure` | datetime ISO | Horodatage |
| `Id_entrepot` | int | Entrepôt mesuré |

### Alerte
| Champ | Type | Description |
|-------|------|-------------|
| `Id_alert` | int | Identifiant |
| `message` | string | Description lisible |
| `type_alerte` | string | `temperature` \| `humidite` \| `expiration` \| `expiration_proche` \| `lot_ancien` |
| `niveau` | string | `warning` \| `critical` |
| `created_at` | datetime ISO | Horodatage |
| `Id_lot` | int | Lot concerné |

### Entrepôt
| Champ | Type |
|-------|------|
| `Id_entrepot` | int |
| `nom` | string |
| `localisation` | string |
| `capacite_stockage` | int |

---

## 4. Règles métier (pour info côté front)

- **Seuils Brésil** : température 26–32°C, humidité 53–57% (modifiables en base via la table `pays`).
- **Alerte conditions** : créée si une mesure dépasse les seuils.
- **Alerte lot ancien** : lot stocké depuis plus de 365 jours.
- **Alerte expiration** : lot expiré (`critical`) ou expirant sous 30 jours (`warning`).
- **Anti-duplication** : une alerte identique n'est pas recréée.
- **Email** : un email est envoyé au responsable à chaque nouvelle alerte.

---

## 5. Exemples d'appels (fetch)

```javascript
const API = "http://localhost:8001";

// Lire les KPIs du Brésil
const kpis = await fetch(`${API}/countries/brazil/kpis`).then(r => r.json());

// Lister les lots
const lots = await fetch(`${API}/countries/brazil/lots`).then(r => r.json());

// Créer un lot
await fetch(`${API}/countries/brazil/lots`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    date_stockage: "2026-06-01",
    date_expiration: "2027-06-01",
    quantite: 500,
    statut: "conforme",
    Id_entrepot: 1
  })
});

// Supprimer un lot
await fetch(`${API}/countries/brazil/lots/3`, { method: "DELETE" });

// Ajouter une mesure
await fetch(`${API}/countries/brazil/mesures`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ temperature: 29.0, humidite: 55.0, Id_entrepot: 1 })
});

// Lire les alertes
const alerts = await fetch(`${API}/countries/brazil/alerts`).then(r => r.json());
```
