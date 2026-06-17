---
name: http-service-patterns
description: >
  Define como serviços HTTP devem ser criados em `src/http/`.
  Use ao criar um novo service, adicionar endpoints a um service existente,
  definir tipos de request/response, ou decidir sobre tratamento de erros,
  constantes de domínio e estrutura de funções.
---

# Padrão de HTTP Services

## Estrutura e localização

Arquivos ficam em `src/http/<domínio>Service.ts`.

Um arquivo por domínio. O service agrupa todos os endpoints de um recurso e retorna dados tipados — sem lógica de negócio, sem tratamento de erros, sem conhecimento de estado de UI.

A camada HTTP compartilhada já existe e **não deve ser recriada**:

- `src/http/api.ts` — instância axios com interceptor de autenticação e timeout configurado
- `src/http/ApiError.ts` — classe `ApiError` com factory methods (`fromHttp`, `network`, `unknown`)
- `src/http/errorMapper.ts` — função `mapApiErrorToMessage` para exibição de erros na UI

## Funções e exports

Exportar funções nomeadas. Nunca usar classe, namespace ou default export.

```ts
export async function listAreas(): Promise<AreaResponse[]> {
  const response = await api.get<AreaResponse[]>('/areas');
  return response.data;
}

export async function createArea(payload: CreateAreaPayload): Promise<AreaResponse> {
  const response = await api.post<AreaResponse>('/areas', payload);
  return response.data;
}

export async function updateArea(areaId: string, payload: UpdateAreaPayload): Promise<AreaResponse> {
  const response = await api.put<AreaResponse>(`/areas/${areaId}`, payload);
  return response.data;
}

export async function deleteArea(areaId: string): Promise<void> {
  await api.delete(`/areas/${areaId}`);
}
```

## Tipos

Declarar todos os tipos no mesmo arquivo e exportá-los. Usar `import type` nos consumidores.

Sufixos obrigatórios:

- `<Domínio>Response` — shape retornado pela API
- `Create<Domínio>Payload` — body de criação
- `Update<Domínio>Payload` — body de atualização (geralmente `Partial<Create<Domínio>Payload>`)

```ts
export type AreaResponse = {
  id: string;
  name: string;
  seniorityRates: SeniorityRates;
};

export type CreateAreaPayload = {
  name: string;
  seniorityRates: SeniorityRates;
};

export type UpdateAreaPayload = Partial<CreateAreaPayload>;
```

## Constantes de domínio

Quando a API usa valores string fixos (enums de backend), representar como objeto `as const` com sufixo `_KEYS`. Exportar também o tipo derivado e, quando necessário, um mapa de labels para exibição.

```ts
export const SENIORITY_KEYS = {
  junior: 'junior',
  mid: 'mid',
  senior: 'senior',
  lead: 'lead',
} as const;

export type SeniorityKey = (typeof SENIORITY_KEYS)[keyof typeof SENIORITY_KEYS];

export const SENIORITY_LABELS: Record<SeniorityKey, string> = {
  junior: 'Júnior',
  mid: 'Pleno',
  senior: 'Sênior',
  lead: 'Lead',
};
```

## Autenticação

O interceptor em `api.ts` já injeta o Bearer token em todas as requisições. O service não deve saber nada sobre autenticação.

## Tratamento de erros

O service não captura erros. Deixar propagar para o chamador (página ou hook), que decide como exibir o erro usando `mapApiErrorToMessage`.

## Testes de services HTTP

Ao testar um service em `src/http/`, prefira `msw` para exercitar a chamada HTTP pela borda real do módulo:

- intercepte a URL final do endpoint com `http.get/post/put/delete(...)`;
- deixe o service usar a instância real `api` para validar `baseURL`, query params, body e headers transversais;
- faça asserções a partir do `request.url`, `request.headers` e do payload retornado pelo handler;
- limpe o estado de autenticação/cache entre testes quando o interceptor depender do usuário autenticado.

Agrupe os testes em sub-`describe` por função exportada: um `describe('<service>')` no topo e, dentro dele, um `describe('<função>')` para cada função do service (ex.: `listAreas`, `createArea`). Os `it` ficam dentro do `describe` da função correspondente. Hooks compartilhados (`beforeEach` de autenticação/cache) permanecem no `describe` de topo para valer em todas as funções.

Esse padrão é preferível a mockar `api.get`/`api.post`, porque preserva o contrato observado pelo service sem acoplar o teste à implementação interna do Axios.

`msw` deve ficar concentrado em testes de integração com backend (services, páginas/hooks que consomem services). Não use `msw` para validar apenas configuração de rotas.

## Exemplo completo

