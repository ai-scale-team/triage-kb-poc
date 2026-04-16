# Triage KB · Guía de despliegue

App de edición de casos que usa un repositorio como backend compartido: cada caso es un archivo JSON, cada guardado es un commit.

## Arquitectura

```
operadores (navegador) ──► API del hosting ──► repo privado
                                                 │
                                                 ├── cases/CASE-2026-001.json
                                                 ├── cases/CASE-2026-002.json
                                                 └── index.html  ◄── lo sirve Pages
                                                                      │
                                                                      │  git pull / API
                                                                      ▼
                                                           proceso ingesta → MongoDB
```

- La app es **un solo archivo HTML**.
- Cada caso es **un archivo JSON** en `cases/`.
- Cada guardado es **un commit** (historial + autoría automática).
- El token vive **solo en el navegador** de cada operador.

---

## Opción A · GitHub Pages (requiere plan Pro/Team para repos privados)

### 1. Crear el repo

```bash
# En GitHub: crea un repo privado, ej. "triage-kb"
git clone git@github.com:TU-ORG/triage-kb.git
cd triage-kb
mkdir cases && touch cases/.gitkeep
cp /ruta/a/triage-kb.html index.html
git add . && git commit -m "initial: kb app + empty cases folder"
git push
```

### 2. Activar GitHub Pages

1. Repo → **Settings → Pages**
2. Source: `Deploy from a branch`
3. Branch: `main` / root → **Save**
4. Espera ~1 min. URL: `https://TU-ORG.github.io/triage-kb/`

> GitHub Pages para repos privados requiere **GitHub Pro** (usuarios) o **Team** (organizaciones).

### 3. Personal Access Token (cada operador)

**Fine-grained (recomendado):**

1. [github.com/settings/tokens?type=beta](https://github.com/settings/tokens?type=beta) → **Generate new token**
2. Resource owner: tu organización
3. Repository access: *Only select repositories* → `triage-kb`
4. Permissions → **Contents: Read and write**
5. Expiration: 90 días
6. Copia el token (`github_pat_...`) — solo se muestra una vez.

**Classic (alternativa):**

1. [github.com/settings/tokens](https://github.com/settings/tokens) → **Generate new token (classic)**
2. Scope: `repo`
3. Copia el token (`ghp_...`).

### 4. Primera conexión (cada operador)

1. Abrir la URL → se abre el modal **⚙ Config**.
2. Rellenar:
   - **owner**: `TU-ORG`
   - **repo**: `triage-kb`
   - **branch**: `main`
   - **cases path**: `cases`
   - **token**: el PAT
   - **author name**: tu nombre (aparecerá en los commits)
3. **Probar y guardar**.

La config se guarda en `localStorage`. Si cambias de navegador hay que volver a meterla.

---

## Opción B · GitLab Pages (gratis con repos privados) — para la migración futura

Si os aprueban el repo pago de GitLab, la app funciona igual pero habrá que hacer **un cambio mínimo** en el HTML para que hable con la API de GitLab en lugar de la de GitHub (ver sección "Adaptar el HTML a GitLab" más abajo).

### 1. Crear el repo y la CI para Pages

En GitLab, crear proyecto privado `triage-kb`, subir el HTML y `cases/.gitkeep`, y añadir un `.gitlab-ci.yml` en la raíz:

```yaml
pages:
  stage: deploy
  script:
    - mkdir -p public
    - cp index.html public/index.html
  artifacts:
    paths:
      - public
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

Al primer push a `main`, GitLab ejecuta el job y publica en `https://TU-GRUPO.gitlab.io/triage-kb/`.

### 2. Controlar el acceso

GitLab Pages hereda los permisos del proyecto por defecto: **Settings → General → Visibility → Pages** = `Only project members`. Así solo los miembros del proyecto (los 2 operadores) pueden abrir la URL tras loguearse en GitLab.

### 3. Personal Access Token en GitLab

Cada operador:

1. **User Settings → Access Tokens**
2. Scopes: marca solo **`api`** (o `read_repository` + `write_repository` si quieres ser más fino)
3. Expiration: 90 días
4. Copia el token (`glpat-...`).

### 4. Adaptar el HTML a GitLab

La API de GitLab es distinta a la de GitHub. Cuando migres, hay que cambiar dos cosas en el HTML:

**a) El modal de config** debe pedir `gitlab host`, `project path` y `token` en lugar de `owner/repo`.

**b) Las funciones `gh*` deben reemplazarse por llamadas equivalentes de GitLab.** Equivalencias clave:

