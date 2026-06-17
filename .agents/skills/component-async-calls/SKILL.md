---
name: component-async-calls
description: >
  Define o padrão de hooks para efeitos de montagem e chamadas assíncronas em componentes
  React do Health Panel. Use quando precisar executar lógica na montagem de um componente
  ou disparar chamadas assíncronas com controle de estado de carregamento.
---

# Hooks para efeitos assíncronos em componentes React

## Hooks disponíveis

Os três hooks ficam em `src/hooks/`:

| Hook | Arquivo | Propósito |
|---|---|---|
| `useDidMount` | `use-did-mount.ts` | Executa um efeito apenas uma vez, na montagem |
| `useAsync` | `use-async.ts` | Envolve uma função assíncrona com estado de carregamento e proteção contra atualizações em componentes desmontados |
| `useIsMountedRef` | `use-is-mounted-ref.ts` | Retorna um `ref` que indica se o componente está montado — usado internamente pelo `useAsync` |

## useDidMount

Use para qualquer inicialização ou chamada que deva ocorrer apenas na montagem do componente.
Evita a necessidade de declarar uma lista de dependências vazia explicitamente.

A chamada deve passar por um **service** (que encapsula `../http/api`), nunca chamar `api.get`/`api.post` direto no componente.

```tsx
import useDidMount from '../hooks/use-did-mount';

import { configService } from '../services/config-service';

function MyComponent() {
  useDidMount(() => {
    configService.getConfig().then((config) => {
      setConfig(config);
    });
  });
}
```

Prefira `useDidMount` a `useEffect(() => { ... }, [])` — a intenção fica explícita e não
exige o comentário `// eslint-disable-next-line`.

## useAsync

Use quando precisar expor o estado de carregamento de uma chamada assíncrona para o
componente, ou quando o callback pode ser chamado múltiplas vezes (ex: botão de ação).

```tsx
import useAsync from '../hooks/use-async';
import useDidMount from '../hooks/use-did-mount';

import { itemService } from '../services/item-service';

function MyComponent() {
  const { loading, call: loadData } = useAsync(async (id: string) => {
    const item = await itemService.getById(id);
    setData(item);
  }, []);

  useDidMount(() => {
    loadData('initial-id');
  });

  if (loading) return <p>Carregando…</p>;
  // ...
}
```

O `useAsync` garante que `setLoading(false)` não seja chamado se o componente for
desmontado antes do término da chamada — prevenindo o aviso
`"Can't perform a React state update on an unmounted component"`.

## Quando usar cada um

| Cenário | Hook |
|---|---|
| Buscar dados na montagem sem precisar do estado `loading` | `useDidMount` |
| Buscar dados na montagem com estado `loading` visível | `useAsync` + `useDidMount` |
| Ação do usuário que dispara chamada assíncrona repetível | `useAsync` |
| Verificação simples de estado antes de atualizar state | `useIsMountedRef` (indireto via `useAsync`) |

## O que não fazer

1. Não use `useEffect(() => { ... }, [])` — use `useDidMount`.
2. Não atualize state após await sem checar se o componente ainda está montado — use
   `useAsync` para isso.
