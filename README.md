# Mini Documentação: @tanstack/react-query

## Índice

1. [Introdução](#introdução)
2. [Instalação](#instalação)
3. [Configuração Básica](#configuração-básica)
4. [Estrutura de Organização](#estrutura-de-organização)
5. [Hooks Básicos](#hooks-básicos)
6. [Manipulação de Mutações](#manipulação-de-mutações)
7. [Uso Avançado](#uso-avançado)
8. [Exemplos Práticos](#exemplos-práticos)
9. [Dicas e Boas Práticas](#dicas-e-boas-práticas)

## Introdução

O TanStack Query (anteriormente conhecido como React Query) é uma biblioteca de gerenciamento de estado para dados assíncronos em aplicações React. Ele facilita a obtenção, o cache, a sincronização e a atualização do estado do servidor.

**Principais benefícios:**

- Gerenciamento automático de cache
- Deduplicação de requisições
- Background updates
- Refetch automático
- Manipulação de dados obsoletos (stale data)
- Paginação e carregamento infinito
- Prefetching de dados

## Instalação

### Para projetos Vite + React + TypeScript:

```bash
# NPM
npm install @tanstack/react-query

# Yarn
yarn add @tanstack/react-query

# PNPM
pnpm add @tanstack/react-query
```

### Para projetos Next.js:

```bash
# NPM
npm install @tanstack/react-query @tanstack/react-query-devtools

# Yarn
yarn add @tanstack/react-query @tanstack/react-query-devtools

# PNPM
pnpm add @tanstack/react-query @tanstack/react-query-devtools
```

## Configuração Básica

### Configuração em aplicações Vite:

```tsx
// src/main.tsx
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";

// Criar uma instância do QueryClient
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // Configurações globais para todas as queries
      staleTime: 60 * 1000, // 1 minuto
      cacheTime: 5 * 60 * 1000, // 5 minutos
      refetchOnWindowFocus: false,
      retry: 1,
    },
  },
});

ReactDOM.createRoot(document.getElementById("root") as HTMLElement).render(
  <React.StrictMode>
    <QueryClientProvider client={queryClient}>
      <App />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  </React.StrictMode>
);
```

### Configuração em aplicações Next.js:

```tsx
// src/pages/_app.tsx
import type { AppProps } from "next/app";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";
import { useState } from "react";

export default function App({ Component, pageProps }: AppProps) {
  // Criar uma nova instância do QueryClient para cada sessão
  // Isso é importante para evitar compartilhamento de estado entre usuários
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 60 * 1000,
            cacheTime: 5 * 60 * 1000,
            refetchOnWindowFocus: false,
            retry: 1,
          },
        },
      })
  );

  return (
    <QueryClientProvider client={queryClient}>
      <Component {...pageProps} />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

## Estrutura de Organização

Uma estrutura recomendada para organizar o código do React Query:

```
src/
├── api/
│   ├── axios.ts              # Configuração do Axios
│   └── endpoints.ts          # Definição de endpoints
├── hooks/
│   ├── query/
│   │   ├── useUsers.ts       # Hook para obter usuários
│   │   ├── usePosts.ts       # Hook para obter posts
│   │   └── index.ts          # Exporta todos os hooks
│   └── mutation/
│       ├── useCreateUser.ts  # Hook para criar usuário
│       ├── useUpdatePost.ts  # Hook para atualizar post
│       └── index.ts          # Exporta todos os hooks de mutação
└── services/
    ├── userService.ts        # Serviços relacionados a usuários
    └── postService.ts        # Serviços relacionados a posts
```

### Configuração do Axios:

```tsx
// src/api/axios.ts
import axios from "axios";

const api = axios.create({
  baseURL: "https://api.example.com",
  headers: {
    "Content-Type": "application/json",
  },
});

// Interceptor para adicionar token de autorização
api.interceptors.request.use((config) => {
  const token = localStorage.getItem("token");
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export default api;
```

### Definição de endpoints:

```tsx
// src/api/endpoints.ts
export const ENDPOINTS = {
  USERS: "/users",
  POSTS: "/posts",
  COMMENTS: "/comments",
};
```

## Hooks Básicos

### Serviço de usuário:

```tsx
// src/services/userService.ts
import api from "../api/axios";
import { ENDPOINTS } from "../api/endpoints";

export interface User {
  id: number;
  name: string;
  email: string;
}

export interface UserFilters {
  search?: string;
  page?: number;
  limit?: number;
}

export const userService = {
  getUsers: async (filters?: UserFilters): Promise<User[]> => {
    const { data } = await api.get(ENDPOINTS.USERS, { params: filters });
    return data;
  },

  getUserById: async (id: number): Promise<User> => {
    const { data } = await api.get(`${ENDPOINTS.USERS}/${id}`);
    return data;
  },
};
```

### Hook de consulta:

```tsx
// src/hooks/query/useUsers.ts
import { useQuery, UseQueryOptions } from "@tanstack/react-query";
import { userService, User, UserFilters } from "../../services/userService";

// Chaves para as queries
export const QUERY_KEYS = {
  USERS: "users",
  USER_DETAILS: "user-details",
};

// Hook para obter lista de usuários
export const useUsers = (filters?: UserFilters, options?: UseQueryOptions<User[]>) => {
  return useQuery<User[]>({
    queryKey: [QUERY_KEYS.USERS, filters], // A chave muda quando os filtros mudam
    queryFn: () => userService.getUsers(filters),
    ...options,
  });
};

// Hook para obter detalhes de um usuário
export const useUserDetails = (id: number, options?: UseQueryOptions<User>) => {
  return useQuery<User>({
    queryKey: [QUERY_KEYS.USER_DETAILS, id],
    queryFn: () => userService.getUserById(id),
    enabled: !!id, // Só executa se id for fornecido
    ...options,
  });
};
```

### Usando o hook em um componente:

```tsx
// src/components/UserList.tsx
import React, { useState } from "react";
import { useUsers } from "../hooks/query/useUsers";

export const UserList = () => {
  const [page, setPage] = useState(1);
  const { data, isLoading, isError, error } = useUsers({ page, limit: 10 });

  if (isLoading) return <div>Carregando...</div>;
  if (isError) return <div>Erro ao carregar: {error.message}</div>;

  return (
    <div>
      <h2>Lista de Usuários</h2>
      <ul>
        {data?.map((user) => (
          <li key={user.id}>
            {user.name} - {user.email}
          </li>
        ))}
      </ul>
      <button onClick={() => setPage((p) => Math.max(1, p - 1))} disabled={page === 1}>
        Anterior
      </button>
      <span> Página {page} </span>
      <button onClick={() => setPage((p) => p + 1)}>Próxima</button>
    </div>
  );
};
```

## Manipulação de Mutações

### Serviço para criação de usuário:

```tsx
// src/services/userService.ts (adicionando ao arquivo existente)
export interface CreateUserData {
  name: string;
  email: string;
  password: string;
}

// Adicionando ao userService existente
export const userService = {
  // ... outros métodos

  createUser: async (userData: CreateUserData): Promise<User> => {
    const { data } = await api.post(ENDPOINTS.USERS, userData);
    return data;
  },

  updateUser: async (id: number, userData: Partial<User>): Promise<User> => {
    const { data } = await api.patch(`${ENDPOINTS.USERS}/${id}`, userData);
    return data;
  },

  deleteUser: async (id: number): Promise<void> => {
    await api.delete(`${ENDPOINTS.USERS}/${id}`);
  },
};
```

### Hook de mutação:

```tsx
// src/hooks/mutation/useCreateUser.ts
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { userService, CreateUserData, User } from "../../services/userService";
import { QUERY_KEYS } from "../query/useUsers";

export const useCreateUser = () => {
  const queryClient = useQueryClient();

  return useMutation<User, Error, CreateUserData>({
    mutationFn: (userData) => userService.createUser(userData),
    onSuccess: (newUser) => {
      // Atualiza o cache após a criação bem-sucedida
      queryClient.invalidateQueries({ queryKey: [QUERY_KEYS.USERS] });

      // Opcional: Atualize o cache diretamente sem fazer nova requisição
      queryClient.setQueryData<User[]>([QUERY_KEYS.USERS], (oldData) => (oldData ? [...oldData, newUser] : [newUser]));
    },
  });
};
```

### Hook de atualização:

```tsx
// src/hooks/mutation/useUpdateUser.ts
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { userService, User } from "../../services/userService";
import { QUERY_KEYS } from "../query/useUsers";

interface UpdateUserVariables {
  id: number;
  data: Partial<User>;
}

export const useUpdateUser = () => {
  const queryClient = useQueryClient();

  return useMutation<User, Error, UpdateUserVariables>({
    mutationFn: ({ id, data }) => userService.updateUser(id, data),
    onSuccess: (updatedUser) => {
      // Invalidar a query para forçar uma nova requisição
      queryClient.invalidateQueries({ queryKey: [QUERY_KEYS.USERS] });

      // Atualizar o cache de detalhes do usuário
      queryClient.invalidateQueries({
        queryKey: [QUERY_KEYS.USER_DETAILS, updatedUser.id],
      });
    },
  });
};
```

### Usando a mutação em um componente:

```tsx
// src/components/UserForm.tsx
import React, { useState } from "react";
import { useCreateUser } from "../hooks/mutation/useCreateUser";

export const UserForm = () => {
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");

  const createUser = useCreateUser();

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();

    createUser.mutate(
      { name, email, password },
      {
        onSuccess: () => {
          // Limpar o formulário após sucesso
          setName("");
          setEmail("");
          setPassword("");
          alert("Usuário criado com sucesso!");
        },
        onError: (error) => {
          alert(`Erro ao criar usuário: ${error.message}`);
        },
      }
    );
  };

  return (
    <form onSubmit={handleSubmit}>
      <h2>Criar Novo Usuário</h2>
      <div>
        <label htmlFor="name">Nome:</label>
        <input id="name" type="text" value={name} onChange={(e) => setName(e.target.value)} required />
      </div>
      <div>
        <label htmlFor="email">Email:</label>
        <input id="email" type="email" value={email} onChange={(e) => setEmail(e.target.value)} required />
      </div>
      <div>
        <label htmlFor="password">Senha:</label>
        <input id="password" type="password" value={password} onChange={(e) => setPassword(e.target.value)} required />
      </div>
      <button type="submit" disabled={createUser.isPending}>
        {createUser.isPending ? "Criando..." : "Criar Usuário"}
      </button>
      {createUser.isError && <div style={{ color: "red" }}>Erro: {createUser.error.message}</div>}
    </form>
  );
};
```

## Uso Avançado

### Configuração de Queries Infinitas:

```tsx
// src/services/postService.ts
import api from "../api/axios";
import { ENDPOINTS } from "../api/endpoints";

export interface Post {
  id: number;
  title: string;
  body: string;
  userId: number;
}

export interface PostsResponse {
  data: Post[];
  meta: {
    totalCount: number;
    pageCount: number;
    currentPage: number;
  };
}

export const postService = {
  getPosts: async (page = 1, limit = 10): Promise<PostsResponse> => {
    const { data } = await api.get(ENDPOINTS.POSTS, {
      params: { page, limit },
    });
    return data;
  },
};
```

### Hook de Consulta Infinita:

```tsx
// src/hooks/query/usePosts.ts
import { useInfiniteQuery } from "@tanstack/react-query";
import { postService, PostsResponse } from "../../services/postService";

export const QUERY_KEYS = {
  POSTS: "posts",
};

export const usePosts = () => {
  return useInfiniteQuery<PostsResponse>({
    queryKey: [QUERY_KEYS.POSTS],
    queryFn: ({ pageParam = 1 }) => postService.getPosts(pageParam, 10),
    getNextPageParam: (lastPage) => {
      // Se a página atual for menor que o total de páginas, retorna o próximo número de página
      const { currentPage, pageCount } = lastPage.meta;
      return currentPage < pageCount ? currentPage + 1 : undefined;
    },
    getPreviousPageParam: (firstPage) => {
      // Se a página atual for maior que 1, retorna a página anterior
      const { currentPage } = firstPage.meta;
      return currentPage > 1 ? currentPage - 1 : undefined;
    },
  });
};
```

### Componente com carregamento infinito:

```tsx
// src/components/InfinitePostList.tsx
import React from "react";
import { usePosts } from "../hooks/query/usePosts";

export const InfinitePostList = () => {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage, status, error } = usePosts();

  if (status === "pending") return <div>Carregando...</div>;
  if (status === "error") return <div>Erro: {error.message}</div>;

  return (
    <div>
      <h2>Lista de Posts</h2>
      {data.pages.map((page, i) => (
        <React.Fragment key={i}>
          {page.data.map((post) => (
            <div key={post.id} style={{ margin: "10px 0", padding: "10px", border: "1px solid #ddd" }}>
              <h3>{post.title}</h3>
              <p>{post.body}</p>
            </div>
          ))}
        </React.Fragment>
      ))}

      <button onClick={() => fetchNextPage()} disabled={!hasNextPage || isFetchingNextPage}>
        {isFetchingNextPage ? "Carregando mais..." : hasNextPage ? "Carregar Mais" : "Não há mais posts"}
      </button>
    </div>
  );
};
```

### Uso de prefetching:

```tsx
// src/components/UserListWithPrefetch.tsx
import React, { useState } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { useUsers, QUERY_KEYS } from "../hooks/query/useUsers";
import { userService } from "../services/userService";

export const UserListWithPrefetch = () => {
  const [page, setPage] = useState(1);
  const queryClient = useQueryClient();
  const { data, isLoading, isError, error } = useUsers({ page, limit: 10 });

  // Prefetch da próxima página quando o mouse passar sobre o botão de próxima
  const prefetchNextPage = () => {
    queryClient.prefetchQuery({
      queryKey: [QUERY_KEYS.USERS, { page: page + 1, limit: 10 }],
      queryFn: () => userService.getUsers({ page: page + 1, limit: 10 }),
    });
  };

  if (isLoading) return <div>Carregando...</div>;
  if (isError) return <div>Erro ao carregar: {error.message}</div>;

  return (
    <div>
      <h2>Lista de Usuários</h2>
      <ul>
        {data?.map((user) => (
          <li key={user.id}>
            {user.name} - {user.email}
          </li>
        ))}
      </ul>
      <button onClick={() => setPage((p) => Math.max(1, p - 1))} disabled={page === 1}>
        Anterior
      </button>
      <span> Página {page} </span>
      <button onClick={() => setPage((p) => p + 1)} onMouseEnter={prefetchNextPage}>
        Próxima
      </button>
    </div>
  );
};
```

## Exemplos Práticos

### Otimização com Query Keys:

```tsx
// src/hooks/query/useProjectData.ts
import { useQuery } from "@tanstack/react-query";
import api from "../../api/axios";

interface Project {
  id: number;
  name: string;
  status: "active" | "completed" | "on-hold";
}

// Usando chaves estruturadas para melhor organização
export const projectKeys = {
  all: ["projects"] as const,
  lists: () => [...projectKeys.all, "list"] as const,
  list: (filters: object) => [...projectKeys.lists(), filters] as const,
  details: () => [...projectKeys.all, "detail"] as const,
  detail: (id: number) => [...projectKeys.details(), id] as const,
};

// Hook para obter projetos filtrados
export const useProjects = (status?: Project["status"]) => {
  return useQuery({
    queryKey: projectKeys.list({ status }),
    queryFn: async () => {
      const { data } = await api.get("/projects", { params: { status } });
      return data as Project[];
    },
  });
};

// Hook para obter detalhes de um projeto
export const useProjectDetails = (id: number) => {
  return useQuery({
    queryKey: projectKeys.detail(id),
    queryFn: async () => {
      const { data } = await api.get(`/projects/${id}`);
      return data as Project;
    },
    enabled: !!id,
  });
};
```

### Custom hook para reuso de lógica:

```tsx
// src/hooks/query/useGenericQuery.ts
import { useQuery, UseQueryOptions, QueryKey } from "@tanstack/react-query";
import { AxiosError } from "axios";

// Hook genérico para qualquer tipo de consulta
export function useGenericQuery<TData>(
  queryKey: QueryKey,
  queryFn: () => Promise<TData>,
  options?: UseQueryOptions<TData, AxiosError>
) {
  return useQuery<TData, AxiosError>({
    queryKey,
    queryFn,
    retry: (failureCount, error) => {
      // Não tenta novamente para erros 4xx
      if (error.response?.status && error.response.status >= 400 && error.response.status < 500) {
        return false;
      }
      // Tenta novamente até 3 vezes para outros erros
      return failureCount < 3;
    },
    ...options,
  });
}
```

### Tratamento de erros avançado:

```tsx
// src/hooks/query/useReportData.ts
import { useQuery, useQueryClient } from "@tanstack/react-query";
import api from "../../api/axios";
import { toast } from "react-toastify"; // Supondo que você use react-toastify

export const useReportData = (reportId: string) => {
  const queryClient = useQueryClient();

  return useQuery({
    queryKey: ["report", reportId],
    queryFn: async () => {
      try {
        const { data } = await api.get(`/reports/${reportId}`);
        return data;
      } catch (error) {
        // Tratamento personalizado para diferentes códigos de erro
        if (error.response) {
          switch (error.response.status) {
            case 401:
              toast.error("Sessão expirada. Por favor, faça login novamente.");
              // Redirecionar para login
              break;
            case 403:
              toast.error("Você não tem permissão para acessar este relatório.");
              break;
            case 404:
              toast.warning("Relatório não encontrado.");
              break;
            default:
              toast.error(`Erro ao carregar relatório: ${error.response.data.message || "Erro desconhecido"}`);
          }
        } else {
          toast.error("Erro de conexão ao servidor. Verifique sua internet.");
        }
        throw error;
      }
    },
    // Tentar novamente apenas para erros de rede
    retry: (failureCount, error) => {
      return !error.response && failureCount < 3;
    },
    onError: () => {
      // Limpar cache de dados relacionados em caso de erro
      queryClient.invalidateQueries({ queryKey: ["user-reports"] });
    },
  });
};
```

## Dicas e Boas Práticas

### 1. Estrutura de Chaves de Consulta

Organize suas chaves de consulta de forma hierárquica:

```tsx
// Mau exemplo
["users", { page: 1 }][("userDetails", 123)][
  // Bom exemplo
  ("users", "list", { page: 1 })
][("users", "detail", 123)];
```

### 2. Aproveite o Poder do Stale-While-Revalidate

```tsx
// Configurando staleTime para diferentes tipos de dados
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // Dados raramente alterados podem ter um staleTime maior
      staleTime: 60 * 1000, // 1 minuto como padrão
    },
  },
});

// Para dados específicos que mudam com frequência
useQuery({
  queryKey: ["notifications"],
  queryFn: fetchNotifications,
  staleTime: 5 * 1000, // 5 segundos
});

// Para dados de referência que raramente mudam
useQuery({
  queryKey: ["countries"],
  queryFn: fetchCountries,
  staleTime: 24 * 60 * 60 * 1000, // 24 horas
});
```

### 3. Uso de Suspense para Carregamento Declarativo

```tsx
// src/App.tsx
import { Suspense } from "react";
import { QueryClientProvider, QueryClient } from "@tanstack/react-query";
import UserList from "./components/UserList";

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      suspense: true, // Habilitar modo suspense
    },
  },
});

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Suspense fallback={<div>Carregando...</div>}>
        <UserList />
      </Suspense>
    </QueryClientProvider>
  );
}

