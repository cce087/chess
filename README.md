# ♟️ Ajedrez Online Gratis — Supabase

Partidas de ajedrez en el navegador: crea una sala, comparte el enlace y juega en tiempo real. También puedes jugar contra la máquina (3 niveles), guardar partidas en PGN y ver repeticiones jugada a jugada. Todo en un único `index.html`, sin build ni backend propio.

## ✨ Funcionalidades

- **Partida online en tiempo real** (Supabase Realtime)
  - Crear sala → enlace `?room=XXXXX` → el rival entra y juega negras
  - Sincronización por FEN + turno vía tabla `partidas`
- **Jugar contra máquina** (sin sala)
  - `bajo`: aleatorio · `medio`: glotón (capturas, jaques, coronación) · `alto`: Stockfish.js (depth 12, Skill 12) con fallback a glotón
- **Tablero completo**: arrastrar + clic-para-mover, validación con chess.js, coronación automática a dama
- **Piezas capturadas** con imágenes por color, etiquetas de jugador, estado (jaque, mate, tablas, ahogado)
- **Guardar partida**: PGN descargable + historial en `localStorage` + repetición con barra (inicio/anterior/siguiente/final/volver en vivo)
- **Nombres persistentes** en `localStorage` (`ajedrez_nombre`, máx 20 caracteres)
- **Guardia anti-CDN-caído**: avisa qué librería falló en vez de morir en silencio

## 🧱 Stack (todo CDN, sin `package.json`)

| Librería | Versión | CDN | Uso |
|----------|---------|-----|-----|
| jQuery | 3.7.1 | code.jquery.com | DOM + eventos tablero |
| chess.js | 0.10.3 | cdnjs | reglas y validación |
| chessboard-js | 1.0.0 | cdnjs | tablero arrastrable (+ CSS) |
| supabase-js | 2.47.10 | jsdelivr (UMD) | salas + realtime (`window.supabase`) |
| stockfish.js | 10.0.2 | jsdelivr (carga dinámica solo en nivel alto) | motor local |

Piezas: `chessboardjs.com/img/chesspieces/wikipedia/{piece}.png`.

## 🗄️ Supabase: tabla `partidas`

```sql
create table public.partidas (
  id text primary key,          -- código sala, ej. 'K7Q2P'
  fen text not null,            -- posición FEN actual
  turno text not null,           -- 'w' | 'b'
  creador text,                 -- 'w' siempre (el creador juega blancas)
  nombre_w text,
  nombre_b text
);
```

Flujo: `insert` al crear → `update {fen, turno}` en cada jugada → `update {nombre_b}` al unirse → `select + channel postgres_changes UPDATE` para realtime.

> ⚠️ **RLS obligatorio**: la key del frontend es la `anon publishable` (pública por diseño). Activa Row Level Security y políticas restrictivas (solo `select/insert/update` sobre filas de sala, sin `delete`, sin service_role en cliente). Sin RLS cualquiera con el enlace —o adivinando IDs de 5 caracteres— puede leer/sobrescribir partidas.

## 🚀 Uso local y despliegue

```bash
git clone https://github.com/cce087/chess.git
cd chess
# Opción rápida: abre index.html con doble clic (el modo bot funciona 100% offline salvo CDNs)
# Opción recomendada: sírvelo por HTTP para evitar bloqueos de file://
python3 -m http.server 8000
# abre http://localhost:8000
```

Despliegue: cualquier hosting estático (GitHub Pages, Netlify, Cloudflare Pages). No hay build. Solo sube `index.html`.

## 🔒 Seguridad (resumen auditoría 2026-10-04)

- **TruffleHog**: 0 secretos verificados. La `supabaseAnonKey` (`sb_publishable_…`, `index.html:94`) es publicable por diseño — no rotar como si fuese privada, pero **verificar RLS**.
- **Semgrep**: 5 `missing-integrity` (CSS chessboard + jquery + chess.js + chessboard-js + supabase-js). Añadir `integrity="sha384-…"` + `crossorigin="anonymous"` (hashes en el informe).
- **Carga dinámica de Stockfish** (`index.html:414`) sin `integrity` — añadir `s.integrity` + `s.crossOrigin` al crear el `<script>`.
- **`innerHTML`**: `link-partida` (URL construida localmente) y `renderCaptured` (constantes) — riesgo bajo, pero migrar a `textContent`/`createElement` donde sea posible.
- **IDs de sala de 5 chars** (`Math.random`, ~60M combinaciones): aceptable para juego casual, insuficiente contra enumeración dedicada — considerar IDs más largos + rate-limit/RLS.
- **Sin CI/CD**: añadir workflow (TruffleHog + Semgrep + SRI-check), `dependabot.yml` con `cooldown`, pre-commit hooks. Ver repos hermanos como plantilla.
- **Dependencias CDN antiguas**: `chess.js@0.10.3` y `chessboard-js@1.0.0` sin mantenimiento — evaluar migrar a `@chess/chess.js` y `chessboard-element`.

## 📄 Licencia

Sin `LICENSE` declarado. Añade uno (MIT recomendado para proyecto web abierto) si quieres contribuciones.
