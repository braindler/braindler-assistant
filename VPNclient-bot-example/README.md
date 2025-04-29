# VPNclient-bot

.json workflow для n8n 

## Структура файлов

## База данных

```mysql
-- Таблица пользователей
DROP TABLE users;
CREATE TABLE users (
    id BIGINT PRIMARY KEY, -- telegram_id
    username VARCHAR(255),
    first_name VARCHAR(255),
    last_name VARCHAR(255),
    language_code VARCHAR(10),
    is_premium BOOLEAN DEFAULT FALSE,
    referrer_id BIGINT, -- тоже ссылается на telegram_id
    balance DECIMAL(10, 2) DEFAULT 0,
    tarif DECIMAL(10, 2) DEFAULT 0,
    activated BOOLEAN DEFAULT FALSE,
    utm VARCHAR(255),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    activated_at DATETIME DEFAULT NULL,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (referrer_id) REFERENCES users(id)
);

-- Таблица устройств
CREATE TABLE devices (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    device_name VARCHAR(255),
    marzban_id VARCHAR(255),
    config_url TEXT,
    status ENUM('active', 'inactive', 'deleted') DEFAULT 'active',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Таблица подписок
CREATE TABLE subscriptions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    plan_name VARCHAR(255),
    device_limit INT DEFAULT 1,
    price DECIMAL(10, 2),
    expires_at DATETIME,
    auto_renew BOOLEAN DEFAULT FALSE,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Таблица транзакций
CREATE TABLE transactions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    amount DECIMAL(10, 2),
    type ENUM('payment', 'referral_bonus', 'manual_adjustment'),
    comment TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

## Схема
```mermaid
graph TD
  Start[/Пользователь вводит /start/] --> Welcome[Приветствие и информация о балансе]
  Welcome --> Menu{Главное меню}
  
  Menu --> Devices[/Мои устройства/]
  Menu --> AddDevice[/Добавить устройство/]
  Menu --> RemoveDevice[/Удалить устройство/]
  Menu --> Invite[/Пригласить друга/]
  Menu --> Help[/Помощь/]
  Menu --> Privacy[/Конфиденциальность/]

  Devices --> ShowDevices[Показать список устройств с типами и ID]
  ShowDevices --> DeviceLink[Получить ссылку для настройки]

  AddDevice --> AddWarning[Предупреждение о стоимости]
  AddWarning --> DeviceType{Выберите тип устройства}
  DeviceType --> iOS[iPhone/iPad]
  DeviceType --> Android[Android]
  DeviceType --> Comp[Компьютер]
  DeviceType --> TV[Android TV]

  Comp --> OSChoice{Mac или Windows}
  OSChoice --> Mac[MacOS]
  OSChoice --> Windows[Windows]

  iOS --> GenLink[Генерация ссылки и инструкция]
  Android --> GenLink
  Mac --> GenLink
  Windows --> GenLink
  TV --> GenLink

  RemoveDevice --> DeviceList[Выбор устройства из списка]
  DeviceList --> RemoveConfirm[Подтверждение удаления]
  RemoveConfirm --> Check24h{Проверка: >24ч с момента добавления?}
  Check24h -- Нет --> BlockRemove[Ошибка: нельзя удалить]
  Check24h -- Да --> VPNCheck[Предупреждение: отключите VPN]
  VPNCheck --> RemoveSuccess[Удалено + обновление баланса]
  RemoveSuccess --> LastDeviceCheck{Последнее устройство?}
  LastDeviceCheck -- Да --> FreezeCharges[Остановка списаний]
  LastDeviceCheck -- Нет --> Menu

  Invite --> GenRefLink[Генерация реферальной ссылки]
  GenRefLink --> ShowRef[Условия: +50₽ / +100₽ другу]

  Help --> Topics{Выбор темы}
  Topics --> Topic1[VPN не работает]
  Topics --> Topic2[Инструкция по установке]
  Topics --> Topic3[Не могу найти ссылку]
  Topics --> Topic4[Низкая скорость]
  Topics --> Topic5[Телевизор (Android TV)]
  Topics --> Topic6[Заморозка баланса]
  Topics --> Topic7[Альтернативный протокол]

  Privacy --> Policy[Описание политики конфиденциальности]

  style Start fill:#e0f7fa,stroke:#00acc1,stroke-width:2px
  style Menu fill:#f1f8e9
  style Devices,AddDevice,RemoveDevice,Invite,Help,Privacy fill:#fff3e0
  style ShowDevices,AddWarning,DeviceType,DeviceList,GenRefLink,Topics fill:#e3f2fd
  style Welcome fill:#dcedc8
