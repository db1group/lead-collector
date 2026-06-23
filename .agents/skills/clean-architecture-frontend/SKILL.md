---
name: clean-architecture-frontend
description: >
  Padrão de Clean Architecture adotado neste projeto para o frontend React + TypeScript.
  Cobre as três camadas (domain, application, infrastructure), a interface de repositório,
  os casos de uso, as implementações concretas e a ponte para o React via Context + useRef.
  Use esta skill ao criar ou editar qualquer parte das camadas domain/, application/,
  infrastructure/ ou hooks/ — inclusive ao adicionar novos domínios funcionais.
metadata:
  author: clovis-cli
  type: technical-skill
---

## Visão Geral

O projeto adota Clean Architecture com três camadas explícitas e uma ponte para o React:

```
domain/          ← entidades puras e contratos (interfaces de repositório)
application/     ← casos de uso; dependem apenas de domain/
infrastructure/  ← implementações concretas (ex.: localStorage); única camada que acessa DOM
hooks/           ← ponte React: instancia os use cases e expõe via Context
```

As camadas dependem apenas das camadas internas — nunca o contrário.

---

## Camada 1 — Domain

### Entidade

Tipo TypeScript puro. Sem lógica de infraestrutura, sem dependências externas.

```ts
// src/domain/lead.ts
export type Temperature = 'cold' | 'warm' | 'hot';

export interface Lead {
  id: string;           // UUID v4, gerado na criação; imutável
  name: string;
  company: string;
  jobTitle: string;     // '' quando não preenchido (nunca undefined/null)
  phone: string;
  email: string;
  identifiedPain: string;
  temperature: Temperature;
  nextStep: string;
  createdAt: string;    // ISO 8601; imutável
}
```

**Regras de entidade:**
- Campos opcionais são sempre `string` vazia `''` na entidade — nunca `undefined` nem `null`.
- `id` e `createdAt` são gerados na criação e nunca modificados.
- A entidade não revalida regras de negócio do formulário (ex.: "phone ou email") — essa validação pertence ao schema Zod no formulário.

### Interface de Repositório

Define o contrato de persistência sem nenhuma referência a `localStorage`, banco ou rede.

```ts
// src/domain/lead-repository.ts
import type { Lead } from '@/domain/lead';

export interface ILeadRepository {
  save(lead: Lead): void;
  delete(id: string): void;
  findAll(): Lead[];  // sem ordenação garantida — ordenação é responsabilidade do use case
}
```

---

## Camada 2 — Application (Casos de Uso)

Cada caso de uso é uma classe com um único método `execute`. Recebe a dependência via
construtor (injeção de dependência explícita) e depende apenas de `domain/`.

### Padrão de caso de uso

```ts
// src/application/create-lead-use-case.ts
import type { Lead, Temperature } from '@/domain/lead';
import type { ILeadRepository } from '@/domain/lead-repository';

export interface CreateLeadInput {
  name: string;
  company: string;
  jobTitle: string;
  phone: string;
  email: string;
  identifiedPain: string;
  temperature: Temperature;
  nextStep: string;
}

export class CreateLeadUseCase {
  private repository: ILeadRepository;

  constructor(repository: ILeadRepository) {
    this.repository = repository;
  }

  execute(input: CreateLeadInput): Lead {
    const lead: Lead = {
      id: crypto.randomUUID(),
      createdAt: new Date().toISOString(),
      ...input,
    };
    this.repository.save(lead);
    return lead;
  }
}
```

```ts
// src/application/list-leads-use-case.ts
export class ListLeadsUseCase {
  private repository: ILeadRepository;

  constructor(repository: ILeadRepository) {
    this.repository = repository;
  }

  // Ordenação aqui, não no repositório
  execute(): Lead[] {
    return this.repository
      .findAll()
      .sort((a, b) => b.createdAt.localeCompare(a.createdAt));
  }
}
```

```ts
// src/application/delete-lead-use-case.ts
export class DeleteLeadUseCase {
  private repository: ILeadRepository;

  constructor(repository: ILeadRepository) {
    this.repository = repository;
  }

  execute(id: string): void {
    this.repository.delete(id);
  }
}
```

