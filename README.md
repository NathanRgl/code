#!/bin/bash
# Le shebang : Indique au système que ce script doit être exécuté par le programme 'bash'.

# =========================================================
# 1. CONFIGURATION
# Les variables sont comme des boîtes pour stocker des informations.
# =========================================================

VM_IP="172.16.20.60" # 'VM_IP' stocke l'adresse IP de la machine cible.
SSH_USER="wilder"    # 'SSH_USER' stocke le nom d'utilisateur pour la connexion SSH.
SSH_PORT="22"        # 'SSH_PORT' stocke le numéro de port pour la connexion SSH (22 est standard).
SUDO_PASSWORD=""     # 'SUDO_PASSWORD' est vide au départ, il stockera le mot de passe sudo.

# =========================================================
# 2. FONCTIONS DE CONNEXION (Les 'outils' du script)
# Ces fonctions permettent d'exécuter des actions à distance.
# =========================================================

# Début de la définition de la fonction 'run_remote'.
run_remote() {
    # ssh : Le programme pour établir une connexion sécurisée à distance.
    # -o ConnectTimeout=10 : Une option pour que SSH attende 10 secondes maximum avant d'abandonner.
    # -p $SSH_PORT : Utilise le port stocké dans la variable 'SSH_PORT'.
    # $SSH_USER@$VM_IP : Spécifie l'utilisateur et l'IP de la machine cible.
    # "$1" : Représente le PREMIER argument (la commande) passé à cette fonction.
    ssh -o ConnectTimeout=10 -p $SSH_PORT $SSH_USER@$VM_IP "$1"
} # Fin de la fonction 'run_remote'.

# Début de la définition de la fonction 'run_remote_sudo' (pour les commandes Administrateur).
run_remote_sudo() {
    # [ -z "$SUDO_PASSWORD" ] : Teste si la variable 'SUDO_PASSWORD' est vide.
    if [ -z "$SUDO_PASSWORD" ]; then
        echo "Mot de passe SUDO requis pour l'utilisateur $SSH_USER."
        # read : Lit l'entrée de l'utilisateur.
        # -s : Mode silencieux (cache les caractères tapés).
        # -p : Affiche un message (prompt) avant de lire.
        read -s -p "Entrez le mot de passe sudo: " SUDO_PASSWORD
        echo # Saute une ligne pour la propreté.
        # export : Rend la variable disponible pour les commandes suivantes.
        export SUDO_PASSWORD
    fi
    # echo "$SUDO_PASSWORD" | : Envoie le mot de passe à la commande suivante via un "pipe" (|).
    # sudo -S : Permet à 'sudo' de lire le mot de passe via le "pipe".
    # "$1" : Exécute la commande passée en argument avec les droits sudo sur la VM.
    echo "$SUDO_PASSWORD" | ssh -o ConnectTimeout=10 -p $SSH_PORT $SSH_USER@$VM_IP "sudo -S $1"
} # Fin de la fonction 'run_remote_sudo'.

# Fonction pour afficher le titre principal du script.
print_header() {
    clear # Efface le contenu de la fenêtre du terminal.
    echo "==============================================" # Affiche une ligne décorative.
    echo "  GESTIONNAIRE DE VM LINUX (SCRIPT DÉBUTANT)" # Affiche le titre.
    echo "  Connecté à : $VM_IP" # Affiche l'IP actuelle.
    echo "=============================================="
} # Fin de la fonction 'print_header'.

# Fonction pour tester si la connexion fonctionne.
test_connection() {
    # run_remote "echo 'Test réussi'" : Tente d'exécuter la commande 'echo' à distance.
    # &> /dev/null : Redirige la sortie normale ET l'erreur vers la "poubelle" (rien n'est affiché).
    if run_remote "echo 'Test réussi'" &> /dev/null; then
        return 0 # Retourne 0 (le code de succès en Bash) si la connexion marche.
    else
        return 1 # Retourne 1 (le code d'échec en Bash) si la connexion échoue.
    fi
} # Fin de la fonction 'test_connection'.

# =========================================================
# 3. NOUVEL OUTIL : OUTILS D'APPRENTISSAGE (DIAGNOSTIC)
# Ces fonctions aident à comprendre les bases du réseau.
# =========================================================

