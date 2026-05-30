# Space Art Pack — Solar System

Пиксельный арт солнечной системы для космической игры. 8 планет + Солнце + ракета.

**Общие параметры:**
- Размер кадра спрайта: **40×40 px** (для планет и Солнца), **14×19 px** (для ракеты)
- Стиль: pixel art, плавная цветовая обводка, освещение сверху-слева (направление света: dx=-4, dy=-4)
- Фон: `#0A1424` (тёмно-синий космос)
- Обводка планет: `#1A1228`
- Технология: процедурная отрисовка через шумовую функцию `fractalNoise`. Каждый кадр вычисляется заново, что даёт плавную анимацию вращения без перескоков пикселей.

---

## Превью всех объектов

![Solar System Overview](sprites/all_overview.png)

Интерактивная версия с анимацией: [`planets_preview.html`](planets_preview.html)

---

## Утилитарные функции (общие)

Эти функции используются всеми планетами. Они выполняют математику для сферического маппирования и генерации текстур.

```javascript
// Константы кадра
const SPRITE_W = 40, SPRITE_H = 40;
const CX = (SPRITE_W - 1) / 2;
const CY = (SPRITE_H - 1) / 2;

function inCircle(x, y, radius) {
  const dx = x - CX, dy = y - CY;
  return dx*dx + dy*dy <= radius*radius;
}

function brightness(x, y, radius) {
  const dx = x - CX, dy = y - CY;
  const lx = dx - (-4), ly = dy - (-4);
  return 1.0 - Math.sqrt(lx*lx + ly*ly) / (radius * 1.7);
}

function sphereUV(x, y, radius) {
  const dx = (x - CX) / radius;
  const dy = (y - CY) / radius;
  const r2 = dx*dx + dy*dy;
  if (r2 > 1) return null;
  const z = Math.sqrt(Math.max(0, 1 - r2));
  const v = Math.asin(dy) / Math.PI + 0.5;
  const cosLat = Math.cos(Math.asin(dy));
  let u;
  if (cosLat < 0.01) u = 0.5;
  else u = Math.atan2(dx, z) / (Math.PI * 2) + 0.5;
  return { u, v };
}

function fractalNoise(u, v, scale, octaves) {
  let val = 0;
  let amp = 1;
  let freq = scale;
  let norm = 0;
  for (let i = 0; i < octaves; i++) {
    const n = Math.sin(u * freq * Math.PI * 2) * Math.cos(v * freq * Math.PI) +
              Math.sin(u * freq * Math.PI * 2.7 + 1.3) * Math.cos(v * freq * Math.PI * 1.3 + 0.7);
    val += n * amp;
    norm += amp;
    amp *= 0.5;
    freq *= 2;
  }
  return val / norm;
}

```

**Как работает анимация:** в главном цикле `rotation` плавно увеличивается каждый кадр. Внутри `drawXxx` функция превращает каждый пиксель в сферические координаты `(u, v)`, добавляет `rotation` к `u`, и через `fractalNoise(u + rotation, v)` получает значение шума. При сдвиге `rotation` шум плавно меняется — это даёт **непрерывное вращение без перескоков**.

---

## Солнце (Sun)

![Солнце](sprites/sun.png)

Жёлто-оранжевая звезда с белым раскалённым ядром. Освещение **радиальное** (от центра наружу), а не сбоку как у планет — Солнце само источник света. На поверхности видна "грануляция" (мелкая текстура) и тёмные пятна (sunspots), которые медленно дрейфуют.

**Радиус:** 14 px  
**Палитра:** От белого центра к красной короне: белый раскал, светло-жёлтый, тёплый жёлтый, оранжевый, тёмно-оранжевый, красная кромка.  
**Особенности:** Радиальное освещение, грануляция поверхности, тёмные пятна.

**Палитра:**

| Код | Hex | Назначение |
|-----|-----|------------|
| `W` | `#FFF8DC` | самый светлый блик |
| `L` | `#FCE490` | светлый тон |
| `l` | `#F8B840` | средне-светлый |
| `m` | `#F48C28` | средний |
| `dk` | `#D45018` | тёмный |
| `rd` | `#A02C10` | красный край |
| `D` | `#1A1228` | обводка |

**Параметры анимации:**

- Размер на экране: `1.3` × базовый scale (рекомендация: scale=4)
- Скорость вращения: `0.0002` рад/мс (full rotation ≈ 5000 сек)

**JS-функция отрисовки:**

