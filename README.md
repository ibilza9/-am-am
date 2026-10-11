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
create policy venues_read on public.venues for select to authenticated using (true);
create policy venues_insert on public.venues for insert to authenticated with check (created_by = auth.uid());
create policy saved_own on public.saved_venues for all to authenticated
  using (user_id = auth.uid()) with check (user_id = auth.uid());

create policy follows_read on public.follows for select to authenticated
  using (follower = auth.uid() or followed = auth.uid());
create policy follows_insert on public.follows for insert to authenticated with check (follower = auth.uid());
create policy follows_delete on public.follows for delete to authenticated using (follower = auth.uid());

grant select, update on public.profiles to authenticated;
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

## 2. Publicar en GitHub Pages
1. Crea un repositorio y sube `index.html`.
2. **Settings → Pages → Deploy from a branch → main / (root)**.
3. Tu app quedará en `https://TU-USUARIO.github.io/TU-REPO/`. Úsala como *Site URL* en Supabase.

## 3. Reglas de uso de los servicios gratuitos
- **Atribución:** el mapa muestra «© OpenStreetMap contributors» y Explorar indica «Datos © colaboradores de OpenStreetMap (ODbL)». No lo quites.
- **Teselas de OpenStreetMap:** valen para una beta con poco tráfico. No hagas descargas masivas. Si la app crece, cambia a un proveedor de teselas propio o de pago.
- **Overpass (servidor público):** se usa solo al pulsar *Buscar*, con un radio máximo de 3 km y caché en memoria. Si ves «servidor saturado», espera unos segundos.
- **Nominatim:** solo al buscar una ciudad (no se autocompleta mientras escribes), como exige su política.
- **Supabase gratuito:** tiene límites de base de datos, usuarios y almacenamiento, y puede pausar proyectos inactivos. Compruébalos en su web antes de abrir la beta.
- Son costes previstos, no una garantía de disponibilidad ilimitada.

## Qué se guarda ahora y qué falta
Se guardan en Supabase: perfiles, a quién sigues (y la búsqueda de personas en Descubrir), locales guardados, y tus notas y etiquetas (tu catálogo / lista de deseos).
Siguen **en memoria** (se pierden al recargar): reseñas, fotos, grupos, chats, comentarios y listas por tipo.
Ver las publicaciones de otras personas en Descubrir y chatear con ellas necesita tablas propias (reseñas, mensajes, fotos en Supabase Storage).
No hay ningún dato de ejemplo: todo lo que ves lo has creado tú.
