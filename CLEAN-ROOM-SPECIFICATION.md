# Grocy — Clean-Room Specification

> **Version:** Based on Grocy 4.6.0 (2026-03-06)
>
> This document describes the functional behavior of Grocy — a self-hosted, web-based groceries and household management application — without reference to internal implementation details. It is intended to serve as a clean-room specification for independent reimplementation.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture & Platform Requirements](#2-architecture--platform-requirements)
3. [Authentication & Authorization](#3-authentication--authorization)
4. [Core Modules](#4-core-modules)
   - 4.1 [Stock Management](#41-stock-management)
   - 4.2 [Shopping Lists](#42-shopping-lists)
   - 4.3 [Recipes & Meal Planning](#43-recipes--meal-planning)
   - 4.4 [Chores](#44-chores)
   - 4.5 [Tasks](#45-tasks)
   - 4.6 [Batteries](#46-batteries)
   - 4.7 [Equipment](#47-equipment)
   - 4.8 [Calendar](#48-calendar)
5. [Generic Entity System (Userfields & User Entities)](#5-generic-entity-system)
6. [Data Model](#6-data-model)
7. [REST API](#7-rest-api)
8. [Barcode Support](#8-barcode-support)
9. [Label & Thermal Printing](#9-label--thermal-printing)
10. [Localization](#10-localization)
11. [Configuration](#11-configuration)
12. [Feature Flags](#12-feature-flags)

---

## 1. Overview

Grocy is described as "ERP beyond your fridge." It is a web-based, self-hosted household management solution that tracks:

- **Groceries / stock** — what you have, where it is, when it expires, what it costs
- **Shopping lists** — what you need to buy
- **Recipes** — ingredients, fulfillment status relative to current stock, meal planning
- **Chores** — recurring household tasks with scheduling and user assignment
- **Tasks** — one-off to-do items with due dates and categories
- **Batteries** — charge cycle tracking for rechargeable batteries/devices
- **Equipment** — a catalog of household equipment with manuals/documentation
- **Calendar** — a unified calendar view aggregating due dates across all modules

All data is stored in a single SQLite database. The application is designed for single-household use but supports multiple user accounts with granular permissions.

---

## 2. Architecture & Platform Requirements

### Runtime

- **Server-side:** PHP 8.5+ with extensions: `fileinfo`, `pdo_sqlite`, `gd`, `ctype`, `intl`, `zlib`, `mbstring`
- **Database:** SQLite 3.40+
- **Client-side:** Modern browser (Firefox, Chrome, Edge)

### Application Structure

- **Web framework:** Slim (PHP micro-framework) with PSR-7 request/response
- **Templating:** Blade templates
- **ORM:** LessQL (lightweight SQL query builder)
- **Frontend:** Server-rendered HTML with JavaScript enhancements
- **Database migrations:** Numbered SQL and PHP migration files executed sequentially

### Deployment

- Unpack release into a web-accessible directory
- Webserver document root points to `public/`
- Configuration file at `data/config.php` (copied from `config-dist.php`)
- The `data/` directory must be writable (contains the SQLite database, user uploads, and config)
- URL rewriting recommended but optional (configurable)
- Also available as a Docker container and a Windows desktop application

---

## 3. Authentication & Authorization

### Authentication Methods

The system supports pluggable authentication via middleware:

1. **Default Auth (session-based):** Username/password login with session cookies. Sessions expire after 30 days by default, with an option to stay logged in permanently. Passwords are hashed with Argon2id.
2. **Reverse Proxy Auth:** Trusts an HTTP header (e.g., `REMOTE_USER`) set by a reverse proxy.
3. **LDAP Auth:** Authenticates against an LDAP/Active Directory server.
4. **API Key Auth:** For API access, users can create API keys that are passed via an `GROCY-API-KEY` HTTP header.
5. **Disabled Auth:** Authentication can be fully disabled (all requests use the default/first user).

In non-production modes (`dev`, `demo`, `prerelease`), authentication is automatically disabled.

### Session Management

- Sessions are stored in the database with a random 50-character key
- Session cookie name: `grocy_session`
- Session validity is checked on every request; last-used timestamp is updated without changing the database modification time

### Authorization (Permissions)

The system uses a hierarchical permission model. Each user has a set of permissions. Available permissions:

| Permission | Description |
|---|---|
| `ADMIN` | Full access to everything |
| `USERS`, `USERS_CREATE`, `USERS_EDIT`, `USERS_READ`, `USERS_EDIT_SELF` | User management |
| `STOCK`, `STOCK_PURCHASE`, `STOCK_CONSUME`, `STOCK_TRANSFER`, `STOCK_INVENTORY`, `STOCK_OPEN`, `STOCK_EDIT` | Stock operations |
| `SHOPPINGLIST`, `SHOPPINGLIST_ITEMS_ADD`, `SHOPPINGLIST_ITEMS_DELETE` | Shopping list operations |
| `RECIPES`, `RECIPES_MEALPLAN` | Recipe and meal plan access |
| `CHORES`, `CHORE_TRACK_EXECUTION`, `CHORE_UNDO_EXECUTION` | Chore operations |
| `TASKS`, `TASKS_MARK_COMPLETED`, `TASKS_UNDO_EXECUTION` | Task operations |
| `BATTERIES`, `BATTERIES_TRACK_CHARGE_CYCLE`, `BATTERIES_UNDO_CHARGE_CYCLE` | Battery operations |
| `EQUIPMENT` | Equipment catalog access |
| `CALENDAR` | Calendar access |
| `MASTER_DATA_EDIT` | Edit master data (products, locations, etc.) |

Permissions are resolved hierarchically (e.g., `ADMIN` implies all other permissions; `STOCK` implies all `STOCK_*` sub-permissions). New users receive configurable default permissions (default: `ADMIN`).

---

## 4. Core Modules

### 4.1 Stock Management

Stock management is the central module. It tracks physical inventory of products.

#### Master Data

- **Products:** Each product has a name (unique), description, default location, purchase quantity unit, stock quantity unit, a conversion factor between them, barcode(s), minimum stock amount, default best-before days, and various behavioral flags.
- **Locations:** Named storage locations (e.g., "Fridge", "Pantry"). Products have a default location, but individual stock entries can be in different locations.
- **Quantity Units:** Named units (e.g., "Piece", "Bottle", "Pack") with support for plural forms. Quantity unit conversions can be defined globally or per-product.
- **Product Groups:** Categories for organizing products (e.g., "Dairy", "Vegetables").
- **Product Barcodes:** Multiple barcodes can be associated with a single product, each optionally tied to a specific quantity unit and amount.
- **Shopping Locations (Stores):** Named stores for price tracking.

#### Stock Entries

Each unit of stock is tracked as a **stock entry** with:
- Product reference
- Amount (decimal)
- Best-before date (optional)
- Purchased date
- Price (optional)
- Location
- Shopping location (store where purchased)
- Stock ID (groups entries from the same purchase transaction)
- Open/unopened status
- A note field

#### Stock Transactions

All stock changes are recorded as **stock log** entries. Transaction types:

| Type | Description |
|---|---|
| `purchase` | Adding stock via purchase |
| `consume` | Removing stock via consumption |
| `inventory-correction` | Adjusting stock via inventory count |
| `transfer_from` / `transfer_to` | Moving stock between locations |
| `product-opened` | Marking a stock entry as opened |
| `self-production` | Stock added via recipe consumption (self-produced) |
| `stock-edit-new` / `stock-edit-old` | Direct edits to stock entries |

Each transaction is associated with a `transaction_id` that groups related log entries (e.g., a transfer creates both a `transfer_from` and `transfer_to` entry with the same `transaction_id`). Transactions can be **undone**, which reverses the stock change.

#### Stock Operations

- **Purchase:** Add stock for a product with amount, best-before date, price, location, and store. Supports lookup by barcode.
- **Consume:** Remove stock for a product. Configurable to consume from specific stock entries or automatically by FIFO (first expiring first). Supports "spoiled" flag. Can mark as opened instead of consuming.
- **Transfer:** Move stock entries between locations.
- **Inventory:** Set the absolute stock amount for a product. The system calculates the difference and creates appropriate purchase or consume log entries.
- **Open:** Mark a stock entry as opened (relevant for products where opened items have different shelf life).
- **Merge Products:** Merge all stock, history, and references from one product into another.

#### Stock Views

- **Stock Overview:** Shows all products currently in stock or below minimum stock. Displays current amount, best-before date, value, and status indicators (expired, expiring soon, below minimum).
- **Stock Entries:** Detailed view of individual stock entries.
- **Stock Journal:** Chronological log of all stock transactions.
- **Stock Journal Summary:** Aggregated summary of stock movements.
- **Location Content Sheet:** Printable inventory by location.
- **Volatile Stock:** Returns products that are expiring soon, already expired, or below minimum stock.

#### Price Tracking

When enabled, the system tracks purchase prices per stock entry and per store. Provides price history for products and spending reports.

#### External Barcode Lookup

Products can be looked up by barcode via an external plugin system. A built-in plugin queries Open Food Facts. Custom plugins can be placed in `data/plugins/`.

### 4.2 Shopping Lists

- Multiple named shopping lists (when the multi-list feature is enabled)
- Each item references a product (optional), has an amount, quantity unit, and a note
- Automatic population from: missing products (below min stock), overdue products, expired products, or unfulfilled recipe ingredients
- Items can be checked off (done flag)
- Lists can be cleared (all items or only done items)
- Products can be added/removed programmatically
- Shopping list can be printed (browser print or thermal printer)
- Optional auto-add: products automatically added to a configured shopping list when they fall below minimum stock

### 4.3 Recipes & Meal Planning

#### Recipes

- Each recipe has a name, description, servings count, picture, and ingredient list
- Recipe ingredients reference products with amounts and quantity units
- Ingredients can be marked as: "only check if any amount is in stock", "do not check stock fulfillment", or "variable amount"
- Ingredient groups allow visual grouping
- Recipes can include other recipes (nesting)
- **Fulfillment status:** The system calculates whether current stock satisfies a recipe's ingredients, showing which ingredients are missing and by how much
- **Consume recipe:** Deducts all ingredient amounts from stock
- **Add missing to shopping list:** Adds unfulfilled ingredients to a shopping list
- Recipes can be copied/duplicated
- Recipe types: `normal` (user-created), plus internal types for meal plan aggregation

#### Meal Planning

- A calendar-based meal plan where recipes or products are assigned to specific dates and meal sections
- Meal plan sections are user-defined (e.g., "Breakfast", "Lunch", "Dinner")
- Each section has a sort number and optional time range
- The meal plan can display per-day or per-week views
- Meal plan entries automatically create shadow recipes for per-entry stock fulfillment checking

### 4.4 Chores

Recurring household tasks with flexible scheduling.

#### Chore Properties

- Name, description, active flag
- **Period type:** `manually`, `hourly`, `daily`, `weekly`, `monthly`, `yearly`, or `adaptive` (learns from past execution intervals)
- Period interval (e.g., every N days)
- Period config for weekly (specific days) and monthly (specific day of month)
- Track date only vs. date+time
- Rollover behavior: whether to reschedule from the last done date or the original schedule
- **Assignment type:** `no-assignment`, `random`, `in-alphabetical-order`, or `who-least-did-first`
- Assignment config: which users are in the rotation
- Consumable flag and product link (optionally consume a product when the chore is executed)

#### Chore Operations

- **Track execution:** Record that a chore was done (optionally by a specific user, at a specific time)
- **Undo execution:** Reverse a tracked execution
- **Calculate next assignments:** Recalculate who should do each chore next
- **Chores overview:** Shows all active chores with their next estimated execution time and assigned user
- **Chores journal:** Chronological log of all chore executions
- **Merge chores:** Combine two chores into one, preserving history

### 4.5 Tasks

Simple one-off to-do items.

- Name, description, due date, done flag, done timestamp
- Assigned to a user (optional)
- Category (from user-defined task categories)
- **Operations:** Mark as completed, undo completion
- **Current tasks view:** Shows all incomplete tasks, or all tasks

### 4.6 Batteries

Tracks charge cycles for rechargeable batteries and devices.

- Battery name, description, active flag
- Used in (free-text description of where the battery is used)
- Charge interval days (for estimating next charge time)
- **Track charge cycle:** Record a charge event with timestamp
- **Undo charge cycle:** Reverse a tracked charge
- **Batteries overview:** Shows all active batteries with last charged time and next estimated charge time
- **Batteries journal:** Chronological log of all charge cycles

### 4.7 Equipment

A simple catalog of household equipment.

- Name, description, active flag
- Instruction manual (stored as a file attachment, rendered as Markdown)
- No tracking or journaling — purely informational

### 4.8 Calendar

A unified calendar view that aggregates events from all modules:

- Product due dates (best-before dates)
- Chore schedules
- Task due dates
- Battery charge schedules
- Meal plan entries

Supports iCal export via a shareable URL for integration with external calendar applications.

---

## 5. Generic Entity System

Grocy provides an extensibility mechanism via user-defined entities and fields.

### User Entities

Users can create custom entities (tables) with custom fields. Each user entity gets:
- A database table for storing objects
- CRUD UI pages
- Full REST API access via the generic `/api/objects/{entity}` endpoints

### Userfields

Additional custom fields can be added to any built-in entity (products, chores, recipes, etc.) or user entity. Each userfield defines:
- Entity it belongs to
- Field name and caption
- Data type (text, number, date, etc.)
- Configuration (e.g., select options)
- Show on specific forms/tables

Userfield values are stored separately and joined at query time.

---

## 6. Data Model

### Core Tables

| Table | Purpose |
|---|---|
| `products` | Product master data |
| `locations` | Storage locations |
| `quantity_units` | Units of measurement |
| `quantity_unit_conversions` | Unit conversion factors (global and per-product) |
| `product_groups` | Product categorization |
| `product_barcodes` | Barcode-to-product mappings |
| `shopping_locations` | Store/shop definitions |
| `stock` | Current stock entries |
| `stock_log` | Stock transaction journal |
| `shopping_list` | Shopping list items |
| `recipes` | Recipe definitions |
| `recipes_pos` | Recipe ingredients |
| `recipes_nestings` | Recipe-includes-recipe relationships |
| `meal_plan` | Meal plan entries |
| `meal_plan_sections` | Meal plan section definitions |
| `chores` | Chore definitions |
| `chores_log` | Chore execution journal |
| `batteries` | Battery definitions |
| `battery_charge_cycles` | Battery charge journal |
| `tasks` | Task items |
| `task_categories` | Task categorization |
| `equipment` | Equipment catalog |
| `users` | User accounts |
| `sessions` | Active sessions |
| `api_keys` | API keys |
| `user_permissions` | User-to-permission mappings |
| `userentities` | User-defined entity definitions |
| `userfields` | User-defined field definitions |
| `userobjects` | User-defined entity data |
| `userfield_values` | User-defined field values |
| `migrations` | Database migration tracking |

### Key Views

| View | Purpose |
|---|---|
| `stock_current` | Aggregated current stock per product |
| `stock_missing_products` | Products below minimum stock amount |
| `batteries_current` | Current battery status with next charge estimates |
| `chores_current` | Current chore status with next execution estimates |
| `tasks_current` | Active (incomplete) tasks |
| `recipes_pos_resolved` | Recipe ingredients with fulfillment status |
| `quantity_unit_conversions_resolved` | Fully resolved unit conversions |
| `user_permissions_resolved` | Flattened user permissions (including hierarchy) |

### Conventions

- All tables have an auto-increment integer `id` primary key
- All tables have a `row_created_timestamp` column (defaults to current local datetime)
- Most master data tables have an `active` flag for soft-delete
- Journal/log tables have an `undone` flag and `undone_timestamp` for undo support
- Date fields use `DATE` type (YYYY-MM-DD), datetime fields use `DATETIME` type
- Amounts are stored as `REAL` (decimal) values

---

## 7. REST API

The API is available under the `/api` prefix and uses JSON for request/response bodies.

### Authentication

API requests are authenticated via one of:
- Session cookie (`grocy_session`)
- `GROCY-API-KEY` HTTP header

### OpenAPI Specification

A full OpenAPI (Swagger) specification is available at `/api/openapi/specification` and can be browsed via the built-in Swagger UI at `/api`.

### Endpoint Categories

#### System
- `GET /api/system/info` — Version, OS, database integrity info
- `GET /api/system/time` — Current server time
- `GET /api/system/db-changed-time` — Last database modification time (for cache invalidation)
- `GET /api/system/config` — Public configuration values
- `GET /api/system/localization-strings` — Current locale's translation strings

#### Generic CRUD
- `GET /api/objects/{entity}` — List all objects (supports query filters)
- `GET /api/objects/{entity}/{id}` — Get single object
- `POST /api/objects/{entity}` — Create object
- `PUT /api/objects/{entity}/{id}` — Update object
- `DELETE /api/objects/{entity}/{id}` — Delete object
- `GET/PUT /api/userfields/{entity}/{id}` — Get/set userfield values

#### Stock
- `GET /api/stock` — Current stock (all products)
- `GET /api/stock/volatile` — Expiring, expired, and below-minimum products
- `GET /api/stock/products/{id}` — Product details with stock info
- `POST /api/stock/products/{id}/add` — Purchase/add stock
- `POST /api/stock/products/{id}/consume` — Consume stock
- `POST /api/stock/products/{id}/transfer` — Transfer stock
- `POST /api/stock/products/{id}/inventory` — Inventory correction
- `POST /api/stock/products/{id}/open` — Mark stock as opened
- `POST /api/stock/products/{idKeep}/merge/{idRemove}` — Merge products
- `GET /api/stock/products/by-barcode/{barcode}` — Lookup by barcode (+ add/consume/transfer/inventory/open variants)
- `GET/PUT /api/stock/entry/{id}` — Get/edit individual stock entries
- `GET /api/stock/bookings/{id}` — Get booking details
- `POST /api/stock/bookings/{id}/undo` — Undo a booking
- `GET /api/stock/transactions/{id}` — Get transaction details
- `POST /api/stock/transactions/{id}/undo` — Undo a transaction
- `GET /api/stock/barcodes/external-lookup/{barcode}` — External barcode lookup
- `GET /api/stock/locations/{id}/entries` — Stock entries at a location
- `GET /api/stock/products/{id}/locations` — Product stock by location
- `GET /api/stock/products/{id}/entries` — Product stock entries
- `GET /api/stock/products/{id}/price-history` — Product price history

#### Shopping List
- `POST /api/stock/shoppinglist/add-missing-products` — Add all below-minimum products
- `POST /api/stock/shoppinglist/add-overdue-products` — Add all overdue products
- `POST /api/stock/shoppinglist/add-expired-products` — Add all expired products
- `POST /api/stock/shoppinglist/clear` — Clear a shopping list
- `POST /api/stock/shoppinglist/add-product` — Add a product
- `POST /api/stock/shoppinglist/remove-product` — Remove a product

#### Recipes
- `GET /api/recipes/{id}/fulfillment` — Recipe fulfillment status
- `GET /api/recipes/fulfillment` — All recipes fulfillment
- `POST /api/recipes/{id}/consume` — Consume a recipe's ingredients from stock
- `POST /api/recipes/{id}/add-not-fulfilled-products-to-shoppinglist` — Add missing ingredients to shopping list
- `POST /api/recipes/{id}/copy` — Duplicate a recipe

#### Chores
- `GET /api/chores` — Current chore statuses
- `GET /api/chores/{id}` — Chore details
- `POST /api/chores/{id}/execute` — Track chore execution
- `POST /api/chores/executions/{id}/undo` — Undo execution
- `POST /api/chores/executions/calculate-next-assignments` — Recalculate assignments
- `POST /api/chores/{idKeep}/merge/{idRemove}` — Merge chores

#### Batteries
- `GET /api/batteries` — Current battery statuses
- `GET /api/batteries/{id}` — Battery details
- `POST /api/batteries/{id}/charge` — Track charge cycle
- `POST /api/batteries/charge-cycles/{id}/undo` — Undo charge cycle

#### Tasks
- `GET /api/tasks` — Current tasks
- `POST /api/tasks/{id}/complete` — Mark task complete
- `POST /api/tasks/{id}/undo` — Undo task completion

#### Users
- `GET/POST /api/users` — List/create users
- `PUT/DELETE /api/users/{id}` — Edit/delete user
- `GET/POST/PUT /api/users/{id}/permissions` — Manage user permissions
- `GET /api/user` — Current user info
- `GET/PUT/DELETE /api/user/settings/{key}` — User settings

#### Files
- `PUT /api/files/{group}/{fileName}` — Upload file
- `GET /api/files/{group}/{fileName}` — Download file
- `DELETE /api/files/{group}/{fileName}` — Delete file

#### Calendar
- `GET /api/calendar/ical` — iCal feed
- `GET /api/calendar/ical/sharing-link` — Get/generate iCal sharing URL

#### Printing
- `GET /api/print/shoppinglist/thermal` — Print shopping list to thermal printer
- Various `*/printlabel` endpoints for label printing

### CORS

The API handles CORS preflight `OPTIONS` requests and returns `204 No Content`.

---

## 8. Barcode Support

### Grocycode

Grocy has its own barcode format called **Grocycode** for internal identification:
- Configurable format: 1D (Code128) or 2D (DataMatrix)
- Encodes entity type and ID (e.g., product, stock entry, chore, battery, recipe)
- Used for printing labels that can be scanned to quickly interact with entities

### Product Barcodes

- Products can have multiple associated barcodes
- Each barcode can optionally specify a quantity unit and amount (e.g., a "6-pack" barcode)
- Barcode lookup is used throughout the stock operations (purchase, consume, transfer, etc.)

### External Barcode Lookup

- Plugin-based system for looking up product information from external databases
- Built-in plugin: Open Food Facts
- Custom plugins can be added in `data/plugins/`

---

## 9. Label & Thermal Printing

### Label Printer

- Triggered via a configurable webhook URL
- Can run server-side or client-side
- Supports custom parameters (e.g., font family)
- Available for: products, stock entries, chores, batteries, recipes

### Thermal Printer

- Supports ESC/POS protocol receipt printers
- Can connect via network (IP/port) or local USB/serial
- Used primarily for shopping list printing
- Configurable: print quantity names, print notes

---

## 10. Localization

- Default language: English (embedded in code)
- Additional languages via translation files in `localization/` directory
- Translations managed via Transifex
- Languages at 70%+ completion are included in releases
- Per-user language override via user settings
- Quantity unit plural forms are supported (varies by locale rules)
- Missing localization strings can be reported to the server for tracking
- RTL languages are not supported

---

## 11. Configuration

Configuration is managed via `data/config.php` with three priority levels:

1. **Override files:** `.txt` files in `data/settingoverrides/` (highest priority)
2. **Environment variables:** Prefixed with `GROCY_` (e.g., `GROCY_BASE_URL`)
3. **Config file:** `data/config.php` using `Setting()` calls

### Key Settings

| Setting | Default | Description |
|---|---|---|
| `MODE` | `production` | `production`, `dev`, `demo`, or `prerelease` |
| `DEFAULT_LOCALE` | `en` | Default language |
| `CURRENCY` | `USD` | ISO 4217 currency code for display |
| `ENERGY_UNIT` | `kcal` | Energy unit label |
| `BASE_URL` | `/` | Base URL for the application |
| `BASE_PATH` | `` | URL path prefix when in a subdirectory |
| `ENTRY_PAGE` | `stock` | Default homepage after login |
| `DISABLE_AUTH` | `false` | Disable authentication entirely |
| `AUTH_CLASS` | `DefaultAuthMiddleware` | Authentication middleware class |
| `STOCK_BARCODE_LOOKUP_PLUGIN` | `OpenFoodFactsBarcodeLookupPlugin` | External barcode lookup plugin |
| `GROCYCODE_TYPE` | `2D` | Barcode format (`1D` = Code128, `2D` = DataMatrix) |
| `DISABLE_URL_REWRITING` | `false` | For servers without URL rewrite support |

### User Settings

Per-user settings are stored in the database and configurable via the UI. They cover:
- Night mode / dark theme preferences
- Stock display options (decimal places, due-soon days, default amounts)
- Shopping list behavior (auto-add, calendar display, rounding)
- Recipe display preferences
- Chore/battery/task due-soon thresholds
- Calendar event colors
- Screen behavior (keep-on, auto-reload)

---

## 12. Feature Flags

Individual modules and sub-features can be enabled/disabled:

### Module Flags
| Flag | Default |
|---|---|
| `FEATURE_FLAG_STOCK` | `true` |
| `FEATURE_FLAG_SHOPPINGLIST` | `true` |
| `FEATURE_FLAG_RECIPES` | `true` |
| `FEATURE_FLAG_CHORES` | `true` |
| `FEATURE_FLAG_TASKS` | `true` |
| `FEATURE_FLAG_BATTERIES` | `true` |
| `FEATURE_FLAG_EQUIPMENT` | `true` |
| `FEATURE_FLAG_CALENDAR` | `true` |
| `FEATURE_FLAG_LABEL_PRINTER` | `false` |

### Sub-Feature Flags
| Flag | Default |
|---|---|
| `FEATURE_FLAG_STOCK_PRICE_TRACKING` | `true` |
| `FEATURE_FLAG_STOCK_LOCATION_TRACKING` | `true` |
| `FEATURE_FLAG_STOCK_BEST_BEFORE_DATE_TRACKING` | `true` |
| `FEATURE_FLAG_STOCK_PRODUCT_OPENED_TRACKING` | `true` |
| `FEATURE_FLAG_STOCK_PRODUCT_FREEZING` | `true` |
| `FEATURE_FLAG_STOCK_BEST_BEFORE_DATE_FIELD_NUMBER_PAD` | `true` |
| `FEATURE_FLAG_SHOPPINGLIST_MULTIPLE_LISTS` | `true` |
| `FEATURE_FLAG_RECIPES_MEALPLAN` | `true` |
| `FEATURE_FLAG_CHORES_ASSIGNMENTS` | `true` |
| `FEATURE_FLAG_THERMAL_PRINTER` | `false` |
| `FEATURE_FLAG_DISABLE_BROWSER_BARCODE_CAMERA_SCANNING` | `false` |
| `FEATURE_FLAG_AUTO_TORCH_ON_WITH_CAMERA` | `true` |

---

*This specification is based on analysis of Grocy v4.6.0 and describes observable behavior and public interfaces. It is intended for clean-room reimplementation purposes.*