```javascript
function drawSun(rotation, time) {
  const radius = 14;
  const P = PALETTES.sun;
  
  for (let y = 0; y < SPRITE_H; y++) {
    for (let x = 0; x < SPRITE_W; x++) {
      if (!inCircle(x, y, radius)) continue;
      const d = Math.sqrt((x-CX)**2 + (y-CY)**2);
      const uv = sphereUV(x, y, radius);
      if (!uv) continue;
      
      const uRot = (uv.u + rotation) % 1.0;
      // Радиальная яркость — от центра к краю
      const radial = 1.0 - d / radius;
      // Грануляция поверхности
      const granule = fractalNoise(uRot, uv.v, 4, 3) * 0.15;
      // Тёмные пятна
      const sunspots = fractalNoise(uRot + 0.5, uv.v + 0.3, 2, 2);
      
      const bright = radial + granule;
      
      let base;
      if (d > radius - 0.7) base = P.rd;
      else if (d > radius - 1.5) base = P.dk;
      else if (bright > 0.85) base = P.W;
      else if (bright > 0.65) base = P.L;
      else if (bright > 0.45) base = P.l;
      else if (bright > 0.25) base = P.m;
      else base = P.dk;
      
      // Тёмные пятна
      if (sunspots > 0.55 && d < radius - 2) {
        if (base === P.W) base = P.L;
        else if (base === P.L) base = P.m;
        else if (base === P.l) base = P.dk;
        else if (base === P.m) base = P.dk;
      }
      
      setPixel(x, y, base);
    }
  }
}
```

---

## Меркурий (Mercury)

![Меркурий](sprites/mercury.png)

Маленькая серо-коричневая планета. Поверхность покрыта кратерами разных размеров — это даёт характерную "побитую" текстуру. Без атмосферы, поэтому видны чёткие тени.

**Радиус:** 9 px  
**Палитра:** Три серо-коричневых оттенка + тёмная обводка.  
**Особенности:** Кратеры через шум (scale=5) + блики ободков (scale=8).

**Палитра:**

| Код | Hex | Назначение |
|-----|-----|------------|
| `L` | `#A89A88` | светлый тон |
| `m` | `#74675A` | средний |
| `d` | `#4A3D32` | тёмный |
| `D` | `#1A1228` | обводка |

**Параметры анимации:**

- Размер на экране: `0.55` × базовый scale (рекомендация: scale=4)
- Скорость вращения: `0.00013` рад/мс (full rotation ≈ 7692 сек)

**JS-функция отрисовки:**

```javascript
function drawMercury(rotation, time) {
  const radius = 9;
  const P = PALETTES.mercury;
  
  for (let y = 0; y < SPRITE_H; y++) {
    for (let x = 0; x < SPRITE_W; x++) {
      if (!inCircle(x, y, radius)) continue;
      const d = Math.sqrt((x-CX)**2 + (y-CY)**2);
      const b = brightness(x, y, radius);
      const uv = sphereUV(x, y, radius);
      if (!uv) continue;
      
      const uRot = (uv.u + rotation) % 1.0;
      // Шум кратеров и теней
      const noise = fractalNoise(uRot, uv.v, 5, 2);
      const noise2 = fractalNoise(uRot + 0.7, uv.v + 0.3, 8, 2);
      
      // Базовый цвет по освещению
      let base;
      if (d > radius - 0.7) base = b < 0.5 ? P.D : P.d;
      else if (b > 0.65) base = P.L;
      else if (b > 0.4) base = P.m;
      else if (b > 0.15) base = P.d;
      else base = P.D;
      
      // Модификация шумом
      if (noise > 0.4) {
        // Тёмные кратеры
        if (base === P.L) base = P.m;
        else if (base === P.m) base = P.d;
      } else if (noise2 > 0.5 && b > 0.45) {
        // Светлые ободки
        if (base === P.m) base = P.L;
        else if (base === P.d) base = P.m;
      }
      
      setPixel(x, y, base);
    }
  }
}
```

---

## Венера (Venus)

![Венера](sprites/venus.png)

Жёлто-бежевая планета с густой атмосферой. Облачные завихрения создают мягкую, без чётких деталей поверхность. Все тени плавные, край размытый.

**Радиус:** 12 px  
**Палитра:** Светло-жёлтый → тёмно-оранжевый, четыре оттенка.  
**Особенности:** Облачные завихрения через крупный шум (scale=2) + мелкие блики (scale=5).

**Палитра:**

| Код | Hex | Назначение |
|-----|-----|------------|
| `L` | `#F4E090` | светлый тон |
| `l` | `#E0B85C` | средне-светлый |
| `m` | `#B88030` | средний |
| `d` | `#7C4F1A` | тёмный |
| `D` | `#1A1228` | обводка |

**Параметры анимации:**

- Размер на экране: `0.75` × базовый scale (рекомендация: scale=4)
- Скорость вращения: `0.00012` рад/мс (full rotation ≈ 8333 сек)

**JS-функция отрисовки:**

