# Conexión de MOP-X 2.0 con Supabase

Esta guía conecta `index.html` con tu propio proyecto de Supabase para que cada empresa/líder tenga sus datos guardados de forma privada y segura (no en el navegador, como en la versión anterior, sino en una base de datos real con permisos por usuario).

No necesitas saber programar para seguir estos pasos — son copiar y pegar.

---

## Fase 1 — Base de datos, autenticación y almacenamiento (obligatoria, deja la app 100% funcional)

### Paso 1. Crear el proyecto en Supabase

1. Ve a [supabase.com](https://supabase.com) → **Start your project** → crea una cuenta u organización si no tienes una.
2. **New project** → ponle un nombre (ej. `mopx-transformacion-exo`), elige una contraseña de base de datos (guárdala) y una región cercana a tus usuarios (ej. `South America (São Paulo)`).
3. Espera 1-2 minutos a que aprovisione el proyecto.

### Paso 2. Crear las tablas y las políticas de seguridad (RLS)

Ve a **SQL Editor** (menú lateral) → **New query**, pega **todo** el bloque siguiente y presiona **Run**.

```sql
-- Extensión necesaria para generar UUIDs
create extension if not exists "pgcrypto";

-- ── Tabla: companies (una fila por empresa/líder) ──────────────────────────
create table public.companies (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  name text not null,
  sector text,
  country text,
  contact_name text,
  role text,
  tool_stack jsonb not null default '{}'::jsonb,
  roadmap jsonb not null default '{}'::jsonb,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

-- ── Tabla: processes (varios procesos por empresa) ─────────────────────────
create table public.processes (
  id uuid primary key default gen_random_uuid(),
  company_id uuid not null references public.companies(id) on delete cascade,
  user_id uuid not null references auth.users(id) on delete cascade,
  name text not null,
  area text,
  diagnostic jsonb,
  phase_checklist jsonb not null default '{}'::jsonb,
  flow jsonb not null default '{"steps":[]}'::jsonb,
  import_status text not null default 'manual',       -- 'manual' | 'pending_ai' | 'ai_ready'
  source_image_path text,                              -- ruta en Storage si el proceso vino de una imagen
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

-- ── Activar Row Level Security (nadie ve datos de otra empresa) ────────────
alter table public.companies enable row level security;
alter table public.processes enable row level security;

create policy "companies_owner_all" on public.companies
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);

create policy "processes_owner_all" on public.processes
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);

-- ── Índices para que las consultas sean rápidas ─────────────────────────────
create index processes_company_id_idx on public.processes(company_id);
create index companies_user_id_idx on public.companies(user_id);
```

**Qué acabas de crear:** dos tablas y una regla de seguridad que dice, en español simple: *"cada fila solo puede ser leída, creada, modificada o borrada por el usuario dueño de esa fila (`auth.uid() = user_id`)"*. Esto corre **dentro de la base de datos**, no en el HTML — así que aunque alguien mire el código fuente de tu app, no puede acceder a datos de otra empresa.

### Paso 3. Crear el bucket de Storage para las imágenes de procesos

Ve a **Storage** (menú lateral) → **New bucket**:
- Nombre: `process-images`
- **Public bucket:** déjalo **desactivado** (privado — las imágenes de tus clientes no son públicas).

Luego ve otra vez a **SQL Editor** → **New query** y pega esto:

```sql
-- Políticas de Storage: cada usuario solo accede a la carpeta con su propio user_id
create policy "process_images_owner_select" on storage.objects for select
  using (bucket_id = 'process-images' and (storage.foldername(name))[1] = auth.uid()::text);

create policy "process_images_owner_insert" on storage.objects for insert
  with check (bucket_id = 'process-images' and (storage.foldername(name))[1] = auth.uid()::text);

create policy "process_images_owner_delete" on storage.objects for delete
  using (bucket_id = 'process-images' and (storage.foldername(name))[1] = auth.uid()::text);
```

La app ya sube cada imagen con la ruta `TU_USER_ID/nombre-archivo.png` — por eso la política filtra por la primera carpeta del path.

### Paso 4. Activar el inicio de sesión por correo y contraseña

Ve a **Authentication → Providers** → confirma que **Email** está habilitado (lo está por defecto), y que dentro de esa misma fila el interruptor **"Allow new users to sign up"** también esté activado (es una opción distinta a "Confirm email", y si está apagada nadie puede registrarse aunque todo lo demás esté bien).

Tu caso específico: el código vive en **GitHub**, pero lo que la gente visita es la URL que te da **Vercel** al desplegar (algo como `https://mopx-tuproyecto.vercel.app`, o tu dominio propio si conectaste uno). Esa es la URL que le importa a Supabase — GitHub nunca aparece aquí, es solo donde vive el código fuente.

Dos configuraciones a revisar en **Authentication → URL Configuration** (a veces aparece como **Authentication → Settings**):

- **Confirm email**: si lo dejas activado, cada líder debe confirmar su correo antes de poder entrar (más seguro, recomendado para producción). Si lo desactivas, entra inmediatamente después de crear la cuenta — así es como lo dejamos para simplificar mientras se valida la plataforma.
- **Site URL**: pon aquí tu URL real de Vercel, ej. `https://mopx-tuproyecto.vercel.app`. Si más adelante conectas un dominio propio (ej. `https://mopx.transformacionexo.com`), actualiza este valor por ese dominio.
- **Redirect URLs**: agrega esa misma URL de Vercel a esta lista (algunos paneles la llaman "Additional Redirect URLs"). Esto importa para funciones futuras como "olvidé mi contraseña" o enlaces mágicos, que si no están en esta lista, Supabase los rechaza por seguridad.

> **Nota sobre Vercel:** cada vez que despliegas una nueva versión, Vercel también genera URLs de "preview" distintas a la de producción (por ejemplo `https://mopx-tuproyecto-git-main-tuusuario.vercel.app`). Si vas a probar el login únicamente en tu URL de producción, no necesitas agregar esas URLs de preview a Supabase. Si algún día pruebas el login directamente desde una URL de preview, agrégala también a "Redirect URLs" o el login fallará solo en esa URL específica.

### Paso 5. Conectar `index.html` con tu proyecto

Ve a **Settings → API** en Supabase y copia:
- **Project URL** (algo como `https://abcdefgh.supabase.co`)
- **anon public key** (una cadena larga que empieza distinto según el proyecto)

Abre `index.html`, busca estas dos líneas cerca del inicio del bloque `<script>`:

```js
const SUPABASE_URL = 'https://TU-PROYECTO.supabase.co';
const SUPABASE_ANON_KEY = 'TU-ANON-KEY-PUBLICA';
```

Reemplázalas por tus valores reales, guarda el archivo. Listo — **la aplicación ya es completamente funcional**: registro de usuarios, empresas, procesos, diagnóstico, diagramador de flujos, importación de imágenes (guardadas, pendientes de interpretación), arquitectura tecnológica, ruta de 90 días y reporte ejecutivo, todo guardado de forma segura y privada por cuenta.

> **Nota de seguridad:** la `anon key` **es segura de exponer** en el HTML — está diseñada para eso. Es una llave pública que solo permite lo que las políticas de RLS autorizan (es decir, nada, hasta que el usuario inicia sesión y solo sobre sus propias filas). Nunca pongas aquí la `service_role key` — esa sí es secreta y nunca debe estar en un archivo que viaja al navegador.

---

## Fase 2 (futura) — Interpretación de imágenes de procesos con IA

Esta fase **no está activa todavía**. Actívala cuando quieras que, al subir una imagen de un diagrama de proceso, Claude la lea automáticamente y proponga los pasos, actores y tipo de automatización — en vez de mapearlos a mano.

### Por qué necesita un paso extra de servidor

Interpretar una imagen requiere llamar a la API de Anthropic con tu propia clave (de pago). Esa clave **nunca debe estar en el HTML** — cualquiera que abra el código fuente de tu sitio podría copiarla y gastar tu saldo. La forma correcta es una **Supabase Edge Function**: una pequeña función que corre en el servidor de Supabase, guarda tu clave como secreto, y es la única que habla con Anthropic. El navegador nunca ve la clave.

### Paso 1. Consigue tu clave de la API de Anthropic

Crea una cuenta y una clave en [console.anthropic.com](https://console.anthropic.com) (esto es una cuenta de pago independiente de tu suscripción a Claude.ai).

### Paso 2. Instala la CLI de Supabase (una sola vez)

```bash
npm install -g supabase
supabase login
supabase link --project-ref TU-PROJECT-REF
```

(`TU-PROJECT-REF` es la parte del Project URL antes de `.supabase.co`, ej. `abcdefgh`.)

### Paso 3. Guarda tu clave de Anthropic como secreto

```bash
supabase secrets set ANTHROPIC_API_KEY=sk-ant-tu-clave-aqui
```

### Paso 4. Crea la Edge Function

```bash
supabase functions new interpret-process-image
```

Reemplaza el contenido de `supabase/functions/interpret-process-image/index.ts` por:

```typescript
// supabase/functions/interpret-process-image/index.ts
import "jsr:@supabase/functions-js/edge-runtime.d.ts";

const CORS_HEADERS = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Headers": "authorization, x-client-info, apikey, content-type",
  "Access-Control-Allow-Methods": "POST, OPTIONS",
};

const SYSTEM_PROMPT = `Eres un analista de procesos experto en notación BPMN y en la Notación MOP-X
(6 elementos: inicio/fin, tarea, decisión, espera/traspaso, sistema/dato, tipo de automatización).
Vas a recibir la imagen de un diagrama de proceso de negocio (foto, captura o diagrama exportado).
Tu tarea es leerlo y devolver ÚNICAMENTE un JSON válido (sin texto adicional, sin markdown, sin backticks)
con esta forma exacta:

{
  "steps": [
    {
      "actor": "string - carril o área responsable de este paso",
      "task": "string - verbo + objeto, ej. 'Validar factura'",
      "type": "manual" | "rpa" | "ia" | "auto",
      "minutes": "string - tiempo estimado si es visible en el diagrama, si no, cadena vacía",
      "isWait": true | false,
      "system": "string - sistema o dato si es visible, si no, cadena vacía",
      "notes": "string - observación breve, opcional"
    }
  ]
}

Reglas para asignar "type":
- "manual": el paso lo hace una persona sin ayuda de sistemas.
- "rpa": el paso es repetitivo, basado en reglas fijas, alto volumen.
- "ia": el paso requiere criterio, clasificación o juicio (aunque hoy sea manual).
- "auto": el diagrama indica que el paso ya está automatizado (ícono de sistema, robot, o texto explícito).

Si no puedes determinar el tipo con certeza, usa "manual" por defecto.
Devuelve los pasos en el orden en que aparecen en el diagrama, de inicio a fin.`;

Deno.serve(async (req) => {
  if (req.method === "OPTIONS") {
    return new Response(null, { status: 204, headers: CORS_HEADERS });
  }

  try {
    const apiKey = Deno.env.get("ANTHROPIC_API_KEY");
    if (!apiKey) {
      return new Response(JSON.stringify({ error: "ANTHROPIC_API_KEY no configurada" }), {
        status: 500, headers: { ...CORS_HEADERS, "Content-Type": "application/json" },
      });
    }

    const { imageBase64, mediaType } = await req.json();
    if (!imageBase64 || !mediaType) {
      return new Response(JSON.stringify({ error: "Falta imageBase64 o mediaType" }), {
        status: 400, headers: { ...CORS_HEADERS, "Content-Type": "application/json" },
      });
    }

    const allowedTypes = ["image/png", "image/jpeg", "image/webp"];
    if (!allowedTypes.includes(mediaType)) {
      return new Response(JSON.stringify({ error: "Formato no soportado" }), {
        status: 400, headers: { ...CORS_HEADERS, "Content-Type": "application/json" },
      });
    }

    const response = await fetch("https://api.anthropic.com/v1/messages", {
      method: "POST",
      headers: {
        "x-api-key": apiKey,
        "anthropic-version": "2023-06-01",
        "content-type": "application/json",
      },
      body: JSON.stringify({
        model: "claude-sonnet-5",
        max_tokens: 2048,
        system: SYSTEM_PROMPT,
        messages: [
          {
            role: "user",
            content: [
              { type: "image", source: { type: "base64", media_type: mediaType, data: imageBase64 } },
              { type: "text", text: "Lee este diagrama de proceso y devuelve el JSON de pasos como se te indicó." },
            ],
          },
        ],
      }),
    });

    const data = await response.json();
    if (!response.ok) {
      return new Response(JSON.stringify({ error: data.error?.message || "Error llamando a Claude" }), {
        status: 502, headers: { ...CORS_HEADERS, "Content-Type": "application/json" },
      });
    }

    const textBlock = data.content?.find((b: any) => b.type === "text");
    let parsed;
    try {
      const clean = (textBlock?.text || "").replace(/```json|```/g, "").trim();
      parsed = JSON.parse(clean);
    } catch {
      return new Response(JSON.stringify({ error: "Claude no devolvió un JSON válido", raw: textBlock?.text }), {
        status: 502, headers: { ...CORS_HEADERS, "Content-Type": "application/json" },
      });
    }

    return new Response(JSON.stringify(parsed), {
      status: 200, headers: { ...CORS_HEADERS, "Content-Type": "application/json" },
    });
  } catch (err) {
    return new Response(JSON.stringify({ error: String(err) }), {
      status: 500, headers: { ...CORS_HEADERS, "Content-Type": "application/json" },
    });
  }
});
```

Despliega la función:

```bash
supabase functions deploy interpret-process-image
```

> El modelo usado es `claude-sonnet-5`. Anthropic actualiza periódicamente sus modelos — si en el futuro quieres usar uno más nuevo o más económico, consulta [docs.claude.com](https://docs.claude.com) y cambia solo esa cadena.

### Paso 5. Conectar el botón desde `index.html`

Cuando despliegues la función, en la página **Importar por imagen** de `index.html`, dentro de la tarjeta "🤖 Próximamente", añade un botón que llame a la función así:

```javascript
async function interpretImageWithAI(proc){
  const { data: signed } = await sb.storage.from('process-images').createSignedUrl(proc.sourceImagePath, 300);
  const imgResponse = await fetch(signed.signedUrl);
  const blob = await imgResponse.blob();
  const base64 = await new Promise(resolve => {
    const reader = new FileReader();
    reader.onloadend = () => resolve(reader.result.split(',')[1]);
    reader.readAsDataURL(blob);
  });

  const { data, error } = await sb.functions.invoke('interpret-process-image', {
    body: { imageBase64: base64, mediaType: blob.type }
  });
  if (error) { toast('No se pudo interpretar la imagen: ' + error.message); return; }

  // data.steps ya viene en el mismo formato que usa proc.flow.steps
  proc.flow.steps = data.steps.map(s => ({ id: gid(), ...s }));
  proc.importStatus = 'ai_ready';
  saveData();
  toast('Pasos interpretados por IA. Revísalos y ajústalos si hace falta.');
  renderFlujoPage(proc);
}
```

Ese es todo el trabajo de conexión. El resto de la interfaz (Diagramador de Flujos, checklist de fases, diagnóstico) ya está construido para recibir esos pasos sin ningún cambio adicional — la IA simplemente pre-llena lo que hoy se llena a mano.

### Costos aproximados de esta fase

Cada imagen interpretada consume tokens de la API de Anthropic (cobro por uso, no por suscripción). Una imagen de diagrama típica con el modelo indicado cuesta centavos de dólar por interpretación. Revisa el precio vigente en [anthropic.com/pricing](https://www.anthropic.com/pricing) antes de activarla a gran escala.

---

## Preguntas frecuentes

**¿Puedo tener varias empresas con la misma cuenta?**
Hoy el diseño es una empresa por cuenta (un líder = un login = una empresa). Si eres consultor y manejas varios clientes, crea una cuenta de correo distinta por cliente, o pide una extensión del esquema para soportar varias empresas por usuario.

**¿Qué pasa si alguien borra `index.html` de GitHub Pages?**
Nada le pasa a tus datos — viven en Supabase, no en el archivo. Puedes volver a publicar el HTML en cualquier momento y los usuarios inician sesión igual.

**¿Cómo hago respaldos?**
Supabase incluye respaldos automáticos diarios en los planes pagos. En el plan gratuito, puedes exportar manualmente desde **Database → Backups**, o programar un export periódico con `pg_dump`.
