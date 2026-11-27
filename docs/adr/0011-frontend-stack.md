# ADR-0011. Frontend: TypeScript 6, React Router SPA, big.js, типы из OpenAPI

**Статус:** предложено (Фаза 3, 27.11.2026) · **Журнал решений:** D-027

## Контекст

Предложенный стек: React 19, TypeScript, Vite, Tailwind, shadcn/ui, React Router, TanStack Query, React Hook Form, Zod, i18next, Recharts. Проверка npm 27.11.2026: TypeScript 7.0.2 вышел, но `typescript-eslint` 8.74 поддерживает TypeScript < 6.1; Vite 8.4.0 вышел 26.11.2026; React Router 8.5 требует React ≥ 19.2.7 и Node ≥ 22.22.

## Решение

TypeScript 6.0.x; Vite 8.3.x; React Router 8.5.x в режиме SPA (data router, без SSR); Node 24 LTS для сборки. Добавить `big.js` (деньги без float) и `openapi-typescript` + `openapi-fetch` (типы из OpenAPI). Глобальный менеджер состояния не нужен: серверное состояние — TanStack Query, касса — `useReducer` + `localStorage`.

## Рассмотренные альтернативы

- **TypeScript 7** — после поддержки в `typescript-eslint`.
- **React Router framework mode / Next.js** — SSR не нужен для внутреннего приложения за входом.
- **Zustand/Redux** — нет общего клиентского состояния, оправдывающего библиотеку.
- **Ручные TypeScript-типы DTO** — расходятся с сервером; отклонено.

## Последствия

Обновление до TypeScript 7 — отдельная задача после выхода совместимого `typescript-eslint`.
