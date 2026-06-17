---
name: unit-tests
description: >
  Padrão de testes unitários do Health Panel rodando em ambiente jsdom (Vitest). Cobre funções
  puras, classes utilitárias, schemas Zod e qualquer lógica sem React. Detalha mocks de pacotes externos,
  asserts assíncronos com `vi.waitFor` e fake timers.
  Componentes e hooks React estão fora deste escopo.
---

# Testes Unitários

Ambiente: **jsdom** (config único em [vite.config.ts](vite.config.ts), bloco `test`). O escopo aqui
é tudo que **não** seja componente ou hook React: funções puras, classes, schemas e services. Como o
ambiente é jsdom, essas unidades podem acessar APIs do browser (`localStorage`, `navigator.geolocation`,
`window.location`, …) — mocke ou configure essas APIs quando o código sob teste depender delas.

## Padrões por tipo de código

### Funções puras (normalizadores, formatadores, utils)

Chame e verifique o retorno. Sem `makeSut`, sem stubs.

```typescript
import { describe, expect, it } from 'vitest';

import { formatDocumentTitle } from './title';

describe('formatDocumentTitle', () => {
  it('falls back to the product name when the tenant has no label', () => {
    expect(formatDocumentTitle({ label: '' }, 'Health Panel')).toBe('Health Panel');
  });

  it('joins tenant label and product name', () => {
    expect(formatDocumentTitle({ label: 'DGS' }, 'Health Panel')).toBe('DGS - Health Panel');
  });
});
```

### Schemas Zod

O projeto usa Zod (com `@hookform/resolvers` em formulários). Teste aceite e rejeição com
`safeParse(...).success`. Cubra:

- payload mínimo válido;
- payload completo com campos opcionais;
- cada variante de discriminated union;
- rejeições explícitas (campo faltando, valor fora do enum, tipo errado).

```typescript
expect(mySchema.safeParse({ name: 'ok' }).success).toBe(true);
expect(mySchema.safeParse({ name: '' }).success).toBe(false);
```

### Classes e módulos com dependências

Para classes de valor com factories estáticas (ex.: uma classe de erro de API), chame a factory e
verifique o resultado:

```typescript
const error = ApiError.fromHttp(404, { message: 'not found' }, raw);

expect(error.status).toBe(404);
expect(error.message).toBe('not found');
```

Quando a unidade depende de outra interna, **passe a dependência por argumento/parâmetro** em vez
de mockar o módulo (ver `tests-best-practices`). `vi.mock` fica reservado à fronteira externa.

## Mocks de fronteira externa

A fronteira externa é o **único** lugar onde `vi.mock` é aceitável: pacotes de terceiros que fazem
I/O ou tocam no ambiente. Em vez de módulos `node:*` (que não existem no front), aqui a fronteira é:

### Camada HTTP (`axios` / `../http/api`)

Mocke o módulo `api` do projeto, não o `axios` diretamente, quando o SUT importa de `../http/api`:

```typescript
vi.mock('../http/api', () => ({
  api: {
    get: vi.fn(),
    post: vi.fn(),
    put: vi.fn(),
    delete: vi.fn(),
  },
}));
```

Para testar o próprio `api.ts` (interceptors, mapeamento de erro), mocke `axios`:

```typescript
vi.mock('axios', () => ({
  default: { create: vi.fn(() => ({ interceptors: { request: { use: vi.fn() }, response: { use: vi.fn() } } })) },
}));
```

### SDKs de autenticação

```typescript
vi.mock('react-oidc-context', () => ({
  useAuth: () => mockAuth,
  hasAuthParams: () => false,
}));
```

`oidc-client-ts` expõe a classe `User` real — geralmente é mais simples **instanciar `User`** com um
payload de teste (via uma factory `createUser` local ao arquivo de teste) do que mocká-la.

### APIs do browser

`window.matchMedia` já é provido por [src/test/setup.ts](src/test/setup.ts). `sessionStorage`/`localStorage`
são limpos no `afterEach` global. Para `window.location`/`history`, use
`window.history.replaceState({}, '', '/rota')` em vez de mockar.

APIs que o jsdom não implementa (ex.: `navigator.geolocation`) devem ser stubadas com
`vi.stubGlobal` (ou `Object.defineProperty`), restaurando no `afterEach`:

```typescript
afterEach(() => { vi.unstubAllGlobals(); });

it('resolves the current position', async () => {
  vi.stubGlobal('navigator', {
    geolocation: {
      getCurrentPosition: vi.fn((success) =>
        success({ coords: { latitude: -23.5, longitude: -46.6 } }),
      ),
    },
  });

  await expect(getCurrentCoords()).resolves.toEqual({ lat: -23.5, lng: -46.6 });
});
```

## Asserts assíncronos

Quando o teste dispara uma operação fire-and-forget e quer verificar o efeito, use `vi.waitFor`
(polling determinístico). **Nunca** use `setTimeout` direto. (Para fluxos que disparam `setState`
de React, use o `waitFor` do `@testing-library/react` — ver `hook-tests`/`component-tests`.)

```typescript
service.start();
await vi.waitFor(() => {
  expect(dependenciaMock.notify).toHaveBeenCalled();
});
```

## Fake timers

```typescript
beforeEach(() => { vi.useFakeTimers(); });
afterEach(() => { vi.useRealTimers(); });

it('should rotate after the interval', async () => {
  // ...
  await vi.advanceTimersByTimeAsync(1000);
  // ...
});
```

`advanceTimersByTimeAsync` (em vez de `advanceTimersByTime`) garante que microtasks/`setState`
agendados sejam aplicados antes de continuar.