# NOUVEAU : Fonction de diagnostic réseau simple.
simple_network_diag() {
    echo "--- DIAGNOSTIC DE CONNEXION SIMPLE ---"
    
    # Étape 1 : Tester la joignabilité (ping).
    echo "1. Tentative de 'ping' vers la VM ($VM_IP) :"
    # ping -c 3 : Envoie 3 paquets de test.
    # $VM_IP : L'adresse IP de la machine cible.
    ping -c 3 $VM_IP 

    # $? est le code de retour de la dernière commande (ping).
    if [ $? -eq 0 ]; then
        echo "✅ Le ping local vers la VM a réussi. La VM est probablement allumée et le réseau fonctionne."
    else
        echo "❌ Échec du ping. La VM est peut-être éteinte ou le pare-feu la bloque."
    fi

    echo "------------------------------------"
    
    # Étape 2 : Vérifier si le port SSH est ouvert (commande simple).
    echo "2. Tentative de connexion au port $SSH_PORT (SSH) :"
    # nc -z : netcat en mode 'zero-I/O' (ne fait que vérifier l'écoute).
    # -w 1 : Attend 1 seconde maximum.
    # $VM_IP $SSH_PORT : L'IP et le port à tester.
    nc -z -w 1 $VM_IP $SSH_PORT

    if [ $? -eq 0 ]; then
        echo "✅ Le port SSH ($SSH_PORT) semble ouvert sur la VM."
    else
        echo "❌ Le port SSH semble fermé. Problème de service ou de pare-feu."
    fi
}

# NOUVEAU : Fonction pour voir l'utilisation CPU en temps réel.
show_realtime_cpu() {
    echo "--- UTILISATION CPU ET PROCESSUS EN TEMPS RÉEL (TOP) ---"
    echo "Vous allez voir une mise à jour en direct. Appuyez sur 'q' pour quitter."
    # ssh -t : Lance une session SSH interactive.
    # top : Commande qui affiche les processus et l'utilisation des ressources en direct.
    ssh -t -p $SSH_PORT $SSH_USER@$VM_IP "top"
}

# =========================================================
# 5. GESTION DES UTILISATEURS (Fonctions existantes)
# (Le contenu reste identique à la version ultra-détaillée précédente)
# =========================================================

# Créer un utilisateur
create_user() {
    echo "--- CRÉATION D'UTILISATEUR ---"
    read -p "Nom d'utilisateur à créer : " username 
    if run_remote "id $username" &> /dev/null; then
        echo "❌ ÉCHEC : L'utilisateur '$username' existe déjà sur la VM."
        return 1
    fi
    read -p "Voulez-vous vraiment créer l'utilisateur '$username' ? (o/n) : " confirm
    if [[ $confirm != "o" ]]; then
        echo "Opération annulée."
        return 1
    fi
    if run_remote_sudo "useradd -m $username"; then
        read -s -p "Définissez le mot de passe : " password 
        echo
        if run_remote_sudo "echo '$username:$password' | chpasswd"; then
            echo "✅ Utilisateur '$username' créé avec mot de passe."
        else
            echo "❌ ERREUR : Le mot de passe n'a pas pu être défini."
        fi
    else
        echo "❌ ERREUR : La commande useradd a échoué."
    fi
}

# Supprimer un utilisateur
delete_user() {
    echo "--- SUPPRESSION D'UTILISATEUR ---"
    read -p "Nom d'utilisateur à supprimer : " username
    if ! run_remote "id $username" &> /dev/null; then
        echo "❌ ÉCHEC : L'utilisateur '$username' n'existe pas."
        return 1
    fi
    read -p "ATTENTION : Voulez-vous vraiment SUPPRIMER l'utilisateur '$username' ? (o/n) : " confirm
    if [[ $confirm != "o" ]]; then
        echo "Opération annulée."
        return 1
    fi
    read -p "Supprimer son répertoire personnel aussi ? (o/n) : " delete_home
    if [[ $delete_home == "o" ]]; then
        run_remote_sudo "userdel -r $username" 
    else
        run_remote_sudo "userdel $username"
    fi
    if [ $? -eq 0 ]; then
        echo "✅ Utilisateur '$username' supprimé."
    else
        echo "❌ Échec de la suppression."
    fi
}

