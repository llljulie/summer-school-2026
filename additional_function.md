# Пробовала cursor чуть чуть, остальное deepseek
# В основном старалась разобраться с анализом, т к это направление новое для меня и более привлекательное



# План реализации отображения изображений
# Фича, про которую говорили на лекции (возможная реализация есть в этом файле, просто то, что сгенерила нейронка)

## 1. Симптом (проблема)

В приложении «Вертикаль» в настоящее время **отсутствует визуальный контент**, который мог бы:

- Повысить привлекательность карточек тренировок
- Улучшить узнаваемость инструкторов
- Визуально выделить зоны скалодрома (болдеринг vs трассы)
- Создать более привлекательный пользовательский опыт
- Дать клиентам представление о скалодроме и тренировках до записи

**Текущее состояние:** приложение использует только текстовую информацию и иконки.

---

## 2. Цель

Реализовать **систему отображения изображений** в клиентском приложении, которая позволит:

1. **Отображать изображения в карточках слотов** — фото зон/форматов тренировок
2. **Показывать аватарки инструкторов** — для узнаваемости в расписании и оценках
3. **Использовать изображения для маркетинга** — привлекательные фото скалодрома
4. **Обеспечить быструю загрузку** — кэширование и оптимизация

---

## 3. Требования

### 3.1. Функциональные требования

| ID | Требование | Приоритет |
|----|------------|-----------|
| IMG-01 | В карточке слота отображается **изображение зоны/формата** тренировки | P2 |
| IMG-02 | В карточке слота отображается **фото инструктора** (аватар) | P2 |
| IMG-03 | На экране деталей слота отображается **крупное изображение** зоны | P2 |
| IMG-04 | В профиле инструктора (при оценке) отображается его фото | P3 |
| IMG-05 | На экране успешной записи отображается **привлекательное изображение** | P3 |
| IMG-06 | В экране профиля клиента отображается **аватар пользователя** | P3 |
| IMG-07 | Изображения загружаются **асинхронно** без блокировки UI | P1 |
| IMG-08 | Изображения **кэшируются** для экономии трафика и скорости | P1 |
| IMG-09 | При недоступности изображения показывается **заглушка (placeholder)** | P1 |
| IMG-10 | Поддерживаются изображения в форматах JPEG, PNG, WebP | P2 |
| IMG-11 | Изображения **оптимизированы** под мобильные устройства | P2 |

### 3.2. Нефункциональные требования

| ID | Требование | Метрика |
|----|------------|---------|
| IMG-N1 | Время загрузки изображения (из кэша) | < 100 мс |
| IMG-N2 | Время загрузки изображения (из сети) | < 2 секунд |
| IMG-N3 | Размер кэша изображений | ≤ 50 МБ |
| IMG-N4 | Поддержка Retina-дисплеев | Да |
| IMG-N5 | Экономия трафика (WebP, сжатие) | Да |

---

## 4. Возможная реализация

### 4.1. Общая архитектура

┌─────────────────────────────────────────────────────────────────┐
│ Presentation Layer │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ SlotCard (Compose) │ InstructorAvatar │ │
│ │ SlotDetailScreen │ ProfileAvatar │ │
│ └─────────────────────────────────────────────────────────┘ │
└──────────────────────────┬──────────────────────────────────────┘
│ (Image Request)
┌──────────────────────────▼──────────────────────────────────────┐
│ Image Loading Layer │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ ImageLoader (Coil / Glide) │ │
│ │ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ │ │
│ │ │ Cache │ │ Transformer│ │ Decoder │ │ │
│ │ └─────────────┘ └─────────────┘ └─────────────┘ │ │
│ └─────────────────────────────────────────────────────────┘ │
└──────────────────────────┬──────────────────────────────────────┘
│ (URL)
┌──────────────────────────▼──────────────────────────────────────┐
│ Data Layer / API │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ GET /images/{type}/{id}.{ext} │ │
│ │ GET /instructors/{id}/avatar │ │
│ │ GET /zones/{id}/image │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘

text

### 4.2. Выбор библиотеки

**Рекомендация: Coil (Kotlin Image Loader)**

| Критерий | Coil | Glide | Picasso |
|----------|------|-------|---------|
| **Поддержка Compose** | ✅ Нативная | ⚠️ Через доп. библиотеку | ❌ |
| **Kotlin-first** | ✅ | ❌ | ❌ |
| **Размер библиотеки** | ~1.5 MB | ~3 MB | ~1 MB |
| **Кэширование** | ✅ | ✅ | ✅ |
| **WebP поддержка** | ✅ | ✅ | ⚠️ |
| **Актуальность** | ✅ Активна | ✅ Активна | ⚠️ Устаревает |

