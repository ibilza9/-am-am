# Ñam Ñam · beta a coste cero

`index.html` es toda la app. Se compone de:

| Pieza | Servicio | Coste |
|---|---|---|
| Mapa | Leaflet + teselas de OpenStreetMap | 0 € |
| Bares y restaurantes | Overpass API (datos de OpenStreetMap) + Nominatim (buscar ciudades) | 0 € |
| Cuentas y base de datos | Supabase (plan gratuito) | 0 € |
| Alojamiento | GitHub Pages | 0 € |

Al principio del `<script>` hay un bloque `CONFIG`:

```js
const CONFIG={
  OSM:true,                                              // mapa y locales reales
  SUPABASE:{url:'https://TU-PROYECTO.supabase.co', key:'TU_CLAVE_PUBLICABLE'}
};
```
Con `SUPABASE:null` las cuentas se guardan solo en el navegador (para probar). Con `OSM:false` Explorar queda desactivado.

## Cómo se llenan los locales
Los **usuarios añaden los locales que les interesan**: buscan una zona, pulsan *Guardar* o *Valorar* y solo entonces el local se guarda
en tu base de datos. No se importa ninguna ciudad entera (Overpass lo limita y la base de datos gratuita se llenaría de locales que nadie usa).

## 1. Supabase
1. Crea un proyecto en https://supabase.com.
2. **SQL Editor** → pega y ejecuta:

```sql
create table public.profiles (
  id uuid primary key references auth.users on delete cascade,
  username text unique not null check (username ~ '^[a-z0-9._]{3,20}$'),
  name text not null,
  bio text default '',
  pub boolean default true,
  created_at timestamptz default now()
);
create table public.venues (
  id bigint generated always as identity primary key,
  osm_type text not null,
  osm_id bigint not null,
  name text not null,
  kind text,
  cuisine text,
  city text,
  lat double precision,
  lng double precision,
  address text,
  created_by uuid references auth.users,
  created_at timestamptz default now(),
  unique (osm_type, osm_id)
);
create table public.saved_venues (
  user_id uuid references auth.users on delete cascade,
  venue_id bigint references public.venues on delete cascade,
  note text default '',
  tags text[] default '{}',
  created_at timestamptz default now(),
  primary key (user_id, venue_id)
);
create table public.follows (
  follower uuid references auth.users on delete cascade,
  followed uuid references auth.users on delete cascade,
  created_at timestamptz default now(),
  primary key (follower, followed),
  check (follower <> followed)
);

alter table public.profiles enable row level security;
alter table public.venues enable row level security;
alter table public.saved_venues enable row level security;
alter table public.follows enable row level security;

create policy profiles_read on public.profiles for select to authenticated using (true);
create policy profiles_update_own on public.profiles for update to authenticated
  using (auth.uid() = id) with check (auth.uid() = id);
create policy profiles_insert_own on public.profiles for insert to authenticated with check (id = auth.uid());
create policy venues_read on public.venues for select to authenticated using (true);
create policy venues_insert on public.venues for insert to authenticated with check (created_by = auth.uid());
create policy saved_own on public.saved_venues for all to authenticated
  using (user_id = auth.uid()) with check (user_id = auth.uid());

create policy follows_read on public.follows for select to authenticated
  using (follower = auth.uid() or followed = auth.uid());
create policy follows_insert on public.follows for insert to authenticated with check (follower = auth.uid());
create policy follows_delete on public.follows for delete to authenticated using (follower = auth.uid());

grant select, insert, update on public.profiles to authenticated;
grant select, insert, delete on public.follows to authenticated;
grant select, insert on public.venues to authenticated;
grant select, insert, update, delete on public.saved_venues to authenticated;

-- Crea el perfil automáticamente al registrarse (también con Google)
create function public.handle_new_user() returns trigger
language plpgsql security definer set search_path = '' as $$
declare u text; n int := 0;
begin
  u := regexp_replace(lower(coalesce(new.raw_user_meta_data->>'username', split_part(new.email,'@',1))), '[^a-z0-9._]', '', 'g');
  if length(u) < 3 then u := u || 'user'; end if;
  u := left(u, 16);
  while exists (select 1 from public.profiles where username = u) loop
    n := n + 1; u := left(u, 16) || n::text;
  end loop;
  insert into public.profiles (id, username, name)
  values (new.id, u, coalesce(new.raw_user_meta_data->>'name', new.raw_user_meta_data->>'full_name', split_part(new.email,'@',1)));
  return new;
end $$;
create trigger on_auth_user_created after insert on auth.users
  for each row execute function public.handle_new_user();

-- Comprobar si un nombre de usuario está libre (se usa antes de registrarse)
create function public.username_available(u text) returns boolean
language sql security definer set search_path = '' as $$
  select not exists (select 1 from public.profiles where username = lower(u))
$$;
grant execute on function public.username_available(text) to anon, authenticated;
```