```javascript
function drawVenus(rotation, time) {
  const radius = 12;
  const P = PALETTES.venus;
  
  for (let y = 0; y < SPRITE_H; y++) {
    for (let x = 0; x < SPRITE_W; x++) {
      if (!inCircle(x, y, radius)) continue;
      const d = Math.sqrt((x-CX)**2 + (y-CY)**2);
      const b = brightness(x, y, radius);
      const uv = sphereUV(x, y, radius);
      if (!uv) continue;
      
      const uRot = (uv.u + rotation) % 1.0;
      // Облачные завихрения — крупный шум
      const noiseCloud = fractalNoise(uRot, uv.v, 2, 3);
      // Мелкие детали
      const noiseDetail = fractalNoise(uRot + 0.4, uv.v + 0.2, 5, 2);
      
      let base;
      if (d > radius - 0.7) base = b < 0.4 ? P.D : P.d;
      else if (b > 0.75) base = P.L;
      else if (b > 0.5) base = P.l;
      else if (b > 0.25) base = P.m;
      else base = P.d;
      
      // Светлые облачные пятна
      if (noiseCloud > 0.2) {
        if (base === P.m) base = P.l;
        else if (base === P.l) base = P.L;
      } else if (noiseCloud < -0.3) {
        // Тёмные провалы
        if (base === P.L) base = P.l;
        else if (base === P.l) base = P.m;
      }
      
      // Мелкие детали — яркие точки
      if (noiseDetail > 0.5 && b > 0.4) {
        if (base === P.l) base = P.L;
      }
      
      setPixel(x, y, base);
    }
  }
}
```

---

## Земля (Earth)

![Земля](sprites/earth.png)

Сине-зелёная планета с белыми облаками и полярными шапками. Континенты процедурно генерируются через шум — выглядят как естественные материковые массы, а не точные карты. Облака движутся **независимо** от планеты — заметно отдельным слоем.

**Радиус:** 12 px  
**Палитра:** Океан (3 оттенка синего), суша (2 оттенка зелёного), белый (облака/шапки).  
**Особенности:** Шум континентов (порог > 0.15) + независимый шум облаков (порог > 0.5) + полярные шапки по широте > 0.42.

**Палитра:**

| Код | Hex | Назначение |
|-----|-----|------------|
| `B` | `#5C95D0` | светлый океан |
| `b` | `#2C5C9C` | средний океан |
| `o` | `#1A3A6E` | тёмный океан |
| `G` | `#7CC480` | светлая суша |
| `g` | `#3A7A48` | тёмная суша |
| `L` | `#F0F4F8` | светлый тон |
| `D` | `#1A1228` | обводка |

**Параметры анимации:**

- Размер на экране: `0.8` × базовый scale (рекомендация: scale=4)
- Скорость вращения: `0.00015` рад/мс (full rotation ≈ 6667 сек)

**JS-функция отрисовки:**

```javascript
function drawEarth(rotation, time) {
  const radius = 12;
  const P = PALETTES.earth;
  const t = time * 0.001;
  // Облака дрейфуют независимо от планеты
  const cloudDrift = t * 0.020;
  
  for (let y = 0; y < SPRITE_H; y++) {
    for (let x = 0; x < SPRITE_W; x++) {
      if (!inCircle(x, y, radius)) continue;
      const d = Math.sqrt((x-CX)**2 + (y-CY)**2);
      const b = brightness(x, y, radius);
      const uv = sphereUV(x, y, radius);
      if (!uv) continue;
      
      const uRotLand = (uv.u + rotation) % 1.0;
      const uRotCloud = (uv.u + rotation + cloudDrift) % 1.0;
      
      // Шум континентов
      const noiseLand = fractalNoise(uRotLand, uv.v, 2.5, 3);
      // Шум облаков (другая частота, движется отдельно)
      const noiseCloud = fractalNoise(uRotCloud + 0.3, uv.v + 0.2, 4, 2);
      
      // Полярные шапки (по широте)
      const polarDist = Math.abs(uv.v - 0.5);
      const isPole = polarDist > 0.42;
      
      const isLand = noiseLand > 0.15;
      const isCloud = noiseCloud > 0.5;
      
      let base;
      if (isPole) {
        if (d > radius - 0.7) base = b < 0.4 ? P.D : P.o;
        else base = P.L;
      } else if (isCloud && !isLand) {
        // Облака только над океаном (на суше реже)
        base = P.L;
      } else if (isLand) {
        // Суша
        if (d > radius - 0.7) base = b < 0.4 ? P.D : P.g;
        else if (b > 0.55) base = P.G;
        else base = P.g;
      } else {
        // Океан
        if (d > radius - 0.7) base = b < 0.4 ? P.D : P.o;
        else if (b > 0.65) base = P.B;
        else if (b > 0.35) base = P.b;
        else base = P.o;
      }
      
      setPixel(x, y, base);
    }
  }
}
```

---

## Марс (Mars)

![Марс](sprites/mars.png)

Красно-оранжевая планета с белыми полярными шапками и тёмными "мария" (древние лавовые равнины). Текстура мерцает за счёт пыли в атмосфере.

**Радиус:** 10 px  
**Палитра:** От светло-оранжевого до тёмно-красного + белые шапки.  
**Особенности:** Шум мария (scale=2, порог > 0.3) + шум кратеров (scale=6, порог > 0.6) + шапки по широте.

**Палитра:**

