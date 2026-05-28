# ⚡ Parallélisme Maximal — Système de Tâches

> Projet académique — Implémentation en Python d'un moteur d'exécution de tâches avec **parallélisme maximal automatique**, basé sur les conditions de Bernstein et un graphe de précédence.

---

## Contexte

Projet réalisé dans le cadre d'un cours de **programmation concurrente et parallèle**.  
L'objectif est d'implémenter un système capable d'exécuter un ensemble de tâches avec dépendances en maximisant le parallélisme, c'est-à-dire en supprimant automatiquement les contraintes de précédence inutiles grâce aux **conditions de Bernstein**.

> Fork du projet original : [chaoukiksr/maximal_parallelism](https://github.com/chaoukiksr/maximal_parallelism)

---

## Table des matières

- [Principe](#principe)
- [Conditions de Bernstein](#conditions-de-bernstein)
- [Architecture](#architecture)
- [Fonctionnalités](#fonctionnalités)
- [Exemple — Pipeline de commande](#exemple--pipeline-de-commande)
- [Installation & utilisation](#installation--utilisation)
- [Structure des fichiers](#structure-des-fichiers)

---

## Principe

Un **système de tâches** est un ensemble de tâches `T1, T2, …, Tn` avec :
- des **variables lues** (`reads`) et **variables écrites** (`writes`) par chaque tâche
- un **graphe de précédence** définissant l'ordre d'exécution obligatoire

Le moteur analyse automatiquement ce graphe et supprime les contraintes inutiles entre deux tâches indépendantes (au sens de Bernstein), permettant de les exécuter **en parallèle via des threads Python**.

```
Graphe initial (manuel)       →      Graphe optimisé (maxParallel)
T1 → T4                               T1 → T4  (lecture/écriture partagée : conservée)
T1 → T7  (inutile)            →       T7 libérée  (aucun conflit détecté)
```

---

## Conditions de Bernstein

Deux tâches `Ti` et `Tj` peuvent s'exécuter en parallèle si et seulement si leurs accès aux variables ne créent aucun conflit :

| Conflit | Condition | Description |
|---|---|---|
| **RAW** | `reads(Ti) ∩ writes(Tj) ≠ ∅` | Tj écrit ce que Ti lit |
| **WAR** | `writes(Ti) ∩ reads(Tj) ≠ ∅` | Ti écrit ce que Tj lit |
| **WAW** | `writes(Ti) ∩ writes(Tj) ≠ ∅` | Les deux écrivent la même variable |

Si aucun de ces conflits n'existe, la précédence entre `Ti` et `Tj` est **supprimée**.

```python
def bernstein(self, Ti, Tj):
    RAW = set(Ti.reads) & set(Tj.writes)
    WAR = set(Ti.writes) & set(Tj.reads)
    WAW = set(Ti.writes) & set(Tj.writes)
    return not (RAW or WAR or WAW)
```

---

## Architecture

```
maxpar.py          ← Moteur principal
├── Task           ← Représente une tâche (nom, reads, writes, run)
├── TaskSystem     ← Gestionnaire du système de tâches
│   ├── bernstein()     ← Test d'indépendance entre deux tâches
│   ├── maxParallel()   ← Supprime les précédences inutiles
│   ├── runSeq()        ← Exécution séquentielle (topological sort)
│   ├── run()           ← Exécution parallèle (threading)
│   ├── getDependencies() ← Retourne les dépendances d'une tâche
│   └── draw()          ← Visualisation du graphe (networkx + matplotlib)
└── validate()     ← Vérifie noms uniques, dépendances valides, absence de cycle

text.py            ← Exemple : pipeline de traitement d'une commande e-commerce
```

---

## Fonctionnalités

| Méthode | Description |
|---|---|
| `validate()` | Vérifie l'intégrité du système : noms uniques, tâches inconnues, cycles |
| `bernstein(Ti, Tj)` | Teste si deux tâches sont indépendantes (RAW / WAR / WAW) |
| `maxParallel()` | Réduit le graphe de précédence au minimum nécessaire |
| `runSeq()` | Exécute les tâches en ordre topologique (séquentiel) |
| `run()` | Exécute les tâches en parallèle par vagues (`threading.Thread`) |
| `getDependencies(nom)` | Retourne les dépendances d'une tâche donnée |
| `draw()` | Affiche le graphe de précédence avec `networkx` et `matplotlib` |

---

## Exemple — Pipeline de commande

`text.py` modélise le traitement d'une commande e-commerce en **12 tâches** :

```
T1  VerifStock   ──→  T4  ReservStock ──┐
T2  VerifPaie   ──→  T5  ValidPaie   ──┼──→  T7  ConfCommande ──→  T9  PrepColis ──→  T11 Expedition ──→  T12 Tracking
T3  VerifAddr   ──→  T6  CalcLivr    ──┘         │                                        ↑
                                                  └──→  T8  GenFacture ──→  T10 EnvConfirm ──────────────────────────┘
```

Les 3 premières tâches (`VerifStock`, `VerifPaie`, `VerifAddr`) n'ont aucune dépendance : elles s'exécutent **en parallèle** dès le démarrage.  
`maxParallel()` analyse ensuite le reste du graphe et supprime toute précédence superflue.

```python
s1 = TaskSystem(tasks, precedence)
s1.maxParallel()   # optimisation automatique
s1.run()           # exécution parallèle
s1.draw()          # visualisation du graphe
```

---

## Installation & utilisation

**Prérequis :** Python 3.8+

```bash
pip install networkx matplotlib
```

### Lancer l'exemple

```bash
python text.py
```

### Créer son propre système de tâches

```python
from maxpar import Task, TaskSystem

x = None

t1 = Task()
t1.name = "T1"
t1.reads = []
t1.writes = ["x"]
t1.run = lambda: globals().update(x=42)

t2 = Task()
t2.name = "T2"
t2.reads = ["x"]
t2.writes = []
t2.run = lambda: print(x)

precedence = {"T1": [], "T2": ["T1"]}

s = TaskSystem([t1, t2], precedence)
s.maxParallel()
s.run()
```

---

## Structure des fichiers

```
MaximalParallelism/
├── maxpar.py    # Moteur : Task, TaskSystem, validate, bernstein, run, draw
└── text.py      # Exemple : pipeline commande e-commerce (12 tâches)
```

---

## Auteurs

- [naitkaci-anis](https://github.com/naitkaci-anis)
- [chaoukiksr](https://github.com/chaoukiksr)

---

## Licence

MIT
