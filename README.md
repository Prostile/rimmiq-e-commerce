# Full-Stack E-commerce Платформа "rimmiq"

## 1. О проекте

`rimmiq` — это full-stack e-commerce платформа, разработанная с нуля. Проект представляет собой полноценный интернет-магазин с уникальной функцией — **конструктором кастомных сумок**, позволяющим пользователям в реальном времени создавать персонализированный продукт.

**Статус проекта:** Проект является частной коммерческой инициативой, находящейся в активной разработке. В связи с этим, исходный код представляет собой коммерческую тайну и не может быть опубликован. Этот репозиторий служит профессиональной витриной, демонстрирующей архитектуру, реализованный функционал и используемый технологический стек. 

## 2. Демонстрация работы

#### Стандартный e-commerce цикл

Платформа реализует полный и привычный для пользователя цикл покупки: от просмотра каталога до страницы оформления заказа. 
Так как сайт находится на стадии разработки, большинство изображений отсутствуют.

<p align="center">
  <img src="https://github.com/Prostile/rimmiq-e-commerce/blob/main/rimmiq.gif" alt="Демонстрация страницы отслеживания заказа" width="80%"/>
</p>

## 3. Решаемая бизнес-проблема

Проект был разработан для решения ключевых проблем малого авторского бренда, стремящегося уйти от зависимости от маркетплейсов:

*   **Высокие комиссии и отсутствие контроля:** Маркетплейсы забирают значительный процент от прибыли и не позволяют выстраивать прямую коммуникацию с клиентом.
*   **Размытие бренда:** На общей витрине маркетплейса невозможно передать уникальную историю и ценности бренда.
*   **Ограниченный функционал:** Отсутствие возможности предложить клиентам уникальный опыт, такой как создание кастомного продукта.

`rimmiq` решает эти проблемы, предоставляя бренду собственный, полностью контролируемый канал продаж с уникальным функционалом и возможностью выстраивать долгосрочные отношения с клиентами.

## 4. Архитектура и технические решения

Проект построен на отказоустойчивой, событийно-ориентированной микросервисной архитектуре. Все компоненты полностью контейнеризированы с помощью Docker и взаимодействуют друг с другом через API-шлюз или брокер сообщений RabbitMQ.

```mermaid
graph TD;
    subgraph "Клиент"
        User[💻 Пользователь в браузере];
    end

    subgraph "Внешний мир"
        Admin[📱 Администратор в Telegram];
        SDEK_API[🚚 СДЭК API];
        PaymentGateway[💳 Платежный шлюз ЮKassa];
    end
    
    subgraph "Наша Система (Единая Docker-сеть)"
        
        subgraph "Входной Уровень"
            ApiGateway[🚀 API Gateway Go];
        end

        subgraph "Сервисы Бизнес-Логики (Python/FastAPI)"
            ProductService[📦 Product Service];
            OrderService[🛒 Order Service];
        end

        subgraph "Асинхронные Воркеры (Python)"
            TelegramService[🤖 Telegram Service];
            NotificationService[✉️ Notification Service];
            ProductWorker[🛠️ Product Worker];
            OrderWorker[⚙️ Order Worker];
        end

        subgraph "Хранилища Данных"
            PostgreSQL[🗄️ PostgreSQL DB];
            RabbitMQ[🐇 RabbitMQ];
        end
    end

    %% --- Потоки данных ---
    
    %% 1. Пользовательский путь
    User -- HTTPS --> ApiGateway;
    ApiGateway -- HTTP --> ProductService;
    ApiGateway -- HTTP --> OrderService;
    
    %% 2. Взаимодействие с базой данных
    ProductService --- PostgreSQL;
    OrderService --- PostgreSQL;
    ProductWorker --- PostgreSQL;
    OrderWorker --- PostgreSQL;

    %% 3. Взаимодействие с RabbitMQ (События и Команды)
    OrderService -- Публикует События --> RabbitMQ;
    TelegramService -- Публикует Команды --> RabbitMQ;
    
    TelegramService -- Читает События --> RabbitMQ;
    NotificationService -- Читает События --> RabbitMQ;
    ProductWorker -- Читает События --> RabbitMQ;
    OrderWorker -- Читает Команды --> RabbitMQ;

    %% 4. Взаимодействие между сервисами
    OrderService -- Запрос цен --> ProductService;
    
    %% 5. Взаимодействие с внешними системами
    Admin <--> TelegramService;
    OrderService -- Запрос на оплату --> PaymentGateway;
    PaymentGateway -- Webhook --> OrderService;
    %% NotificationService -- Отправка email --> SendGrid API (будущая интеграция)
    %% DeliveryService -- Запросы --> SDEK_API (будущая интеграция)

    %% --- Стилизация для наглядности ---
    style User fill:#D6EAF8,stroke:#3498DB
    style Admin fill:#E8DAEF,stroke:#8E44AD
    style ApiGateway fill:#D5F5E3,stroke:#2ECC71
    style RabbitMQ fill:#FADBD8,stroke:#E74C3C
    style PostgreSQL fill:#D2B4DE,stroke:#8E44AD
    style ProductService fill:#FEF9E7,stroke:#F1C40F
    style OrderService fill:#FEF9E7,stroke:#F1C40F
    style TelegramService fill:#EBF5FB,stroke:#3498DB
    style NotificationService fill:#EBF5FB,stroke:#3498DB
    style ProductWorker fill:#FDEDEC,stroke:#E74C3C
    style OrderWorker fill:#FDEDEC,stroke:#E74C3C
```

*   **`API Gateway` (Go):** Единственная точка входа для фронтенда. Написан на Go для максимальной производительности, надежности и минимального потребления ресурсов. Маршрутизирует запросы к соответствующим внутренним сервисам.
*   **`Product Service` (Python, FastAPI):** "Источник правды" для всего, что касается товаров, компонентов, их остатков и цен.
*   **`Order Service` (Python, FastAPI):** Управляет полным жизненным циклом заказов: от создания и расчета стоимости до списания остатков и обновления статусов. Оркестрирует взаимодействие с другими сервисами.
*   **`Notification Service` (Python):** Асинхронный сервис, который подписывается на события из RabbitMQ (например, `order_paid`) и отвечает за отправку email-уведомлений клиентам.
*   **`Telegram Service` (Python, Aiogram):** Панель управления для администратора. Асинхронно получает уведомления о новых заказах и позволяет управлять ими (например, отметить заказ как "отправленный"), отправляя команды обратно в систему через RabbitMQ.
*   **`Database` (PostgreSQL):** Центральная база данных с полной изоляцией схем по принципу **Schema per Service** для предотвращения конфликтов и обеспечения независимости разработки.
*   **`Message Broker` (RabbitMQ):** "Нервная система" проекта. Реализована четкая топология:
    *   **`events_exchange` (fanout):** Для широковещательных событий (`order_paid`), на которые могут подписаться несколько сервисов.
    *   **`commands_exchange` (direct):** Для адресных команд (`ship_order`), которые должен выполнить один конкретный сервис.

## 5. Технологический стек

*   **Языки:** Go, Python, TypeScript, SQL
*   **Backend:** FastAPI, Aiogram
*   **Frontend:** React, Zustand, Swiper
*   **Базы данных и брокеры:** PostgreSQL, RabbitMQ
*   **Инфраструктура и DevOps:** Docker, Docker Compose, Nginx
*   **Библиотеки:** SQLAlchemy (с Alembic для миграций)
