---
name: page-patterns
description: >
  Define como componentes de página devem ser criados em `src/pages/`.
  Use ao criar uma nova página, adicionar fetch de dados a uma página existente,
  decidir sobre estrutura de estado async, extração de sub-componentes,
  ou padrões de formulário com react-hook-form.
---

# Padrão de Pages

## Estrutura e localização

Arquivos ficam em `src/pages/<NomeDaPágina>.tsx`.

Uma página por arquivo. A página é o alvo direto de uma rota — ela é dona do fetch de dados e do estado de nível superior. Sub-componentes específicos da página ficam em `src/components/<nome-da-pagina>/`.

## Export

Páginas usam `export default`. Nunca named export para componente de página.

## Estado async

Operações assíncronas que precisam de feedback de loading usam o hook `useAsync`. Ele gerencia o estado `loading` internamente, protege contra updates em componente desmontado e retorna um `call` estável. O `error` fica a cargo do chamador via `useState`.

```tsx
import useAsync from '../hooks/use-async';

const [items, setItems] = useState<AreaResponse[]>([]);
const [error, setError] = useState<string | null>(null);

const { loading, call: fetchItems } = useAsync(async () => {
  setError(null);
  try {
    const data = await listAreas();
    setItems(data);
  } catch (err) {
    setError(mapApiErrorToMessage(err));
  }
}, []);
```

`useAsync` não é obrigatório em toda operação — para ações pontuais onde o loading não precisa ser refletido na UI (ex: delete com remoção otimista), `useState` manual ainda é válido.

## Inicialização de dados

Usar `useDidMount` para disparar o fetch inicial. Não usar `useEffect` diretamente.

```tsx
import useDidMount from '../hooks/use-did-mount';

useDidMount(() => {
  fetchItems();
});
```

## Exibição de erros

Erros são exibidos via componente `<ErrorMessage>`. Nunca exibir strings de erro diretamente no JSX ou usar `alert`.

```tsx
{error && <ErrorMessage message={error} />}
```

## Loading state

Usar placeholders skeleton para áreas de conteúdo durante o carregamento. Não usar spinners isolados para blocos grandes de conteúdo.

```tsx
{loading && (
  <div className="grid gap-4">
    {Array.from({ length: 3 }).map((_, i) => (
      <div key={i} className="h-20 rounded-lg bg-muted animate-pulse" />
    ))}
  </div>
)}
```

## Layout

Envolver o conteúdo da página em `<div className="container mx-auto p-6">` ou no componente `PageContainer` se ele existir no projeto.

## Formulários

Formulários usam `react-hook-form` com o componente `<Form>` do sistema de UI. A submissão usa `useAsync` para gerenciar o `loading`. Nunca usar `useState` para controlar campos individuais de formulário.

```tsx
const [error, setError] = useState<string | null>(null);

const form = useForm<CreateAreaPayload>({
  defaultValues: { name: '', seniorityRates: { junior: 0, mid: 0, senior: 0, lead: 0 } },
});

const { loading, call: submit } = useAsync(async (values: CreateAreaPayload) => {
  setError(null);
  try {
    await createArea(values);
    navigate(-1);
  } catch (err) {
    setError(mapApiErrorToMessage(err));
  }
}, [navigate]);
```

Para páginas que servem tanto criação quanto edição (mesma rota com `:id` opcional), detectar o modo pela presença do param e carregar os dados existentes com `form.reset()`.

```tsx
const { id } = useParams<{ id?: string }>();
const isEditing = Boolean(id);

const { call: loadItem } = useAsync(async () => {
  if (!id) return;
  try {
    const data = await getArea(id);
    form.reset(data);
  } catch (err) {
    setError(mapApiErrorToMessage(err));
  }
}, [id]);
```

## Params de URL

Ler com `useParams` tipado. O tipo deve ser `string` mesmo que o valor represente um ID numérico — a conversão é responsabilidade do service ou use case.

```tsx
const { id } = useParams<{ id: string }>();
```

## Exemplo — página de listagem (`AdminAreas.tsx`)

```tsx
import { useState } from 'react';
import type { AreaResponse } from '../http/areasService';
import { listAreas } from '../http/areasService';
import { mapApiErrorToMessage } from '../http/errorMapper';
import { ErrorMessage } from '../components/ErrorMessage';
import useAsync from '../hooks/use-async';
import useDidMount from '../hooks/use-did-mount';

export default function AdminAreas() {
  const [areas, setAreas] = useState<AreaResponse[]>([]);
  const [error, setError] = useState<string | null>(null);

  const { loading, call: fetchAreas } = useAsync(async () => {
    setError(null);
    try {
      const data = await listAreas();
      setAreas(data);
    } catch (err) {
      setError(mapApiErrorToMessage(err));
    }
  }, []);

  useDidMount(() => {
    fetchAreas();
  });

  return (
    <div className="container mx-auto p-6">
      <h1 className="text-2xl font-bold mb-6">Áreas</h1>

      {error && <ErrorMessage message={error} />}

      {loading && (
        <div className="grid gap-4">
          {Array.from({ length: 3 }).map((_, i) => (
            <div key={i} className="h-20 rounded-lg bg-muted animate-pulse" />
          ))}
        </div>
      )}

      {!loading && !error && (
        <ul className="grid gap-4">
          {areas.map((area) => (
            <li key={area.id}>{area.name}</li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

## Registro na rota

Toda página nova deve ser registrada em `src/routes/index.tsx` com lazy loading. Consulte a skill `routes` para o padrão.

## O que não fazer

1. não colocar lógica de negócio ou transformação de dados no JSX;
2. não usar `useEffect` diretamente para fetch inicial — usar `useDidMount`;
3. não gerenciar `loading` manualmente quando `useAsync` já o faz;
4. não exibir erros com `alert` ou strings brutas no JSX;
5. não controlar campos de formulário com `useState` — usar `react-hook-form`;
6. não usar `export default` com nome diferente do arquivo;
