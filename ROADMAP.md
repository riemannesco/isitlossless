# Roadmap — isitlossless

Détecter les faux FLAC (MP3/AAC/Opus réencodés en FLAC), estimer le codec et le débit d'origine, avec un score de confiance calibré. CLI open source multiplateforme.

## Règles du jeu

- **Tout le code de `src/` et des tests est écrit par moi.** Claude relit, questionne, pointe la spec, explique les erreurs. Claude peut écrire l'outillage hors produit (CI, scripts de dataset).
- Chaque fonctionnalité : issue → branche → tests → implémentation → `fmt` + `clippy` → PR → revue → merge.
- Conventional Commits (`feat:`, `fix:`, `test:`, `chore:`, `docs:`…).
- Chaque phase se termine par une release GitHub taguée.

## Décisions actées

| # | Décision |
|---|----------|
| D1 | Claude n'écrit pas le code produit ni les tests |
| D2 | Ordre : décodeur → heuristique spectrale (baseline) → classifieur qui doit battre la baseline |
| D3 | Entrée FLAC uniquement jusqu'à v0.3 ; frontière nette `décodeur → PCM` |
| D4 | Oracle de test : PCM de ffmpeg, comparé échantillon par échantillon + MD5 de STREAMINFO |
| D5 | Un seul crate lib + bin ; workspace seulement si une vraie frontière apparaît |
| D6 | CI fmt/clippy/test sur Linux/macOS/Windows, double licence MIT/Apache-2.0 |
| D7 | Sources du dataset : provenance tracée dans un manifeste ; on publie le générateur, pas la musique |
| D8 | Minimiser les faux positifs ; 3 sorties (`lossless probable` / `transcodé (source estimée)` / `indéterminé`) ; confiance calibrée |
| D9 | Pas d'échéance : on privilégie la profondeur à la vitesse |

---

## Phase 0 — Hygiène (avant toute ligne de logique)

- [ ] `README.md` minimal : problème, statut « WIP », licence
- [ ] `LICENSE-MIT` + `LICENSE-APACHE`, champ `license = "MIT OR Apache-2.0"` dans `Cargo.toml`
- [ ] Métadonnées `Cargo.toml` : `description`, `repository`, `rust-version`
- [ ] Section `[lints]` dans `Cargo.toml` (ex. `clippy::pedantic` en `warn`, `unsafe_code = "forbid"`)
- [ ] `src/lib.rs` + `src/main.rs` mince
- [ ] `.github/workflows/ci.yml` : fmt, clippy `-D warnings`, test — matrice 3 OS
- [ ] Premier commit `chore: initial project setup`, push sur GitHub
- [ ] Protection de `main` : merge uniquement via PR avec CI verte

**Questions à te poser :** pourquoi `-D warnings` en CI mais pas en local ? À quoi sert `rust-version` ? Pourquoi `Cargo.lock` est-il commité pour un binaire ?

---

## Phase 1 — Décodeur FLAC (v0.1)

