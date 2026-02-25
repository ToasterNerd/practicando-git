## Plan: GitHub Actions — Deploy PRE/PRO para Next.js estático

**TL;DR**: Crear dos workflows de GitHub Actions que buildeen la app Next.js en CI y suban solo los archivos estáticos (`out/`) al servidor vía rsync+SSH. PRE se despliega en cada push a `main`, PRO en cada tag nuevo. No se necesita Makefile ni Node.js en el servidor. Hay que configurar 6 secrets en GitHub y generar un par de claves SSH para que el runner se conecte al VPS.

---

### 1. Crear estructura de archivos

Crear la carpeta `.github/workflows/` con dos archivos:

- `.github/workflows/deploy-pre.yml` — workflow de despliegue a PRE
- `.github/workflows/deploy-pro.yml` — workflow de despliegue a PRO

### 2. Workflow PRE — `.github/workflows/deploy-pre.yml`

**Trigger**: `on: push → branches: [main]`

**Steps del job**:
1. `actions/checkout@v4` — descarga el código
2. `actions/setup-node@v4` — instala Node.js (versión 18 o 20, según tu proyecto)
3. `npm ci` — instala dependencias
4. `npm run build` — genera la carpeta `out/`
5. `shimataro/ssh-key-action@v2` — configura la clave SSH privada
6. `rsync` — sube el **contenido** de `out/` al `public_html` del subdominio PRE en el VPS

El comando rsync sería:
```
rsync -avz --delete -e "ssh -i ~/.ssh/github-actions" out/ USER@HOST:/ruta/a/pre/public_html/
```

Con `--delete` para que los archivos viejos que ya no existen se borren del servidor.

### 3. Workflow PRO — `.github/workflows/deploy-pro.yml`

**Trigger**: `on: push → tags: ['*']`

**Steps del job**: Idénticos a PRE, pero:
- El rsync apunta al `public_html` del dominio PRO
- Se usa `DEPLOY_PRO_PATH` en vez de `DEPLOY_PRE_PATH`

### 4. Configurar Secrets en GitHub

Ir a **Settings → Secrets and variables → Actions** en el repositorio y crear estos 6 secrets:

| Secret | Qué contiene | Ejemplo |
|---|---|---|
| `SSH_PRIVATE_KEY` | Clave privada SSH (la que generás en el paso 5) | Contenido completo del archivo `id_ed25519` |
| `SSH_KNOWN_HOSTS` | Fingerprint del servidor para evitar prompt interactivo | Resultado de `ssh-keyscan -H tu-ip-servidor` |
| `SSH_USER` | Usuario SSH del VPS | `admin` o `root` |
| `SSH_HOST` | IP o dominio del VPS | `123.45.67.89` |
| `DEPLOY_PRE_PATH` | Ruta absoluta al `public_html` de PRE | `/home/admin/web/pre.tudominio.com/public_html/` |
| `DEPLOY_PRO_PATH` | Ruta absoluta al `public_html` de PRO | `/home/admin/web/tudominio.com/public_html/` |

> Las rutas dependen de cómo VestaCP organice los sitios (normalmente `/home/USER/web/DOMINIO/public_html/`).

### 5. Generar par de claves SSH

En tu máquina local o en el VPS:

```bash
ssh-keygen -t ed25519 -C "github-actions" -f github-actions
```

Esto genera dos archivos:
- `github-actions` (clave privada) → va al secret `SSH_PRIVATE_KEY`
- `github-actions.pub` (clave pública) → se agrega al `~/.ssh/authorized_keys` del usuario del VPS

### 6. Obtener SSH_KNOWN_HOSTS

Desde cualquier máquina con acceso al servidor:

```bash
ssh-keyscan -H tu-ip-servidor
```

Copiar toda la salida y pegarla en el secret `SSH_KNOWN_HOSTS`.

### 7. Configurar Next.js para export estático

En tu `next.config.js` (o `next.config.mjs`), asegurar que tenga:

```js
output: 'export'
```

Esto hace que `npm run build` genere la carpeta `out/` con HTML/CSS/JS estáticos.

### 8. (Opcional) Agregar `.gitignore` al repo

Incluir al menos:
- `node_modules/`
- `out/`
- `.env`
- `.env.local`

---

## Verificación

1. **PRE**: Hacer un push a `main` → ir a la pestaña **Actions** del repo en GitHub → verificar que el workflow corre sin errores y que los archivos aparecen en `public_html` del subdominio PRE
2. **PRO**: Crear un tag y pushearlo (`git tag v1.0.0 && git push origin v1.0.0`) → verificar deploy a PRO
3. Comprobar en el navegador que ambos sitios cargan correctamente

## Decisiones de diseño

- **Build en CI, no en servidor**: como la app genera archivos estáticos, es más eficiente y seguro buildear en GitHub Actions y subir solo `out/`. No se necesita Makefile ni Node.js en el VPS.
- **No se usa `appleboy/ssh-action`**: al no necesitar ejecutar comandos en el servidor (no hay build ni restart server-side), basta con rsync.
- **`--delete` en rsync**: para mantener el servidor limpio de archivos huérfanos de builds anteriores.
- **actions/checkout@v4 y setup-node@v4**: versiones actualizadas (v3 ya está deprecada).
