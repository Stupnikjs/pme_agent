---

## 🎯 Objectif Version 0 (Prototype)

> **Cas d'usage :** Déposer un bulletin PDF dans un dossier ──> Extraire les champs clés ──> Générer une ligne dans `salaires.csv`.

**Champs cibles :**
- `nom` (String)
- `mois` (String)
- `brut` (f64)
- `net` (f64)
- `cotisations` (f64)

---

## 📋 Feuille de Route & Checklist de Développement

### Phase 1 : Jeu de Données de Test (Faux documents)
- [ ] **1.1** Créer le dossier `documents/` et `output/`.
- [ ] **1.2** Générer ou créer 3 à 5 faux bulletins de paie PDF annotés clairement **"EXEMPLE / TEST"** avec des mises en page légèrement différentes.

---

### Phase 2 : Pipeline Déterministe (Sans IA)
- [ ] **2.1 Initialisation Rust**
  - Projet `cargo new assistant-rh`.
  - Ajouter les crates : `notify`, `pdf-extract`, `regex`, `serde`, `csv`, `tokio`.
- [ ] **2.2 Surveillance de dossier (`notify`)**
  - Écouter les événements de création de fichiers `.pdf` dans `documents/`.
- [ ] **2.3 Extraction du texte brut (`pdf-extract`)**
  - Lire le flux binaire du PDF et ressortir une chaîne `String` propre.
- [ ] **2.4 Extraction par règles (`regex` + `serde`)**
  - Créer la structure Rust `RecordPaye`.
  - Écrire les expressions régulières pour isoler le Net à payer, le Brut, le Nom et le Mois.
- [ ] **2.5 Exportation CSV (`csv`)**
  - Écrire ou ajouter la ligne extraite dans `output/salaires.csv`.

---

### Phase 3 : OCR & Fallback IA Local (Quand le déterministe échoue)
- [ ] **3.1 Intégration OCR (PDFs scannés)**
  - Ajouter un fallback d'OCR (binding Tesseract) si `pdf-extract` ne renvoie aucun texte.
- [ ] **3.2 Intégration LLM Local (Ollama / `ollama-rs`)**
  - Si les regex échouent (`None`), envoyer le texte brut à un modèle local (`qwen2.5` ou `llama3.2`).
  - Forcer la sortie JSON contrainte pour peupler la structure `RecordPaye`.
- [ ] **3.3 Garde-fous et auto-correction**
  - Valider la désérialisation Serde.
  - Relancer le LLM en cas d'erreur de typage.

---

### Phase 4 : Interface Graphique & Assistant Conversationnel
- [ ] **4.1 Interface GUI (`Tauri` + React/Svelte/HTML)**
  - Afficher le statut des dossiers surveillés.
  - Afficher le tableau des documents analysés avec bouton d'exportation Excel/CSV.
- [ ] **4.2 Évolution "Assistant RAG Local"**
  - Découper les contrats/bulletins en blocs pour recherche vectorielle locale.
  - Permettre de poser des questions en langage naturel (*"Quel était le salaire brut de Paul en mars ?"*).

---

## 🛠️ Stack Technique

| Rôle | Outil / Crate Rust |
| :--- | :--- |
| **Langage** | Rust (Édition 2021) |
| **Surveillance Fichiers** | `notify` |
| **Parsing PDF** | `pdf-extract` / `lopdf` |
| **Extraction Déterministe**| `regex` + `serde` |
| **Génération CSV** | `csv` |
| **Inférence IA (Phase 3)** | `ollama-rs` (ou `llama-cpp-rs`) |
| **Interface (Phase 4)** | `Tauri` |