**Regras dos casos de uso:**
- Um único método público: `execute`.
- Recebem `ILeadRepository` (interface), nunca a implementação concreta.
- Toda lógica de negócio (ordenação, geração de ID, timestamp) fica aqui, não no repositório.
- Não importam nada de React, DOM, `localStorage` nem infraestrutura.

---

## Camada 3 — Infrastructure

Implementações concretas da interface de repositório. É a **única** camada que acessa
DOM APIs (`localStorage`, `fetch`, etc.).

### Implementação de produção

```ts
// src/infrastructure/local-storage-lead-repository.ts
import type { Lead } from '@/domain/lead';
import type { ILeadRepository } from '@/domain/lead-repository';

const STORAGE_KEY = 'lead-collector:leads';

export class LocalStorageLeadRepository implements ILeadRepository {
  save(lead: Lead): void {
    const leads = this.readAll();
    leads.push(lead);
    localStorage.setItem(STORAGE_KEY, JSON.stringify(leads));
  }

  delete(id: string): void {
    const leads = this.readAll().filter(l => l.id !== id);
    localStorage.setItem(STORAGE_KEY, JSON.stringify(leads));
  }

  findAll(): Lead[] {
    return this.readAll();
  }

  private readAll(): Lead[] {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) return [];
    try {
      return JSON.parse(raw) as Lead[];
    } catch {
      return [];
    }
  }
}
```

### Implementação para testes

```ts
// src/infrastructure/in-memory-lead-repository.ts
export class InMemoryLeadRepository implements ILeadRepository {
  private leads: Lead[] = [];

  save(lead: Lead): void { this.leads.push(lead); }
  delete(id: string): void { this.leads = this.leads.filter(l => l.id !== id); }
  findAll(): Lead[] { return [...this.leads]; }
}
```

**Regras da infraestrutura:**
- Nunca importada diretamente por `domain/` ou `application/`.
- `InMemoryLeadRepository` é usada **exclusivamente nos testes** como substituta da
  implementação de produção — sem necessidade de mockar módulos internos.
- **Interfaces em `domain/`, implementações juntas em `infrastructure/`**: cada interface
  (`IXxx`) fica em seu próprio arquivo em `domain/` — mesmo padrão de `ILeadRepository`.
  A implementação concreta (produção) e a implementação de teste ficam **juntas** em um
  único arquivo em `infrastructure/` com o mesmo nome base da interface (sem o prefixo
  `I`). Exemplo: `domain/window-opener.ts` → apenas `IWindowOpener`;
  `infrastructure/window-opener.ts` → `BrowserWindowOpener` + `InMemoryWindowOpener`.

---

## Ponte React — hooks/

A camada `hooks/` faz a ponte entre a Clean Architecture e o React. O Provider:
1. Recebe a `repository` via prop (injeção de dependência).
2. Instancia os casos de uso uma única vez via `useRef`.
3. Expõe o estado e as ações via Context.

### Por que `useRef` para os casos de uso?

Casos de uso são objetos estáveis — não mudam entre renders. Se instanciados no corpo
do componente, uma nova instância seria criada a cada render, forçando `useCallback` e
`useEffect` a listá-los como dependências (ou a usar `eslint-disable` como workaround).

`useRef` garante que a instância é criada **uma única vez** e a referência permanece
estável para sempre, permitindo deps vazias `[]` legítimas.

```tsx
// src/hooks/use-leads.tsx
import { createContext, useCallback, useContext, useEffect, useRef, useState } from 'react';

export function LeadsProvider({ repository, children }: LeadsProviderProps) {
  const [leads, setLeads] = useState<Lead[]>([]);

  // ✅ Instâncias estáveis — criadas uma única vez no mount
  const listRef   = useRef(new ListLeadsUseCase(repository));
  const createRef = useRef(new CreateLeadUseCase(repository));
  const deleteRef = useRef(new DeleteLeadUseCase(repository));

  useEffect(() => {
    setLeads(listRef.current.execute());
  }, []); // [] legítimo: ref nunca muda

  const addLead = useCallback((input: CreateLeadInput): Lead => {
    const lead = createRef.current.execute(input);
    setLeads(listRef.current.execute());
    return lead;
  }, []); // [] legítimo

  const removeLead = useCallback((id: string): void => {
    deleteRef.current.execute(id);
    setLeads(listRef.current.execute());
  }, []); // [] legítimo

  return (
    <LeadsContext.Provider value={{ leads, addLead, removeLead }}>
      {children}
    </LeadsContext.Provider>
  );
}

// Hook consumidor — lança erro descritivo se chamado fora do Provider
export function useLeads(): LeadsContextValue {
  const context = useContext(LeadsContext);
  if (!context) throw new Error('useLeads must be used within LeadsProvider');
  return context;
}
```

