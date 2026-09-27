# CS2 Lua Executor

Мощный внутриигровой Lua-экзекутор для **Counter-Strike 2**, построенный на базе перехвата **DirectX 11 (Kiero / MinHook)** и графического интерфейса **Dear ImGui**.

Позволяет писать, загружать, тестировать и исполнять Lua-скрипты в реальном времени с поддержкой прямого чтения/записи памяти процесса, отрисовки оверлеев поверх игры, математических преобразований координат (World-to-Screen) и автоматического поиска актуальных смещений (Pattern Scanning).

---

## 📑 Содержание
- [Особенности](#-особенности)
- [Горячие клавиши](#-горячие-клавиши)
- [Lua API Документация](#-lua-api-документация)
  - [Глобальные константы и смещения](#глобальные-константы-и-смещения)
  - [Работа с памятью](#работа-с-памятью)
  - [Игровые сущности и матрица](#игровые-сущности-и-матрица)
  - [Отрисовка (Batched Drawing)](#отрисовка-batched-drawing)
  - [Ввод и системные утилиты](#ввод-и-системные-утилиты)
- [Примеры скриптов](#-примеры-скриптов)
  - [1. Получение информации об игроке](#1-получение-информации-об-игроке)
  - [2. Базовый Box ESP](#2-базовый-box-esp)
- [Файловая система и конфигурация](#-файловая-система-и-конфигурация)
- [Сборка проекта](#-сборка-проекта)
- [Внедрение (Инжект)](#-внедрение-инжект)

---

## 🚀 Особенности

* **DirectX 11 Hook**: Перехват метода `IDXGISwapChain::Present` через Kiero / MinHook для бесшовного рендеринга поверх кадров игры.
* **Dear ImGui UI**: Кастомный фиолетово-тёмный интерфейс с удобным многострочным текстовым редактором кода, подсветкой ошибок и управлением конфигурациями.
* **Dynamic Pattern Scanner**: Встроенный сигнатурный сканер, находящий базовые адреса и смещения в `client.dll` на лету, с сохранением в локальный кэш.
* **Memory Cache System**: Высокопроизводительный кэш памяти (до 10 МБ) с автоматической инвалидацией при записи для оптимизации сотен вызовов чтения за кадр.
* **Batched Rendering Engine**: Очередь отрисовки геометрических примитивов (боксы, линии, текст, круги) прямо из скриптов Lua без блокировок основного потока.
* **CS2 Console Integration**: Перенаправление вывода `print()` не только в отдельную консоль отладки, но и во внутреннюю консоль разработчика CS2 через `tier0.dll:Msg`.
* **Config Manager**: Автоматическое сохранение, загрузка и удаление пользовательских скриптов из папки документов пользователя.

---

## ⌨️ Горячие клавиши

| Клавиша | Действие |
| :--- | :--- |
| **`HOME`** | Открыть / скрыть главное меню экзекутора и окно менеджера конфигураций |
| **`F5`** | Запустить / перезапустить выполнение текущего открытого Lua-скрипта (включает цикл рендеринга) |

---

## 📚 Lua API Документация

### Глобальные константы и смещения
После завершения сканирования сигнатур в глобальную область видимости Lua автоматически экспортируются следующие переменные:

| Переменная | Описание |
| :--- | :--- |
| `CLIENT_BASE` | Базовый адрес модуля `client.dll` в памяти |
| `DW_LOCALPLAYER` | Смещение до указателя локального игрока (`dwLocalPlayerPawn`) |
| `DW_LOCALPLAYERCTRL` | Смещение контроллера локального игрока (`dwLocalPlayerController`) |
| `DW_ENTITYLIST` | Смещение списка сущностей (`dwEntityList`) |
| `DW_VIEWMATRIX` | Смещение матрицы проекции (`dwViewMatrix`) |
| `OFFSET_HEALTH` | Смещение здоровья (`m_iHealth`) |
| `OFFSET_TEAM` | Смещение номера команды (`m_iTeamNum`) |
| `OFFSET_ORIGIN` | Смещение координат позиции (`m_vecOrigin`) |
| `OFFSET_VIEWOFFSET` | Смещение высоты глаз / обзора (`m_vecViewOffset`) |
| `OFFSET_PLAYERPAWN` | Смещение хэндла пешки игрока (`m_hPlayerPawn`) |
| `OFFSET_VELOCITY` | Смещение вектора скорости (`m_vecVelocity`) |
| `OFFSET_ARMOR` | Смещение брони (`m_ArmorValue`) |
| `OFFSET_FLAGS` | Смещение флагов состояния (`m_fFlags`) |
| `OFFSET_SCOPED` | Смещение состояния прицеливания (`m_bIsScoped`) |
| `OFFSET_SHOTSFIRED` | Смещение счётчика выстрелов (`m_iShotsFired`) |
| `OFFSET_EYEANGLES` | Смещение углов обзора (`m_angEyeAngles`) |
| `OFFSET_LIFESTATE` | Смещение состояния жизни (`m_lifeState`) |
| `OFFSET_SPOTTED` | Смещение флага обнаружения (`m_bSpotted`) |
| `OFFSET_FLASHDURATION`| Смещение таймера ослепления (`m_flFlashDuration`) |

---

### Работа с памятью

#### `ReadInt32(address)`
Читает 32-битное целое число (`int32_t`) по указанному адресу.
* **Аргументы:** `address` (number)
* **Возвращает:** целое число

#### `ReadFloat32(address)`
Читает число с плавающей точкой одинарной точности (`float`) по адресу.
* **Аргументы:** `address` (number)
* **Возвращает:** дробное число

#### `ReadBool(address)`
Читает булево значение (`bool`) по адресу.
* **Аргументы:** `address` (number)
* **Возвращает:** `true` или `false`

#### `ReadVector3(address)`
Читает вектор из 3-х чисел float (`Vec3: x, y, z`).
* **Аргументы:** `address` (number)
* **Возвращает:** `x, y, z` (3 числа)

#### `WriteInt32(address, value)`
Записывает 32-битное целое число в память и инвалидирует кэш.
* **Аргументы:** `address` (number), `value` (integer)
* **Возвращает:** `bool` (успешность записи)

#### `WriteFloat32(address, value)`
Записывает число `float` в память и инвалидирует кэш.
* **Аргументы:** `address` (number), `value` (number)
* **Возвращает:** `bool` (успешность записи)

---

### Игровые сущности и матрица

#### `GetLocalPlayer()`
Возвращает адрес текущей пешки локального игрока (`g_localPawn`).
* **Возвращает:** `uintptr_t` (number)

#### `GetLocalPlayerController()`
Возвращает адрес контроллера локального игрока.
* **Возвращает:** `uintptr_t` (number)

#### `GetEntityList()`
Возвращает базовый адрес списка сущностей (`dwEntityList`).
* **Возвращает:** `uintptr_t` (number)

#### `GetEntity(index)`
Возвращает адрес пешки сущности (`PlayerPawn`) по индексу из EntityList (1-64).
* **Аргументы:** `index` (integer)
* **Возвращает:** `uintptr_t` (number) или `0`, если сущность не найдена

#### `GetViewMatrix()`
Возвращает абсолютный адрес матрицы вида `dwViewMatrix`.
* **Возвращает:** `uintptr_t` (number)

#### `WorldToScreen(worldX, worldY, worldZ, viewMatrixAddr)`
Переводит 3D координаты игрового мира в экранные 2D пиксели.
* **Аргументы:** `worldX`, `worldY`, `worldZ` (number), `viewMatrixAddr` (number)
* **Возвращает:** `screenX, screenY, isVisible` (числа X, Y и булево `isVisible`)

#### `GetClientBase()`
Возвращает адрес загрузки модуля `client.dll`.
* **Возвращает:** `number`

#### `OffsetsReady()`
Проверяет, завершилось ли сканирование/загрузка смещений и валидны ли они.
* **Возвращает:** `bool`

---

### Отрисовка (Batched Drawing)

Все функции отрисовки помещают примитивы в буфер, который отрисовывается на `ImGui::GetBackgroundDrawList()` во время вызова `hkPresent`.

#### `DrawBox(x, y, w, h, r, g, b, [a=255], [thickness=1.0])`
Рисует контурный прямоугольник.

#### `DrawFilledBox(x, y, w, h, r, g, b, [a=255])`
Рисует закрашенный прямоугольник.

#### `DrawLine(x1, y1, x2, y2, r, g, b, [a=255], [thickness=1.0])`
Рисует отрезок линии.

#### `DrawText(x, y, text, r, g, b, [a=255])`
Рисует текст шрифтом ImGui по экранным координатам.

#### `DrawCircle(x, y, radius, r, g, b, [a=255], [thickness=1.0])`
Рисует контур окружности.

#### `DrawFilledCircle(x, y, radius, r, g, b, [a=255])`
Рисует закрашенную окружность.

---

### Ввод и системные утилиты

* **`print(...)`**: Печатает переданные параметры в оверлей-консоль экзекутора и в CS2 Console (`tier0.dll`).
* **`GetScreenSize()`**: Возвращает `screenWidth, screenHeight` текущего монитора.
* **`GetDistance(x1, y1, z1, x2, y2, z2)`**: Вычисляет евклидово расстояние между двумя 3D точками.
* **`IsKeyPressed(vKey)`**: Возвращает `true`, если виртуальная клавиша нажата (через `GetAsyncKeyState`).
* **`GetCursorPos()`**: Возвращает позицию курсора мыши `x, y`.
* **`SetCursorPos(x, y)`**: Устанавливает положение курсора мыши на экране.
* **`GetTickCount()`**: Возвращает системное время `GetTickCount64()`.
* **`Sleep(ms)`**: Приостанавливает поток на указанное количество миллисекунд.

---

## 💡 Примеры скриптов

### 1. Получение информации об игроке
```lua
if not OffsetsReady() then
    print("[WAIT] Offsets are still scanning...")
    return
end

local localPawn = GetLocalPlayer()
if localPawn ~= 0 then
    local health = ReadInt32(localPawn + OFFSET_HEALTH)
    local armor = ReadInt32(localPawn + OFFSET_ARMOR)
    local team = ReadInt32(localPawn + OFFSET_TEAM)
    local x, y, z = ReadVector3(localPawn + OFFSET_ORIGIN)

    print(string.format("HP: %d | Armor: %d | Team: %d", health, armor, team))
    print(string.format("Pos: %.1f, %.1f, %.1f", x, y, z))
end
```

### 2. Базовый Box ESP
```lua
if not OffsetsReady() then return end

local localPawn = GetLocalPlayer()
if localPawn == 0 then return end

local localTeam = ReadInt32(localPawn + OFFSET_TEAM)
local vm = GetViewMatrix()

for i = 1, 64 do
    local entity = GetEntity(i)
    if entity ~= 0 and entity ~= localPawn then
        local health = ReadInt32(entity + OFFSET_HEALTH)
        local team = ReadInt32(entity + OFFSET_TEAM)

        -- Только живые враги
        if health > 0 and team ~= localTeam then
            local x, y, z = ReadVector3(entity + OFFSET_ORIGIN)
            local sx, sy, vis = WorldToScreen(x, y, z, vm)

            if vis then
                local headSx, headSy, headVis = WorldToScreen(x, y, z + 72.0, vm)
                if headVis then
                    local h = math.abs(sy - headSy)
                    local w = h / 2.0
                    local boxX = sx - (w / 2.0)
                    local boxY = headSy

                    -- Отрисовка рамки игрока
                    DrawBox(boxX, boxY, w, h, 255, 60, 60, 255, 1.5)
                    
                    -- Отрисовка здоровья
                    DrawText(boxX, boxY - 14, string.format("HP: %d", health), 255, 255, 255, 255)
                end
            end
        end
    end
end
```

---

## 📁 Файловая система и конфигурация

При инициализации библиотека автоматически создает рабочую директорию в профиле пользователя:
```text
C:\Users\<Имя_Пользователя>\Documents\luaexecutor\
├── cs2_offsets.ini         # Бинарный кэш сигнатур и оффсетов
└── configs\                # Пользовательские скрипты (*.lua)
    ├── default.lua
    └── ...
```

В окне **Configs (`HOME` -> Configs)** доступны:
* Создание и сохранение текущего буфера под новым именем.
* Загрузка скрипта в редактор (**Load**).
* Быстрый запуск выбранного скрипта (**Run**).
* Перезапись файла текущим содержимым (**Overwrite**).
* Удаление конфигурации (**Del**).
* Открытие директории конфигураций в проводнике Windows (**Open Folder**).

---

## 🛠 Сборка проекта

### Требования
* **Windows 10 / 11 (x64)**
* **Visual Studio 2022** (с установленным набором инструментов C++ Desktop Development v143+)
* **DirectX SDK** (или Windows SDK с D3D11)
* **Lua 5.4** (установленная через [vcpkg](https://github.com/microsoft/vcpkg): `vcpkg install lua:x64-windows`)

### Шаги сборки
1. Откройте решение `ImGui DirectX 11 Kiero Hook.sln` в Visual Studio.
2. Выберите конфигурацию **Release** и платформу **x64**.
3. Убедитесь, что библиотеки D3D11, DXGI и Lua слинкованы корректно.
4. Выполните сборку: `Build -> Build Solution` (`Ctrl + Shift + B`).
5. Собранная библиотека будет находиться в папке:
   ```text
   ImGui DirectX 11 Kiero Hook\x64\Release\luaexecutor.dll
   ```

---

## 💉 Внедрение (Инжект)

1. Запустите игру **Counter-Strike 2**.
2. Убедитесь, что `lua54.dll` доступна в системном каталоге (`System32`) или в одной папке с DLL (можно использовать батник `rsfsdf.bat`).
3. Используйте любой надежный инжектор динамических библиотек для архитектуры **x64** (например, Process Hacker, Xenos или ручной инжектор).
4. Заинжектите скомпилированную библиотеку в процесс `cs2.exe`.
5. При успешной инициализации откроется консоль с приветствием `[t.me/luaexecutorcs2] CS2 Lua Executor loaded!`.
6. Нажмите клавишу **`HOME`** в игре для открытия графического меню.