| Код | Hex | Назначение |
|-----|-----|------------|
| `L` | `#F08C4C` | светлый тон |
| `l` | `#D8682C` | средне-светлый |
| `m` | `#A8451C` | средний |
| `d` | `#702C14` | тёмный |
| `W` | `#F8E4D0` | самый светлый блик |
| `D` | `#1A1228` | обводка |

**Параметры анимации:**

- Размер на экране: `0.68` × базовый scale (рекомендация: scale=4)
- Скорость вращения: `0.00013` рад/мс (full rotation ≈ 7692 сек)

**JS-функция отрисовки:**

```javascript
function drawMars(rotation, time) {
  const radius = 10;
  const P = PALETTES.mars;
  
  for (let y = 0; y < SPRITE_H; y++) {
    for (let x = 0; x < SPRITE_W; x++) {
      if (!inCircle(x, y, radius)) continue;
      const d = Math.sqrt((x-CX)**2 + (y-CY)**2);
      const b = brightness(x, y, radius);
      const uv = sphereUV(x, y, radius);
      if (!uv) continue;
      
      const uRot = (uv.u + rotation) % 1.0;
      // Шум тёмных областей "мария"
      const noiseDark = fractalNoise(uRot, uv.v, 2, 3);
      // Шум кратеров (мелкая частота)
      const noiseCraters = fractalNoise(uRot + 0.5, uv.v + 0.3, 6, 2);
      
      // Полярные шапки
      const polarDist = Math.abs(uv.v - 0.5);
      const isPole = polarDist > 0.42;
      
      let base;
      if (isPole) {
        if (d > radius - 0.7) base = b < 0.4 ? P.D : P.d;
        else base = P.W;
      } else {
        // Базовый цвет
        if (d > radius - 0.7) base = b < 0.4 ? P.D : P.d;
        else if (b > 0.7) base = P.L;
        else if (b > 0.5) base = P.l;
        else if (b > 0.3) base = P.m;
        else base = P.d;
        
        // Тёмные мария
        if (noiseDark > 0.3) {
          if (base === P.L) base = P.l;
          else if (base === P.l) base = P.m;
          else if (base === P.m) base = P.d;
        } else if (noiseCraters > 0.6 && b > 0.4) {
          // Светлые блики (кратеры)
          if (base === P.m) base = P.l;
          else if (base === P.l) base = P.L;
        }
      }
      
      setPixel(x, y, base);
    }
  }
}
```

---

## Юпитер (Jupiter)

![Юпитер](sprites/jupiter.png)

Газовый гигант с горизонтальными полосами разных оттенков и Большим Красным Пятном. Полосы изгибаются по сферической геометрии — не прямые линии. БКП — крупный овал, "уходит за горизонт" при вращении.

**Радиус:** 16 px  
**Палитра:** Бежевый→коричневый, 4 оттенка + красные тона для БКП.  
**Особенности:** 12 горизонтальных полос по широте с волной (sin(u*6+v*4)) + овальное БКП на lon=0.3, lat=0.18.

**Палитра:**

| Код | Hex | Назначение |
|-----|-----|------------|
| `L` | `#F0DCB0` | светлый тон |
| `l` | `#C89860` | средне-светлый |
| `m` | `#8C5C2C` | средний |
| `d` | `#583612` | тёмный |
| `R` | `#C84020` | светлое пятно |
| `r` | `#8C2810` | тёмное пятно |
| `D` | `#1A1228` | обводка |

**Параметры анимации:**

- Размер на экране: `1.2` × базовый scale (рекомендация: scale=4)
- Скорость вращения: `8e-05` рад/мс (full rotation ≈ 12500 сек)

**JS-функция отрисовки:**