```ts
import { api } from './api';

export type SeniorityRates = {
  junior: number;
  mid: number;
  senior: number;
  lead: number;
};

export type AreaResponse = {
  id: string;
  name: string;
  seniorityRates: SeniorityRates;
};

export type CreateAreaPayload = {
  name: string;
  seniorityRates: SeniorityRates;
};

export type UpdateAreaPayload = Partial<CreateAreaPayload>;

export const SENIORITY_KEYS = {
  junior: 'junior',
  mid: 'mid',
  senior: 'senior',
  lead: 'lead',
} as const;

export type SeniorityKey = (typeof SENIORITY_KEYS)[keyof typeof SENIORITY_KEYS];

export const SENIORITY_LABELS: Record<SeniorityKey, string> = {
  junior: 'Júnior',
  mid: 'Pleno',
  senior: 'Sênior',
  lead: 'Lead',
};

export async function listAreas(): Promise<AreaResponse[]> {
  const response = await api.get<AreaResponse[]>('/areas');
  return response.data;
}

export async function createArea(payload: CreateAreaPayload): Promise<AreaResponse> {
  const response = await api.post<AreaResponse>('/areas', payload);
  return response.data;
}

export async function updateArea(areaId: string, payload: UpdateAreaPayload): Promise<AreaResponse> {
  const response = await api.put<AreaResponse>(`/areas/${areaId}`, payload);
  return response.data;
}

export async function deleteArea(areaId: string): Promise<void> {
  await api.delete(`/areas/${areaId}`);
}
```

## Exemplo de teste

Use o próprio resolver do `msw` como um spy (`vi.fn`). Isso evita variáveis mutáveis (`let requestUrl`) e helpers de `null`: o `expect(resolver).toHaveBeenCalledOnce()` já garante que a requisição aconteceu antes de ler `mock.calls`, de onde se obtém o `request` completo (URL, query params, headers e body).

```ts
import { http, HttpResponse, type HttpResponseResolver } from 'msw';
import { beforeEach, describe, expect, it, vi } from 'vitest';

import { listAreas } from './areasService';

import { environment } from '@/environment';
import { server } from '@/test/msw/server';

const AREAS_URL = `${environment.apiBaseUrl}/areas`;

function mockAreasCatalog(response: Record<string, unknown>[] = []) {
  const resolver = vi.fn<HttpResponseResolver>(() => HttpResponse.json(response));

  server.use(http.get(AREAS_URL, resolver));

  // Retorna o request capturado, garantindo que a chamada ocorreu.
  return () => {
    expect(resolver).toHaveBeenCalledOnce();

    const lastCall = resolver.mock.calls[0];

    if (!lastCall) {
      throw new Error('Expected MSW resolver to be called once.');
    }

    const [{ request }] = lastCall;

    return request;
  };
}

describe('areasService', () => {
  beforeEach(() => {
    localStorage.clear();
  });

  describe('listAreas', () => {
    it('should request the areas catalog with the active filter', async () => {
      const getRequest = mockAreasCatalog();

      await listAreas({ active: true });

      const request = getRequest();
      const url = new URL(request.url);

      expect(url.searchParams.get('active')).toBe('true');
      // O interceptor de auth injeta o Bearer token — dá pra validar headers transversais.
      expect(request.headers.get('authorization')).toMatch(/^Bearer /);
    });

    it('should resolve with the response payload', async () => {
      const response = [{ id: 'area-1', name: 'QA' }];
      const getRequest = mockAreasCatalog(response);

      await expect(listAreas({})).resolves.toStrictEqual(response);

      expect(new URL(getRequest().url).search).toBe('');
    });
  });

  // describe('createArea', () => { ... });
  // describe('updateArea', () => { ... });
});
```

Pontos do padrão:

- um `describe` de topo com o nome do service e um `describe` aninhado por função (`listAreas`, `createArea`, ...) com seus `it`;
- `vi.fn<HttpResponseResolver>(...)` tipa o resolver e dá acesso a `resolver.mock.calls`;
- leia `resolver.mock.calls[0]` (não `lastCall`) e proteja com um guard antes de desestruturar — assim o `request` sai sem `!` e sem depender de `noUncheckedIndexedAccess`;
- asserções saem do `request`: `new URL(request.url).searchParams` para query, `request.headers.get(...)` para headers, `await request.json()` para body;
- valide o payload de retorno com `await expect(service(...)).resolves.toStrictEqual(...)`.

## O que não fazer

1. não instanciar `axios` diretamente — sempre importar `api` de `./api`;
2. não adicionar lógica de autenticação ou headers de auth no service;
3. não usar `try/catch` no service — erros devem propagar;
4. não retornar o response completo do axios — sempre `response.data`;
5. não usar `export default` — apenas named exports;
6. não declarar tipos em arquivos separados — co-localizar no próprio service;
7. não testar services mockando `api` quando `msw` puder validar a mesma chamada pela borda HTTP real;
8. não usar `msw` em testes cujo objetivo é apenas provar RBAC/navegação/registro de rotas.