# Lister les utilisateurs
list_users() {
    echo "--- LISTE DES UTILISATEURS ---"
    run_remote "cat /etc/passwd | cut -d: -f1 | sort"
}

# =========================================================
# 6. GESTION VM (Fonctions existantes)
# (Le contenu reste identique à la version ultra-détaillée précédente)
# =========================================================

connect_vm() { echo "--- CONNEXION INTERACTIVE SSH ---"; echo "Tapez 'exit' pour revenir au menu."; ssh -p $SSH_PORT $SSH_USER@$VM_IP; }
execute_script() { echo "--- EXÉCUTION DE SCRIPT DISTANT ---"; read -p "Chemin du script à exécuter sur la VM : " script_path; if run_remote "bash $script_path"; then echo "✅ Script exécuté."; else echo "❌ Erreur d'exécution."; fi }
reboot_vm() { echo "--- REDÉMARRAGE DE LA VM ---"; read -p "ATTENTION : Voulez-vous vraiment REDÉMARRER la VM ? (o/n) : " confirm; if [[ $confirm == "o" ]]; then if run_remote_sudo "shutdown -r now"; then echo "✅ Commande de redémarrage envoyée. La connexion sera perdue."; else echo "❌ Erreur lors de l'envoi de la commande shutdown."; fi else echo "Opération annulée."; fi }
stop_vm() { echo "--- ARRÊT DE LA VM ---"; read -p "ATTENTION : Voulez-vous vraiment ARRÊTER la VM ? (o/n) : " confirm; if [[ $confirm == "o" ]]; then if run_remote_sudo "shutdown -h now"; then echo "✅ Commande d'arrêt envoyée. La connexion sera perdue."; else echo "❌ Erreur lors de l'envoi de la commande shutdown."; fi else echo "Opération annulée."; fi }
show_open_ports() { echo "--- PORTS OUVERTS (LISTEN) ---"; run_remote "ss -tulpn"; }
show_network_info() { echo "--- INFORMATIONS RÉSEAU ---"; run_remote "ip addr show && echo '---' && ip route"; }
show_os_version() { echo "--- VERSION DU SYSTÈME D'EXPLOITATION ---"; run_remote "cat /etc/os-release && echo '---' && uname -a"; }
list_packages() { echo "--- PAQUETS INSTALLÉS (20 premiers) ---"; run_remote "dpkg -l 2>/dev/null | head -20 || echo 'dpkg non disponible (non Debian)'"; }
list_services() { echo "--- SERVICES ACTIFS (15 premiers) ---"; run_remote "systemctl list-units --type=service --state=running 2>/dev/null | head -15 || echo 'Systemctl non disponible'"; }
system_info() { echo "--- INFOS SYSTÈME RAPIDES ---"; run_remote 'echo "Uptime: $(uptime)" && echo "--- CPU ---" && lscpu | grep "Model name" | head -1 && echo "--- Mémoire ---" && free -h && echo "--- Disque ---" && df -h'; }
show_detailed_system_info() { echo "--- INFOS SYSTÈME DÉTAILLÉES ---"; run_remote "hostnamectl && echo '--- CPU ---' && lscpu && echo '--- Mémoire ---' && free -h"; }
create_directory() { echo "--- CRÉATION DE RÉPERTOIRE ---"; read -p "Chemin du répertoire : " dir_path; if run_remote "[ -d $dir_path ]"; then echo "❌ ÉCHEC : Le répertoire '$dir_path' existe déjà."; return 1; fi; read -p "Voulez-vous vraiment créer le répertoire '$dir_path' ? (o/n) : " confirm; if [[ $confirm != "o" ]]; then echo "Opération annulée."; return 1; fi; if run_remote_sudo "mkdir -p '$dir_path' && chmod 755 '$dir_path'"; then echo "✅ Répertoire créé."; else echo "❌ Erreur à la création."; fi }
delete_directory() { echo "--- SUPPRESSION DE RÉPERTOIRE ---"; read -p "Chemin du répertoire : " dir_path; if ! run_remote "[ -d $dir_path ]"; then echo "❌ ÉCHEC : Le répertoire '$dir_path' n'existe pas."; return 1; fi; read -p "ATTENTION : Voulez-vous vraiment SUPPRIMER le répertoire '$dir_path' ? (o/n) : " confirm; if [[ $confirm == "o" ]]; then if run_remote_sudo "rm -rf '$dir_path'"; then echo "✅ Répertoire supprimé."; else echo "❌ Erreur à la suppression."; fi else echo "Opération annulée."; fi }
enable_firewall() { echo "--- ACTIVER PARE-FEU (UFW) ---"; read -p "Voulez-vous vraiment ACTIVER le pare-feu UFW ? (o/n) : " confirm; if [[ $confirm != "o" ]]; then echo "Opération annulée."; return 1; fi; run_remote_sudo "ufw enable 2>/dev/null && ufw status verbose 2>/dev/null || echo 'UFW non disponible ou erreur'"; }
show_ram_info() { echo "--- INFOS MÉMOIRE RAM ---"; run_remote "free -h && echo '---' && grep -i 'memtotal\|memfree\|swaptotal\|swapfree' /proc/meminfo"; }
check_updates() { echo "--- RECHERCHE ET INSTALLATION DES MISES À JOUR ---"; read -p "Voulez-vous vraiment mettre à jour la liste des paquets et installer les mises à jour ? (o/n) : " confirm; if [[ $confirm != "o" ]]; then echo "Opération annulée."; return 1; fi; if run_remote_sudo "apt update 2>/dev/null && apt upgrade -y 2>/dev/null"; then echo "✅ Mises à jour terminées."; else echo "❌ Erreur lors des mises à jour (APT non disponible ?)"; fi }
show_system_model() { echo "--- INFOS MATÉRIELLES ---"; run_remote_sudo "dmidecode -s system-manufacturer 2>/dev/null || echo 'Fabricant: Non disponible' ; dmidecode -s system-product-name 2>/dev/null || echo 'Modèle: Non disponible'"; }
check_sudo_config() { echo "--- UTILISATEURS DU GROUPE SUDO ---"; run_remote "grep -E '^sudo:' /etc/group"; }
search_logs_user() { echo "--- RECHERCHE LOGS (USER) ---"; read -p "Nom d'utilisateur à chercher : " username; run_remote_sudo "grep -i '$username' /var/log/auth.log 2>/dev/null | tail -20 || echo 'Fichier non trouvé'"; }
search_logs_computer() { echo "--- RECHERCHE LOGS (HÔTE) ---"; read -p "Nom d'hôte ou mot-clé à chercher : " hostname; run_remote_sudo "grep -i '$hostname' /var/log/syslog 2>/dev/null | tail -20 || echo 'Fichier non trouvé'"; }
show_disk_usage() { echo "--- UTILISATION DES DISQUES (ESPACE LIBRE) ---"; run_remote "df -h"; }
show_processes() { echo "--- PROCESSUS EN COURS (Top 15 CPU) ---"; run_remote "ps aux --sort=-%cpu | head -15"; }

