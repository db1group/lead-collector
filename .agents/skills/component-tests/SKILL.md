---
name: component-tests
description: >
  Padrão de testes de componentes e páginas do Health Panel com Vitest +
  @testing-library/react, em ambiente jsdom. Cobre componentes
  reutilizáveis em `src/components/`, páginas em `src/pages/`, providers em `src/providers/`, etc.
  Detalha queries por role/text/label, act, interação via user-event, wrappers de provider e routing
  (`MemoryRouter`/`createMemoryRouter`) e o padrão de teste de integração.
---

# Testes de Componente

Ambiente: **jsdom**. Os componentes montam no DOM real do jsdom e são consultados via Testing Library.

## Stack

- **Renderer**: `render` e `screen` de `@testing-library/react`.
- **Interação**: `@testing-library/user-event` (`userEvent.setup()` → `click`, `type`, `keyboard`, `tab`).
- **Sincronização assíncrona**: `await screen.findBy...` ou `await waitFor(() => expect(...))` de `@testing-library/react`.
- **Cleanup**: automático no `afterEach` global ([src/test/setup.ts](src/test/setup.ts)).

⚠️ **Use `waitFor` do `@testing-library/react`, não `vi.waitFor`.** O `waitFor` da testing-library envolve cada polling em `act` automaticamente — sem isso o React loga warnings de "update was not wrapped in act" quando o hook recebe `setState` de callback async fire-and-forget.

## Warning de act = falha, nunca ignore

Um teste que loga `... was not wrapped in act(...)` **não passou**, mesmo com as asserções verdes. O warning significa que houve um update de estado que o teste **não esperou** concluir — ou seja, você afirmou sobre um estado intermediário, não sobre o que o usuário veria. Tratar como erro é regra do projeto, não opção.

**Nunca "resolva" um warning de act escondendo-o.** É proibido:

- suprimir ou mockar `console.error` para o warning sumir;
- remover ou enfraquecer a asserção só para o teste ficar verde;
- empacotar código no chute em `act(...)` até o log calar.

A correção é **sempre** esperar o update estabilizar antes de afirmar — `findBy*`, `waitFor` ou `await act(async () => …)` conforme o caso (ver «Esperando coisas assíncronas» e «act vs waitFor»). Se você não sabe qual update ficou pendente, esse é o bug a investigar — não a esconder.

### Como aguardar um update antes de continuar

Interações de `user-event` já rodam dentro de `act` — basta `await` cada uma. O warning aparece quando há um update **depois** da interação (rede, timer, microtask); é esse update que você precisa ancorar.

**Elemento que aparece após o update → `findBy*`** (espera até existir, com `act` automático):

```tsx
await user.click(screen.getByRole('button', { name: 'Salvar' }));

// ❌ afirma no estado intermediário → warning de act
// expect(screen.getByText('Salvo')).toBeTruthy();

// ✅ espera o elemento aparecer antes de afirmar
expect(await screen.findByText('Salvo')).toBeTruthy();
```

**Condição que não é "um elemento apareceu" — callback chamado, atributo mudou, elemento sumiu → `waitFor`** (faz polling até passar):

```tsx
await user.click(screen.getByRole('button', { name: 'Excluir' }));

await waitFor(() => {
  expect(onDelete).toHaveBeenCalledTimes(1);
});

// para desaparecimento, espere o elemento sair antes de seguir:
await waitFor(() => {
  expect(screen.queryByText('Carregando…')).toBeNull();
});
```

**Update disparado por timer/promise que você controla → `await act(async () => …)`** (avança e libera o flush dentro do `act`):

```tsx
vi.useFakeTimers();
render(<MensagemAtrasada />); // dispara setState após 1s

await act(async () => {
  await vi.advanceTimersByTimeAsync(1000);
});

expect(screen.getByText('Pronto')).toBeTruthy();
```

## Queries e asserções — priorize role/text/label

Consulte o DOM pela ótica do usuário, em ordem de preferência: `getByRole` → `getByLabelText` → `getByText` → `getByTestId` (último recurso). `getBy*` e `findBy*` **lançam** quando o elemento não existe, então já funcionam como asserção; use `queryBy*` + `.toBeNull()` para afirmar ausência.