```


# Техническое задание на разработку Telegram бота VPNclient-bot

## 1. Общее описание
Telegram бот VPNclient-bot предназначен для управления VPN-сервисом, включая настройку устройств, управление подпиской и балансом.

## 2. Основные функции бота

### 2.1. Главное меню (/start)
- Приветствие пользователя с именем
- Отображение текущего баланса и срока действия (~N дней)
- Информация о тарифе (100₽/мес за 1 устройство) и количестве активных устройств
- Рекомендация подписаться на канал @VPNclient_news
- Кнопка "Мои устройства" для повторного скачивания настроек
- Реферальная программа (50₽ за приглашенного друга)

### 2.2. Управление устройствами (/devices)
#### 2.2.1. Просмотр списка устройств
- Отображение количества активных устройств и месячной платы
- Предупреждение о необходимости удалять неиспользуемые устройства
- Список устройств с идентификаторами и типами:
  - App (iPhone/iPad)
  - Vless (Android)
  - Comp (Mac/Windows)
  - TV (Android TV)

#### 2.2.2. Добавление устройства
1. Предупреждение о стоимости (100₽/мес за каждое устройство)
2. Выбор типа устройства:
   - iOS (iPhone, iPad)
   - Android
   - Компьютер (MacOS, Windows)
   - Телевизор (Android TV)
3. Для компьютера - уточнение типа (Mac/Windows)
4. Генерация уникальной ссылки для настройки
5. Инструкция по настройке для каждого типа устройства

#### 2.2.3. Удаление устройства
1. Выбор устройства из списка
2. Подтверждение удаления с предупреждением отключить VPN
3. Проверка: нельзя удалить устройство, добавленное менее 24 часов назад
4. При удалении последнего устройства - уведомление о прекращении списаний

### 2.3. Реферальная программа (/invite)
- Генерация персональной реферальной ссылки
- Условия: 50₽ за приглашенного друга (друг получает 100₽)

### 2.4. Помощь (/help)
- Список тем для помощи:
  - VPN не работает
  - Инструкция по установке
  - Не могу найти ссылку
  - Низкая скорость
  - Телевизор (Android TV)
  - Заморозка баланса
  - Альтернативный протокол

### 2.5. Конфиденциальность (/privacy)
- Политика конфиденциальности с описанием:
  - Использование только Telegram ID
  - Отсутствие сбора персональных данных
  - Условия хранения и удаления данных

## 3. Бизнес-логика и алгоритмы

### 3.1. Управление балансом
- Расчет оставшегося срока действия баланса (~N дней)
- Списание абонентской платы: 100₽/мес за каждое активное устройство
- При удалении всех устройств - заморозка списаний

### 3.2. Работа с устройствами
- Ограничение: нельзя удалить устройство, добавленное менее 24 часов назад
- При добавлении устройства:
  - Генерация уникальной ссылки формата VPNclient://[уникальный_токен]
  - Сохранение типа устройства и времени добавления
- При удалении устройства:
  - Проверка активности VPN (только предупреждение)
  - Обновление списка активных устройств
  - Пересчет абонентской платы

### 3.3. Реферальная система
- Генерация уникальной ссылки формата https://t.me/VPNclient-bot?start=[user_id]
- Начисление 50₽ при успешной регистрации друга
- Начисление 100₽ новому пользователю

## 4. Тексты сообщений

### 4.1. Основные сообщения
**Приветствие:**
Рады видеть вас снова, [Имя]!

Ваш баланс [Сумма]₽ (~[Дней] дней), аккаунт активен
Тариф 100₽/мес за 1 устройство, активно [N] устройств.

Если вы потеряли настройки - вы можете повторно их скачать, нажав на кнопку "Мои устройства".

👭 Пригласите друзей в наш сервис и получите 50₽ на баланс за каждого друга. Ваши друзья получат 100₽ на баланс!

📌 Обязательно (!!) подпишитесь на наш канал @VPNclient_news!


**Список устройств:**
У вас активно [N] устройств, цена 100₽/мес. за устройство. Ваша абонентская плата [Сумма]₽ в месяц.

❌ Удаляйте из бота устройства, которые вы не используете, чтобы не платить абонентскую плату за них!

🗝 Нажмите на идентификатор устройства в списке, чтобы получить ссылку для настройки приложения VPN.

Список ваших устройств:


### 4.2. Удаление устройства
**Предупреждение:**
ВНИМАНИЕ! Выключите VPN перед удалением устройства!


**Ошибка (менее 24 часов):**
Нельзя удалить устройство [ID] [Тип], прошло менее 24 часов


**Успешное удаление:**
Устройство [ID] [Тип] удалено!


**Удаление последнего устройства:**
❗️ Внимание! Было удалено последнее устройство. Абоненская плата с вашего баланса теперь не взимается.


### 4.3. Добавление устройства
**Предупреждение:**
🙋‍♀️ Внимание! Вы добавляете настройки для еще одного устройства. Стоимость 100р/мес за каждое дополнительное устройство.


**Инструкция для iOS:**
1. Установите приложение VPNclient из AppStore
2. Скопируйте ссылку настройки в буфер обмена. Настройки всегда доступны в разделе "Мои устройства".
3. Запустите приложение VPNclient, вставьте скопированную ссылку и разрешите приложению добавление конфигураций VPN.
4. Включите VPN.


**Инструкция для Windows:**
1. Скачайте и установите приложение VPNclient для Windows: [ссылка]
2. Скопируйте ссылку в буфер обмена. Ссылка для настройки VPN всегда доступна в разделе "Мои устройства".
3. Запустите приложение VPNclient и вставьте скопированную ссылку.
4. Включите VPN.


## 5. Технические требования
- Интеграция с платежной системой для пополнения баланса
- Хранение данных о пользователях и их устройствах
- Генерация уникальных VPN-конфигураций
- Логирование действий пользователей
- Система учета рефералов

## 6. Дополнительные требования
- Регулярные напоминания о подписке на канал @VPNclient_news
- Проверка подписки пользователя на канал
- Запрет на использование сервиса для скачивания торрентов
- Возможность обновления приложений (VPNclient для iPhone, HitRay для Android)
