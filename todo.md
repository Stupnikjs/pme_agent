# Guide de construction — Assistant interne RAG en Rust
### Cas : Menuiserie Dubois (procédures internes)

---

## Vue d'ensemble de l'architecture

```
Question employé
      │
      ▼
[API Rust (axum)]
      │
      ├──► 1. Embedding de la question (via Ollama)
      │
      ├──► 2. Recherche des chunks pertinents (similarité cosinus)
      │
      ├──► 3. Construction du prompt (contexte + question)
      │
      ├──► 4. Appel LLM (via Ollama)
      │
      ▼
Réponse générée + sources citées
```

Deux passes hors-ligne préalables : découpage des documents en chunks, puis calcul des embeddings de chaque chunk (une seule fois, à l'ingestion).

---

## Étape 0 — Préparer l'environnement

1. Installer Rust : `curl https://sh.rustup.rs -sSf | sh`
2. Installer Ollama : télécharger depuis le site officiel, puis :
   ```bash
   ollama pull llama3.1        # modèle de génération
   ollama pull nomic-embed-text # modèle d'embeddings, léger et rapide
   ```
3. Vérifier qu'Ollama tourne : `curl http://localhost:11434/api/tags`

---

## Étape 1 — Récupérer et nettoyer les documents

- Convertir le classeur Word en texte brut ou Markdown (garder les titres de section, ils serviront de métadonnées).
- Découper le contenu par sections logiques (ex: "Sécurité atelier", "Congés", "Commandes fournisseurs").
- Stocker en fichiers `.md` séparés dans un dossier `docs/` — plus simple à maintenir pour la gérante que du Word.

Pas de code ici : c'est un travail manuel/éditorial, mais crucial — la qualité du découpage conditionne la qualité des réponses.

---

## Étape 2 — Chunking (découpage en passages)

Objectif : découper chaque document en passages de ~200-400 mots, avec un léger chevauchement (overlap) pour ne pas couper une idée en deux.

```toml
# Cargo.toml (dépendances de base pour tout le projet)
[dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
reqwest = { version = "0.12", features = ["json"] }
axum = "0.7"
walkdir = "2"
```

```rust
// src/chunking.rs
pub struct Chunk {
    pub id: String,
    pub source_doc: String,
    pub text: String,
}

pub fn chunk_text(doc_name: &str, text: &str, max_words: usize, overlap: usize) -> Vec<Chunk> {
    let words: Vec<&str> = text.split_whitespace().collect();
    let mut chunks = Vec::new();
    let mut start = 0;
    let mut idx = 0;

    while start < words.len() {
        let end = (start + max_words).min(words.len());
        let chunk_text = words[start..end].join(" ");
        chunks.push(Chunk {
            id: format!("{doc_name}-{idx}"),
            source_doc: doc_name.to_string(),
            text: chunk_text,
        });
        if end == words.len() { break; }
        start += max_words - overlap;
        idx += 1;
    }
    chunks
}
```

---

## Étape 3 — Générer les embeddings via Ollama

Chaque chunk devient un vecteur numérique. On appelle l'API embeddings d'Ollama.

```rust
// src/embeddings.rs
use serde::{Deserialize, Serialize};

#[derive(Serialize)]
struct EmbedRequest<'a> {
    model: &'a str,
    prompt: &'a str,
}

#[derive(Deserialize)]
struct EmbedResponse {
    embedding: Vec<f32>,
}

pub async fn embed(client: &reqwest::Client, text: &str) -> anyhow::Result<Vec<f32>> {
    let resp = client
        .post("http://localhost:11434/api/embeddings")
        .json(&EmbedRequest { model: "nomic-embed-text", prompt: text })
        .send()
        .await?
        .json::<EmbedResponse>()
        .await?;
    Ok(resp.embedding)
}
```

---

## Étape 4 — Construire l'index vectoriel (stockage local simple)

Pour ce volume (40 pages), pas besoin d'une base vectorielle dédiée (Qdrant, etc.) — un fichier JSON en mémoire suffit largement.

```rust
// src/index.rs
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Clone)]
pub struct IndexedChunk {
    pub id: String,
    pub source_doc: String,
    pub text: String,
    pub embedding: Vec<f32>,
}

pub fn cosine_similarity(a: &[f32], b: &[f32]) -> f32 {
    let dot: f32 = a.iter().zip(b).map(|(x, y)| x * y).sum();
    let norm_a: f32 = a.iter().map(|x| x * x).sum::<f32>().sqrt();
    let norm_b: f32 = b.iter().map(|x| x * x).sum::<f32>().sqrt();
    dot / (norm_a * norm_b + 1e-8)
}

pub fn search<'a>(
    query_embedding: &[f32],
    index: &'a [IndexedChunk],
    top_k: usize,
) -> Vec<&'a IndexedChunk> {
    let mut scored: Vec<(&IndexedChunk, f32)> = index
        .iter()
        .map(|c| (c, cosine_similarity(query_embedding, &c.embedding)))
        .collect();
    scored.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());
    scored.into_iter().take(top_k).map(|(c, _)| c).collect()
}
```

**Script d'ingestion** (`src/bin/ingest.rs`) : lit `docs/`, chunk, embed, sauvegarde `index.json`. Se relance à chaque mise à jour des procédures (une commande manuelle, pas besoin d'automatiser tout de suite).

