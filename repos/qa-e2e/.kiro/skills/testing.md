---
description: Convenciones de testing del proyecto — QA con Playwright, BDD y evaluación de agentes
---

# Skill: QA Testing

## Estructura de carpetas

```
tests/
  e2e/          → Tests E2E con Playwright (.spec.ts)
  features/     → Feature files Gherkin (.feature)
  bdd/          → Generado automáticamente por playwright-bdd (no editar)
pages/          → Page Objects (un archivo por página)
steps/          → Step definitions BDD
```

## Convenciones de naming

- Archivos de test: `<flujo>.spec.ts` (kebab-case)
- Page Objects: `<NombrePagina>Page.ts` (PascalCase)
- Step definitions: `<feature>.steps.ts`
- Feature files: `<flujo>.feature` (kebab-case, en español)

## Page Object Model (POM)

- Cada página tiene su propio archivo en `pages/`
- El constructor recibe `page: Page` y declara todos los Locators como `readonly`
- Prioridad de locators: `getByRole` > `getByLabel` > `getByText` > `data-testid` > CSS
- Los métodos encapsulan acciones (ej. `sendMessage`), nunca assertions
- Las assertions van en los tests o step definitions, no en los Page Objects

## Gherkin / BDD

- Scenarios escritos en español
- Given = estado inicial del sistema
- When = acción del usuario
- Then = resultado observable verificable
- Reusar steps existentes antes de crear nuevos
- Un scenario = una funcionalidad concreta
- Usar `Background` para precondiciones comunes a todos los scenarios del feature

## Playwright — Buenas prácticas

- Usar `waitFor` y `expect` de Playwright, nunca `setTimeout`
- Configuración de evidencia automática: `trace: 'on-first-retry'`, `screenshot: 'only-on-failure'`
- `baseURL` siempre desde `process.env.BASE_URL`
- Ejecutar con `--ui` en desarrollo, headless en CI

## Instrucciones para el coding agent

1. Antes de generar tests web, usar el MCP de Playwright para inspeccionar el DOM real y obtener locators precisos
2. Generar Page Objects primero, luego los tests que los consumen
3. Al agregar step definitions, buscar steps reutilizables antes de crear nuevos
4. Si un test falla, usar el trace viewer para diagnosticar antes de proponer fixes
5. Mantener cada Page Object enfocado en una sola página o componente