3. **Authentication → Providers**: deja activo *Email*. Para **Google**: crea un *OAuth Client ID* (tipo web) en
   https://console.cloud.google.com (no requiere facturación), pon como *Authorized redirect URI* la que te muestra Supabase
   (`https://TU-PROYECTO.supabase.co/auth/v1/callback`) y copia el *Client ID* y el *Secret* en Supabase.
4. **Authentication → URL Configuration**: *Site URL* = la dirección de tu GitHub Pages; añádela también a *Redirect URLs*.
5. Para vincular Google a una cuenta existente, activa **Allow manual linking** en los ajustes de Authentication.
6. Para la beta puedes desactivar *Confirm email* (el envío de correos del plan gratuito tiene un límite bajo por hora); si lo dejas activo,
   la app avisa de que hay que confirmar el correo.
7. **Project Settings → API**: copia *Project URL* y la clave *anon / publishable* en `CONFIG.SUPABASE`.
   Esa clave es pública por diseño; la seguridad la dan las políticas (RLS) del SQL de arriba.

## 1b. SQL parte 2: reseñas, fotos, comentarios, grupos, chats, listas y notificaciones
Ejecuta esto **después** del script anterior (SQL Editor → pegar → Run):

```sql
alter table public.saved_venues add column if not exists position int default 0;

create table public.reviews (
  id bigint generated always as identity primary key,
  user_id uuid not null references public.profiles(id) on delete cascade,
  venue_id bigint not null references public.venues(id),
  rating numeric(4,2) not null check (rating >= 0 and rating <= 10),
  title text not null,
  body text default '',
  price smallint check (price between 1 and 4),
  photos text[] default '{}',
  tagged text[] default '{}',
  in_feed boolean default true,
  created_at timestamptz default now()
);
create table public.review_likes (
  review_id bigint references public.reviews on delete cascade,
  user_id uuid references public.profiles(id) on delete cascade,
  primary key (review_id, user_id)
);
create table public.review_comments (
  id bigint generated always as identity primary key,
  review_id bigint not null references public.reviews on delete cascade,
  user_id uuid not null references public.profiles(id) on delete cascade,
  body text not null check (length(body) between 1 and 500),
  created_at timestamptz default now()
);
create table public.spaces (
  id bigint generated always as identity primary key,
  kind text not null check (kind in ('group','dm')),
  name text,
  created_by uuid not null references public.profiles(id),
  created_at timestamptz default now()
);
create table public.space_members (
  space_id bigint references public.spaces on delete cascade,
  user_id uuid references public.profiles(id) on delete cascade,
  primary key (space_id, user_id)
);
create table public.space_messages (
  id bigint generated always as identity primary key,
  space_id bigint not null references public.spaces on delete cascade,
  user_id uuid not null references public.profiles(id) on delete cascade,
  body text,
  review_id bigint references public.reviews on delete set null,
  is_new boolean default false,
  created_at timestamptz default now(),
  check (body is not null or review_id is not null)
);
create table public.space_reviews (
  space_id bigint references public.spaces on delete cascade,
  review_id bigint references public.reviews on delete cascade,
  user_id uuid not null references public.profiles(id) on delete cascade,
  primary key (space_id, review_id)
);
create table public.space_wish (
  space_id bigint references public.spaces on delete cascade,
  venue_id bigint references public.venues on delete cascade,
  added_by uuid references public.profiles(id) on delete set null,
  position int default 0,
  primary key (space_id, venue_id)
);
create table public.lists (
  id bigint generated always as identity primary key,
  user_id uuid not null references public.profiles(id) on delete cascade,
  name text not null,
  unique (user_id, name)
);
create table public.list_items (
  list_id bigint references public.lists on delete cascade,
  review_id bigint references public.reviews on delete cascade,
  position int default 0,
  primary key (list_id, review_id)
);
create table public.notifications (
  id bigint generated always as identity primary key,
  user_id uuid not null references public.profiles(id) on delete cascade,
  from_user uuid not null references public.profiles(id) on delete cascade,
  type text not null check (type in ('tag','comment','follow')),
  review_id bigint references public.reviews on delete cascade,
  read boolean default false,
  created_at timestamptz default now()
);

-- Ayuda para las reglas de grupos y chats (evita recursión)
create function public.is_space_member(sid bigint) returns boolean
language sql security definer stable set search_path = '' as $$
  select exists (select 1 from public.space_members where space_id = sid and user_id = auth.uid())
$$;
grant execute on function public.is_space_member(bigint) to authenticated;

alter table public.reviews enable row level security;
alter table public.review_likes enable row level security;
alter table public.review_comments enable row level security;
alter table public.spaces enable row level security;
alter table public.space_members enable row level security;
alter table public.space_messages enable row level security;
alter table public.space_reviews enable row level security;
alter table public.space_wish enable row level security;
alter table public.lists enable row level security;
alter table public.list_items enable row level security;
alter table public.notifications enable row level security;

-- Reseñas: las tuyas, las de perfiles públicos, las de quien sigues y las compartidas en tus grupos/chats
create policy reviews_read on public.reviews for select to authenticated using (
  user_id = auth.uid()
  or (in_feed and exists (select 1 from public.profiles p where p.id = reviews.user_id and p.pub))
  or (in_feed and exists (select 1 from public.follows f where f.follower = auth.uid() and f.followed = reviews.user_id))
  or exists (select 1 from public.space_reviews sr where sr.review_id = reviews.id and public.is_space_member(sr.space_id))
);
create policy reviews_insert on public.reviews for insert to authenticated with check (user_id = auth.uid());
create policy reviews_update on public.reviews for update to authenticated using (user_id = auth.uid()) with check (user_id = auth.uid());
create policy reviews_delete on public.reviews for delete to authenticated using (user_id = auth.uid());

create policy likes_read on public.review_likes for select to authenticated
  using (exists (select 1 from public.reviews r where r.id = review_likes.review_id));
create policy likes_insert on public.review_likes for insert to authenticated
  with check (user_id = auth.uid() and exists (select 1 from public.reviews r where r.id = review_likes.review_id));
create policy likes_delete on public.review_likes for delete to authenticated using (user_id = auth.uid());

create policy comments_read on public.review_comments for select to authenticated
  using (exists (select 1 from public.reviews r where r.id = review_comments.review_id));
create policy comments_insert on public.review_comments for insert to authenticated
  with check (user_id = auth.uid() and exists (select 1 from public.reviews r where r.id = review_comments.review_id));
create policy comments_delete on public.review_comments for delete to authenticated
  using (user_id = auth.uid() or exists (select 1 from public.reviews r where r.id = review_comments.review_id and r.user_id = auth.uid()));

create policy spaces_read on public.spaces for select to authenticated using (created_by = auth.uid() or public.is_space_member(id));
create policy spaces_insert on public.spaces for insert to authenticated with check (created_by = auth.uid());
create policy members_read on public.space_members for select to authenticated using (user_id = auth.uid() or public.is_space_member(space_id));
create policy members_insert on public.space_members for insert to authenticated
  with check (exists (select 1 from public.spaces s where s.id = space_members.space_id and s.created_by = auth.uid()));
create policy members_leave on public.space_members for delete to authenticated using (user_id = auth.uid());
create policy messages_read on public.space_messages for select to authenticated using (public.is_space_member(space_id));
create policy messages_insert on public.space_messages for insert to authenticated
  with check (user_id = auth.uid() and public.is_space_member(space_id));
create policy sreviews_read on public.space_reviews for select to authenticated using (public.is_space_member(space_id));
create policy sreviews_insert on public.space_reviews for insert to authenticated
  with check (user_id = auth.uid() and public.is_space_member(space_id)
    and exists (select 1 from public.reviews r where r.id = space_reviews.review_id and r.user_id = auth.uid()));
create policy swish_all on public.space_wish for all to authenticated
  using (public.is_space_member(space_id)) with check (public.is_space_member(space_id));
create policy lists_own on public.lists for all to authenticated using (user_id = auth.uid()) with check (user_id = auth.uid());
create policy list_items_own on public.list_items for all to authenticated
  using (exists (select 1 from public.lists l where l.id = list_items.list_id and l.user_id = auth.uid()))
  with check (exists (select 1 from public.lists l where l.id = list_items.list_id and l.user_id = auth.uid()));
create policy notif_read on public.notifications for select to authenticated using (user_id = auth.uid());
create policy notif_insert on public.notifications for insert to authenticated with check (from_user = auth.uid());
create policy notif_update on public.notifications for update to authenticated using (user_id = auth.uid()) with check (user_id = auth.uid());
create policy notif_delete on public.notifications for delete to authenticated using (user_id = auth.uid());

grant select, insert, update, delete on public.reviews, public.review_likes, public.review_comments,
  public.spaces, public.space_members, public.space_messages, public.space_reviews, public.space_wish,
  public.lists, public.list_items, public.notifications to authenticated;

-- Fotos: bucket público de lectura; solo cada usuario puede subir y borrar en su carpeta
insert into storage.buckets (id, name, public) values ('review-photos', 'review-photos', true)
  on conflict (id) do nothing;
create policy photos_insert_own on storage.objects for insert to authenticated
  with check (bucket_id = 'review-photos' and (storage.foldername(name))[1] = auth.uid()::text);
create policy photos_delete_own on storage.objects for delete to authenticated
  using (bucket_id = 'review-photos' and (storage.foldername(name))[1] = auth.uid()::text);
```