| Operación | GitHub | GitLab |
|---|---|---|
| Listar archivos | `GET /repos/{o}/{r}/contents/{path}` | `GET /projects/{id}/repository/tree?path={path}` |
| Leer archivo | `GET /repos/{o}/{r}/contents/{path}` (devuelve base64) | `GET /projects/{id}/repository/files/{url-encoded-path}/raw?ref={branch}` |
| Crear/actualizar | `PUT /repos/.../contents/{path}` (con `sha`) | `POST/PUT /projects/{id}/repository/files/{url-encoded-path}` |
| Eliminar | `DELETE /repos/.../contents/{path}` (con `sha`) | `DELETE /projects/{id}/repository/files/{url-encoded-path}` |
| Auth header | `Authorization: Bearer ghp_...` | `PRIVATE-TOKEN: glpat-...` |

GitLab no necesita `sha` para update/delete (usa el `branch` + `last_commit_id` opcional como control de concurrencia). Y los paths se envían URL-encoded (`cases/CASE-2026-001.json` → `cases%2FCASE-2026-001.json`).

**Cuando tengas luz verde para migrar, pídemelo y te genero la versión GitLab del HTML ya adaptada** — no tiene sentido hacerlo ahora porque quizás acaban aprobándote el plan Pro de GitHub y no necesitas migrar.

---

## Flujo de trabajo diario (idéntico en ambas opciones)

| Acción | En la app | En el repo |
|---|---|---|
| Crear caso | `+ Nuevo caso` → rellenar → `✓ Commit al repo` | commit `add(kb): CASE-YYYY-NNN` |
| Editar caso | click en la lista → modificar → `✓ Commit al repo` | commit `update(kb): ...` |
| Eliminar | `× Eliminar` | commit `remove(kb): ...` |
| Renombrar | cambiar `case_id` y guardar | commit `chore(kb): rename X → Y` |
| Ver cambios de otro | botón `↻` en la sidebar | re-fetch de la API |
| Importar en bulk | `↓ Importar JSON` → pegar o subir archivos | un commit por caso |

---

## Consumir los casos desde tu proceso de ingesta

### Opción 1 — `git pull` + script
```bash
cd /path/to/triage-kb && git pull
for f in cases/*.json; do
  mongoimport --db triage --collection kb --file "$f" \
    --mode=upsert --upsertFields=case_id
done
```

### Opción 2 — API desde Python (GitHub)
```python
import requests
HEADERS = {"Authorization": f"Bearer {TOKEN}",
           "Accept": "application/vnd.github+json"}

r = requests.get("https://api.github.com/repos/TU-ORG/triage-kb/contents/cases",
                 headers=HEADERS).json()

for f in r:
    data = requests.get(f["download_url"], headers=HEADERS).json()
    if data.get("metadata", {}).get("validated"):
        collection.replace_one({"case_id": data["case_id"]}, data, upsert=True)
```

### Opción 3 — GitHub Actions (ingesta automática en cada commit)
```yaml
# .github/workflows/ingest.yml
on:
  push:
    branches: [main]
    paths: ['cases/**']
jobs:
  ingest:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - run: pip install pymongo
      - run: python scripts/ingest_to_mongo.py
        env:
          MONGO_URI: ${{ secrets.MONGO_URI }}
```

(En GitLab sería equivalente con `.gitlab-ci.yml` en lugar de Actions.)

---

## Cambiar el schema en el futuro

1. **Editar el HTML** → sustituir el objeto `DEFAULT_SCHEMA` dentro del `<script>` → push.
2. **Subir un `schema.json` en la raíz del repo** → la app lo carga automáticamente al sincronizar. Así el schema vive versionado junto a los casos sin tocar el HTML.

---

## Seguridad

- El repo debe ser **privado**.
- El token vive solo en el navegador del operador — se envía directo a la API.
- Si un token se filtra, revócalo — no afecta al resto de operadores.
- Fine-grained PATs con expiración a 90 días = buena higiene.
- Si alguien abandona el equipo: revocar su token. La app no requiere tocar nada.
