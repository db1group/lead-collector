---
name: tests-best-practices
description: >
  Regras transversais a todos os testes do Health Panel — unitários, de hooks e de componentes
  React DOM. Define quando mockar (e quando não), como prover dependências via props e context
  providers, uso de fakes (`vi.fn`), convenções de nomenclatura e estilo, metas de cobertura
  e padrões de asserção. Use sempre que estiver escrevendo, modificando ou revisando
  qualquer arquivo `*.test.{ts,tsx}`.
---

# Boas práticas de teste

Esta skill cobre as regras que valem para **qualquer** teste do projeto. Cada nível (unitário, hook, componente) tem skill específica com detalhes próprios.

## Stack comum

- **Framework**: Vitest com `globals: true`, ambiente `jsdom`.
- **Config**: bloco `test` em [vite.config.ts](vite.config.ts) — `environment: 'jsdom'`, `setupFiles: ['./src/test/setup.ts']`. O setup provê `window.matchMedia`, e faz `cleanup()` + `sessionStorage.clear()` no `afterEach`.
- **DOM**: `@testing-library/react` (`render`, `screen`, `renderHook`, `act`, `waitFor`, `cleanup`) e `@testing-library/user-event` para interação.
- **Comandos**: `yarn test` (uma rodada), `yarn test:cov` (cobertura), `yarn test -u` (atualizar snapshots).

## Convenções de arquivo

- Nome: `*.test.ts` ou `*.test.tsx` ao lado do arquivo testado.
- Um arquivo de teste por arquivo de produção.
- Imports **sem** extensão (`from './session'`, não `from './session.ts'`), seguindo o restante do código.

## Regra de ouro: `vi.mock` só para fronteira externa

**Dependências internas (qualquer coisa em `src/`) não são mockadas via `vi.mock`.** Em vez disso, dependências são fornecidas pelo mecanismo nativo do React:

- **props** — passe valores e callbacks (`vi.fn()`) direto no componente;
- **context providers reais** — envolva o SUT com o provider de verdade passando um valor de teste.

`vi.mock` é usado **só** para fronteira externa:

- ✅ Camada HTTP: `axios` ou o módulo `../http/api`.
- ✅ SDKs de autenticação: `react-oidc-context`, `oidc-client-ts`.
- ✅ APIs do browser que o ambiente não cobre.
- ❌ Hooks, helpers, lib, providers e componentes nossos (qualquer coisa em `src/`).

### Por que prover em vez de mockar

Mockar código nosso acopla o teste à estrutura de import e mascara mudanças de API — quando a interface muda, o mock continua "funcionando" e o teste passa em verde com produção quebrada. Prover a dependência real (provider/props) ou um **fake que respeita o contrato** quebra o teste quando a API muda, que é exatamente o feedback que queremos.

```tsx
// ✗ Não faça — mocka um provider nosso
vi.mock('../providers/SettingsProvider', () => ({ useSettings: () => fakeSettings }));

// ✓ Faça — usa o provider real com um valor de teste
render(
  <SettingsProvider value={fakeSettings}>
    <ComponentUnderTest />
  </SettingsProvider>,
);
```

> **Exceção — testes de shell/bootstrap rasos.** Quando o objetivo é validar apenas o *roteamento* de um componente de topo (ex.: qual subárvore monta para cada rota), é aceitável stubar os filhos pesados com `vi.mock` para isolar o shell e testar só a decisão de rota. Fora desse caso, prefira render real.

## Padrões de fake

### Fakes de funções e callbacks

Use `vi.fn()` para callbacks passados por prop e afira o efeito:

```tsx
const onSubmit = vi.fn();
render(<MyForm onSubmit={onSubmit} />);
// ... interação ...
expect(onSubmit).toHaveBeenCalledWith({ email: 'a@b.com' });
```

### Fakes de tipos externos completos

Quando o teste precisa de um objeto de uma SDK externa (ex.: `AuthContextProps` do
`react-oidc-context`), construa o **shape inteiro** numa factory local, com overrides — assim o teste
quebra se o contrato externo mudar e o fake protege contra ausência silenciosa de campos:

