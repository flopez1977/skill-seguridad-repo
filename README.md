# skill-seguridad-repo

Skill de [Claude Code](https://claude.com/claude-code) que monta **vigilancia automática de seguridad y calidad** en un repositorio de GitHub. Todo con herramientas gratuitas, también en repos privados.

Le dices en qué repo estás y monta tres piezas:

| Pieza | Qué vigila | Cuándo actúa |
|---|---|---|
| **Dependabot** | Las librerías de terceros que usa tu proyecto | Cuando se publica un aviso de seguridad que te afecta |
| **CI (GitHub Actions)** | Que tus tests sigan pasando | En cada push y cada Pull Request |
| **Semgrep** | Tu propio código, por patrones peligrosos | En cada push y cada Pull Request |

Cubre **PHP/WordPress** (composer, PHPUnit) y **Node/JavaScript** (npm).

---

## El problema que resuelve

Si trabajas solo, o en proyectos que terminas y no vuelves a abrir en meses, no hay nadie mirando:

- Una librería que usas puede tener un agujero de seguridad publicado hace tres semanas.
- Un cambio pequeño puede haber roto algo que funcionaba, y tus tests solo corren cuando te acuerdas.
- Puedes haber escrito una consulta SQL sin proteger, o un formulario de administración sin comprobar permisos.

Nada de eso avisa solo. Todo eso se descubre tarde.

---

## Instalación

```bash
git clone https://github.com/flopez1977/skill-seguridad-repo.git ~/.claude/skills/seguridad-repo
```

Claude Code carga las skills de `~/.claude/skills/` al arrancar la sesión.

---

## Uso

Desde una sesión de Claude Code, en la carpeta de tu proyecto:

```
monta el CI en este repo
```

O cualquier variante: *"añade Dependabot"*, *"quiero que se ejecuten los tests solos"*, *"análisis estático"*, *"que me avise si una librería tiene un fallo"*.

También se activa sola al arrancar un repo nuevo que vaya a llevar tests.

### Requisitos

- `gh` (GitHub CLI) autenticado, con los permisos `repo` y `workflow`
- Un proyecto con `composer.json` o `package.json`
- Tests que pasen en local (la skill lo comprueba antes de montar nada)

---

## Qué hace la skill, en orden

1. **Reconoce el repo**: remoto, rama, workflows que ya existan. Si ya hay algo montado, lo lee antes de escribir.
2. **Detecta los ecosistemas** presentes y avisa si Dependabot va a aportar poco (por ejemplo, un proyecto sin dependencias de producción).
3. **Corre los tests en local.** Si ya fallan, para: un CI rojo desde el día uno se confunde con deuda previa y se acaba desactivando.
4. **Determina la versión mínima del lenguaje** y compara lo declarado con lo que el código usa de verdad.
5. **Escribe los tres ficheros** de configuración.
6. **Abre un PR y verifica los checks leyendo el log**, no solo el tick verde.
7. **Fusiona y activa Dependabot.**

---

## La parte que más se falla: la versión mínima del lenguaje

**La versión de destino la fija el entorno donde corre el código, no tu máquina.**

Es habitual desarrollar sobre una versión muy por delante de la de producción: la máquina de desarrollo va al día y el entorno de destino puede ir dos ramas mayores por detrás.

Escribir `match`, promoción en constructor o `str_starts_with` (todo PHP 8.0) en un plugin destinado a un hosting con 7.4 **no da un aviso: da un fatal error de parseo el día del despliegue**, porque el fichero ni siquiera se puede leer.

La skill dedica un paso entero a esto y, cuando la batería de tests no puede correr en la versión mínima (PHPUnit 10 exige PHP 8.1+), monta al menos un trabajo de sintaxis con `php -l`, que sí corre en cualquier versión y detecta exactamente ese fallo.

---

## Reglas de Semgrep

Del registro oficial, sin escribir ninguna:

| Proyecto | Packs |
|---|---|
| PHP genérico | `p/php`, `p/security-audit`, `p/secrets` |
| Plugin o tema de WordPress | los anteriores + `r/php.wordpress-plugins` |
| Node / JS / TS | `p/javascript`, `p/typescript`, `p/security-audit`, `p/secrets` |

`r/php.wordpress-plugins` trae reglas de CSRF, SQL injection, comprobación de permisos, manipulación e inclusión de ficheros, SSRF, redirección abierta, inyección de objetos y ejecución de código y comandos.

---

## Errores conocidos, ya resueltos dentro de la skill

Estos costaron una tarde. Van documentados dentro para que no cuesten otra:

| Síntoma | Causa real |
|---|---|
| `HTTP 429` de `codeload.github.com` | Límite de las IPs compartidas de los runners. `COMPOSER_AUTH` con el `GITHUB_TOKEN` **no** lo evita. Se resuelve con reintento |
| El segundo job de la matriz redescarga todo | La clave de caché llevaba la versión del lenguaje dentro. Hay que cachear la caché de descargas, con clave compartida |
| `workflow_dispatch` da 404 en una rama nueva | Exige que el workflow ya exista en la rama por defecto. Para probar, abrir un PR |
| Semgrep en verde pero no detecta nada | Falta `--error`: sin ese flag `semgrep scan` termina en 0 aunque encuentre hallazgos |
| `Base branch was modified` al fusionar | Otra sesión pusheó a la base mientras corrían los checks |

---

## Qué NO monta, y por qué

- **CodeQL / GitHub Code Security** — no soporta PHP, y es de pago en repos privados.
- **GitHub Secret Protection** — de pago, y en repos de cuenta personal ni siquiera está disponible: exige organización. Alternativa gratis: [gitleaks](https://github.com/gitleaks/gitleaks) como hook de pre-commit, que además bloquea antes de que el commit exista.
- **Servicios de revisión de PR con IA de pago** — si ya revisas código con IA en tu entorno de desarrollo, se duplica.

Las exclusiones son por **disponibilidad y encaje**, no por precio. Si una opción de pago cuesta poco y aporta algo que vas a usar, la skill la plantea con la cifra delante y decides tú.

---

## Licencia

MIT.