Las fotos se reducen en el navegador (máximo 1280 px, JPEG) antes de subirlas para que el 1 GB gratuito de Storage dé para miles.
El bucket es de lectura pública: cualquiera con la dirección exacta de una foto puede verla (las direcciones llevan un código aleatorio).

### Si ya ejecutaste la parte 1 antes de esta versión
Ejecuta además este pequeño bloque (permite que la app repare tu perfil si el automático falló):
```sql
create policy profiles_insert_own on public.profiles for insert to authenticated with check (id = auth.uid());
grant insert on public.profiles to authenticated;
```

### Si no te deja registrarte o entrar con Google
En la pantalla de entrada pulsa **«¿Problemas para entrar? Comprobar conexión»**. Te dice, con ✅ / ⚠️ / ❌, si la URL y la clave son correctas,
si «Confirm email» y Google están activados, si el SQL parte 1 está instalado y cuál es la dirección de tu página (debe estar en Site URL y Redirect URLs).
Los mensajes de error de la app ahora explican la causa real. Las causas más habituales:
- No se ejecutó el SQL parte 1 entero (error «Database error saving new user» o «Falta el SQL parte 1»).
- «Confirm email» activado y el plan gratuito sin correos (desactívalo en Authentication → Sign In / Providers → Email).
- Google no activado, o la *Callback URL* de Supabase no está en «Authorized redirect URIs» de Google (error `redirect_uri_mismatch`).
- La dirección de tu página no está en Site URL / Redirect URLs.
- Clave copiada de otro proyecto o incompleta.
- Error **401** en `rpc/username_available` (consola, F12): o la clave no vale, o falta el permiso para usuarios sin sesión. Ejecuta `grant execute on function public.username_available(text) to anon, authenticated;`. El registro ya no depende de esa comprobación.