```javascript
function drawJupiter(rotation, time) {
  const radius = 16;
  const P = PALETTES.jupiter;
  
  for (let y = 0; y < SPRITE_H; y++) {
    for (let x = 0; x < SPRITE_W; x++) {
      if (!inCircle(x, y, radius)) continue;
      const d = Math.sqrt((x-CX)**2 + (y-CY)**2);
      const b = brightness(x, y, radius);
      const uv = sphereUV(x, y, radius);
      if (!uv) { setPixel(x, y, P.m); continue; }
      
      const uRot = (uv.u + rotation) % 1.0;
      const base = jupiterBand(uv.v, uRot * Math.PI * 2);
      
      let c;
      if (d > radius - 0.7) c = b < 0.4 ? P.D : P.d;
      else if (b < 0.2) {
        if (base === 'L' || base === 'l') c = P.m;
        else if (base === 'm') c = P.d;
        else c = P.d;
      } else if (b < 0.4) {
        if (base === 'L') c = P.l;
        else c = P[base];
      } else if (b > 0.8) {
        if (base === 'm') c = P.l;
        else if (base === 'l') c = P.L;
        else c = P[base];
      } else {
        c = P[base];
      }
      setPixel(x, y, c);
    }
  }
  
  // БКП — большой и заметный, движется с долготой
  const spotLon = 0.3;
  const spotLat = 0.18;
  const lon = (spotLon - rotation + 1.0) % 1.0;
  const lonRad = (lon - 0.5) * Math.PI;
  const latRad = spotLat * Math.PI;
  const cosLat = Math.cos(latRad);
  const px = CX + Math.sin(lonRad) * radius * cosLat;
  const py = CY + Math.sin(latRad) * radius;
  const z = Math.cos(lonRad) * cosLat;
  
  if (z > 0.2) {
    const cxi = Math.round(px), cyi = Math.round(py);
    const visibility = z;
    // Овал 5x3 — но сжимаем по x у горизонта
    const ovalW = 3 * visibility;
    const ovalH = 1.5;
    for (let dy = -2; dy <= 2; dy++) {
      for (let dx = -4; dx <= 4; dx++) {
        if ((dx*dx)/(ovalW*ovalW) + (dy*dy)/(ovalH*ovalH) <= 1.0) {
          const nx = cxi + dx, ny = cyi + dy;
          if (inCircle(nx, ny, radius - 1)) setPixel(nx, ny, P.r);
        }
      }
    }
    for (let dy = -1; dy <= 1; dy++) {
      for (let dx = -3; dx <= 3; dx++) {
        const innerW = ovalW * 0.7, innerH = ovalH * 0.6;
        if ((dx*dx)/(innerW*innerW) + (dy*dy)/(innerH*innerH) <= 1.0) {
          const nx = cxi + dx, ny = cyi + dy;
          if (inCircle(nx, ny, radius - 1)) setPixel(nx, ny, P.R);
        }
      }
    }
    if (inCircle(cxi, cyi, radius - 1)) setPixel(cxi, cyi, P.r);
  }
}
```

---

## Сатурн (Saturn)

![Сатурн](sprites/saturn.png)

Бежевый газовый гигант с **горизонтальными кольцами**. Кольца — эллипс с эффектом "сзади/спереди": задняя часть видна выше планеты, передняя перекрывает планету снизу.

**Радиус:** 10 px  
**Палитра:** Бежевые полосы (как Юпитер, но мягче) + 2 оттенка для колец.  
**Особенности:** Полосы + эллиптические кольца (внешний 18x4, внутренний 12x2.7) с эффектом перекрытия.

**Палитра:**

| Код | Hex | Назначение |
|-----|-----|------------|
| `L` | `#F0DCB0` | светлый тон |
| `l` | `#D0A878` | средне-светлый |
| `m` | `#945C28` | средний |
| `d` | `#583612` | тёмный |
| `K` | `#E0C088` | светлое кольцо |
| `k` | `#90683C` | тёмное кольцо |
| `D` | `#1A1228` | обводка |

**Параметры анимации:**

- Размер на экране: `1.05` × базовый scale (рекомендация: scale=4)
- Скорость вращения: `0.00025` рад/мс (full rotation ≈ 4000 сек)

**JS-функция отрисовки:**

```javascript
function drawSaturn(rotation, time) {
  const radius = 10;
  const P = PALETTES.saturn;
  
  // Параметры кольца
  const ringOuterA = 18, ringOuterB = 4;
  const ringInnerA = 12, ringInnerB = 2.7;
  
  function ringColorAt(dx, dy) {
    if ((dx/ringInnerA)**2 + (dy/ringInnerB)**2 <= 1.0) return null;
    if ((dx/ringOuterA)**2 + (dy/ringOuterB)**2 > 1.0) return null;
    const rNorm = Math.abs(dx)/ringOuterA + Math.abs(dy)/ringOuterB * 0.5;
    if (rNorm > 0.92) return P.k;
    if (rNorm > 0.75) return P.K;
    if (rNorm > 0.65) return P.k;
    return P.K;
  }
  
  // 1. Задняя половина колец
  for (let y = 0; y < SPRITE_H; y++) {
    for (let x = 0; x < SPRITE_W; x++) {
      const dy = y - CY;
      if (dy >= 0) continue;
      const dx = x - CX;
      const c = ringColorAt(dx, dy);
      if (c && !inCircle(x, y, radius)) {
        setPixel(x, y, c);
      }
    }
  }
  
  // 2. Планета (с полосами и вращением)
  for (let y = 0; y < SPRITE_H; y++) {
    for (let x = 0; x < SPRITE_W; x++) {
      if (!inCircle(x, y, radius)) continue;
      const d = Math.sqrt((x-CX)**2 + (y-CY)**2);
      const b = brightness(x, y, radius);
      const uv = sphereUV(x, y, radius);
      if (!uv) { setPixel(x, y, P.m); continue; }
      
      // Полосы с лёгкой волной + вращение для текстуры
      const wave = Math.sin((uv.u + rotation) * Math.PI * 4 + uv.v * 4) * 0.01;
      const base = saturnBand(uv.v + wave);
      
      let c;
      if (d > radius - 0.7) c = b < 0.4 ? P.D : P.d;
      else if (b < 0.2) c = base === 'm' ? P.d : P.m;
      else if (b < 0.4) c = base === 'L' ? P.l : P[base];
      else if (b > 0.8) c = base === 'm' ? P.l : P.L;
      else c = P[base];
      setPixel(x, y, c);
    }
  }
  
  // 3. Передняя половина колец
  for (let y = 0; y < SPRITE_H; y++) {
    for (let x = 0; x < SPRITE_W; x++) {
      const dy = y - CY;
      if (dy < 0) continue;
      const dx = x - CX;
      const c = ringColorAt(dx, dy);
      if (c) setPixel(x, y, c);
    }
  }
}
```

