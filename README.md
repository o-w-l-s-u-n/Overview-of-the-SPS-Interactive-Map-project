# SPS Map: интерактивная карта индустриальных проектов

<img src="./spsmapgif.gif" alt="map gif"  />

> Дашборд для «СПС» собирает на одном экране карту строительства, актуальные новости и витрину специалистов. Всё написано на Next.js 14 и TypeScript, а данные в реальном времени подкачиваются из корпоративных API.

## Архитектура и стек

- фронтенд на **Next.js 14 (app router)**, **React 18** и **Tailwind CSS**;
- **react-simple-maps** отвечает за ортографическую карту с плавным зумом, **@pbe/react-yandex-maps** — за детальный вид выбранного объекта;
- **SWR** периодически опрашивает API с профи и новостями, обеспечивая автообновление без полной перезагрузки;
- **framer-motion** и **swiper** дают анимацию и автопрокрутку, а кастомные CSS-переходы сглаживают смену масштаба и градиенты фона.

## Фокус на объекте: автонавигация по маркерам

Карта сама подсвечивает интересные проекты, объединяя автоплей и ручное управление. При клике или автоцикле камера масштабируется к координатам, а плавность контролируется собственным debounce.

```tsx
// src/components/map/Map.tsx
const handleMarkerClick = (project: objectProject, isAuto?: boolean) => {
  const autoFunc = () => {
    if (!project?.geo) return;
    setPosition({ coordinates: [project.geo[1], project.geo[0]], zoom: 0.8 });
    setSmoothZoom(true);
    setTimeout(() => {
      handleZoomIn();
    }, 1000);
  };

  selectProject(project);
  if (project?.geo && !isAuto) {
    setPosition({ coordinates: [project.geo[1], project.geo[0]], zoom: 2 });
  } else {
    autoFunc();
  }
  setSmoothZoom(true);
  setTimeout(() => setSmoothZoom(false), 500);
};
```

Раз в **23 секунды** `useEffect` переключает активный проект, перемешивая массив, чтобы выдача не выглядела статичной. Пользователь кликнул по маркеру — автоплей ставится на паузу и возобновляется спустя минуту (`setTimeout(() => setAutoplay(true), 60000)`), поэтому оператор может спокойно изучить карточку.

## Оркестрация состояния

Вся карта и меню работают поверх лёгкого контекста, который держит только ссылку на активный проект. Это позволяет синхронно открывать карточку, подсвечивать маркер и переключать галерею без prop drilling.

```tsx
// src/components/projectMenu/ProjectMenuContext.tsx
export function ProjectMenuProvider({ children }: { children: React.ReactNode }) {
  const [activeProject, setActiveProject] = useState<objectProject | null>(null);

  const selectProject = (project: objectProject | null) => {
    setActiveProject(project);
  };

  return (
    <ProjectMenuContext.Provider value={{ activeProject, selectProject }}>
      {children}
    </ProjectMenuContext.Provider>
  );
}
```

`ProjectMenu`, `Map` и `ScrollProfi` слушают один и тот же контекст: выбор карточки профи или новости автоматически подсвечивает объект и запускает галерею.

## Точные регионы без тяжёлых геоданных

Чтобы подсветка региона не рвала производительность, мы упрощаем исходный **GeoJSON** через `simplify-geojson`, а принадлежность объекта считаем прямо в браузере через `point-in-polygon`. Это позволяет покрашивать область только для выбранного проекта, не перегружая состояние.

```tsx
// src/components/map/Map.tsx
const polygons = useMemo(
  () => (
    <Geographies geography={simplifiedGeoJSON}>
      {({ geographies }) =>
        geographies.map((geo) => {
          let isActive = false;
          const coordinates = geo.geometry.coordinates.flat(2);

          if (activeProject?.geo && position.zoom !== 0.8) {
            isActive = pointInPolygon(
              [
                Math.floor(activeProject.geo[1]),
                Math.floor(activeProject.geo[0]),
              ],
              coordinates,
            );
          }

          return (
            <Geography
              key={geo.rsmKey}
              geography={geo}
              className={`${activeProject ? "opacity-70" : ""} transition-all duration-500`}
              style={{ /* динамическая заливка и stroke */ }}
            />
          );
        })
      }
    </Geographies>
  ),
  [activeProject, simplifiedGeoJSON, position],
);
```

Добавочный бонус — карта остаётся отзывчивой даже на медиастендах, где процессор и видеоядро ограничены.

## Живые данные из корпоративных API

Панель специалистов и новостей обновляется автоматически: SWR периодически опрашивает API, а `dedupingInterval` защищает от лишних запросов. Рендер идёт в клиентском компоненте, поэтому данные оживают без SSR‑задержек.