## 1c. SQL parte 3: cuentas públicas/privadas, solicitudes de seguimiento y avisos nuevos
Ejecuta esto **después** de las partes 1 y 1b (SQL Editor → pegar → Run):

```sql
-- Seguimientos con solicitud (las cuentas privadas necesitan aprobación)
alter table public.follows add column if not exists status text not null default 'accepted';
alter table public.follows drop constraint if exists follows_status_check;
alter table public.follows add constraint follows_status_check check (status in ('accepted','pending'));

drop policy if exists follows_read on public.follows;
drop policy if exists follows_insert on public.follows;
drop policy if exists follows_update on public.follows;
drop policy if exists follows_delete on public.follows;
create policy follows_read on public.follows for select to authenticated
  using (follower = auth.uid() or followed = auth.uid());
create policy follows_insert on public.follows for insert to authenticated
  with check (follower = auth.uid() and (status = 'pending'
    or exists (select 1 from public.profiles p where p.id = followed and p.pub)));
create policy follows_update on public.follows for update to authenticated
  using (followed = auth.uid()) with check (followed = auth.uid() and status = 'accepted');
create policy follows_delete on public.follows for delete to authenticated
  using (follower = auth.uid() or followed = auth.uid());
grant select, insert, update, delete on public.follows to authenticated;

-- Las reseñas de una cuenta privada solo las ven seguidores ya aceptados
drop policy if exists reviews_read on public.reviews;
create policy reviews_read on public.reviews for select to authenticated using (
  user_id = auth.uid()
  or (in_feed and exists (select 1 from public.profiles p where p.id = reviews.user_id and p.pub))
  or (in_feed and exists (select 1 from public.follows f
        where f.follower = auth.uid() and f.followed = reviews.user_id and f.status = 'accepted'))
  or exists (select 1 from public.space_reviews sr
        where sr.review_id = reviews.id and public.is_space_member(sr.space_id))
);

-- Nuevos tipos de aviso: solicitud de seguimiento y solicitud aceptada
alter table public.notifications drop constraint if exists notifications_type_check;
alter table public.notifications add constraint notifications_type_check
  check (type in ('tag','comment','follow','request','accepted'));

-- El perfil nuevo respeta el tipo de cuenta elegido al registrarse
create or replace function public.handle_new_user() returns trigger
language plpgsql security definer set search_path = '' as $$
declare u text; n int := 0;
begin
  u := regexp_replace(lower(coalesce(new.raw_user_meta_data->>'username', split_part(new.email,'@',1))), '[^a-z0-9._]', '', 'g');
  if length(u) < 3 then u := u || 'user'; end if;
  u := left(u, 16);
  while exists (select 1 from public.profiles where username = u) loop
    n := n + 1; u := left(u, 16) || n::text;
  end loop;
  insert into public.profiles (id, username, name, pub)
  values (new.id, u,
    coalesce(new.raw_user_meta_data->>'name', new.raw_user_meta_data->>'full_name', split_part(new.email,'@',1)),
    coalesce((new.raw_user_meta_data->>'pub')::boolean, true));
  return new;
end $$;
```
Aunque el trigger falle o no esté actualizado, la app aplica la elección de cuenta pública/privada al primer inicio de sesión.

