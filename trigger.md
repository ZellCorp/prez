#!/bin/bash

# Répertoire où se trouve ce script (.git/hooks)
HOOKS_DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"

# On se place dans le dossier des hooks pour que Java trouve bien la classe
cd "$HOOKS_DIR"

# Vérifie si Java et Javac (le compilateur) sont installés
if ! command -v java &> /dev/null || ! command -v javac &> /dev/null; then
    echo "⚠️  [Git Hook] Java ou Javac n'est pas installé/dans le PATH. Hook ignoré."
    exit 0
fi

# 1. Compilation du fichier Java 8 (génère JiraHook.class)
javac JiraHook.java

# Vérifie si la compilation a réussi
if [ $? -ne 0 ]; then
    echo "❌ [Git Hook] Erreur de compilation du script JiraHook.java"
    exit 0
fi

# 2. Exécution de la classe compilée (sans l'extension .java ni .class)
java JiraHook

# Quitte sans erreur pour ne pas bloquer Git
exit 0
