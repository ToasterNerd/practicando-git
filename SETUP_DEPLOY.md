# Setup Deploy PRE/PRO — Pasos manuales

Todo lo automatizado (workflows, next.config) ya está creado. Este doc cubre lo que tenés que hacer manualmente.

---

## 1. Generar claves SSH

Desde tu máquina local:

```bash
ssh-keygen -t ed25519 -C "github-actions" -f github-actions
```

Cuando te pida `Enter passphrase`, podés dejarla vacía (presionar `Enter` dos veces). Para este flujo de GitHub Actions no es necesario usar passphrase.

Esto genera:
- `github-actions` → clave **privada** (va al secret `SSH_PRIVATE_KEY`)
- `github-actions.pub` → clave **pública** (va al servidor)

### Agregar la clave pública al VPS

Conectarte al VPS y agregar la clave pública:

```bash
ssh tu-usuario@tu-ip-servidor
cat >> ~/.ssh/authorized_keys << 'EOF'
(pegar contenido de github-actions.pub acá)
EOF
chmod 600 ~/.ssh/authorized_keys
```

### Verificar que se creó correctamente

En la misma sesión SSH, ejecutar:

```bash
grep "github-actions" ~/.ssh/authorized_keys
ls -la ~/.ssh
ls -la ~/.ssh/authorized_keys
```

Deberías ver:
- una línea con `github-actions` en `authorized_keys`
- `~/.ssh` con permisos `700` (drwx------)
- `~/.ssh/authorized_keys` con permisos `600` (-rw-------)

Si no coincide, corregir con:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

---

## 2. Obtener SSH_KNOWN_HOSTS

Desde cualquier máquina con acceso al servidor:

```bash
ssh-keyscan -p 5452 -H tu-ip-servidor
```

También lo podés hacer entrando por PuTTY al servidor y ejecutando el comando con tu IP real, por ejemplo:

```bash
ssh-keyscan -p 5452 -H 179.43.121.178
```

Para el secret `SSH_KNOWN_HOSTS`, copiar solo las líneas que **no** empiezan con `#` (las líneas de clave, por ejemplo `ecdsa-sha2-nistp256`, `ssh-ed25519`, `ssh-rsa`).

Pegarlas tal cual, en múltiples líneas, dentro del secret `SSH_KNOWN_HOSTS`.

> Si usás un puerto SSH distinto de `22` (ej: `5452`), es obligatorio usar `ssh-keyscan -p <puerto> ...` para que el host quede guardado como `[host]:puerto` y la validación de host key funcione.

---

## 3. Configurar Secrets en GitHub

Ir a tu repositorio en GitHub → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

Crear estos 7 secrets:

| Secret | Qué poner | Ejemplo |
|---|---|---|
| `SSH_PRIVATE_KEY` | Contenido completo del archivo `github-actions` (la clave privada, incluyendo las líneas `-----BEGIN...` y `-----END...`) | `-----BEGIN OPENSSH PRIVATE KEY-----` ... |
| `SSH_KNOWN_HOSTS` | Salida completa de `ssh-keyscan -p 5452 -H tu-ip` | `\|1\|abc...= ssh-ed25519 AAAA...` |
| `SSH_USER` | Usuario SSH del VPS | `admin` |
| `SSH_HOST` | IP o dominio del VPS | `123.45.67.89` |
| `SSH_PORT` | Puerto SSH del VPS (no siempre es 22) | `5452` |
| `DEPLOY_PRE_PATH` | Ruta absoluta al `public_html` de PRE (con `/` al final) | `/home/admin/web/pre.midominio.com/public_html/` |
| `DEPLOY_PRO_PATH` | Ruta absoluta al `public_html` de PRO (con `/` al final) | `/home/admin/web/midominio.com/public_html/` |

> **Importante**: Las rutas en VestaCP suelen ser `/home/USUARIO/web/DOMINIO/public_html/`. Asegurate de poner la `/` al final.

> **Recordá**: Los secrets solo se pueden crear y modificar, nunca ver. Si te equivocás, simplemente actualizá el valor.

> **Nota**: Los workflows además refrescan `known_hosts` en runtime con `ssh-keyscan -p $SSH_PORT -H $SSH_HOST` para evitar errores por mismatch de puerto/host key.

---

## 4. Verificar que funciona

### robots.txt por ambiente (automático)

- En **PRE** el workflow genera `robots.txt` con `Disallow: /`
- En **PRO** el workflow genera `robots.txt` con `Allow: /`

Esto evita que PRE se indexe y garantiza que `robots.txt` no se pierda aunque el deploy use `rsync --delete`.

### PRE (automático en cada push a main)

```bash
git add .
git commit -m "setup deploy workflows"
git push origin main
```

→ Ir a GitHub → pestaña **Actions** → verificar que el workflow "Deploy to PRE" corre OK.
→ Abrir `pre.tudominio.com` en el navegador.

### PRO (automático en cada tag nuevo)

```bash
git tag v1.0.0
git push origin v1.0.0
```

→ Ir a GitHub → pestaña **Actions** → verificar que "Deploy to PRO" corre OK.
→ Abrir `tudominio.com` en el navegador.

---

## 5. Troubleshooting

| Problema | Solución |
|---|---|
| `Permission denied (publickey)` | La clave pública no está en `authorized_keys` del VPS, o el secret `SSH_PRIVATE_KEY` está mal copiado |
| `Host key verification failed` | El secret `SSH_KNOWN_HOSTS` está vacío o incorrecto. Regenerar con `ssh-keyscan -H tu-ip` |
| `ssh: connect to host ... Network is unreachable` | Revisar `SSH_HOST` y `SSH_PORT` (deben ser públicos y correctos). Evitar dominios con proxy (ej. Cloudflare proxied) para SSH y usar IP pública o DNS directo |
| `ERR_PNPM_FETCH_403` en `pnpm/action-setup` | Usar `npm` en CI para install/build (workflows actuales ya están ajustados a `npm install` + `npm run build`) |
| `npm ERR! ERESOLVE unable to resolve dependency tree` | En CI usar `npm install --legacy-peer-deps` (workflows actuales ya quedaron así) |
| `rsync: connection unexpectedly closed` | Verificar que el usuario SSH tenga permisos de escritura en la ruta de deploy |
| Build falla con `output: export` | Alguna página usa features incompatibles con static export (API routes, `getServerSideProps`, etc.) |
| `out/` está vacío | Verificar que `next.config.ts` tenga `output: "export"` |

---

## Resumen de archivos del sistema de deploy

| Archivo | Para qué |
|---|---|
| `.github/workflows/deploy-pre.yml` | Workflow: push a main → build → rsync a PRE |
| `.github/workflows/deploy-pro.yml` | Workflow: nuevo tag → build → rsync a PRO |
| `next.config.ts` | Tiene `output: "export"` para generar `out/` |
