# Discovery Answers — lead-collector

## Objetivo do sistema

Aplicação SPA mobile-first para captação ágil de leads em eventos presenciais. Sem autenticação, sem servidor próprio, sem sincronização. Dados persistem exclusivamente no dispositivo do usuário e são compartilhados via WhatsApp.

## Tipo de projeto

Greenfield — scaffold inicial do Vite + React + TypeScript gerado, sem código de negócio implementado.

## Escopo

Projeto inteiro.

## Stack declarada

- React 19 + TypeScript
- Tailwind CSS
- Vite
- Vitest + React Testing Library (testes)
- Shadcn/ui (componentes de UI via CLI oficial)

## Restrições declaradas pelo usuário

| # | Restrição |
|---|-----------|
| 1 | TypeScript sempre. |
| 2 | Boas práticas de clean code. |
| 3 | Sem estados globais (sem Zustand, Redux, etc.). |
| 4 | Preferência por componentes Radix/shadcn; evitar HTML puro ou componentes customizados. |
| 5 | Testes de unidade e de componentes para 100% dos arquivos executáveis (exceção: arquivos de meta-configuração, templates e startup como `App.tsx`, `index.ts`). |
| 6 | Componentes shadcn instalados via CLI oficial (`npx shadcn@latest add`). |
| 7 | Vitest + React Testing Library para testes. |
| 8 | Arquitetura limpa com injeção de dependência por abstrações (interfaces) para evitar mocks em arquivos `.ts` do projeto. |
| 9 | Mocks permitidos apenas para APIs do DOM e bibliotecas de terceiros. |

## Decisões transversais

| Decisão | Escolha | Origem |
|---------|---------|--------|
| Estratégia de persistência local | **IndexedDB via biblioteca `idb`** | Resposta humana (gap `persistence-strategy`) |
| Documentação adicional | **ADRs** (Architecture Decision Records em Markdown, pasta `docs/adr/`) | Resposta humana (gap `additional-docs`) |

## Documentação adicional

ADRs (Architecture Decision Records) em Markdown. Registrar decisões arquiteturais relevantes na pasta `docs/adr/` do projeto.