# =========================================================
# 7. MENUS (Ajout du nouveau menu Outils)
# =========================================================

# Nouveau menu pour les outils d'apprentissage et de diagnostic.
learning_tools_menu() {
    while true; do
        print_header
        echo "=== MENU OUTILS D'APPRENTISSAGE ==="
        echo "1. Diagnostic Réseau Simple (Ping, Port SSH)"
        echo "2. Utilisation CPU en Temps Réel (Commande 'top')"
        echo "3. Retour au menu principal"
        echo
        read -p "Choisissez une option [1-3]: " choice

        case $choice in
            1) simple_network_diag ;;
            2) show_realtime_cpu ;;
            3) break ;;
            *) echo "Option invalide" ; sleep 2 ;;
        esac
        read -p "Appuyez sur Entrée pour continuer..."
    done
}

user_management_menu() {
    while true; do
        print_header
        echo "=== MENU UTILISATEURS ==="
        echo "1. Créer un utilisateur (avec vérification et confirmation)"
        echo "2. Supprimer un utilisateur (avec vérification et confirmation)"
        echo "3. Lister les utilisateurs"
        echo "4. Retour au menu principal"
        echo
        read -p "Choisissez une option [1-4]: " choice

        case $choice in
            1) create_user ;;
            2) delete_user ;;
            3) list_users ;;
            4) break ;;
            *) echo "Option invalide" ; sleep 2 ;;
        esac
        read -p "Appuyez sur Entrée pour continuer..."
    done
}

