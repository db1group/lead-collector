---
name: hook-tests
description: >
  Padrão de testes de hooks React do Health Panel rodando em ambiente jsdom (Vitest). Cobre hooks
  reutilizáveis em `src/hooks/`
  e hooks de feature. Detalha `renderHook`, distinção entre `act`/`waitFor`/`vi.waitFor`, wrapper
  de context providers e `MemoryRouter` para hooks que usam `useNavigate`.
---

# Testes de Hook

Ambiente: **jsdom**. O DOM existe para o `react-dom` montar a árvore interna do `renderHook` — os asserts são sobre o valor retornado pelo hook, não sobre `document`.

## Stack

- **Renderer**: `renderHook` de `@testing-library/react`.
- **Sincronização**: `act` e `waitFor` de `@testing-library/react`.

⚠️ **Use `waitFor` do `@testing-library/react`, não `vi.waitFor`.** O `waitFor` da testing-library envolve cada polling em `act` automaticamente — sem isso o React loga warnings de "update was not wrapped in act" quando o hook recebe `setState` de callback async fire-and-forget.

## Convenções de arquivo

- Nome: `use-*.test.ts` ou `use-*.test.tsx` ao lado do hook. Use `.tsx` apenas quando o teste tiver JSX (ex.: wrapper `<MemoryRouter>` ou um `useHarness` com elementos).

## Padrão por tipo de hook

### Hook sem dependências externas

Chamar `renderHook(() => useHook())` e acessar `result.current`. Para cobrir cleanup de `useEffect`, dispare `unmount()` retornado pelo `renderHook`:

```tsx
import { renderHook } from '@testing-library/react';

it('should expose a ref that flips to true after mount', () => {
  const { result } = renderHook(() => useIsMountedRef());

  expect(result.current.current).toBe(true);
});

it('should flip the ref back to false after unmount', () => {
  const { result, unmount } = renderHook(() => useIsMountedRef());

  unmount();

  expect(result.current.current).toBe(false);
});
```

### Hook com ação que dispara `setState`

Envolva a chamada que muda estado em `act`. Para promises retornadas pelo hook, `await act(async () => { ... })` espera o ciclo completo.

```tsx
import { act, renderHook } from '@testing-library/react';

it('should flip loading=true during the call and back to false when settled', async () => {
  let release: (value: string) => void = () => undefined;
  const callback = vi.fn(() => new Promise<string>((resolve) => {
    release = resolve;
  }));

  const { result } = renderHook(() => useAsync(callback, [callback]));

  let pending: Promise<unknown> | undefined;
  act(() => {
    pending = result.current.call();
  });
  expect(result.current.loading).toBe(true);

  await act(async () => {
    release('done');
    await pending;
  });
  expect(result.current.loading).toBe(false);
});
```

### Hook que recebe `state`/`setState` como argumento

Quando o hook não dona o próprio estado e espera `state` + `setState` por argumento, hospede-o num pequeno `useHarness` que cria o `useState` local e devolve `state` + a API do hook:

```tsx
function useHarness(initial: ScreenState, onComplete: OnComplete) {
  const [state, setState] = useState<ScreenState>(initial);
  const api = useMyHook({ state, setState, onComplete });
  return { state, api };
}

function setup(initial: ScreenState, onComplete = vi.fn()) {
  const { result } = renderHook(() => useHarness(initial, onComplete));
  return { result, onComplete };
}

it('should transition to the next state', () => {
  const { result, onComplete } = setup(buildInitialState());

  act(() => result.current.api.actions.doStep());

  expect(onComplete).toHaveBeenCalledTimes(1);
});
```

`result.current.state` sempre devolve o estado mais recente. `result.current.api.actions` também — **não desestruturar fora do `result.current`**, senão captura uma versão velha.

### Hook que consome um context provider

Quando o hook resolve um valor de context, envolva-o com o **provider real** via `wrapper`, passando um valor de teste:

```tsx
import { renderHook } from '@testing-library/react';

import { SettingsProvider } from '../providers/SettingsProvider';

const { result } = renderHook(() => useSettings(), {
  wrapper: ({ children }) => (
    <SettingsProvider value={fakeSettings}>{children}</SettingsProvider>
  ),
});

expect(result.current.theme).toBe('dark');
```

Para hooks que dependem de uma SDK externa, mocke **a SDK** (fronteira externa), não o hook nosso — ver `tests-best-practices`.

### Hook que faz chamada HTTP

Mocke o módulo `../http/api` (fronteira externa) e afira o estado resultante com `waitFor`:

```tsx
vi.mock('../http/api', () => ({ api: { get: vi.fn() } }));

it('should reach the loaded state after fetching', async () => {
  vi.mocked(api.get).mockResolvedValue({ data: { id: 1 } });

  const { result } = renderHook(() => useResource());

  await waitFor(() => {
    expect(result.current.data).toEqual({ id: 1 });
  });
});
```

Use `waitFor` (do `@testing-library/react`) sempre que a transição depender de algo assíncrono que escapa do `act` (ex.: resposta de rede resolvida).

### Hooks que usam `useNavigate`

Hooks que chamam `useNavigate` (do `react-router`) precisam de um `MemoryRouter` como wrapper:

```tsx
import { MemoryRouter } from 'react-router';
import { renderHook } from '@testing-library/react';

const { result } = renderHook(() => useMyHook(), {
  wrapper: ({ children }) => (
    <MemoryRouter initialEntries={['/start']}>{children}</MemoryRouter>
  ),
});
```

Para asserir navegação, observe o **side effect** (estado final ou DOM) em vez de espiar a função. Para checar a rota diretamente, use `createMemoryRouter` (ver `component-tests`).

## `act` vs `waitFor`

- **`act(() => syncAction())`** — quando a ação muda estado sincronamente (ex.: `setState` dentro do handler). Garante que o React commite antes de continuar.
- **`await act(async () => { ... })`** — quando a ação retorna uma promise cujo settlement causa mais updates. Espera todas as effects flushed.
- **`await waitFor(() => expect(...))`** (de `@testing-library/react`) — quando a transição depende de um fluxo assíncrono externo (resposta de rede, timer). Polling determinístico, com `act` automático em cada tentativa. **Nunca** use `vi.waitFor` em hook tests — ele não envolve em `act` e gera warnings do React.
