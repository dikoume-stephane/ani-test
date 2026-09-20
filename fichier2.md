# Documentation du Système - Projet mbog

## 1. Présentation Générale
Le projet **mbog** repose sur une architecture C++20 hautement modulaire, multi-threadée et axée sur les performances temps réel.
Ce document constitue la référence technique principale pour l'équipe de développement et les collaborateurs externes.

## 2. Architecture Logicielle

L'architecture du système repose sur un modèle en **couches strictes et indépendantes**. Chaque composant est découplé grâce à des interfaces d'abstraction (API internes), garantissant une haute modularité, une isolation des pannes et la possibilité de faire évoluer un sous-système sans impacter les autres.

+-----------------------------------------------------------+|               Interface Applicative (API)                 |+-----------------------------------------------------------+| (Appels / Événements)+-----------------------------------------------------------+|              Moteur d'Exécution (Kernel)                  |+-----------------------------------------------------------+/                                           / (Ordonnancement)                            \ (I/O Asynchrones)+---------------------------+       +-----------------------+|  Sous-système de Stockage |       |     Couche Réseau     |+---------------------------+       +-----------------------+

### 2.1. Couche Réseau 
Cette couche est responsable de la communication avec le monde extérieur. Elle abstrait la complexité des protocoles de transport et maximise le débit de données.

* **Gestion des connexions asynchrones :** Utilisation d'un modèle d'I/O non bloquant basé sur des multiplexeurs natifs (comme `epoll` sous Linux, `kqueue` sous macOS ou `IOCP` sous Windows). Cela permet de gérer des dizaines de milliers de connexions simultanées avec un nombre minimal de threads système.
* **Sockets bas niveau :** Configuration fine des sockets TCP/UDP (activation de `TCP_NODELAY` pour réduire la latence, ajustement des tailles de tampons `SO_RCVBUF` et `SO_SNDBUF`).
* **Gestion des buffers de paquets :** Implémentation d'un pool de buffers pré-alloués pour éviter la fragmentation de la mémoire lors de la réception/envoi de frames, évitant ainsi les allocations dynamiques fréquentes (*Garbage Collection pressure*).
* **Sécurité des flux :** Intégration d'une sous-couche TLS/SSL optionnelle pour le chiffrement des données en transit, gérée de manière transparente avant la transmission au noyau.

### 2.2. Moteur d'Exécution (Kernel)
Le Kernel est le cœur décisionnel du système. Il orchestre les ressources matérielles et distribue la charge de travail de manière optimale.

