---
name: seguridad-repo
description: >
  Monta vigilancia automática de seguridad y calidad en un repositorio de GitHub:
  Dependabot (dependencias vulnerables), CI con tests (GitHub Actions) y Semgrep
  (análisis estático del código propio). Todo en versión gratuita. Actívala cuando
  el usuario diga "monta el CI", "añade Dependabot", "quiero que se ejecuten los
  tests solos", "análisis estático", "Semgrep", "seguridad del repo", "auditar
  dependencias", "que me avise si una librería tiene un fallo", o cuando arranque
  un repo nuevo que vaya a contener código con tests. Cubre PHP/WordPress y
  Node/JavaScript. NO cubre la seguridad de webs de terceros ya desplegadas
  (WordPress de un cliente): eso es otra herramienta.
---

# Montar vigilancia automática en un repositorio

## Qué monta y qué NO

Tres piezas, todas gratuitas en repos privados:

| Pieza | Vigila | Actúa |
|---|---|---|
| **Dependabot** | Librerías de terceros que usa el proyecto | Cuando se publica un aviso de seguridad |
| **CI (GitHub Actions)** | Que los tests sigan pasando | En cada push y cada PR |
| **Semgrep** | El código propio, por patrones peligrosos | En cada push y cada PR |

**No cubre:** la seguridad de un sitio ya desplegado (core, plugins y temas de
terceros de un WordPress ajeno). Eso es un problema distinto y otra herramienta.

---

## Antes de tocar nada

### 1. Reconocimiento

```bash
git remote -v && git branch --show-current && git status --short
gh auth status          # hacen falta los scopes `repo` y `workflow`
ls .github/workflows/ 2>/dev/null   # ¿ya hay algo montado?
```

Si ya existen workflows, **leerlos antes de escribir nada**. No duplicar ni pisar.

### 2. Detectar ecosistemas

- `composer.json` → PHP
- `package.json` → Node
- Ambos → dos entradas en Dependabot y, probablemente, dos jobs de tests

Mirar si hay dependencias **de producción** o solo de desarrollo. Si `require`
está prácticamente vacío, decirlo: Dependabot aportará poco en ese repo. No
prometer valor que no va a llegar.

### 3. Correr los tests en local PRIMERO

```bash
vendor/bin/phpunit        # PHP
npm test                  # Node
```

**Esto no es opcional.** Si los tests ya fallan antes de montar el CI, el rojo
posterior confundirá el montaje con deuda previa. Verde en local antes de seguir.

### 4. Determinar la versión mínima del lenguaje — el paso que más se falla

**La versión de destino la fija el entorno donde corre el código, NO la máquina
de desarrollo.** Es habitual desarrollar en una versión muy por delante de la de
producción y no enterarse hasta el despliegue.

Para un plugin de WordPress:

```bash
grep -i 'Requires PHP' *.php readme.txt
grep '"php"' composer.json
php -v                                   # la de la máquina, para contrastar
```

Y comprobar qué usa el código de verdad:

```bash
grep -rnE 'enum |readonly |: never\b' includes/   # PHP 8.1+
grep -rnE 'match\s*\(|\?->' includes/             # PHP 8.0+
grep -rnE '__construct\([^)]*(private|protected|public)' includes/  # 8.0+
```

Tres resultados posibles, y hay que decir cuál es:

- **Declara más de lo que usa** → está cerrando entornos gratis. Proponer bajarlo.
- **Declara menos de lo que usa** → bomba de relojería. Es un fatal error de
  parseo en el entorno viejo, no un aviso. Urgente.
- **Coincide** → la matriz de CI se construye alrededor de esa versión.

**Preguntar al usuario a qué versión va destinado el proyecto si no está claro.**
No suponerlo por la versión de la máquina.

---

## Los ficheros

### `.github/dependabot.yml`

```yaml
version: 2

updates:
  - package-ecosystem: "composer"   # o "npm"
    directory: "/"
    schedule:
      interval: "monthly"
    open-pull-requests-limit: 5
    commit-message:
      prefix: "deps"

  # Las actions son código de terceros con acceso al repo. Se mantienen igual.
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "monthly"
    open-pull-requests-limit: 3
    commit-message:
      prefix: "ci"
```

