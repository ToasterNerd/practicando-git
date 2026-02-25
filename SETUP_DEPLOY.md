# Deploy PRE/PRO con GitHub Actions — Guía completa

CI/CD para Next.js estático. Build en GitHub Actions, deploy vía rsync+SSH a un VPS (DonWeb/VestaCP).

- **PRE**: se despliega automáticamente en cada push a `main`
- **PRO**: se despliega automáticamente en cada tag nuevo

---

## Paso 1 — Configurar Next.js para export estático

En `next.config.ts` debe estar:

```ts
output: "export"
```

Esto hace que `npm run build` genere la carpeta `out/` con HTML/CSS/JS estáticos.

---

## Paso 2 — Generar par de claves SSH

Desde tu máquina local:

```bash
ssh-keygen -t ed25519 -C "github-actions" -f github-actions
```

Dejar passphrase vacía (Enter dos veces). Genera dos archivos:

| Archivo | Uso |
|---|---|
| `github-actions` | Clave **privada** → va al secret `SSH_PRIVATE_KEY` en GitHub |
| `github-actions.pub` | Clave **pública** → va al servidor en `~/.ssh/authorized_keys` |

---

## Paso 3 — Cargar la clave pública en el VPS

Desde PowerShell (te pide la contraseña del servidor):

```powershell
Get-Content .\github-actions.pub | ssh -4 -p 5352 root@149.50.143.22 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && chmod 700 ~/.ssh"
```

> Reemplazá `5352`, `root` y `149.50.143.22` por tu puerto, usuario e IP reales.

### Verificar permisos (opcional)

Conectarse al servidor y comprobar:

```bash
grep "github-actions" ~/.ssh/authorized_keys   # debe aparecer la clave
ls -la ~/.ssh/                                  # debe ser drwx------ (700)
ls -la ~/.ssh/authorized_keys                   # debe ser -rw------- (600)
```

---

## Paso 4 — Obtener SSH_KNOWN_HOSTS

Desde cualquier máquina con acceso al servidor:

```bash
ssh-keyscan -p 5352 -H 149.50.143.22
```

> **Importante**: si el puerto SSH no es `22`, es obligatorio usar `-p <puerto>` para que el host quede guardado como `[host]:puerto`.

Copiar **toda la salida** (excepto las líneas que empiezan con `#`) y pegarla en el secret `SSH_KNOWN_HOSTS`.

---

## Paso 5 — Configurar los 7 Secrets en GitHub

Ir al repositorio en GitHub → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**.

| Secret | Qué poner | Ejemplo |
|---|---|---|
| `SSH_PRIVATE_KEY` | Contenido **completo** del archivo `github-actions` (incluye `-----BEGIN...` y `-----END...`) | — |
| `SSH_KNOWN_HOSTS` | Salida de `ssh-keyscan -p 5352 -H tu-ip` (paso 4) | — |
| `SSH_USER` | Usuario SSH del VPS | `root` |
| `SSH_HOST` | IP pública del VPS (**no** un dominio con proxy Cloudflare) | `149.50.143.22` |
| `SSH_PORT` | Puerto SSH del VPS | `5352` |
| `DEPLOY_PRE_PATH` | Ruta absoluta al `public_html` de PRE (con `/` al final) | `/home/root/web/pre.midominio.com/public_html/` |
| `DEPLOY_PRO_PATH` | Ruta absoluta al `public_html` de PRO (con `/` al final) | `/home/root/web/midominio.com/public_html/` |

> Las rutas en VestaCP suelen ser `/home/USUARIO/web/DOMINIO/public_html/`.

---

## Paso 6 — Verificar el deploy

### PRE — push a main

```bash
git add .
git commit -m "setup deploy"
git push origin main
```

→ GitHub → pestaña **Actions** → verificar que "Deploy to PRE" pasa OK.

### PRO — crear un tag

```bash
git tag v1.0.0
git push origin v1.0.0
```

→ GitHub → pestaña **Actions** → verificar que "Deploy to PRO" pasa OK.

---

## Cómo funciona internamente

### Flujo del workflow

1. `actions/checkout@v4` — descarga el código
2. `actions/setup-node@v4` — instala Node 20
3. `npm install --legacy-peer-deps` — instala dependencias
4. Genera `robots.txt` según el ambiente (PRE: `Disallow: /`, PRO: `Allow: /`)
5. `npm run build` — genera la carpeta `out/`
6. `shimataro/ssh-key-action@v2` — configura la clave SSH privada en el runner
7. `ssh-keyscan` — refresca `known_hosts` en runtime con el puerto correcto
8. `rsync` — sube `out/` al `public_html` del servidor con `--delete`

### robots.txt por ambiente

- **PRE** → `Disallow: /` (evita indexación de buscadores)
- **PRO** → `Allow: /` (permite indexación)

Se genera automáticamente antes del build. No hace falta tocarlo manualmente.

### Flags de rsync/SSH

| Flag | Por qué |
|---|---|
| `-4` | Fuerza IPv4 (los runners de GitHub a veces fallan con IPv6) |
| `-p $SSH_PORT` | Puerto SSH personalizado |
| `-o ConnectTimeout=20` | Timeout para no colgar el workflow |
| `--delete` | Borra archivos viejos que ya no existen en el build |

---

## Archivos del sistema de deploy

| Archivo | Función |
|---|---|
| `.github/workflows/deploy-pre.yml` | Workflow: push a `main` → build → rsync a PRE |
| `.github/workflows/deploy-pro.yml` | Workflow: nuevo tag → build → rsync a PRO |
| `next.config.ts` | `output: "export"` para generar `out/` |
| `public/robots.txt` | robots.txt por defecto (se sobreescribe en CI) |

---

## Troubleshooting

| Problema | Causa / Solución |
|---|---|
| `Permission denied (publickey)` | La clave pública no está en `authorized_keys` del VPS. Repetir paso 3. También verificar que `SSH_PRIVATE_KEY` esté completo en el secret |
| `Host key verification failed` | `SSH_KNOWN_HOSTS` incorrecto o generado sin `-p puerto`. Repetir paso 4 |
| `Network is unreachable` | IP incorrecta en `SSH_HOST`, o el runner intenta IPv6. Verificar la IP y que el workflow use `-4` |
| `Connection timed out` | Puerto SSH incorrecto en `SSH_PORT`, o firewall bloqueando |
| `ERESOLVE unable to resolve dependency tree` | Usar `npm install --legacy-peer-deps` (ya configurado en workflows) |
| `rsync: connection unexpectedly closed` | El usuario SSH no tiene permisos de escritura en la ruta de deploy |
| Build falla con `output: export` | Alguna página usa features server-only (`getServerSideProps`, API routes, etc.) |
| `out/` vacío | Verificar que `next.config.ts` tenga `output: "export"` |