---

## Уран (Uranus)

![Уран](sprites/uranus.png)

Бирюзовая планета с **наклонёнными на 45° кольцами** (в реальности ось наклонена на 98°, но 45° лучше читается визуально). Кольца повёрнуты диагонально.

**Радиус:** 9 px  
**Палитра:** Бирюзово-голубые тона + 2 оттенка для колец.  
**Особенности:** Кольца с поворотом на π/4 через матрицу cosT/sinT.

**Палитра:**

| Код | Hex | Назначение |
|-----|-----|------------|
| `L` | `#B8E8E0` | светлый тон |
| `l` | `#7CC4C0` | средне-светлый |
| `m` | `#3A8088` | средний |
| `d` | `#1A4A55` | тёмный |
| `K` | `#A0D4D0` | светлое кольцо |
| `k` | `#508088` | тёмное кольцо |
| `D` | `#1A1228` | обводка |

**Параметры анимации:**

- Размер на экране: `0.85` × базовый scale (рекомендация: scale=4)
- Скорость вращения: `8e-05` рад/мс (full rotation ≈ 12500 сек)

**JS-функция отрисовки:**

```javascript
function drawUranus(rotation, time) {
  const radius = 9;
  const P = PALETTES.uranus;
  
  // Кольца наклонены на 45° (между вертикальным и горизонтальным)
  const ringOuterA = 4, ringOuterB = 16;
  const ringInnerA = 2.7, ringInnerB = 12;
  
  // Угол наклона: 45° = π/4
  // Поворот координат: dx', dy' = поворот (dx, dy) на -45°
  const tiltAngle = Math.PI / 4;
  const cosT = Math.cos(tiltAngle);
  const sinT = Math.sin(tiltAngle);
  
  function ringColorAt(dx, dy) {
    // Поворачиваем координаты для проверки эллипса
    const dxr = dx * cosT + dy * sinT;
    const dyr = -dx * sinT + dy * cosT;
    
    if ((dxr/ringInnerA)**2 + (dyr/ringInnerB)**2 <= 1.0) return null;
    if ((dxr/ringOuterA)**2 + (dyr/ringOuterB)**2 > 1.0) return null;
    const rNorm = Math.abs(dxr)/ringOuterA * 0.5 + Math.abs(dyr)/ringOuterB;
    if (rNorm > 0.92) return P.k;
    if (rNorm > 0.75) return P.K;
    if (rNorm > 0.65) return P.k;
    return P.K;
  }
  
  // Передняя/задняя половина определяется по знаку dxr (повёрнутой координаты)
  function isFront(dx, dy) {
    const dxr = dx * cosT + dy * sinT;
    return dxr >= 0;
  }
  
  // 1. Задние кольца
  for (let y = 0; y < SPRITE_H; y++) {
    for (let x = 0; x < SPRITE_W; x++) {
      const dx = x - CX, dy = y - CY;
      if (isFront(dx, dy)) continue;
      const c = ringColorAt(dx, dy);
      if (c && !inCircle(x, y, radius)) setPixel(x, y, c);
    }
  }
  
  // 2. Планета
  for (let y = 0; y < SPRITE_H; y++) {
    for (let x = 0; x < SPRITE_W; x++) {
      if (!inCircle(x, y, radius)) continue;
      const d = Math.sqrt((x-CX)**2 + (y-CY)**2);
      const b = brightness(x, y, radius);
      
      // Слабая текстура полос (под наклоном из-за оси)
      const uv = sphereUV(x, y, radius);
      let tex = 0;
      if (uv) {
        tex = Math.sin((uv.u + rotation) * Math.PI * 6) * 0.05;
      }
      
      let c;
      const bAdj = b + tex;
      if (d > radius - 0.7) c = bAdj < 0.4 ? P.D : P.d;
      else if (bAdj > 0.7) c = P.L;
      else if (bAdj > 0.45) c = P.l;
      else if (bAdj > 0.2) c = P.m;
      else c = P.d;
      setPixel(x, y, c);
    }
  }
  
  // 3. Передние кольца
  for (let y = 0; y < SPRITE_H; y++) {
    for (let x = 0; x < SPRITE_W; x++) {
      const dx = x - CX, dy = y - CY;
      if (!isFront(dx, dy)) continue;
      const c = ringColorAt(dx, dy);
      if (c) setPixel(x, y, c);
    }
  }
}
```

---

## Нептун (Neptune)

![Нептун](sprites/neptune.png)

Глубокий синий газовый гигант. Тонкие горизонтальные полосы по широте + тёмные шторма (Большое Тёмное Пятно) + светлые облачные штрихи, движущиеся **быстрее планеты**.

