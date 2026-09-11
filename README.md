# MOP-X 2.0 — Plataforma de Optimización de Procesos con IA

Plataforma web para que líderes empresariales documenten, diagnostiquen y prioricen la mejora de sus procesos antes de invertir en automatización o IA. Es el tercer modelo de la trilogía de **Transformación ExO** de Juan Carlos Alzate García, junto al **MIEDE X 2.0** (evolución digital organizacional) y el **MRPD 2.0** (reinvención personal).

**Qué es esto, y qué no es:**
MOP-X 2.0 no es un BPM ni una plataforma de orquestación (BOAT). No ejecuta procesos ni conecta sistemas. Es el paso anterior: te da claridad sobre cómo son tus procesos hoy, en qué nivel de madurez están y dónde tiene sentido meter IA con criterio — el mapa que necesitas antes de invertir en una herramienta de esa liga mayor.

---

## Funcionalidades

- **Diagnóstico de madurez** por proceso (5 niveles, cuestionario de 10 dimensiones).
- **Diagramador de flujos** con la Notación MOP-X (6 elementos): carriles por actor, tipo de automatización (manual / RPA / IA / ya automatizado), esperas y traspasos.
- **Importar procesos por imagen**: sube fotos o capturas de diagramas existentes (PNG, JPG, WEBP) y quedan guardadas junto al proceso como referencia para mapear. La interpretación automática con IA está documentada y lista para activarse (ver `SUPABASE_SETUP.md`, Fase 2).
- **Oportunidades de mejora**: compara y prioriza varios procesos a la vez, sin necesitar IA.
- **Arquitectura tecnológica**: selector de herramientas por los 6 bloques del modelo (BPM, RPA, IA/Claude, analítica, integración, gobierno).
- **Ruta de 90 días** y **reporte ejecutivo** imprimible.
- **Cuenta privada por líder/empresa** con autenticación real y datos aislados por Row Level Security — nadie ve los datos de otra empresa, ni siquiera con acceso a la base de datos compartida.

---

## Arquitectura

Aplicación de **un solo archivo HTML** (`index.html`) sin build ni framework — se abre directamente en el navegador o se publica como sitio estático (GitHub Pages, Netlify, Vercel, o cualquier hosting).

| Capa | Tecnología |
|---|---|
| Interfaz | HTML + CSS + JavaScript vanilla (sin frameworks) |
| Autenticación | Supabase Auth (correo + contraseña) |
| Base de datos | Supabase Postgres, con Row Level Security |
| Archivos (imágenes de procesos) | Supabase Storage (bucket privado) |
| IA para interpretar imágenes *(fase futura)* | Supabase Edge Function → API de Claude (Anthropic) |

No hay backend propio que mantener: Supabase cumple ese rol completo (base de datos, autenticación, archivos y, cuando la actives, la función que habla con la IA).

---

## Puesta en marcha

### 1. Conectar con Supabase

Sigue **`SUPABASE_SETUP.md`** paso a paso — crea el proyecto, ejecuta el SQL de tablas y seguridad, crea el bucket de imágenes, y pega tu URL y clave pública en `index.html`. Toma unos 10-15 minutos la primera vez.

### 2. Publicar `index.html`

Este proyecto se despliega así: el código vive en **GitHub**, y **Vercel** es quien lo publica y le da la URL pública.

**Opción A — Vercel (la que usa este proyecto):**
1. Sube este repositorio a GitHub (ya hecho si estás leyendo esto desde ahí).
2. En [vercel.com](https://vercel.com) → **Add New → Project** → importa este repositorio de GitHub.
3. Como es HTML plano, en **Framework Preset** elige **Other** (no Next.js ni ningún framework) para que Vercel sirva `index.html` tal cual, sin intentar compilarlo.
4. **Deploy**. Vercel te da una URL como `https://tu-proyecto.vercel.app` — esa es la que debes poner como **Site URL** en Supabase (ver `SUPABASE_SETUP.md`, Paso 4).
5. Cada vez que hagas `git push` a la rama principal, Vercel vuelve a desplegar automáticamente con los cambios.

**Opción B — GitHub Pages (alternativa gratuita, sin Vercel):**
1. Ve a **Settings → Pages** en tu repositorio → **Source: Deploy from a branch** → elige `main` y la carpeta `/root`.
2. En unos minutos tu app estará en `https://tu-usuario.github.io/tu-repo/`. Usa esa URL como Site URL en Supabase en vez de la de Vercel.

**Opción C — abrirlo localmente (solo para pruebas rápidas):**
Doble clic en `index.html`. El login con Supabase puede fallar en `file://` por restricciones del navegador — usa el **modo demo** (botón en la pantalla de inicio) para probar la interfaz sin depender de la conexión.

### 3. Primer uso

1. Abre tu URL de Vercel (o la que hayas publicado).
2. Crea una cuenta (correo + contraseña).
3. Completa el perfil de tu empresa.
4. Registra tu primer proceso crítico y corre el diagnóstico.

> ¿Solo quieres ver cómo funciona la plataforma sin conectar nada todavía? Usa el botón **"Probar sin conexión (modo demo)"** en la pantalla de inicio — guarda los datos solo en tu navegador, para validar la interfaz mientras se resuelve la conexión real.

---

## Estructura del repositorio

```
├── index.html            # La aplicación completa (interfaz + lógica)
├── README.md              # Este archivo
└── SUPABASE_SETUP.md      # Instrucciones de conexión: SQL, Storage, Auth, y la Edge Function de IA (fase futura)
```

---

## Seguridad y privacidad

- Cada empresa/líder tiene su propia cuenta; sus datos están aislados por **Row Level Security** a nivel de base de datos, no solo en la interfaz.
- La clave (`anon key`) que vive en `index.html` es pública por diseño — no otorga acceso a nada sin una sesión autenticada, y las políticas de RLS limitan cada sesión a sus propias filas.
- Ninguna clave secreta (como la de la API de Anthropic, cuando se active la Fase 2) se coloca jamás en este archivo — vive únicamente como secreto de servidor en una Supabase Edge Function.
- Las imágenes de procesos se guardan en un bucket **privado**, accesible solo mediante URLs firmadas de corta duración.

---

## Sobre el modelo MOP-X 2.0

Desarrollado por **Juan Carlos Alzate García** — Chief AI Officer, ExO Coach certificado, creador del MIEDE X 2.0 y el MRPD 2.0. 30 años transformando organizaciones y líderes en LATAM.

MOP-X 2.0 © 2026 · Transformación ExO
