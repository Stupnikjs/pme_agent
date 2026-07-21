---

## Phase 1 : Environnement & Ingestion de Documents

L'objectif de cette phase est de réussir à lire n'importe quel PDF et d'en extraire un texte exploitable.

* [ ] **1.1 Initialisation du projet**
  * Créer le projet Rust : `cargo new pdf_agent_pme`
  * Configurer le fichier `Cargo.toml` avec les dépendances de base (`tokio`, `serde`, `serde_json`, `anyhow`).

* [ ] **1.2 Pipeline de Parsing PDF**
  * **Cas PDF Natifs :** Intégrer `pdf-extract` ou `lopdf` pour extraire rapidement le texte des documents contenant du texte sélectionnable.
  * **Cas PDF Scannés / Images :** Configurer un fallback d'OCR (binding C++ avec `tesseract` via `tesseract-rs`, ou appel système vers un moteur d'OCR local).
  * **Nettoyage :** Implémenter une fonction de normalisation du texte (suppression du bruit, gestion des sauts de ligne excessifs).

---

## Phase 2 : Moteur d'Inférence LLM Local

Connexion entre le code Rust et le modèle de langage tournant en local.

* [ ] **2.1 Choix de l'approche d'Inférence**
  * **Option A (Recommandée pour démarrer) - Client Ollama :**
    * Installer Ollama sur la machine hôte.
    * Télécharger un modèle adapté au suivi d'instructions et au JSON (ex: `qwen2.5:7b` ou `llama3.2`).
    * Utiliser le crate `ollama-rs` (ou `reqwest`) pour communiquer via l'API HTTP locale.
  * **Option B (100% Autonome) - Embarqué :**
    * Intégrer `llama-cpp-rs` ou `candle-core` (Hugging Face) pour charger directement les fichiers d'exécutables/poids `.gguf` dans l'application.

* [ ] **2.2 Définition des Schémas de Données (Serde)**
  * Définir les structures Rust représentant les données à extraire.
  * *Exemple :*
    ```rust
    #[derive(Debug, Serialize, Deserialize)]
    pub struct LineItem {
        pub description: String,
        pub quantity: f64,
        pub unit_price: f64,
        pub total_price: f64,
    }

    #[derive(Debug, Serialize, Deserialize)]
    pub struct InvoiceData {
        pub invoice_number: Option<String>,
        pub vendor_name: Option<String>,
        pub date: Option<String>,
        pub total_amount: Option<f64>,
        pub items: Vec<LineItem>,
    }
    ```

---

## Phase 3 : Logique Agent & Structuration Contrainte

Garantir que le LLM retourne une réponse valide et exploitable à 100%.

* [ ] **3.1 Prompts & Contraintes JSON**
  * Rédiger les prompts système orientés extraction de données.
  * Activer le **JSON Mode** ou utiliser le formatage par grammaire (GBNF) d'Ollama / llama.cpp pour forcer le modèle à respecter la structure de l'objet Rust.

* [ ] **3.2 Boucle de Validation et Correction**
  * Tenter de désérialiser la réponse du LLM vers la structure Rust (`serde_json::from_str`).
  * Si la désérialisation échoue (JSON malformé ou champ manquant) :
    * Capturer l'erreur Serde.
    * Relancer automatiquement une requête au LLM en lui fournissant son ancienne réponse et l'erreur de typage pour qu'il la corrige.

---

## Phase 4 : Interface & Intégration PME

Rendre l'outil utilisable par les collaborateurs de l'entreprise.

* [ ] **4.1 Interface Utilisateur / CLI**
  * Développer une CLI robuste avec `clap` pour le traitement par lots en ligne de commande.
  * *(Optionnel)* Développer une interface web légère en Rust avec `axum` ou `actix-web` permettant de glisser-déposer des PDF.

* [ ] **4.2 Connecteurs de Sortie**
  * Exportation automatique des données validées :
    * Génération de fichiers **CSV / Excel**.
    * Insertion directe en **base de données** (PostgreSQL / SQLite via `sqlx`).
    * Webhook HTTP vers l'ERP ou le logiciel comptable de la PME.

---

## Phase 5 : Industrialisation & Déploiement

* [ ] **5.1 Optimisation des Performances**
  * Traitement asynchrone des fichiers volumineux avec `tokio`.
  * Mise en cache des résultats de parsing pour éviter les ré-inférences inutiles.

* [ ] **5.2 Packaging**
  * Compiler le binaire de production optimisé (`cargo build --release`).
  * Conteneuriser la solution avec Docker ou distribuer l'exécutable sous forme de binaire unique pré-configuré.
