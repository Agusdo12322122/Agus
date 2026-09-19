# Análisis y ejecución — OpenHiggsfield AI

Análisis del repositorio [wide-trace/open-higgsfield](https://github.com/wide-trace/open-higgsfield),
del cual este repositorio (`Agusdo12322122/Agus`) es un fork casi idéntico
(mismo código fuente; el historial local añade dos commits propios: *remove
unsupported models* y *add Seedance 2 and 2.5*). El análisis y la ejecución
se hicieron sobre el código de este repo, verificando además que el
upstream original clona e instala igual.

## Qué es

Un estudio de generación de imagen y video con IA, self-hosted, pensado
como alternativa gratuita y open-source a Higgsfield AI. Un solo prompt bar
para 38 modelos (8 de imagen, 30 de video: Soul 2, Soul Cinema, Seedance
2/2.5, Kling 3, Wan, Flux, Ideogram, Recraft, LTX, MiniMax, PixVerse, Grok,
Qwen, etc.), usando la propia API key del usuario contra una plataforma de
generación compatible.

## Stack

- Next.js 16 (App Router, Turbopack) + React 19, TypeScript, Zustand para
  estado, CSS plano (sin framework de estilos).
- Gestor de paquetes: pnpm (10.x), workspace de un solo paquete.
- Persistencia de historial: IndexedDB en el navegador (no hay base de
  datos backend).
- Subida de archivos: Vercel Blob (tokens de subida emitidos por
  `src/app/api/blob/route.ts`).

## Arquitectura (según el código, confirma lo que dice el README)

- **Un único objeto de generación**: `{ model, prompt, media, settings }`.
  La UI arma ese objeto y lo pasa a una *server action*
  (`src/generation/actions.ts`); el navegador nunca llama directo a la API
  de generación — esto evita exponer la API key en el cliente.
- **`src/generation/platform.ts`** implementa el cliente HTTP hacia la
  plataforma de generación: `POST /{model}` para encolar, y
  `GET /requests/{id}/status` para pollear, con
  `Authorization: Key <api_key>`.
- **`src/generation/catalog/`** es la fuente de verdad de modelos: cada
  archivo (`kling-3.ts`, `seedance-2.5.ts`, `wan-3.ts`, ...) declara sus
  propios settings permitidos (aspect ratio, resolución, duración, audio,
  formato de salida) y roles de media (start frame, end frame,
  referencias). El picker y el panel de settings se renderizan a partir de
  ese catálogo, sin listas paralelas hardcodeadas.
- **`src/proxy.ts`** es un middleware minimalista que solo asigna una
  cookie de `deviceId` (usada para llevar historial por dispositivo/sesión),
  no hace proxy de la API en sí.
- La API key del usuario se guarda vía server action en una cookie
  `httpOnly`; si falta, se abre un modal de forma obligatoria (nunca falla
  en silencio).

## Ejecución realizada

1. `pnpm install` — sin errores, 62 paquetes instalados (Node 22.22,
   pnpm 10.33).
2. `pnpm dev` (Next.js 16.3.3 con Turbopack) — servidor arriba en ~0.4s en
   `http://localhost:3000`.
3. `GET /` → `HTTP 200`, título `OpenHiggsfield AI — Open source AI
   studio`.
4. Verificación visual con Chromium (Playwright) headless:
   - Al no haber `HF_API_BASE_URL`/API key configurada, se abre
     automáticamente el modal **"API key"** pidiendo `id:secret` — tal
     como documenta el README ("A missing key opens the modal — it never
     fails silently").
   - Al cerrar el modal, el estudio completo renderiza: tabs *Image /
     Video / Assets / Favorites*, selector de modelo (*Seedance 2.5*),
     barra de settings (16:9, 5s, 720p, Audio, MP4, batch 1/4), composer
     con prompt bar y sugerencias de prompts de ejemplo, botón
     *Generate*.
5. Se repitió la instalación y arranque directamente sobre este repo
   (no solo sobre el clon del upstream), confirmando el mismo resultado
   (`HTTP 200`, servidor estable, sin errores en consola).

No se probó la generación real de imágenes/video porque requiere una
API key de plataforma válida (`HF_API_BASE_URL` + key `id:secret`) y un
token de Vercel Blob (`OPEN_HIGGSFIELD_READ_WRITE_TOKEN`), credenciales que
no están disponibles en este entorno y que el propio proyecto no incluye
(por diseño: "Your own platform key").

## Variables de entorno necesarias para funcionalidad completa

```
HF_API_BASE_URL=                      # origen de la API de generación (solo servidor)
OPEN_HIGGSFIELD_READ_WRITE_TOKEN=     # token read-write de Vercel Blob
```

Sin ellas, la app funciona igual como shell de UI (arranca, navega, abre el
modal de key) pero cualquier intento real de generar o subir un archivo
fallará al no tener backend configurado.

## Conclusión

El repositorio está en buen estado: instala y arranca sin fricción con las
herramientas declaradas (`pnpm`, Node ≥18, Next 16), el código coincide con
lo que documenta el README (arquitectura basada en catálogo, server actions
como único llamador a la API, key del usuario en cookie httpOnly) y la UI
resultante es funcional de punta a punta salvo por las llamadas de red que
dependen de credenciales externas que están fuera del alcance de este
repositorio por diseño.
