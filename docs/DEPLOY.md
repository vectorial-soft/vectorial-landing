# Deploying `vectorial-soft` to Cloudflare Pages

Guide para publicar este sitio (Vite + React + Tailwind) en Cloudflare Pages y conectarlo al dominio propio. Basado en el mismo flujo con el que se deployó `lalmazan.com`.

**Repo:** `vectorial-soft/vectorial-landing` (branch `master`)
**Dominio destino:** `vectorial-soft.com`
**Registrar del dominio:** donde lo hayas comprado (Squarespace / GoDaddy / Namecheap / …)

---

## Prerrequisitos

- Cuenta gratis en [Cloudflare](https://dash.cloudflare.com).
- El repo está en GitHub y tienes permisos para autorizar Cloudflare como GitHub App.
- El dominio está registrado (aunque siga apuntando a otro lado).
- `npm run build` corre limpio localmente (Node 20.19+ o 22.12+).

## Node version pin (importante)

Vite 6 exige Node ≥ 20.19 o ≥ 22.12. Cloudflare Pages por default usa una versión más vieja. **Pinea la versión** creando un archivo `.nvmrc` en la raíz del repo:

```
22
```

Y como respaldo, agrégalo también como variable de entorno del proyecto en Cloudflare Pages: `NODE_VERSION=22` (paso 8 abajo).

---

## Fase 1 — Crear el proyecto en Cloudflare Pages

1. Haz push de todo lo pendiente a `master`:
   ```powershell
   git add .
   git status                # revisa que no entren secretos
   git commit -m "Prep for Cloudflare Pages deploy"
   git push origin master
   ```

2. En [dash.cloudflare.com](https://dash.cloudflare.com) → sidebar **Workers & Pages** → **Create** → tab **Pages** → **Connect to Git**.

3. Autoriza el GitHub App de Cloudflare para el repo `vectorial-soft/vectorial-landing`.

4. Selecciona el repo → **Begin setup**.

5. Configuración del proyecto:
   - **Project name:** `vectorial-landing` (esto define el subdominio `vectorial-landing.pages.dev`).
   - **Production branch:** `master`.

6. **Build settings**:
   - **Framework preset:** `Vite`
   - **Build command:** `npm run build`
   - **Build output directory:** `dist`
   - **Root directory:** dejar vacío (raíz del repo).

7. **Environment variables** → expande *Variables and Secrets* y agrega:
   - `NODE_VERSION` = `22`

8. **Save and Deploy**. Primer build tarda 2–4 min. Cuando termine, abre `https://vectorial-landing.pages.dev` para verificar que se ve bien.

### Errores comunes en el primer build

| Error | Causa | Fix |
|---|---|---|
| `You are using Node.js 20.16` | falta el pin | agrega `.nvmrc` con `22` y/o env var `NODE_VERSION=22` |
| `Cannot find module '@rolldown/binding-...'` | Node incompatible con Vite/Rolldown | mismo fix: sube Node |
| `Invalid _redirects configuration: Infinite loop detected` | regla `/* /index.html 200` | borra `public/_redirects` (no lo necesitas para este sitio) |
| `Could not resolve import "../../../docs/xxx"` | asset importado que no está commiteado | mueve a `public/` o commítealo |

---

## Fase 2 — Conectar el dominio (una sola vez por dominio)

Dos opciones. **Recomendada: mover los nameservers a Cloudflare** — así los DNS de todos los subdominios futuros (`api.tudominio.com`, `blog.tudominio.com`, …) se manejan desde un solo panel.

### 2A — Nameservers a Cloudflare (recomendado)

1. En Cloudflare → arriba **+ Add a domain** → escribe tu dominio → plan **Free**.

2. Cloudflare escanea los DNS actuales del registrar y los importa. Acepta lo que detecte. Al final te da **2 nameservers** tipo:
   ```
   xxx.ns.cloudflare.com
   yyy.ns.cloudflare.com
   ```
   Cópialos.

3. Ve al panel del registrar del dominio (Squarespace / GoDaddy / etc.) → busca la sección **"Nameservers"** o **"Domain Nameservers"** (NO confundir con "DNS records"). Cambia a **custom / external nameservers** y pega los dos de Cloudflare. Guarda.

4. Regresa a Cloudflare → **Domains → tu dominio**. Espera a que el status pase de **Pending** a **Active** (5–30 min típico, máx 24 h). Puedes darle **"Check nameservers now"**.

5. **Borra en Cloudflare DNS los A/CNAME viejos que apuntaban al registrar anterior** (Squarespace/etc.). Van a estar en **Domain → DNS → Records**. Conserva:
   - MX records (si tienes email en Google Workspace / similar)
   - TXT de verificación (`google-site-verification`, etc.)

6. **Conectar el dominio al proyecto de Pages**:
   - Workers & Pages → `vectorial-landing` → tab **Custom domains** → **Set up a custom domain**.
   - Escribe `vectorial-soft.com` → Continue → **Activate domain**.
   - Repite con `www.vectorial-soft.com` (para que ambos funcionen).

7. SSL se activa solo en ~1 min. Abre `https://vectorial-soft.com` en incógnito para verificar.

### 2B — Mantener el DNS del registrar

Más rápido pero te obliga a manejar futuros subdominios desde el registrar.

1. Workers & Pages → `vectorial-landing` → Custom domains → **Set up a custom domain** → escribe `vectorial-soft.com`.
2. Cloudflare te dará un CNAME target tipo `vectorial-landing.pages.dev`.
3. En el registrar agrega:
   - `CNAME @` → `vectorial-landing.pages.dev` (si tu registrar permite CNAME en apex)
   - `CNAME www` → `vectorial-landing.pages.dev`
   - Si no permite CNAME en `@`, usa los A records que te dará Cloudflare.
4. Espera propagación (5–30 min) y verifica.

---

## Fase 3 — Post-deploy checklist

- **OG image**: crea `public/og-image.png` de 1200×630 y referéncialo en `<meta property="og:image">`. Sin esto, los shares en LinkedIn se ven feos.
- **`robots.txt`** en `public/robots.txt`:
  ```
  User-agent: *
  Allow: /

  Sitemap: https://vectorial-soft.com/sitemap.xml
  ```
- **`sitemap.xml`** en `public/sitemap.xml` con las URLs principales.
- **Meta tags** en `index.html`: `description`, `og:*`, `twitter:*`, `canonical`, JSON-LD si aplica.
- **Google Search Console**: agrega el dominio, verifica vía DNS (1 clic si estás en Cloudflare) y sube el sitemap.
- **Cloudflare Web Analytics** (gratis, sin cookies): Analytics & Logs → Web Analytics → Add site.
- Test final: Lighthouse en Chrome DevTools debería dar 90+ en Performance/Accessibility/SEO.

---

## Iteración diaria después del setup

```powershell
# desarrollas y commiteas normal
git add .
git commit -m "message"
git push origin master
```

Cloudflare detecta el push y redeploya automático en 2–3 min. Cada PR también obtiene un preview deployment con URL propia — útil para revisar cambios antes de mergear.

Para forzar un redeploy manual sin push (por ejemplo tras cambiar env vars):
- Workers & Pages → `vectorial-landing` → **Deployments** → botón **Retry deployment** en la última entrada.

---

## Rollback

En Cloudflare Pages cada build queda archivado. Si un deploy nuevo rompe algo:

- **Deployments** → localiza el build anterior que funcionaba → menú de tres puntos → **Rollback to this deployment**. Producción vuelve al build viejo en segundos, sin tocar git.

---

## Notas / gotchas

- **No metas el dominio de Cloudflare en Cloudflare Pages directamente si sigue apuntando a otro provider**: primero mueve nameservers (Fase 2A) o borra los A records viejos (Fase 2B), si no te sale el error `Hostname already has externally managed DNS records`.
- **DNS records se editan en Cloudflare, no en el registrar** una vez movidos los nameservers. Cualquier cambio en el panel del registrar (Squarespace/GoDaddy/…) queda ignorado.
- **Cache de Cloudflare**: si haces un cambio y no se refleja, hard reload (Ctrl+Shift+R) o purga cache desde Cloudflare (**Caching → Configuration → Purge Everything**).
- **Email seguirá funcionando** siempre que conserves los MX + TXT de verificación cuando limpies el DNS en la Fase 2A.