```tsx
// src/app/page.tsx
const { data: profiData, isLoading: profiLoading } = useSWR(
  "https://apiUrl",
  fetcher,
  {
    refreshInterval: 3_600_000, // 1 час
    dedupingInterval: 3_600_000,
  },
);

const { data: newsData, isLoading: newsLoading } = useSWR(
  "https://lk.sps38.pro/api/news-list?limit=999&page=0&order=DESC&sort=create",
  fetcher,
  {
    refreshInterval: 600_000, // 10 минут
    dedupingInterval: 600_000,
  },
);
```

`ScrollProfi` делит массив специалистов на пять потоков и раздаёт их пяти независимым `Swiper`: первая полоса стартует с задержкой, последующие подхватывают её с интервалом, создавая эффект «живой» ленты слайдов.

## Двусторонние карточки, чтобы экономить пространство

Три верхних карточки — «Проект», «Срок», «Сумма» — сделаны двусторонними. Внутри `InfoTopCard` используется **framer-motion** с `useMotionValue`, чтобы отслеживать угол поворота и автоматом переключать содержимое при превышении 90°. Каждая карточка вращается по собственной оси и показывает вторую грань (например, заказчика
или отрасль), сохраняя компактную высоту в **150 px**.

```tsx
// src/components/projectMenu/InfoTopCard.tsx
<motion.div
  initial={{
    opacity: 0,
    y: 40,
    [`rotate${yRotation ? "Y" : "X"}`]: -45,
  }}
  animate={{
    opacity: 1,
    y: 0,
    [`rotate${yRotation ? "Y" : "X"}`]: isFlipped ? 180 : 0,
  }}
  transition={{
    type: "spring",
    mass: cardIndex === 0 ? 7 : 3,
    damping: 30,
    stiffness: 100,
  }}
>
  <Card className="h-full backdrop-blur-3xl border-none p-0">
    <CardHeader>
      <CardTitle className={isContentFlipped ? "text-stone-950" : "text-stone-500"}>
        {isContentFlipped ? title2 : title}
      </CardTitle>
      {/* переключение иконок в зависимости от стороны */}
    </CardHeader>
    <CardContent>
      <div className={`text-2xl font-bold ${isContentFlipped ? (color2 ?? "text-amber-700") : "text-red-900"}`}>
        {isContentFlipped ? value2 : value}
      </div>
      <p className={isContentFlipped ? "text-stone-800" : "text-stone-500"}>
        {isContentFlipped ? description2 : description}
      </p>
    </CardContent>
  </Card>
</motion.div>
```

Чтобы поверхность не «дергалась», при маунте карточка измеряет свою высоту, а общий контейнер задаёт `perspective: 800`, создавая ощущение объёмного вращения. Карточки flip-ятся каскадом: `delay = 1500 * (totalCards - cardIndex)` — зритель видит красивую волну вместо одновременного переключения.

## Галерея и мини‑карта: анимация без лишних кликов

`GalleryCarousel` работает на `embla-carousel-autoplay`: фотографии проектов сменяются каждые **2 секунды**, при наведении autoplay останавливается. Поверх изображения добавлен glare‑эффект, который даёт ощущение «глянцевых» карточек.

Рядом располагается мини‑карта на базе `@pbe/react-yandex-maps`; она получает `Placemark` с теми же координатами, что и основной маркер. **framer-motion** анимирует появление блока снизу, чтобы переход к деталям был плавным.

## UX для медиастендов и витрин

- На старте страница рассчитывает текущую дату и время и обновляет их каждую секунду — инфобар внизу экрана полностью автономен.
- Через **6 часов** срабатывает `window.location.reload()` — операторские панели автоматически подтягивают свежую версию приложения.
- Кнопки зума и панель выбора проектов появляются только когда **нет активного проекта** — интерфейс не перегружает внимание в режиме автопоказа.
- Tailwind‑конфигурация собирает контрастный футер с паттерном (`linear-gradient`) без внешних изображений, что упрощает брендинг и делает адаптивный фон.

## Что в итоге

- Один экран сочетает макроуровень (ортографическая карта) и микроуровень (Yandex‑вид, галерея, контент).
- Двусторонние карточки экономят место, а **framer-motion** делает flip‑анимацию многослойной и «живой».
- Кеширование SWR и упрощённый GeoJSON позволяют держать FPS даже на медленных устройствах.
- Архитектура компонент (`Map`, `ProjectMenu`, `ScrollProfi`, `NewsList`) переиспользует контексты и хуки и легко масштабируется на новые источники данных.
- Автообновления, задержки автоплея и state‑machine на таймерах гарантируют, что витрина останется интерактивной, даже если ей никто не управляет.
