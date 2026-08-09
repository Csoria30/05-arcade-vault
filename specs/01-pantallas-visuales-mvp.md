# SPEC 01 — MVP visual: pantallas de Arcade Vault

> **Estado:** Aprobado
> **Depende de:** Ninguno
> **Fecha:** 2026-08-08
> **Objetivo:** Portar las cinco pantallas del prototipo de referencia (Biblioteca, Detalle, Reproductor, Auth y Salón de la Fama) a rutas reales de Next.js App Router como una interfaz puramente visual, sin lógica de juego real ni persistencia.

## Por qué existe este spec

El prototipo en `references/templates/` es una SPA aislada (React + Babel por CDN, sin build) que enruta por hash y persiste sesión/puntuaciones en `localStorage`. Este spec decide portarla a la arquitectura nativa de Next.js App Router (rutas de archivo reales) y reemplazar la persistencia en `localStorage` por estado en memoria (React Context), porque el pedido es solo la capa visual — no hay backend ni se requiere que la sesión sobreviva a un recargado de página.

## Alcance

**Incluye:**

- Pantalla **Biblioteca** (`/`): hero, buscador, chips de categoría, grid de tarjetas de juego con efecto tilt al pasar el mouse.
- Pantalla **Detalle** (`/juegos/[id]`): info del juego, tags, stats, leaderboard mock de 10 posiciones, botón "Jugar ahora".
- Pantalla **Reproductor** (`/juegos/[id]/jugar`): HUD (jugador, puntuación, vidas, nivel), marco CRT animado con puntuación que sube sola, pausa/fin, modal de fin de partida con guardado decorativo de puntuación.
- Pantalla **Auth** (`/login`): tabs iniciar sesión / crear cuenta, formulario, botón "jugar como invitado", botones sociales ficticios.
- Pantalla **Salón de la Fama** (`/salon`): tabs por juego, podio top 3, tabla de 12 posiciones, fila "tu mejor marca" si hay sesión iniciada.
- **Nav** compartido (logo, links, contador de créditos estático, botón login/logout, menú hamburguesa responsive) y **footer**, integrados en `app/layout.tsx`.
- Estado de sesión de usuario (login falso) compartido entre pantallas vía React Context, sin backend.
- Datos mock de los 8 juegos y generación determinista de leaderboards, portados de `references/templates/data.jsx`.
- Todos los estilos, fuentes y animaciones ya existentes en `app/globals.css` y `app/layout.tsx` se reutilizan tal cual (ya están portados del tema del template).

**Fuera de alcance (para specs futuros):**

- Cualquier lógica de juego real (los 8 juegos siguen siendo solo tarjetas + un mock visual de "reproductor").
- Backend, API routes, base de datos o autenticación real (OAuth, validación de contraseña).
- Persistencia entre sesiones (`localStorage`, cookies, IndexedDB): el estado vive solo en memoria del cliente y se resetea al recargar la página.
- Sistema de créditos funcional (el contador "CRÉDITOS · 03" queda estático, como en el template).
- Tests automatizados (el repo no tiene test runner configurado).
- SEO/metadata específico por pantalla más allá del `metadata` ya definido en `app/layout.tsx`.

## Modelo de datos

Este spec porta la data mock existente en `references/templates/data.jsx` a TypeScript, sin cambiar su forma. Vive en `lib/data.ts`:

```ts
export type Game = {
  id: string;
  title: string;
  short: string;
  long: string;
  cat: "ARCADE" | "PUZZLE" | "SHOOTER" | "VERSUS";
  cover: string; // clase CSS de app/globals.css, p.ej. "cover-bricks"
  color: "cyan" | "magenta" | "yellow" | "green";
  best: number;
  plays: string;
};

export const GAMES: Game[]; // los 8 juegos del template
export const CATS: string[]; // ["TODOS", "ARCADE", "PUZZLE", "SHOOTER", "VERSUS"]
export const PLAYERS: string[]; // nombres para generar leaderboards

export type ScoreRow = {
  rank: number;
  name: string;
  score: number;
  date: string;
};
export function seededScores(seed: number, count?: number): ScoreRow[];
```

Estado de sesión, en `lib/session.tsx` (Context + provider cliente):

```ts
export type SessionUser = { name: string } | null;

type SessionContextValue = {
  user: SessionUser;
  login: (user: SessionUser) => void;
  logout: () => void;
};
```

No hay ninguna otra estructura de datos nueva: no se modela el resultado de "guardar puntuación" porque no se persiste (ver Decisiones).

## Plan de implementación