**Радиус:** 10 px  
**Палитра:** От светло-голубого до глубокого синего, 4 оттенка.  
**Особенности:** Полосы sin(v*PI*6)*0.06 + шум штормов (scale=2.5) + быстрые облака (cloudDrift = t*0.025).

**Палитра:**

| Код | Hex | Назначение |
|-----|-----|------------|
| `L` | `#7CA4E8` | светлый тон |
| `l` | `#3C68B8` | средне-светлый |
| `m` | `#1A3A88` | средний |
| `d` | `#0C1F58` | тёмный |
| `D` | `#1A1228` | обводка |

**Параметры анимации:**

- Размер на экране: `0.85` × базовый scale (рекомендация: scale=4)
- Скорость вращения: `0.00012` рад/мс (full rotation ≈ 8333 сек)

**JS-функция отрисовки:**

```javascript
function drawNeptune(rotation, time) {
  const radius = 10;
  const P = PALETTES.neptune;
  const t = time * 0.001;
  // Облака движутся быстрее планеты — характерная черта Нептуна
  const cloudDrift = t * 0.025;
  
  for (let y = 0; y < SPRITE_H; y++) {
    for (let x = 0; x < SPRITE_W; x++) {
      if (!inCircle(x, y, radius)) continue;
      const d = Math.sqrt((x-CX)**2 + (y-CY)**2);
      const b = brightness(x, y, radius);
      const uv = sphereUV(x, y, radius);
      if (!uv) continue;
      
      const uRot = (uv.u + rotation) % 1.0;
      const uCloud = (uv.u + rotation + cloudDrift) % 1.0;
      
      // Полосы по широте (Нептун — газовый гигант)
      const bandMod = Math.sin(uv.v * Math.PI * 6) * 0.06;
      // Тёмные шторма
      const noiseStorms = fractalNoise(uRot, uv.v, 2.5, 3);
      // Светлые облачные штрихи (движутся быстрее)
      const noiseClouds = fractalNoise(uCloud * 1.5 + 0.3, uv.v + 0.1, 4, 2);
      
      const bAdj = b + bandMod;
      
      let base;
      if (d > radius - 0.7) base = b < 0.4 ? P.D : P.d;
      else if (bAdj > 0.7) base = P.L;
      else if (bAdj > 0.45) base = P.l;
      else if (bAdj > 0.2) base = P.m;
      else if (bAdj > 0.05) base = P.d;
      else base = P.D;
      
      // Тёмные шторма
      if (noiseStorms < -0.4) {
        if (base === P.L) base = P.l;
        else if (base === P.l) base = P.m;
        else if (base === P.m) base = P.d;
      } else if (noiseClouds > 0.5 && b > 0.4) {
        // Светлые облачные штрихи
        if (base === P.m) base = P.l;
        else if (base === P.l) base = P.L;
        else if (base === P.dk) base = P.m;
      }
      
      setPixel(x, y, base);
    }
  }
}
```

---

## Ракета (Rocket)

![Rocket](sprites/rocket.png)

Статичный спрайт 14×19. Белый корпус с серыми тенями, синий иллюминатор, красные плавники и носовой конус. Пульсирующий огонь из сопел (4 слоя: красный→оранжевый→жёлтый→ярко-жёлтое ядро).

**Размер спрайта:** 14×19 px

**Палитра:**

| Код | Hex | Назначение |
|-----|-----|------------|
| `D` | `#1A2A3E` | обводка |
| `W` | `#D8D0C0` | корпус белый |
| `G` | `#A5A39A` | тень корпуса |
| `g` | `#6E6E72` | тёмная тень |
| `B` | `#3F6580` | иллюминатор средний |
| `b` | `#1F3A52` | иллюминатор тёмный |
| `L` | `#8FB8D0` | блик иллюминатора |
| `R` | `#E1611A` | красный (носовой конус, плавники) |
| `r` | `#B84C1C` | тёмно-красный |

**Пиксельная карта (sprite):**

```
......DD......
.....DRRD.....
.....DRRD.....
....DRrrRD....
....DWGGWD....
...DWWWWWWD...
...DWWWWWGD...
...DWWbbWGD...
...DWbLBBGD...
...DWbBBBGD...
...DWWbbWGD...
...DWWWWWGD...
...DWWWWWGD...
..DRWWWWWGRD..
.DRrWWWWWGrRD.
.DRrDWWWGDrRD.
.DrDDGGGgDDrD.
..D..DggD..D..
.....DDDD.....
```

Точка = прозрачный пиксель. Каждая буква = код цвета из палитры.

**JS-функция отрисовки** (с пульсирующим огнём):

