# OSCAR-Prompt-Generator

Transforme une **demande brute** en **prompt structuré** selon la méthode mnémotechnique **OSCAR**, via l'API Anthropic. Deux scripts équivalents, sans interface : Python (bibliothèque standard uniquement) et PowerShell 7.

## Sommaire

- [La méthode OSCAR](#la-méthode-oscar)
- [Fonctionnement](#fonctionnement)
- [Prérequis](#prérequis)
- [Installation](#installation)
- [Configuration](#configuration)
- [Utilisation](#utilisation)
- [Options](#options)
- [Format de sortie](#format-de-sortie)
- [Sécurité et confidentialité](#sécurité-et-confidentialité)
- [Limites](#limites)
- [Statut](#statut)
- [Structure du dépôt](#structure-du-dépôt)
- [Contribuer](#contribuer)
- [Licence](#licence)

## La méthode OSCAR

OSCAR est une mnémotechnique en cinq éléments pour rédiger un prompt. Ce n'est **pas** une méthode officielle d'Anthropic : c'est un moyen simple de ne rien oublier d'essentiel.

| Lettre | Élément | Contenu |
|---|---|---|
| **O** | Objectif | Le but final visé, pas seulement la tâche. Tâche : écrire un mail. Objectif : convaincre un prospect de prendre un appel cette semaine. |
| **S** | Spécificités et sources | Informations clés, fichiers, chiffres, exemples : ce qui rend la demande unique. |
| **C** | Contexte | Qui vous êtes, où vous en êtes, contraintes, enjeux. |
| **A** | Action | Un verbe d'action précis : rédige, analyse, reformule, compare, structure. |
| **R** | Résultat | Le format attendu du livrable : e-mail avec objet, tableau comparatif, post, etc. |

## Fonctionnement

1. Vous fournissez une demande brute, en argument ou via stdin / pipeline.
2. Le script y ajoute, si elles existent, vos informations stables (rôle, public, ton, contraintes) lues dans un fichier de **contexte permanent**.
3. Un appel à l'API Messages produit un prompt en cinq blocs OSCAR.
4. Ce qui manque n'est pas inventé : le bloc concerné contient `[À COMPLÉTER : ...]`, et une liste des points à compléter ou à vérifier suit le prompt.

Le script **génère** le prompt ; il ne l'exécute pas.

## Prérequis

- Une clé API Anthropic.
- Python 3 (aucun paquet à installer) **ou** PowerShell 7+.
- Un accès réseau à `api.anthropic.com`.

## Installation

```bash
git clone https://github.com/valorisa/OSCAR-Prompt-Generator.git
cd OSCAR-Prompt-Generator
```

Termux (Android) :

```bash
pkg install python git
```

## Configuration

### Clé API

Linux, macOS, Termux :

```bash
export ANTHROPIC_API_KEY="votre-clé"
```

PowerShell (session courante) :

```powershell
$env:ANTHROPIC_API_KEY = 'votre-clé'
```

PowerShell (persistant pour l'utilisateur) :

```powershell
[Environment]::SetEnvironmentVariable('ANTHROPIC_API_KEY', 'votre-clé', 'User')
```

### Modèle

Le modèle par défaut est `claude-sonnet-5-5`. Pour en changer, utilisez l'option `--model` / `-Model` ou la variable d'environnement `ANTHROPIC_MODEL`. Vérifiez que le nom correspond à un modèle disponible sur votre compte.

### Contexte permanent (optionnel mais recommandé)

Créez `~/.oscar/context.txt` avec ce qui ne change pas d'une demande à l'autre. Il alimente automatiquement le bloc **C**.

```bash
mkdir -p ~/.oscar
cat > ~/.oscar/context.txt << 'EOF'
Je suis [rôle]. Mon public : [qui].
Ton : [direct, formel, etc.]. Langue : [langue].
Contraintes récurrentes : [longueur, interdits, etc.].
EOF
```

PowerShell :

```powershell
New-Item -ItemType Directory -Force "$HOME/.oscar" | Out-Null
Set-Content "$HOME/.oscar/context.txt" -Encoding utf8 -Value @'
Je suis [rôle]. Mon public : [qui].
Ton : [direct, formel, etc.]. Langue : [langue].
Contraintes récurrentes : [longueur, interdits, etc.].
'@
```

Un fichier vide ou absent est accepté : le script fonctionne sans contexte permanent, avec davantage de `[À COMPLÉTER]` dans le résultat.

## Utilisation

Python :

```bash
python oscar.py "écris un post Instagram sur le petit déjeuner"
echo "compare ces deux offres" | python oscar.py
python oscar.py -o prompt.md "ma demande"
```

PowerShell :

```powershell
./oscar.ps1 "écris un post Instagram sur le petit déjeuner"
"compare ces deux offres" | ./oscar.ps1
./oscar.ps1 -OutFile prompt.md "ma demande"
```

Pour inspecter la requête sans appeler l'API (aucune clé nécessaire) :

```bash
python oscar.py --dry-run "ma demande"
```

```powershell
./oscar.ps1 -DryRun "ma demande"
```

## Options

| Python | PowerShell | Rôle | Défaut |
|---|---|---|---|
| `request` (positionnel) | `-Request` (positionnel) | Demande brute (sinon lue sur stdin / pipeline) | — |
| `-c`, `--context FILE` | `-ContextFile` | Fichier de contexte permanent | `~/.oscar/context.txt` |
| `--no-context` | `-NoContext` | Ignore le contexte permanent | — |
| `-l`, `--lang` | `-Language` | Langue du prompt généré | `français` |
| `-m`, `--model` | `-Model` | Modèle utilisé | `ANTHROPIC_MODEL` ou `claude-sonnet-5-5` |
| `--max-tokens` | `-MaxTokens` | Longueur maximale de la réponse | `2000` |
| `--timeout` | `-TimeoutSec` | Délai de la requête, en secondes | `60` |
| `-o`, `--output` | `-OutFile` | Écrit le résultat dans un fichier | sortie standard |
| `--dry-run` | `-DryRun` | Affiche le payload sans appeler l'API | — |

## Format de sortie

La réponse comporte toujours deux parties :

````text
## Prompt OSCAR
(le prompt prêt à coller, dans un bloc de code, en cinq blocs :
 OBJECTIF, SPÉCIFICITÉS ET SOURCES, CONTEXTE, ACTION, RÉSULTAT)

## À compléter ou à vérifier
(liste des [À COMPLÉTER] et des hypothèses faites, ou « Rien. »)
````

Le prompt généré contient une consigne de ne rien inventer (données, statistiques, sources) et de poser la question si une information manque.

## Sécurité et confidentialité

- Ne versionnez jamais votre clé API. Utilisez uniquement une variable d'environnement.
- La demande brute et le contenu de `context.txt` sont envoyés à l'API Anthropic. N'y mettez rien de confidentiel que vous ne voudriez pas transmettre.
- La demande et le contexte sont encadrés par des balises et présentés au modèle comme des données à structurer. Cela réduit le risque qu'un texte collé dans la demande soit interprété comme une instruction, sans l'exclure totalement.

## Limites

- La qualité du prompt dépend de ce que vous fournissez : une demande très courte sans contexte permanent donne surtout des `[À COMPLÉTER]`.
- Un prompt structuré réduit le risque d'une réponse générique, il ne le supprime pas. Relisez et itérez.
- Le modèle peut mal remplir un bloc malgré les consignes. Relisez la section « À compléter ou à vérifier ».
- OSCAR est une mnémotechnique, pas un standard.

## Statut

- Python : compilation, mode `--dry-run`, lecture sur stdin et erreur en l'absence de clé API vérifiés.
- PowerShell : syntaxe à valider avec `./oscar.ps1 -DryRun "test"`.
- Appel API réel : non vérifié de bout en bout dans la version initiale.

Retirez cette section une fois ces points validés.

## Structure du dépôt

```text
OSCAR-Prompt-Generator/
├── oscar.py      # version Python (stdlib uniquement)
├── oscar.ps1     # version PowerShell 7
└── README.md
```

## Contribuer

Les issues et pull requests sont les bienvenues. Pour toute modification du prompt système, gardez les deux scripts synchronisés : le texte est dupliqué dans `oscar.py` et `oscar.ps1`.

## Licence

À définir (ajouter un fichier `LICENSE`).