// No componente, não precisamos mais verificar isLoading
// src/components/UserList.tsx
function UserList() {
  const { data } = useUsers();

  return (
    <ul>
      {data.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

### 4. Uso de Window Focus Refetching com Cautela

```tsx
// Configuração global
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      refetchOnWindowFocus: false, // Desabilitado por padrão
    },
  },
});

// Habilitado seletivamente para dados críticos
useQuery({
  queryKey: ["critical-data"],
  queryFn: fetchCriticalData,
  refetchOnWindowFocus: true,
});
```

### 5. Otimização para SSR com Next.js

```tsx
// pages/users.tsx
import { dehydrate, QueryClient } from "@tanstack/react-query";
import { GetServerSideProps } from "next";
import { userService } from "../services/userService";
import { UserList } from "../components/UserList";
import { QUERY_KEYS } from "../hooks/query/useUsers";

export const getServerSideProps: GetServerSideProps = async () => {
  const queryClient = new QueryClient();

  // Pré-carregue os dados durante SSR
  await queryClient.prefetchQuery({
    queryKey: [QUERY_KEYS.USERS, { page: 1, limit: 10 }],
    queryFn: () => userService.getUsers({ page: 1, limit: 10 }),
  });

  return {
    props: {
      // Desidrate o cache para o cliente
      dehydratedState: dehydrate(queryClient),
    },
  };
};

// Componente de página
export default function UsersPage() {
  return <UserList />;
}
```

### 6. Usando Query Observers para Lógica Avançada

```tsx
// src/components/UserCounter.tsx
import { useQueryClient } from "@tanstack/react-query";
import { useEffect, useState } from "react";
import { QUERY_KEYS } from "../hooks/query/useUsers";

export const UserCounter = () => {
  const queryClient = useQueryClient();
  const [count, setCount] = useState(0);

  useEffect(() => {
    // Criar um observer para a query de usuários
    const observer = queryClient.getQueryCache().find({
      queryKey: [QUERY_KEYS.USERS],
    });

    // Atualizar contagem quando os dados mudarem
    const unsubscribe = observer?.subscribe(() => {
      const data = queryClient.getQueryData([QUERY_KEYS.USERS]);
      if (data && Array.isArray(data)) {
        setCount(data.length);
      }
    });

    return () => {
      unsubscribe?.();
    };
  }, [queryClient]);

  return <div>Total de usuários: {count}</div>;
};
```

### 7. Lidando com Dependências de Query

```tsx
// src/hooks/query/useUserPosts.ts
import { useQuery } from "@tanstack/react-query";
import { postService } from "../../services/postService";

export const useUserPosts = (userId: number | undefined) => {
  // Primeiro, obter os detalhes do usuário
  const userQuery = useQuery({
    queryKey: ["user", userId],
    queryFn: () => getUserById(userId!),
    enabled: !!userId,
  });

  // Depois, obter os posts desse usuário (depende da query anterior)
  const postsQuery = useQuery({
    queryKey: ["posts", userId],
    queryFn: () => postService.getPostsByUserId(userId!),
    // Só executa se o userId existir e a query do usuário foi bem-sucedida
    enabled: !!userId && !!userQuery.data,
  });

  return {
    user: userQuery.data,
    posts: postsQuery.data,
    isLoading: userQuery.isLoading || postsQuery.isLoading,
    isError: userQuery.isError || postsQuery.isError,
  };
};
```

### 8. Otimização de Cache

```tsx
// src/hooks/mutation/useUpdateUserOptimistic.ts
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { userService, User } from "../../services/userService";

interface UpdateUserData {
  id: number;
  updates: Partial<User>;
}

export const useUpdateUserOptimistic = () => {
  const queryClient = useQueryClient();

  return useMutation<User, Error, UpdateUserData>({
    mutationFn: ({ id, updates }) => userService.updateUser(id, updates),

    // Atualização otimista - aplicar mudanças antes da resposta da API
    onMutate: async ({ id, updates }) => {
      // Cancelar queries relacionadas
      await queryClient.cancelQueries({ queryKey: ["user", id] });

      // Snapshot do estado anterior para rollback se necessário
      const previousUser = queryClient.getQueryData<User>(["user", id]);

      // Atualizar o cache otimisticamente
      if (previousUser) {
        queryClient.setQueryData<User>(["user", id], {
          ...previousUser,
          ...updates,
        });
      }

      // Retornar o snapshot para usar no onError
      return { previousUser };
    },

    // Em caso de erro, reverter para o estado anterior
    onError: (err, { id }, context) => {
      if (context?.previousUser) {
        queryClient.setQueryData(["user", id], context.previousUser);
      }
      console.error("Erro ao atualizar usuário:", err);
    },

    // Depois da mutação, invalidar queries relacionadas
    onSettled: (_, __, { id }) => {
      queryClient.invalidateQueries({ queryKey: ["user", id] });
      queryClient.invalidateQueries({ queryKey: ["users"] });
    },
  });
};
```

### 9. Configurando Cache Time e Stale Time

```tsx
// Diferença entre cacheTime e staleTime

/*
staleTime: Tempo que os dados permanecem "frescos". Dados frescos não serão 
refetchados automaticamente quando a query for remontada ou quando window 
ganhar foco.

cacheTime: Tempo de inatividade após o qual os dados são removidos do cache. 
Os dados persistem no cache enquanto há observadores ou até que o cacheTime expire.
*/

// Exemplo de configuração para diferentes tipos de dados
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60 * 1000, // 1 minuto (padrão)
      cacheTime: 15 * 60 * 1000, // 15 minutos (padrão)
    },
  },
});

// Para dados estáticos como configurações do sistema
useQuery({
  queryKey: ["app-config"],
  queryFn: fetchAppConfig,
  staleTime: Infinity, // Nunca fica obsoleto
  cacheTime: Infinity, // Nunca é removido do cache
});

// Para dados de alta volatilidade como cotações de ações
useQuery({
  queryKey: ["stock-price", tickerSymbol],
  queryFn: () => fetchStockPrice(tickerSymbol),
  staleTime: 10 * 1000, // 10 segundos
  refetchInterval: 30 * 1000, // Atualiza a cada 30 segundos automaticamente
});
```

### 10. Implementação de Cancel Queries

```tsx
// src/services/searchService.ts
import axios, { CancelToken } from "axios";

export const searchService = {
  search: async (term: string, cancelToken: CancelToken) => {
    const { data } = await axios.get("/api/search", {
      params: { q: term },
      cancelToken,
    });
    return data;
  },
};

// src/hooks/query/useSearch.ts
import { useQuery, useQueryClient } from "@tanstack/react-query";
import axios from "axios";
import { searchService } from "../../services/searchService";
import { useEffect } from "react";

export const useSearch = (term: string) => {
  const queryClient = useQueryClient();

  // Cancelar queries anteriores quando o termo mudar
  useEffect(() => {
    return () => {
      queryClient.cancelQueries({ queryKey: ["search", term] });
    };
  }, [term, queryClient]);

  return useQuery({
    queryKey: ["search", term],
    queryFn: () => {
      const source = axios.CancelToken.source();

      // Retorne a Promise com o cancelToken
      const promise = searchService.search(term, source.token);

      // Anexe a função cancel à Promise
      // @ts-ignore
      promise.cancel = () => {
        source.cancel("Query foi cancelada por React Query");
      };

      return promise;
    },
    enabled: term.length > 2, // Só busca se tiver pelo menos 3 caracteres
  });
};
```

## Exemplos Práticos

### Formulário com Validação e Submit:

```tsx
// src/components/CreatePostForm.tsx
import React from "react";
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
import api from "../api/axios";

// Schema de validação
const postSchema = z.object({
  title: z.string().min(3, { message: "Título deve ter pelo menos 3 caracteres" }),
  body: z.string().min(10, { message: "Conteúdo deve ter pelo menos 10 caracteres" }),
});

type PostFormData = z.infer<typeof postSchema>;

export const CreatePostForm = () => {
  const queryClient = useQueryClient();
  const {
    register,
    handleSubmit,
    reset,
    formState: { errors },
  } = useForm<PostFormData>({
    resolver: zodResolver(postSchema),
  });

  const createPost = useMutation({
    mutationFn: (data: PostFormData) => api.post("/posts", data),
    onSuccess: () => {
      // Invalidar e refetch as queries relacionadas
      queryClient.invalidateQueries({ queryKey: ["posts"] });
      reset(); // Resetar formulário
    },
  });

  const onSubmit = (data: PostFormData) => {
    createPost.mutate(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <label htmlFor="title">Título</label>
        <input id="title" {...register("title")} />
        {errors.title && <p className="error">{errors.title.message}</p>}
      </div>

      <div>
        <label htmlFor="body">Conteúdo</label>
        <textarea id="body" {...register("body")} rows={5} />
        {errors.body && <p className="error">{errors.body.message}</p>}
      </div>

      <button type="submit" disabled={createPost.isPending}>
        {createPost.isPending ? "Enviando..." : "Criar Post"}
      </button>

      {createPost.isError && <p className="error">Erro ao criar post: {createPost.error.message}</p>}
    </form>
  );
};
```

### Implementação de Pesquisa com Debounce:

```tsx
// src/components/SearchComponent.tsx
import React, { useState, useEffect } from "react";
import { useQuery } from "@tanstack/react-query";
import api from "../api/axios";
import debounce from "lodash/debounce";

interface SearchResult {
  id: number;
  title: string;
  description: string;
}

export const SearchComponent = () => {
  const [searchTerm, setSearchTerm] = useState("");
  const [debouncedTerm, setDebouncedTerm] = useState("");

  // Configurar debounce para evitar múltiplas requisições
  useEffect(() => {
    const handler = debounce(() => {
      setDebouncedTerm(searchTerm);
    }, 500);

    handler();

    return () => {
      handler.cancel();
    };
  }, [searchTerm]);

  // Realizar a busca apenas com o termo debounced
  const { data, isLoading, isError } = useQuery({
    queryKey: ["search-results", debouncedTerm],
    queryFn: async () => {
      if (!debouncedTerm.trim()) return [];
      const { data } = await api.get<SearchResult[]>("/search", {
        params: { q: debouncedTerm },
      });
      return data;
    },
    enabled: debouncedTerm.trim().length > 2,
  });

  const handleSearch = (e: React.ChangeEvent<HTMLInputElement>) => {
    setSearchTerm(e.target.value);
  };

  return (
    <div>
      <input type="text" placeholder="Buscar..." value={searchTerm} onChange={handleSearch} />

      {isLoading && <div>Buscando...</div>}
      {isError && <div>Erro ao realizar busca</div>}

      {data && data.length > 0 ? (
        <ul>
          {data.map((item) => (
            <li key={item.id}>
              <h3>{item.title}</h3>
              <p>{item.description}</p>
            </li>
          ))}
        </ul>
      ) : debouncedTerm && !isLoading ? (
        <div>Nenhum resultado encontrado para "{debouncedTerm}"</div>
      ) : null}
    </div>
  );
};
```

### Componente de Status de Conexão:

```tsx
// src/components/ConnectionStatus.tsx
import { useQueryClient } from "@tanstack/react-query";
import { useEffect, useState } from "react";

export const ConnectionStatus = () => {
  const queryClient = useQueryClient();
  const [isOnline, setIsOnline] = useState(navigator.onLine);

  useEffect(() => {
    const handleOnline = () => {
      setIsOnline(true);
      // Refetch todas as queries ativas quando a conexão for restaurada
      queryClient.refetchQueries();
    };

    const handleOffline = () => {
      setIsOnline(false);
    };

    window.addEventListener("online", handleOnline);
    window.addEventListener("offline", handleOffline);

    return () => {
      window.removeEventListener("online", handleOnline);
      window.removeEventListener("offline", handleOffline);
    };
  }, [queryClient]);

  return (
    <div className={`connection-status ${isOnline ? "online" : "offline"}`}>
      {isOnline ? "✅ Online" : "❌ Offline"}
    </div>
  );
};
```

## Conclusão

O TanStack Query (@tanstack/react-query) é uma biblioteca poderosa para gerenciamento de estado assíncrono em aplicações React. Ele simplifica significativamente o gerenciamento de dados do servidor, oferecendo recursos avançados de cache, sincronização de estado e revalidação.

Ao estruturar adequadamente seus hooks, serviços e componentes, você pode criar uma arquitetura de dados robusta, eficiente e fácil de manter.

Principais pontos a lembrar:

1. Use QueryClientProvider para configuração global
2. Organize suas consultas com chaves estruturadas
3. Aproveite o cache e a revalidação automática
4. Utilize otimizações como prefetching e cache-invalidation
5. Implemente mutações com estratégias otimistas quando apropriado
6. Considere as particularidades de SSR ao usar com Next.js

Mantenha-se atualizado com a [documentação oficial](https://tanstack.com/query/latest) para aproveitar as mais recentes funcionalidades e melhores práticas.
