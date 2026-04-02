# Arquitectura: Tarjeta de Pre-Cumpleaños Interactiva

## Descripción del Proyecto

Página web estática e interactiva que funciona como tarjeta de pre-cumpleaños. Presenta una sección hero con animación parallax ligada al scroll y una sección de regalos con física simulada (cuerpos que caen y responden a clics del usuario).

---

## Stack Tecnológico

| Herramienta        | Versión  | Rol                                      |
|--------------------|----------|------------------------------------------|
| Astro              | ^6.1.3   | Framework para páginas estáticas         |
| Bun                | ^1.3.11  | Package manager y runtime de desarrollo  |
| Matter.js          | ^0.20.0  | Motor de física 2D para la sección de regalos |
| TypeScript         | Strict   | Tipado estático en todos los componentes |

---

## Estructura de Archivos

```
y-io/
├── ARCHITECTURE.md          ← Este archivo
├── astro.config.mjs         ← Configuración de Astro (output: static)
├── package.json
├── tsconfig.json
├── public/                  ← Assets estáticos (imágenes, fonts)
└── src/
    ├── layouts/
    │   └── Layout.astro     ← Layout base: <html>, <head>, estilos globales
    ├── components/
    │   ├── Hero.astro        ← Sección hero con animación de scroll
    │   └── GiftsSection.astro ← Sección de regalos con canvas de Matter.js
    └── pages/
        └── index.astro      ← Página principal; compone Hero + GiftsSection
```

---

## Arquitectura de Componentes

### `Layout.astro`
- Envuelve toda la página con `<html lang="es">` y `<head>` completo (meta, viewport, title)
- Define variables CSS globales (colores, tipografía)
- Aplica `scroll-behavior: smooth` y `overflow-x: hidden`

### `Hero.astro`
- Ocupa `100vh`
- Muestra un texto central (placeholder configurable)
- Contiene un `<script>` que escucha el evento `scroll` y actualiza la propiedad CSS `--scroll-progress`
- El CSS aplica `transform: translateX(calc(var(--scroll-progress) * 80px))` al texto para el efecto parallax
- Para el efecto "wave", cada letra es un `<span>` individual con un `transition-delay` incremental calculado desde JS

### `GiftsSection.astro`
- Se ubica debajo del hero, con `min-height: 100vh`
- Contiene un `<canvas>` que actúa como escenario de física
- Un `<div id="gift-overlay">` oculto que muestra el mensaje al hacer clic en una caja
- Toda la lógica de Matter.js vive en el `<script>` de este componente

### `index.astro`
- Importa y compone `<Layout>`, `<Hero>` y `<GiftsSection>`
- No contiene lógica propia

---

## Decisiones Técnicas Clave

### 1. Inicialización diferida con `IntersectionObserver`

Matter.js **no** se inicializa al cargar la página. En su lugar, un `IntersectionObserver` observa la sección de regalos y dispara `initMatter()` solo cuando la sección alcanza el 30% del viewport:

```js
const observer = new IntersectionObserver((entries) => {
  if (entries[0].isIntersecting) {
    initMatter();
    observer.disconnect(); // se dispara solo una vez
  }
}, { threshold: 0.3 });
observer.observe(document.getElementById('gifts-section'));
```

**Por qué:** Evita que el motor de física consuma CPU mientras el usuario solo está leyendo el Hero. Mejora el rendimiento inicial.

### 2. Detección de clics en cuerpos de Matter.js

Matter.js no tiene un sistema de eventos de clic nativo en el canvas. La solución:

1. Escuchar `click` en el elemento `<canvas>`
2. Calcular la posición del clic relativa al canvas con `getBoundingClientRect()`
3. Usar `Matter.Query.point(bodies, point)` para encontrar qué cuerpo fue clickeado

```js
canvas.addEventListener('click', (e) => {
  const { left, top } = canvas.getBoundingClientRect();
  const point = { x: e.clientX - left, y: e.clientY - top };
  const [hit] = Matter.Query.point(Composite.allBodies(engine.world), point);

  if (hit && !['wall', 'floor'].includes(hit.label)) {
    Composite.remove(engine.world, hit);
    showGiftMessage(hit.label);
  }
});
```

### 3. Cuerpos estáticos como límites del canvas

El suelo y las paredes son `Bodies.rectangle()` con `isStatic: true`. Se les asigna el `label: 'wall'` o `label: 'floor'` para excluirlos de la detección de clics.

### 4. Animación parallax sin librería externa

El efecto parallax usa exclusivamente una variable CSS actualizada por JS:

```js
// JS: actualiza la variable en cada frame de scroll
window.addEventListener('scroll', () => {
  const progress = window.scrollY / (document.documentElement.scrollHeight - window.innerHeight);
  heroEl.style.setProperty('--scroll-progress', progress.toString());
});
```

```css
/* CSS: la variable controla el desplazamiento */
.hero-text {
  transform: translateX(calc(var(--scroll-progress, 0) * 80px));
  transition: transform 0.05s linear;
}
```

---

## Comandos Principales

```bash
# Instalar dependencias
bun install

# Servidor de desarrollo (hot reload)
bun run dev          # → http://localhost:4321

# Build de producción (genera /dist)
bun run build

# Previsualizar el build de producción
bun run preview

# Verificar tipos TypeScript
bun run astro check
```

---

## Notas sobre Matter.js: Colisiones y Clics

### Detección de colisiones
Matter.js expone un sistema de eventos para colisiones:
```js
Matter.Events.on(engine, 'collisionStart', ({ pairs }) => {
  for (const pair of pairs) {
    const { bodyA, bodyB } = pair;
    // bodyA y bodyB son los dos cuerpos que colisionaron
  }
});
```
Las colisiones entre cajas y suelo se usan opcionalmente para efectos de sonido o animaciones visuales al impactar.

### `Matter.Query.point` vs. Mouse Constraint
- `Mouse.create()` + `MouseConstraint` permite arrastrar cuerpos, pero no distingue un clic simple de un drag.
- `Matter.Query.point()` es más preciso para detectar un tap/clic discreto sobre un cuerpo específico.