---

## Étape 5 — Construire le prompt et appeler le LLM

```rust
// src/generate.rs
use serde::{Deserialize, Serialize};

#[derive(Serialize)]
struct ChatRequest<'a> {
    model: &'a str,
    prompt: String,
    stream: bool,
}

#[derive(Deserialize)]
struct ChatResponse {
    response: String,
}

pub async fn answer(client: &reqwest::Client, question: &str, context: &str) -> anyhow::Result<String> {
    let prompt = format!(
        "Tu es l'assistant interne de la Menuiserie Dubois. Réponds uniquement à partir du contexte ci-dessous. \
        Si l'information n'y figure pas, dis-le clairement.\n\n\
        CONTEXTE:\n{context}\n\n\
        QUESTION: {question}\n\
        RÉPONSE:"
    );

    let resp = client
        .post("http://localhost:11434/api/generate")
        .json(&ChatRequest { model: "llama3.1", prompt, stream: false })
        .send()
        .await?
        .json::<ChatResponse>()
        .await?;
    Ok(resp.response)
}
```

---

## Étape 6 — Exposer une API HTTP (axum)

```rust
// src/main.rs
use axum::{routing::post, Json, Router, extract::State};
use std::sync::Arc;

#[derive(serde::Deserialize)]
struct AskRequest { question: String }

#[derive(serde::Serialize)]
struct AskResponse { answer: String, sources: Vec<String> }

async fn ask(
    State(state): State<Arc<AppState>>,
    Json(payload): Json<AskRequest>,
) -> Json<AskResponse> {
    let q_embedding = embeddings::embed(&state.client, &payload.question).await.unwrap();
    let top_chunks = index::search(&q_embedding, &state.index, 3);
    let context = top_chunks.iter().map(|c| c.text.clone()).collect::<Vec<_>>().join("\n---\n");
    let sources = top_chunks.iter().map(|c| c.source_doc.clone()).collect();

    let answer = generate::answer(&state.client, &payload.question, &context).await.unwrap();
    Json(AskResponse { answer, sources })
}

#[tokio::main]
async fn main() {
    // charger index.json au démarrage, construire AppState, lancer axum sur 0.0.0.0:3000
    let app = Router::new().route("/ask", post(ask));
    axum::serve(tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap(), app).await.unwrap();
}
```

---

## Étape 7 — Interface minimale

Pas besoin de React ici : une page HTML statique avec un `<input>` et un appel `fetch('/ask')` en JS vanilla suffit pour le MVP. Servie directement par axum via `tower-http::services::ServeDir`.

---

## Étape 8 — Tests avec de vrais employés

- Faire tester par 2-3 personnes avec leurs vraies questions.
- Noter les cas où la réponse est fausse ou incomplète → souvent un problème de chunking (mauvais découpage) plutôt qu'un problème de modèle.
- Ajuster `max_words`/`overlap` et re-lancer `ingest`.

---

## Étape 9 — Déploiement chez le client

- Compiler en release (`cargo build --release`) → binaire unique + `index.json` + Ollama installé sur un mini-PC du bureau.
- Lancer au démarrage via un service systemd simple.
- Aucune donnée ne sort de l'entreprise : argument clé pour la gérante.

---

## Pistes d'amélioration (phase 2, à facturer séparément)

- Ré-ingestion automatique quand un document change (watcher de fichiers).
- Historique de conversation (mémoire courte) si les employés posent des questions de suivi.
- Authentification basique si plusieurs services doivent avoir des réponses différentes.
