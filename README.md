# README – OfficePulse

Cette solution combine un workflow Make (fichier JSON à copier-coller), un webhook Slack en trigger principal, un webhook secondaire pour bypasser le timeout Slack, et un agent Dust (Claude 4 en mode déterministe) connecté à des onglets Google Sheets. Quatre commandes slash (`/pulse`, `/hr`, `/om`, `/management`) permettent d'interroger directement l'agent Dust depuis Slack, qui puise dans les données de pulse_log et renvoie la réponse dans Slack. Un workflow Form Slack gère la déclaration / annulation des présences, et un tableau Looker Studio affiche en temps réel les statistiques de fréquentation.

---

## 1. Documentation technique

### 1.1 Guide d'installation et de configuration

#### 1.1.1 Importer le workflow Make

1. Ouvrez Make et créez un nouveau scénario.
2. Copiez-collez le JSON contenu dans `blueprint.json` (workflow Make prêt à l'emploi).
3. Ce workflow Make est structuré ainsi :
   - **Webhook Trigger** (Make écoute une requête POST)
   - **Bypass du timeout Slack** : réception immédiate de la slash command → réponse 200 OK → Make continue de traiter en arrière-plan pour interroger l'agent Dust
   - **Module "Dust"** : appel à l'agent spécialisé (prompt paramétré, modèle Claude 4 déterministe, RAG sur les onglets Google Sheets)
   - **Module "Slack – Create a message"** : envoie du résultat de Dust dans Slack (canal public ou DM)

#### 1.1.2 Créer et configurer l'application Slack

1. Allez sur [Slack API](https://api.slack.com) et cliquez sur **Create App** → **From scratch**.
2. Dans **OAuth & Permissions**, ajoutez au Bot Token les scopes suivants :
   - `commands`
   - `chat:write`
   - `im:write`
   - `mpim:write`
   - `channels:read`
   - `groups:read`
   - `im:read`
   - `mpim:read`
   - `users:read`
   - `channels:history`
   - `groups:history`
   - `im:history`
   - `mpim:history`

   Ces scopes permettent d'utiliser les slash commands, d'envoyer des messages dans les canaux ou en DM, et de lire l'historique des canaux.

3. Dans **Slash Commands** → **Create New Command** :
   - **Command** : `/pulse` (répétez pour `/hr`, `/om`, `/management`)
   - **Request URL** : collez l'URL HTTPS du Webhook Trigger dans Make (ex. `https://hook.make.com/abcdef123456`)
4. Cliquez sur **Install to Workspace** et autorisez les scopes.

#### 1.1.3 Paramétrage de Dust

1. Créez vos agents dans Dust (Dashboard Dust) :
   - Un agent "Collaborateurs" (prompt basique)
   - Un agent "PEAgent" pour RH
   - Un agent "ManagerAgent" pour management
   - Un agent "OfficeMgmtAgent" pour office management
   - Un agent "NotificationAgent" pour notifications ciblées (Guillaume Deramchi)

2. **Modèle** : sélectionnez Claude 4 en mode déterministe pour garantir des réponses stables.

3. **RAG Configuration** : connectez les onglets Google Sheets (`pulse_log`, `notifications_sent`, `user_info`) pour que l'agent Dust puisse récupérer en temps réel :
   - Qui vient au bureau (`pulse_log`)
   - Qui a déjà été notifié (`notifications_sent`)
   - Équipe, centres d'intérêt, localisation (`user_info`)

4. **Prompts** : rédigez un prompt CIOE (Contexte/Instructions/Output/Exemple) pour chaque agent, en explicitant la logique de filtrage sur les onglets Sheets.

### 1.2 Documentation des API et intégrations développées

#### Backend Node.js (Express)

- **Endpoint POST `/slack/events`** : traite les Slash Commands et les Events API
  - Si `type === "url_verification"`, renvoyer `{ "challenge": "<valeur>" }`
  - Si `command === "/officepulse"`, ack 200 immédiatement, appelle Dust, puis poste la réponse via `response_url`
- **Endpoint GET `/health`** : retour `{ "status": "UP", "timestamp": <ms> }` pour le monitoring
- **Endpoint POST `/make/webhook`** : reçoit les appels Make (cron, annulations…), orchestre les opérations (Dust, Google Sheets, Slack)

#### Google Sheets (via googleapis)

Fonctions principales :

```javascript
// getRows(spreadsheetId, sheetName, filterDate) → liste d'objets filtrés
// deleteRowByKey(spreadsheetId, sheetName, columnName, lookupValue) → supprime la ligne correspondant à "row_key"
// appendRow(spreadsheetId, sheetName, valuesArray) → ajoute une ligne
```

- Champs clés dans `pulse_log` : `user_id`, `nom`, `equipe`, `date`, `localisation`, `interet`, `heure_arrivee`
- Champs clés dans `notifications_sent` : `date`, `userA_id`, `userB_id`, `motif`

#### Slack WebClient

- Vérification des signatures dans le middleware Express
- Méthode `conversations.open({ users })` → obtient `channel.id` pour DM
- Méthode `chat.postMessage({ channel, text })` → envoie du message en canal ou DM

#### Workflow Form Slack (Workflow Builder)

Actions disponibles :
- "Google Sheets – Add a row" pour enregistrer une venue
- "Google Sheets – Delete a spreadsheet row" pour gérer l'annulation (via `row_key`)

### 1.3 Description de l'architecture finale

```mermaid
flowchart TD
  SlackApp[Slack App (/pulse,/hr,/om,/management)] -->|Slash Command| Backend(Node.js)
  Backend -->|RAG calls| DustAPI
  DustAPI -->|Réponses IA| Backend
  Backend -->|Lire/Écrire| GoogleSheets((Google Sheets))
  n8nMake[n8n / Make] -->|Scheduler/Webhook| Backend
  n8nMake -->|Lire/Écrire| GoogleSheets
  GoogleSheets -->|Données (Présence/Notifs/User)| DustAPI
  Backend -->|Poster message| SlackApp
  Dashboards[Looker Studio] -->|Visualisation| GoogleSheets
```

- **Slack App** : gère les Slash Commands (`/pulse`, `/hr`, `/om`, `/management`), les shortcuts Form, et les DM
- **Backend Node.js** : route `/slack/events` (vérifie signature, ack 200, appelle Dust, poste réponse), route `/make/webhook` pour les scénarios Make/n8n, route `/health` pour monitoring
- **Dust API** : ensemble d'agents IA (Claude 4 déterministe) configurés en RAG sur les onglets Google Sheets
- **Google Sheets** :
  - `pulse_log` : l'historique/journal des présences
  - `notifications_sent` : log des notifications envoyées (évite doublons)
  - `user_info` : fiches utilisateurs (équipe, centres d'intérêt, localisation)
- **n8n / Make** : orchestrateur de workflows cron et Webhooks (notifications, annulations, appel à Dust)
- **Looker Studio** : dashboard en lecture seule connecté à Google Sheets pour visualiser taux d'occupation, répartition équipe/zone, historique mensuel

---

## 2. Guide utilisateur

### 2.1 Utilisation des commandes Slash

#### `/pulse` – Contrôle de présence

Permet de lister qui est ou n'est pas au bureau pour un jour donné.

```
/pulse qui sera au bureau demain ?
```

→ Make reçoit la request, répond 200 immédiatement, appelle l'agent "Collaborateurs" Dust qui lit `pulse_log` pour `date = demain` et envoie la liste dans Slack.

#### `/hr` – Ressources Humaines

Pour poser des questions RH (taux d'engagement, équipe faible, recommandations…).

```
/hr quelle équipe a eu le taux de présence le plus bas la semaine dernière ?
```

→ Dust lit `pulse_log` sur la période, agrège par équipe, renvoie la réponse.

#### `/om` – Office Management

Pour gérer occupation, ménage, repas, sécurité…

```
/om occupation zone Jupiter demain
```

→ Dust lit `pulse_log`, `room_bookings`, `office_feedback`, calcule le taux d'occupation, renvoie un résumé.

#### `/management` – Management

Pour disponibilité d'équipe, suivi de projets, performance…

```
/management qui de l'équipe Dev est disponible demain matin ?
```

→ Dust croise `pulse_log` et, si besoin, `project_tracker`, `performance_metrics`, renvoie la liste.

### 2.2 Workflow Form Slack (Déclaration / Annulation)

1. Créer un workflow dans Slack → **Workflow Builder** → **Create** → **From scratch**
2. **Trigger** : choisissez **Shortcut** (par ex. "Je viens demain")
3. **Formulaire** : deux champs obligatoires :
   - Date (format jj/mm/aaaa)
   - Zone (ex. "Paris", "Lyon", "Toulouse")
4. **Étape Google Sheets – Add Row** : enregistre dans `pulse_log` (`user_id`, `nom`, `date`, `zone`, `row_key = date_userID`)
5. Pour l'annulation, créez un Shortcut "Annuler ma venue" : le formulaire ne demande qu'une Date
   - **Étape Google Sheets – Delete a spreadsheet row** : supprime la ligne dans `pulse_log` correspondant à `row_key = date_userID`
6. À chaque soumission, Slack envoie une confirmation : "Ta venue a bien été enregistrée/annulée."

### 2.3 Notifications intelligentes (exemple Guillaume Deramchi)

1. **Déclencheur Scheduler** dans Make → 18 h (Europe/Paris)
2. **Google Sheets – Get rows** sur `pulse_log` pour `date = demain`
3. **Filtrer** : si `user_id = U_GDeramchi` est absent et s'il existe au moins un membre `equipe = Dev` prévu, sélectionner le collègue partageant un intérêt ou la même zone
4. **Search rows** sur `notifications_sent` pour éviter le doublon le même jour
5. **Slack – Create a message** : envoie un DM à `U_GDeramchi` :

> Salut Guillaume ! Demain, Lucie Martin (Dev – passionnée de lecture) sera au bureau. Ce serait sympa de te joindre à elle pour échanger !

en utilisant `conversations.open` puis `chat.postMessage`.

6. **Google Sheets – Add row** dans `notifications_sent` (`date`, "U_GDeramchi", `userB_id`, "collègue_équipe")

### 2.4 Tableau de bord Looker Studio

1. Connecter Looker Studio à Google Sheets (`pulse_log`)
2. Créer un rapport affichant :
   - Taux d'occupation journaliers (%)
   - Répartition par équipe et zone
   - Historique mensuel
3. Configurer l'actualisation automatique (toutes les heures)
4. Partager en lecture seule avec les RH, managers et office management

---

## 3. Guide utilisateur additionnel

En complément des commandes Slash décrites ci-dessus, voici quelques conseils par profil, procédures d'urgence et FAQ courantes.

### 3.1 Collaborateurs (@HybrideCoach)

- **Consulter les présences** :
  ```
  @HybrideCoach Qui est au bureau aujourd'hui ?
  ```

- **Déclarer / annuler sa venue** :
  - Utilisez le workflow externe (Shortcut "Je viens demain") pour enregistrer
  - Pour annuler, utilisez le Shortcut "Annuler ma venue" et indiquez la date à supprimer

- **Demander un profil** :
  ```
  @HybrideCoach Profil complet Emma Petit
  ```

- **Infos zones** :
  ```
  @HybrideCoach Capacité de Pluto ?
  ```

- **Bonnes pratiques** :
  - Toujours préciser la date au format jj/mm/aaaa pour les requêtes futures ou passées
  - Les réponses du bot ne dépassent pas 5 phrases ; si la période ou l'intention n'est pas claire, le bot demandera une précision

### 3.2 Managers (@ManagerAgent)

- **Présence du jour par équipe** :
  ```
  @ManagerAgent Présence team Growth aujourd'hui
  ```

- **Prévisions hebdo** :
  ```
  @ManagerAgent Plan Dev semaine du 09/06/2025
  ```

- **Couverture & recommandations** :
  - L'agent signale sur- ou sous-occupation et propose 2 actions chiffrées (ex. ouvrir une salle de coworking, organiser un afterwork)

- **Export CSV/PDF** :
  ```
  @ManagerAgent Export présence mai 2025 équipe Sales
  ```

- **Note** : si la période n'est pas assez précise, l'agent demandera une date ou une plage claire

### 3.3 Office Management (@OfficeMgmtAgent)

- **Occupation par zone** :
  ```
  @OfficeMgmtAgent Occupation zone Jupiter demain
  ```

- **Commande repas** :
  ```
  @OfficeMgmtAgent Déjeuners à prévoir le 05/06/2025
  ```

- **Planning ménage** :
  ```
  @OfficeMgmtAgent Besoins ménage semaine prochaine
  ```

- **Export évacuation** :
  ```
  @OfficeMgmtAgent Export urgence évacuation
  ```

- **Alerte surcharge** : si le taux dépasse 90 % (indiqué dans la réponse), action immédiate (réserver un coworking, alerter sécurité)

### 3.4 Équipes RH (@HR_Agent)

- **Synthèse présence + engagement** :
  ```
  @HR_Agent Synthèse T2 2025
  ```

- **Taux de présence** :
  ```
  @HR_Agent Taux de présence semaine du 02/06/2025
  ```

- **Commentaires Slack** :
  - Inclus si un message dépasse 5 réactions (analysés automatiquement)

- **Actions RH** :
  - L'agent peut générer jusqu'à 23 recommandations, datées ou chiffrées (ex. organiser un atelier Bien-Être, proposer un afterwork, etc.)

---

## 4. Procédures d'urgence

### 4.1 Extraction rapide "Présents au bureau"

1. Dans Slack, tapez :
   ```
   @OfficeMgmtAgent Export urgence évacuation
   ```

2. L'agent génère un lien vers un fichier `.pdf` sécurisé (par ex. `evac_list_<date>.pdf`)
3. Téléchargez et transmettez aux équipes de sécurité ou aux autorités

### 4.2 Accès offline ou panne de bot

1. Ouvrir Google Sheets (`pulse_log`)
2. Filtrer la colonne `date` sur aujourd'hui
3. Copier la colonne `user_email`
4. Coller dans un fichier `.txt` et sauvegarder localement
5. Alerter le canal `#office-pulse-support` en précisant l'incident

### 4.3 Capacité dépassée / alerte foule

- Vérifier la capacité officielle de chaque zone (ex. "Jupiter : 50 places")
- Si le taux dépasse 90 % pendant plus de 2 jours, alerter `#facilities` et envisager :
  - Réserver un coworking proche
  - Programmer un split day (team A lundi, team B mardi)

---

## 5. FAQ – Questions courantes

| Question | Réponse |
|----------|---------|
| Comment déclarer ma venue ? | Utilisez le Shortcut "Je viens demain" dans Slack → remplissez date + zone → Google Sheets ajoute la ligne automatiquement. |
| Comment annuler ma venue ? | Utilisez le Shortcut "Annuler ma venue" → indiquez la date → Google Sheets supprime la ligne via la colonne `row_key`. |
| Le bot ne répond pas ? | 1. Vérifiez que l'app Slack est installée et que les Request URLs sont correctes (HTTPS).<br>2. Contrôlez que le token bot et le Signing Secret sont valides. |
| Comment tester Dust en local ? | 1. Lancez `npm start` dans `backend/`.<br>2. Exposez en HTTPS avec ngrok (`ngrok http 4000`).<br>3. Dans Slack, modifiez la Request URL pour pointer vers ngrok. |
| Où sont stockées les données ? | Dans Google Sheets :<br>`pulse_log` pour les présences, `notifications_sent` pour l'historique des notifications, `user_info` pour les infos statiques. |
| Puis-je ajouter d'autres commandes ? | Oui : créez un nouveau slash command (`/nouvelleCommande`) dans Slack → pointez-le vers le même Webhook Make → ajustez le prompt Dust si besoin. |
| Comment modifier un agent Dust ? | Depuis le Dashboard Dust, éditez le prompt (Contexte/Instructions/Output). |
| Le bot peut-il envoyer des DM ? | Oui, pour les notifications ciblées (ex. Guillaume Deramchi) en utilisant `conversations.open` puis `chat.postMessage`. |

---

## 6. Code source et ressources

### 6.1 Structure du dépôt Git

```
/
├─ backend/
│   ├─ src/                # Code Node.js (Express, Dust, Google Sheets)
│   ├─ dist/               # Build compilé (TypeScript → JavaScript)
│   ├─ package.json
│   ├─ .env.example        # Exemple de variables d'environnement
│   └─ README.md           # Documentation spécifique au backend
├─ workflows/
│   ├─ blueprint.json      # Workflow Make (à copier-coller)
│   ├─ README.md           # Notes complémentaires sur le workflow
│   └─ assets/             # Éventuels visuels / diagrammes JSON
├─ docs/                   # Documentation supplémentaire (schémas, diagrammes)
├─ README.md               # Ce fichier principal
└─ .gitignore
```

### 6.2 Dépendances & prérequis techniques

- **Node.js** ≥ 16.x (Express, @slack/web-api, googleapis, dotenv)
- **Make** (Integromat) : compte actif pour importer `blueprint.json`
- **Dust** : Workspace configuré avec agents et API Key
- **ngrok** (ou équivalent) pour le développement local HTTPS
- **Google Cloud Service Account** : accès à Google Sheets (`pulse_log`, `notifications_sent`, `user_info`)
- **Workflow Builder Slack** (intégration "Google Sheets for Workflow Builder") pour les formulaires

### 6.3 Ressources graphiques et assets

- Diagrammes Mermaid intégrés dans la doc (README)
- Captures d'écran de configuration Slack & Make (répertoire `docs/screenshots/`)
- Fichier `schéma_architecture.drawio` (ou équivalent) pour l'architecture détaillée
- **Looker Studio** : lien vers le rapport (partagé en lecture seule), templates JSON si export configuré

### 6.4 Guide d'installation détaillé

Le guide d'installation détaillé se trouve dans `backend/README.md` (inclut l'installation Docker, systemd, PaaS, variables d'environnement).

---

## 7. Liens utiles

1. [Slack – Slash Commands](https://api.slack.com/slash-commands)
2. [Slack – OAuth & Permissions](https://api.slack.com/authentication/oauth-v2)
3. [Slack – Incoming Webhooks](https://api.slack.com/incoming-webhooks)
4. [Slack – Interactivity & Shortcuts](https://api.slack.com/interactivity/handling)
5. [Dust Documentation](https://docs.dust.tt/)
6. [Make – Google Sheets Modules](https://www.make.com/en/help/googlesheets)
7. [Looker Studio](https://lookerstudio.google.com/)
8. [StackOverflow – timeout Slack command](https://stackoverflow.com/questions/34896954/how-to-avoid-slack-command-timeout-error)
9. [Make – Scheduler trigger](https://www.make.com/en/help/scheduler)
10. [Make – Slack Create a message](https://www.make.com/en/help/slack)
