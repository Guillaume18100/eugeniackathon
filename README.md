# Solution de Gestion de Présence Bureau avec Slack, Make et Dust

Cette solution combine un workflow Make (fichier JSON à copier-coller), un webhook Slack en trigger principal, un webhook secondaire pour bypasser le timeout Slack, et un agent Dust (Claude 4 en mode déterministe) connecté à des onglets Google Sheets. Quatre commandes slash (/pulse, /hr, /om, /management) permettent d'interroger directement l'agent Dust depuis Slack, qui puise dans les données de pulse_log et renvoie la réponse dans Slack. Un workflow Form Slack gère la déclaration / annulation des présences, et un tableau Looker Studio affiche en temps réel les statistiques de fréquentation.

---

## Guide Utilisateur

### 1. Import et configuration du workflow Make

1. Ouvrir Make et créer un nouveau scénario.
2. Copier-coller le JSON contenu dans `blueprint.json` (workflow Make prêt à l'emploi).
3. Ce workflow Make est structuré ainsi :
   - **Webhook Trigger** (Make écoute une requête POST)
   - **Bypass du timeout Slack** : réception immédiate de la slash command → réponse 200 OK → Make continue de traiter en arrière-plan pour interroger l'agent Dust.
   - **Module "Dust"** : appel à l'agent spécialisé (prompt paramétré, modèle Claude 4 déterministe, RAG sur les onglets Google Sheets).
   - **Module "Slack – Create a message"** : envoie du résultat de Dust dans Slack (canal public ou DM).

### 2. Création de l'application Slack

1. Allez sur [Slack API](https://api.slack.com/apps) et cliquez sur **Create App** → **From scratch**.
2. Dans **OAuth & Permissions**, ajoutez au bot token les scopes suivants :
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
   - **Command** : `/pulse`
   - **Request URL** : collez l'URL HTTPS du Webhook Trigger dans Make (ex. `https://hook.make.com/abcdef123456`).
   - Répétez pour `/hr`, `/om` et `/management`, en pointant vers le même Webhook si Make différencie via le paramètre command.
4. Cliquez sur **Install to Workspace** et autorisez les scopes.

### 3. Paramétrage de Dust

1. Créer vos agents dans Dust (Dashboard Dust) :
   - Un agent "Collaborateurs" (prompt basique).
   - Un agent "PEAgent" pour RH.
   - Un agent "ManagerAgent" pour management.
   - Un agent "OfficeMgmtAgent" pour office management.
   - Un agent "NotificationAgent" pour notifications ciblées (Guillaume Deramchi).
2. **Modèle** : sélectionnez Claude 4 en mode déterministe pour garantir des réponses stables.
3. **RAG Configuration** : connectez les onglets Google Sheets (pulse_log, notifications_sent, user_info) pour que l'agent Dust puisse récupérer en temps réel :
   - Qui vient au bureau (pulse_log).
   - Qui a déjà été notifié (notifications_sent).
   - Équipe, centres d'intérêt, localisation (user_info).
4. **Prompts** : rédigez un prompt CIOE (Contexte/Instructions/Output/Exemple) pour chaque agent, en explicitant la logique de filtrage sur les onglets Sheets.

### 4. Utilisation des commandes Slash

#### `/pulse` – Contrôle de présence
Permet de lister qui est ou n'est pas au bureau pour un jour donné.

**Exemple :**
```
/pulse qui sera au bureau demain ?
```

Make reçoit la request, répond 200 immédiatement, appelle l'agent "Collaborateurs" Dust qui lit pulse_log pour date = demain et envoie la liste dans Slack.

#### `/hr` – Ressources Humaines
Pour poser des questions RH (taux d'engagement, équipe faible, recommandations…)

**Exemple :**
```
/hr quelle équipe a eu le taux de présence le plus bas la semaine dernière ?
```

Dust lit pulse_log sur la période, agrège par équipe, renvoie la réponse.

#### `/om` – Office Management
Pour gérer occupation, ménage, repas, sécurité…

**Exemple :**
```
/om occupation zone Jupiter demain
```

Dust lit pulse_log, room_bookings, office_feedback, calcule le taux d'occupation, renvoie un résumé.

#### `/management` – Management
Pour disponibilité d'équipe, suivi de projets, performance…

**Exemple :**
```
/management qui de l'équipe Dev est disponible demain matin ?
```

Dust croise pulse_log et éventuellement project_tracker, performance_metrics, renvoie la liste.

### 5. Workflow Form Slack (Déclaration / Annulation)

1. Dans Slack → **Workflow Builder** → **Create** → **From scratch**.
2. **Trigger** : choisissez Shortcut (par ex. "Je viens demain").
3. **Formulaire** : deux champs :
   - Date (jj/mm/aaaa)
   - Zone (ex. "Paris", "Lyon", "Toulouse")
4. **Étape Google Sheets – Add Row** : enregistre dans pulse_log (user_id, nom, date, zone, row_key = date_userID).
5. Pour l'annulation, créez un Shortcut "Annuler ma venue" avec un formulaire demandant uniquement la Date.
   - **Étape Google Sheets – Delete a spreadsheet row** : supprime la ligne correspondant à row_key = date_userID.
6. À chaque soumission, Slack notifie l'utilisateur par message "Ta venue a bien été enregistrée/annulée."

### 6. Notifications intelligentes (exemple Guillaume Deramchi)

1. **Déclencheur Scheduler** dans Make → 18 h (Europe/Paris).
2. **Google Sheets – Get rows** sur pulse_log pour date = demain.
3. **Filtrer** : si user_id = U_GDeramchi est absent et si l'un des equipe = Dev vient, sélectionner le collègue partageant un intérêt ou la même zone.
4. **Search rows** sur notifications_sent pour éviter le doublon le même jour.
5. **Slack – Create a message** : envoie un DM à U_GDeramchi :

   > Salut Guillaume ! Demain, Lucie Martin (Dev – passionnée de lecture) sera au bureau. Ce serait sympa de te joindre à elle pour échanger !

   En utilisant conversations.open puis chat.postMessage.

6. **Google Sheets – Add row** dans notifications_sent (date, "U_GDeramchi", userB_id, "collègue_équipe").

### 7. Tableau de bord Looker Studio

1. Connecter Looker Studio à Google Sheets (pulse_log).
2. Créer un rapport affichant :
   - Taux d'occupation journaliers (%)
   - Répartition par équipe et zone
   - Historique mensuel
3. Configurer une actualisation automatique (toutes les heures).
4. Partager en lecture seule avec les RH, managers, et office management.

---

## Liens utiles

1. [Slack – Slash Commands](https://docs.slack.dev/bolt-js/concepts#commands)
2. [Slack – OAuth & Permissions](https://api.slack.com/authentication/oauth-v2)
3. [Slack – Incoming Webhooks](https://api.slack.com/messaging/webhooks)
4. [Slack – Interactivity & Shortcuts](https://api.slack.com/interactivity/handling)
5. [Dust Documentation](https://docs.dust.tt/)
6. [Make – Google Sheets Modules](https://www.make.com/en/help/app/google-sheets)
7. [Looker Studio](https://lookerstudio.google.com/)
8. [StackOverflow – timeout Slack command](https://stackoverflow.com/questions/34896954/how-to-avoid-slack-command-timeout-error)
9. [Make – Scheduler trigger](https://www.make.com/en/help/tools/schedule)
10. [Make – Slack Create a message](https://www.make.com/en/help/app/slack)
