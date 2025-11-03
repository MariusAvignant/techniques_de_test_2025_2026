

## 1) Tests de comportement (fonctionnels)

### 1.1 Entrée côté Client -> Triangulator (ce que Triangulator reçoit)
- Requête valide : JSON avec point_set_id -> Triangulator doit appeler le PointSetManager avec cet ID puis répondre 200 + octet‑stream.
- Entrées invalides : point_set_id manquant/vidé/mauvais type -> Triangulator répond 400 JSON (message clair), n’appelle pas l’amont.
- Corps non‑JSON : Triangulator répond 400.

### 1.2 Appel sortant Triangulator -> PointSetManager (ce que Triangulator envoie) 
- URL/Verbe : construit GET /pointsets/{id}/binary (id reçu) avec timeout raisonnable.
- Gestion erreurs amont :
  - 404/5xx/timeout côté PSM -> Triangulator répond 502 JSON.
  - Binaire amont invalide -> Triangulator répond 502 « invalid PointSet from upstream ».

### 1.3 Traitement interne (ce que Triangulator fait)
- Décodage PointSet : buffer correct -> liste de points ; buffer incorrect -> erreur contrôlée.
- Triangulation (version simple) :
  - Si moins de 3 points -> 0 triangle.
  - Si 3 points -> 1 triangle.
  - Si plus de 3 points -> produire des triangles corrects (pas d’indices invalides, pas de triangles avec aire ≈ 0, résultat cohérent).
  - Cas non valides (ex : points tous alignés / doublons) -> pas de crash, résultat vide ou minimal acceptable.
- Encodage Triangles : format binaire correct ; indices hors bornes -> erreur contrôlée.
### 1.4 Sortie Triangulator -> Client (ce que Triangulator envoie)
- Succès : 200 + Content-Type: application/octet-stream, binaire décodable en Triangles.
- Erreurs : 400 (entrée), 501 (non implémenté), 502 (amont/codec amont), messages JSON sobres.

### 1.5 Propriétés transverses
- Idempotence : même point_set_id (même PSM) -> même binaire de réponse.
- Traçabilité minimale : pas d’exception brute dans les réponses.

---

## 2) Tests de performance
*(balises @pytest.mark.perf, exclus par défaut)*

### 2.1 Codec
- Encodage/Décodage PointSet : 10⁴–10⁵ points — mesurer latence (ms) et ratio octets/point.
- Stabilité mémoire : pas de pic anormal (sanity via taille binaire attendue).

### 2.2 Triangulation
- Jeux croissants : 1e3, 5e3, 1e4 points (si faisable).
- Mesures : temps total et par point ; absence de timeouts côté PSM mock.
- Seuils : à fixer après premier run (fail si > X× baseline).

### 2.3 API bout‑en‑bout
- Latence POST /triangulations : P50/P95 sur petits et moyens jeux (PSM mock in‑memory).

---

## 3) Qualité de code
- Lint : ruff sans erreurs bloquantes.
- Couverture :
  - Phase TDD initiale (tests avant code) : *mesure informative seulement*, pas d’objectif chiffré (le code est encore des stubs/placeholder).
  - Après première implémentation fonctionnelle : objectif ≥ 90% global, avec priorité sur le codec et le mapping d’erreurs.
- Doc : pdoc3 (quand les modules existent) ; build facultatif pendant la phase TDD initiale.
- CI locale : ordre recommandé lint -> unit_test ; ajouter coverage et doc après création du code.

---

## 4) Jeux de test (références rapides)
- PointSet : vide, 1 point, 3 points, 4+ points ; valeurs extrêmes ; buffers corrompus.
- Triangles : 0/1/N faces ; indices hors bornes.
- PSM mock : id connu (binaire valide), id inconnu (404), panne (5xx/timeout), binaire corrompu.

---

## 5) Scénarios clés (résumé)
1) OK : Client->Triangulator (id) -> Triangulator->PSM (GET) -> Triangulator décode/triangule/encode -> Client reçoit 200 + octet‑stream. 
2) Entrée invalide : Triangulator ne contacte pas PSM et répond 400. 
3) PSM échoue : Triangulator mappe en 502. 
4) N≥4 (phase initiale) : Triangulator répond 501.

