# Guide Complet - Projet Minishell

## Table des Matières
1. [Introduction](#introduction)
2. [Objectifs du Projet](#objectifs-du-projet)
3. [Architecture Générale](#architecture-générale)
4. [Étapes de Développement](#étapes-de-développement)
5. [Concepts Clés](#concepts-clés)
6. [Implémentation des Built-ins](#implémentation-des-built-ins)
7. [Gestion des Commandes Externes](#gestion-des-commandes-externes)
8. [Gestion des Erreurs](#gestion-des-erreurs)
9. [Tests et Validation](#tests-et-validation)
10. [Conseils et Pièges à Éviter](#conseils-et-pièges-à-éviter)

---

## Introduction

Le projet Minishell consiste à créer un interpréteur de commandes UNIX simplifié basé sur TCSH. C'est une première étape vers la réalisation du projet 42sh.

### Contraintes Techniques
- **Binaire** : `mysh`
- **Langage** : C
- **Compilation** : Makefile avec règles `all`, `re`, `clean`, `fclean`
- **Fonctions autorisées** : malloc, free, exit, opendir, readdir, closedir, getcwd, chdir, fork, stat, lstat, fstat, open, close, getline, strtok, strtok_r, read, write, execve, access, isatty, wait, waitpid, wait3, wait4, signal, kill, getpid, strerror, perror, strsignal

---

## Objectifs du Projet

### Fonctionnalités Obligatoires
1. Afficher un prompt (`$>`) et attendre une commande
2. Exécuter des commandes simples (pas de pipes ni redirections)
3. Chercher les commandes dans le PATH ou via chemin absolu
4. Gérer l'environnement (copie et manipulation)
5. Implémenter les built-ins : `cd`, `setenv`, `unsetenv`, `env`, `exit`
6. Gérer les erreurs avec messages appropriés
7. Retourner le bon code de sortie

### Restrictions
- `env` ne prend aucun argument
- `unsetenv` ne supporte pas le wildcard `*`
- Pas de pipes, redirections ou autres fonctionnalités avancées

---

## Architecture Générale

### Structure du Projet

```
minishell/
├── include/
│   └── shell.h          # Déclarations et structures
├── src/
│   ├── main.c           # Point d'entrée et boucle principale
│   ├── builtins/
│   │   ├── cd.c         # Implémentation de cd
│   │   ├── env.c        # Implémentation de env
│   │   ├── setenv.c     # Implémentation de setenv
│   │   ├── unsetenv.c   # Implémentation de unsetenv
│   │   └── exit.c       # Implémentation de exit
│   ├── execution/
│   │   ├── exec.c       # Exécution des commandes externes
│   │   └── path.c       # Recherche dans PATH
│   ├── parsing/
│   │   └── parser.c     # Parsing de la ligne de commande
│   ├── environment/
│   │   └── env_utils.c  # Gestion de l'environnement
│   └── utils/
│       ├── string.c     # Utilitaires pour strings
│       └── error.c      # Gestion des erreurs
└── Makefile
```

### Structures de Données Essentielles

```c
// Liste chaînée pour l'environnement
typedef struct s_env {
    char *key;              // Nom de la variable
    char *value;            // Valeur de la variable
    struct s_env *next;     // Élément suivant
} t_env;

// Contexte du shell
typedef struct s_shell {
    t_env *env;             // Liste des variables d'environnement
    char **paths;           // Chemins du PATH parsés
    int last_exit_status;   // Code de sortie de la dernière commande
} t_shell;

// Commande parsée
typedef struct s_cmd {
    char **args;            // Arguments de la commande
    int argc;               // Nombre d'arguments
} t_cmd;
```

---

## Étapes de Développement

### Phase 1 : Mise en Place de Base

#### 1.1 Créer la Boucle Principale
```c
int main(int ac, char **av, char **envp)
{
    t_shell shell;
    char *line;
    
    init_shell(&shell, envp);  // Initialiser le shell avec l'environnement
    
    while (1) {
        write(1, "$> ", 3);     // Afficher le prompt
        line = read_line();      // Lire la ligne de commande
        
        if (line == NULL)        // EOF (Ctrl+D)
            break;
            
        if (line[0] != '\0')     // Si la ligne n'est pas vide
            execute_line(&shell, line);
            
        free(line);
    }
    
    cleanup_shell(&shell);
    return shell.last_exit_status;
}
```

#### 1.2 Lecture de la Ligne de Commande
```c
char *read_line(void)
{
    char *line = NULL;
    size_t len = 0;
    ssize_t read;
    
    read = getline(&line, &len, stdin);
    
    if (read == -1) {
        free(line);
        return NULL;
    }
    
    // Retirer le '\n' final
    if (line[read - 1] == '\n')
        line[read - 1] = '\0';
        
    return line;
}
```

### Phase 2 : Gestion de l'Environnement

#### 2.1 Initialisation de l'Environnement
```c
void init_shell(t_shell *shell, char **envp)
{
    shell->env = create_env_list(envp);
    shell->paths = get_paths(shell->env);
    shell->last_exit_status = 0;
}

t_env *create_env_list(char **envp)
{
    t_env *head = NULL;
    t_env *current = NULL;
    int i = 0;
    
    while (envp[i]) {
        t_env *new = malloc(sizeof(t_env));
        
        // Parser "KEY=VALUE"
        char *equal = strchr(envp[i], '=');
        if (equal) {
            new->key = strndup(envp[i], equal - envp[i]);
            new->value = strdup(equal + 1);
        }
        
        new->next = NULL;
        
        if (head == NULL)
            head = new;
        else
            current->next = new;
            
        current = new;
        i++;
    }
    
    return head;
}
```

#### 2.2 Recherche dans l'Environnement
```c
char *get_env_value(t_env *env, const char *key)
{
    while (env) {
        if (strcmp(env->key, key) == 0)
            return env->value;
        env = env->next;
    }
    return NULL;
}
```

### Phase 3 : Parsing de la Ligne de Commande

#### 3.1 Découpage en Arguments
```c
t_cmd *parse_command(char *line)
{
    t_cmd *cmd = malloc(sizeof(t_cmd));
    int capacity = 10;
    
    cmd->args = malloc(sizeof(char *) * capacity);
    cmd->argc = 0;
    
    char *token = strtok(line, " \t");
    
    while (token) {
        if (cmd->argc >= capacity) {
            capacity *= 2;
            cmd->args = realloc(cmd->args, sizeof(char *) * capacity);
        }
        
        cmd->args[cmd->argc++] = strdup(token);
        token = strtok(NULL, " \t");
    }
    
    cmd->args[cmd->argc] = NULL;  // Terminer avec NULL
    
    return cmd;
}
```

### Phase 4 : Exécution des Commandes

#### 4.1 Dispatcher Built-ins vs Externes
```c
void execute_line(t_shell *shell, char *line)
{
    t_cmd *cmd = parse_command(line);
    
    if (cmd->argc == 0) {
        free_command(cmd);
        return;
    }
    
    // Vérifier si c'est un built-in
    if (is_builtin(cmd->args[0]))
        shell->last_exit_status = execute_builtin(shell, cmd);
    else
        shell->last_exit_status = execute_external(shell, cmd);
        
    free_command(cmd);
}

int is_builtin(char *cmd)
{
    if (strcmp(cmd, "cd") == 0)
        return 1;
    if (strcmp(cmd, "env") == 0)
        return 1;
    if (strcmp(cmd, "setenv") == 0)
        return 1;
    if (strcmp(cmd, "unsetenv") == 0)
        return 1;
    if (strcmp(cmd, "exit") == 0)
        return 1;
    return 0;
}
```

#### 4.2 Exécution des Commandes Externes
```c
int execute_external(t_shell *shell, t_cmd *cmd)
{
    char *path = find_command(shell->paths, cmd->args[0]);
    
    if (path == NULL) {
        write(2, cmd->args[0], strlen(cmd->args[0]));
        write(2, ": Command not found.\n", 21);
        return 1;
    }
    
    pid_t pid = fork();
    
    if (pid == -1) {
        perror("fork");
        return 1;
    }
    
    if (pid == 0) {  // Processus enfant
        char **envp = env_to_array(shell->env);
        execve(path, cmd->args, envp);
        
        // Si execve échoue
        perror("execve");
        exit(1);
    }
    
    // Processus parent
    int status;
    waitpid(pid, &status, 0);
    
    free(path);
    
    if (WIFEXITED(status))
        return WEXITSTATUS(status);
    if (WIFSIGNALED(status) && WTERMSIG(status) == SIGSEGV) {
        write(1, "Segmentation fault\n", 19);
        return 139;
    }
    
    return 1;
}
```

#### 4.3 Recherche dans le PATH
```c
char *find_command(char **paths, char *cmd)
{
    // Si le chemin est absolu ou relatif
    if (cmd[0] == '/' || cmd[0] == '.') {
        if (access(cmd, X_OK) == 0)
            return strdup(cmd);
        return NULL;
    }
    
    // Recherche dans PATH
    for (int i = 0; paths[i]; i++) {
        char *full_path = malloc(strlen(paths[i]) + strlen(cmd) + 2);
        sprintf(full_path, "%s/%s", paths[i], cmd);
        
        if (access(full_path, X_OK) == 0)
            return full_path;
            
        free(full_path);
    }
    
    return NULL;
}

char **get_paths(t_env *env)
{
    char *path_value = get_env_value(env, "PATH");
    
    if (path_value == NULL)
        return NULL;
        
    // Parser PATH en tableau
    int count = 1;
    for (int i = 0; path_value[i]; i++)
        if (path_value[i] == ':')
            count++;
            
    char **paths = malloc(sizeof(char *) * (count + 1));
    char *path_copy = strdup(path_value);
    char *token = strtok(path_copy, ":");
    int i = 0;
    
    while (token) {
        paths[i++] = strdup(token);
        token = strtok(NULL, ":");
    }
    
    paths[i] = NULL;
    free(path_copy);
    
    return paths;
}
```

---

## Implémentation des Built-ins

### cd - Change Directory

```c
int builtin_cd(t_shell *shell, t_cmd *cmd)
{
    char *path;
    char cwd[4096];
    char *oldpwd;
    
    // Sauvegarder le répertoire actuel
    if (getcwd(cwd, sizeof(cwd)) == NULL) {
        perror("getcwd");
        return 1;
    }
    
    // Déterminer le chemin cible
    if (cmd->argc == 1 || strcmp(cmd->args[1], "~") == 0) {
        // cd sans argument ou cd ~ -> HOME
        path = get_env_value(shell->env, "HOME");
        if (path == NULL) {
            write(2, "cd: HOME not set\n", 17);
            return 1;
        }
    } else if (strcmp(cmd->args[1], "-") == 0) {
        // cd - -> OLDPWD
        path = get_env_value(shell->env, "OLDPWD");
        if (path == NULL) {
            write(2, ": No such file or directory.\n", 29);
            return 1;
        }
        write(1, path, strlen(path));
        write(1, "\n", 1);
    } else {
        path = cmd->args[1];
    }
    
    // Vérifier l'existence et les permissions
    if (access(path, F_OK) == -1) {
        write(2, path, strlen(path));
        write(2, ": No such file or directory.\n", 29);
        return 1;
    }
    
    if (access(path, R_OK) == -1) {
        write(2, path, strlen(path));
        write(2, ": Permission denied.\n", 21);
        return 1;
    }
    
    // Changer de répertoire
    if (chdir(path) != 0) {
        write(2, path, strlen(path));
        write(2, ": Not a directory.\n", 19);
        return 1;
    }
    
    // Mettre à jour OLDPWD et PWD
    set_env_value(shell->env, "OLDPWD", cwd);
    
    if (getcwd(cwd, sizeof(cwd)) != NULL)
        set_env_value(shell->env, "PWD", cwd);
    
    return 0;
}
```

### env - Display Environment

```c
int builtin_env(t_shell *shell, t_cmd *cmd)
{
    if (cmd->argc > 1) {
        write(2, "env: too many arguments\n", 24);
        return 1;
    }
    
    t_env *current = shell->env;
    
    while (current) {
        write(1, current->key, strlen(current->key));
        write(1, "=", 1);
        write(1, current->value, strlen(current->value));
        write(1, "\n", 1);
        current = current->next;
    }
    
    return 0;
}
```

### setenv - Set Environment Variable

```c
int builtin_setenv(t_shell *shell, t_cmd *cmd)
{
    // Pas d'argument -> afficher l'environnement
    if (cmd->argc == 1)
        return builtin_env(shell, cmd);
    
    // Trop d'arguments
    if (cmd->argc > 3) {
        write(2, "setenv: Too many arguments.\n", 28);
        return 1;
    }
    
    // Valider le nom de variable
    if (!is_valid_var_name(cmd->args[1])) {
        write(2, "setenv: Variable name must contain ", 35);
        write(2, "alphanumeric characters.\n", 25);
        return 1;
    }
    
    char *value = (cmd->argc == 3) ? cmd->args[2] : "";
    
    set_env_value(shell->env, cmd->args[1], value);
    
    // Si on modifie PATH, mettre à jour les paths
    if (strcmp(cmd->args[1], "PATH") == 0) {
        free_array(shell->paths);
        shell->paths = get_paths(shell->env);
    }
    
    return 0;
}

int is_valid_var_name(char *name)
{
    if (!name || !name[0])
        return 0;
        
    // Premier caractère doit être lettre ou underscore
    if (!isalpha(name[0]) && name[0] != '_')
        return 0;
        
    // Autres caractères : alphanumériques ou underscore
    for (int i = 1; name[i]; i++) {
        if (!isalnum(name[i]) && name[i] != '_')
            return 0;
    }
    
    return 1;
}

void set_env_value(t_env *env, const char *key, const char *value)
{
    // Chercher si la variable existe déjà
    t_env *current = env;
    t_env *last = NULL;
    
    while (current) {
        if (strcmp(current->key, key) == 0) {
            // Remplacer la valeur
            free(current->value);
            current->value = strdup(value);
            return;
        }
        last = current;
        current = current->next;
    }
    
    // Ajouter une nouvelle variable
    t_env *new = malloc(sizeof(t_env));
    new->key = strdup(key);
    new->value = strdup(value);
    new->next = NULL;
    
    if (last)
        last->next = new;
}
```

### unsetenv - Unset Environment Variable

```c
int builtin_unsetenv(t_shell *shell, t_cmd *cmd)
{
    if (cmd->argc < 2) {
        write(2, "unsetenv: Too few arguments.\n", 29);
        return 1;
    }
    
    for (int i = 1; i < cmd->argc; i++) {
        remove_env_var(&shell->env, cmd->args[i]);
    }
    
    // Si on supprime PATH, mettre à jour les paths
    if (cmd->argc > 1 && strcmp(cmd->args[1], "PATH") == 0) {
        free_array(shell->paths);
        shell->paths = get_paths(shell->env);
    }
    
    return 0;
}

void remove_env_var(t_env **env, const char *key)
{
    t_env *current = *env;
    t_env *prev = NULL;
    
    while (current) {
        if (strcmp(current->key, key) == 0) {
            if (prev)
                prev->next = current->next;
            else
                *env = current->next;
                
            free(current->key);
            free(current->value);
            free(current);
            return;
        }
        prev = current;
        current = current->next;
    }
}
```

### exit - Exit Shell

```c
int builtin_exit(t_shell *shell, t_cmd *cmd)
{
    int exit_code = 0;
    
    if (cmd->argc > 1) {
        // Parser le code de sortie
        exit_code = atoi(cmd->args[1]);
    } else {
        // Utiliser le code de sortie de la dernière commande
        exit_code = shell->last_exit_status;
    }
    
    cleanup_shell(shell);
    exit(exit_code);
}
```

---

## Gestion des Erreurs

### Messages d'Erreur Conformes à TCSH

```c
void print_error(const char *cmd, const char *msg)
{
    write(2, cmd, strlen(cmd));
    write(2, ": ", 2);
    write(2, msg, strlen(msg));
    write(2, "\n", 1);
}

// Exemples d'utilisation :
// Command not found
print_error(cmd, "Command not found.");

// Permission denied
print_error(cmd, "Permission denied.");

// Not a directory
print_error(path, "Not a directory.");

// No such file or directory
print_error(path, "No such file or directory.");
```

### Gestion des Signaux

```c
void handle_segfault(int status)
{
    if (WIFSIGNALED(status) && WTERMSIG(status) == SIGSEGV) {
        write(1, "Segmentation fault\n", 19);
    }
}
```

---

## Tests et Validation

### Cas de Test Essentiels

#### Test 1 : Commandes Simples
```bash
echo "ls" | ./mysh
echo "pwd" | ./mysh
echo "echo hello world" | ./mysh
```

#### Test 2 : Built-ins
```bash
# cd
echo -e "cd /tmp\npwd" | ./mysh

# env
echo "env" | ./mysh

# setenv
echo -e "setenv TEST value\nenv" | ./mysh

# unsetenv
echo -e "unsetenv PATH\nls" | ./mysh

# exit
echo "exit 42" | ./mysh
echo $?  # Doit afficher 42
```

#### Test 3 : Gestion des Erreurs
```bash
# Commande inexistante
echo "commandeinexistante" | ./mysh

# Permission denied
echo "touch /root/test" | ./mysh

# Répertoire inexistant
echo "cd /repertoireinexistant" | ./mysh
```

#### Test 4 : Chemins Absolus et Relatifs
```bash
echo "/bin/ls" | ./mysh
echo "./myprogram" | ./mysh
```

#### Test 5 : Codes de Sortie
```bash
echo -e "ls\nexit" | ./mysh
echo $?  # Doit être 0

echo -e "ls fichier_inexistant\nexit" | ./mysh
echo $?  # Doit être 2
```

### Script de Test Automatique

```bash
#!/bin/bash

RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m'

test_count=0
pass_count=0

run_test() {
    local name="$1"
    local command="$2"
    local expected="$3"
    
    test_count=$((test_count + 1))
    
    result=$(echo "$command" | ./mysh 2>&1)
    
    if [ "$result" == "$expected" ]; then
        echo -e "${GREEN}✓${NC} $name"
        pass_count=$((pass_count + 1))
    else
        echo -e "${RED}✗${NC} $name"
        echo "  Expected: $expected"
        echo "  Got: $result"
    fi
}

# Tests
run_test "Simple ls" "ls" "$(ls)"
run_test "pwd" "pwd" "$(pwd)"
run_test "Command not found" "commandeinexistante" \
    "commandeinexistante: Command not found."

echo ""
echo "Passed: $pass_count/$test_count"
```

---

## Conseils et Pièges à Éviter

### Conseils Importants

1. **Gestion Mémoire**
   - Toujours libérer la mémoire allouée
   - Utiliser Valgrind pour détecter les fuites : `valgrind --leak-check=full ./mysh`

2. **Variables d'Environnement**
   - Ne jamais modifier directement `envp`, toujours faire une copie
   - Mettre à jour PATH après `setenv PATH=...`

3. **Parsing**
   - Gérer les espaces multiples entre arguments
   - Gérer les tabulations comme des espaces

4. **Fork et Execve**
   - Toujours vérifier le retour de `fork()`
   - Ne jamais oublier `wait()` dans le parent
   - Convertir l'environnement en tableau pour `execve`

5. **Codes de Sortie**
   - Retourner le bon code de sortie de la dernière commande
   - 0 = succès, autre = erreur
   - Segmentation fault = 139 (généralement)

### Pièges Courants

1. **Path avec "./" ou "/"**
```c
// FAUX
if (access(cmd, X_OK) == 0)
    execve(cmd, args, env);

// CORRECT
if (cmd[0] == '/' || cmd[0] == '.') {
    if (access(cmd, X_OK) == 0)
        execve(cmd, args, env);
}
```

2. **Oublier de NULL-terminer les tableaux**
```c
// Pour execve, le tableau d'arguments DOIT être NULL-terminé
args[argc] = NULL;
```

3. **Gérer les espaces vides**
```c
if (line[0] == '\0')  // Ne rien faire pour une ligne vide
    continue;
```

4. **cd sans argument**
```c
// cd sans argument doit aller au HOME
if (argc == 1)
    path = get_env("HOME");
```

5. **Fuites mémoire avec strtok**
```c
// strtok modifie la chaîne, faire une copie si nécessaire
char *line_copy = strdup(line);
token = strtok(line_copy, " ");
// ... utilisation
free(line_copy);
```

### Checklist Avant Rendu

- [ ] Le Makefile compile sans warnings avec `-Wall -Wextra -Werror`
- [ ] Les règles `clean` et `fclean` fonctionnent
- [ ] Aucune fuite mémoire (Valgrind)
- [ ] Tous les built-ins fonctionnent correctement
- [ ] Les commandes externes s'exécutent
- [ ] Les codes de sortie sont corrects
- [ ] Les messages d'erreur sont conformes à TCSH
- [ ] Le PATH est géré correctement
- [ ] Les chemins absolus/relatifs fonctionnent
- [ ] Gestion de Ctrl+D (EOF)

---

## Ressources Complémentaires

### Man Pages Utiles
```bash
man fork
man execve
man wait
man getenv
man chdir
man access
man strtok
```

### Commandes pour Tester TCSH
```bash
# Installer tcsh
sudo apt-get install tcsh

# Lancer tcsh
tcsh

# Tester les messages d'erreur
commandeinexistante
cd /repertoireinexistant
```

### Debugging
```bash
# Compiler avec symboles de debug
gcc -g -o mysh *.c

# Utiliser GDB
gdb ./mysh
(gdb) run
(gdb) backtrace  # Après un crash

# Vérifier les fuites mémoire
valgrind --leak-check=full --show-leak-kinds=all ./mysh
```

---

## Conclusion

Le projet Minishell est une excellente introduction à la programmation système en C. Il nécessite une bonne compréhension de :
- La gestion des processus (fork, exec, wait)
- La manipulation des chaînes de caractères
- La gestion de la mémoire dynamique
- Les appels système UNIX

Prenez le temps de bien comprendre chaque concept avant de coder, testez régulièrement et n'hésitez pas à consulter les man pages. Bon courage !