**Mensual, no semanal.** En un proyecto de una sola persona, un PR de
dependencias cada semana se ignora y se convierte en ruido.

### `.github/workflows/tests.yml` (ejemplo PHP)

```yaml
name: Tests

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: tests-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # Verifica que el codigo PARSEA en la version minima declarada. Necesario
  # cuando la bateria de tests no puede correr ahi (PHPUnit 10 exige PHP 8.1+).
  # Un error de sintaxis en produccion tumba el sitio entero.
  lint:
    name: Sintaxis en PHP 8.0
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: shivammathur/setup-php@v2
        with:
          php-version: '8.0'
          coverage: none
      - run: |
          find . -path ./vendor -prune -o -path ./dist -prune -o -name '*.php' -print \
            | xargs -n1 -P4 php -l

  phpunit:
    name: PHPUnit (PHP ${{ matrix.php }})
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      max-parallel: 1
      matrix:
        php: ['8.1', '8.3']

    env:
      COMPOSER_AUTH: '{"github-oauth":{"github.com":"${{ secrets.GITHUB_TOKEN }}"}}'

    steps:
      - uses: actions/checkout@v7

      - uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ matrix.php }}
          coverage: none

      # Se cachea la cache de descargas de composer, NO vendor/.
      # vendor/ depende de la version de PHP; los zips descargados no.
      # Con clave por version, cada rama de la matriz redescarga todo.
      - uses: actions/cache@v6
        with:
          path: ~/.cache/composer/files
          key: composer-files-${{ hashFiles('composer.lock') }}
          restore-keys: composer-files-

      # codeload.github.com devuelve HTTP 429 a las IPs compartidas de los
      # runners, incluso en una instalacion unica con cache fria. El token no
      # lo evita. El reintento es necesario, no cosmetico.
      - name: Instalar dependencias
        run: |
          for intento in 1 2 3; do
            if composer install --prefer-dist --no-progress --no-interaction; then
              exit 0
            fi
            echo "Intento $intento fallido. Esperando $((intento * 30))s..."
            sleep $((intento * 30))
          done
          exit 1

      - run: vendor/bin/phpunit
```

Para Node, la estructura es la misma cambiando `setup-php` por
`actions/setup-node@v4` (con `cache: 'npm'`, que ya gestiona la caché) y el
comando por `npm ci && npm test`.

### `.github/workflows/semgrep.yml`

```yaml
name: Semgrep

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

permissions:
  contents: read

jobs:
  analisis:
    name: Analisis estatico
    runs-on: ubuntu-latest

    # Imagen oficial en contenedor: el unico paso de terceros es checkout.
    container:
      image: semgrep/semgrep

    # PRIMERA VEZ en un repo con codigo ya escrito: descomentar esta linea.
    # continue-on-error: true

    steps:
      - uses: actions/checkout@v7
      - run: |
          semgrep scan \
            --error \
            --config=p/php \
            --config=p/security-audit \
            --config=p/secrets \
            --config=r/php.wordpress-plugins \
            --metrics=off \
            --exclude=vendor \
            --exclude=dist \
            --exclude=tests
```

**Packs por tipo de proyecto:**

| Proyecto | `--config` |
|---|---|
| PHP genérico | `p/php`, `p/security-audit`, `p/secrets` |
| Plugin/tema WordPress | los anteriores + `r/php.wordpress-plugins` |
| Node/JS/TS | `p/javascript`, `p/typescript`, `p/security-audit`, `p/secrets` |

`r/php.wordpress-plugins` trae reglas de CSRF, SQL injection, comprobación de
permisos, manipulación e inclusión de ficheros, SSRF, redirección abierta,
inyección de objetos y ejecución de código y comandos.

**`--error` es obligatorio para que bloquee.** Sin ese flag, `semgrep scan`
termina con código 0 aunque encuentre hallazgos, y quitar `continue-on-error`
no sirve de nada.

---

## Informativo primero, bloqueante después

En un repo con código ya escrito, la primera pasada de Semgrep suele sacar
hallazgos. Si eso deja el CI en rojo desde el día uno, se acaba desactivando.

