# Mesa de revisión

App para gestionar el proceso de revisión de las 28 publicaciones del mes entre la agencia, el diseñador y el cliente:

- **Etapa 1 · Idea, copy y accionables de diseño** — la agencia carga la idea, el copy y las indicaciones para diseño. El cliente valida o pide correcciones.
- **Etapa 2 · Diseño** — se habilita recién cuando el cliente validó la Etapa 1. Ahí se sube el diseño (link de Drive o archivo) y el cliente puede comentar directamente sobre un punto de la imagen, además de validar o pedir correcciones.
- **Publicidad** — además de las 28 publicaciones del mes, hay una sección aparte con 6 placas de publicidad (solo en formato Story o Feed), con el mismo circuito de validación pero claramente identificadas para no confundirlas con el contenido orgánico.

Es un único archivo HTML sin instalación ni build. Para que los comentarios y estados se vean en tiempo real entre todas las personas que entran desde el link, la app se conecta a **Supabase** (Postgres + tiempo real + almacenamiento de archivos, gratis para este volumen de uso). Esta guía te lleva paso a paso.

## 1. Crear el proyecto de Supabase (gratis, ~5 minutos)

1. Entrá a [supabase.com](https://supabase.com), creá una cuenta y hacé clic en **"New project"**.
2. Elegí una organización, ponele un nombre al proyecto (ej: "mesa-de-revision"), definí una contraseña de base de datos (guardala, no se usa en el archivo pero convine tenerla) y la región más cercana. Esperá 1-2 minutos a que se aprovisione.

## 2. Crear las tablas

Adentro del proyecto, andá a **SQL Editor** (ícono en el menú izquierdo) → **New query**, pegá esto y ejecutalo (▶ Run):

```sql
create table publicaciones (
  id text primary key,
  title text not null,
  stage1 jsonb not null,
  stage2 jsonb not null
);

create table comentarios (
  id uuid primary key default gen_random_uuid(),
  pub_id text not null references publicaciones(id) on delete cascade,
  stage int not null,
  parent_id uuid references comentarios(id) on delete cascade,
  kind text not null,
  author jsonb not null,
  text text,
  status text,
  status_change_to text,
  pin jsonb,
  created_at bigint not null
);

alter table publicaciones enable row level security;
alter table comentarios enable row level security;

create policy "public read pub" on publicaciones for select using (true);
create policy "public insert pub" on publicaciones for insert with check (true);
create policy "public update pub" on publicaciones for update using (true);

create policy "public read com" on comentarios for select using (true);
create policy "public insert com" on comentarios for insert with check (true);
create policy "public update com" on comentarios for update using (true);
create policy "public delete com" on comentarios for delete using (true);

alter publication supabase_realtime add table publicaciones;
alter publication supabase_realtime add table comentarios;
```

> Como la app no tiene login (cualquiera con el link entra poniendo su nombre), estas políticas dejan leer y escribir a cualquiera que tenga la clave del proyecto. Es razonable para una herramienta interna de bajo riesgo, pero cualquiera con el link técnicamente podría escribir directo en la base si se lo propusiera.

## 3. Crear el bucket para las imágenes (opcional, solo si van a subir archivos)

Si prefieren trabajar siempre con links de Drive, pueden saltear este paso: la app funciona igual sin él, solo va a avisar que la subida de archivos no está disponible.

1. Andá a **Storage** → **New bucket**.
2. Nombralo exactamente `disenos`.
3. Activá **"Public bucket"** → **Create bucket**.

## 4. Copiar las credenciales

Andá a **Project Settings** (ícono de engranaje) → **API**. Vas a ver:

- **Project URL** (algo como `https://xxxxx.supabase.co`)
- **anon public key** (una clave larga, empieza con `eyJ...`)

## 5. Pegar la configuración en el archivo

Abrí `index.html` con cualquier editor de texto, buscá este bloque cerca del final del archivo:

```js
const SUPABASE_CONFIG = {
  url: "PEGA_TU_SUPABASE_URL",
  anonKey: "PEGA_TU_SUPABASE_ANON_KEY",
};
```

Reemplazá los dos valores por los que copiaste en el paso anterior, y guardá el archivo.

> Estos datos no son secretos: es normal y seguro que la URL y la clave `anon` aparezcan en el código de una app web. La protección real la dan las políticas de la base de datos (paso 2).

## 6. Subir a GitHub Pages

1. Creá un repositorio nuevo en GitHub (puede ser privado o público).
2. Subí `index.html` y este `README.md` a la raíz del repositorio.
3. Andá a **Settings → Pages**, en "Source" elegí la rama `main` y la carpeta `/ (root)` → **Save**.
4. GitHub te va a dar una URL parecida a `https://tu-usuario.github.io/tu-repo/`. Puede tardar 1-2 minutos en estar activa.

## 7. Usarla

Compartí esa URL con el cliente, el diseñador y quien haga falta de la agencia. La primera vez que cada uno entra, pone su nombre, elige su rol (Agencia / Diseñador / Cliente) y su empresa — queda guardado en su navegador, no hace falta repetirlo.

Cosas para saber:

- Las 28 publicaciones y las 6 publicidades ya vienen creadas la primera vez que alguien entra; no hace falta agregarlas, solo completarlas. Arriba del carrusel hay un botón para alternar entre "Publicaciones" y "Publicidad".
- Cada ítem tiene fecha, franja horaria y formato además de la idea, el copy y los accionables de diseño. En publicaciones el formato es Reel / Carrusel / Placa única / Story; en publicidad, solo Story o Feed. Solo la Agencia y el Diseñador pueden completar o editar estos campos; el Cliente los ve de solo lectura y participa dejando comentarios.
- La Etapa 2 se destraba automáticamente cuando el Cliente marca la Etapa 1 como "Validado". Una vez que el Cliente valida una etapa, queda cerrada para siempre: nadie puede modificarla ni revertirla desde la app.
- El botón **"🔗 Copiar link"** dentro de cada ítem copia un link directo (funciona tanto para publicaciones como para publicidades).
- Si necesitás cambiar la cantidad, buscá `const TOTAL_PUBS = 28;` y `const TOTAL_ADS = 6;` en `index.html`.

## 8. Actualización: meses, dashboard y calendario

Si ya habías creado las tablas con el SQL del paso 2, corré esto una sola vez en el **SQL Editor** para agregar el sistema de meses (no borra nada de lo que ya tenías):

```sql
create table if not exists meses (
  id text primary key,
  label text not null,
  created_at bigint not null
);
alter table publicaciones add column if not exists month_id text references meses(id);

alter table meses enable row level security;
create policy "public read meses" on meses for select using (true);
create policy "public insert meses" on meses for insert with check (true);

alter publication supabase_realtime add table meses;
```

Después de correr esto, al volver a abrir la app se va a crear automáticamente el mes actual (por ejemplo "Marzo 2026") con sus 28 publicaciones y 6 publicidades. Si habías cargado datos de prueba en las filas viejas (`pub1`, `ad1`, etc., sin mes asignado), esas filas quedan huérfanas y dejan de mostrarse — no molestan, pero si querés borrarlas para dejar la base limpia, podés correr:

```sql
delete from publicaciones where month_id is null;
```

Novedades en la app:
- Arriba aparece una fila de solapas con los meses. El botón **"+ Nuevo mes"** (solo lo ve la Agencia o el Diseñador) crea un mes nuevo con sus 34 ítems en blanco, sin tocar los meses anteriores.
- Debajo del selector de mes hay tres pestañas: **Tablero** (lo de siempre), **Progreso** (resumen de cuántas están validadas, pendientes, etc.) y **Calendario** (vista de mes con cada publicación en su fecha).
- Una vez que el diseño de una publicación queda Validado, aparece un estado nuevo, **"Estado de publicación"** (Todavía no / Listo para publicar / Publicado), que marca la Agencia o el Diseñador a mano. Es el que se refleja en el Calendario — no se calcula solo con la fecha, así que si algo se posterga o se adelanta, alguien del equipo lo tiene que marcar.