**Вывод:** Coil — лучший выбор для Jetpack Compose проекта.

### 4.3. Настройка библиотеки

**build.gradle.kts:**
```kotlin
dependencies {
    // Coil — основной
    implementation("io.coil-kt:coil-compose:2.5.0")
    
    // Coil — расширения для GIF (опционально)
    implementation("io.coil-kt:coil-gif:2.5.0")
    
    // Coil — SVG поддержка (опционально)
    implementation("io.coil-kt:coil-svg:2.5.0")
}
Настройка ImageLoader (Application):

kotlin
class VerticalApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        
        // Настройка глобального ImageLoader
        Coil.setImageLoader(
            ImageLoader.Builder(this)
                .diskCache(
                    DiskCache.Builder()
                        .directory(cacheDir.resolve("image_cache"))
                        .maxSizeBytes(50 * 1024 * 1024) // 50 MB
                        .build()
                )
                .memoryCache(
                    MemoryCache.Builder()
                        .maxSizePercent(0.25) // 25% от доступной памяти
                        .build()
                )
                .crossfade(true)
                .respectCacheHeaders(false) // Кастомное кэширование
                .build()
        )
    }
}
4.4. Типы изображений и URL-схема
Структура URL:

text
https://cdn.vertical-climbing.ru/images/{type}/{id}.{ext}?w={width}&h={height}&fit={fit}

Типы изображений:
- zones/{zone_id}.webp  — изображение зоны
- formats/{format_id}.webp — изображение формата тренировки
- instructors/{instructor_id}.jpg — фото инструктора
- gyms/{gym_id}.webp — общее фото скалодрома
- clients/{client_id}.jpg — аватар клиента
- promo/success.webp — промо-изображение для экрана успеха
API-расширение (для бэкенда):

kotlin
// Добавляем поля в существующие DTO
data class Instructor(
    val id: String,
    val full_name: String,
    val is_active: Boolean,
    val avatar_url: String? // URL аватара
)

data class TrainingSlot(
    val id: String,
    // ... существующие поля
    val zone_image_url: String?, // URL изображения зоны
    val format_image_url: String? // URL изображения формата
)

data class TrainingZone(
    val id: String,
    val code: String,
    val title: String,
    val image_url: String? // URL изображения зоны
)

data class TrainingFormat(
    val id: String,
    val code: String,
    val title: String,
    val max_capacity: Int,
    val duration_minutes: Int,
    val image_url: String? // URL изображения формата
)
4.5. Реализация UI-компонентов
Базовый компонент изображения:

kotlin
@Composable
fun VerticalImage(
    url: String?,
    contentDescription: String?,
    modifier: Modifier = Modifier,
    placeholderRes: Int = R.drawable.ic_placeholder_image,
    errorRes: Int = R.drawable.ic_error_image,
    contentScale: ContentScale = ContentScale.Crop,
    loading: @Composable () -> Unit = {
        Box(
            modifier = Modifier
                .fillMaxSize()
                .background(Color.LightGray),
            contentAlignment = Alignment.Center
        ) {
            CircularProgressIndicator(
                modifier = Modifier.size(32.dp),
                color = VerticalTheme.Primary
            )
        }
    }
) {
    if (url.isNullOrEmpty()) {
        // Заглушка если URL нет
        Image(
            painter = painterResource(id = placeholderRes),
            contentDescription = contentDescription,
            modifier = modifier,
            contentScale = contentScale
        )
    } else {
        AsyncImage(
            model = ImageRequest.Builder(LocalContext.current)
                .data(url)
                .crossfade(true)
                .placeholder(placeholderRes)
                .error(errorRes)
                .build(),
            contentDescription = contentDescription,
            modifier = modifier,
            contentScale = contentScale,
            loading = loading
        )
    }
}
Карточка слота с изображением:

kotlin
@Composable
fun SlotCard(
    slot: TrainingSlot,
    onBookClick: () -> Unit,
    modifier: Modifier = Modifier
) {
    Card(
        modifier = modifier.fillMaxWidth(),
        shape = RoundedCornerShape(12.dp),
        elevation = CardDefaults.cardElevation(defaultElevation = 4.dp)
    ) {
        Column {
            // Изображение зоны/формата
            VerticalImage(
                url = slot.zone_image_url ?: slot.format_image_url,
                contentDescription = "Зона ${slot.zone.title}",
                modifier = Modifier
                    .fillMaxWidth()
                    .height(140.dp),
                contentScale = ContentScale.Crop
            )
            
            // Информация о слоте
            Column(
                modifier = Modifier.padding(16.dp)
            ) {
                // ... существующее содержимое карточки
                Row(
                    verticalAlignment = Alignment.CenterVertically,
                    horizontalArrangement = Arrangement.SpaceBetween,
                    modifier = Modifier.fillMaxWidth()
                ) {
                    // Инструктор с аватаром
                    InstructorChip(
                        instructor = slot.instructor,
                        modifier = Modifier.weight(1f)
                    )
                    
                    // Кнопка записи
                    VerticalButton(
                        text = "Записаться",
                        onClick = onBookClick,
                        enabled = slot.free_places > 0 && slot.slot_status == "open"
                    )
                }
            }
        }
    }
}
Аватар инструктора:

kotlin
@Composable
fun InstructorAvatar(
    instructor: Instructor,
    size: Dp = 40.dp,
    modifier: Modifier = Modifier
) {
    Box(
        modifier = modifier.size(size)
    ) {
        VerticalImage(
            url = instructor.avatar_url,
            contentDescription = "Фото ${instructor.full_name}",
            modifier = Modifier
                .fillMaxSize()
                .clip(CircleShape)
                .border(
                    width = 2.dp,
                    color = VerticalTheme.Primary,
                    shape = CircleShape
                ),
            placeholderRes = R.drawable.ic_instructor_placeholder,
            contentScale = ContentScale.Crop
        )
        
        // Рейтинг на аватаре (опционально)
        if (instructor.stats?.ratings_count ?: 0 > 0) {
            Box(
                modifier = Modifier
                    .align(Alignment.BottomEnd)
                    .background(
                        color = VerticalTheme.Primary,
                        shape = RoundedCornerShape(4.dp)
                    )
                    .padding(horizontal = 4.dp, vertical = 2.dp)
            ) {
                Text(
                    text = "★ ${String.format("%.1f", instructor.stats?.avg_rating ?: 0.0)}",
                    style = MaterialTheme.typography.labelSmall,
                    color = Color.White
                )
            }
        }
    }
}
Экран деталей слота:

kotlin
@Composable
fun SlotDetailScreen(
    slot: TrainingSlot,
    onBack: () -> Unit,
    onBook: () -> Unit
) {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .verticalScroll(rememberScrollState())
    ) {
        // Большое изображение зоны
        VerticalImage(
            url = slot.zone_image_url,
            contentDescription = "Зона ${slot.zone.title}",
            modifier = Modifier
                .fillMaxWidth()
                .height(280.dp)
                .background(Color.Black),
            contentScale = ContentScale.Crop
        )
        
        // Затемнение для текста
        Box(
            modifier = Modifier
                .fillMaxWidth()
                .height(280.dp)
                .background(
                    Brush.verticalGradient(
                        colors = listOf(
                            Color.Transparent,
                            Color.Black.copy(alpha = 0.6f)
                        ),
                        startY = 180f,
                        endY = 280f
                    )
                )
                .padding(16.dp),
            contentAlignment = Alignment.BottomStart
        ) {
            Column {
                Text(
                    text = slot.format.title,
                    color = Color.White,
                    style = MaterialTheme.typography.headlineMedium
                )
                Text(
                    text = slot.zone.title,
                    color = Color.White.copy(alpha = 0.8f),
                    style = MaterialTheme.typography.bodyMedium
                )
            }
        }
        
        // ... остальное содержимое экрана деталей
    }
}
Экран успешной записи:

kotlin
@Composable
fun BookingSuccessScreen(
    booking: Booking,
    onBackToSchedule: () -> Unit,
    onGoToBookings: () -> Unit
) {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(24.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        // Промо-изображение
        VerticalImage(
            url = "https://cdn.vertical-climbing.ru/images/promo/success.webp",
            contentDescription = "Вы записаны!",
            modifier = Modifier
                .size(200.dp)
                .clip(CircleShape),
            placeholderRes = R.drawable.ic_success_placeholder
        )
        
        // ... остальное содержимое экрана успеха
    }
}
4.6. Оптимизация и кэширование
Конфигурация кэша:

kotlin
object ImageCacheConfig {
    const val DISK_CACHE_SIZE_MB = 50
    const val MEMORY_CACHE_PERCENT = 0.25
    const val MAX_IMAGE_WIDTH = 1024
    const val MAX_IMAGE_HEIGHT = 1024
    const val IMAGE_QUALITY = 85 // JPEG качество
}

class VerticalImageLoader(
    context: Context
) : ImageLoader(context) {
    override fun newBuilder(): Builder {
        return Builder(context)
            .diskCache(
                DiskCache.Builder()
                    .directory(File(context.cacheDir, "images"))
                    .maxSizeBytes(50 * 1024 * 1024)
                    .build()
            )
            .memoryCache(
                MemoryCache.Builder()
                    .maxSizePercent(0.25)
                    .build()
            )
            .bitmapPool(BitmapPool.Builder().maxSizePercent(0.25).build())
            .components {
                // Добавляем кастомные трансформации
                add(ImageTransformation { input ->
                    // Оптимизация размера изображения
                    ResizeTransformation(1024, 1024).transform(input)
                })
            }
    }
}
4.7. Обработка ошибок
kotlin
sealed class ImageLoadingState {
    object Loading : ImageLoadingState()
    data class Success(val bitmap: Bitmap) : ImageLoadingState()
    data class Error(val throwable: Throwable) : ImageLoadingState()
}

@Composable
fun SafeImage(
    url: String?,
    contentDescription: String?,
    modifier: Modifier = Modifier,
    fallbackImage: Int = R.drawable.ic_error_image
) {
    var state by remember(url) { mutableStateOf<ImageLoadingState>(ImageLoadingState.Loading) }
    
    LaunchedEffect(url) {
        if (url == null) {
            state = ImageLoadingState.Error(Throwable("URL is null"))
            return@LaunchedEffect
        }
        
        try {
            val loader = Coil.imageLoader(LocalContext.current)
            val request = ImageRequest.Builder(LocalContext.current)
                .data(url)
                .build()
            val result = loader.execute(request)
            if (result is SuccessResult) {
                state = ImageLoadingState.Success(result.drawable.toBitmap())
            } else {
                state = ImageLoadingState.Error(Throwable("Failed to load image"))
            }
        } catch (e: Exception) {
            state = ImageLoadingState.Error(e)
        }
    }
    
    when (state) {
        is ImageLoadingState.Loading -> {
            Box(
                modifier = modifier.background(Color.LightGray),
                contentAlignment = Alignment.Center
            ) {
                CircularProgressIndicator(modifier = Modifier.size(24.dp))
            }
        }
        is ImageLoadingState.Success -> {
            Image(
                bitmap = (state as ImageLoadingState.Success).bitmap.asImageBitmap(),
                contentDescription = contentDescription,
                modifier = modifier,
                contentScale = ContentScale.Crop
            )
        }
        is ImageLoadingState.Error -> {
            Image(
                painter = painterResource(id = fallbackImage),
                contentDescription = contentDescription,
                modifier = modifier,
                contentScale = ContentScale.Crop
            )
        }
    }
}
4.8. Плейсхолдеры и заглушки
kotlin
object ImagePlaceholders {
    // Иконки для заглушек
    val ZONE_BOULDERING = R.drawable.ic_zone_bouldering
    val ZONE_ROPE = R.drawable.ic_zone_rope
    val INSTRUCTOR = R.drawable.ic_instructor_placeholder
    val CLIENT = R.drawable.ic_client_placeholder
    val SLOT = R.drawable.ic_slot_placeholder
    val SUCCESS = R.drawable.ic_success_placeholder
    val ERROR = R.drawable.ic_error_image
    
    // Градиенты для фона заглушек
    val PLACEHOLDER_GRADIENT = listOf(
        Color(0xFFE0E0E0),
        Color(0xFFBDBDBD)
    )
}

5. API-контракт для изображений
Расширение OpenAPI:

yaml
paths:
  /images/{type}/{id}:
    get:
      summary: Получение изображения
      parameters:
        - name: type
          in: path
          required: true
          schema:
            type: string
            enum: [zone, format, instructor, client, promo]
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
        - name: width
          in: query
          schema:
            type: integer
            minimum: 64
            maximum: 2048
        - name: height
          in: query
          schema:
            type: integer
            minimum: 64
            maximum: 2048
        - name: fit
          in: query
          schema:
            type: string
            enum: [crop, contain, fill]
            default: crop
      responses:
        '200':
          description: Изображение
          content:
            image/webp: {}
            image/jpeg: {}
            image/png: {}
        '404':
          description: Изображение не найдено
DTO-расширение:

yaml
components:
  schemas:
    Instructor:
      properties:
        # ... существующие поля
        avatar_url:
          type: string
          format: uri
          description: URL аватара инструктора
    
    TrainingZone:
      properties:
        # ... существующие поля
        image_url:
          type: string
          format: uri
          description: URL изображения зоны
    
    TrainingFormat:
      properties:
        # ... существующие поля
        image_url:
          type: string
          format: uri
          description: URL изображения формата тренировки