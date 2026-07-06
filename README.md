# Enterprise Multichannel ERP System
> Serverless Enterprise Resource Planning Platform Hosted on Google Apps Script and Powered by Google Sheets

[![Vite](https://img.shields.io/badge/Vite-8.1.3-646CFF?style=flat-square&logo=vite)](https://vite.dev/)
[![React](https://img.shields.io/badge/React-19.0.0-61DAFB?style=flat-square&logo=react)](https://react.dev/)
[![Google Apps Script](https://img.shields.io/badge/Google_Apps_Script-Serverless-34A853?style=flat-square&logo=google)](https://developers.google.com/apps-script)
[![Google Sheets](https://img.shields.io/badge/Google_Sheets-Database-34A853?style=flat-square&logo=googlesheets)](https://www.google.com/sheets/about/)
[![esbuild](https://img.shields.io/badge/esbuild-0.28.1-FFCF00?style=flat-square&logo=esbuild)](https://esbuild.github.io/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.4.5-3178C6?style=flat-square&logo=typescript)](https://www.typescriptlang.org/)
[![License](https://img.shields.io/badge/License-MIT-blue.svg?style=flat-square)](LICENSE)

---

## 1. Executive Summary

This Multichannel ERP System is a serverless enterprise resource planning platform engineered to bridge B2B e-commerce operational data with high-fidelity analytical reporting. Designed for modern multi-channel merchants, the platform consolidates transaction logs, product inventories, and financial ledgers from Tokopedia, Shopee, Facebook, Instagram, and Direct Stores into a single, cohesive source of truth.

By leveraging Google Apps Script (GAS) as a serverless execution environment and Google Sheets as an idempotent relational database, the system reduces total cost of ownership (TCO) while offering high-performance, real-time analytics, modular RBAC, and system-wide diagnostic monitoring.

---

## 2. Architecture & Data Flow Diagram

The application uses an optimized compilation pipeline that bundles a React Single Page Application (SPA) into a single HTML file deployed directly onto Google Apps Script. 

```mermaid
flowchart TD
    %% Global vertical direction
    direction TB

    subgraph Client_App["Client-Side Architecture (React Single Page Application)"]
        direction TB
        
        subgraph View_Layers["UI Component Tree (src/pages/)"]
            direction TB
            UI_Dash["Dashboard (Overview & KPIs)"]
            --> UI_Anal["Analytics (Local Drill-down)"]
            --> UI_Sales["Sales Orders (Management)"]
            --> UI_Prod["Products (Inventory)"]
            --> UI_Users["Team Access (RBAC UI)"]
            --> UI_Set["Settings (Common Meta Form)"]
            --> UI_Sup["Support (Support Coordinates)"]
        end

        subgraph State_Engine["State & Cache Layer (src/utils/store.js)"]
            direction TB
            Store["Subscription State Store"]
            --> LocalPref["Local Storage Preferences (sidebar, density)"]
            --> SessionUser["Session Storage (Active User Credentials)"]
            --> EntityCache["Performance Cache (Products, Orders, Ledger, Logs)"]
        end

        subgraph Client_Bridge["Client Bridge Layer (src/api/)"]
            direction TB
            GasClient["gasClient.js API Adapter"]
            --> MockStore["Mock In-Memory Store (Offline Development Mode)"]
        end
        
        UI_Console["Diagnostic Terminal (src/components/errorConsole.js)"]
    end

    subgraph Compiler_CI["CI/CD Compilation & Pipeline (Nusa-Compiler)"]
        direction TB
        ViteComp["Vite Bundler (vite.config.js)"]
        --> ViteSingle["vite-plugin-singlefile"]
        --> NusaComp["Nusa-Compiler (scripts/build-gas.mjs)"]
        --> ClaspPush["Google clasp (clasp push)"]
    end

    subgraph Server_Core["Google Apps Script Serverless Core (gas-src/)"]
        direction TB
        GetEndpoint["doGet(e) Entry Point"]
        --> Router["01-main.gs API Gateway Router"]
        --> AuthMid["02-auth.gs Authentication Middleware (RBAC Guards)"]
        
        subgraph Server_Modules["Database RPC Handlers (gas-src/modules/)"]
            direction TB
            Mod_Prod["10-products.gs (Inventory Handler)"]
            --> Mod_Sales["11-sales.gs (Orders/Archive Handlers)"]
            --> Mod_Ledger["12-ledger.gs (Bookkeeping reconciliation)"]
            --> Mod_Users["13-users.gs (Team management)"]
            --> Mod_Logs["14-logs.gs (Diagnostic audit logs)"]
        end
    end

    subgraph Sheet_DB["Relational Spreadsheet Database (Google Sheets)"]
        direction TB
        DbEngine["SpreadsheetApp Database Engine"]
        --> SetupSeed["00-setup.gs & 00-seeder.gs (Idempotent schema seeder)"]
        
        subgraph Sheet_Tables["Relational Google Sheets Tables"]
            direction TB
            T_Prod[("Products Table")]
            --> T_Orders[("SalesOrders Table")]
            --> T_Arch_Ord[("SalesOrders_Archive Table")]
            --> T_Ledger[("FinanceLedger Table")]
            --> T_Arch_Led[("FinanceLedger_Archive Table")]
            --> T_Users[("Users Table")]
            --> T_Logs[("SystemLogs Table")]
            --> T_Set[("Settings Table")]
        end
    end

    %% Client Relations
    View_Layers -->|Subscribe/Mutate| Store
    Store -->|Call API| GasClient
    Store -.->|Catch Errors| UI_Console

    %% Compilation Relations
    Client_Bridge -->|Bundle SPA| ViteComp
    webapp_html["dist/index.html (renamed to webapp.html)"] -->|Deploy| ClaspPush
    code_gs["dist-gas/code.gs"] -->|Deploy| ClaspPush

    %% Server Relations
    ClaspPush -->|Push to script| GetEndpoint
    GetEndpoint -->|Serve index.html| View_Layers
    
    GasClient -->|google.script.run JSON-RPC| Router
    AuthMid -->|Delegate requests| Server_Modules
    Server_Modules -->|Write/Read Query| DbEngine

    %% Database Relations
    DbEngine <-->|Products CRUD| T_Prod
```

### 2.1 Compilation & Deployment Workflow
1. **Frontend Compilation**: Vite compiles the React codebase. The plugin `vite-plugin-singlefile` bundles all Javascript, CSS stylesheets, and assets inline into `dist/index.html`.
2. **Backend Compilation**: The custom compiler script (`scripts/build-gas.mjs`) reads backend modules in `gas-src/` and concatenates them into `dist-gas/code.gs`, adding origin headers for diagnostic purposes.
3. **Synchronization**: Google Clasp synchronizes `dist-gas/` assets directly to the target Google Apps Script container.

---

## 3. Database Schema Specification

The Google Sheets database runs an idempotent relational schema. Each sheet represents a table with static headers enforced automatically at initialization.

### 3.1 Products Table
Defines active inventories, unit costs, and minimum thresholds.
* **Columns**: `id`, `sku`, `name`, `category`, `priceBuy`, `priceSell`, `stock`, `minStock`, `status`, `createdAt`, `updatedAt`, `createdBy`, `updatedBy`

### 3.2 SalesOrders Table
Consolidates transaction records across multi-channel integrations.
* **Columns**: `id`, `orderNumber`, `channel`, `customerName`, `totalAmount`, `paymentStatus`, `orderStatus`, `items` (JSON-string), `createdAt`, `updatedAt`, `createdBy`, `updatedBy`, `region`

### 3.3 FinanceLedger Table
Stores atomic bookkeeping logs for cash-flow ledger reconciliation.
* **Columns**: `id`, `transactionNumber`, `type` (Income/Expense), `category`, `amount`, `referenceId`, `description`, `transactionDate`, `createdAt`, `updatedAt`, `createdBy`, `updatedBy`

### 3.4 Users Table
Maintains team credentials, authorization status, and RBAC roles.
* **Columns**: `id`, `name`, `email`, `role` (Admin/Gudang/Sales/Finance/Viewer), `status`, `createdAt`, `updatedAt`, `createdBy`, `updatedBy`, `password` (SHA-256 hash)

### 3.5 Settings Table
Stores system configurations, invoice rules, metadata, and support coordinates.
* **Columns**: `id` (Primary Key), `value`, `updatedAt`, `updatedBy`

---

## 4. Platform Modules & Features

### 4.1 Executive Overview (Dashboard)
* **Real-time KPI Tracking**: Monitors Gross Merchandise Value (GMV), Total Orders, Low Stock Alerts, and Financial Balances.
* **Aspect-Ratio Optimized Sparklines**: Micro-charts display daily trajectories stretched responsively (`preserveAspectRatio="none"`) across all grids.
* **Granular Time-Scale Controls**: Toggles display scale (D/W/M/Q) locally without affecting adjacent widgets.

### 4.2 Local Drill-Down Analytics
* **Flexible Granularity**: Allows scaling analysis by Day (D), Week (W), Month (M), or Quarter (Q).
* **Multi-channel Revenue Share**: Interactively maps revenue contribution percentages with integrated ASC and DESC sorting toggles.
* **Bento Performance Indicators**: Automatically displays highest-performing channel, Average Order Value (AOV), and total orders in balanced info containers.

### 4.3 RBAC Security & Management
* **Role-Based Permissions**: Restricts or grants page accesses automatically based on roles:
  * **Admin**: Unrestricted read/write access to all settings, users, databases, and logs.
  * **Viewer/Gudang/Sales/Finance**: Scoped views matching functional operational responsibilities.

### 4.4 Diagnostic Console
* **Runtime Diagnostic Log**: Renders systemic background warnings and exception traces directly inside an integrated terminal UI.
* **Verification & Audit**: Logs administrative changes, data seeds, and transaction states automatically in the `SystemLogs` table.

---

## 5. Development & Deployment Guide

### 5.1 Local Development Environment
Ensure you have Node.js 18+ installed on your workspace.

1. **Install Dependencies**:
   ```bash
   npm install
   ```
2. **Start Vite Dev Server**:
   ```bash
   npm run dev
   ```
   *Runs local React environment with mock data bindings inside `src/api/gasClient.js`.*

### 5.2 Build & Bundle
Compile the client SPA and backend scripts into Google-compliant assets:
```bash
npm run build:all
```
This script compiles:
* `dist/index.html` (single-file output)
* `dist-gas/code.gs` (concatenated GAS backend code)
* Copies setup & seeder files to `dist-gas/`

### 5.3 Pushing to Google Apps Script
Ensure `@google/clasp` is authorized and bound to your target Google Spreadsheet script.

1. **Login to Google Account**:
   ```bash
   npx clasp login
   ```
2. **Synchronize Code**:
   ```bash
   npx clasp push -f
   ```

---

## 6. Installation & Database Seeding

After pushing the compiled assets to your script editor:
1. Open the linked Google Spreadsheet.
2. Select **Extensions** > **Apps Script**.
3. Select and execute the function **`runSetup()`** in `setup.gs` to create the database schemas and initialize default settings.
4. Select and execute the function **`runSeeder()`** in `seeder.gs` to populate the sheets with standard mock transaction data for analytics simulation.
5. Select **Deploy** > **New deployment** > **Web App** to serve the application link to users.

---
*Worksense Systems. Engineered for scalable multi-channel operational workflows.*
