<div align="center" style="font-size: 2em; font-weight: bold; text-decoration: none;">OneDash</div>
<div align="center" style="font-size: 1.5em; font-weight: bold; text-decoration: none;">Design Documentation</div>
<div align="center" style="font-size: 1.2em; font-weight: bold; text-decoration: none;">A Unified Dashboard Experience</div>

**Author:** Bishoy Michail

**Project Start:** Friday, 13th June 2025

**Version:** V1.0 (Initial Draft)

## Overview
OneDash is a modular dashboard platform designed to unify information and services across multiple domains into a single, cohesive user interface. This documentation outlines the system architecture, design principlies, service structure, and implementation decisions made during the development lifecycle.

The following will be discussed.

```
1. User Interface
2. API Gateway / Aggregator Layer
3. Microservice Ecosystem
4. Shared Services & Infrastructure
5. Data Storage & Persistence
6. Event & Messaging Backbone
7. Cross-Cutting Concerns (security, Logging, Monitoring, CI/CD)
```
<!-- 
![A diagrammatic stack to visualise how the system will interact on a high level](Design.png)
 -->

<br>

# 1. Frontend Application
## 1.1 Repo Design
- The entire dashboard is a single-page `React` app acting as a personal "Web OS" shell.
- UI Widgets are `lazy-loaded` via `React.lazy` and `Suspense`, minimising initial bundle size.
- Widgets only mount when they enter the viewport (using `Intersection Observer`) to conserve resources.
- Each widget encapsulates a microservice's view. Shows key metrics at a galnce, is clickable to expand into a full-screen detail view, and is wrapped in an `Error Boundary1 to isolate failures.

## 1.2 UI Design Pattern
### 1.2a UI Summary (Layout)
- **Theme**: Full screen pure black background (`#00000`) with neon green accents (`#00FF66`).
- **Top Status Bar**: Logo, command palette trigget (Ctrl + k), clock, notification bell, and user avatar.
- **Bottom Dock**: Glowing icons for each major service, hover highlights with neon glow; icons on the right side.
- **Main Workspace** A responsive CSS grid of draggable resizable widget windows, each with a title bar (icon, title, window controls) and content area.
- **Interactions**: Widgets animate in on load, support context menus (refresh, settings, close) and double click to maximise.
- **Themes**: Able to choose style dynamically including certain set themes.

### 1.2b Data Fetching & Realtime Updates
- **Data Ingestion Workers:**
  - **Scheduled:** cron-like jobs *(e.g. finance transactions daily at 23:59)*
  - **Event-Driven:** *(e.g. IMAP idle for new emails || Apple shortcut callbacks for health wake-up events)*
  - Workers persist data to their service's databases and emit domain events *(e.g. TransactionInserted)* to a message broker.
- **API Gateway (Aggregation Layer):**
  - Routes REST calls from the UI to appropriate microservice with JWT authentication.
  - Subscribes to the message broker to capture domain events in real-tiem.
- **UI Real-Time Flow:**
  - **Initial Snapshot:** Widget issues a one-time REST request for the current data snapshot.
  - **WebSocket Subscription**: Client opens a WS to the Gateway, subscribes to a channel *(e.g. financeSummary)* with its `lastSeenEventID`.
  - **Incremental Updates:** Gateway pushes only new events (with strictly increasing `eventID`) which merge into the client cache.
  - **Reliability:** WS keep-alives (ping/pong), exponential backoff reconnects, token refresh, and ack-based replay of missed events. On prolonged failure, widget falls back to low-frequency polling until WS recovers.

### 1.2c State Management
- **Unified Store with Redux Toolkit & RDK Query:**
  - A single `Redux Store` combines both `UI slices` (theme, layout, widget visibility) and `API slices` (domain data via `RTK Query`).
  - `createAPI` defines endpoints, and handles caching, invalidation, and fetching lifecycles.
  - A custom UI slice `uiSlice` manages theme toggles, layout config, and global UI state.
- **Store Configuration:** 
  ```ts
  import { configureStore } from '@reduxjs/toolkit';
  import { api } from './services/api';       // RTK Query apiSlice
  import uiReducer from './slices/uiSlice';   // UI slice for theme/layout
  
  export const store = configureStore({
    reducer: {
      ui: uiReducer,
      [api.reducerPath]: api.reducer,
    },
    middleware: (getDefault) => 
      getDefault().concat(api.middleware),
  });
  ```
- **Component Usage:**
```ts
 import { useSelector } from 'react-redux';
 import { useGetFinanceSummaryQuery } from './services/api';
 
 function FinanceWidget() {
   const theme = useSelector((state) => state.ui.theme);
   const { data, isLoading, error } = useGetFinanceSummaryQuery();
   // ... render chart based on data and theme ...
 }
```
  



