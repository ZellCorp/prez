#!/bin/bash

# Répertoire où se trouve ce script (donc .git/hooks)
HOOKS_DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"

# Vérifie si Java est installé
if ! command -v java &> /dev/null; then
    echo "⚠️  [Git Hook] Java n'est pas installé ou n'est pas dans le PATH. Le hook Jira est ignoré."
    exit 0
fi

# Exécute directement le fichier Java (Fonctionnalité native de Java 11+)
java "$HOOKS_DIR/JiraHook.java"

# Quitte sans erreur pour ne jamais bloquer le workflow git
exit 0