### Wiring no App.tsx

```tsx
// src/App.tsx — única camada que instancia a implementação concreta
import { LocalStorageLeadRepository } from '@/infrastructure/local-storage-lead-repository';

const repository = new LocalStorageLeadRepository();

function App() {
  return (
    <LeadsProvider repository={repository}>
      {/* componentes filhos */}
    </LeadsProvider>
  );
}
```

---

## Exemplo de etrutura de Arquivos (não corresponde a todos os arquivos do projeto)

```
src/
├── domain/
│   ├── lead.ts                 # Entidade + tipo Temperature
│   ├── lead-repository.ts      # Interface ILeadRepository
│   └── window-opener.ts        # Interface IWindowOpener
├── application/
│   ├── create-lead-use-case.ts
│   ├── list-leads-use-case.ts
│   └── delete-lead-use-case.ts
├── infrastructure/
│   ├── lead-repository.ts      # LocalStorageLeadRepository + InMemoryLeadRepository
│   └── window-opener.ts        # BrowserWindowOpener + InMemoryWindowOpener
└── hooks/
    └── use-leads.tsx           # Context + Provider + hook consumidor
```

> **Regra de arquivos de porta/interface**: interfaces ficam em `domain/` em arquivo
> próprio (sem o prefixo `I` no nome do arquivo; ex.: `IWindowOpener` →
> `domain/window-opener.ts`). A implementação de produção e a implementação de teste
> ficam **juntas** em um único arquivo em `infrastructure/` com o mesmo nome base
> (ex.: `infrastructure/window-opener.ts` contém `BrowserWindowOpener` e
> `InMemoryWindowOpener`).

---

## Testes

- **Use cases** e **repositório em memória**: testados com `InMemoryLeadRepository` — sem mock de módulos internos.
- **`LocalStorageLeadRepository`**: testado com `localStorage` stubado via `vi.stubGlobal`.
- **Provider/hook**: testado com `renderHook` + wrapper `<LeadsProvider repository={inMemoryRepo}>`.
- **Componentes**: testados com `render` + wrapper `<LeadsProvider>` — nunca mockando `useLeads`.

```ts
// Teste de use case — sem dependência de React ou DOM
it('execute salva o lead e retorna com id e createdAt', () => {
  const repo = new InMemoryLeadRepository();
  const useCase = new CreateLeadUseCase(repo);

  const lead = useCase.execute({ name: 'Ana', company: 'ACME', ... });

  expect(repo.findAll()).toHaveLength(1);
  expect(lead.id).toBeDefined();
  expect(lead.createdAt).toMatch(/^\d{4}-\d{2}-\d{2}T/);
});
```

---

## Regras Globais do Padrão

1. **Fluxo de dependência**: `hooks/` → `application/` → `domain/` ← `infrastructure/`. Nunca invertido.
2. **Sem mocks de módulos internos** — usar `InMemoryLeadRepository` em vez de `vi.mock('@/domain/...')`.
3. **`eslint-disable` para `react-hooks/exhaustive-deps` é proibido** nos Providers deste projeto — sinal de que o use case não está em `useRef`.
4. **`App.tsx` é o único lugar** onde `LocalStorageLeadRepository` é instanciado diretamente.
5. **Cobertura 100%** para `domain/`, `application/`, `infrastructure/` e `hooks/`.
6. **Interfaces em `domain/`, implementações juntas em `infrastructure/`** — cada interface (`IXxx`) fica em seu próprio arquivo em `domain/`; a implementação de produção e a de teste ficam **juntas** em um único arquivo em `infrastructure/` com o mesmo nome base (sem o prefixo `I`).