### Qué cambia con las cuentas privadas
- Quien quiera seguir a una cuenta privada envía una **solicitud** (botón «Solicitar» → «Solicitado»). El dueño la ve en Perfil → Notificaciones y pulsa **Aceptar** o **Rechazar**.
- Una cuenta pública se sigue al instante y recibe el aviso «ha empezado a seguirte», con botón **Seguir de vuelta**.
- Las reseñas de una cuenta privada solo las ven seguidores aceptados (y los miembros de grupos/chats donde la comparta).

## Mapa: zona habitual
Explorar muestra por defecto los locales de **la zona donde más tiempo has usado la app**. Para calcularlo, mientras la app está abierta y has dado permiso de ubicación,
cada pocos minutos se suma tiempo a una cuadrícula de unos 1,1 km. Esos datos se guardan **solo en el navegador de tu dispositivo** (no se envían a Supabase)
y puedes borrarlos con «Olvidar mis zonas». Si no hay historial, usa tu ubicación actual (te pide permiso). Una web no puede seguir tu ubicación en segundo plano.

## 2. Publicar en GitHub Pages
1. Crea un repositorio público y sube **todo el contenido de esta carpeta**: `index.html`, `manifest.webmanifest`, `sw.js` y la carpeta `icons` (con sus 3 imágenes). Los archivos tienen que quedar en la raíz del repositorio.
2. **Settings → Pages → Deploy from a branch → main / (root)**.
3. Tu app quedará en `https://TU-USUARIO.github.io/TU-REPO/`. Úsala como *Site URL* en Supabase.

