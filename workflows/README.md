# 🚀 GLPI NOTIF — version optimisée

> **Un ticket GLPI est créé → tout le monde le sait en moins d'une minute.**
> WhatsApp Direction + WhatsApp Support + webhook externe, jamais de doublon, jamais d'échec silencieux.

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

## Ce qui a changé (avant ➜ après)

| Avant | Après |
|---|---|
| 2 nœuds Code (`$items()` obsolète) + nettoyage HTML dupliqué 3 fois | **1 seul nœuds Code** : filtre + nettoyage + message construit une fois |
| `DATE(t.date) = CURDATE()` → index inutilisable, ticket de 23 h55 perdu à minuit | `t.date >= NOW() - INTERVAL 24 HOUR` → **sargable**, couvre la minuit |
| Clé `X-API-Key` **en clair** dans le nœud HTTP | Credential n8n `httpHeaderAuth` 🔒 |
| Corps JSON cassé si la description contient `"` ou un saut de ligne | `JSON.stringify()` → corps **toujours valide** |
| Aucun retry, échec = workflow à l'arrêt ou perte silencieuse | **3 retries** sur chaque I/O + sortie d'erreur câblée → nœud `6 Échec` |
| Insert en échouant sur doublon = spam WhatsApp à chaque minute | `skipOnConflict` → marquage **idempotent** |
| `SELECT * FROM glpi WHERE 1` | `SELECT id FROM glpi` |

## Import en 3 minutes

1. **Importer** : n8n → *Workflows* → ⋯ → *Import from File* → `glpi-notif-optimise.json`
2. **Créer la credential webhook** : *Credentials* → *New* → **Header Auth** →
   Name : `GLPI Webhook X-API-Key` · Header Name : `X-API-Key` · Valeur : votre clé,
   puis la sélectionner sur le nœud `4.3 Webhook HTTP` (seul avertissement de validation).
3. **Vérifier** les credentials MySQL / WhatsApp (réutilisées automatiquement sur la même instance), puis **activer**.

## 🔧 Recommandations

- **Index SQL** (la requête tourne toutes les minutes) :
  ```sql
  ALTER TABLE glpi_tickets ADD INDEX idx_status_date (status, date);
  ```
- **Error workflow** : *Settings → Error Workflow* d'une seconde workflow Error Trigger
  pour être alerté ailleurs que dans le tableau des exécutions (MCP ne peut pas le faire, c'est un réglage UI).
- **Sémantique** : « au moins une fois ». Si un envoi échoue après 3 retries, l'exécution
  échoue bruyamment et le ticket n'est **pas** marqué → il sera renvoyé à la minute suivante
  (possibilité de doublon sur les canaux qui avaient réussi). Jamais de perte silencieuse.

## ⚠️ Avant d'activer

Testez d'abord avec une exécution manuelle : les nœuds WhatsApp et le webhook envoient
**de vrais messages**. Je peux déployer et tester cette version sur votre instance n8n si vous le souhaitez.
