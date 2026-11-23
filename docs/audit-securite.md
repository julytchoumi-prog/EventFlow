# Rapport d'audit de sécurité – EventFlow

**Date :** 23 novembre 2026
**Projet :** EventFlow
**Branche :** mission-lundi
**Type d'audit :** Audit de sécurité DevSecOps
**Périmètre :** Git, dépendances Python, code Python et pipeline CI/CD.

---

## 1. Contexte et objectifs

Dans le cadre de la mission QA EventFlow, un audit de sécurité a été réalisé afin d'identifier les faiblesses potentielles du projet.

Les objectifs sont les suivants :

- Détecter les secrets exposés dans le dépôt Git.
- Identifier les dépendances Python présentant des vulnérabilités connues.
- Rechercher les pratiques de programmation potentiellement dangereuses.
- Vérifier la sécurité des workflows GitHub Actions.
- Proposer des mesures correctives adaptées aux risques identifiés.

## 2. Méthodologie et outils utilisés

Quatre outils ont été utilisés.

| Outil | Objectif |
|---|---|
| Gitleaks | Détecter les secrets exposés dans les fichiers et l'historique Git |
| pip-audit | Identifier les vulnérabilités connues dans les dépendances Python |
| Bandit | Analyser statiquement le code Python |
| Zizmor | Auditer les workflows GitHub Actions |

Les analyses ont été réalisées le 23 novembre 2026.

## 3. Synthèse des résultats

### 3.1 Gitleaks

Une véritable clé API a été détectée dans l'historique Git.

- Règle : `generic-api-key`
- Fichier historique : `qa/utils/xray_export.py`
- Ligne : 11
- Commit : `cc42802`
- Service identifié : Xray Cloud.
- Le fichier a été supprimé dans un commit ultérieur, mais le commit initial reste présent dans plusieurs branches distantes.

**Statut :** incident identifié, révocation en attente de l'intervention du propriétaire de la clé.

### 3.2 pip-audit

Résultat : 6 occurrences de vulnérabilités signalées dans un package, correspondant à 3 identifiants distincts.

| Package | Version | Identifiant | Version corrective indiquée |
|---|---|---|---|
| requests | 2.31.0 | PYSEC-2026-1873 | 2.32.0 |
| requests | 2.31.0 | PYSEC-2026-1872 | 2.32.4 |
| requests | 2.31.0 | PYSEC-2026-2275 | 2.33.0 |

**Statut :** analyse effectuée, mise à jour et vérification à réaliser.

### 3.3 Bandit

Résultats de l'analyse :

| Gravité | Nombre |
|---|---:|
| High | 1 |
| Medium | 9 |
| Low | 98 |
| Undefined | 0 |
| **Total** | **108** |

Les principales alertes examinées sont :

- B501 : désactivation de la vérification SSL.
- B113 : appels HTTP sans timeout.
- B608 : construction SQL par interpolation de chaîne.

**Statut :** analyse effectuée, constats prioritaires identifiés.

### 3.4 Zizmor

Résultat : 12 findings, dont 6 suppressed.

Parmi les alertes affichées :

| Gravité | Nombre |
|---|---:|
| High | 4 |
| Medium | 1 |
| Low | 1 |

Les problèmes identifiés concernent :

- `template-injection` : injection potentielle dans une commande shell.
- `excessive-permissions` : permissions trop larges.
- `unpinned-uses` : actions GitHub non épinglées par SHA.
- `artipacked` : persistance des identifiants Git.

**Statut :** analyse effectuée, mesures de durcissement à prévoir.

## 4. Constats détaillés

### SEC-01 – Exposition d'une clé API Xray Cloud

**Outil :** Gitleaks
**Gravité :** Critique
**Fichier :** `qa/utils/xray_export.py` (historique Git)
**Commit :** `cc42802`

Une véritable clé API a été détectée dans l'historique Git.

Bien que le fichier ait été supprimé par la suite, son ancien contenu reste accessible dans l'historique des branches distantes.

**Risque :** utilisation non autorisée des permissions associées à la clé.

**Recommandations :**
- Identifier le propriétaire de la clé.
- Révoquer immédiatement la clé compromise.
- Utiliser des variables d'environnement ou des secrets sécurisés.
- Nettoyer l'historique Git après coordination avec le formateur.
- Relancer Gitleaks après correction.

**État :** en attente de révocation.

### SEC-02 – Dépendance requests présentant des vulnérabilités

**Outil :** pip-audit
**Gravité :** À évaluer selon les avis de sécurité
**Package :** requests 2.31.0

Trois identifiants de vulnérabilité distincts ont été détectés.

**Risque :** exposition à des vulnérabilités connues de la bibliothèque HTTP.

**Recommandations :**
- Étudier les avis de sécurité correspondants.
- Mettre à jour vers une version corrigée compatible.
- Relancer pip-audit après la mise à jour.

**État :** correction à effectuer.

### SEC-03 – Désactivation de la vérification SSL

**Outil :** Bandit – B501
**Gravité :** High
**Fichier :** `qa/utils/api_client.py`
**Ligne :** 22

L'appel HTTP utilise `verify=False`, ce qui désactive la vérification du certificat SSL/TLS.

**Risque :** exposition potentielle aux attaques de type Man-in-the-Middle.

**Recommandations :**
- Supprimer `verify=False`.
- Conserver la vérification SSL par défaut.
- Configurer correctement les certificats si nécessaire.

**État :** constat confirmé dans le code, correction à effectuer.

### SEC-04 – Absence de timeout sur les requêtes HTTP

**Outil :** Bandit – B113
**Gravité :** Medium
**Fichiers :** `qa/tests/api/test_promo.py` et `qa/utils/api_client.py`

Plusieurs appels HTTP ne définissent pas de délai maximal d'attente.