```javascript
function drawRocket(cx, cy, scale, bob) {
  for (let y = 0; y < ROCKET_H; y++) {
    for (let x = 0; x < ROCKET_W; x++) {
      const ch = ROCKET_SPRITE[y][x];
      const color = ROCKET_PALETTE[ch];
      if (!color) continue;
      ctx.fillStyle = `rgb(${color[0]},${color[1]},${color[2]})`;
      ctx.fillRect(
        Math.floor(cx + (x - ROCKET_W/2) * scale),
        Math.floor(cy + (y - ROCKET_H/2) * scale + bob),
        scale, scale
      );
    }
  }
  // Огонь снизу — пульсирующий
  const t = performance.now() * 0.005;
  const flicker = Math.sin(t) * 0.5 + 0.5;
  const fy = cy + (ROCKET_H/2) * scale + bob;
  const fx = cx;
  // Слои огня
  const flame_red = `rgb(184, 76, 28)`;
  const flame_orange = `rgb(225, 97, 26)`;
  const flame_yellow = `rgb(248, 160, 48)`;
  
  ctx.fillStyle = flame_red;
  ctx.fillRect(Math.floor(fx - 2*scale), Math.floor(fy), 4*scale, scale*2);
  ctx.fillRect(Math.floor(fx - scale), Math.floor(fy + 2*scale), 2*scale, scale*2);
  
  ctx.fillStyle = flame_orange;
  ctx.fillRect(Math.floor(fx - scale), Math.floor(fy), 2*scale, scale*2);
  ctx.fillRect(Math.floor(fx - 0.5*scale), Math.floor(fy + 2*scale), scale, scale * (1 + flicker));
  
  ctx.fillStyle = flame_yellow;
  ctx.fillRect(Math.floor(fx - 0.5*scale), Math.floor(fy), scale, scale * 1.5);
}
```

---

## Звёздное поле (фон)

Бонус для космического фона. Два типа звёзд: обычные точки (синевато-белые) и крестики (золотистые, более редкие).

**Цвета:**
- Обычная звезда: `rgba(200, 220, 255, α)` где α пульсирует
- Звезда-крестик: `rgba(250, 205, 96, α)`

**Рекомендуемая плотность:** ~240 звёзд на canvas 960×560 (примерно одна звезда на 2200 px²)

```javascript
// Инициализация
const stars = [];
for (let i = 0; i < 240; i++) {
  stars.push({
    x: Math.random() * canvas.width,
    y: Math.random() * canvas.height,
    size: Math.random() < 0.6 ? 1 : (Math.random() < 0.88 ? 2 : 3),
    twinkle: Math.random() * Math.PI * 2,
    twinkleSpeed: 0.01 + Math.random() * 0.04,
    type: Math.random() < 0.93 ? 'dot' : 'cross',
  });
}

// В цикле рендера
stars.forEach(s => {
  s.twinkle += s.twinkleSpeed;
  const a = 0.3 + 0.5 * Math.sin(s.twinkle);
  if (s.type === 'cross') {
    ctx.fillStyle = `rgba(250, 205, 96, ${a * 0.85})`;
    ctx.fillRect(s.x - 1, s.y, 3, 1);
    ctx.fillRect(s.x, s.y - 1, 1, 3);
  } else {
    ctx.fillStyle = `rgba(200, 220, 255, ${a})`;
    ctx.fillRect(s.x, s.y, s.size, s.size);
  }
});
```

---

## Структура файлов в этом пакете

```
space-art-pack/
├── README.md                  ← этот документ
├── planets_preview.html       ← интерактивный превью всех объектов
└── sprites/
    ├── all_overview.png       ← общий обзор всех 10 спрайтов
    ├── sun.png
    ├── mercury.png
    ├── venus.png
    ├── earth.png
    ├── mars.png
    ├── jupiter.png
    ├── saturn.png
    ├── uranus.png
    ├── neptune.png
    └── rocket.png
```

Каждый PNG в папке `sprites/` — один кадр объекта, отрендеренный с rotation=0.1..0.9 для разнообразия. Реальные кадры в игре генерируются процедурно — PNG нужны для предпросмотра и спецификации.

---

## Как использовать в игре

**Подход 1 — Процедурно (рекомендуется):**

Скопируй утилитарные функции и нужные `drawXxx` в свой JS-код. В каждом кадре вызывай `drawXxx(rotation, performance.now())`. Это даёт живое анимированное небо с малыми затратами по памяти.

**Подход 2 — Спрайты:**

Используй готовые PNG из `sprites/` как статичные изображения. Для анимации можно генерировать 8-16 кадров и склеить в спрайт-лист. Минус — больше памяти и фиксированная скорость вращения.

**Технические замечания:**

- Все спрайты используют `image-rendering: pixelated` (CSS) или `imageSmoothingEnabled = false` (canvas), чтобы при масштабировании сохранялся пиксельный вид.
- При рендере на canvas рекомендуется использовать off-screen `ImageData` для попиксельной отрисовки, потом `ctx.drawImage` с масштабом для финального вывода. Это значительно быстрее, чем `ctx.fillRect` для каждого пикселя на основном canvas.
- Скорость вращения подобрана так, что движение видно, но не отвлекает. Сатурн вращается чуть быстрее остальных как акцент.

---

*Art pack version 1.0 — собран в ходе совместной разработки.*