1. Crear `lib/data.ts` con `GAMES`, `CATS`, `PLAYERS` y `seededScores`, tipado en TypeScript, portado de `references/templates/data.jsx`.
2. Crear `lib/session.tsx` con el `SessionProvider` (Context de React, `"use client"`) y el hook `useSession()`; envolver `{children}` en `app/layout.tsx` con el provider.
3. Crear `components/nav.tsx` (puerto de `nav.jsx`) usando `useSession()` para el botón login/logout, con menú desktop, hamburguesa y panel móvil; integrarlo junto con el footer (puerto del `<footer>` de `app.jsx`) en `app/layout.tsx`.
4. Crear `components/game-card.tsx` (tarjeta con tilt hover, puerto de `GameCard` en `biblioteca.jsx`) y reescribir `app/page.tsx` como la pantalla Biblioteca (hero, buscador, chips, grid), puerto de `biblioteca.jsx`, eliminando el contenido de scaffold de `create-next-app`.
5. Crear `app/juegos/[id]/page.tsx` — pantalla Detalle, puerto de `detalle.jsx`; si el `id` no existe en `GAMES`, llamar a `notFound()`.
6. Crear `app/juegos/[id]/jugar/page.tsx` — pantalla Reproductor, puerto de `reproductor.jsx`; usa `useSession()` para prellenar el nombre del jugador en el modal de fin de partida.
7. Crear `app/login/page.tsx` — pantalla Auth, puerto de `auth.jsx`; al enviar el formulario o pulsar "invitado" llama a `session.login(...)` y navega a `/` con `useRouter().push`.
8. Crear `app/salon/page.tsx` — pantalla Salón de la Fama, puerto de `salon.jsx`; usa `useSession()` para mostrar (o no) la fila "tu mejor marca".
9. Revisar los tres breakpoints responsive (840px nav, 900px detalle, 720px salón/tabla) contra `app/globals.css` y confirmar que no falta ninguna clase del template en `app/globals.css` (ya portado; solo ajustar si algo quedó pendiente).

## Criterios de aceptación

- [ ] `/` muestra la Biblioteca con hero, buscador y chips de categoría; buscar por texto y filtrar por categoría actualiza el grid sin recargar la página.
- [ ] Cada tarjeta de juego navega a `/juegos/[id]` al hacer click en la tarjeta o en el botón "JUGAR".
- [ ] `/juegos/[id]` muestra info del juego, tags, stats y un leaderboard de 10 posiciones; visitar `/juegos/un-id-inexistente` devuelve 404.
- [ ] El botón "JUGAR AHORA" en Detalle navega a `/juegos/[id]/jugar`.
- [ ] `/juegos/[id]/jugar` muestra el HUD (jugador, puntuación, vidas, nivel) y el marco CRT animado; "PAUSA" detiene el incremento de puntuación y "REANUDAR" lo retoma; "FIN" abre el modal de fin de partida.
- [ ] En el modal de fin de partida: el input de nombre es editable, "GUARDAR PUNTUACIÓN" reemplaza el input por el mensaje "▸ PUNTUACIÓN GUARDADA\_" sin persistir nada, "JUGAR DE NUEVO" reinicia el HUD a su estado inicial y "VOLVER AL VAULT" navega a `/`.
- [ ] `/login` muestra los tabs "INICIAR SESIÓN"/"CREAR CUENTA"; enviar el formulario o pulsar "JUGAR COMO INVITADO" actualiza la sesión y redirige a `/`.
- [ ] Tras iniciar sesión, el Nav en cualquier pantalla (Biblioteca, Detalle, Reproductor, Salón) muestra el nombre de usuario en vez de "Iniciar Sesión".
- [ ] Recargar la página (F5) después de iniciar sesión vuelve a mostrar "Iniciar Sesión" en el Nav (sin persistencia).
- [ ] `/salon` muestra tabs por juego, un podio con los 3 primeros puestos y una tabla de 12 posiciones; cambiar de tab regenera el ranking del juego seleccionado.
- [ ] Por debajo de 840px de ancho, el Nav oculta los links y el contador de créditos y muestra el botón hamburguesa, que abre el panel lateral con los mismos enlaces.
- [ ] `app/page.tsx` ya no contiene ningún resto del scaffold de `create-next-app` (logo de Next.js/Vercel, texto "To get started...").
- [ ] `npm run build` compila sin errores de TypeScript ni de ESLint.

## Decisiones

- **Sí:** rutas reales de Next.js App Router en vez de replicar el enrutado por hash del template. Es la convención nativa del framework que ya usa el proyecto y da URLs compartibles por pantalla.
- **Sí:** esquema de URLs anidado `/juegos/[id]` y `/juegos/[id]/jugar` en vez de rutas planas (`/detalle/[id]`, `/jugar/[id]`). Refleja que "jugar" depende de un juego existente.
- **Sí:** estado de sesión en un Context de React (`lib/session.tsx`) que envuelve `app/layout.tsx`. Sobrevive la navegación entre pantallas (client-side) sin necesitar `localStorage`.
- **No:** persistir sesión o puntuaciones en `localStorage`/backend, a diferencia del template (`av_user`, `av_scores`). El pedido es solo la parte visual; el estado se resetea al recargar.
- **Sí:** el botón "GUARDAR PUNTUACIÓN" solo dispara el feedback visual del template (mensaje de confirmación), sin escribir el resultado en ningún lado.
- **Sí:** la pantalla Reproductor mantiene el mock completo del template (HUD + CRT animado + puntuación que sube sola + modal de fin), porque ya es un placeholder visual, no un motor de juego real — no contradice "no implementar ningún juego".
- **Sí:** ruta de login en `/login` (no `/auth`, que era el nombre interno de la pantalla en el template). Sigue la convención estándar de apps web.
- **No:** convertir los estilos existentes (`app/globals.css`) a utilidades de Tailwind. Ya están portados del template como CSS global y cubren animaciones/efectos complejos (CRT, tilt, clip-path) que no ganan nada expresándose en Tailwind.

## Qué **no** incluye este spec

- Lógica de juego real para ninguno de los 8 juegos listados.
- Backend, API routes, base de datos o autenticación real.
- Persistencia de sesión o puntuaciones entre recargas de página.
- Sistema de créditos funcional.
- Tests automatizados.

Cada uno de estos, si se implementa, va en su propio spec.
