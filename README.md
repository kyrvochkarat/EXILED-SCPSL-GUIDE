# 🔌 Полное руководство по разработке плагинов для EXILED 9.x

### Автор: [vityanvsk](https://github.com/vityanvsk) aka **kyrvochkarat**

---

> **Это руководство — от нуля до продвинутого уровня.**
> Если ты никогда не писал плагины для SCP:SL — добро пожаловать. Здесь разберём всё: от «что такое C#» до патчинга игровых методов через Harmony.

---

## 📑 Оглавление

1. [Введение и архитектура](#1--введение-и-архитектура)
2. [Подготовка окружения](#2--подготовка-окружения)
3. [Создание первого плагина](#3--создание-первого-плагина)
4. [Жизненный цикл плагина](#4--жизненный-цикл-плагина)
5. [Система событий (Events)](#5--система-событий-events)
6. [Работа с игроками (Player API)](#6--работа-с-игроками-player-api)
7. [Конфигурация плагина](#7--конфигурация-плагина)
8. [Команды (RemoteAdmin / Client Console / Server Console)](#8--команды)
9. [Coroutines и MEC (Unity-совместимые таймеры)](#9--coroutines-и-mec)
10. [Работа с Unity Engine из плагина](#10--работа-с-unity-engine-из-плагина)
11. [Mono Runtime — что нужно знать](#11--mono-runtime--что-нужно-знать)
12. [Harmony — патчинг игрового кода](#12--harmony--патчинг-игрового-кода)
13. [Работа с базой данных и файлами](#13--работа-с-базой-данных-и-файлами)
14. [Сетевые запросы (HTTP/WebSocket)](#14--сетевые-запросы)
15. [Лучшие практики и анти-паттерны](#15--лучшие-практики-и-анти-паттерны)
16. [Отладка и профилирование](#16--отладка-и-профилирование)
17. [Публикация плагина](#17--публикация-плагина)
18. [FAQ](#18--faq)
19. [Полезные ссылки](#19--полезные-ссылки)

---

## 1. 🏗 Введение и архитектура

### Как вообще устроен сервер SCP:SL?

```
┌─────────────────────────────────────────────────┐
│                  SCP:SL Server                  │
│  ┌───────────────────────────────────────────┐  │
│  │           Unity Engine (2022.x)           │  │
│  │  ┌─────────────┐  ┌───────────────────┐   │  │
│  │  │  Mono/.NET   │  │   Mirror (Netcode)│   │  │
│  │  └──────┬──────┘  └───────────────────┘   │  │
│  └─────────┼─────────────────────────────────┘  │
│            │                                     │
│  ┌─────────▼─────────────────────────────────┐  │
│  │        NorthwoodLib / Game Assembly        │  │
│  │   (PluginAPI, PlayerRoles, Items, etc.)    │  │
│  └─────────┬─────────────────────────────────┘  │
│            │                                     │
│  ┌─────────▼─────────────────────────────────┐  │
│  │              EXILED Framework              │  │
│  │  ┌────────┐ ┌────────┐ ┌──────────────┐   │  │
│  │  │Loader  │ │Events  │ │  Harmony      │   │  │
│  │  │        │ │System  │ │  Patches      │   │  │
│  │  └────────┘ └────────┘ └──────────────┘   │  │
│  └─────────┬─────────────────────────────────┘  │
│            │                                     │
│  ┌─────────▼──────┐                             │
│  │  Твой плагин   │  ← ты здесь                │
│  │  (MyPlugin.dll)│                             │
│  └────────────────┘                             │
└─────────────────────────────────────────────────┘
```

### Ключевые технологии

| Технология | Роль | Почему важно |
|---|---|---|
| **Unity Engine** | Игровой движок | Вся физика, объекты, `GameObject`, `Transform`, `MonoBehaviour` |
| **Mono Runtime** | Среда выполнения .NET | Сервер работает на Mono, а не на CoreCLR. Это влияет на доступные API |
| **Mirror** | Сетевой фреймворк | Синхронизация состояния между сервером и клиентами |
| **Harmony** | Библиотека патчинга | Позволяет перехватывать и модифицировать ЛЮБОЙ метод игры в runtime |
| **EXILED** | Фреймворк для плагинов | Обёртка над всем вышеперечисленным: события, API игроков, загрузчик |
| **MEC** | More Effective Coroutines | Замена стандартных Unity-корутин для серверной среды |

---

## 2. 🛠 Подготовка окружения

### Что нужно установить

#### 1. IDE (среда разработки)
- **[JetBrains Rider](https://www.jetbrains.com/rider/)** — лучший выбор (платный, но есть студенческая лицензия)
- **[Visual Studio 2022](https://visualstudio.microsoft.com/)** — бесплатная Community Edition
- **[VS Code](https://code.visualstudio.com/)** + C# Dev Kit — лёгкий вариант

#### 2. .NET SDK
```bash
# Установи .NET SDK 8.0 (для сборки)
# https://dotnet.microsoft.com/download/dotnet/8.0
dotnet --version
# Должно вывести 8.x.x
```

#### 3. Локальный сервер SCP:SL
Обязательно для тестирования. Установи через SteamCMD или из Steam:
```bash
# SteamCMD
steamcmd +force_install_dir ./scpsl-server +login anonymous +app_update 996560 validate +quit
```

#### 4. EXILED
```bash
# Linux
curl -sSL https://github.com/ExMod-Team/EXILED/releases/latest/download/Exiled.Installer-Linux -o Exiled.Installer
chmod +x Exiled.Installer
./Exiled.Installer

# Windows — скачай Exiled.Installer-Win.exe из Releases
```

### Структура папок EXILED

```
~/.config/EXILED/               # Linux
%APPDATA%/EXILED/               # Windows
├── Configs/
│   ├── {port}-config.yml       # Конфиг сервера
│   └── {port}-translations/    # Переводы плагинов
├── Plugins/
│   ├── MyPlugin.dll            # ← Сюда кладёшь свой плагин
│   └── dependencies/           # ← Зависимости плагинов
└── Logs/
```

---

## 3. 🚀 Создание первого плагина

### Шаг 1: Создание проекта

```bash
mkdir MyFirstPlugin
cd MyFirstPlugin
dotnet new classlib -n MyFirstPlugin -f net48
cd MyFirstPlugin
```

> ⚠️ **Почему `net48`?**
> Сервер SCP:SL работает на **Mono**, который совместим с .NET Framework 4.8. Это не .NET 6/7/8 — это важно помнить!

### Шаг 2: Подключение зависимостей

Есть два способа:

#### Способ А: NuGet (рекомендуется)

Отредактируй файл `.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net48</TargetFramework>
    <LangVersion>latest</LangVersion>
    <!-- Не копируем зависимости EXILED в выходную папку -->
    <CopyLocalLockFileAssemblies>false</CopyLocalLockFileAssemblies>
  </PropertyGroup>

  <ItemGroup>
    <!-- Основные пакеты EXILED -->
    <PackageReference Include="EXILED" Version="9.5.1" />
    
    <!-- Если NuGet-пакет EXILED недоступен нужной версии, 
         используй Способ Б -->
  </ItemGroup>

</Project>
```

#### Способ Б: Ручные ссылки на DLL

Скопируй DLL из папки установленного EXILED и сделай ссылки:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net48</TargetFramework>
    <LangVersion>latest</LangVersion>
  </PropertyGroup>

  <ItemGroup>
    <!-- Путь к папке с DLL EXILED (настрой под себя) -->
    <Reference Include="Exiled.API">
      <HintPath>..\refs\Exiled.API.dll</HintPath>
      <Private>false</Private>
    </Reference>
    <Reference Include="Exiled.Events">
      <HintPath>..\refs\Exiled.Events.dll</HintPath>
      <Private>false</Private>
    </Reference>
    <Reference Include="Exiled.Loader">
      <HintPath>..\refs\Exiled.Loader.dll</HintPath>
      <Private>false</Private>
    </Reference>
    
    <!-- Unity и игровые сборки -->
    <Reference Include="UnityEngine.CoreModule">
      <HintPath>..\refs\UnityEngine.CoreModule.dll</HintPath>
      <Private>false</Private>
    </Reference>
    <Reference Include="Assembly-CSharp">
      <HintPath>..\refs\Assembly-CSharp.dll</HintPath>
      <Private>false</Private>
    </Reference>
    <Reference Include="Mirror">
      <HintPath>..\refs\Mirror.dll</HintPath>
      <Private>false</Private>
    </Reference>
    <Reference Include="0Harmony">
      <HintPath>..\refs\0Harmony.dll</HintPath>
      <Private>false</Private>
    </Reference>
    
    <!-- MEC -->
    <Reference Include="Assembly-CSharp-firstpass">
      <HintPath>..\refs\Assembly-CSharp-firstpass.dll</HintPath>
      <Private>false</Private>
    </Reference>
  </ItemGroup>

</Project>
```

> 💡 **Где взять DLL?**
> Все нужные DLL лежат в папке:
> - Linux: `~/.config/EXILED/` и `/home/{user}/.config/SCP Secret Laboratory/`
> - Windows: `%APPDATA%/SCP Secret Laboratory/` и `SCPSL_Data/Managed/`

### Шаг 3: Главный класс плагина

Создай файл `Plugin.cs`:

```csharp
using System;
using Exiled.API.Features;
using Exiled.API.Enums;

namespace MyFirstPlugin
{
    /// <summary>
    /// Главный класс плагина. Наследуется от Plugin<Config>.
    /// EXILED автоматически найдёт этот класс и загрузит плагин.
    /// </summary>
    public class Plugin : Plugin<Config>
    {
        // Singleton-паттерн для доступа к экземпляру плагина из любого места
        public static Plugin Instance { get; private set; }

        // === Метаданные плагина ===
        
        public override string Name => "MyFirstPlugin";
        public override string Author => "vityanvsk";
        public override Version Version => new Version(1, 0, 0);
        public override Version RequiredExiledVersion => new Version(9, 0, 0);
        
        // Приоритет загрузки: чем выше, тем раньше загрузится
        public override PluginPriority Priority => PluginPriority.Medium;

        /// <summary>
        /// Вызывается при включении плагина (старт сервера или hot-reload).
        /// </summary>
        public override void OnEnabled()
        {
            Instance = this;
            
            // Регистрируем обработчики событий
            RegisterEvents();
            
            Log.Info($"Plugin {Name} v{Version} by {Author} has been enabled!");
            base.OnEnabled();
        }

        /// <summary>
        /// Вызывается при выключении плагина.
        /// </summary>
        public override void OnDisabled()
        {
            // Отписываемся от событий (ОБЯЗАТЕЛЬНО!)
            UnregisterEvents();
            
            Instance = null;
            
            Log.Info($"Plugin {Name} has been disabled!");
            base.OnDisabled();
        }

        /// <summary>
        /// Вызывается после загрузки всех плагинов.
        /// Полезно для межплагинного взаимодействия.
        /// </summary>
        public override void OnReloaded()
        {
            Log.Debug("All plugins have been reloaded.");
            base.OnReloaded();
        }

        private void RegisterEvents()
        {
            // Пока пусто, заполним в следующих разделах
        }

        private void UnregisterEvents()
        {
            // Пока пусто
        }
    }
}
```

### Шаг 4: Конфигурация

Создай файл `Config.cs`:

```csharp
using System.ComponentModel;
using Exiled.API.Interfaces;

namespace MyFirstPlugin
{
    /// <summary>
    /// Класс конфигурации. EXILED автоматически сериализует его в YAML.
    /// </summary>
    public class Config : IConfig
    {
        // Обязательное свойство — включён ли плагин
        [Description("Включён ли плагин?")]
        public bool IsEnabled { get; set; } = true;

        // Режим отладки
        [Description("Включить ли режим отладки?")]
        public bool Debug { get; set; } = false;

        // === Твои кастомные настройки ===
        
        [Description("Сообщение приветствия при заходе игрока.")]
        public string WelcomeMessage { get; set; } = "Добро пожаловать на сервер!";

        [Description("Количество секунд до показа приветствия.")]
        public float WelcomeDelay { get; set; } = 3f;
    }
}
```

### Шаг 5: Сборка и установка

```bash
dotnet build -c Release

# Готовый файл: bin/Release/net48/MyFirstPlugin.dll
# Скопируй его в ~/.config/EXILED/Plugins/
```

### Шаг 6: Проверка

Запусти сервер и проверь консоль:
```
[MyFirstPlugin] Plugin MyFirstPlugin v1.0.0 by vityanvsk has been enabled!
```

🎉 **Поздравляю! Твой первый плагин работает.**

---

## 4. 🔄 Жизненный цикл плагина

```
Сервер запускается
       │
       ▼
┌──────────────┐
│ EXILED Loader│ — Сканирует папку Plugins/ на наличие DLL
└──────┬───────┘
       │  Для каждой DLL:
       │  1. Загружает сборку через Mono
       │  2. Ищет класс, наследующий Plugin<TConfig>
       │  3. Десериализует конфиг из YAML
       │  4. Сортирует по Priority
       │
       ▼
  OnEnabled()          ← Инициализация
       │
       ▼
  OnReloaded()         ← Все плагины загружены
       │
       ▼
  ┌─ Сервер работает ─┐
  │  Events...         │ ← Обработка событий
  │  Commands...       │
  │  Coroutines...     │
  └────────────────────┘
       │
       ▼
  OnDisabled()         ← Завершение / перезагрузка
```

**Важное правило:** Всё, что ты создаёшь в `OnEnabled()`, **должно** быть уничтожено в `OnDisabled()`. Иначе при hot-reload плагина будет утечка памяти и двойные обработчики.

---

## 5. 📡 Система событий (Events)

Система событий — **сердце** разработки плагинов EXILED. Вместо прямого изменения кода игры ты подписываешься на события.

### Структура обработчика

Создай файл `EventHandlers.cs`:

```csharp
using Exiled.API.Features;
using Exiled.Events.EventArgs.Player;
using Exiled.Events.EventArgs.Server;
using Exiled.Events.EventArgs.Map;
using Exiled.Events.EventArgs.Warhead;
using Exiled.Events.EventArgs.Scp096;

namespace MyFirstPlugin
{
    /// <summary>
    /// Все обработчики событий вынесены в отдельный класс.
    /// Это лучшая практика — не засоряй главный класс плагина.
    /// </summary>
    public class EventHandlers
    {
        // ========================
        //    СОБЫТИЯ ИГРОКОВ
        // ========================

        /// <summary>
        /// Вызывается, когда игрок прошёл аутентификацию и зашёл на сервер.
        /// </summary>
        public void OnVerified(VerifiedEventArgs ev)
        {
            Log.Info($"Игрок {ev.Player.Nickname} ({ev.Player.UserId}) зашёл на сервер.");
            
            // Показываем приветствие
            ev.Player.Broadcast(5, Plugin.Instance.Config.WelcomeMessage);
        }

        /// <summary>
        /// Вызывается, когда игрок получает урон.
        /// Можно модифицировать или отменить!
        /// </summary>
        public void OnHurting(HurtingEventArgs ev)
        {
            // ev.IsAllowed = false — отменяет урон
            // ev.Amount — количество урона (можно изменить)
            
            Log.Debug($"{ev.Attacker?.Nickname} нанёс {ev.Amount} урона игроку {ev.Player.Nickname}");

            // Пример: удвоить урон
            // ev.Amount *= 2;

            // Пример: запретить урон по союзникам (Friendly Fire)
            if (ev.Attacker != null && ev.Attacker.Role.Team == ev.Player.Role.Team)
            {
                ev.IsAllowed = false;
                ev.Attacker.ShowHint("Нельзя стрелять по своим!", 3);
            }
        }

        /// <summary>
        /// Вызывается, когда игрок умирает.
        /// </summary>
        public void OnDied(DiedEventArgs ev)
        {
            Log.Info($"{ev.Player.Nickname} был убит. " +
                     $"Убийца: {ev.Attacker?.Nickname ?? "Неизвестно"}");
        }

        /// <summary>
        /// Вызывается, когда игрок подбирает предмет.
        /// </summary>
        public void OnPickingUpItem(PickingUpItemEventArgs ev)
        {
            Log.Debug($"{ev.Player.Nickname} подбирает {ev.Pickup.Type}");
            
            // Пример: запретить подбирать MicroHID
            if (ev.Pickup.Type == ItemType.MicroHID)
            {
                ev.IsAllowed = false;
                ev.Player.ShowHint("MicroHID отключён на этом сервере!", 3);
            }
        }

        /// <summary>
        /// Вызывается, когда игрок получает роль (спавнится).
        /// </summary>
        public void OnChangingRole(ChangingRoleEventArgs ev)
        {
            Log.Debug($"{ev.Player.Nickname}: {ev.Player.Role.Type} → {ev.NewRole}");
        }

        // ========================
        //    СОБЫТИЯ СЕРВЕРА
        // ========================

        /// <summary>
        /// Вызывается в начале раунда.
        /// </summary>
        public void OnRoundStarted()
        {
            Log.Info("═══════════════════════════════");
            Log.Info("       РАУНД НАЧАЛСЯ!          ");
            Log.Info($"  Игроков: {Player.List.Count}");
            Log.Info("═══════════════════════════════");
            
            Map.Broadcast(5, "<b>Раунд начался! Удачи!</b>");
        }

        /// <summary>
        /// Вызывается, когда раунд заканчивается.
        /// </summary>
        public void OnRoundEnded(RoundEndedEventArgs ev)
        {
            Log.Info($"Раунд закончился! Победитель: {ev.LeadingTeam}");
        }

        /// <summary>
        /// Вызывается при ожидании игроков (лобби).
        /// </summary>
        public void OnWaitingForPlayers()
        {
            Log.Info("Сервер ожидает игроков...");
        }

        // ========================
        //    СОБЫТИЯ КАРТЫ
        // ========================

        /// <summary>
        /// Вызывается, когда SCP-914 обрабатывает предмет.
        /// </summary>
        public void OnUpgradingPickup(UpgradingPickupEventArgs ev)
        {
            Log.Debug($"SCP-914: {ev.Pickup.Type} на режиме {ev.KnobSetting}");
        }

        // ========================
        //    СОБЫТИЯ БОЕГОЛОВКИ
        // ========================

        /// <summary>
        /// Вызывается при запуске боеголовки.
        /// </summary>
        public void OnStarting(StartingEventArgs ev)
        {
            Log.Warn("БОЕГОЛОВКА АКТИВИРОВАНА!");
            Map.Broadcast(10, "<color=red><b>⚠ БОЕГОЛОВКА ЗАПУЩЕНА! ⚠</b></color>");
        }
    }
}
```

### Регистрация событий в Plugin.cs

Обнови методы `RegisterEvents()` и `UnregisterEvents()`:

```csharp
private EventHandlers _handlers;

private void RegisterEvents()
{
    _handlers = new EventHandlers();

    // Player Events
    Exiled.Events.Handlers.Player.Verified       += _handlers.OnVerified;
    Exiled.Events.Handlers.Player.Hurting        += _handlers.OnHurting;
    Exiled.Events.Handlers.Player.Died           += _handlers.OnDied;
    Exiled.Events.Handlers.Player.PickingUpItem  += _handlers.OnPickingUpItem;
    Exiled.Events.Handlers.Player.ChangingRole   += _handlers.OnChangingRole;

    // Server Events
    Exiled.Events.Handlers.Server.RoundStarted      += _handlers.OnRoundStarted;
    Exiled.Events.Handlers.Server.RoundEnded         += _handlers.OnRoundEnded;
    Exiled.Events.Handlers.Server.WaitingForPlayers  += _handlers.OnWaitingForPlayers;

    // Warhead Events
    Exiled.Events.Handlers.Warhead.Starting += _handlers.OnStarting;
}

private void UnregisterEvents()
{
    // ОБЯЗАТЕЛЬНО отписываемся в ТОЧНО ТАКОМ ЖЕ порядке!
    Exiled.Events.Handlers.Player.Verified       -= _handlers.OnVerified;
    Exiled.Events.Handlers.Player.Hurting        -= _handlers.OnHurting;
    Exiled.Events.Handlers.Player.Died           -= _handlers.OnDied;
    Exiled.Events.Handlers.Player.PickingUpItem  -= _handlers.OnPickingUpItem;
    Exiled.Events.Handlers.Player.ChangingRole   -= _handlers.OnChangingRole;

    Exiled.Events.Handlers.Server.RoundStarted      -= _handlers.OnRoundStarted;
    Exiled.Events.Handlers.Server.RoundEnded         -= _handlers.OnRoundEnded;
    Exiled.Events.Handlers.Server.WaitingForPlayers  -= _handlers.OnWaitingForPlayers;

    Exiled.Events.Handlers.Warhead.Starting -= _handlers.OnStarting;

    _handlers = null;
}
```

### Полный список категорий событий

| Категория | Пространство имён | Примеры |
|---|---|---|
| `Player` | `Exiled.Events.Handlers.Player` | Verified, Dying, Died, Hurting, Shooting, ChangingRole, Escaping... |
| `Server` | `Exiled.Events.Handlers.Server` | WaitingForPlayers, RoundStarted, RoundEnded, RespawningTeam... |
| `Map` | `Exiled.Events.Handlers.Map` | Decontaminating, GeneratorActivating, ExplodingGrenade... |
| `Warhead` | `Exiled.Events.Handlers.Warhead` | Starting, Stopping, Detonated |
| `Scp049` | `Exiled.Events.Handlers.Scp049` | FinishingRecall, StartingRecall, Attacking... |
| `Scp079` | `Exiled.Events.Handlers.Scp079` | GainingLevel, InteractingTesla, Pinging... |
| `Scp096` | `Exiled.Events.Handlers.Scp096` | Enraging, CalmingDown, AddingTarget... |
| `Scp106` | `Exiled.Events.Handlers.Scp106` | Attacking, Teleporting... |
| `Scp173` | `Exiled.Events.Handlers.Scp173` | Blinking, PlacingTantrum... |
| `Scp914` | `Exiled.Events.Handlers.Scp914` | UpgradingPickup, UpgradingInventoryItem... |
| `Scp939` | `Exiled.Events.Handlers.Scp939` | PlayingSound, Lunging... |
| `Scp3114` | `Exiled.Events.Handlers.Scp3114` | Disguising, Revealed... |
| `Item` | `Exiled.Events.Handlers.Item` | ChangingDurability, ChangingAmmo... |

### Свойство `IsAllowed`

Большинство событий с суффиксом `-ing` (происходящие ДО действия) имеют свойство `IsAllowed`:

```csharp
// Событие ДО действия — можно отменить
public void OnHurting(HurtingEventArgs ev)
{
    ev.IsAllowed = false; // Отменить урон
    ev.Amount = 0;        // Или обнулить
}

// Событие ПОСЛЕ действия — уже нельзя отменить
public void OnDied(DiedEventArgs ev)
{
    // ev.IsAllowed не существует — игрок уже мёртв
}
```

---

## 6. 👤 Работа с игроками (Player API)

Класс `Exiled.API.Features.Player` — это мощная обёртка над игровыми объектами.

```csharp
using Exiled.API.Features;
using Exiled.API.Features.Items;
using PlayerRoles;
using UnityEngine;

// === ПОЛУЧЕНИЕ ИГРОКОВ ===

// Все игроки на сервере
foreach (Player player in Player.List)
{
    Log.Info(player.Nickname);
}

// Поиск по разным критериям
Player p1 = Player.Get("Nickname");           // По нику
Player p2 = Player.Get(1);                     // По ID
Player p3 = Player.Get("76561198xxxxxxxxx");   // По SteamID / UserId
Player p4 = Player.Get(referenceHub);          // По ReferenceHub

// Количество
int alive = Player.List.Count(p => p.IsAlive);
int scps = Player.List.Count(p => p.IsScp);


// === ИНФОРМАЦИЯ ОБ ИГРОКЕ ===

Player player = ...;

string nick    = player.Nickname;       // Ник
string oderId  = player.UserId;         // Steam ID / Discord ID
string ip      = player.IPAddress;      // IP-адрес
int    id      = player.Id;             // Внутренний ID на сервере
bool   dnp     = player.DoNotTrack;     // Включена ли защита от трекинга
string country = player.Country;        // Код страны (US, RU, etc.)


// === РОЛЬ И СОСТОЯНИЕ ===

RoleTypeId role    = player.Role.Type;    // Текущая роль
Team       team    = player.Role.Team;    // Команда
bool       isAlive = player.IsAlive;      // Жив ли
bool       isScp   = player.IsScp;        // Является ли SCP
bool       isHuman = player.IsHuman;      // Человек ли


// === ЗДОРОВЬЕ ===

float hp    = player.Health;            // Текущее HP
float maxHp = player.MaxHealth;         // Максимальное HP
float ahp   = player.ArtificialHealth;  // Artificial HP (щит)
player.Health = 100;                     // Установить HP
player.Heal(50);                         // Подлечить на 50
player.Hurt(25);                         // Нанести 25 урона
player.Kill("причина");                  // Убить


// === ПОЗИЦИЯ И ПЕРЕМЕЩЕНИЕ ===

Vector3 pos = player.Position;           // Текущая позиция
player.Position = new Vector3(0, 1000, 0); // Телепортация
Quaternion rot = player.Rotation;        // Вращение

// Телепорт к другому игроку
player.Position = otherPlayer.Position;

// Телепорт в комнату
Room room = Room.List.First(r => r.Type == RoomType.Lcz914);
player.Teleport(room);


// === ИНВЕНТАРЬ ===

// Выдать предметы
player.AddItem(ItemType.GunE11SR);
player.AddItem(ItemType.Medkit);
player.AddItem(ItemType.KeycardO5);

// Выдать с боеприпасами
Item gun = player.AddItem(ItemType.GunE11SR);
if (gun is Firearm firearm)
{
    firearm.Ammo = firearm.MaxAmmo;
}

// Добавить боеприпасы
player.AddAmmo(AmmoType.Nato556, 200);

// Очистить инвентарь
player.ClearInventory();

// Проверить наличие предмета
bool hasKeycard = player.Items.Any(i => i.Type == ItemType.KeycardO5);

// Удалить конкретный предмет
Item itemToRemove = player.Items.FirstOrDefault(i => i.Type == ItemType.Medkit);
if (itemToRemove != null)
    player.RemoveItem(itemToRemove);

// Текущий предмет в руках
Item currentItem = player.CurrentItem;


// === РОЛЬ ===

// Сменить роль
player.Role.Set(RoleTypeId.ClassD);
player.Role.Set(RoleTypeId.NtfCaptain, Exiled.API.Enums.SpawnReason.ForceClass);
player.Role.Set(RoleTypeId.Scp173);


// === ЭФФЕКТЫ ===

player.EnableEffect(Exiled.API.Enums.EffectType.Scp207, 1);   // Cola-эффект 1x
player.EnableEffect(Exiled.API.Enums.EffectType.Invisible, 10); // Невидимость на 10 сек
player.DisableEffect(Exiled.API.Enums.EffectType.Scp207);
player.DisableAllEffects();

// Получить эффект
var effect = player.GetEffect(Exiled.API.Enums.EffectType.Bleeding);
if (effect.IsEnabled)
    Log.Info("Игрок истекает кровью!");


// === ИНТЕРФЕЙС ===

player.Broadcast(5, "Сообщение на 5 секунд");
player.ClearBroadcasts();
player.ShowHint("Подсказка внизу экрана", 3f);
player.SendConsoleMessage("Сообщение в консоль клиента", "green");


// === АДМИНИСТРИРОВАНИЕ ===

player.Ban(3600, "Причина бана");         // Бан на 1 час
player.Kick("Причина кика");              // Кик
player.Mute();                            // Мут
player.UnMute();                          // Размут
player.IsGodModeEnabled = true;           // Годмод
player.IsNoclipPermitted = true;          // Ноклип
```

---

## 7. ⚙ Конфигурация плагина

### Продвинутый конфиг

```csharp
using System.Collections.Generic;
using System.ComponentModel;
using Exiled.API.Interfaces;
using PlayerRoles;
using UnityEngine;

namespace MyFirstPlugin
{
    public class Config : IConfig
    {
        [Description("Включён ли плагин?")]
        public bool IsEnabled { get; set; } = true;

        [Description("Режим отладки.")]
        public bool Debug { get; set; } = false;

        // === Простые типы ===
        
        [Description("Приветственное сообщение.")]
        public string WelcomeMessage { get; set; } = "Добро пожаловать!";

        [Description("Множитель урона.")]
        public float DamageMultiplier { get; set; } = 1.0f;

        [Description("Максимальное количество SCP.")]
        public int MaxScpCount { get; set; } = 4;

        // === Коллекции ===
        
        [Description("Список предметов, которые запрещено подбирать.")]
        public List<ItemType> BannedItems { get; set; } = new List<ItemType>
        {
            ItemType.MicroHID,
            ItemType.Jailbird
        };

        [Description("Словарь: роль → стартовые предметы.")]
        public Dictionary<RoleTypeId, List<ItemType>> StartingItems { get; set; } = 
            new Dictionary<RoleTypeId, List<ItemType>>
        {
            [RoleTypeId.ClassD] = new List<ItemType> { ItemType.Flashlight, ItemType.Coin },
            [RoleTypeId.Scientist] = new List<ItemType> { ItemType.KeycardScientist, ItemType.Medkit }
        };

        // === Вложенные классы ===
        
        [Description("Настройки боеголовки.")]
        public WarheadSettings Warhead { get; set; } = new WarheadSettings();
    }

    public class WarheadSettings
    {
        [Description("Автоматически запускать боеголовку?")]
        public bool AutoStart { get; set; } = false;

        [Description("Через сколько секунд после начала раунда?")]
        public float AutoStartDelay { get; set; } = 900f;

        [Description("Можно ли отменить?")]
        public bool AllowCancel { get; set; } = true;
    }
}
```

### Сгенерированный YAML

После запуска сервера EXILED создаст конфиг:

```yaml
my_first_plugin:
  is_enabled: true
  debug: false
  welcome_message: 'Добро пожаловать!'
  damage_multiplier: 1
  max_scp_count: 4
  banned_items:
    - MicroHID
    - Jailbird
  starting_items:
    ClassD:
      - Flashlight
      - Coin
    Scientist:
      - KeycardScientist
      - Medkit
  warhead:
    auto_start: false
    auto_start_delay: 900
    allow_cancel: true
```

### Обращение к конфигу из кода

```csharp
// Из любого места через singleton
string msg = Plugin.Instance.Config.WelcomeMessage;
bool autoNuke = Plugin.Instance.Config.Warhead.AutoStart;

// Из обработчика событий
public void OnHurting(HurtingEventArgs ev)
{
    ev.Amount *= Plugin.Instance.Config.DamageMultiplier;
}
```

### Translations (переводы)

Для текстов, которые видят игроки, используй `ITranslation`:

```csharp
using System.ComponentModel;
using Exiled.API.Interfaces;

namespace MyFirstPlugin
{
    public class Translation : ITranslation
    {
        [Description("Сообщение при заходе.")]
        public string WelcomeMsg { get; set; } = "Welcome to the server!";

        [Description("Сообщение при смерти.")]
        public string DeathMsg { get; set; } = "You have been killed by {attacker}!";

        [Description("Сообщение при запуске боеголовки.")]
        public string NukeStarted { get; set; } = "⚠ WARHEAD ACTIVATED! ⚠";
    }
}
```

Обнови главный класс:

```csharp
public class Plugin : Plugin<Config, Translation>
{
    // Доступ через: Plugin.Instance.Translation.WelcomeMsg
}
```

---

## 8. 💻 Команды

EXILED поддерживает три типа команд:

| Тип | Где вводится | Интерфейс |
|---|---|---|
| **RemoteAdmin** | Консоль RA (клавиша `~`) | `ICommand` |
| **Client Console** | Консоль клиента (клавиша `` ` ``) | `ICommand` |
| **Server Console** | Консоль сервера | `ICommand` |

### RemoteAdmin-команда

```csharp
using System;
using CommandSystem;
using Exiled.API.Features;
using Exiled.Permissions.Extensions;
using RemoteAdmin;

namespace MyFirstPlugin.Commands
{
    /// <summary>
    /// Команда для RemoteAdmin консоли.
    /// Использование: heal [игрок] [количество]
    /// </summary>
    [CommandHandler(typeof(RemoteAdminCommandHandler))]
    public class HealCommand : ICommand
    {
        public string Command => "heal";
        
        public string[] Aliases => new[] { "hp", "лечить" };
        
        public string Description => "Вылечить игрока на указанное количество HP.";

        public bool Execute(ArraySegment<string> arguments, ICommandSender sender, out string response)
        {
            // Проверка прав
            if (!sender.CheckPermission("myplugin.heal"))
            {
                response = "У тебя нет прав на эту команду!";
                return false;
            }

            // Проверка аргументов
            if (arguments.Count < 2)
            {
                response = "Использование: heal <игрок> <количество>";
                return false;
            }

            // Поиск игрока
            Player target = Player.Get(arguments.At(0));
            if (target == null)
            {
                response = $"Игрок '{arguments.At(0)}' не найден!";
                return false;
            }

            // Парсинг количества
            if (!float.TryParse(arguments.At(1), out float amount) || amount <= 0)
            {
                response = "Некорректное количество HP!";
                return false;
            }

            // Лечим
            target.Heal(amount);

            response = $"Игрок {target.Nickname} вылечен на {amount} HP. " +
                       $"Текущее здоровье: {target.Health}/{target.MaxHealth}";
            return true;
        }
    }
}
```

### Client Console-команда

```csharp
[CommandHandler(typeof(ClientCommandHandler))]
public class InfoCommand : ICommand
{
    public string Command => ".info";
    public string[] Aliases => new[] { ".i" };
    public string Description => "Показать информацию о себе.";

    public bool Execute(ArraySegment<string> arguments, ICommandSender sender, out string response)
    {
        // Получаем игрока из sender
        if (!Player.TryGet(sender, out Player player))
        {
            response = "Ты не игрок!";
            return false;
        }

        response = $"=== Информация ===\n" +
                   $"Ник: {player.Nickname}\n" +
                   $"ID: {player.Id}\n" +
                   $"Роль: {player.Role.Type}\n" +
                   $"HP: {player.Health}/{player.MaxHealth}\n" +
                   $"Позиция: {player.Position}\n" +
                   $"Предметов: {player.Items.Count}/8";
        return true;
    }
}
```

### Server Console-команда

```csharp
[CommandHandler(typeof(GameConsoleCommandHandler))]
public class StatusCommand : ICommand
{
    public string Command => "pluginstatus";
    public string[] Aliases => new[] { "ps" };
    public string Description => "Статус плагина.";

    public bool Execute(ArraySegment<string> arguments, ICommandSender sender, out string response)
    {
        var plugin = Plugin.Instance;
        response = $"=== {plugin.Name} v{plugin.Version} ===\n" +
                   $"Автор: {plugin.Author}\n" +
                   $"Включён: {plugin.Config.IsEnabled}\n" +
                   $"Игроков онлайн: {Player.List.Count}";
        return true;
    }
}
```

### Родительская команда с подкомандами

```csharp
// Основная команда: mypl
[CommandHandler(typeof(RemoteAdminCommandHandler))]
public class MyPluginParentCommand : ParentCommand
{
    public override string Command => "mypl";
    public override string[] Aliases => new[] { "mp" };
    public override string Description => "Основная команда плагина.";

    public override void LoadGeneratedCommands()
    {
        // Регистрируем подкоманды
        RegisterCommand(new GiveAllCommand());
        RegisterCommand(new TeleportAllCommand());
    }

    protected override bool ExecuteParent(ArraySegment<string> arguments, ICommandSender sender, out string response)
    {
        response = "Доступные подкоманды: giveall, tpall\n" +
                   "Использование: mypl <подкоманда> [аргументы]";
        return false;
    }
}

// Подкоманда: mypl giveall <предмет>
public class GiveAllCommand : ICommand
{
    public string Command => "giveall";
    public string[] Aliases => new[] { "ga" };
    public string Description => "Выдать предмет всем игрокам.";

    public bool Execute(ArraySegment<string> arguments, ICommandSender sender, out string response)
    {
        if (arguments.Count < 1)
        {
            response = "Использование: mypl giveall <ItemType>";
            return false;
        }

        if (!Enum.TryParse(arguments.At(0), true, out ItemType itemType))
        {
            response = $"Неизвестный предмет: {arguments.At(0)}";
            return false;
        }

        int count = 0;
        foreach (Player player in Player.List)
        {
            if (!player.IsAlive) continue;
            player.AddItem(itemType);
            count++;
        }

        response = $"Выдано {itemType} всем живым игрокам ({count} чел.)";
        return true;
    }
}
```

---

## 9. ⏱ Coroutines и MEC

### Почему не `Task.Delay` и не `Thread.Sleep`?

**SCP:SL сервер — однопоточный** (как и Unity). Используя `Thread.Sleep`, ты **заморозишь весь сервер**. А `Task.Run` создаст код в другом потоке, что вызовет **race conditions** при обращении к Unity API.

**MEC (More Effective Coroutines)** — это библиотека, которая позволяет писать асинхронный код внутри основного потока Unity.

### Основы MEC

```csharp
using MEC;
using System.Collections.Generic;
using Exiled.API.Features;
using UnityEngine;

public class EventHandlers
{
    // Хранилище активных корутин (для возможности остановки)
    private CoroutineHandle _broadcastCoroutine;
    private CoroutineHandle _countdownCoroutine;

    public void OnRoundStarted()
    {
        // Запуск корутины
        _broadcastCoroutine = Timing.RunCoroutine(BroadcastLoop());
        _countdownCoroutine = Timing.RunCoroutine(NukeCountdown(300f));
    }

    /// <summary>
    /// Пример: бесконечный цикл с выводом сообщения каждые 60 секунд.
    /// </summary>
    private IEnumerator<float> BroadcastLoop()
    {
        while (Round.IsStarted)
        {
            // Ждём 60 секунд
            yield return Timing.WaitForSeconds(60f);

            // После ожидания выполняем действие
            Map.Broadcast(5, $"<b>Живых: {Player.List.Count(p => p.IsAlive)}</b>");
        }
    }

    /// <summary>
    /// Пример: обратный отсчёт до события.
    /// </summary>
    private IEnumerator<float> NukeCountdown(float seconds)
    {
        Log.Info($"Боеголовка запустится через {seconds} секунд!");

        // Ждём указанное время
        yield return Timing.WaitForSeconds(seconds);

        // Проверяем, что раунд ещё идёт
        if (!Round.IsStarted)
            yield break; // Выход из корутины

        Warhead.Start();
        Warhead.IsLocked = true;
        Map.Broadcast(10, "<color=red>БОЕГОЛОВКА АКТИВИРОВАНА! БЕЖАТЬ НЕЛЬЗЯ!</color>");
    }

    /// <summary>
    /// Пример: действие с задержкой для конкретного игрока.
    /// </summary>
    public void OnVerified(VerifiedEventArgs ev)
    {
        Timing.RunCoroutine(DelayedWelcome(ev.Player));
    }

    private IEnumerator<float> DelayedWelcome(Player player)
    {
        // Ждём 3 секунды
        yield return Timing.WaitForSeconds(3f);

        // ВАЖНО: проверяем, что игрок ещё на сервере!
        if (player == null || !player.IsConnected)
            yield break;

        player.Broadcast(8, "Добро пожаловать! Нажми ` для списка команд.");
        
        yield return Timing.WaitForSeconds(5f);

        if (player == null || !player.IsConnected)
            yield break;

        player.ShowHint("Приятной игры!", 3f);
    }

    /// <summary>
    /// Остановка всех корутин (вызывай в OnDisabled!)
    /// </summary>
    public void StopAllCoroutines()
    {
        Timing.KillCoroutines(_broadcastCoroutine);
        Timing.KillCoroutines(_countdownCoroutine);
    }
}
```

### Полезные методы MEC

```csharp
// Ожидание
yield return Timing.WaitForSeconds(5f);          // Ждать 5 секунд
yield return Timing.WaitForOneFrame;              // Ждать 1 кадр (tick)
yield return Timing.WaitUntilDone(otherCoroutine); // Ждать другую корутину

// Запуск
CoroutineHandle handle = Timing.RunCoroutine(MyCoroutine());

// Остановка
Timing.KillCoroutines(handle);

// Запуск с тегом (удобно для массовой остановки)
Timing.RunCoroutine(MyCoroutine(), "myPlugin_round");

// Остановить все корутины с тегом
Timing.KillCoroutines("myPlugin_round");

// Пауза / Возобновление
Timing.PauseCoroutines(handle);
Timing.ResumeCoroutines(handle);
```

### ⚠ Частые ошибки с MEC

```csharp
// ❌ НЕПРАВИЛЬНО — бесконечный цикл без yield заморозит сервер!
private IEnumerator<float> Bad()
{
    while (true)
    {
        DoSomething(); // Сервер зависнет!
    }
}

// ✅ ПРАВИЛЬНО — всегда есть yield
private IEnumerator<float> Good()
{
    while (true)
    {
        DoSomething();
        yield return Timing.WaitForSeconds(1f);
    }
}

// ❌ НЕПРАВИЛЬНО — не проверяешь, жив ли игрок после yield
private IEnumerator<float> Bad2(Player player)
{
    yield return Timing.WaitForSeconds(10f);
    player.Kill(); // NullReferenceException, если игрок вышел!
}

// ✅ ПРАВИЛЬНО
private IEnumerator<float> Good2(Player player)
{
    yield return Timing.WaitForSeconds(10f);
    if (player?.IsConnected != true) yield break;
    player.Kill();
}
```

---

## 10. 🎮 Работа с Unity Engine из плагина

### Основы: GameObject и Component

SCP:SL построен на Unity, поэтому все игровые объекты — это `GameObject` с привязанными `Component`.

```csharp
using UnityEngine;
using Exiled.API.Features;
using Mirror;

// === РАБОТА С ПОЗИЦИЯМИ ===

// Vector3 — основная структура для позиций и направлений
Vector3 position = new Vector3(0, 1000, 0);  // x, y, z
Vector3 direction = Vector3.forward;          // (0, 0, 1)
float distance = Vector3.Distance(pos1, pos2);

// Quaternion — повороты
Quaternion rotation = Quaternion.Euler(0, 90, 0); // Поворот на 90° по Y
Quaternion look = Quaternion.LookRotation(direction);


// === РАБОТА С КОМНАТАМИ ===

using Exiled.API.Features;
using Exiled.API.Enums;

// Получить все комнаты
foreach (Room room in Room.List)
{
    Log.Debug($"Комната: {room.Type} | Зона: {room.Zone} | Позиция: {room.Position}");
}

// Найти конкретную комнату
Room heavyArmory = Room.List.FirstOrDefault(r => r.Type == RoomType.HczArmory);
if (heavyArmory != null)
{
    player.Teleport(heavyArmory);
}

// Комнаты по зонам
var lczRooms = Room.List.Where(r => r.Zone == ZoneType.LightContainment);


// === РАБОТА С ДВЕРЬМИ ===

using Exiled.API.Features.Doors;

foreach (Door door in Door.List)
{
    Log.Debug($"Дверь: {door.Type} | Открыта: {door.IsOpen} | Заблокирована: {door.IsLocked}");
}

// Открыть/закрыть
Door gate = Door.List.FirstOrDefault(d => d.Type == DoorType.GateA);
gate?.Open();   // Или: gate.IsOpen = true;
gate?.Close();
gate?.Lock(300, DoorLockType.AdminCommand); // Заблокировать на 300 сек
gate?.Unlock();


// === РАБОТА С ЛИФТАМИ ===

using Exiled.API.Features;

foreach (Lift lift in Lift.List)
{
    Log.Debug($"Лифт: {lift.Type} | Уровень: {lift.CurrentLevel}");
}


// === РАБОТА С ГЕНЕРАТОРАМИ ===

foreach (Generator generator in Generator.List)
{
    generator.IsEngaged = true; // Активировать генератор
}


// === РАБОТА С ТЕСЛА ===

using Exiled.API.Features;

foreach (TeslaGate tesla in TeslaGate.List)
{
    tesla.Trigger(); // Активировать теслу
    // tesla.IsIdleMode = true; // Отключить
}


// === РАБОТА С РАГДОЛЛАМИ ===

foreach (Ragdoll ragdoll in Ragdoll.List)
{
    Log.Debug($"Труп: {ragdoll.Role} | Владелец: {ragdoll.Owner?.Nickname}");
}


// === СПАВН ПРЕДМЕТОВ НА КАРТЕ ===

using Exiled.API.Features.Pickups;

// Создать предмет на земле
Pickup pickup = Pickup.CreateAndSpawn(
    ItemType.KeycardO5,
    new Vector3(0, 1000, 0), // Позиция
    Quaternion.identity       // Поворот
);

// Удалить через 30 секунд
Timing.RunCoroutine(DestroyAfterDelay(pickup, 30f));

private IEnumerator<float> DestroyAfterDelay(Pickup pickup, float delay)
{
    yield return Timing.WaitForSeconds(delay);
    pickup?.Destroy();
}
```

### Raycast (Луч из глаз игрока)

```csharp
using UnityEngine;

/// <summary>
/// Бросает луч из глаз игрока и находит, куда он смотрит.
/// </summary>
public Player GetPlayerLookingAt(Player source, float maxDistance = 100f)
{
    // Начало луча — позиция камеры игрока
    Ray ray = new Ray(source.CameraTransform.position, source.CameraTransform.forward);

    if (Physics.Raycast(ray, out RaycastHit hit, maxDistance))
    {
        // Проверяем, попали ли мы в игрока
        var hub = hit.collider.GetComponentInParent<ReferenceHub>();
        if (hub != null)
        {
            return Player.Get(hub);
        }
    }

    return null;
}
```

### Создание своего MonoBehaviour (осторожно!)

```csharp
using UnityEngine;

/// <summary>
/// Компонент Unity, привязанный к игроку.
/// ВНИМАНИЕ: это продвинутый приём, используй осторожно!
/// </summary>
public class PlayerGlow : MonoBehaviour
{
    private Player _player;
    private float _timer;

    public void Init(Player player)
    {
        _player = player;
    }

    private void Update()
    {
        _timer += Time.deltaTime;
        
        if (_timer >= 1f)
        {
            _timer = 0;
            // Действие каждую секунду
            if (_player?.IsConnected == true)
            {
                _player.ShowHint("✨ Ты светишься! ✨", 1.1f);
            }
            else
            {
                Destroy(this); // Удаляем компонент, если игрок вышел
            }
        }
    }
}

// Использование:
// var glow = player.GameObject.AddComponent<PlayerGlow>();
// glow.Init(player);

// Удаление:
// UnityEngine.Object.Destroy(player.GameObject.GetComponent<PlayerGlow>());
```

---

## 11. 🐒 Mono Runtime — что нужно знать

### Что такое Mono?

**Mono** — это open-source реализация .NET Framework. Сервер SCP:SL использует Mono вместо .NET Core / .NET 8 Runtime. Это накладывает **ограничения**.

### Что работает ✅

```csharp
// Стандартные коллекции
List<T>, Dictionary<K,V>, HashSet<T>, Queue<T>

// LINQ
using System.Linq;
players.Where(p => p.IsAlive).Select(p => p.Nickname).ToList();

// Reflection
typeof(Player).GetMethods();
Activator.CreateInstance(type);

// JSON (Newtonsoft.Json — идёт в комплекте с EXILED)
using Newtonsoft.Json;
string json = JsonConvert.SerializeObject(obj);

// File I/O
using System.IO;
File.ReadAllText(path);
File.WriteAllText(path, data);

// Регулярные выражения
using System.Text.RegularExpressions;

// XML
using System.Xml;

// Crypto
using System.Security.Cryptography;
```

### Что может работать с ограничениями ⚠️

```csharp
// HttpClient — работает, но с нюансами
using System.Net.Http;

// async/await — работает, но помни:
// Нельзя обращаться к Unity API из другого потока!
// Используй Timing.CallDelayed или Timing.RunCoroutine

// Task.Run — создаёт фоновый поток (без доступа к Unity!)
await Task.Run(() => {
    // Тут можно делать тяжёлые вычисления
    // Но НЕЛЬЗЯ обращаться к Player, Map, GameObject и т.д.
});

// Span<T>, Memory<T> — частичная поддержка
// System.Text.Json — может не работать (используй Newtonsoft.Json)
```

### Что НЕ работает ❌

```csharp
// .NET 6+ специфичные API
// Minimal APIs, .NET Generic Host
// System.Text.Json (на старых Mono)
// DateOnly, TimeOnly (C# 10+)

// Нативные AOT-компилированные библиотеки
// Source Generators (при компиляции под net48 могут быть проблемы)
```

### Потокобезопасность

```csharp
// ❌ ОПАСНО — обращение к Unity API из другого потока
Task.Run(() => {
    Player.List.First().Kill(); // КРАШ!
});

// ✅ ПРАВИЛЬНО — выполнение в главном потоке через MEC
Task.Run(async () => {
    string data = await FetchFromApi(); // Тяжёлая операция в фоне
    
    // Возвращаемся в главный поток
    Timing.RunCoroutine(ProcessData(data));
});

private IEnumerator<float> ProcessData(string data)
{
    yield return Timing.WaitForOneFrame;
    
    // Теперь безопасно обращаться к Unity API
    foreach (Player player in Player.List)
    {
        player.Broadcast(5, data);
    }
}
```

### Где лежит Mono?

```
SCP Secret Laboratory/
├── SCPSL_Data/
│   ├── Managed/           ← Все DLL игры + Unity + Mono
│   │   ├── Assembly-CSharp.dll    ← Код игры
│   │   ├── UnityEngine.CoreModule.dll
│   │   ├── Mirror.dll
│   │   ├── mscorlib.dll          ← Mono BCL
│   │   └── System.*.dll
│   └── MonoBleedingEdge/  ← Рантайм Mono
│       └── lib/
│           └── mono/
```

---

## 12. 🔧 Harmony — патчинг игрового кода

### Зачем нужен Harmony?

**EXILED Events покрывают ~80% потребностей.** Но иногда нужно изменить поведение, для которого нет события. Тогда на помощь приходит **Harmony** — библиотека, которая позволяет **перехватывать и модифицировать любой метод** в runtime.

### Как работает Harmony

```
Оригинальный метод: PlayerStats.DealDamage()
       │
       ▼
┌──────────────────┐
│   Prefix Patch   │ ← Выполняется ДО оригинального метода
│   (return false  │   Может отменить вызов оригинала
│    = skip orig.) │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│ Оригинальный код │ ← Может быть пропущен Prefix'ом
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│  Postfix Patch   │ ← Выполняется ПОСЛЕ оригинального метода
│  (может изменить │   Может изменить результат
│   __result)      │
└──────────────────┘
       │
       ▼
┌──────────────────┐
│ Transpiler Patch │ ← Модифицирует IL-код метода напрямую
│ (продвинутый)    │   Самый мощный, но и самый сложный
└──────────────────┘
```

### Настройка Harmony в плагине

```csharp
using System;
using HarmonyLib;
using Exiled.API.Features;

public class Plugin : Plugin<Config>
{
    // Экземпляр Harmony — уникальный идентификатор
    private Harmony _harmony;

    public override void OnEnabled()
    {
        Instance = this;
        
        // Создаём экземпляр Harmony с уникальным ID
        _harmony = new Harmony($"com.vityanvsk.{Name}.{DateTime.Now.Ticks}");
        
        // Автоматически находит и применяет ВСЕ патчи в сборке
        _harmony.PatchAll();
        
        // Или патчим конкретный класс:
        // _harmony.PatchAll(typeof(MyPatch));
        
        Log.Info($"Applied {_harmony.GetPatchedMethods().Count()} Harmony patches.");
        
        base.OnEnabled();
    }

    public override void OnDisabled()
    {
        // ОБЯЗАТЕЛЬНО отменяем патчи при выключении!
        _harmony?.UnpatchAll(_harmony.Id);
        _harmony = null;
        
        base.OnDisabled();
    }
}
```

### Prefix Patch — перехват ДО метода

```csharp
using HarmonyLib;
using InventorySystem.Items.ThrowableProjectiles;
using Exiled.API.Features;

namespace MyFirstPlugin.Patches
{
    /// <summary>
    /// Пример: перехватываем бросок гранаты и логируем.
    /// </summary>
    [HarmonyPatch(typeof(ThrowableItem), nameof(ThrowableItem.ServerThrow))]
    public static class GrenadeThrowPatch
    {
        /// <summary>
        /// Prefix вызывается ДО оригинального метода.
        /// 
        /// Параметры:
        /// - __instance: объект, у которого вызван метод (this)
        /// - Можно перечислить параметры оригинального метода
        /// 
        /// Возвращаемое значение:
        /// - true: выполнить оригинальный метод
        /// - false: пропустить оригинальный метод
        /// </summary>
        public static bool Prefix(ThrowableItem __instance)
        {
            // Получаем игрока, который бросил
            Player player = Player.Get(__instance.Owner);
            
            if (player == null) return true; // Пропускаем, если не нашли

            Log.Info($"{player.Nickname} бросил {__instance.ItemTypeId}!");

            // Пример: запрет бросать гранаты для Class-D
            if (player.Role.Type == PlayerRoles.RoleTypeId.ClassD)
            {
                player.ShowHint("Class-D не могут бросать гранаты!", 3);
                return false; // Отменяем бросок
            }

            return true; // Разрешаем бросок
        }
    }
}
```

### Postfix Patch — перехват ПОСЛЕ метода

```csharp
using HarmonyLib;
using PlayerRoles;
using Exiled.API.Features;

namespace MyFirstPlugin.Patches
{
    /// <summary>
    /// Пример: после смены роли выдаём бонусное HP.
    /// </summary>
    [HarmonyPatch(typeof(PlayerRoleManager), nameof(PlayerRoleManager.ServerSetRole))]
    public static class RoleChangePatch
    {
        /// <summary>
        /// Postfix вызывается ПОСЛЕ оригинального метода.
        /// 
        /// Параметры:
        /// - __instance: объект (this)
        /// - __result: возвращаемое значение метода (если есть)
        /// - Параметры оригинального метода
        /// </summary>
        public static void Postfix(PlayerRoleManager __instance, RoleTypeId newRole)
        {
            Player player = Player.Get(__instance.Hub);
            if (player == null) return;

            // После смены роли на NTF Captain — даём бонусные HP
            if (newRole == RoleTypeId.NtfCaptain)
            {
                // Нужно подождать 1 фрейм, чтобы роль полностью применилась
                Timing.RunCoroutine(GiveBonusHp(player));
            }
        }

        private static IEnumerator<float> GiveBonusHp(Player player)
        {
            yield return Timing.WaitForOneFrame;
            
            if (player?.IsConnected != true) yield break;
            
            player.ArtificialHealth += 50;
            player.ShowHint("+50 AHP за звание капитана!", 3);
        }
    }
}
```

### Transpiler Patch — модификация IL (продвинутый)

```csharp
using System.Collections.Generic;
using System.Reflection.Emit;
using HarmonyLib;

namespace MyFirstPlugin.Patches
{
    /// <summary>
    /// Transpiler модифицирует IL-инструкции метода.
    /// Это самый мощный, но и самый сложный тип патча.
    /// 
    /// Используй только если Prefix/Postfix недостаточно!
    /// </summary>
    [HarmonyPatch(typeof(SomeClass), nameof(SomeClass.SomeMethod))]
    public static class TranspilerExample
    {
        public static IEnumerable<CodeInstruction> Transpiler(
            IEnumerable<CodeInstruction> instructions, 
            ILGenerator generator)
        {
            var codes = new List<CodeInstruction>(instructions);

            for (int i = 0; i < codes.Count; i++)
            {
                // Ищем конкретную инструкцию
                if (codes[i].opcode == OpCodes.Ldc_R4 && 
                    codes[i].operand is float f && 
                    f == 100f)
                {
                    // Заменяем значение 100f на 200f
                    codes[i].operand = 200f;
                    Log.Debug("Transpiler: заменил 100f на 200f");
                }
            }

            return codes;
        }
    }
}
```

### Как найти нужный метод для патча?

1. **dnSpy** / **ILSpy** — декомпиляторы .NET:
   - Открой `Assembly-CSharp.dll` из папки `Managed/`
   - Найди нужный класс и метод
   - Посмотри его сигнатуру

2. **Rider / Visual Studio** — перейди к определению через NuGet-пакет

3. **Примеры целей для патчей:**

```csharp
// Система урона
[HarmonyPatch(typeof(PlayerStatsSystem.PlayerStats), nameof(PlayerStatsSystem.PlayerStats.DealDamage))]

// Система дверей
[HarmonyPatch(typeof(Interactables.Interobjects.DoorUtils.DoorVariant), nameof(Interactables.Interobjects.DoorUtils.DoorVariant.ServerInteract))]

// Система SCP-914
[HarmonyPatch(typeof(Scp914.Scp914Controller), nameof(Scp914.Scp914Controller.ServerInteract))]
```

### ⚠ Важные правила Harmony

```
1. ВСЕГДА отменяй патчи в OnDisabled()
2. Используй уникальный Harmony ID
3. Prefix, возвращающий void — НЕ может отменить метод
   (нужно возвращать bool)
4. Не патчь то, что можно сделать через Events
5. Тестируй патчи тщательно — ошибка может крашнуть сервер
6. После обновления игры патчи МОГУТ сломаться
   (методы переименованы, сигнатуры изменены)
```

---

## 13. 💾 Работа с базой данных и файлами

### JSON-файлы (самый простой способ)

```csharp
using System.IO;
using Newtonsoft.Json;
using Exiled.API.Features;

namespace MyFirstPlugin
{
    public class DataManager
    {
        private readonly string _dataPath;
        private PlayerData _data;

        public DataManager()
        {
            // Папка для данных плагина
            string pluginDir = Path.Combine(Paths.Plugins, "MyFirstPlugin");
            Directory.CreateDirectory(pluginDir);
            _dataPath = Path.Combine(pluginDir, "player_data.json");
        }

        public void Load()
        {
            if (File.Exists(_dataPath))
            {
                string json = File.ReadAllText(_dataPath);
                _data = JsonConvert.DeserializeObject<PlayerData>(json) ?? new PlayerData();
            }
            else
            {
                _data = new PlayerData();
            }
            
            Log.Info($"Loaded data: {_data.Players.Count} players.");
        }

        public void Save()
        {
            string json = JsonConvert.SerializeObject(_data, Formatting.Indented);
            File.WriteAllText(_dataPath, json);
            Log.Debug("Data saved.");
        }

        public int GetKills(string oderId)
        {
            return _data.Players.TryGetValue(oderId, out var stats) ? stats.Kills : 0;
        }

        public void AddKill(string oderId, string nickname)
        {
            if (!_data.Players.ContainsKey(oderId))
            {
                _data.Players[oderId] = new PlayerStats { Nickname = nickname };
            }

            _data.Players[oderId].Kills++;
            _data.Players[oderId].Nickname = nickname; // Обновляем ник
        }
    }

    public class PlayerData
    {
        public Dictionary<string, PlayerStats> Players { get; set; } = new();
    }

    public class PlayerStats
    {
        public string Nickname { get; set; }
        public int Kills { get; set; }
        public int Deaths { get; set; }
        public int RoundsPlayed { get; set; }
    }
}
```

### SQLite (для больших объёмов данных)

Добавь NuGet-пакет:
```xml
<PackageReference Include="System.Data.SQLite.Core" Version="1.0.118" />
```

```csharp
using System.Data.SQLite;
using System.IO;
using Exiled.API.Features;

namespace MyFirstPlugin
{
    public class DatabaseManager
    {
        private readonly string _dbPath;
        private SQLiteConnection _connection;

        public DatabaseManager()
        {
            string pluginDir = Path.Combine(Paths.Plugins, "MyFirstPlugin");
            Directory.CreateDirectory(pluginDir);
            _dbPath = Path.Combine(pluginDir, "database.db");
        }

        public void Initialize()
        {
            _connection = new SQLiteConnection($"Data Source={_dbPath};Version=3;");
            _connection.Open();

            // Создаём таблицу
            using var cmd = _connection.CreateCommand();
            cmd.CommandText = @"
                CREATE TABLE IF NOT EXISTS players (
                    user_id TEXT PRIMARY KEY,
                    nickname TEXT NOT NULL,
                    kills INTEGER DEFAULT 0,
                    deaths INTEGER DEFAULT 0,
                    playtime_seconds INTEGER DEFAULT 0,
                    first_join TEXT,
                    last_join TEXT
                )";
            cmd.ExecuteNonQuery();

            Log.Info("Database initialized.");
        }

        public void AddKill(string oderId, string nickname)
        {
            using var cmd = _connection.CreateCommand();
            cmd.CommandText = @"
                INSERT INTO players (user_id, nickname, kills, first_join, last_join)
                VALUES (@id, @nick, 1, datetime('now'), datetime('now'))
                ON CONFLICT(user_id) DO UPDATE SET
                    nickname = @nick,
                    kills = kills + 1,
                    last_join = datetime('now')";
            cmd.Parameters.AddWithValue("@id", oderId);
            cmd.Parameters.AddWithValue("@nick", nickname);
            cmd.ExecuteNonQuery();
        }

        public (int kills, int deaths) GetStats(string oderId)
        {
            using var cmd = _connection.CreateCommand();
            cmd.CommandText = "SELECT kills, deaths FROM players WHERE user_id = @id";
            cmd.Parameters.AddWithValue("@id", oderId);

            using var reader = cmd.ExecuteReader();
            if (reader.Read())
            {
                return (reader.GetInt32(0), reader.GetInt32(1));
            }
            return (0, 0);
        }

        public void Close()
        {
            _connection?.Close();
            _connection?.Dispose();
        }
    }
}
```

> ⚠ **Важно:** Если используешь SQLite, положи нативные библиотеки (`.so` / `.dll`) в `Plugins/dependencies/`.

### LiteDB (альтернатива — NoSQL для .NET)

```xml
<PackageReference Include="LiteDB" Version="5.0.19" />
```

```csharp
using LiteDB;

public class LiteDbManager
{
    private LiteDatabase _db;

    public void Init()
    {
        _db = new LiteDatabase(Path.Combine(Paths.Plugins, "MyFirstPlugin", "data.db"));
    }

    public void SavePlayer(PlayerRecord record)
    {
        var col = _db.GetCollection<PlayerRecord>("players");
        col.Upsert(record);
    }

    public PlayerRecord GetPlayer(string oderId)
    {
        var col = _db.GetCollection<PlayerRecord>("players");
        return col.FindById(oderId);
    }

    public void Close() => _db?.Dispose();
}

public class PlayerRecord
{
    [BsonId]
    public string UserId { get; set; }
    public string Nickname { get; set; }
    public int Kills { get; set; }
}
```

---

## 14. 🌐 Сетевые запросы

### HTTP-запросы (к API, вебхуки Discord)

```csharp
using System;
using System.Net.Http;
using System.Text;
using System.Threading.Tasks;
using Newtonsoft.Json;
using Exiled.API.Features;
using MEC;
using System.Collections.Generic;

namespace MyFirstPlugin
{
    public class WebhookManager
    {
        // HttpClient переиспользуется — НЕ создавай новый для каждого запроса!
        private static readonly HttpClient HttpClient = new HttpClient
        {
            Timeout = TimeSpan.FromSeconds(10)
        };

        /// <summary>
        /// Отправить сообщение в Discord через Webhook.
        /// </summary>
        public static void SendDiscordMessage(string webhookUrl, string message)
        {
            // Запускаем в фоновом потоке, чтобы не блокировать сервер
            Task.Run(async () =>
            {
                try
                {
                    var payload = new
                    {
                        content = message,
                        username = "SCP:SL Server"
                    };

                    string json = JsonConvert.SerializeObject(payload);
                    var content = new StringContent(json, Encoding.UTF8, "application/json");

                    HttpResponseMessage response = await HttpClient.PostAsync(webhookUrl, content);

                    if (!response.IsSuccessStatusCode)
                    {
                        Log.Warn($"Discord webhook failed: {response.StatusCode}");
                    }
                }
                catch (Exception ex)
                {
                    Log.Error($"Discord webhook error: {ex.Message}");
                }
            });
        }

        /// <summary>
        /// Отправить embed в Discord.
        /// </summary>
        public static void SendDiscordEmbed(string webhookUrl, string title, string description, int color = 0x00FF00)
        {
            Task.Run(async () =>
            {
                try
                {
                    var payload = new
                    {
                        embeds = new[]
                        {
                            new
                            {
                                title,
                                description,
                                color,
                                timestamp = DateTime.UtcNow.ToString("o"),
                                footer = new { text = "SCP:SL Server" }
                            }
                        }
                    };

                    string json = JsonConvert.SerializeObject(payload);
                    var content = new StringContent(json, Encoding.UTF8, "application/json");
                    await HttpClient.PostAsync(webhookUrl, content);
                }
                catch (Exception ex)
                {
                    Log.Error($"Discord embed error: {ex.Message}");
                }
            });
        }

        /// <summary>
        /// GET-запрос к API и обработка результата в главном потоке.
        /// </summary>
        public static void FetchAndProcess(string url, Action<string> onSuccess)
        {
            Task.Run(async () =>
            {
                try
                {
                    string result = await HttpClient.GetStringAsync(url);

                    // Возвращаемся в главный поток через MEC
                    Timing.RunCoroutine(ProcessInMainThread(result, onSuccess));
                }
                catch (Exception ex)
                {
                    Log.Error($"HTTP GET error: {ex.Message}");
                }
            });
        }

        private static IEnumerator<float> ProcessInMainThread(string data, Action<string> callback)
        {
            yield return Timing.WaitForOneFrame;
            callback?.Invoke(data);
        }
    }
}
```

### Использование в EventHandlers

```csharp
public void OnDied(DiedEventArgs ev)
{
    // Отправляем в Discord при каждом убийстве
    string message = $"☠ **{ev.Player.Nickname}** был убит " +
                     $"игроком **{ev.Attacker?.Nickname ?? "???"}**";
    
    WebhookManager.SendDiscordMessage(
        Plugin.Instance.Config.DiscordWebhookUrl, 
        message
    );
}

public void OnRoundStarted()
{
    WebhookManager.SendDiscordEmbed(
        Plugin.Instance.Config.DiscordWebhookUrl,
        "🎮 Раунд начался!",
        $"Игроков: {Player.List.Count}",
        color: 0x00FF00 // Зелёный
    );
}
```

---

## 15. ✅ Лучшие практики и анти-паттерны

### ✅ DO (Делай так)

```csharp
// ✅ Singleton через Instance
public static Plugin Instance { get; private set; }

// ✅ Отписывайся от ВСЕХ событий в OnDisabled
public override void OnDisabled()
{
    UnregisterEvents();
    _harmony?.UnpatchAll(_harmony.Id);
    Instance = null;
    base.OnDisabled();
}

// ✅ Проверяй игрока на null после yield
yield return Timing.WaitForSeconds(5f);
if (player?.IsConnected != true) yield break;

// ✅ Используй try-catch в обработчиках событий
public void OnVerified(VerifiedEventArgs ev)
{
    try
    {
        // Код...
    }
    catch (Exception ex)
    {
        Log.Error($"Error in OnVerified: {ex}");
    }
}

// ✅ Выноси обработчики в отдельный класс
// ✅ Логируй важные действия через Log.Info/Debug/Warn/Error
// ✅ Используй конфиги для всех настраиваемых значений
// ✅ Убивай корутины в OnDisabled
// ✅ Используй Private=false для ссылок на EXILED DLL (не копировать в output)
```

### ❌ DON'T (Не делай так)

```csharp
// ❌ Thread.Sleep — заморозит сервер!
Thread.Sleep(5000);

// ❌ Бесконечный цикл без yield
while (true) { DoStuff(); }

// ❌ Обращение к Unity API из другого потока
Task.Run(() => { player.Kill(); }); // КРАШ!

// ❌ Забыл отписаться от событий
// При hot-reload обработчик вызовется дважды!

// ❌ Хранение Player объектов в статических коллекциях без очистки
static List<Player> vipPlayers = new(); // Утечка памяти!

// ❌ Исключения без try-catch в обработчиках
// Одно необработанное исключение может сломать цепочку обработки

// ❌ Жёстко заданные значения вместо конфига
if (player.Health < 50) // Почему 50? Сделай настраиваемым!

// ❌ Огромная логика в одном файле Plugin.cs
// Разделяй на EventHandlers, Commands, Managers и т.д.

// ❌ Использование Harmony там, где есть EXILED Event
// Harmony — последнее средство!
```

### Структура хорошего проекта

```
MyPlugin/
├── MyPlugin.sln
├── README.md
├── LICENSE
├── MyPlugin/
│   ├── MyPlugin.csproj
│   ├── Plugin.cs              ← Главный класс (минимум логики)
│   ├── Config.cs              ← Конфигурация
│   ├── Translation.cs         ← Переводы
│   ├── EventHandlers/         ← Обработчики событий
│   │   ├── PlayerHandlers.cs
│   │   ├── ServerHandlers.cs
│   │   └── ScpHandlers.cs
│   ├── Commands/              ← Команды
│   │   ├── HealCommand.cs
│   │   └── StatsCommand.cs
│   ├── Patches/               ← Harmony-патчи
│   │   └── CustomPatch.cs
│   ├── Services/              ← Бизнес-логика
│   │   ├── DataManager.cs
│   │   └── WebhookManager.cs
│   ├── Models/                ← Модели данных
│   │   └── PlayerData.cs
│   └── Extensions/            ← Методы расширения
│       └── PlayerExtensions.cs
└── refs/                      ← DLL для ссылок (если не NuGet)
```

---

## 16. 🐛 Отладка и профилирование

### Чтение логов

```bash
# Логи EXILED
tail -f ~/.config/EXILED/Logs/$(date +%Y-%m-%d).txt

# Логи сервера
tail -f ~/.config/SCP\ Secret\ Laboratory/LogOutput.txt
```

### Уровни логирования

```csharp
Log.Debug("Детальная информация (только если Debug=true в конфиге)");
Log.Info("Информационное сообщение");
Log.Warn("Предупреждение — что-то подозрительное");
Log.Error("Ошибка — что-то пошло не так");

// С форматированием
Log.Info($"Игрок {player.Nickname} (HP: {player.Health}) зашёл в комнату {room.Type}");
```

### Отладка через RemoteAdmin

```csharp
[CommandHandler(typeof(RemoteAdminCommandHandler))]
public class DebugCommand : ICommand
{
    public string Command => "mpdebug";
    public string[] Aliases => Array.Empty<string>();
    public string Description => "Debug info.";

    public bool Execute(ArraySegment<string> arguments, ICommandSender sender, out string response)
    {
        var sb = new StringBuilder();
        sb.AppendLine("=== MyPlugin Debug ===");
        sb.AppendLine($"Players: {Player.List.Count}");
        sb.AppendLine($"Round: {Round.IsStarted} ({Round.ElapsedTime})");
        sb.AppendLine($"Config.DamageMultiplier: {Plugin.Instance.Config.DamageMultiplier}");
        
        // Harmony патчи
        var patchedMethods = Harmony.GetAllPatchedMethods();
        sb.AppendLine($"Harmony patches: {patchedMethods.Count()}");
        
        foreach (var method in patchedMethods)
        {
            var info = Harmony.GetPatchInfo(method);
            sb.AppendLine($"  {method.DeclaringType?.Name}.{method.Name}: " +
                         $"P:{info.Prefixes.Count} Po:{info.Postfixes.Count} T:{info.Transpilers.Count}");
        }

        response = sb.ToString();
        return true;
    }
}
```

### Замер производительности

```csharp
using System.Diagnostics;

public void OnHurting(HurtingEventArgs ev)
{
    var sw = Stopwatch.StartNew();
    
    // ... твой код ...
    
    sw.Stop();
    if (sw.ElapsedMilliseconds > 5)
    {
        Log.Warn($"OnHurting took {sw.ElapsedMilliseconds}ms! (should be <1ms)");
    }
}
```

### Типичные ошибки и их решения

| Ошибка | Причина | Решение |
|---|---|---|
| `NullReferenceException` | Игрок вышел / объект уничтожен | Проверяй на `null` после `yield` |
| `TypeLoadException` | Неправильная версия DLL | Обнови зависимости |
| `MissingMethodException` | Метод переименован после обновления игры | Обнови патч |
| `InvalidOperationException: Collection was modified` | Модификация коллекции в foreach | Используй `.ToList()` или цикл `for` |
| Обработчик вызывается дважды | Забыл отписаться в `OnDisabled` | Добавь `-=` для всех событий |
| Сервер зависает | `Thread.Sleep` или цикл без `yield` | Используй MEC |

---

## 17. 📦 Публикация плагина

### Подготовка к релизу

#### 1. README.md

```markdown
# MyAwesomePlugin

Плагин для [EXILED](https://github.com/ExMod-Team/EXILED) фреймворка.

## Описание
Краткое описание того, что делает плагин.

## Возможности
- ✅ Фича 1
- ✅ Фича 2
- ✅ Фича 3

## Требования
- SCP:SL Server **v13.x**
- EXILED **v9.0.0+**

## Установка
1. Скачай `MyAwesomePlugin.dll` из [Releases](link)
2. Положи в `~/.config/EXILED/Plugins/`
3. Перезапусти сервер
4. Настрой конфиг в `~/.config/EXILED/Configs/{port}-config.yml`

## Конфигурация
```yaml
my_awesome_plugin:
  is_enabled: true
  debug: false
  # ... пример конфига
```

## Команды
| Команда | Описание | Права |
|---|---|---|
| `heal <player> <amount>` | Лечит игрока | `myplugin.heal` |

## Разрешения
| Право | Описание |
|---|---|
| `myplugin.heal` | Доступ к команде heal |
| `myplugin.admin` | Полный доступ |
```

#### 2. .gitignore

```gitignore
bin/
obj/
*.user
*.suo
.vs/
.idea/
refs/
*.nupkg
```

#### 3. GitHub Actions для автосборки

Создай файл `.github/workflows/build.yml`:

```yaml
name: Build Plugin

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: '8.0.x'
    
    - name: Restore dependencies
      run: dotnet restore
    
    - name: Build
      run: dotnet build -c Release --no-restore
    
    - name: Upload artifact
      uses: actions/upload-artifact@v4
      with:
        name: MyPlugin
        path: MyPlugin/bin/Release/net48/MyPlugin.dll
```

#### 4. Автоматический релиз

```yaml
name: Release

on:
  push:
    tags: [ 'v*' ]

jobs:
  release:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: '8.0.x'
    
    - name: Build
      run: dotnet build -c Release
    
    - name: Create Release
      uses: softprops/action-gh-release@v1
      with:
        files: MyPlugin/bin/Release/net48/MyPlugin.dll
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

#### 5. Версионирование

Используй [SemVer](https://semver.org/):
- `1.0.0` → `1.0.1` — баг-фикс
- `1.0.0` → `1.1.0` — новая фича (обратно совместимая)
- `1.0.0` → `2.0.0` — breaking changes

---

## 18. ❓ FAQ

### Q: Мой плагин не загружается!
**A:** Проверь:
1. DLL лежит в `EXILED/Plugins/`
2. `TargetFramework` = `net48`
3. Есть класс, наследующий `Plugin<Config>`
4. `IsEnabled = true` в конфиге
5. Версия EXILED совместима с `RequiredExiledVersion`
6. В консоли нет ошибок `TypeLoadException`

### Q: Событие не срабатывает!
**A:** Проверь:
1. Подписался в `OnEnabled` через `+=`
2. Сигнатура метода совпадает (правильный тип `EventArgs`)
3. Другой плагин не отменяет событие через `IsAllowed = false`
4. Включён ли debug-режим для отладки

### Q: Как обновить плагин после обновления игры?
**A:**
1. Обнови EXILED до последней версии
2. Обнови ссылки на DLL / NuGet-пакеты
3. Проверь Harmony-патчи — методы могли измениться
4. Пересобери и протестируй

### Q: Как использовать CustomItems / CustomRoles?
**A:** EXILED имеет модули `Exiled.CustomItems` и `Exiled.CustomRoles`. Это отдельная большая тема — смотри [документацию EXILED](https://github.com/ExMod-Team/EXILED).

### Q: Можно ли использовать async/await?
**A:** Да, но осторожно:
- Для HTTP-запросов и I/O — `Task.Run` с callback через MEC
- **Никогда** не обращайся к Unity API из `Task.Run`
- Для задержек используй MEC, а не `Task.Delay`

### Q: Как взаимодействовать с другими плагинами?
**A:**
```csharp
// Проверить, загружен ли плагин
var otherPlugin = Exiled.Loader.Loader.Plugins
    .FirstOrDefault(p => p.Name == "OtherPlugin");

if (otherPlugin != null)
{
    Log.Info($"OtherPlugin v{otherPlugin.Version} найден!");
}

// Для прямого API-взаимодействия:
// Добавь DLL другого плагина как Reference и вызывай его публичные методы
```

### Q: Мой Harmony-патч крашит сервер!
**A:**
1. Проверь, что метод существует: `AccessTools.Method(typeof(X), "MethodName")`
2. Убедись, что `Prefix` возвращает `bool`, а не `void` (если хочешь отменять)
3. Оберни код в `try-catch`
4. Проверь `__instance` на `null`
5. Не создавай бесконечные циклы в патчах

---

## 19. 🔗 Полезные ссылки

| Ресурс | Ссылка |
|---|---|
| **EXILED GitHub** | [github.com/ExMod-Team/EXILED](https://github.com/ExMod-Team/EXILED) |
| **EXILED Docs** | [exmod-team.github.io/EXILED](https://exmod-team.github.io/EXILED/) |
| **EXILED Discord** | [discord.gg/exiled](https://discord.gg/PyUkWTg) |
| **Harmony Wiki** | [harmony.pardeike.net](https://harmony.pardeike.net/) |
| **Unity Scripting API** | [docs.unity3d.com](https://docs.unity3d.com/ScriptReference/) |
| **MEC Documentation** | [trello.com/b/aGFEBfho](https://trello.com/b/aGFEBfho) |
| **dnSpy (декомпилятор)** | [github.com/dnSpy/dnSpy](https://github.com/dnSpy/dnSpy) |
| **ILSpy** | [github.com/icsharpcode/ILSpy](https://github.com/icsharpcode/ILSpy) |
| **SCP:SL Official** | [scpslgame.com](https://scpslgame.com/) |
| **Northwood Studios** | [store.steampowered.com/app/700330](https://store.steampowered.com/app/700330/) |

---

## 📝 Лицензия

Это руководство распространяется под лицензией **MIT**. Свободно используй, изменяй и распространяй.

---

<div align="center">

**Сделано с ❤️ для сообщества SCP:SL**

*Автор: [vityanvsk](https://github.com/vityanvsk) aka kyrvochkarat*

*Если это руководство помогло тебе — поставь ⭐ на GitHub!*

</div>

---

> **Последнее обновление:** 2025
> **Совместимость:** EXILED 9.x | SCP:SL 13.x | .NET Framework 4.8 / Mono