```tsx
import { render, screen } from '@testing-library/react';

it('renders the form title and email field', () => {
  render(<LoginForm />);

  expect(screen.getByRole('heading', { name: 'Entrar na conta' })).toBeTruthy();
  expect(screen.getByLabelText('E-mail')).toBeTruthy();
});
```

> **Matchers**: `@testing-library/jest-dom` não está instalado. Não use `toBeInTheDocument`/`toHaveValue`.
> Use `getBy*` (lança se ausente), `queryBy*` + `.toBeNull()`, e `.textContent`/`.value` para conteúdo.

## Convenções de arquivo

- Nome: `*.test.tsx` ao lado do arquivo testado (componentes, páginas e providers).

## Particularidades de mock

Componentes que recebem tudo por prop não precisam de mock algum — passe `vi.fn()` para callbacks e valores fixos para dados. Para componentes/páginas que consomem context, envolva com o **provider real** (ver abaixo). `vi.mock` fica reservado à fronteira externa:

- ❌ **Não mocke sub-componentes nossos** — eles rodam de verdade (exceto shell tests de roteamento, ver `tests-best-practices`).
- ❌ Não mocke hooks/providers nossos — use o provider real com valor de teste.
- ✅ Mocke `../http/api`, `react-oidc-context`, `oidc-client-ts` (fronteira externa).

### Wrapper com providers reais

```tsx
function renderWithProviders(ui: ReactElement) {
  return render(
    <SettingsProvider value={fakeSettings}>
      <SessionProvider>{ui}</SessionProvider>
    </SettingsProvider>,
  );
}
```

Quando o provider depende de uma SDK externa (ex.: um provider de sessão que consome `useAuth` do `react-oidc-context`), mocke a SDK e construa um contexto de teste completo numa factory local (ver `tests-best-practices`).

## Eventos do usuário — `user-event`

Para interações (clique, digitação, teclado), use `@testing-library/user-event`, que simula eventos reais do browser e já envolve as atualizações em `act`:

```tsx
import userEvent from '@testing-library/user-event';

it('should call onSubmit with the typed email', async () => {
  const user = userEvent.setup();
  const onSubmit = vi.fn();
  render(<LoginForm onSubmit={onSubmit} />);

  await user.type(screen.getByLabelText('Email'), 'a@b.com');
  await user.click(screen.getByRole('button', { name: 'Login' }));

  expect(onSubmit).toHaveBeenCalledWith({ email: 'a@b.com' });
});
```

Para teclado puro (setas, Enter, Esc) use `user.keyboard('{Enter}')`, `user.keyboard('{ArrowDown}')`, `user.keyboard('{Escape}')`. Sempre `await` cada interação.

Quando a interação dispara um efeito assíncrono (rede, transição), ancore a asserção com `findBy*` ou `waitFor`:

```tsx
await user.click(screen.getByRole('button', { name: 'Carregar' }));

expect(await screen.findByText('Concluído')).toBeTruthy();
```

## Páginas/componentes com routing

Quando o componente usa `useNavigate`/`Link`/loaders, envolva com `MemoryRouter`:

```tsx
import { MemoryRouter } from 'react-router';

render(
  <MemoryRouter initialEntries={['/setup']}>
    <SetupPage />
  </MemoryRouter>,
);
```

### Verificando navegação — `createMemoryRouter` + `router.state.location.pathname`

Quando o teste precisa **afirmar que uma ação disparou um `navigate(...)`**, prefira `createMemoryRouter` + `RouterProvider`. Isso expõe `router.state.location.pathname`, fonte de verdade da navegação, independente do que está renderizado:

```tsx
import { createMemoryRouter, RouterProvider, type RouteObject } from 'react-router';

const routes: RouteObject[] = [
  { path: '/login', element: <LoginPage /> },
  { path: '/home', element: <div>home</div> },
];

function renderPage() {
  const router = createMemoryRouter(routes, { initialEntries: ['/login'] });
  const view = render(<RouterProvider router={router} />);
  return { view, router };
}

it('should navigate to /home after a successful login', async () => {
  const user = userEvent.setup();
  const { router } = renderPage();

  await user.click(screen.getByRole('button', { name: 'Login' }));

  await waitFor(() => {
    expect(router.state.location.pathname).toBe('/home');
  });
});
```

