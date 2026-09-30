# Instructivo — De prototipo a proyecto real (TripTogether)

Esta guía explica paso a paso cómo pasar del prototipo (un solo archivo que corre adentro de Claude) a un **proyecto React real**, que corra en cualquier computadora con `npm run dev`, con un **backend real en Supabase** para el calendario compartido, las fotos y la ubicación de los amigos.

No hace falta seguir todo en un solo día. Está pensado para ir tildando secciones.

---

## 1. Qué vamos a armar (arquitectura)

```
┌─────────────────────┐ ┌───────────────────────────┐
│ App React (Vite) │ <----> │ Supabase │
│ corre en el celu/PC │ │ (backend ya armado, │
│ │ │ no hay que programarlo) │
│ - Mapa (Leaflet) │ │ - Base de datos (Postgres) │
│ - Cámara (navegador)│ │ - Storage (fotos) │
│ - Calendario │ │ - Realtime (ubicaciones) │
└─────────────────────┘ └───────────────────────────┘
```

La idea clave: **no programamos un servidor propio**. Supabase ya es el backend — nos da base de datos, guardado de archivos y actualización en tiempo real "gratis", y el equipo solo se conecta a él con una librería (`@supabase/supabase-js`).

---

## 2. Herramientas que hay que instalar en la compu

| Herramienta | Para qué | Link |
|---|---|---|
| **Node.js** (versión 18 o superior) | Correr el proyecto React | https://nodejs.org |
| **VS Code** (o el editor que usen) | Escribir el código | https://code.visualstudio.com |
| **Cuenta en Supabase** (gratis) | El backend | https://supabase.com |
| **Git** (opcional pero recomendado) | Subir el código a GitHub | https://git-scm.com |

Para chequear que Node está instalado, abrir una terminal y escribir:

```bash
node -v
npm -v
```

Si tira un número de versión, está listo.

---

## 3. Crear el proyecto React con Vite

Vite es la herramienta que arma el "esqueleto" del proyecto (mucho más rápido y simple que Create React App).

En la terminal, pararse en la carpeta donde quieran guardar el proyecto y correr:

```bash
npm create vite@latest triptogether -- --template react
cd triptogether
npm install
```

Esto crea la carpeta `triptogether` con un proyecto React básico. Para probar que anda:

```bash
npm run dev
```

Va a decir algo como `Local: http://localhost:5173/`. Abrir esa URL en el navegador — tienen que ver la pantalla default de Vite + React.

---

## 4. Instalar las librerías que la app necesita

Adentro de la carpeta `triptogether`:

```bash
npm install lucide-react leaflet @supabase/supabase-js
```

- `lucide-react`: los íconos que ya usamos en el prototipo.
- `leaflet`: el mapa (en el prototipo lo cargábamos desde internet con un `<script>`; en el proyecto real se instala como paquete, que es más prolijo y confiable).
- `@supabase/supabase-js`: el cliente para hablar con el backend.

---

## 5. Estructura de carpetas sugerida

```
triptogether/
├── src/
│ ├── App.jsx → componente principal (los tabs)
│ ├── supabaseClient.js → conexión a Supabase
│ ├── components/
│ │ ├── MapTab.jsx
│ │ ├── PhotosTab.jsx
│ │ └── CalendarTab.jsx
│ └── main.jsx → punto de entrada (ya viene con Vite)
├── .env → claves de Supabase (no se sube a GitHub)
├── .gitignore
├── package.json
└── index.html
```

Muevan el código que ya tienen del archivo `trip-planner-app.jsx` a `src/App.jsx`, y separen `MapTab`, `PhotosTab` y `CalendarTab` en sus propios archivos dentro de `src/components/` (cada uno con su `export default function`). Esto no es obligatorio pero hace que el proyecto sea mucho más fácil de mantener entre tres personas trabajando a la vez.

---

## 6. Armar el backend en Supabase

### 6.1. Crear el proyecto

1. Entrar a https://supabase.com y crear una cuenta (o iniciar sesión, si ya tenés).
2. Crear un **New Project**. Anotar la contraseña de la base de datos que les pida (la van a necesitar).
3. Esperar un par de minutos a que el proyecto termine de crearse.

### 6.2. Crear las tablas

Ir a **SQL Editor** (en el menú de la izquierda) → **New query**, pegar esto y ejecutar:

```sql
-- Un "viaje" agrupa a todo un grupo de amigos
create table trips (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  code text unique not null, -- código corto para unirse al viaje (ej: "MAR2026")
  created_at timestamptz default now()
);

-- Cada persona del grupo, dentro de un viaje
create table members (
  id uuid primary key default gen_random_uuid(),
  trip_id uuid references trips(id) on delete cascade,
  name text not null,
  created_at timestamptz default now()
);

-- Última ubicación conocida de cada miembro
create table locations (
  member_id uuid primary key references members(id) on delete cascade,
  lat double precision not null,
  lng double precision not null,
  updated_at timestamptz default now()
);

-- Actividades del calendario compartido
create table events (
  id uuid primary key default gen_random_uuid(),
  trip_id uuid references trips(id) on delete cascade,
  title text not null,
  date date not null,
  created_at timestamptz default now()
);

-- Fotos del viaje (guardamos la URL, el archivo va en Storage)
create table photos (
  id uuid primary key default gen_random_uuid(),
  trip_id uuid references trips(id) on delete cascade,
  url text not null,
  created_at timestamptz default now()
);
```

### 6.3. Habilitar Realtime en `locations`

Para que la ubicación de los amigos se actualice sola en el mapa de todos, sin recargar la página:

1. Ir a **Database → Replication**.
2. Buscar la tabla `locations` y activar el toggle de Realtime.

### 6.4. Crear el bucket de Storage para las fotos

1. Ir a **Storage** → **New bucket**.
2. Nombre: `trip-photos`. Marcarlo como **público** (así las fotos se pueden ver con una URL directa, sin login).

### 6.5. Permisos (Row Level Security)

Por defecto Supabase bloquea el acceso a las tablas hasta que definas políticas. Para este proyecto de escuela, la forma más simple es permitir lectura y escritura libre (ya que no hay login de usuarios, solo un código de viaje compartido):

```sql
alter table trips enable row level security;
alter table members enable row level security;
alter table locations enable row level security;
alter table events enable row level security;
alter table photos enable row level security;

create policy "acceso libre trips" on trips for all using (true) with check (true);
create policy "acceso libre members" on members for all using (true) with check (true);
create policy "acceso libre locations" on locations for all using (true) with check (true);
create policy "acceso libre events" on events for all using (true) with check (true);
create policy "acceso libre photos" on photos for all using (true) with check (true);
```

> Esto está bien para una demo escolar. Si más adelante quieren que no cualquiera pueda borrar los datos de cualquier viaje, se agrega login con Supabase Auth y políticas más estrictas — pero no hace falta para entregar el proyecto.

### 6.6. Conseguir las claves de conexión

Ir a **Project Settings → API**. Vas a necesitar dos datos:
- **Project URL**
- **anon public key**

---

## 7. Conectar React con Supabase

### 7.1. Archivo `.env`

En la raíz del proyecto (`triptogether/.env`):

```
VITE_SUPABASE_URL=https://tu-proyecto.supabase.co
VITE_SUPABASE_ANON_KEY=tu-clave-anon-publica
```

Agregar `.env` al `.gitignore` para no subir las claves a GitHub.

### 7.2. Cliente de Supabase — `src/supabaseClient.js`

```js
import { createClient } from "@supabase/supabase-js";

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseKey = import.meta.env.VITE_SUPABASE_ANON_KEY;

export const supabase = createClient(supabaseUrl, supabaseKey);
```

### 7.3. Reemplazar el calendario (antes usaba `window.storage`)

```js
import { supabase } from "../supabaseClient";

// Leer eventos de un viaje
async function loadEvents(tripId) {
  const { data, error } = await supabase
    .from("events")
    .select("*")
    .eq("trip_id", tripId)
    .order("date", { ascending: true });
  if (error) console.error(error);
  return data ?? [];
}

// Agregar un evento
async function addEvent(tripId, title, date) {
  const { error } = await supabase.from("events").insert({ trip_id: tripId, title, date });
  if (error) console.error(error);
}

// Borrar un evento
async function removeEvent(id) {
  const { error } = await supabase.from("events").delete().eq("id", id);
  if (error) console.error(error);
}
```

### 7.4. Ubicación de amigos en tiempo real

Cada dispositivo actualiza su propia ubicación cada tanto:

```js
navigator.geolocation.watchPosition(async (pos) => {
  await supabase.from("locations").upsert({
    member_id: myMemberId,
    lat: pos.coords.latitude,
    lng: pos.coords.longitude,
    updated_at: new Date().toISOString(),
  });
});
```

