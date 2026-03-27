# 🛡️ Rapport d'Audit de Sécurité - Application "Manipulate Me"

**Date de l'audit** : 05 Février 2026
**Cible** : `app.py` (Streamlit Web App)
**Auditeur** : Agent AntiGravity (Rôle: Senior Cybersecurity Auditor)

---

## 1. Synthèse Managériale

### Note de Sécurité Globale : **D+**

L'application est fonctionnelle pour un prototype éducatif (Proof of Concept), mais elle présente des défauts structurels majeurs qui la rendraient impropre à un déploiement en production réelle. Si l'objectif pédagogique (démontrer la vulnérabilité aux Prompt Injections) est atteint, les pratiques de développement sécurisé (DevSecOps) sont absentes sur la couche application.

### Résumé des Risques Principaux
1.  **Exposition des Secrets** : Les mots de passe du jeu sont écrits "en dur" dans le code source (`hardcoded`), ce qui est une faille critique si le code est partagé.
2.  **Absence de Gestion de Session Sécurisée** : L'application efface tout contexte au rechargement (F5), et l'absence de persistance ou de logs limite toute investigation post-incident, bien que cela protège paradoxalement la confidentialité à long terme.
3.  **Dépendance Critique API** : L'application n'a aucune résilience face aux pannes ou limites de l'API Groq (pas de mécanisme de `backoff` ou de gestion fine des erreurs), menaçant la disponibilité.

---

## 2. Analyse des Risques (Modèle DIC)

Cette section analyse l'application selon les trois piliers de la sécurité de l'information.

### 🛑 Disponibilité (Availablity)
*   **Risque : Élevé**. L'application dépend entièrement d'une API tierce (Groq). Une coupure réseau, une clé invalide, ou un dépassement de quota (Rate Limiting) rend l'application immédiatement inutilisable.
*   **Implémentation actuelle** : Un simple bloc `try/except` capture les erreurs de manière générique (`st.error(f"Erreur API : {e}")`). C'est insuffisant pour une expérience utilisateur robuste.

### ⚠️ Intégrité (Integrity)
*   **Risque : Critique (Intentionnel)**. Le cœur du jeu repose sur la *perte d'intégrité* du modèle IA via Prompt Injection. L'utilisateur peut forcer l'IA à dévier de son but initial (garder le secret).
*   **Risque Structurel** : Il n'y a aucune validation des entrées utilisateur (`Input Validation`). Un utilisateur malveillant pourrait théoriquement tenter d'injecter non pas du prompt, mais du code HTML/JS malveillant (bien que Streamlit filtre une partie, le risque de Markdown Injection persiste).

### 🔒 Confidentialité (Confidentiality)
*   **Risque : Moyen**.
    *   **Points Positifs** : La clé API est masquée (`type="password"`) et n'est stockée que dans la RAM (Session State) le temps de l'utilisation.
    *   **Points Négatifs** : Les conversations sont envoyées à un tiers (Groq) sans avertissement explicite de confidentialité (GDPR/RGPD). Les secrets du jeu sont visibles par quiconque a accès au fichier `.py`.

---

## 3. Liste des Vulnérabilités Techniques

| ID | Sévérité | Vulnérabilité | Description Technique |
| :--- | :--- | :--- | :--- |
| **VULN-001** | 🔴 Critique | **Hardcoded Secrets** | Les variables `SECRET_LEVEL_1`, `2`, `3` sont définies en clair dans `app.py`. Tout accès au script révèle les réponses du jeu. |
| **VULN-002** | 🟠 Élevée | **Lack of Secret Management** | La clé API est demandée à chaque session utilisateur via l'UI, ce qui encourage le copier-coller (risque de fuite presse-papier/épaule). Elle devrait être gérée via des variables d'environnement serveur. |
| **VULN-003** | 🟡 Moyenne | **Unsanitized Inputs** | La variable `prompt := st.chat_input` est envoyée brute à l'API. Aucune limite de longueur (risque de déni de service par coût/tokens) ni filtre de contenu. |
| **VULN-004** | 🟡 Moyenne | **Information Disclosure** | Les erreurs Python complètes (` Exception as e`) sont affichées à l'utilisateur via `st.error(e)`, pouvant révéler des détails d'implémentation de la stack. |
| **VULN-005** | 🔵 Faible | **No Audit Logging** | Aucune trace des tentatives d'injection n'est conservée. Impossible de rejouer une attaque pour l'analyser a posteriori. |

---

## 4. Recommandations de Correction

Voici les actions correctives prioritaires pour transformer ce prototype en application sécurisée.

### A. Gestion des Secrets (Priorité 1)
Ne jamais stocker de secrets dans le code. Utilisez `st.secrets` de Streamlit ou des variables d'environnement (`.env`).

**Correction proposée :**
1. Créer un fichier `.streamlit/secrets.toml` (non commité sur Git).
2. Modifier le code pour lire depuis ce fichier.

```python
# .streamlit/secrets.toml
[game]
level_1 = "STARTUP_123"
level_2 = "OMEGA"
level_3 = "ILOVEMYINCUBATOR"

# app.py
SECRET_LEVEL_1 = st.secrets["game"]["level_1"]
# ...
```

### B. Gestion des Clés API (Priorité 2)
Au lieu de demander la clé à l'utilisateur (sauf si c'est un choix pédagogique assume), chargez-la depuis l'environnement serveur.

```python
# app.py
import os
from dotenv import load_dotenv

load_dotenv() # Pour le local
api_key = os.getenv("GROQ_API_KEY")

if not api_key:
    # Fallback sur l'input utilisateur seulement si la var d'env manque
    api_key = st.text_input("Clé API", type="password")
```

### C. Validation des Inputs (Priorité 3)
Restreindre la taille des messages pour éviter les abus de l'API.

```python
# app.py
MAX_CHARS = 500
if prompt := st.chat_input(...):
    if len(prompt) > MAX_CHARS:
        st.warning(f"Message trop long ! Limite : {MAX_CHARS} caractères.")
        st.stop()
    # ... suite du code
```
