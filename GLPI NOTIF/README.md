# 🚀 GLPI NOTIF

> **Un ticket GLPI est créé → tout le monde le sait en moins d'une minute.**
> WhatsApp Direction + WhatsApp Support + webhook externe, jamais de doublon, jamais d'échec silencieux.

## Fonctionnalités

- **Détection temps réel** : les nouveaux tickets GLPI (statut NEW) sont relevés toutes les minutes, sur une fenêtre glissante de 24 h
- **Triple notification** : WhatsApp Direction (+261 32 398 488), WhatsApp Support (+261 330 531 966) et POST webhook externe (JSON toujours valide)
- **Anti-doublon persistant** : chaque ticket est marqué dans la table de suivi, marquage idempotent — aucun envoi répété
- **Message propre** : HTML décodé et supprimé, espaces normalisés, description tronquée à 3 000 caractères, message construit une seule fois
- **Résilient** : 3 tentatives automatiques sur chaque sortie, et toute erreur fait échouer l'exécution de façon visible (nœud `6 Échec`) — jamais d'échec silencieux
- **Sécurisé** : la clé API du webhook est gérée par le système de credentials n8n, pas dans le workflow

## Le flux

```
⏱ Cron 1 min
   │
   ├─ 2.1  Tickets GLPI (statut NEW, dernières 24 h)   ← MySQL GLPI
   ├─ 2.2  IDs déjà notifiés                            ← MySQL suivi (61.4)
   │
   └─ 3  Préparer notifications  ← filtre les nouveaux, nettoie le HTML,
        │                           construit le message UNE seule fois
        ├─ 4.1  WhatsApp Direction  (+261 32 398 488)
        ├─ 4.2  WhatsApp Support    (+261 330 531 966)
        ├─ 4.3  Webhook HTTP        (POST 192.168.60.253:8080)
        └─ 5    Marquer comme notifié (anti-doublon, idempotent)

        ⤷ toute erreur → 6 Échec → exécution en erreur visible dans n8n
```

## Import en 3 minutes

1. **Importer** : n8n → *Workflows* → ⋯ → *Import from File* → `GLPI NOTIF.json`
2. **Créer la credential webhook** : *Credentials* → *New* → **Header Auth** →
   Name : `GLPI Webhook X-API-Key` · Header Name : `X-API-Key` · Valeur : votre clé,
   puis la sélectionner sur le nœud `4.3 Webhook HTTP`.
3. **Vérifier** les credentials MySQL / WhatsApp (réutilisées automatiquement sur la même instance), puis **activer**.

## 🔧 Recommandations

- **Index SQL** (la requête tourne toutes les minutes) :
  ```sql
  ALTER TABLE glpi_tickets ADD INDEX idx_status_date (status, date);
  ```
- **Error workflow** : *Settings → Error Workflow* d'une seconde workflow Error Trigger
  pour être alerté ailleurs que dans le tableau des exécutions (réglage UI, pas automatisable).
- **Sémantique** : « au moins une fois ». Si un envoi échoue après 3 tentatives, l'exécution
  échoue bruyamment et le ticket n'est **pas** marqué → il sera renvoyé à la minute suivante
  (possibilité de doublon sur les canaux qui avaient réussi). Jamais de perte silencieuse.

## ⚠️ Avant d'activer

Testez d'abord avec une exécution manuelle : les nœuds WhatsApp et le webhook envoient
**de vrais messages**.
