# Guide Utilisateur - Solution Slack & Dust

## 📋 Vue d'ensemble

Notre solution intègre Slack et Dust pour créer un système de gestion intelligent du bureau avec un chatbot spécialisé disponible 24h/7j. Elle comprend :

- **Agents IA spécialisés** pour différents métiers
- **Système de présence** automatisé  
- **Notifications intelligentes** personnalisées
- **Tableau de bord** en temps réel

## 🚀 Installation et Configuration

### 1. Configuration du Workflow

Le workflow complet est disponible dans le fichier [`blueprint.json`](blueprint.json).

**Architecture technique :**
- Webhook en trigger
- Bypass du timeout Slack avec un webhook secondaire
- Intégration avec l'agent Dust
- Retour de réponse sur le canal Slack

### 2. Configuration Slack API

1. Créer un projet sur [Slack API](https://api.slack.com/apps)
2. Ajouter les autorisations nécessaires :
   - Webhooks
   - Accès aux messages
   - Bot tokens
3. Créer les slash commands
4. Configurer l'URL HTTPS pour la connexion

### 3. Configuration Dust

- Paramétrer les agents avec des prompts détaillés
- Utiliser Claude 4 en mode déterministe
- Connecter les onglets Google Sheets

## 🤖 Commandes Slash Disponibles

| Commande | Description |
|----------|-------------|
| `/pulse` | **Contrôle de présence** - Savoir qui est au bureau |
| `/hr` | **Ressources Humaines** - Questions RH et enjeux du brief |
| `/om` | **Office Management** - Gestion des bureaux |
| `/management` | **Management** - Enjeux de management |

## 📊 Fonctionnalités

### Système de Présence
- **Messages quotidiens** automatiques dans un canal dédié
- **Formulaire simple** : date et zone à remplir
- **Annulation** possible en citant simplement la date
- **Automation complète** de la gestion des données

### Notifications Intelligentes
Exemple avec Guillaume Deramchi :
- Analyse de la présence de son équipe
- Notification la veille si un membre de l'équipe vient au bureau
- Message personnalisé encourageant la venue

### Visualisation des Données
- **Looker Studio** connecté à la base de données
- **Interface graphique** en temps réel
- **Visualisation** des statistiques de présence

## 💡 Résumé des Capacités

✅ **Chatbot ultra-complet** avec commandes spécialisées par métier  
✅ **Système de présence** automatisé et intelligent  
✅ **Notifications intelligentes** personnalisées  
✅ **Tableau de bord** Looker Studio pour la visualisation  
✅ **Intégration native** Slack pour une expérience fluide