**Risque :** blocage prolongé des requêtes et des tests, avec une consommation inutile des ressources.

**Recommandations :**
- Ajouter un paramètre `timeout` adapté aux appels HTTP.
- Vérifier le comportement en cas de dépassement du délai.

**État :** amélioration de robustesse à effectuer.

### SEC-05 – Construction SQL par interpolation

**Outil :** Bandit – B608
**Gravité :** Medium
**Fichier :** `qa/utils/db.py`
**Ligne :** 29

La fonction `order_payment()` construit une requête SQL avec une f-string et insère directement la valeur `order_id`.

**Risque :** injection SQL potentielle si la valeur provient d'une source non fiable.

**Recommandations :**
- Utiliser des paramètres SQL nommés.
- Transmettre les valeurs séparément à SQLAlchemy.
- Vérifier les données transmises à la requête.

**État :** construction dangereuse confirmée, exploitabilité non démontrée.

### SEC-06 – Injection potentielle dans GitHub Actions

**Outil :** Zizmor – template-injection
**Gravité :** High
**Fichier :** `.github/workflows/ci.yml`
**Ligne :** 23

Une expression GitHub `${{ github.event.inputs.seed_bugs }}` est directement intégrée dans un script shell.

**Risque :** injection de commandes si l'entrée est contrôlable par un acteur malveillant.

**Recommandations :**
- Transmettre la valeur par une variable d'environnement.
- Utiliser cette variable dans le script shell.
- Éviter l'expansion directe d'entrées non fiables dans les commandes.

**État :** correction à effectuer.

### SEC-07 – Permissions GitHub Actions trop larges

**Outil :** Zizmor – excessive-permissions
**Gravité :** Medium
**Fichier :** `.github/workflows/ci.yml`

Le job utilise les permissions par défaut sans bloc `permissions` explicite.

**Risque :** attribution de privilèges supérieurs aux besoins du job.

**Recommandations :**
- Définir explicitement les permissions.
- Appliquer le principe du moindre privilège.
- Accorder uniquement les permissions nécessaires.

**État :** correction à effectuer.

### SEC-08 – Actions GitHub non épinglées par SHA

**Outil :** Zizmor – unpinned-uses
**Gravité :** High selon l'outil
**Fichier :** `.github/workflows/ci.yml`

Les actions suivantes sont référencées par tag :

- `actions/checkout@v7`
- `actions/setup-python@v7`
- `actions/upload-artifact@v7`

**Risque :** dépendance à des références qui ne garantissent pas l'intégrité immuable du code exécuté.

**Recommandations :**
- Épingler les actions sur leur SHA Git complet.
- Ajouter un commentaire indiquant la version correspondante.
- Prévoir une procédure de mise à jour des SHA.

**État :** correction à effectuer.

### SEC-09 – Persistance des identifiants Git

**Outil :** Zizmor – artipacked
**Gravité :** Low selon l'outil
**Fichier :** `.github/workflows/ci.yml`

L'étape `actions/checkout@v7` ne définit pas `persist-credentials: false`.

**Risque :** persistance du token Git dans la configuration locale du dépôt pendant le job.

**Recommandations :**
- Désactiver la persistance si le workflow n'a pas besoin d'écrire dans le dépôt.

**État :** correction à évaluer.

## 5. Plan de remédiation

| Priorité | Action | État |
|---|---|---|
| P1 | Révoquer la clé API Xray | En attente |
| P1 | Supprimer `verify=False` | À faire |
| P1 | Corriger l'injection potentielle du workflow | À faire |
| P1 | Épingler les actions GitHub par SHA | À faire |
| P2 | Corriger la construction SQL | À faire |
| P2 | Mettre à jour requests | À faire |
| P2 | Restreindre les permissions GitHub Actions | À faire |
| P3 | Ajouter les timeouts HTTP | À faire |
| P3 | Désactiver la persistance des identifiants | À évaluer |

## 6. Plan de réponse aux incidents

En cas de découverte d'un secret exposé :

1. Identifier le type de secret et le service concerné.
2. Déterminer son emplacement et son historique d'exposition.
3. Identifier le propriétaire du secret.
4. Révoquer ou désactiver le secret compromis.
5. Générer un nouveau secret si nécessaire.
6. Le stocker dans un gestionnaire de secrets sécurisé.
7. Nettoyer l'historique Git après coordination.
8. Relancer Gitleaks pour vérifier le résultat.
9. Documenter l'incident et les mesures correctives.

Dans le cas de la clé Xray Cloud, la révocation est en attente de l'intervention du propriétaire.

## 7. Faux positifs et limites de l'analyse

Les outils d'analyse statique peuvent générer des alertes nécessitant une vérification humaine.

- Bandit B113 présente une confiance faible : les appels concernés doivent être examinés dans leur contexte.
- Bandit B608 présente une confiance faible : le risque de construction SQL est confirmé, mais aucune exploitation effective n'a été démontrée.
- Les findings Zizmor supprimés (suppressed) ne sont pas considérés comme des vulnérabilités confirmées sans examen complémentaire.
- Une détection Gitleaks doit être vérifiée pour distinguer une véritable clé d'un exemple fictif. Dans notre cas, la clé a été confirmée comme réelle.
- Les résultats d'un outil statique ne démontrent pas, à eux seuls, qu'une vulnérabilité a été exploitée.

## Conclusion

L'audit a permis d'identifier plusieurs faiblesses dans les secrets, les dépendances, le code Python et le pipeline CI/CD d'EventFlow.

Les priorités sont la révocation de la clé API exposée, la sécurisation des communications HTTP, la correction de la construction SQL et le renforcement du workflow GitHub Actions.

Les corrections devront être suivies de nouvelles analyses afin de vérifier leur efficacité.