## 2b. Instalarla en el móvil (sin tiendas de aplicaciones)
La app es instalable (PWA): con icono propio, a pantalla completa y sin barra del navegador. Es gratis y no necesita Google Play ni App Store.
- **Android (Chrome):** abre tu dirección de GitHub Pages → menú ⋮ → *Instalar aplicación* (o *Añadir a la pantalla de inicio*).
- **iPhone (Safari, no Chrome):** abre tu dirección → botón Compartir → *Añadir a pantalla de inicio* → Añadir.

Qué funciona en el móvil: cámara y galería para las fotos, ubicación («Cerca de mí», con permiso), mapa, y todo lo guardado en Supabase.
Sin conexión se abre la app, pero los datos, el mapa y las búsquedas necesitan internet.
No hay notificaciones push al móvil: los avisos se ven dentro de la app, en Perfil → Notificaciones.
En iPhone, iniciar sesión con Google desde la app instalada puede abrirse en Safari y no volver a la app; si pasa, usa correo y contraseña.
Para publicar una versión nueva: sube los archivos a GitHub y cambia el número en `V = 'nam-nam-vN'` de `sw.js` para que los móviles se actualicen.

## 3. Reglas de uso de los servicios gratuitos
- **Atribución:** el mapa muestra «© OpenStreetMap contributors» y Explorar indica «Datos © colaboradores de OpenStreetMap (ODbL)». No lo quites.
- **Teselas de OpenStreetMap:** valen para una beta con poco tráfico. No hagas descargas masivas. Si la app crece, cambia a un proveedor de teselas propio o de pago.
- **Overpass (servidor público):** se usa solo al pulsar *Buscar*, con un radio máximo de 3 km y caché en memoria. Si ves «servidor saturado», espera unos segundos.
- **Nominatim:** solo al buscar una ciudad (no se autocompleta mientras escribes), como exige su política.
- **Supabase gratuito:** tiene límites de base de datos, usuarios y almacenamiento, y puede pausar proyectos inactivos. Compruébalos en su web antes de abrir la beta.
- Son costes previstos, no una garantía de disponibilidad ilimitada.

## Qué se guarda
Todo en Supabase: perfiles, a quién sigues, locales guardados con nota, etiquetas y orden, reseñas con fotos, likes, comentarios,
etiquetas con notificación, grupos y chats (mensajes, restaurantes visitados con la media y listas «Por probar»), listas por tipo y notificaciones.
Los chats se actualizan solos cada pocos segundos mientras están abiertos, sin servicios de pago.

Aviso: el SQL y las reglas de seguridad no se han podido ejecutar contra un Supabase real al prepararlo. Cualquier error al ejecutarlo
sale en el SQL Editor con el número de línea; si pasa, pégamelo y lo corrijo.
Si algo no carga después, abre la consola del navegador (F12): Supabase indica qué política o tabla falla.

No hay ningún dato de ejemplo: todo lo que ves lo has creado tú.