A asserção testa o **comportamento real** (`navigate`), não o efeito visual.

> Para fluxos que escrevem direto em `window.location` (ex.: redirect de auth), afira
> `window.location.pathname`/`.search`/`.hash` após `window.history.replaceState(...)` no arrange.

## Padrão por tipo

### Componentes reutilizáveis (`src/components/`) — asserts explícitos por estado

Componentes de apresentação: para cada estado/prop relevante, renderize e afira o que o usuário vê (texto, role, atributos). Prefira asserções explícitas a snapshots — em DOM, snapshots de markup completo são frágeis e ruidosos.

```tsx
it('should render the disabled state', () => {
  render(<Button disabled>Salvar</Button>);

  expect(screen.getByRole('button', { name: 'Salvar' }).hasAttribute('disabled')).toBe(true);
});
```

Use `asFragment().toMatchSnapshot()` (ver abaixo) apenas para componentes de apresentação visualmente estáveis cujo markup você quer travar.

### Páginas (`src/pages/`) e providers — teste de integração

O teste de uma página é um **teste de integração**: hooks, componentes filhos e providers rodam de verdade; os únicos substitutos são os mocks de fronteira externa (`../http/api`, `react-oidc-context`). Valide o fluxo (qual conteúdo aparece em cada estado, efeitos de transição, navegação), não o pixel.

```tsx
it('shows the user name after the session is loaded', async () => {
  mockAuth = createAuthContext({ isAuthenticated: true, user: createUser({ name: 'Maria' }) });

  render(<AppProviders><ProfilePage /></AppProviders>);

  expect(await screen.findByText('Maria')).toBeTruthy();
});
```

Cubra:

- render inicial (qual conteúdo aparece logo após o mount);
- caminho feliz até o estado final;
- caminhos alternativos (erro, vazio, não autorizado);
- efeitos fire-once (callback chamado uma vez por transição).

## Esperando coisas assíncronas

Nunca `await new Promise(r => setTimeout(r, N))` solto. Use:

- `await screen.findByText(...)` / `findByRole(...)` — espera o elemento aparecer;
- `await waitFor(() => expect(...))` (de `@testing-library/react`) — polling até a condição valer, com `act` automático.

Para componentes com timers (animações, debounce):

```typescript
beforeEach(() => { vi.useFakeTimers(); });
afterEach(() => { vi.useRealTimers(); });

it('shows the message after the delay', async () => {
  render(<DelayedMessage />);
  await act(async () => { await vi.advanceTimersByTimeAsync(1000); });
  expect(screen.getByText('Pronto')).toBeTruthy();
});
```

> Com fake timers + `user-event`, configure `userEvent.setup({ advanceTimers: vi.advanceTimersByTime })`.

## Snapshots — quando usar

Em DOM, prefira asserts explícitos. Reserve snapshots para componentes de apresentação estáveis:

- Capture o fragmento, não a tela inteira: `expect(asFragment()).toMatchSnapshot();`.
- Um snapshot por estado visual distinto; congele componentes animados com fake timers antes.
- Atualize com `yarn test -u`.

## API do `render` (`@testing-library/react`)

```typescript
const { container, asFragment, rerender, unmount } = render(<Comp />);
// `screen` é a forma idiomática de consultar — prefira-o a `container.querySelector`.

screen.getByRole('button', { name: '...' })  // lança se ausente
screen.queryByText('...')                      // null se ausente
await screen.findByText('...')                 // espera aparecer
rerender(<Comp prop={novo} />)                 // re-renderiza com novas props
unmount()                                      // desmonta + dispara cleanups
asFragment()                                   // DocumentFragment para snapshot
```

## `act` vs `waitFor`

- **`act(() => syncAction())`** — quando a ação muda estado sincronamente (ex.: `setState` dentro do handler). Garante que o React commite antes de continuar.
- **`await act(async () => { ... })`** — quando a ação retorna uma promise cujo settlement causa mais updates. Espera todas as effects flushed.
- **`await waitFor(() => expect(...))`** (de `@testing-library/react`) — quando a transição depende de um fluxo assíncrono externo (resposta de rede, timer). Polling determinístico, com `act` automático em cada tentativa. **Nunca** use `vi.waitFor` em hook tests — ele não envolve em `act` e gera warnings do React.