Référence : [RFC 9639](https://www.rfc-editor.org/rfc/rfc9639) (spec FLAC officielle).

Fixtures : génère de petits fichiers avec `flac` / `ffmpeg` (16 et 24 bits, mono/stéréo, `-0` à `-8`, blocksizes variés, signal silencieux, sinus, bruit). Garde-les petits et libres de droits (générés) pour la CI.

| Étape | Contenu | Notion Rust travaillée |
|-------|---------|------------------------|
| 1.1 | `BitReader` : lire n bits (1–64), entiers signés, codes unaires | slices, emprunts, `u64`/décalages, tests unitaires |
| 1.2 | Marqueur `fLaC` + en-têtes de blocs metadata, parsing de `STREAMINFO` (ignorer proprement les autres) | `struct`, `enum`, `Result`, type d'erreur maison |
| 1.3 | En-tête de frame : sync code, blocksize, sample rate, entier UTF-8 « coded number », CRC-8 | pattern matching, `TryFrom` |
| 1.4 | Sous-frames `CONSTANT` et `VERBATIM` → premier fichier décodé de bout en bout | `Vec`, ownership des buffers |
| 1.5 | Résidu codé Rice (partitions, paramètre d'échappement) | boucles serrées, débordements |
| 1.6 | Sous-frames `FIXED` (ordres 0–4) | — |
| 1.7 | Sous-frames `LPC` (coefficients quantifiés, shift) | **`i64` pour la prédiction** : pourquoi ? |
| 1.8 | Décorrélation stéréo (left/side, right/side, mid/side) + wasted bits | — |
| 1.9 | CRC-16 de frame, MD5 du PCM complet (`--verify`) | crate `md-5`, traits `Read` |
| 1.10 | Test d'intégration oracle ffmpeg sur tout le corpus | `tests/`, `std::process::Command` |
| 1.11 | Bench `cargo bench` (criterion) vs `flac -d` | profiling, perfs |

**Release v0.1 :** `isitlossless decode in.flac out.wav` + `--verify`.

**Décisions à trancher en début de phase :** erreurs (`thiserror` dans la lib, `anyhow` dans le bin ?), API publique du décodeur (itérateur de frames ou tout en mémoire ?), parsing d'arguments (`clap` derive ?).

---

## Phase 2 — Analyse spectrale + baseline heuristique (v0.2)

| Étape | Contenu |
|-------|---------|
| 2.1 | Fenêtrage (Hann), STFT, spectre moyen en dB |
| 2.2 | Export du spectrogramme en PNG (`--spectrogram out.png`) : indispensable pour vérifier à l'œil |
| 2.3 | Détection de la fréquence de coupure (chute brutale + « shelf » au-dessus) |
| 2.4 | Règles heuristiques coupure → estimation (≈16 kHz → MP3 128k, ≈19–20 kHz → MP3 V0/320, etc.) |
| 2.5 | Script de génération du dataset (ffmpeg : MP3 CBR/VBR, AAC, Opus, Vorbis × débits → FLAC) |
| 2.6 | Évaluation : matrice de confusion, taux de faux positifs sur les vrais FLAC |

**Release v0.2 :** `isitlossless check *.flac`, verdict en 3 états.

**Décisions à trancher :** FFT écrite à la main ou `rustfft`/`realfft` ? Analyse du fichier entier ou échantillonnage de segments (perf) ? Format du rapport (texte, `--json`) ?

**Point de vigilance (D7) :** les FLAC que tu télécharges peuvent eux-mêmes être des faux. C'est précisément ce que l'outil détecte. Avant d'entrer dans le dataset, chaque source passe par la baseline v0.2 + un coup d'œil au spectrogramme, et sa provenance est notée dans le manifeste.

---

## Phase 3 — Classifieur (v0.3)

| Étape | Contenu |
|-------|---------|
| 3.1 | Étude des codecs : passe-bas par encodeur (LAME, FhG, ffmpeg-aac, fdk…), blocs MDCT, trous spectraux du modèle psychoacoustique |
| 3.2 | Extraction des features **en Rust** (une seule implémentation, partagée entraînement / inférence) |
| 3.3 | Split train/test **par album ou artiste**, jamais par morceau (sinon fuite de données) |
| 3.4 | Entraînement, comparaison à la baseline sur le même jeu de test |
| 3.5 | Calibration (diagramme de fiabilité, Platt / isotonic) |
| 3.6 | Modèle embarqué dans le binaire, inférence en Rust |

**Release v0.3 :** `transcodé — source estimée : MP3 ~192 kbps (confiance 0.87)`.

**Décisions à trancher :** entraînement en Python (scikit-learn/PyTorch → ONNX → `tract`) ou tout en Rust (`linfa`, `burn`) ? Modèle simple sur features à la main (gradient boosting) ou CNN sur spectrogramme ?

---

## Phase 4 — Distribution (v1.0)

- [ ] Binaires précompilés (`cargo-dist`) pour Linux/macOS/Windows
- [ ] Publication sur crates.io
- [ ] README avec démo (GIF / spectrogrammes avant/après) et résultats chiffrés
- [ ] Article de blog : « J'ai écrit un décodeur FLAC pour apprendre Rust »