1. Arrancar con `continue-on-error: true`.
2. Triar: arreglar lo real; silenciar lo que no aplica con `// nosemgrep` **y una
   justificación al lado**.
3. Cuando esté limpio, quitar `continue-on-error`.

**Excepción:** si la primera pasada da 0 hallazgos, no hay nada que triar y se
puede arrancar bloqueante directamente. Decirlo y proponerlo.

---

## Probar el montaje

**`workflow_dispatch` no sirve para una rama nueva:** exige que el workflow ya
exista en la rama por defecto. La forma de probar es abrir un PR.

```bash
git checkout -b ci/seguridad
git add .github && git commit -m "ci: tests, Semgrep y Dependabot"
git push -u origin ci/seguridad
gh pr create --title "..." --body "..."

# Esperar y mirar
gh pr checks <n>
```

**Leer el log, no solo el tick verde.** Comprobar como mínimo:

```bash
# ¿Cuántas reglas cargó Semgrep de verdad?
gh api repos/OWNER/REPO/actions/jobs/<id>/logs | grep -E 'Ran [0-9]+ rules'

# ¿Hizo falta el reintento? ¿Se restauró la caché?
gh api repos/OWNER/REPO/actions/jobs/<id>/logs | grep -iE '429|Intento|Cache'
```

Un check verde puede esconder que Semgrep corrió con cuatro reglas, o que la
caché nunca se usa y cada ejecución descarga todo.

---

## Cerrar

```bash
gh pr merge <n> --merge --delete-branch
git checkout main && git pull --ff-only && git remote prune origin

gh api -X PUT repos/OWNER/REPO/vulnerability-alerts       # alertas
gh api -X PUT repos/OWNER/REPO/automated-security-fixes   # PRs de arreglo
```

Verificar que Dependabot quedó activo (`200` = activo, `404` = no):

```bash
gh api repos/OWNER/REPO/dependabot/alerts -i | head -1
```

Y anotar la entrada en el `LOG.md` del repo.

---

## Errores conocidos

| Síntoma | Causa | Solución |
|---|---|---|
| `HTTP 429` de `codeload.github.com` | Límite de las IPs compartidas de los runners. `COMPOSER_AUTH` no lo evita | Reintento con espera creciente |
| El segundo job de la matriz redescarga todo | Clave de caché con la versión del lenguaje dentro | Cachear la caché de descargas, con clave compartida |
| `workflow_dispatch` da 404 | El workflow no está en la rama por defecto | Abrir un PR |
| Semgrep verde pero no detecta nada | Falta `--error`, o los packs no aplican al lenguaje | Añadir `--error`; verificar el recuento de reglas en el log |
| Avisos de Node 20 obsoleto | Actions en versiones antiguas | Subir a la última (`gh api repos/actions/checkout/releases/latest`) |
| Fallos raros y dispersos | Puede ser una caída de GitHub | `curl -s https://www.githubstatus.com/api/v2/status.json` |
| `Base branch was modified` al fusionar | Puede ser otra sesión, **o una caída del proveedor**. No suponerlo: en el caso real que originó esta nota el commit "nuevo" de la base era de tres semanas antes y lo había traído el propio `git pull` de apertura | Mirar la **fecha** del commit de la base (`git log -1 --format=%ad`), no solo su presencia, y el estado del proveedor. Si hay solapamiento real de ficheros, integrar la base en la rama; si no, reintentar. Nunca forzar |

---

## Lo que NO se monta, y por qué

- **CodeQL / GitHub Code Security** — no soporta PHP, y es de pago en repos
  privados.
- **GitHub Secret Protection** — de pago, y en repos de cuenta personal ni
  siquiera está disponible (exige organización). Alternativa gratis: `gitleaks`
  como hook de pre-commit, que además bloquea antes de que el commit exista.
- **Servicios de revisión de PR con IA de pago** — si ya hay revisión de código
  con IA en el entorno de desarrollo, se duplica.

Estas exclusiones son por **disponibilidad y encaje**, no por precio. Si una
opción de pago cuesta poco y aporta algo que se va a usar de verdad, se plantea
con la cifra delante y decide el usuario.