Y todos los dispositivos se suscriben a los cambios para verlos en el mapa:

```js
useEffect(() => {
  const channel = supabase
    .channel("locations-channel")
    .on(
      "postgres_changes",
      { event: "*", schema: "public", table: "locations" },
      (payload) => {
        // actualizar el marcador correspondiente en el mapa
      }
    )
    .subscribe();

  return () => supabase.removeChannel(channel);
}, []);
```

Esto es lo que reemplaza la simulación que teníamos antes: ahora si dos compañeros abren la app en sus celulares, se ven de verdad en el mapa del otro.

### 7.5. Subir fotos a Storage

```js
async function uploadPhoto(tripId, file) {
  const fileName = `${tripId}/${Date.now()}-${file.name}`;
  const { error: uploadError } = await supabase.storage
    .from("trip-photos")
    .upload(fileName, file);
  if (uploadError) { console.error(uploadError); return; }

  const { data } = supabase.storage.from("trip-photos").getPublicUrl(fileName);
  await supabase.from("photos").insert({ trip_id: tripId, url: data.publicUrl });
}
```

Como la cámara del prototipo ya genera la foto como `dataUrl` (un canvas), hay que convertirla a `Blob`/`File` antes de subirla:

```js
function dataUrlToFile(dataUrl, filename) {
  const arr = dataUrl.split(",");
  const mime = arr[0].match(/:(.*?);/)[1];
  const bstr = atob(arr[1]);
  let n = bstr.length;
  const u8arr = new Uint8Array(n);
  while (n--) u8arr[n] = bstr.charCodeAt(n);
  return new File([u8arr], filename, { type: mime });
}
```

---

## 8. Cómo se une cada amigo al mismo viaje

Como no hay login, la forma más simple para un proyecto escolar es un **código de viaje**:

1. Uno del grupo crea el viaje (`insert` en `trips`, con un `code` como "MAR2026").
2. Cada amigo, al abrir la app por primera vez, escribe ese código.
3. La app busca el `trip_id` correspondiente (`select * from trips where code = ...`) y lo guarda en el `localStorage` del navegador, para no tener que escribirlo de nuevo cada vez.

---

## 9. Correr el proyecto en la computadora

Una vez que el código está acomodado en `src/`:

```bash
npm run dev
```

Y abrir `http://localhost:5173` en el navegador. Para probar el mapa y la cámara hace falta **HTTPS o localhost** (los navegadores no dan permiso de cámara/ubicación en `http://` normal) — `localhost` ya cumple esa condición, así que no hay que hacer nada extra para probarlo en la compu.

Para probarlo desde el **celular** conectado a la misma wifi: correr `npm run dev -- --host` y usar la IP que muestre la terminal (ej: `http://192.168.0.15:5173`). Ojo: en ese caso el navegador del celular puede pedir HTTPS para la cámara — si da problemas, lo más simple es probar la cámara directamente en la compu y dejar la versión "de bolsillo" para cuando esté publicada online (paso 10).

---

## 10. (Más adelante) Publicarla en internet

No hace falta para esta etapa, pero cuando quieran que cualquiera la abra desde un link:

1. Subir el proyecto a GitHub.
2. Crear una cuenta en [Vercel](https://vercel.com) o [Netlify](https://netlify.com) (gratis).
3. Conectar el repositorio — detectan Vite automáticamente.
4. Cargar las mismas variables de entorno (`VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`) en la configuración del sitio.
5. Listo: queda con HTTPS real, que es lo que hace falta para que la cámara y la ubicación funcionen bien en cualquier celular.

---

## Resumen de lo que cambia respecto al prototipo

| Cosa | Antes (prototipo) | Ahora (proyecto real) |
|---|---|---|
| Dónde corre | Solo dentro de Claude | Cualquier compu/celular, con `npm run dev` |
| Mapa | Leaflet cargado desde un `<script>` | Leaflet instalado como paquete npm |
| Calendario | `window.storage` (solo funciona en Claude) | Tabla `events` en Supabase |
| Fotos | Se guardaban en memoria, se perdían al cerrar | Supabase Storage, quedan guardadas |
| Ubicación de amigos | Simulada (puntos random cerca tuyo) | Real: cada celular manda su posición a Supabase y los demás la ven al toque |

Con esto ya tienen la base completa. El siguiente paso es acomodar el código del prototipo dentro de esta nueva estructura — eso lo podemos hacer juntos cuando quieras arrancar con las mejoras de lógica y UI.