* **Gestion de la mémoire :** Implémentation d'un allocateur de mémoire personnalisé (*Memory Pool*) dédié aux objets à cycle de vie court. Le Kernel segmente la mémoire en arènes pour garantir des allocations en temps constant (O(1)) et éliminer les fuites de mémoire.
* **Boucle d'événements (Event Loop) :** Un cœur réactif basé sur le pattern *Reactor* ou *Proactor*. Le Kernel scrutement en permanence la file d'attente des événements (notifications réseau, fins d'I/O disque, timers) et distribue les tâches aux workers associés.
* **Ordonnancement des tâches (Scheduling) :** Utilisation d'un ordonnanceur à vol de travail (*Work-Stealing Scheduler*). Les threads de calcul légers (*Green Threads* ou *Coroutines*) sont répartis sur les cœurs CPU physiques disponibles. Si un thread applicatif se retrouve inactif, il "vole" une tâche à la file d'un autre thread pour maximiser le parallélisme.

### 2.3. Sous-système de Stockage (Storage Engine)
Ce sous-système garantit l'intégrité, la durabilité et la rapidité d'accès aux données persistées sur disque.

* **Formats binaires personnalisés :** Les données ne sont pas stockées en texte brut (JSON/XML), mais sérialisées dans un format binaire optimisé (ex: alignement des octets, compression par blocs). Cela réduit l'empreinte disque et accélère drastiquement les phases de lecture/écriture (I/O).
* **Persistance et Journalisation :** Implémentation d'un journal d'écriture en amont (*Write-Ahead Logging - WAL*). Toute modification est d'abord écrite de manière séquentielle dans un fichier de log (très rapide) avant d'être appliquée à la structure de données principale, assurant une résilience totale en cas de crash.
* **Gestion des caches LRU (Least Recently Used) :** Un cache en mémoire vive (RAM) structuré, subdivisé en segments chaud/froid. Lorsque le cache est plein, les pages de données les moins consultées sont automatiquement évincées ou écrites sur disque. Le cache utilise des verrous fins (*Fine-grained locking*) ou des structures *Lock-free* pour éviter les goulots d'étranglement en environnement multithread.

### 2.4. Interface Applicative (API)
La couche API est la façade publique du système, conçue pour offrir une expérience de développement fluide et sécurisée.

* **Abstraction haut niveau :** Masquage complet de la complexité des sockets, de la gestion mémoire et des structures de fichiers. Les développeurs manipulent des concepts métiers clairs à travers des fonctions documentées.
* **Contrats Consommateurs et Producteurs :** Définition stricte des interfaces pour l'injection de données (*Producers*) et la récupération/écoute de flux (*Consumers*). L'API supporte nativement des patterns comme le *Publish-Subscribe* ou le *Request-Response*.
* **Contrôle de flux et Contre-pression (Backpressure) :** Si un consommateur est trop lent par rapport au débit du système, l'API expose des mécanismes de signalement pour ralentir intelligemment les producteurs en amont, évitant ainsi la saturation de la mémoire du serveur.

## 3. Configuration requise
Pour compiler et exécuter le projet dans des conditions optimales :
- **Compilateur :** GCC 11+, Clang 13+ ou MSVC 2022 (support C++20 obligatoire)
- **Outil de Build :** CMake 3.20 ou supérieur, Ninja (recommandé)
- **Bibliothèques Tierces :**
  - OpenSSL 3.0+ (sécurité des flux)
  - zlib (compression des données)
  - GoogleTest 1.12+ (bancs de tests unitaires)

## 4. Arborescence détaillée du Dépôt
Le dépôt respecte une structure stricte pour isoler les responsabilités :
- `Applications/` : Points d'entrée des exécutables, outils CLI et démonstrations.
- `Config/` : Fichiers YAML/JSON de configuration par environnement.
- `Docs/` : Spécifications techniques et manuels d'utilisation.
- `Kernel/` : Cœur dur du système.
  - `Core/` : Allocateurs mémoire, structures de données spécialisées.
  - `Network/` : Protocoles, wrappers de sockets.
  - `Storage/` : Drivers d'E/S et gestionnaires de fichiers.
- `Tests/` : Suite de tests unitaires, d'intégration et de montée en charge.

## 5. Procédure de Compilation
Pour réaliser un build local propre :
1. Créez un répertoire dédié à la compilation : `mkdir build && cd build`
2. Configurez le projet avec CMake : `cmake -DCMAKE_BUILD_TYPE=Release ..`
3. Lancez la compilation parallèle : `cmake --build . --parallel`
4. Exécutez la suite de tests : `ctest --output-on-failure`

## 6. Qualité de Code et Intégration Continue (CI)
Toute modification poussée sur le dépôt déclenche un pipeline automatique :
- **Analyse Statique :** Contrôle des règles avec Clang-Tidy et Cppcheck.
- **Tests Unitaires :** Validation de la couverture de code (cible de 85% minimum).
- **Vérification Mémoire :** Passage des tests sous Valgrind et AddressSanitizer (ASan).
- **Formateur :** Vérification stricte du style via Clang-Format.

## 7. Planning et Délais de Rigueur (16 Semaines)
Le projet s'étale sur un cycle intensif de 4 mois (16 semaines) réparti en 8 sprints de 2 semaines :
- **Sprints 1-2 (S1-S4) :** Mise en place du Kernel, allocateurs mémoire et boucles réseau.
- **Sprints 3-4 (S5-S8) :** Implémentation du moteur de stockage et des mécanismes de persistance.
- **Sprints 5-6 (S9-S12) :** Développement des API haut niveau et des outils de ligne de commande.
- **Sprint 7 (S13-S14) :** Optimisation des performances, benchmarks et résolution des fuites mémoire.
- **Sprint 8 (S15-S16) :** Tirs de charge, finalisation de la documentation et préparation de la v1.0.

## 8. Conventions et Directives Git
- Chaque modification doit faire l'objet d'un commit atomique (un seul sujet par commit).
- Les messages de commit doivent respecter le format *Conventional Commits* (`feat:`, `fix:`, `perfe:`, `docs:`).
- Aucun push direct sur la branche `main` n'est autorisé ; passage obligatoire par une Pull Request avec relecture.
