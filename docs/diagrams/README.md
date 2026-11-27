# Диаграммы

Все диаграммы — Mermaid: GitHub отображает их прямо в Markdown. Синтаксис каждой проверен рендером `@mermaid-js/mermaid-cli` 11 (27.11.2026).

| Диаграмма | Файл |
|---|---|
| Контекст системы (C4, уровень 1) | [context.md](context.md) |
| Контейнеры (C4, уровень 2) | [container.md](container.md) |
| Компоненты backend: модули и зависимости (C4, уровень 3) | [component.md](component.md) |
| ER-диаграмма | [er.md](er.md) |
| Варианты использования | [use-cases.md](use-cases.md) |
| Развёртывание и CI/CD | [deployment.md](deployment.md) |
| Последовательность: аутентификация и обновление токена | [sequence-auth.md](sequence-auth.md) |
| Последовательность: приёмка поставки | [sequence-purchase-receiving.md](sequence-purchase-receiving.md) |
| Последовательность: продажа на кассе | [sequence-pos-sale.md](sequence-pos-sale.md) |
| Последовательность: распределение FEFO | [sequence-fefo.md](sequence-fefo.md) |

Нотация C4 нарисована через `flowchart`, а не экспериментальный `C4Context` Mermaid: так диаграммы стабильно отображаются на GitHub.
