# Radar VdL · Centros comerciales — puesta en producción

URL final: **https://valordeley.es/hub/recursos/retail**

## Contenido del paquete

| Archivo | Para qué |
| --- | --- |
| `public/hub/recursos/retail/index.html` | La landing completa (mapa, cuestionario, resultado, formulario y generación del PDF). |
| `public/hub/recursos/retail/jspdf.umd.min.js` | Librería que genera el PDF en el navegador (servida desde vuestro dominio). |
| `n8n/radar-vdl-workflow.json` | Flujo de n8n: recibe el lead y el PDF, envía un email a comercial@valordeley.es y otro al lead con el PDF adjunto. |

## 1. n8n (primero, porque necesitas su URL)

1. En n8n: **Workflows → Import from file** → `n8n/radar-vdl-workflow.json`.
2. Abre los dos nodos **Email a comercial** y **Email al lead** y asígnales una credencial **SMTP** del buzón que enviará los correos (p. ej. comercial@valordeley.es). Si el remitente es otro, cambia también el campo *From Email*.
3. Nodo **Webhook Radar VdL**: comprueba que *Allowed Origins (CORS)* es `https://valordeley.es` (si pruebas desde otro dominio, añádelo separado por comas).
4. **Activa** el workflow y copia la **Production URL** del nodo Webhook (termina en `/webhook/radar-vdl`).

## 2. Pegar la URL del webhook en la landing

En `index.html`, busca `CONFIG` y sustituye:

```js
WEBHOOK_URL: "https://TU-INSTANCIA-N8N/webhook/radar-vdl",
```

por la Production URL del paso anterior.

## 3. Publicar en la web (Next.js)

La web es Next.js (servida tras nginx). Pasos para el equipo de desarrollo:

1. Copiar la carpeta `public/hub/recursos/retail/` dentro de `apps/web/public/` (queda en `apps/web/public/hub/recursos/retail/`).
2. Next.js no sirve `index.html` automáticamente en una carpeta, así que hay que añadir una *rewrite* en `next.config` (en `beforeFiles`, para que gane a cualquier ruta dinámica de `/hub/recursos/[slug]`):

```js
async rewrites() {
  return {
    beforeFiles: [
      { source: '/hub/recursos/retail', destination: '/hub/recursos/retail/index.html' },
    ],
  };
}
```

   Si `next.config` ya tiene `rewrites()` devolviendo un array, se mete esa regla en `beforeFiles` y el array existente pasa a `afterFiles`.
3. Desplegar como siempre y abrir https://valordeley.es/hub/recursos/retail.
4. Si nginx tiene una CSP propia, debe permitir: `fonts.googleapis.com` y `fonts.gstatic.com` (tipografías), y en `connect-src` el dominio de n8n. Hoy la web no envía CSP.

## 4. Prueba de extremo a extremo

1. Abre la URL, busca un centro, elige un tamaño de SBA y completa las 12 preguntas.
2. Rellena el formulario con un email tuyo y pulsa **Recibir mi informe**.
3. Debes ver «Recibido, …» en la página, una ejecución correcta en n8n, un email en comercial@valordeley.es (con los datos y el PDF) y otro en tu buzón con el PDF.
4. Si sale «No hemos podido registrar el envío», revisa en n8n que el workflow está activo, la URL es la de *Production* (no la de *Test*) y el origen CORS es correcto.

## 5. Recomendado

- Enlazar el recurso desde https://valordeley.es/hub/recursos (tarjeta nueva).
- Si usáis analítica o gestor de cookies en la web, añadir su script al `<head>` de `index.html`: esta página es independiente y no hereda el layout de Next.js.
- Qué recibe el webhook (multipart/form-data): `name, email, role, phone, center, city, province, sba, visitsEstimate, knownContacts, centersInGroup, score, level, createdAt, fileName, pdfBase64` y `data` (JSON con todo, incluidas las 12 respuestas y la nota por eje).
- Ajustes rápidos en `CONFIG` dentro de `index.html`: `VISITS_PER_M2` (visitas por m² de SBA y año, hoy 152) y `BOOKING_URL` (Calendly).