```tsx
function createAuthContext(overrides: Partial<AuthContextProps> = {}): AuthContextProps {
  return {
    isAuthenticated: false,
    isLoading: false,
    signinSilent: vi.fn().mockResolvedValue(undefined),
    signinRedirect: vi.fn().mockResolvedValue(undefined),
    // ... demais campos do contrato ...
    user: undefined,
    ...overrides,
  } as AuthContextProps;
}
```

### Wrapper com providers reais

Centralize a montagem da árvore de providers num helper quando o arranjo se repete:

```tsx
function renderWithProviders(ui: ReactElement, settings = fakeSettings) {
  return render(<SettingsProvider value={settings}>{ui}</SettingsProvider>);
}
```

## Asserções: side effect observável > spying

Prefira asserir o **resultado observável** (DOM, navegação, callback) ao invés de espiar chamadas internas:

```tsx
// ✗ Acopla ao detalhe de implementação
expect(useNavigate).toHaveBeenCalledWith('/done');

// ✓ Observa o efeito real
expect(window.location.pathname).toBe('/project/edit/42');
expect(screen.getByText('Concluído')).toBeTruthy();
expect(onSubmit).toHaveBeenCalledWith({ email: 'a@b.com' });
```

Para navegação, observe `window.location.pathname` ou `router.state.location.pathname` com
`createMemoryRouter` (ver `component-tests`).

> **Matchers**: `@testing-library/jest-dom` não está instalado. Use queries que **lançam** quando o
> elemento não existe (`getByRole`, `getByText`) como asserção implícita, `queryBy*` + `.toBeNull()`
> para ausência, e `.textContent` para conteúdo — como nos testes existentes.

## Cobertura

Meta operacional: **100% das linhas executáveis**.

Cubra sempre:
1. Happy path.
2. Cada branch (`if/else`, ternários, `??`, `||`).
3. Cada variante de discriminated union.
4. Erros e rejeições esperadas.
5. Edge cases (vazio, limites, valores opcionais ausentes).

Quando uma branch for **genuinamente defensiva e inalcançável** pelo contrato público, simplifique o código eliminando a guarda em vez de adicionar testes inúteis. Não esconda cobertura de código alcançável com `/* v8 ignore */`.

Inspecione áreas descobertas com `yarn test --coverage`.

## Estilo

- Testes em inglês, descrevendo **comportamento observável** (`it('should …', ...)`), seguindo os testes existentes.
- Um arrange-act-assert por `it`.
- Valide contrato externo (input/output/DOM/interações), não estado interno privado.
- Centralize setup repetido em `renderWithProviders`/`setup`/factories; setup único fica direto no `it`.
- Fakes/factories locais ao teste, com nomes claros (`createAuthContext`, `createUser`, `fakeTenant`).
- `describe` no nome da classe/função/componente; `it('should …', ...)` no comportamento.

### Estreitamento de tipo em discriminated unions

Quando uma asserção precisa acessar campos que **só existem em uma das variantes** de uma `discriminated union`, use um `throw` para estreitar o tipo antes de continuar:

```ts
const state = getState();
if (state.kind !== 'failed') throw new Error('Expected to be in failed state');
expect(state.message).toContain('connection failed');
expect(state.retryable).toBe(true);
```

Em vez do antipattern `expect(state.kind).toBe('X')` seguido de `if (state.kind === 'X') { ... }`, que duplica a verificação e indenta as asserts.

**Não use o `throw` como reflexo.** Só vale a pena quando o TypeScript precisa do estreitamento para o código abaixo. Se a sequência for apenas `expect(state.kind).toBe('X')` sem acessar campos discriminados depois, a versão simples basta.

## Checklist antes de submeter

- [ ] Nenhum `vi.mock` aponta para arquivo dentro de `src/` (exceto shell tests de roteamento).
- [ ] Dependências internas providas via provider real ou props, não mockadas.
- [ ] Fakes de tipos externos cobrem o contrato inteiro (com overrides).
- [ ] Asserções no efeito observável (DOM/navegação/callback), não no espelhamento interno.
- [ ] Cobertura mantida (rodar `yarn test --coverage` quando incerto).