vm_management_menu() {
    while true; do
        print_header
        echo "=== GESTION DÉTAILLÉE DE LA VM ==="
        echo "1.  Se connecter à la VM"
        echo "2.  Exécuter un script distant"
        echo "3.  Redémarrer la VM (CONFIRMATION)"
        echo "4.  Arrêter la VM (CONFIRMATION)"
        echo "5.  Ports ouverts"
        echo "6.  Informations réseau"
        echo "7.  Version OS"
        echo "8.  Paquets installés (Top 20)"
        echo "9.  Services en cours (Top 15)"
        echo "10. Infos système (Rapides)"
        echo "11. Infos système détaillées"
        echo "12. Créer répertoire (VÉRIFICATION & CONFIRMATION)"
        echo "13. Supprimer répertoire (VÉRIFICATION & CONFIRMATION)"
        echo "14. Activer pare-feu (CONFIRMATION)"
        echo "15. Infos Mémoire RAM"
        echo "16. Mises à jour et installation (CONFIRMATION)"
        echo "17. Marque/Modèle du matériel"
        echo "18. Configuration Sudo"
        echo "19. Recherche logs par utilisateur"
        echo "20. Recherche logs par hôte/mot-clé"
        echo "21. Utilisation disque"
        echo "22. Processus en cours (Top CPU)"
        echo "23. Retour au menu principal"
        echo
        read -p "Choisissez une option [1-23]: " choice

        case $choice in
            1) connect_vm ;;
            2) execute_script ;;
            3) reboot_vm ;;
            4) stop_vm ;;
            5) show_open_ports ;;
            6) show_network_info ;;
            7) show_os_version ;;
            8) list_packages ;;
            9) list_services ;;
            10) system_info ;;
            11) show_detailed_system_info ;;
            12) create_directory ;;
            13) delete_directory ;;
            14) enable_firewall ;;
            15) show_ram_info ;;
            16) check_updates ;;
            17) show_system_model ;;
            18) check_sudo_config ;;
            19) search_logs_user ;;
            20) search_logs_computer ;;
            21) show_disk_usage ;;
            22) show_processes ;;
            23) break ;;
            *) echo "Option invalide" ; sleep 2 ;;
        esac
        read -p "Appuyez sur Entrée pour continuer..."
    done
}

main_menu() {
    while true; do
        print_header
        echo "=== MENU PRINCIPAL ==="
        echo "1. Gérer les utilisateurs (Ajout/Suppression)"
        echo "2. Gérer la VM (Infos, Mises à jour, Logs, etc.)"
        echo "3. Outils d'Apprentissage et Diagnostic 💡"
        echo "4. Tester la connexion SSH"
        echo "5. Quitter le script"
        echo
        read -p "Choisissez une option [1-5]: " choice

        case $choice in
            1) user_management_menu ;;
            2) vm_management_menu ;;
            3) learning_tools_menu ;; # Nouvelle option ici !
            4) test_connection; read -p "Appuyez sur Entrée pour continuer..." ;;
            5)
                echo "Au revoir !"
                exit 0
                ;;
            *)
                echo "Option invalide"
                sleep 2
                ;;
        esac
    done
}

# =========================================================
# 8. DÉMARRAGE DU SCRIPT
# =========================================================

echo "Démarrage du Gestionnaire VM..."
echo "IP cible: $VM_IP"
echo "Utilisateur: $SSH_USER"
echo

if test_connection; then 
    echo "✅ Connexion SSH OK. Le script est prêt à fonctionner."
else
    echo "❌ Échec de la connexion SSH. Vérifiez l'IP, le port ou l'utilisateur."
    read -p "Voulez-vous quand même continuer sans connexion établie ? (o/n) : " continue_anyway
    if [[ $continue_anyway != "o" ]]; then
        exit 1
    fi
fi

echo
read -p "Appuyez sur Entrée pour lancer le gestionnaire..."
main_menu
