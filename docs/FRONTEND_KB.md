# FRONTEND_KB.md — saas-driving-school-web

Base de conhecimento para implementar o front-end do SaaS de gestão de escolas de condução (Angola): **React + TypeScript + Vite**, elegante, com **tema claro e escuro**, identidade visual por escola e consumo da API descrita em `BACKEND_KB.md`.

---

## 0. Como usar este documento (instruções para o Codex)

1. Leia este documento inteiro. Os contratos de API (rotas, permissões, formas JSON) estão no anexo C e em `BACKEND_KB.md`; se divergirem, vale o `BACKEND_KB.md`.
2. Implemente por fases (secção 17). Cada fase termina com testes e critérios de aceitação.
3. Se algo não estiver especificado, escolha a opção mais simples que respeite as regras abaixo e registe a decisão em `docs/DECISOES.md`.
4. Ortografia do Acordo Ortográfico em todos os textos (ativo, ação, exceção); português de Angola (pt-AO); frases em *sentence case*.

### Regras invioláveis

- **F1. Service → Hook → Component.** Componentes nunca chamam `axios`/`fetch`. Cada funcionalidade tem `services/` (HTTP puro), `hooks/` (TanStack Query) e `components/`/`pages/`.
- **F2. Dados do servidor só via TanStack Query.** Sem cópias em stores globais. Estado de UI local ou na URL.
- **F3. Permissões no UI** escondem/desativam ações (`<Pode>`, guards de rota) mas **nunca substituem** a validação da API (que responde 403/404).
- **F4. Toda a lista é paginada no servidor** e o **estado da lista vive na URL** (`page`, `limit`, `ordenar`, `ordem`, `search`, filtros), para que voltar, recarregar e partilhar links funcione.
- **F5. Claro e escuro desde o primeiro ecrã.** Nenhuma cor fixa (`#fff`, `text-gray-500`, `bg-white`...) em componentes; apenas tokens semânticos (secção 6).
- **F6. Identidade da escola em tempo de execução:** cores, tema padrão, fonte e logótipo vêm de `/auth/me` (e de `/publico/escolas/:slug/identidade` no login) e aplicam-se por variáveis CSS.
- **F7. Dinheiro é `string`.** Formatar com helpers; somar no cliente só com `decimal.js`; nunca `parseFloat` para cálculo.
- **F8. Ficheiros e relatórios vêm do backend** (PDF/XLSX prontos). O front só descarrega/abre; nunca gera PDF nem Excel.
- **F9. Acessibilidade AA** (teclado, foco visível, contraste ≥ 4.5:1, rótulos, `prefers-reduced-motion`).
- **F10. Sem texto em inglês visível ao utilizador**, exceto siglas de uso corrente (PDF, TPA, NIF).

### Assunções (confirmar ou ajustar)

| Assunção | Impacto se mudar |
|---|---|
| Roteamento com **React Router** (data router). | Só a camada de rotas. |
| Tailwind CSS + shadcn/ui (Radix). Tokens como variáveis CSS HSL e modo escuro por classe `.dark` em `<html>`. | Configuração do tema. |
| Fonte padrão **Plus Jakarta Sans** (fontsource, sem CDN). | Só tokens de tipografia. |
| Refresh token em cookie `httpOnly` (o front só guarda o access token em memória). | Fluxo de autenticação. |
| Formato habitual do BI angolano (9 dígitos + 2 letras + 3 dígitos) usado só como **dica**, sem bloquear. | Validação suave. |

---

## 1. Produto e princípios de UX

**Quem usa:** secretaria e gestores (desktop, uso intenso), financeiro, coordenação pedagógica, instrutores (muitas vezes no telemóvel), administradores. O estudante **ainda não** usa o sistema (portal é fase futura, mas a UI não deve impedi-lo).

**Tarefas principais:** (1) encontrar um estudante em segundos; (2) ver em que ponto da formação ele está; (3) agendar aulas sem conflitos; (4) registar e consultar pagamentos; (5) filtrar turmas por **ano** e ver a ocupação; (6) emitir documentos e exportar relatórios com a marca da escola.

Princípios: calma e densidade informativa (é uma ferramenta de trabalho); a **pesquisa de estudante está sempre à mão**; ações destrutivas pedem confirmação e motivo; todo o estado vazio diz o que fazer a seguir; nada de decoração sem função.

---

## 2. Stack e decisões fixas

| Área | Decisão |
|---|---|
| Build | Vite + React (versão estável atual) + TypeScript `strict` |
| Rotas | React Router (`createBrowserRouter`, rotas com `lazy`) |
| Dados | TanStack Query v5 (`placeholderData: keepPreviousData` nas listas) |
| Tabelas | TanStack Table v8 com paginação e ordenação **manuais** (servidor) |
| UI | shadcn/ui sobre Radix + Tailwind CSS; ícones `lucide-react`; toasts `sonner`; paleta de comandos `cmdk` (componente `Command` do shadcn) |
| Formulários | React Hook Form + Zod (`@hookform/resolvers`) |
| HTTP | `axios` (uma instância, interceptors) |
| Gráficos | Recharts (via componentes de gráfico do shadcn), carregado em *lazy* |
| Datas | `date-fns` + locale `pt`; `date-fns-tz` para `Africa/Luanda` |
| Dinheiro | `decimal.js` |
| Fonte | `@fontsource-variable/plus-jakarta-sans` (+ alternativas permitidas na secção 6.4, carregadas sob demanda) |
| Testes | Vitest + Testing Library + MSW; `jest-axe`; Playwright (fumo) |
| Qualidade | ESLint (regra de imports proibidos: componentes não importam `axios`), Prettier, `tsc --noEmit` |

Variáveis de ambiente (`.env.example`):
```
VITE_API_URL=http://localhost:3000/api/v1
VITE_BASE_DOMAIN=            # ex.: escolas.ao  (vazio = sem subdomínio; usar ?escola=slug)
```

---

## 3. Estrutura de pastas

```
src/
├─ main.tsx, App.tsx
├─ app/
│  ├─ router.tsx                  # rotas + lazy + guards
│  ├─ providers.tsx               # QueryClient, Tema, Auth, Toaster
│  └─ layouts/                    # AppShell.tsx, AuthLayout.tsx, PlataformaShell.tsx
├─ components/
│  ├─ ui/                         # shadcn (button, input, dialog, sheet, tabs, ...)
│  └─ shared/                     # DataTable, FiltrosLista, BadgeEstado, BotaoExportar, Pode, InputDinheiro,
│                                 # CampoData, SeletorAno, EstadoVazio, EstadoErro, ConfirmarAcao, PageHeader,
│                                 # AvatarEstudante, ForcaSenha, PesquisaGlobal
├─ features/
│  ├─ auth/  onboarding/  escola/  acesso/  formacao/  estudantes/  matriculas/
│  ├─ turmas/  aulas/  exames/  pessoal/  frota/  financeiro/  documentos/
│  ├─ relatorios/  dashboard/  notificacoes/  auditoria/  plataforma/
│  └─ <feature>/
│     ├─ services/                # <feature>.service.ts   (HTTP puro, tipado)
│     ├─ hooks/                   # use-<feature>.ts       (queries/mutations + chaves)
│     ├─ components/              # componentes da feature
│     ├─ pages/                   # páginas (compõem hooks + componentes)
│     ├─ schemas/                 # Zod
│     └─ types.ts
├─ lib/
│  ├─ http.ts                     # axios + interceptors
│  ├─ erros.ts                    # normalizarErro, mensagens por código
│  ├─ formatar.ts                 # kz(), data(), hora(), telefone(), nomeCurto()
│  ├─ download.ts                 # baixarFicheiro()
│  ├─ identidade/                 # cores.ts (WCAG), tokens.ts, aplicar.ts, fontes.ts
│  ├─ url-state.ts                # useListaUrl()
│  └─ permissoes.ts               # P (códigos), usePode(), temPermissao()
├─ types/                         # api.ts (Paginado, ErroApi), enums.ts (anexo A)
└─ styles/                        # globals.css (tokens)
```

---

## 4. Arquitetura Service → Hook → Component

```ts
// lib/http.ts
import axios, { AxiosError, InternalAxiosRequestConfig } from 'axios';
import { normalizarErro } from './erros';
import type { ErroApi } from '@/types/api';

export const http = axios.create({ baseURL: import.meta.env.VITE_API_URL, withCredentials: true });

let accessToken: string | null = null;
export const definirToken = (t: string | null) => { accessToken = t; };

http.interceptors.request.use((cfg) => {
  if (accessToken) cfg.headers.Authorization = `Bearer ${accessToken}`;
  return cfg;
});

let refreshEmCurso: Promise<string> | null = null;

http.interceptors.response.use(undefined, async (erro: AxiosError<ErroApi>) => {
  const original = erro.config as (InternalAxiosRequestConfig & { _retry?: boolean }) | undefined;
  const renovavel = erro.response?.status === 401 && !!original && !original._retry && !original.url?.startsWith('/auth/');
  if (!renovavel || !original) return Promise.reject(normalizarErro(erro));

  original._retry = true;
  refreshEmCurso ??= http
    .post<{ accessToken: string }>('/auth/refresh')
    .then((r) => r.data.accessToken)
    .finally(() => { refreshEmCurso = null; });
  try {
    const token = await refreshEmCurso;
    definirToken(token);
    original.headers.Authorization = `Bearer ${token}`;
    return http(original);
  } catch {
    definirToken(null);
    window.dispatchEvent(new CustomEvent('sessao-expirada'));
    return Promise.reject(normalizarErro(erro));
  }
});
```

```ts
// features/turmas/services/turmas.service.ts   (HTTP puro, sem React)
import { http } from '@/lib/http';
import type { Paginado } from '@/types/api';
import type { Turma, TurmasQuery, AnoTurmas, CriarTurmaDto } from '../types';

export const turmasService = {
  listar: (q: TurmasQuery) => http.get<Paginado<Turma>>('/turmas', { params: q }).then((r) => r.data),
  anos: () => http.get<AnoTurmas[]>('/turmas/anos').then((r) => r.data),
  obter: (id: string) => http.get<Turma>(`/turmas/${id}`).then((r) => r.data),
  criar: (dto: CriarTurmaDto) => http.post<Turma>('/turmas', dto).then((r) => r.data),
};
```

```ts
// features/turmas/hooks/use-turmas.ts   (chaves + queries + mutations)
import { keepPreviousData, useMutation, useQuery, useQueryClient } from '@tanstack/react-query';
import { toast } from 'sonner';
import { turmasService } from '../services/turmas.service';
import type { TurmasQuery } from '../types';

export const turmasKeys = {
  todas: ['turmas'] as const,
  lista: (q: TurmasQuery) => [...turmasKeys.todas, 'lista', q] as const,
  anos: () => [...turmasKeys.todas, 'anos'] as const,
  detalhe: (id: string) => [...turmasKeys.todas, 'detalhe', id] as const,
};

export const useTurmas = (q: TurmasQuery) =>
  useQuery({ queryKey: turmasKeys.lista(q), queryFn: () => turmasService.listar(q), placeholderData: keepPreviousData });

export const useAnosTurmas = () =>
  useQuery({ queryKey: turmasKeys.anos(), queryFn: turmasService.anos, staleTime: 5 * 60_000 });

export const useCriarTurma = () => {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: turmasService.criar,
    onSuccess: () => { qc.invalidateQueries({ queryKey: turmasKeys.todas }); toast.success('Turma criada'); },
  });
};
```

Regras do padrão:
- Páginas compõem hooks e componentes; **sem** `useEffect` para buscar dados.
- Chaves de query por feature (`xKeys`), sempre incluindo os filtros. Mutations invalidam a chave raiz da feature e as dependentes (ex.: pagamento invalida `financeiro`, `estudantes` e o resumo do estudante).
- Erros de mutation: `onError` mostra toast com a mensagem do código (secção 5.4) e, em erros de validação (422), mapeia `detalhes.campos` para os campos do formulário (`setError`).
- Optimistic updates apenas para ações triviais e reversíveis (marcar notificação como lida). Nunca para dinheiro, agenda ou matrículas.

---

## 5. Contrato com a API

### 5.1 Autenticação

1. `POST /auth/login` → `{ accessToken }` e cookie `rt` (httpOnly). O token fica **só em memória** (`definirToken`).
2. No arranque da app: `POST /auth/refresh` (cookie) → token; depois `GET /auth/me`. Enquanto isto corre, mostrar *splash* (logótipo neutro). Falha → `/login`.
3. 401 em pedidos normais: interceptor faz **um** refresh partilhado e repete o pedido (secção 4). Falha do refresh → evento `sessao-expirada` → limpar cache do Query, ir para `/login?expirou=1` com aviso.
4. `deveAlterarSenha === true` → forçar rota `/primeiro-acesso` (ecrã único de alteração de senha; mesmas regras de força que a API).
5. Logout: `POST /auth/logout`, limpar token e `queryClient.clear()`.
6. `/auth/me` alimenta: utilizador, escola (identidade, onboarding), filiais e **permissões** `{ codigo, escopo, filiais }`.

### 5.2 Tipos base

```ts
// types/api.ts
export interface Paginado<T> {
  data: T[];
  meta: { page: number; limit: number; total: number; totalPages: number; hasNext: boolean; hasPrev: boolean };
}
export interface ErroApi {
  statusCode: number;
  codigo: string;
  mensagem: string;
  detalhes?: { campos?: { campo: string; mensagem: string }[]; [k: string]: unknown };
  requestId?: string;
}
export type Dinheiro = string;      // "22500.00"
export type DataISO = string;       // "2026-03-02"
export type DataHoraISO = string;   // "2026-03-02T09:00:00+01:00"
export type Hora = string;          // "08:00"
```

### 5.3 Permissões no front

```ts
// lib/permissoes.ts (esqueleto)
export type Escopo = 'ESCOLA' | 'FILIAL' | 'PROPRIO';
export interface ConcessaoUI { codigo: string; escopo: Escopo; filiais: string[] | null }

export const temPermissao = (mapa: Map<string, ConcessaoUI>, ...codigos: string[]) => codigos.every((c) => mapa.has(c));
export const usePode = (...codigos: string[]) => { /* lê o mapa do AuthContext (construído de /auth/me) */ };
```
```tsx
// <Pode permissao={P.PAGAMENTO_REGISTAR}> ...botão... </Pode>
// <Pode permissao={[P.PAGAMENTO_ANULAR]} modo="desativar" dica="Sem permissão para anular pagamentos"> ... </Pode>
```
`modo="esconder"` (omissão) remove do DOM; `modo="desativar"` mostra desativado com dica. As rotas usam `<RotaProtegida permissao={...}>` e mostram a página 403 amigável.

### 5.4 Erros: código → mensagem

`normalizarErro` devolve `ErroApi` (mesmo para erros de rede: `codigo: 'SEM_REDE'`). Mensagens para o utilizador (usar `mensagem` da API quando existir; estas são o *fallback* e o texto de UI):

| Código | Mensagem |
|---|---|
| `SEM_REDE` | Sem ligação ao servidor. Verifique a internet e tente de novo. |
| `CREDENCIAIS_INVALIDAS` | Email ou senha incorretos. |
| `SEM_PERMISSAO` | Não tem permissão para esta ação. |
| `NAO_ENCONTRADO` | Este registo já não existe ou não está disponível. |
| `ESCOLA_SUSPENSA` | O acesso desta escola está suspenso. Contacte o suporte. |
| `CONFLITO_AGENDA_INSTRUTOR` | O instrutor já tem uma aula neste horário. |
| `CONFLITO_AGENDA_VEICULO` | O veículo já está reservado neste horário. |
| `CONFLITO_AGENDA_ESTUDANTE` | O estudante já tem uma aula neste horário. |
| `FORA_DA_JANELA` | Este horário está fora das janelas permitidas para aulas práticas. |
| `PAGAMENTO_EM_ATRASO` | O estudante tem pagamentos em atraso. Regularize antes de agendar. |
| `TURMA_SEM_VAGAS` | A turma não tem vagas livres. |
| `CATEGORIA_INCOMPATIVEL` | A categoria da turma é diferente da categoria da matrícula. |
| `MATRICULA_JA_EXISTE` | O estudante já tem uma matrícula em curso nesta categoria. |
| `ESTUDANTE_DOC_DUPLICADO` | Já existe um estudante com este documento de identificação. |
| `TURMA_CODIGO_DUPLICADO` | Já existe uma turma com este código. |
| `INELEGIVEL_EXAME` | O estudante ainda não cumpre os requisitos para este exame. |
| `VALOR_NAO_ALOCADO` | O valor pago tem de ser totalmente distribuído pelas cobranças. |
| `ALOCACAO_EXCEDE_SALDO` | O valor atribuído a uma cobrança é superior ao que falta pagar. |
| `COBRANCA_COM_PAGAMENTOS` | Anule primeiro os pagamentos desta cobrança. |
| `ULTIMO_ADMINISTRADOR` | A escola tem de ter pelo menos um administrador ativo. |
| `ESCALADA_DE_PRIVILEGIOS` | Só pode atribuir permissões que você próprio tem. |
| `EXPORTACAO_DEMASIADO_GRANDE` | Resultado demasiado grande para exportar. Aplique filtros e tente de novo. |
| `FICHEIRO_INVALIDO` | Ficheiro não aceite. Use PDF, JPG, PNG ou WEBP até 5 MB. |
| `CONFLITO_CONCORRENCIA` | Outra pessoa alterou este registo ao mesmo tempo. Tente de novo. |
| `NAO_IMPLEMENTADO` | Esta funcionalidade ainda não está disponível. |
| *(outros)* | Algo correu mal. Tente de novo; se persistir, indique o código `requestId`. |

### 5.5 Dinheiro, datas e telefones

```ts
// lib/formatar.ts
export const kz = (valor: string | number): string =>
  `${new Intl.NumberFormat('pt-AO', { minimumFractionDigits: 2, maximumFractionDigits: 2 }).format(Number(valor))} Kz`;
// datas: format(d, 'dd/MM/yyyy', { locale: pt }); horas: 'HH:mm'; fuso de exibição: Africa/Luanda
```
`InputDinheiro` mantém o valor como **string** `"10000.00"` no formulário e mostra formatado ao sair do campo. Telefone: aceitar `+244`/espaços; guardar só dígitos (9 dígitos). BI: dica suave se `docTipo = BI` e não bater `^\d{9}[A-Z]{2}\d{3}$` ("Formato habitual: 9 dígitos, 2 letras, 3 dígitos"), **sem bloquear**.

### 5.6 Descarregar ficheiros

```ts
// lib/download.ts
import { http } from './http';

function nomeDoCabecalho(cd?: string): string | undefined {
  if (!cd) return undefined;
  const utf8 = /filename\*=UTF-8''([^;]+)/i.exec(cd);
  if (utf8) return decodeURIComponent(utf8[1]);
  return /filename="?([^";]+)"?/i.exec(cd)?.[1];
}

export async function baixarFicheiro(url: string, opcoes?: { params?: object; metodo?: 'GET' | 'POST'; corpo?: object; abrir?: boolean }) {
  const resp = await http.request<Blob>({ url, method: opcoes?.metodo ?? 'GET', params: opcoes?.params, data: opcoes?.corpo, responseType: 'blob' });
  const nome = nomeDoCabecalho(resp.headers['content-disposition']) ?? 'ficheiro';
  const objeto = URL.createObjectURL(resp.data);
  if (opcoes?.abrir) {
    window.open(objeto, '_blank', 'noopener');
  } else {
    const a = document.createElement('a');
    a.href = objeto; a.download = nome; document.body.appendChild(a); a.click(); a.remove();
  }
  setTimeout(() => URL.revokeObjectURL(objeto), 60_000);
  return { nome, numero: resp.headers['x-documento-numero'] as string | undefined, id: resp.headers['x-documento-id'] as string | undefined };
}
```
Erros em `responseType: 'blob'` chegam como `Blob`: converter com `await blob.text()` + `JSON.parse` dentro de `normalizarErro`.

---

## 6. Design system

### 6.1 Direção visual

Ferramenta de trabalho serena, no espírito do ecrã de referência (cartões brancos com contorno fino sobre fundo cinza-claro, *badges* em pílula, separadores segmentados, menu lateral de ações em lista): superfícies neutras e silenciosas, **cor da escola usada com contenção** (ação principal, item ativo, foco, progresso). **O elemento memorável é a Jornada do estudante** (secção 9.5): uma linha de progresso Teoria → Exame teórico → Prática → Exame prático → Carta no cabeçalho de cada estudante. Todo o resto fica discreto.

Evitar: gradientes decorativos; rótulos em MAIÚSCULAS espaçadas acima de títulos; cartões idênticos para tudo; sombras em todos os cartões; animações de entrada em cada secção; marcadores numerados fora de sequências reais; setas `→` nos botões; textos de marketing.

### 6.2 Tokens (variáveis CSS, HSL sem `hsl()`)

Aplicar em `:root` (claro) e `.dark` (escuro). Tailwind mapeia `bg-background`, `text-foreground`, `border-border`, etc. com `hsl(var(--token) / <alpha-value>)`.

| Token | Claro | Escuro |
|---|---|---|
| `--background` | `220 20% 98%` | `224 22% 8%` |
| `--foreground` | `224 25% 12%` | `220 18% 94%` |
| `--card` / `--popover` | `0 0% 100%` | `224 20% 11%` |
| `--card-foreground` / `--popover-foreground` | `224 25% 12%` | `220 18% 94%` |
| `--muted` / `--secondary` | `220 16% 95%` | `224 16% 16%` |
| `--muted-foreground` | `220 9% 40%` | `220 10% 66%` |
| `--secondary-foreground` | `224 25% 16%` | `220 18% 90%` |
| `--border` | `220 14% 90%` | `224 14% 21%` |
| `--input` | `220 14% 86%` | `224 14% 24%` |
| `--sucesso` / `--sucesso-suave` / `--sucesso-texto` | `152 62% 32%` / `152 50% 94%` / `152 62% 24%` | `152 55% 42%` / `152 40% 15%` / `152 55% 62%` |
| `--aviso` / `--aviso-suave` / `--aviso-texto` | `32 95% 38%` / `40 95% 92%` / `28 90% 28%` | `38 92% 50%` / `38 50% 15%` / `38 92% 64%` |
| `--info` / `--info-suave` / `--info-texto` | `213 85% 42%` / `213 90% 95%` / `213 85% 34%` | `213 90% 58%` / `213 50% 17%` / `213 90% 72%` |
| `--perigo` / `--perigo-suave` / `--perigo-texto` | `0 72% 46%` / `0 80% 96%` / `0 72% 38%` | `0 62% 48%` / `0 45% 17%` / `0 90% 72%` |

Tokens **dependentes da identidade** (calculados, secção 6.3): `--primary`, `--primary-foreground`, `--primary-texto`, `--primary-suave`, `--ring`, `--accent`, `--accent-foreground`, `--chart-1..5`.

Teste unitário obrigatório: para cada par `--*-texto` sobre `--*-suave` e `--*-texto` sobre `--card`, `--muted-foreground` sobre `--background` e `--card`, o contraste WCAG tem de ser **≥ 4.5**.

Forma e profundidade: `--radius: 0.75rem` para cartões e painéis (`rounded-xl`), `0.5rem` para campos e botões (`rounded-lg`), pílula (`rounded-full`) para *badges* e avatares. Cartões: contorno de 1 px (`border-border`), **sem sombra**; sombra só em *popovers*, menus, *sheets* e cabeçalho fixo (`shadow-sm`/`shadow-md`).

### 6.3 Tema por escola (identidade)

Entrada (de `/auth/me`): `corPrimaria` (obrigatória), `corSecundaria`, `corDestaque`, `temaPadrao` (`CLARO|ESCURO|SISTEMA`), `fonte`, `logoUrl`, `logoEscuroUrl`, `faviconUrl`.

```ts
// lib/identidade/cores.ts
export type RGB = [number, number, number];
const canal = (c: number) => { const s = c / 255; return s <= 0.03928 ? s / 12.92 : ((s + 0.055) / 1.055) ** 2.4; };
export const luminancia = ([r, g, b]: RGB) => 0.2126 * canal(r) + 0.7152 * canal(g) + 0.0722 * canal(b);
export const contraste = (a: RGB, b: RGB) => {
  const [maior, menor] = [luminancia(a), luminancia(b)].sort((x, y) => y - x);
  return (maior + 0.05) / (menor + 0.05);
};
```

`gerarTokens(identidade, tema): Record<string, string>` calcula, por tema:
1. `fg` = melhor entre branco `#FFFFFF` e o `--foreground` do tema, por contraste com a cor primária.
2. Se `contraste(primária, fg) ≥ 4.5` → `--primary` = cor da escola e `--primary-foreground` = `fg`. Senão ajusta a **luminosidade** (HSL, passos de 2 %) para o lado que exige menor mudança até atingir 4.5 (escurece para texto branco, clareia para texto escuro).
3. `--primary-texto` (links, item de menu ativo, números de destaque): ajusta a luminosidade até contraste ≥ 4.5 contra `--card` do tema.
4. `--primary-suave` = mistura da primária a 12 % (claro) ou 20 % (escuro) sobre `--card`; `--accent` = `--primary-suave`; `--accent-foreground` = `--primary-texto`; `--ring` = `--primary-texto`.
5. `--chart-1` = primária ajustada; `--chart-2` = `corDestaque` (ou `--info`); `--chart-3` = `--sucesso`; `--chart-4` = `--aviso`; `--chart-5` = `--perigo`.

`aplicarIdentidade(raiz, identidade, tema)` escreve as variáveis em `document.documentElement.style` (ou num contentor de pré-visualização). Testes: cores claras (amarelo `#FACC15`), escuras (`#0B1B3A`), saturadas (`#2563EB`) e cinzas produzem `--primary-foreground` com contraste ≥ 4.5 nos dois temas.

**Tema (claro/escuro):** estratégia por classe. Preferência do utilizador (`claro|escuro|sistema`) em `localStorage` (`tema`); omissão = `temaPadrao` da escola; `SISTEMA` segue `prefers-color-scheme`. Script no `index.html` antes do *bundle* para evitar *flash*:

```html
<script>
(function () {
  try {
    var t = localStorage.getItem('tema') || 'sistema';
    var escuro = t === 'escuro' || (t === 'sistema' && window.matchMedia('(prefers-color-scheme: dark)').matches);
    document.documentElement.classList.toggle('dark', escuro);
  } catch (e) {}
})();
</script>
```

**Logótipo:** claro → `logoUrl`; escuro → `logoEscuroUrl`, ou, se não existir, `logoUrl` sobre uma "ficha" `bg-white rounded-md p-1`. Sem logótipo → iniciais da escola num círculo `--primary-suave`. `favicon` e `<title>` = nome da escola.

### 6.4 Tipografia

Uma família (variável) para tudo; números com `font-variant-numeric: tabular-nums` (valores, datas, nº de estudante, tabelas). Famílias permitidas na identidade (`fonte`): **Plus Jakarta Sans** (omissão), Inter, DM Sans, Manrope, Source Sans 3; carregar por *import* dinâmico do pacote *fontsource* correspondente; qualquer outro valor → omissão; *fallback* `ui-sans-serif, system-ui, sans-serif`.

| Papel | Tamanho / altura | Peso |
|---|---|---|
| Título de página | 24 / 32 | 600 |
| Título de secção | 18 / 28 | 600 |
| Título de cartão | 15 / 22 | 600 |
| Corpo e tabelas | 14 / 20 | 400 |
| Apoio, rótulos de campo | 13 / 18 | 500 (rótulos) |
| Legendas | 12 / 16 | 400 |

Largura de texto corrido ≤ 72 caracteres. Rótulos e títulos em *sentence case*, nunca em maiúsculas espaçadas.

### 6.5 Componentes base e variantes

- **Badge de estado** (`BadgeEstado`): recebe `dominio` (ex.: `matricula.estado`) e `valor`; usa o mapa de rótulos/variantes do anexo A (`ROTULOS`). Variantes: `neutro` (muted), `info`, `sucesso`, `aviso`, `perigo` (cada uma com fundo `*-suave` e texto `*-texto`), `discreto` (contorno, texto `muted-foreground`). **Nunca só cor:** o texto do rótulo está sempre presente.
- **Botões:** primário (cor da escola), secundário (contorno), fantasma, destrutivo (`--perigo`); altura 36 px (40 px em ecrãs táteis); ícone + texto quando o ícone ajuda.
- **Cartões:** `rounded-xl border bg-card`; cabeçalho de cartão com título e ação à direita.
- **Tabs:** estilo segmentado (como no ecrã de referência): fundo `--muted`, item ativo `--card` com contorno.
- **Menu lateral de ações** (lista vertical com ícone + texto, item ativo `--primary-suave`/`--primary-texto`).
- **Feedback:** toasts (`sonner`) com o mesmo verbo do botão ("Guardar alterações" → "Alterações guardadas"); diálogos de confirmação com consequência explícita; ações destrutivas pedem **motivo** quando a API o exige.
- **Foco:** `focus-visible:ring-2 ring-ring ring-offset-2 ring-offset-background`.

### 6.6 Movimento

Só em resposta a ações (abrir *sheet*/diálogo, trocar de separador, toast) com 150–200 ms. **Um único momento orquestrado:** ao abrir o ecrã de um estudante, a linha da Jornada preenche-se em 400 ms. Respeitar `prefers-reduced-motion` (sem animação).

---

## 7. Estrutura da aplicação (shell)

Referência: barra superior com botão de recolher o menu à esquerda, **pesquisa de estudante** logo ao lado, e à direita alternador de tema, sino de notificações e utilizador (nome + email).

- **Barra superior (64 px):** botão menu; `PesquisaGlobal`; `AlternadorTema` (claro/escuro/sistema); sino (secção 9.13); menu do utilizador (perfil/alterar senha/terminar sessão).
- **Menu lateral:** 264 px expandido, 72 px recolhido (ícones + dica); em ecrãs < `lg` torna-se *drawer*. Itens agrupados com títulos discretos em *sentence case*; cada item só aparece com a permissão indicada. Estado recolhido guardado em `localStorage`.
- **Cabeçalho de página:** *breadcrumbs* (ícone de início › secção › página), título, descrição curta opcional e ações à direita (`PageHeader`).
- **Conteúdo:** largura máxima 1440 px, espaçamento 24 px; fundo `--background`.

| Grupo | Itens (permissão) |
|---|---|
| Início | Painel (`dashboard:ver`) |
| Estudantes | Estudantes (`estudante:ver`), Matrículas (`matricula:ver`) |
| Formação | Turmas (`turma:ver`), Aulas teóricas (`aula_teorica:ver`), Agenda prática (`aula_pratica:ver`), Avaliações (`avaliacao:ver`), Exames (`exame:ver`) |
| Equipa e frota | Instrutores (`instrutor:ver`), Funcionários (`funcionario:ver`), Veículos (`veiculo:ver`) |
| Financeiro | Cobranças (`cobranca:ver`), Pagamentos (`pagamento:ver`), Despesas (`despesa:ver`), Resumo (`financeiro_relatorio:ver`) |
| Relatórios | Relatórios (`relatorio:ver`) |
| Administração | Utilizadores (`utilizador:ver`), Perfis (`perfil:ver`), Configurações da escola (`escola:ver`), Auditoria (`auditoria:ver`) |

Utilizadores `PLATAFORMA` usam outro shell (`/plataforma`) com Escolas e Planos; não veem os grupos acima.

---

## 8. Rotas

| Rota | Página | Permissão |
|---|---|---|
| `/login` | Login (identidade pública da escola) | público |
| `/primeiro-acesso` | Alterar senha obrigatória | autenticado |
| `/onboarding` | Assistente de configuração (9.1) | `escola:editar` |
| `/` | Painel (9.2) | `dashboard:ver` |
| `/estudantes` | Lista de estudantes (9.3) | `estudante:ver` |
| `/estudantes/novo` | Nova matrícula / novo estudante (9.6) | `estudante:criar` |
| `/estudantes/:id` → `perfil` (omissão), `documentos`, `financas`, `avaliacoes`, `progresso` | Ecrã do estudante (9.4–9.5) | `estudante:ver` (+ permissão de cada separador) |
| `/matriculas` | Lista de matrículas | `matricula:ver` |
| `/turmas` | Lista de turmas com filtro de **ano** (9.7) | `turma:ver` |
| `/turmas/nova` | Criar turma | `turma:criar` |
| `/turmas/:id` | Detalhe da turma: `estudantes`, `aulas`, `informacao` (9.8) | `turma:ver` |
| `/aulas-teoricas` | Lista de aulas teóricas | `aula_teorica:ver` |
| `/agenda` | Agenda de aulas práticas (9.9) | `aula_pratica:ver` |
| `/avaliacoes` | Lista de avaliações | `avaliacao:ver` |
| `/exames`, `/exames/sessoes/:id` | Sessões e inscrições (9.10) | `exame:ver` |
| `/instrutores`, `/instrutores/:id` | Lista e detalhe (agenda, documentos) | `instrutor:ver` |
| `/funcionarios` | Lista | `funcionario:ver` |
| `/veiculos`, `/veiculos/:id` | Frota (manutenções, abastecimentos, documentos) | `veiculo:ver` |
| `/cobrancas`, `/pagamentos`, `/despesas` | Listas financeiras | `cobranca:ver` / `pagamento:ver` / `despesa:ver` |
| `/financeiro` | Resumo financeiro (9.11) | `financeiro_relatorio:ver` |
| `/relatorios` | Catálogo e exportação (9.12) | `relatorio:ver` |
| `/utilizadores`, `/utilizadores/:id` | Utilizadores e acesso (9.14) | `utilizador:ver` |
| `/perfis`, `/perfis/:id` | Perfis e matriz de permissões (9.14) | `perfil:ver` |
| `/configuracoes` → `escola`, `identidade`, `regras`, `janelas`, `categorias`, `curriculos`, `pacotes`, `documentos`, `filiais` | Configurações (9.15) | `escola:ver` (+ cada edição) |
| `/auditoria` | Auditoria | `auditoria:ver` |
| `/notificacoes` | Lista completa | `notificacao:ver` |
| `/plataforma/escolas`, `/plataforma/planos` | Equipa do SaaS | `plataforma` |

Guards: sem sessão → `/login`; `deveAlterarSenha` → `/primeiro-acesso`; escola com `onboardingConcluido === false` → utilizadores com `escola:editar` vão para `/onboarding`, os restantes veem "A escola ainda está a ser configurada"; sem permissão → página 403 com atalho para o painel.

---

## 9. Ecrãs

### 9.1 Onboarding da escola (identidade)

Assistente em passos, guardando cada passo ao avançar (e permitindo voltar). Progresso visível (indicador de passos). Pré-visualização ao vivo à direita em ecrãs largos.

| Passo (chave da API) | Campos | Chamadas |
|---|---|---|
| **Dados da escola** (`DADOS_ESCOLA`) | nome*, nome curto, NIF, email, telefone, website, endereço, província, município, alvará (nº, validade) | `PATCH /escola` |
| **Identidade** (`IDENTIDADE`) | logótipo (claro)*, logótipo (escuro), ícone (favicon), assinatura, carimbo; cor primária*, secundária, de destaque; tema padrão; fonte; cabeçalho e rodapé dos documentos | `PUT /escola/identidade` + `POST /escola/identidade/ficheiros/:tipo` |
| **Filial principal** (`FILIAL`) | nome*, código*, endereço, telefone, email | `GET /filiais` → `PATCH /filiais/:id` (a filial principal já existe) |
| **Categorias** (`CATEGORIAS`) | escolher sugestões (A, B, C, D, E) e/ou criar; por categoria: nome, idade mínima, mínimo de aulas teóricas e práticas; pacote opcional (nome e preço) | `POST /categorias`, `POST /curriculos`, `POST /pacotes` |
| **Equipa** (`EQUIPA`, opcional) | convidar utilizadores com perfil | `POST /utilizadores` |
| **Revisão** (`REVISAO`) | resumo e confirmação | `POST /escola/onboarding/concluir` |

Cada passo concluído chama `POST /escola/onboarding/passos/:passo/concluir`. Os passos `DADOS_ESCOLA`, `IDENTIDADE`, `FILIAL`, `CATEGORIAS` são obrigatórios para concluir.

Passo **Identidade** em detalhe: *upload* com pré-visualização (PNG, JPEG ou WEBP, ≤ 2 MB; **SVG não**), seletor de cor com campo hexadecimal, indicador de contraste, seletor de tema padrão (claro/escuro/sistema) e de fonte (lista da secção 6.4). A pré-visualização mostra um mini-ecrã (barra superior, menu, botão primário, badge, cartão) **em claro e escuro lado a lado**, usando `aplicarIdentidade` num contentor isolado (sem mexer no tema real até guardar), e uma amostra do **cabeçalho de documento** (logótipo, nome, NIF, endereço, linha na cor primária, rodapé).

### 9.2 Painel

Dados: `GET /dashboard`. Faixa de indicadores (Estudantes ativos, Turmas em curso, Aulas hoje, Recebido no mês, Valor em atraso) — números grandes só aqui, sem gradientes; lista **Hoje** (aulas do dia com hora, estudante, instrutor, veículo e estado; clique abre a aula na agenda); lista **Atenção** com contadores acionáveis (documentos a expirar, cobranças em atraso, turmas sem instrutor, licenças a expirar) que levam à lista filtrada respetiva. Blocos cujo valor vier `null` não aparecem.

### 9.3 Lista de estudantes

`GET /estudantes`. Barra de filtros: pesquisa (`search`, 300 ms de *debounce*), estado, categoria, **ano da turma** (`turmaAno`, `SeletorAno`), turma, filial (só se houver > 1), interruptor "Só em atraso"; botão "Limpar filtros"; ações: **Nova matrícula** (primário), **Exportar** (PDF/Excel com os filtros atuais).

Colunas: Estudante (avatar + nome + nº), Documento (BI), Categoria (badge), Turma (código · ano), Estado (badge), Situação financeira ("Em dia" ou valor em atraso em `perigo`), Contacto. Ordenação por nome, número, data de criação. Clique na linha abre `/estudantes/:id`. Menu de linha: Abrir, Nova matrícula, Registar pagamento (com permissão).

### 9.4 Ecrã do estudante

Dados: `GET /estudantes/:id/resumo` (cabeçalho) e, por separador, os endpoints do anexo C.

**Cabeçalho** (cartão, como na referência): avatar (foto ou iniciais com anel `--primary-suave`), nome (título), *badges* (estado, categoria, turma · ano, média), grelha 3×2 de dados com ícone: Nº de estudante, Categoria, Turma, Telefone, Email, Localidade (município, província). Abaixo, a **Jornada** (9.5). Se não houver matrícula: chamada para "Criar matrícula".

**Separadores** (segmentados; só aparecem com permissão): **Perfil**, **Documentos**, **Finanças**, **Avaliações**, **Progresso**. URL: `/estudantes/:id/<separador>`.

**Perfil** — layout de duas colunas como a referência: à esquerda **menu de ações** (lista), à direita o painel da ação escolhida:

| Ação do menu | Painel | API / permissão |
|---|---|---|
| Dados pessoais | formulário completo | `PATCH /estudantes/:id` / `estudante:editar` |
| Contacto | telefone, email, morada, contacto de emergência | `PATCH /estudantes/:id` |
| Matrículas | lista de matrículas (estado, categoria, turma) | `GET /estudantes/:id/matriculas` / `matricula:ver` |
| Nova matrícula | abre o assistente (9.6) com o estudante pré-selecionado | `POST /matriculas` / `matricula:criar` |
| Inscrever em turma / Mudar de turma | escolher turma (filtro de ano, categoria igual, vagas) | `POST /turmas/:id/estudantes`, `.../transferir` / `turma_estudante:gerir` |
| Estado da matrícula | suspender, reativar, concluir, cancelar/desistir (com motivo) | `POST /matriculas/:id/suspender`, `.../reativar`, `.../concluir`, `.../cancelar` / respetivas permissões |
| Registar situação | estado do estudante (ativo, inativo, concluído, desistente) | `PATCH /estudantes/:id/estado` / `estudante:editar` |
| Mensagens | enviar notificação ao estudante | `POST /notificacoes` / `notificacao:enviar` |
| Acesso ao portal | **reservado**: item visível mas desativado, com a dica "Disponível em breve" (portal futuro) | `POST /estudantes/:id/acesso` (responde 501) |
| Eliminar estudante | confirmação com texto de consequência | `DELETE /estudantes/:id` / `estudante:eliminar` |

**Documentos** — duas áreas: (1) **Checklist de anexos** (`GET /estudantes/:id/anexos`): tipo, obrigatório, estado (`Por validar`/`Validado`/`Rejeitado`/`Em falta`), validade (aviso "expira em N dias" a < 30 dias; `perigo` se expirado), ações: carregar/substituir, ver (descarrega), validar/rejeitar (`anexo:validar`), eliminar; tipos sensíveis sem permissão aparecem mascarados ("Documento reservado"). (2) **Documentos emitidos**: botões para gerar Ficha, Declaração de matrícula, Declaração de frequência, Contrato, Certificado (só se matrícula concluída) e Extrato; cada geração chama `POST /documentos/gerar`, abre o PDF num separador e acrescenta-o à lista (`GET /estudantes/:id/documentos-emitidos`) com nº, data, estado e ações (descarregar, anular com motivo).

**Finanças** — quatro números de resumo (Valor líquido, Pago, Em aberto, Em atraso — `GET /estudantes/:id/financas/resumo`); tabela de **cobranças** (descrição, prestação n/total, vencimento, valor, pago, saldo, situação); tabela de **pagamentos** (nº do recibo, data, método, valor, estado; ações: recibo PDF, anular). **Registar pagamento** abre um *sheet*: valor, método, referência (obrigatória exceto numerário), data; alocação **automática** (omissão, por vencimento) ou **manual** (tabela das cobranças abertas com campo de valor por linha, total alocado vs valor pago em tempo real; só submete quando coincidem); `Idempotency-Key` gerada por tentativa. Após sucesso: toast com o nº do recibo e ação "Abrir recibo".

**Avaliações** — média em destaque (`mediaNotas`, escala 0–20), lista (tipo, data, nota/máx., resultado, avaliador, observações), gráfico de linha das notas quando houver ≥ 3 pontos; "Registar avaliação" (`avaliacao:registar`).

**Progresso** — seletor de matrícula (se > 1); barras Teoria (`x/y` aulas, `percentualTeorico`) e Prática; cartões de **Exames** (teórico/prático: tentativas, resultado, data); **Processo da carta** (estado como passos); **Próximas aulas**; ações "Agendar aula" e "Inscrever em exame" (ao abrir, consulta `elegibilidade-exame`; se inelegível mostra a lista de `bloqueios` e só permite forçar com `exame:gerir`).

### 9.5 Jornada do estudante (componente distintivo)

`JornadaEstudante({ progresso })` — linha horizontal em ≥ `md`, lista vertical em ecrãs pequenos. Cinco etapas, **sem numeração**:

| Etapa | Concluída quando | Subtexto |
|---|---|---|
| Teoria | `percentualTeorico ≥ 100` (ou sem mínimo e exame teórico aprovado) | `x de y aulas` |
| Exame teórico | `exameTeoricoAprovado` | resultado/tentativas |
| Prática | `percentualPratico ≥ 100` (ou sem mínimo e exame prático aprovado) | `x de y aulas` |
| Exame prático | `examePraticoAprovado` | resultado/tentativas |
| Carta | `processoCartaEstado === 'ENTREGUE'` | estado do processo |

A primeira etapa não concluída é a **atual** (anel na cor primária, subtexto visível); as concluídas têm marca de verificação (`--sucesso`); as seguintes ficam discretas. Segmentos entre etapas preenchem-se com `--primary` (animação única de 400 ms ao abrir, desligada com `prefers-reduced-motion`). Cada etapa tem `aria-label` ("Teoria: concluída") e o conjunto é uma lista (`ol`). Matrícula em estado final (`CANCELADA`, `DESISTENTE`, `TRANSFERIDA`): etapas em cinzento com o rótulo do estado.

### 9.6 Nova matrícula (assistente)

Página `/estudantes/novo` (ou *sheet* largo a partir do estudante). Passos:

1. **Estudante** — pesquisar existente (mesma pesquisa da barra) ou criar novo (dados mínimos: nome*, documento*, telefone; restantes depois). Se o documento já existir, mostrar o estudante existente (código `ESTUDANTE_DOC_DUPLICADO`).
2. **Matrícula** — categoria*, pacote (preço mostrado), desconto (só com `desconto:aplicar`; exige motivo), estado inicial (Ativa/Pendente).
3. **Turma** (opcional) — turmas da categoria com vagas; filtro de **ano** (omissão: ano atual); mostra ocupação e horário.
4. **Pagamento** — plano de prestações (nº de prestações, primeiro vencimento, intervalo ou mensal) com **pré-visualização** das parcelas (a última absorve o arredondamento: calcular com `decimal.js`); opção "Registar primeiro pagamento agora".
5. **Revisão** — resumo; botão **Criar matrícula**.

Envia `POST /estudantes` com bloco `matricula` (estudante novo) ou `POST /matriculas` (existente). Sucesso → ecrã do estudante, separador Documentos (checklist por completar).

### 9.7 Lista de turmas e filtro por ano (requisito crucial)

`GET /turmas` + `GET /turmas/anos`.

- **Seletor de ano em destaque**, o primeiro controlo da barra de filtros, em segmentos para os anos disponíveis (`2026 (12)`, `2025 (9)`…) mais "Todos"; ocupa também a URL (`?ano=2026`). Omissão: ano atual se existir em `/turmas/anos`, senão o mais recente. Muitos anos → seletor em lista pesquisável.
- Outros filtros: estado, categoria, turno, filial, instrutor, interruptor "Com vagas", pesquisa por código/nome.
- Colunas: Código, Nome, Categoria (badge), **Ano**, Turno + dias (chips `Seg Qua Sex`), Horário, Instrutor responsável, **Ocupação** (barra + `18/25`), Estado, Período (início–fim). Ordenação por ano, data de início, código, nome, vagas livres.
- Ações: **Nova turma**, **Exportar** (PDF/Excel com os filtros, incluindo o ano).
- Formulário **Nova/editar turma**: código (sugerido), nome, categoria*, currículo (auto), filial*, instrutor responsável, turno*, data de início*, data de fim prevista*, vagas*, dias da semana* (chips de seleção), hora de início*/fim*, local, observações. O campo **Ano** é **só de leitura**, calculado ao vivo a partir da data de início ("Ano 2026"); nunca é enviado.

### 9.8 Detalhe da turma

Cabeçalho: código, nome, *badges* (estado, categoria, ano, turno), **indicadores**: vagas ocupadas/livres, aulas realizadas/total, período, instrutor. Ações de estado (Abrir, Iniciar, Concluir, Cancelar com motivo). Separadores:

- **Estudantes** — tabela paginada (nº, nome, estado da matrícula, entrada); "Adicionar estudante" (pesquisa de matrículas da mesma categoria **sem turma**: `GET /matriculas?categoriaId=...&estado=ATIVA&semTurma=true&search=`); por linha: Mudar de turma (escolher turma de destino), Remover da turma (motivo). Exportar lista (PDF/Excel).
- **Aulas** — lista de aulas teóricas; "Gerar aulas do horário" (confirma e mostra `{ criadas, ignoradas }`); por aula: Registar presenças (*sheet* com todos os estudantes e escolha rápida Presente/Falta/Justificada/Atraso, "Marcar todos presentes"), Concluir, Cancelar.
- **Informação** — formulário de edição.

### 9.9 Agenda de aulas práticas

`GET /aulas-praticas` (intervalo `de`/`ate`). **Vista semanal** em grelha CSS (dias × horas), com opção de vista diária (omissão em < `md`) e **vista lista** (tabela paginada). Filtros: instrutor, veículo, filial, pesquisa de estudante, estado. Cartões de aula coloridos por estado (badge); "agora" marcado. Clicar numa célula livre abre **Agendar aula** (*sheet*): estudante/matrícula (pesquisa), data, categoria, e `GET /aulas-praticas/disponibilidade` para propor horários livres; escolher horário, instrutor e veículo dentre os disponíveis; duração padrão da escola. Erros 409 de agenda e 422 (`FORA_DA_JANELA`, `PAGAMENTO_EM_ATRASO`) aparecem **no sheet**, junto ao campo relevante, com a mensagem da secção 5.4. Detalhe da aula: Remarcar, Cancelar (motivo; se tardio, aviso "cancelamento tardio — motivo obrigatório"), Concluir (km inicial/final), Marcar falta. Instrutor com escopo `PROPRIO` vê só as suas aulas.

### 9.10 Exames

Lista de sessões (filtros: tipo, categoria, período, estado) e detalhe da sessão: dados, inscrições (estado, resultado, nota, tentativa), "Inscrever estudante" (pesquisa de matrícula → mostra elegibilidade e bloqueios; "Forçar inscrição" só com `exame:gerir`, com confirmação), registar presença/resultado em linha. Processo da carta: estado em passos (`EM_PREPARACAO` → `SUBMETIDO` → `EM_ANALISE` → `PRONTA` → `ENTREGUE`; `REJEITADO` fora da sequência) com datas e nº do processo, editável com `processo_carta:gerir`.

### 9.11 Financeiro

Listas de **cobranças** (filtros: situação, tipo, vencimento, estudante, filial), **pagamentos** (método, estado, período) e **despesas** (categoria, período, veículo) — todas paginadas, com exportação quando houver relatório correspondente. **Resumo** (`GET /financeiro/resumo`): cartões (Receitas, Em aberto, Em atraso, Despesas, Saldo) e gráfico de barras mensal (receitas vs despesas) com filtro de período. Anular pagamento/cobrança: diálogo com motivo obrigatório e resumo do efeito ("O recibo REC-2026-000045 também será anulado").

### 9.12 Relatórios

`GET /relatorios` lista os relatórios disponíveis (nome, descrição, parâmetros). Cada relatório abre um formulário de parâmetros (período, ano, filial...) validado com Zod e dois botões **PDF** e **Excel** (`BotaoExportar`). Mostrar progresso ("A gerar...") e tratar `EXPORTACAO_DEMASIADO_GRANDE`.

| Código | Relatório | Permissão própria | Parâmetros |
|---|---|---|---|
| `ESTUDANTES_LISTAGEM` | Listagem de estudantes | `estudante:exportar` | mesmos filtros de `/estudantes` |
| `TURMAS_LISTAGEM` | Listagem de turmas | `turma:exportar` | mesmos filtros de `/turmas` (incl. `ano`) |
| `TURMA_LISTA_ESTUDANTES` | Lista de estudantes da turma | `turma:exportar` | `turmaId` |
| `TURMA_PRESENCAS` | Mapa de presenças da turma | `presenca:ver` | `turmaId` |
| `AULAS_PRATICAS_PERIODO` | Aulas práticas do período | `aula_pratica:ver` | `de`, `ate`, `instrutorId?`, `veiculoId?` |
| `INSTRUTORES_CARGA` | Carga horária dos instrutores | `instrutor:ver` | `de`, `ate` |
| `EXAMES_RESULTADOS` | Resultados de exames | `exame:ver` | `de`, `ate`, `tipo?` |
| `FROTA_MANUTENCOES` | Manutenções da frota | `manutencao:ver` | `de`, `ate`, `veiculoId?` |
| `FINANCEIRO_RECEBIMENTOS` | Recebimentos | `financeiro_relatorio:ver` | `de`, `ate`, `metodo?`, `filialId?` |
| `FINANCEIRO_EM_ATRASO` | Cobranças em atraso | `financeiro_relatorio:ver` | `ate?`, `filialId?` |
| `FINANCEIRO_DESPESAS` | Despesas | `financeiro_relatorio:ver` | `de`, `ate`, `categoria?` |
| `FINANCEIRO_RESUMO` | Resumo financeiro | `financeiro_relatorio:ver` | `de`, `ate` |

### 9.13 Notificações

Sino na barra superior: contador (`GET /notificacoes?naoLidas=true&limit=1` → `meta.total`, atualizado de 60 em 60 s com a aba visível); *popover* com as 10 mais recentes (`PATCH /notificacoes/:id/lida`, "Marcar todas como lidas"); página `/notificacoes` com lista paginada.

### 9.14 Utilizadores, perfis e permissões

**Perfis** (`/perfis`): lista (nome, descrição, nº de utilizadores, "sistema"/"personalizado"). **Detalhe do perfil** com a **matriz de permissões**: permissões agrupadas por módulo (acordeão), cada linha com o nome, o código discreto e um controlo segmentado de escopo **Sem acesso / Próprio / Filial / Escola**; marcador de permissão **sensível** (escudo + dica); filtro de pesquisa; "módulo inteiro" aplica um escopo a todas as linhas; antes de guardar, resumo das mudanças ("+3 permissões, −1, 2 escopos alterados"). Perfil protegido (`ADMINISTRADOR`) em modo só de leitura com aviso. **Criar perfil** (nome, descrição, "copiar de..."). **Eliminar** só sem utilizadores.

**Utilizadores** (`/utilizadores`): lista (nome, email, perfis, filiais, estado, último acesso). **Detalhe com separadores**: *Dados*; *Acesso* — perfis atribuídos (com filial opcional), filiais, **exceções individuais** (tabela: permissão, efeito Permitir/Negar, escopo, motivo, expira em; adicionar/remover) e painel **Permissões efetivas** (somente leitura, por módulo, com a origem de cada permissão: "Perfil Secretaria", "Permitido individualmente", "Negado individualmente"). **Anti-escalada na UI:** opções que o utilizador atual não possui aparecem desativadas com dica; a API valida de qualquer forma. Criar utilizador mostra a **senha temporária uma única vez** (copiar, aviso "não será mostrada novamente"). Redefinir senha e desativar pedem confirmação.

### 9.15 Configurações da escola

Separadores: **Escola** (dados), **Identidade** (mesma interface do onboarding, com pré-visualização), **Regras** (durações, antecedência de cancelamento, "agendar sem pagamento em dia", escala e nota mínima), **Janelas** (editor semanal de janelas de aulas práticas por dia, vazio = sem restrição), **Categorias**, **Currículos** (com módulos ordenáveis), **Pacotes**, **Tipos de documento** (obrigatório, validade, sensível), **Filiais**. Cada separador exige a permissão de edição respetiva; sem ela fica só de leitura.

### 9.16 Login

Fundo neutro; cartão central com logótipo e nome da escola (identidade pública por subdomínio — `VITE_BASE_DOMAIN` — ou `?escola=slug`; sem escola identificada, tema neutro do SaaS). Campos email e senha (mostrar/ocultar), erro `CREDENCIAIS_INVALIDAS` em linha, aviso `?expirou=1` ("A sua sessão expirou. Entre de novo.").

---

## 10. Pesquisa global de estudantes

Componente `PesquisaGlobal` na barra superior (como no ecrã de referência): campo com ícone e texto de apoio "Pesquisar estudante por nome, BI ou número". Atalho **Ctrl/⌘ + K** abre a paleta (`Command`).

- `GET /estudantes/pesquisa?q=&limit=8` com *debounce* 250 ms, mínimo 2 caracteres, cancelamento do pedido anterior (`signal`).
- Resultados: avatar, nome, "Nº 1244 · BI 0045…", *badges* de estado e categoria, turma. Setas navegam, **Enter** abre `/estudantes/:id`, **Esc** fecha. Rodapé: "Ver todos os resultados" → `/estudantes?search=<q>`.
- Pré-carregar o resumo do estudante ao passar o rato/foco (`queryClient.prefetchQuery`).
- Estados: vazio inicial (dica de uso), a carregar (esqueleto), sem resultados ("Nenhum estudante encontrado para «q». Verifique a grafia ou procure pelo número do BI."), erro (com "Tentar de novo").
- Só renderiza com `estudante:ver`.

---

## 11. Listas, tabelas e filtros (padrão único)

`DataTable` (TanStack Table, `manualPagination`, `manualSorting`) + `FiltrosLista` + `useListaUrl`.

```ts
// lib/url-state.ts — contrato
export declare function useListaUrl<F extends Record<string, string | number | boolean | undefined>>(opcoes: {
  filtros: z.ZodType<F>;                       // validação/coerção dos filtros a partir da query string
  limitePadrao?: number;                       // 20
  ordenarPadrao?: { campo: string; ordem: 'asc' | 'desc' };
}): {
  query: { page: number; limit: number; ordenar?: string; ordem?: 'asc' | 'desc'; search?: string } & F; // vai direto ao service
  definirFiltros(parcial: Partial<F>): void;   // repõe page = 1
  definirPagina(n: number): void;
  definirLimite(n: number): void;
  definirOrdenacao(campo: string, ordem: 'asc' | 'desc'): void;
  limpar(): void;
  filtrosAtivos: number;
};
```

Regras:
- Estado **na URL**; valores inválidos são descartados. Mudar filtro/pesquisa → `page = 1`. Pesquisa com *debounce* 300 ms.
- Tamanhos de página 10 / 20 / 50 / 100 (omissão 20). Rodapé: "A mostrar 21–40 de 134", anterior/seguinte e salto de página.
- `placeholderData: keepPreviousData` + indicador discreto de atualização (a tabela não pisca).
- Esqueletos (até 10 linhas) no primeiro carregamento; **estado vazio** sem filtros ("Ainda não há turmas. Crie a primeira."), **sem resultados** com filtros ("Nenhuma turma corresponde aos filtros." + "Limpar filtros"); **erro** com "Tentar de novo".
- Cabeçalho fixo, `aria-sort` nas colunas ordenáveis, linhas clicáveis com foco e Enter. Abaixo de `md` a tabela vira **lista de cartões**.
- Exportar usa os mesmos filtros da lista (`BotaoExportar` recebe `query` sem `page/limit`).

---

## 12. Formulários

- RHF + Zod; `mode: 'onBlur'`, revalidação `onChange` após o primeiro erro. Esquemas em `features/<f>/schemas`; mensagens em português; campo obrigatório marcado no rótulo (não só com `*`: texto "obrigatório" para leitores de ecrã).
- Erros da API 422: mapear `detalhes.campos` para `setError`; erros de regra (409/422 de negócio) → mensagem junto ao botão ou ao campo mais relevante.
- Proteção de saída: se o formulário estiver sujo, confirmar antes de sair/fechar.
- Botão de submissão desativa e mostra progresso durante a mutation; nunca duplicar pedidos (idempotência em pagamentos).
- Componentes: `InputDinheiro`, `CampoData` (calendário + digitação `dd/MM/yyyy`), `CampoHora`, `SeletorDiasSemana` (chips Seg…Dom ↔ `1..7`), `SeletorCor` (com contraste), `UploadFicheiro` (tipo/tamanho validados antes de enviar), `ForcaSenha` (3 critérios: 8+ caracteres, letra maiúscula, caractere especial; três barras como na referência).
- Pesquisa em selects de entidades grandes (estudantes, matrículas, turmas) usa `Combobox` com busca remota paginada.

---

## 13. Documentos e exportações

- `BotaoExportar` (menu "PDF"/"Excel"): chama `baixarFicheiro(rota, { params: filtros + formato })`; mostra "A gerar..." e desativa durante o pedido; sucesso → descarrega; erro → toast com a mensagem do código.
- Documentos oficiais (`POST /documentos/gerar`): abrir o PDF num novo separador (`abrir: true`) **e** oferecer "Descarregar"; guardar `X-Documento-Numero` para o toast ("Recibo REC-2026-000045 emitido").
- Nomes de ficheiro vêm do `Content-Disposition`. O front **não** altera nem recompõe documentos.
- Cabeçalho/rodapé e cores dos ficheiros são decididos pelo backend a partir da identidade da escola; o front só garante que a identidade foi guardada antes de pedir a primeira exportação.

| Tipo | Série / prefixo | Entrada | Nota |
|---|---|---|---|
| `RECIBO` | `RECIBO` / `REC` | `pagamentoId` | Idempotente por pagamento: reabre o mesmo ficheiro enquanto não for anulado. |
| `DECLARACAO_MATRICULA` | `DECLARACAO` / `DEC` | `matriculaId` | Matrícula ATIVA, PENDENTE ou SUSPENSA. |
| `DECLARACAO_FREQUENCIA` | `DECLARACAO` / `DEC` | `matriculaId` | Inclui aulas teóricas e práticas realizadas até à data. |
| `CONTRATO_FORMACAO` | `CONTRATO` / `CTR` | `matriculaId` | Inclui preço acordado, desconto, plano de prestações e pacote. |
| `CERTIFICADO_CONCLUSAO` | `CERTIFICADO` / `CERT` | `matriculaId` | Só para matrícula CONCLUIDA. |
| `EXTRATO_FINANCEIRO` | `EXTRATO` / `EXT` | `estudanteId` (+ `de`, `ate`) | Cobranças, pagamentos e saldo. |
| `FICHA_ESTUDANTE` | sem número | `estudanteId` | Sem número. Dados pessoais, matrículas e progresso. |

---

## 14. Estados de UI e texto

**Vocabulário fixo:** Estudante, Matrícula, Turma, Categoria, Pacote, Cobrança, Pagamento, Recibo, Aula prática, Aula teórica, Instrutor, Veículo, Documento, Filial. Verbos de ação: Criar, Guardar alterações, Registar (pagamento, presença, avaliação), Agendar, Remarcar, Cancelar, Anular (pagamentos, cobranças, documentos), Eliminar, Exportar, Descarregar. O mesmo verbo no botão e no toast de sucesso ("Registar pagamento" → "Pagamento registado").

**Vazio:** diz o que é e o que fazer ("Ainda não há aulas agendadas para esta semana. Agende a primeira."). **Erro:** diz o que falhou e como resolver; nunca pede desculpa nem culpa o utilizador. **Confirmações destrutivas:** título com a ação, uma frase com a consequência, campo de motivo quando exigido, botão com o verbo exato ("Anular pagamento").

Datas por extenso em contexto humano ("2 de março de 2026"), compactas em tabelas (`02/03/2026`). Moeda `22 500,00 Kz`. Estados via `BadgeEstado` (anexo A).

---

## 15. Acessibilidade, responsividade e desempenho

- **Teclado:** tudo operável; ordem de foco lógica; *skip link* "Saltar para o conteúdo"; diálogos com *focus trap* (Radix); `Esc` fecha.
- **Contraste** ≥ 4.5:1 (texto) e ≥ 3:1 (componentes); estados nunca só por cor; alvos táteis ≥ 40 px em ecrãs táteis; `aria-live` para toasts e resultados de pesquisa; tabelas com `<caption>` visualmente oculta; gráficos com alternativa em tabela/texto.
- **Responsivo:** menu lateral → *drawer* < `lg`; tabelas → cartões < `md`; agenda → vista diária < `md`; cabeçalho do estudante empilha; formulários em coluna única < `md`. Instrutores usam sobretudo agenda, aulas e avaliações no telemóvel: esses ecrãs são **mobile-first**.
- **Desempenho:** rotas com `lazy`; Recharts e *fontsource* alternativas carregados sob demanda; JS inicial ≤ 250 kB (gzip); sem listas virtualizadas (paginação no servidor); imagens com `loading="lazy"`; `staleTime` de 5 min para catálogos (anos, categorias, permissões); sem *refetch* ao focar a janela nas listas pesadas.

---

## 16. Testes

- **Unitários:** `contraste`/`gerarTokens` (todos os pares ≥ 4.5), `kz`, parser de `Content-Disposition`, `useListaUrl`, cálculo da pré-visualização de prestações, regra de etapas da Jornada, `temPermissao`.
- **Componentes (Testing Library + MSW):** `DataTable` (paginação/ordenação/estados), `PesquisaGlobal` (teclado, estados), formulário de turma (ano calculado, só leitura), registar pagamento (alocação automática/manual, soma), matriz de permissões (escopos, bloqueio de protegido), onboarding (pré-visualização).
- **Acessibilidade:** `jest-axe` nas páginas principais em claro e escuro.
- **Playwright (fumo):** login → pesquisar estudante → abrir → percorrer os 5 separadores → exportar PDF; filtrar turmas por ano; agendar aula com conflito (mensagem correta).
- **Definição de pronto** de um ecrã: tipos completos, estados (a carregar/vazio/erro) tratados, claro e escuro verificados, teclado e leitor de ecrã básicos, responsivo, permissões aplicadas, textos em pt-AO, sem `any`, sem cores fixas.

---

## 17. Plano de implementação por fases

| Fase | Entrega | Pronto quando |
|---|---|---|
| **F0. Base** | Vite + Tailwind + shadcn, tokens claro/escuro, `AlternadorTema`, `http` + interceptors, auth (login, refresh, `/auth/me`, primeiro acesso), shell (barra, menu, breadcrumbs), `DataTable`, `useListaUrl`, `BadgeEstado`, `Pode`, `RotaProtegida`, páginas 403/404 | login funciona; tema claro/escuro sem *flash*; guards a funcionar |
| **F1. Identidade e onboarding** | `lib/identidade`, onboarding completo com pré-visualização, configurações da escola (dados, identidade, regras, janelas, filiais) | uma escola nova percorre o onboarding e o tema/logótipo mudam em toda a app |
| **F2. Formação, pessoas e frota** | categorias, currículos, pacotes, instrutores, funcionários, tipos de documento, veículos (listas, formulários, anexos) | CRUDs paginados com permissões |
| **F3. Estudantes** | lista + pesquisa global, ecrã do estudante (cabeçalho, Perfil, Documentos), exportação da lista | pesquisa abre o estudante por nome, BI e número; separadores com permissões |
| **F4. Matrícula, turmas e aulas** | assistente de matrícula, turmas com **filtro de ano**, detalhe da turma, aulas teóricas e presenças, agenda prática, avaliações | criar turma, inscrever estudantes e agendar aula sem conflitos |
| **F5. Finanças, exames e progresso** | separador Finanças, registar/anular pagamentos, cobranças e despesas, exames e processo da carta, Progresso + **Jornada** | pagamento com alocação manual/automática; elegibilidade de exame com bloqueios |
| **F6. Acesso** | perfis (matriz), utilizadores (acesso, exceções, permissões efetivas), auditoria | criar perfil, atribuir e ver efeitos nas permissões efetivas |
| **F7. Documentos, relatórios, painel, notificações** | geração de documentos, 12 relatórios, painel, sino e página de notificações, shell da plataforma | todos os ficheiros descarregam com o nome correto |
| **F8. Qualidade** | acessibilidade, desempenho, Playwright, revisão de textos | critérios da secção 16 cumpridos |

---

## Anexo A — Enums, rótulos e variantes

Gerado do `schema.sql`. Copiar para `src/types/enums.ts`. Cada enum é um objeto `as const` + tipo; `ROTULOS` dá o rótulo e a variante do `BadgeEstado`.

```ts
// src/types/enums.ts — gerado do schema.sql (não editar à mão)

export const EscolaEstado = ['PENDENTE', 'ATIVA', 'SUSPENSA', 'CANCELADA'] as const;
export type EscolaEstado = (typeof EscolaEstado)[number];
export const EscolaIdentidadeTemaPadrao = ['CLARO', 'ESCURO', 'SISTEMA'] as const;
export type EscolaIdentidadeTemaPadrao = (typeof EscolaIdentidadeTemaPadrao)[number];
export const SubscricaoEstado = ['TESTE', 'ATIVA', 'EM_ATRASO', 'CANCELADA'] as const;
export type SubscricaoEstado = (typeof SubscricaoEstado)[number];
export const UtilizadorTipo = ['PLATAFORMA', 'ESCOLA'] as const;
export type UtilizadorTipo = (typeof UtilizadorTipo)[number];
export const PerfilModeloPermissaoEscopo = ['ESCOLA', 'FILIAL', 'PROPRIO'] as const;
export type PerfilModeloPermissaoEscopo = (typeof PerfilModeloPermissaoEscopo)[number];
export const PerfilPermissaoEscopo = ['ESCOLA', 'FILIAL', 'PROPRIO'] as const;
export type PerfilPermissaoEscopo = (typeof PerfilPermissaoEscopo)[number];
export const UtilizadorPermissaoEfeito = ['ALLOW', 'DENY'] as const;
export type UtilizadorPermissaoEfeito = (typeof UtilizadorPermissaoEfeito)[number];
export const UtilizadorPermissaoEscopo = ['ESCOLA', 'FILIAL', 'PROPRIO'] as const;
export type UtilizadorPermissaoEscopo = (typeof UtilizadorPermissaoEscopo)[number];
export const CurriculoModuloTipo = ['TEORICO', 'PRATICO'] as const;
export type CurriculoModuloTipo = (typeof CurriculoModuloTipo)[number];
export const EstudanteGenero = ['M', 'F'] as const;
export type EstudanteGenero = (typeof EstudanteGenero)[number];
export const EstudanteEstadoCivil = ['SOLTEIRO', 'CASADO', 'UNIAO_DE_FACTO', 'DIVORCIADO', 'VIUVO'] as const;
export type EstudanteEstadoCivil = (typeof EstudanteEstadoCivil)[number];
export const EstudanteDocTipo = ['BI', 'PASSAPORTE', 'CARTA_RESIDENTE', 'OUTRO'] as const;
export type EstudanteDocTipo = (typeof EstudanteDocTipo)[number];
export const EstudanteEstado = ['ATIVO', 'INATIVO', 'CONCLUIDO', 'DESISTENTE'] as const;
export type EstudanteEstado = (typeof EstudanteEstado)[number];
export const InstrutorGenero = ['M', 'F'] as const;
export type InstrutorGenero = (typeof InstrutorGenero)[number];
export const InstrutorDocTipo = ['BI', 'PASSAPORTE', 'CARTA_RESIDENTE', 'OUTRO'] as const;
export type InstrutorDocTipo = (typeof InstrutorDocTipo)[number];
export const InstrutorFuncao = ['PRATICO', 'TEORICO', 'AMBOS'] as const;
export type InstrutorFuncao = (typeof InstrutorFuncao)[number];
export const FuncionarioDocTipo = ['BI', 'PASSAPORTE', 'CARTA_RESIDENTE', 'OUTRO'] as const;
export type FuncionarioDocTipo = (typeof FuncionarioDocTipo)[number];
export const VeiculoCombustivel = ['GASOLINA', 'GASOLEO', 'ELETRICO', 'HIBRIDO', 'OUTRO'] as const;
export type VeiculoCombustivel = (typeof VeiculoCombustivel)[number];
export const VeiculoTransmissao = ['MANUAL', 'AUTOMATICA'] as const;
export type VeiculoTransmissao = (typeof VeiculoTransmissao)[number];
export const VeiculoEstado = ['DISPONIVEL', 'EM_MANUTENCAO', 'INATIVO'] as const;
export type VeiculoEstado = (typeof VeiculoEstado)[number];
export const VeiculoManutencaoTipo = ['PREVENTIVA', 'CORRETIVA', 'INSPECAO'] as const;
export type VeiculoManutencaoTipo = (typeof VeiculoManutencaoTipo)[number];
export const TipoDocumentoAplicaA = ['ESTUDANTE', 'INSTRUTOR', 'VEICULO'] as const;
export type TipoDocumentoAplicaA = (typeof TipoDocumentoAplicaA)[number];
export const AnexoEstado = ['PENDENTE', 'VALIDADO', 'REJEITADO'] as const;
export type AnexoEstado = (typeof AnexoEstado)[number];
export const TurmaTurno = ['MANHA', 'TARDE', 'NOITE', 'FIM_SEMANA'] as const;
export type TurmaTurno = (typeof TurmaTurno)[number];
export const TurmaEstado = ['PLANEADA', 'ABERTA', 'EM_CURSO', 'CONCLUIDA', 'CANCELADA'] as const;
export type TurmaEstado = (typeof TurmaEstado)[number];
export const MatriculaEstado = ['PENDENTE', 'ATIVA', 'SUSPENSA', 'CONCLUIDA', 'CANCELADA', 'DESISTENTE', 'TRANSFERIDA'] as const;
export type MatriculaEstado = (typeof MatriculaEstado)[number];
export const AulaTeoricaEstado = ['AGENDADA', 'REALIZADA', 'CANCELADA'] as const;
export type AulaTeoricaEstado = (typeof AulaTeoricaEstado)[number];
export const PresencaEstado = ['PRESENTE', 'FALTA', 'JUSTIFICADA', 'ATRASO'] as const;
export type PresencaEstado = (typeof PresencaEstado)[number];
export const AulaPraticaTipo = ['NORMAL', 'SIMULACRO_EXAME'] as const;
export type AulaPraticaTipo = (typeof AulaPraticaTipo)[number];
export const AulaPraticaEstado = ['AGENDADA', 'REALIZADA', 'CANCELADA', 'FALTOU'] as const;
export type AulaPraticaEstado = (typeof AulaPraticaEstado)[number];
export const AvaliacaoTipo = ['AULA_PRATICA', 'SIMULACRO_TEORICO', 'SIMULACRO_PRATICO', 'PROVA_INTERNA'] as const;
export type AvaliacaoTipo = (typeof AvaliacaoTipo)[number];
export const AvaliacaoResultado = ['APTO', 'NAO_APTO'] as const;
export type AvaliacaoResultado = (typeof AvaliacaoResultado)[number];
export const CobrancaTipo = ['PACOTE', 'TAXA_EXAME', 'AULA_EXTRA', 'MATERIAL', 'SEGUNDA_VIA', 'OUTRO'] as const;
export type CobrancaTipo = (typeof CobrancaTipo)[number];
export const CobrancaEstado = ['ABERTA', 'ANULADA'] as const;
export type CobrancaEstado = (typeof CobrancaEstado)[number];
export const PagamentoMetodo = ['NUMERARIO', 'TPA', 'MULTICAIXA_EXPRESS', 'TRANSFERENCIA', 'DEPOSITO', 'REFERENCIA', 'OUTRO'] as const;
export type PagamentoMetodo = (typeof PagamentoMetodo)[number];
export const PagamentoEstado = ['CONFIRMADO', 'ANULADO'] as const;
export type PagamentoEstado = (typeof PagamentoEstado)[number];
export const DespesaCategoria = ['COMBUSTIVEL', 'MANUTENCAO', 'SALARIOS', 'RENDA', 'IMPOSTOS', 'MATERIAL', 'OUTRA'] as const;
export type DespesaCategoria = (typeof DespesaCategoria)[number];
export const SessaoExameTipo = ['TEORICO', 'PRATICO'] as const;
export type SessaoExameTipo = (typeof SessaoExameTipo)[number];
export const SessaoExameEstado = ['AGENDADA', 'REALIZADA', 'CANCELADA'] as const;
export type SessaoExameEstado = (typeof SessaoExameEstado)[number];
export const InscricaoExameEstado = ['INSCRITO', 'PRESENTE', 'FALTOU', 'CANCELADO'] as const;
export type InscricaoExameEstado = (typeof InscricaoExameEstado)[number];
export const InscricaoExameResultado = ['PENDENTE', 'APROVADO', 'REPROVADO'] as const;
export type InscricaoExameResultado = (typeof InscricaoExameResultado)[number];
export const ProcessoCartaEstado = ['EM_PREPARACAO', 'SUBMETIDO', 'EM_ANALISE', 'PRONTA', 'ENTREGUE', 'REJEITADO'] as const;
export type ProcessoCartaEstado = (typeof ProcessoCartaEstado)[number];
export const SerieDocumentalTipo = ['ESTUDANTE', 'MATRICULA', 'RECIBO', 'DECLARACAO', 'CONTRATO', 'CERTIFICADO', 'EXTRATO'] as const;
export type SerieDocumentalTipo = (typeof SerieDocumentalTipo)[number];
export const DocumentoEmitidoTipo = ['RECIBO', 'DECLARACAO_MATRICULA', 'CONTRATO_FORMACAO', 'FICHA_ESTUDANTE', 'EXTRATO_FINANCEIRO', 'CERTIFICADO_CONCLUSAO', 'DECLARACAO_FREQUENCIA'] as const;
export type DocumentoEmitidoTipo = (typeof DocumentoEmitidoTipo)[number];
export const NotificacaoCanal = ['INTERNA', 'SMS', 'EMAIL', 'WHATSAPP'] as const;
export type NotificacaoCanal = (typeof NotificacaoCanal)[number];
export const NotificacaoEstado = ['PENDENTE', 'ENVIADA', 'FALHOU', 'LIDA'] as const;
export type NotificacaoEstado = (typeof NotificacaoEstado)[number];
export const CobrancaSituacao = ['ANULADA', 'PAGA', 'EM_ATRASO', 'PARCIAL', 'PENDENTE'] as const;
export type CobrancaSituacao = (typeof CobrancaSituacao)[number];

export type Variante = 'neutro' | 'info' | 'sucesso' | 'aviso' | 'perigo' | 'discreto';

export const ROTULOS = {
  'estudante.estado': {
    ATIVO: { rotulo: 'Ativo', variante: 'sucesso' },
    INATIVO: { rotulo: 'Inativo', variante: 'discreto' },
    CONCLUIDO: { rotulo: 'Concluído', variante: 'info' },
    DESISTENTE: { rotulo: 'Desistente', variante: 'perigo' },
  },
  'matricula.estado': {
    PENDENTE: { rotulo: 'Pendente', variante: 'aviso' },
    ATIVA: { rotulo: 'Ativa', variante: 'sucesso' },
    SUSPENSA: { rotulo: 'Suspensa', variante: 'aviso' },
    CONCLUIDA: { rotulo: 'Concluída', variante: 'info' },
    CANCELADA: { rotulo: 'Cancelada', variante: 'perigo' },
    DESISTENTE: { rotulo: 'Desistente', variante: 'perigo' },
    TRANSFERIDA: { rotulo: 'Transferida', variante: 'discreto' },
  },
  'turma.estado': {
    PLANEADA: { rotulo: 'Planeada', variante: 'discreto' },
    ABERTA: { rotulo: 'Aberta', variante: 'info' },
    EM_CURSO: { rotulo: 'Em curso', variante: 'sucesso' },
    CONCLUIDA: { rotulo: 'Concluída', variante: 'neutro' },
    CANCELADA: { rotulo: 'Cancelada', variante: 'perigo' },
  },
  'turma.turno': {
    MANHA: { rotulo: 'Manhã', variante: 'neutro' },
    TARDE: { rotulo: 'Tarde', variante: 'neutro' },
    NOITE: { rotulo: 'Noite', variante: 'neutro' },
    FIM_SEMANA: { rotulo: 'Fim de semana', variante: 'neutro' },
  },
  'aula_pratica.estado': {
    AGENDADA: { rotulo: 'Agendada', variante: 'info' },
    REALIZADA: { rotulo: 'Realizada', variante: 'sucesso' },
    CANCELADA: { rotulo: 'Cancelada', variante: 'discreto' },
    FALTOU: { rotulo: 'Faltou', variante: 'perigo' },
  },
  'aula_teorica.estado': {
    AGENDADA: { rotulo: 'Agendada', variante: 'info' },
    REALIZADA: { rotulo: 'Realizada', variante: 'sucesso' },
    CANCELADA: { rotulo: 'Cancelada', variante: 'discreto' },
  },
  'presenca.estado': {
    PRESENTE: { rotulo: 'Presente', variante: 'sucesso' },
    FALTA: { rotulo: 'Falta', variante: 'perigo' },
    JUSTIFICADA: { rotulo: 'Falta justificada', variante: 'aviso' },
    ATRASO: { rotulo: 'Atraso', variante: 'aviso' },
  },
  'cobranca.situacao': {
    PENDENTE: { rotulo: 'Pendente', variante: 'neutro' },
    PARCIAL: { rotulo: 'Parcial', variante: 'aviso' },
    PAGA: { rotulo: 'Paga', variante: 'sucesso' },
    EM_ATRASO: { rotulo: 'Em atraso', variante: 'perigo' },
    ANULADA: { rotulo: 'Anulada', variante: 'discreto' },
  },
  'pagamento.estado': {
    CONFIRMADO: { rotulo: 'Confirmado', variante: 'sucesso' },
    ANULADO: { rotulo: 'Anulado', variante: 'discreto' },
  },
  'pagamento.metodo': {
    NUMERARIO: { rotulo: 'Numerário', variante: 'neutro' },
    TPA: { rotulo: 'TPA', variante: 'neutro' },
    MULTICAIXA_EXPRESS: { rotulo: 'Multicaixa Express', variante: 'neutro' },
    TRANSFERENCIA: { rotulo: 'Transferência', variante: 'neutro' },
    DEPOSITO: { rotulo: 'Depósito', variante: 'neutro' },
    REFERENCIA: { rotulo: 'Referência', variante: 'neutro' },
    OUTRO: { rotulo: 'Outro', variante: 'neutro' },
  },
  'anexo.estado': {
    PENDENTE: { rotulo: 'Por validar', variante: 'aviso' },
    VALIDADO: { rotulo: 'Validado', variante: 'sucesso' },
    REJEITADO: { rotulo: 'Rejeitado', variante: 'perigo' },
  },
  'inscricao_exame.estado': {
    INSCRITO: { rotulo: 'Inscrito', variante: 'info' },
    PRESENTE: { rotulo: 'Presente', variante: 'neutro' },
    FALTOU: { rotulo: 'Faltou', variante: 'perigo' },
    CANCELADO: { rotulo: 'Cancelado', variante: 'discreto' },
  },
  'inscricao_exame.resultado': {
    PENDENTE: { rotulo: 'Pendente', variante: 'discreto' },
    APROVADO: { rotulo: 'Aprovado', variante: 'sucesso' },
    REPROVADO: { rotulo: 'Reprovado', variante: 'perigo' },
  },
  'sessao_exame.tipo': {
    TEORICO: { rotulo: 'Teórico', variante: 'neutro' },
    PRATICO: { rotulo: 'Prático', variante: 'neutro' },
  },
  'processo_carta.estado': {
    EM_PREPARACAO: { rotulo: 'Em preparação', variante: 'aviso' },
    SUBMETIDO: { rotulo: 'Submetido', variante: 'info' },
    EM_ANALISE: { rotulo: 'Em análise', variante: 'info' },
    PRONTA: { rotulo: 'Pronta', variante: 'sucesso' },
    ENTREGUE: { rotulo: 'Entregue', variante: 'sucesso' },
    REJEITADO: { rotulo: 'Rejeitado', variante: 'perigo' },
  },
  'veiculo.estado': {
    DISPONIVEL: { rotulo: 'Disponível', variante: 'sucesso' },
    EM_MANUTENCAO: { rotulo: 'Em manutenção', variante: 'aviso' },
    INATIVO: { rotulo: 'Inativo', variante: 'discreto' },
  },
  'avaliacao.resultado': {
    APTO: { rotulo: 'Apto', variante: 'sucesso' },
    NAO_APTO: { rotulo: 'Não apto', variante: 'perigo' },
  },
} as const satisfies Record<string, Record<string, { rotulo: string; variante: Variante }>>;

export type DominioRotulo = keyof typeof ROTULOS;
```

## Anexo B — Códigos de permissão

Gerado do catálogo do backend (105 códigos). Copiar para `src/lib/permissoes.ts`.

```ts
// src/lib/permissoes.ts — gerado do catálogo (não editar à mão)
export const P = {
  UTILIZADOR_VER: 'utilizador:ver',
  UTILIZADOR_CRIAR: 'utilizador:criar',
  UTILIZADOR_EDITAR: 'utilizador:editar',
  UTILIZADOR_DESATIVAR: 'utilizador:desativar',
  UTILIZADOR_REDEFINIR_SENHA: 'utilizador:redefinir_senha',
  PERFIL_VER: 'perfil:ver',
  PERFIL_CRIAR: 'perfil:criar',
  PERFIL_EDITAR: 'perfil:editar',
  PERFIL_ELIMINAR: 'perfil:eliminar',
  PERMISSAO_VER: 'permissao:ver',
  PERMISSAO_ATRIBUIR: 'permissao:atribuir',
  AUDITORIA_VER: 'auditoria:ver',
  ESCOLA_VER: 'escola:ver',
  ESCOLA_EDITAR: 'escola:editar',
  ESCOLA_IDENTIDADE_EDITAR: 'escola_identidade:editar',
  ESCOLA_CONFIGURACAO_EDITAR: 'escola_configuracao:editar',
  FILIAL_VER: 'filial:ver',
  FILIAL_CRIAR: 'filial:criar',
  FILIAL_EDITAR: 'filial:editar',
  FILIAL_DESATIVAR: 'filial:desativar',
  SUBSCRICAO_VER: 'subscricao:ver',
  ESTUDANTE_VER: 'estudante:ver',
  ESTUDANTE_CRIAR: 'estudante:criar',
  ESTUDANTE_EDITAR: 'estudante:editar',
  ESTUDANTE_ELIMINAR: 'estudante:eliminar',
  ESTUDANTE_EXPORTAR: 'estudante:exportar',
  ESTUDANTE_ACESSO_GERIR: 'estudante_acesso:gerir',
  ANEXO_VER: 'anexo:ver',
  ANEXO_CARREGAR: 'anexo:carregar',
  ANEXO_VALIDAR: 'anexo:validar',
  ANEXO_ELIMINAR: 'anexo:eliminar',
  ANEXO_SENSIVEL_VER: 'anexo_sensivel:ver',
  TIPO_DOCUMENTO_VER: 'tipo_documento:ver',
  TIPO_DOCUMENTO_GERIR: 'tipo_documento:gerir',
  DOCUMENTO_GERAR: 'documento:gerar',
  DOCUMENTO_VER: 'documento:ver',
  DOCUMENTO_ANULAR: 'documento:anular',
  CATEGORIA_VER: 'categoria:ver',
  CATEGORIA_GERIR: 'categoria:gerir',
  CURRICULO_VER: 'curriculo:ver',
  CURRICULO_GERIR: 'curriculo:gerir',
  PACOTE_VER: 'pacote:ver',
  PACOTE_GERIR: 'pacote:gerir',
  TURMA_VER: 'turma:ver',
  TURMA_CRIAR: 'turma:criar',
  TURMA_EDITAR: 'turma:editar',
  TURMA_CANCELAR: 'turma:cancelar',
  TURMA_EXPORTAR: 'turma:exportar',
  TURMA_ESTUDANTE_GERIR: 'turma_estudante:gerir',
  MATRICULA_VER: 'matricula:ver',
  MATRICULA_CRIAR: 'matricula:criar',
  MATRICULA_EDITAR: 'matricula:editar',
  MATRICULA_CANCELAR: 'matricula:cancelar',
  MATRICULA_SUSPENDER: 'matricula:suspender',
  MATRICULA_REATIVAR: 'matricula:reativar',
  AULA_TEORICA_VER: 'aula_teorica:ver',
  AULA_TEORICA_CRIAR: 'aula_teorica:criar',
  AULA_TEORICA_EDITAR: 'aula_teorica:editar',
  AULA_TEORICA_CANCELAR: 'aula_teorica:cancelar',
  PRESENCA_VER: 'presenca:ver',
  PRESENCA_REGISTAR: 'presenca:registar',
  AULA_PRATICA_VER: 'aula_pratica:ver',
  AULA_PRATICA_AGENDAR: 'aula_pratica:agendar',
  AULA_PRATICA_REMARCAR: 'aula_pratica:remarcar',
  AULA_PRATICA_CANCELAR: 'aula_pratica:cancelar',
  AULA_PRATICA_REGISTAR: 'aula_pratica:registar',
  AVALIACAO_VER: 'avaliacao:ver',
  AVALIACAO_REGISTAR: 'avaliacao:registar',
  INSTRUTOR_VER: 'instrutor:ver',
  INSTRUTOR_CRIAR: 'instrutor:criar',
  INSTRUTOR_EDITAR: 'instrutor:editar',
  INSTRUTOR_DESATIVAR: 'instrutor:desativar',
  FUNCIONARIO_VER: 'funcionario:ver',
  FUNCIONARIO_CRIAR: 'funcionario:criar',
  FUNCIONARIO_EDITAR: 'funcionario:editar',
  FUNCIONARIO_DESATIVAR: 'funcionario:desativar',
  EXAME_VER: 'exame:ver',
  EXAME_GERIR: 'exame:gerir',
  EXAME_INSCRICAO_CRIAR: 'exame_inscricao:criar',
  EXAME_INSCRICAO_RESULTADO: 'exame_inscricao:resultado',
  PROCESSO_CARTA_VER: 'processo_carta:ver',
  PROCESSO_CARTA_GERIR: 'processo_carta:gerir',
  VEICULO_VER: 'veiculo:ver',
  VEICULO_CRIAR: 'veiculo:criar',
  VEICULO_EDITAR: 'veiculo:editar',
  VEICULO_DESATIVAR: 'veiculo:desativar',
  MANUTENCAO_VER: 'manutencao:ver',
  MANUTENCAO_REGISTAR: 'manutencao:registar',
  ABASTECIMENTO_REGISTAR: 'abastecimento:registar',
  COBRANCA_VER: 'cobranca:ver',
  COBRANCA_CRIAR: 'cobranca:criar',
  COBRANCA_ANULAR: 'cobranca:anular',
  PAGAMENTO_VER: 'pagamento:ver',
  PAGAMENTO_REGISTAR: 'pagamento:registar',
  PAGAMENTO_ANULAR: 'pagamento:anular',
  DESCONTO_APLICAR: 'desconto:aplicar',
  DESPESA_VER: 'despesa:ver',
  DESPESA_REGISTAR: 'despesa:registar',
  DESPESA_ELIMINAR: 'despesa:eliminar',
  FINANCEIRO_RELATORIO_VER: 'financeiro_relatorio:ver',
  RELATORIO_VER: 'relatorio:ver',
  RELATORIO_EXPORTAR: 'relatorio:exportar',
  DASHBOARD_VER: 'dashboard:ver',
  NOTIFICACAO_VER: 'notificacao:ver',
  NOTIFICACAO_ENVIAR: 'notificacao:enviar',
} as const;
export type Permissao = (typeof P)[keyof typeof P];
```

## Anexo C — Contrato de endpoints

Resumo do BACKEND_KB.md, secção 10. Prefixo `/api/v1`. **P** = coleção paginada (`Paginado<T>`), **B** = resposta binária. Total: 199.

#### Autenticação

| Método | Rota | Permissão | Descrição |
|---|---|---|---|
| POST | `/auth/login` | público | Login (email + senha). Devolve `accessToken` e define o cookie httpOnly do refresh token. |
| POST | `/auth/refresh` | público | Roda o refresh token (cookie) e devolve novo `accessToken`. |
| POST | `/auth/logout` | autenticado | Revoga a família de refresh tokens e limpa o cookie. |
| GET | `/auth/me` | autenticado | Utilizador, perfis, permissões efetivas (`codigo`, `escopo`, `filiais`), filiais, identidade resumida e estado do onboarding. |
| POST | `/auth/alterar-senha` | autenticado | Altera a própria senha (exige a senha atual; limpa `deve_alterar_senha`). |
| GET | `/publico/escolas/:slug/identidade` | público | Identidade pública (nome, cores, tema, URL do logótipo) para o ecrã de login por subdomínio. |
| GET | `/saude` | público | Health check (API + base de dados). |

#### Plataforma (equipa do SaaS)

| Método | Rota | Permissão | Descrição |
|---|---|---|---|
| GET | `/plataforma/escolas` | plataforma | Lista escolas. (*paginado*) |
| POST | `/plataforma/escolas` | plataforma | Provisiona escola numa transação: escola + `fn_provisionar_escola` + filial principal + administrador + subscrição TESTE. Devolve a senha temporária uma única vez. |
| GET | `/plataforma/escolas/:id` | plataforma | Detalhe da escola. |
| PATCH | `/plataforma/escolas/:id/estado` | plataforma | Ativa, suspende ou cancela a escola. |
| GET | `/plataforma/planos` | plataforma | Lista planos. (*paginado*) |
| POST | `/plataforma/planos` | plataforma | Cria plano. |
| PATCH | `/plataforma/planos/:id` | plataforma | Edita plano. |
| PUT | `/plataforma/escolas/:id/subscricao` | plataforma | Define a subscrição vigente da escola. |

#### Escola, identidade e onboarding

| Método | Rota | Permissão | Descrição |
|---|---|---|---|
| GET | `/escola` | `escola:ver` | Dados da escola + identidade (com URLs assinadas dos ficheiros) + configuração + estado do onboarding. |
| PATCH | `/escola` | `escola:editar` | Edita dados da escola. |
| PUT | `/escola/identidade` | `escola_identidade:editar` | Atualiza cores, tema, fonte, cabeçalho e rodapé de documentos. |
| POST | `/escola/identidade/ficheiros/:tipo` | `escola_identidade:editar` | Upload multipart. `tipo` ∈ `logo`, `logo-escuro`, `favicon`, `assinatura`, `carimbo`. |
| DELETE | `/escola/identidade/ficheiros/:tipo` | `escola_identidade:editar` | Remove um ficheiro de identidade. |
| PUT | `/escola/configuracao` | `escola_configuracao:editar` | Atualiza regras operacionais (durações, antecedência, notas). |
| GET | `/escola/janelas-aula-pratica` | `escola:ver` | Janelas permitidas para aulas práticas (lista curta; exceção à paginação). |
| PUT | `/escola/janelas-aula-pratica` | `escola_configuracao:editar` | Substitui todas as janelas. |
| POST | `/escola/onboarding/passos/:passo/concluir` | `escola:editar` | Marca um passo do onboarding como concluído. |
| POST | `/escola/onboarding/concluir` | `escola:editar` | Conclui o onboarding (exige os passos obrigatórios). |
| GET | `/escola/subscricao` | `subscricao:ver` | Plano e subscrição da escola. |
| GET | `/filiais` | `filial:ver` | Lista filiais. (*paginado*) |
| POST | `/filiais` | `filial:criar` | Cria filial. |
| GET | `/filiais/:id` | `filial:ver` | Detalhe da filial. |
| PATCH | `/filiais/:id` | `filial:editar` | Edita filial. |
| PATCH | `/filiais/:id/estado` | `filial:desativar` | Ativa ou desativa filial. |

#### Utilizadores, perfis e permissões

| Método | Rota | Permissão | Descrição |
|---|---|---|---|
| GET | `/utilizadores` | `utilizador:ver` | Lista utilizadores (filtros: `search`, `ativo`, `perfilId`, `filialId`). (*paginado*) |
| POST | `/utilizadores` | `utilizador:criar` | Cria utilizador com perfis e filiais; devolve senha temporária uma vez. |
| GET | `/utilizadores/:id` | `utilizador:ver` | Detalhe do utilizador (perfis, filiais, exceções). |
| PATCH | `/utilizadores/:id` | `utilizador:editar` | Edita dados do utilizador. |
| PATCH | `/utilizadores/:id/estado` | `utilizador:desativar` | Ativa ou desativa (revoga sessões). |
| POST | `/utilizadores/:id/redefinir-senha` | `utilizador:redefinir_senha` | Gera nova senha temporária (uma vez) e revoga sessões. |
| PUT | `/utilizadores/:id/perfis` | `utilizador:editar` | Substitui os perfis `[{perfilId, filialId?}]`. Incrementa `perm_version`. |
| PUT | `/utilizadores/:id/filiais` | `utilizador:editar` | Substitui as filiais do utilizador. |
| GET | `/utilizadores/:id/permissoes-efetivas` | `utilizador:ver` | Permissões efetivas calculadas, com a origem de cada uma (perfil, ALLOW, DENY). |
| PUT | `/utilizadores/:id/permissoes` | `permissao:atribuir` | Substitui as exceções individuais `[{codigo, efeito, escopo, motivo?, expiraEm?}]`. |
| GET | `/perfis` | `perfil:ver` | Lista perfis da escola. (*paginado*) |
| POST | `/perfis` | `perfil:criar` | Cria perfil personalizado. |
| GET | `/perfis/:id` | `perfil:ver` | Detalhe do perfil com permissões e escopos. |
| PATCH | `/perfis/:id` | `perfil:editar` | Edita nome e descrição (perfil protegido: proibido). |
| PUT | `/perfis/:id/permissoes` | `perfil:editar` | Substitui as permissões `[{codigo, escopo}]`. Aplica anti-escalada. |
| POST | `/perfis/:id/duplicar` | `perfil:criar` | Duplica um perfil (útil para variantes). |
| DELETE | `/perfis/:id` | `perfil:eliminar` | Elimina perfil sem utilizadores (protegido: proibido). |
| GET | `/permissoes` | `permissao:ver` | Catálogo agrupado por módulo (exceção à paginação: lista estática). |
| GET | `/auditoria` | `auditoria:ver` | Registo de auditoria (filtros: `utilizadorId`, `entidade`, `acao`, `de`, `ate`). (*paginado*) |

#### Configuração da formação

| Método | Rota | Permissão | Descrição |
|---|---|---|---|
| GET | `/categorias` | `categoria:ver` | Lista categorias de carta. (*paginado*) |
| POST | `/categorias` | `categoria:gerir` | Cria item de categorias de carta. |
| GET | `/categorias/:id` | `categoria:ver` | Detalhe. |
| PATCH | `/categorias/:id` | `categoria:gerir` | Edita. |
| DELETE | `/categorias/:id` | `categoria:gerir` | Elimina (soft delete; bloqueado se em uso). |
| GET | `/curriculos` | `curriculo:ver` | Lista currículos. (*paginado*) |
| POST | `/curriculos` | `curriculo:gerir` | Cria item de currículos. |
| GET | `/curriculos/:id` | `curriculo:ver` | Detalhe. |
| PATCH | `/curriculos/:id` | `curriculo:gerir` | Edita. |
| DELETE | `/curriculos/:id` | `curriculo:gerir` | Elimina (soft delete; bloqueado se em uso). |
| GET | `/pacotes` | `pacote:ver` | Lista pacotes. (*paginado*) |
| POST | `/pacotes` | `pacote:gerir` | Cria item de pacotes. |
| GET | `/pacotes/:id` | `pacote:ver` | Detalhe. |
| PATCH | `/pacotes/:id` | `pacote:gerir` | Edita. |
| DELETE | `/pacotes/:id` | `pacote:gerir` | Elimina (soft delete; bloqueado se em uso). |
| GET | `/curriculos/:id/modulos` | `curriculo:ver` | Módulos do currículo. (*paginado*) |
| PUT | `/curriculos/:id/modulos` | `curriculo:gerir` | Substitui os módulos do currículo. |

#### Instrutores e funcionários

| Método | Rota | Permissão | Descrição |
|---|---|---|---|
| GET | `/instrutores` | `instrutor:ver` | Lista instrutores (filtros: `search`, `funcao`, `categoriaId`, `filialId`, `ativo`). (*paginado*) |
| POST | `/instrutores` | `instrutor:criar` | Cria instrutor. |
| GET | `/instrutores/:id` | `instrutor:ver` | Detalhe. |
| PATCH | `/instrutores/:id` | `instrutor:editar` | Edita. |
| PATCH | `/instrutores/:id/estado` | `instrutor:desativar` | Ativa ou desativa. |
| PUT | `/instrutores/:id/categorias` | `instrutor:editar` | Substitui as categorias habilitadas. |
| GET | `/instrutores/:id/agenda` | `aula_pratica:ver` | Agenda do instrutor num intervalo (`de`, `ate`). (*paginado*) |
| GET | `/funcionarios` | `funcionario:ver` | Lista funcionários. (*paginado*) |
| POST | `/funcionarios` | `funcionario:criar` | Cria funcionário. |
| GET | `/funcionarios/:id` | `funcionario:ver` | Detalhe. |
| PATCH | `/funcionarios/:id` | `funcionario:editar` | Edita. |
| PATCH | `/funcionarios/:id/estado` | `funcionario:desativar` | Ativa ou desativa. |

#### Frota

| Método | Rota | Permissão | Descrição |
|---|---|---|---|
| GET | `/veiculos` | `veiculo:ver` | Lista veículos (filtros: `search`, `estado`, `categoriaId`, `filialId`). (*paginado*) |
| POST | `/veiculos` | `veiculo:criar` | Cria veículo. |
| GET | `/veiculos/:id` | `veiculo:ver` | Detalhe. |
| PATCH | `/veiculos/:id` | `veiculo:editar` | Edita. |
| PATCH | `/veiculos/:id/estado` | `veiculo:desativar` | Muda o estado (DISPONIVEL, EM_MANUTENCAO, INATIVO). |
| GET | `/veiculos/:id/manutencoes` | `manutencao:ver` | Histórico de manutenções. (*paginado*) |
| POST | `/veiculos/:id/manutencoes` | `manutencao:registar` | Regista manutenção. |
| GET | `/veiculos/:id/abastecimentos` | `veiculo:ver` | Histórico de abastecimentos. (*paginado*) |
| POST | `/veiculos/:id/abastecimentos` | `abastecimento:registar` | Regista abastecimento. |

#### Anexos (documentos carregados)

| Método | Rota | Permissão | Descrição |
|---|---|---|---|
| GET | `/tipos-documento` | `tipo_documento:ver` | Lista tipos de documento (filtro `aplicaA`). (*paginado*) |
| POST | `/tipos-documento` | `tipo_documento:gerir` | Cria tipo de documento. |
| PATCH | `/tipos-documento/:id` | `tipo_documento:gerir` | Edita tipo de documento. |
| GET | `/estudantes/:id/anexos` | `anexo:ver` | Checklist do estudante: cada tipo ativo com o anexo mais recente, estado e `expirado`. Tipos sensíveis aparecem mascarados sem `anexo_sensivel:ver`. (*paginado*) |
| POST | `/estudantes/:id/anexos` | `anexo:carregar` | Upload multipart (`tipoDocumentoId`, `validade?`, ficheiro). |
| GET | `/instrutores/:id/anexos` | `anexo:ver` | Anexos do instrutor. (*paginado*) |
| POST | `/instrutores/:id/anexos` | `anexo:carregar` | Upload de anexo do instrutor. |
| GET | `/veiculos/:id/anexos` | `anexo:ver` | Anexos do veículo. (*paginado*) |
| POST | `/veiculos/:id/anexos` | `anexo:carregar` | Upload de anexo do veículo. |
| GET | `/anexos/:id/ficheiro` | `anexo:ver` | Descarrega o ficheiro em stream. Tipos sensíveis exigem também `anexo_sensivel:ver`. (*binário*) |
| PATCH | `/anexos/:id/validacao` | `anexo:validar` | Valida ou rejeita `{estado, observacao?}`. |
| DELETE | `/anexos/:id` | `anexo:eliminar` | Elimina (soft delete). |

#### Estudantes

| Método | Rota | Permissão | Descrição |
|---|---|---|---|
| GET | `/estudantes` | `estudante:ver` | Lista estudantes. Filtros: `search`, `estado`, `filialId`, `categoriaId`, `turmaId`, `turmaAno`, `emAtraso`, `ordenar`, `ordem`. (*paginado*) |
| GET | `/estudantes/pesquisa` | `estudante:ver` | Pesquisa rápida da barra global (`q`, `limit` ≤ 10). Declarar ANTES de `/estudantes/:id`. (*paginado*) |
| GET | `/estudantes/exportar` | `estudante:exportar` | Exporta a listagem filtrada (`formato=pdf\|xlsx`). Declarar ANTES de `/estudantes/:id`. (*binário*) |
| POST | `/estudantes` | `estudante:criar` | Cria estudante; aceita bloco opcional `matricula` (cria tudo numa transação). |
| GET | `/estudantes/:id` | `estudante:ver` | Dados pessoais completos. |
| GET | `/estudantes/:id/resumo` | `estudante:ver` | Dados do cabeçalho do ecrã do estudante: identificação, matrícula principal, turma, média, estado e situação financeira. |
| PATCH | `/estudantes/:id` | `estudante:editar` | Edita dados pessoais. |
| PATCH | `/estudantes/:id/estado` | `estudante:editar` | Muda o estado (ATIVO, INATIVO, CONCLUIDO, DESISTENTE). |
| DELETE | `/estudantes/:id` | `estudante:eliminar` | Soft delete (bloqueado se houver pagamentos confirmados). |
| POST | `/estudantes/:id/foto` | `estudante:editar` | Upload multipart da fotografia. |
| GET | `/estudantes/:id/matriculas` | `matricula:ver` | Matrículas do estudante. (*paginado*) |
| GET | `/estudantes/:id/progresso` | `matricula:ver` | Progresso por matrícula (`vw_progresso_matricula`) + próximos passos. |
| GET | `/estudantes/:id/aulas-praticas` | `aula_pratica:ver` | Aulas práticas do estudante. (*paginado*) |
| GET | `/estudantes/:id/avaliacoes` | `avaliacao:ver` | Avaliações e média. (*paginado*) |
| GET | `/estudantes/:id/financas/resumo` | `cobranca:ver` | Totais: líquido, pago, em aberto, em atraso. |
| GET | `/estudantes/:id/cobrancas` | `cobranca:ver` | Cobranças (de `vw_cobranca`). Filtro `situacao`. (*paginado*) |
| GET | `/estudantes/:id/pagamentos` | `pagamento:ver` | Pagamentos com alocações. (*paginado*) |
| GET | `/estudantes/:id/documentos-emitidos` | `documento:ver` | Documentos emitidos ao estudante. (*paginado*) |
| POST | `/estudantes/:id/acesso` | `estudante_acesso:gerir` | RESERVADO (portal futuro): responde 501 `NAO_IMPLEMENTADO`. |

#### Turmas

| Método | Rota | Permissão | Descrição |
|---|---|---|---|
| GET | `/turmas` | `turma:ver` | Lista turmas. Filtros: `ano`, `estado`, `filialId`, `categoriaId`, `turno`, `instrutorId`, `comVagas`, `search`, `ordenar`, `ordem`. (*paginado*) |
| GET | `/turmas/anos` | `turma:ver` | Anos existentes com contagem `[{ano, total}]` para o filtro de ano (exceção à paginação). Declarar ANTES de `/turmas/:id`. |
| GET | `/turmas/exportar` | `turma:exportar` | Exporta a listagem filtrada (`formato=pdf\|xlsx`). Declarar ANTES de `/turmas/:id`. (*binário*) |
| POST | `/turmas` | `turma:criar` | Cria turma. `ano` é calculado a partir de `dataInicio`. |
| GET | `/turmas/:id` | `turma:ver` | Detalhe com ocupação (`vw_turma_resumo`). |
| PATCH | `/turmas/:id` | `turma:editar` | Edita turma (recalcula `ano` se `dataInicio` mudar). |
| PATCH | `/turmas/:id/estado` | `turma:editar` | Transição de estado (PLANEADA → ABERTA → EM_CURSO → CONCLUIDA). |
| POST | `/turmas/:id/cancelar` | `turma:cancelar` | Cancela a turma `{motivo}`; fecha inscrições e cancela aulas agendadas. |
| GET | `/turmas/:id/estudantes` | `turma:ver` | Estudantes atuais da turma. (*paginado*) |
| POST | `/turmas/:id/estudantes` | `turma_estudante:gerir` | Inscreve uma matrícula na turma `{matriculaId}`. |
| DELETE | `/turmas/:id/estudantes/:matriculaId` | `turma_estudante:gerir` | Remove a matrícula da turma `{motivo}` (fecha `matricula_turma`). |
| POST | `/turmas/:id/estudantes/:matriculaId/transferir` | `turma_estudante:gerir` | Muda de turma `{turmaDestinoId, motivo}` numa transação. |
| GET | `/turmas/:id/aulas` | `aula_teorica:ver` | Aulas teóricas da turma. (*paginado*) |
| POST | `/turmas/:id/aulas/gerar` | `aula_teorica:criar` | Gera as aulas do horário recorrente entre `dataInicio` e `dataFimPrevista`. |
| GET | `/turmas/:id/exportar` | `turma:exportar` | Exporta a lista de estudantes da turma (`formato=pdf\|xlsx`). (*binário*) |

#### Matrículas

| Método | Rota | Permissão | Descrição |
|---|---|---|---|
| GET | `/matriculas` | `matricula:ver` | Lista matrículas (filtros: `estudanteId`, `categoriaId`, `estado`, `turmaId`, `semTurma`, `filialId`, `search`). (*paginado*) |
| POST | `/matriculas` | `matricula:criar` | Cria matrícula (turma e plano de prestações opcionais, na mesma transação). |
| GET | `/matriculas/:id` | `matricula:ver` | Detalhe. |
| PATCH | `/matriculas/:id` | `matricula:editar` | Edita observações, pacote e preço/desconto (desconto exige `desconto:aplicar`). |
| POST | `/matriculas/:id/ativar` | `matricula:editar` | PENDENTE → ATIVA. |
| POST | `/matriculas/:id/suspender` | `matricula:suspender` | ATIVA → SUSPENSA `{motivo}`. |
| POST | `/matriculas/:id/reativar` | `matricula:reativar` | SUSPENSA → ATIVA. |
| POST | `/matriculas/:id/cancelar` | `matricula:cancelar` | Encerra `{tipo: CANCELADA\|DESISTENTE\|TRANSFERIDA, motivo}`. |
| POST | `/matriculas/:id/concluir` | `matricula:editar` | ATIVA → CONCLUIDA. |
| GET | `/matriculas/:id/progresso` | `matricula:ver` | Progresso da matrícula. |
| GET | `/matriculas/:id/elegibilidade-exame` | `exame:ver` | Elegibilidade `?tipo=TEORICO\|PRATICO` com a lista de `bloqueios`. |
| GET | `/matriculas/:id/processo-carta` | `processo_carta:ver` | Processo da carta. |
| PUT | `/matriculas/:id/processo-carta` | `processo_carta:gerir` | Cria ou atualiza o processo da carta. |

#### Aulas teóricas e presenças

| Método | Rota | Permissão | Descrição |
|---|---|---|---|
| GET | `/aulas-teoricas` | `aula_teorica:ver` | Lista aulas (filtros: `turmaId`, `instrutorId`, `de`, `ate`, `estado`). (*paginado*) |
| POST | `/aulas-teoricas` | `aula_teorica:criar` | Cria aula avulsa. |
| GET | `/aulas-teoricas/:id` | `aula_teorica:ver` | Detalhe. |
| PATCH | `/aulas-teoricas/:id` | `aula_teorica:editar` | Edita (tema, sala, horário, instrutor). |
| POST | `/aulas-teoricas/:id/concluir` | `aula_teorica:editar` | AGENDADA → REALIZADA. |
| POST | `/aulas-teoricas/:id/cancelar` | `aula_teorica:cancelar` | Cancela `{motivo}`. |
| GET | `/aulas-teoricas/:id/presencas` | `presenca:ver` | Presenças da aula (inclui estudantes sem registo). (*paginado*) |
| PUT | `/aulas-teoricas/:id/presencas` | `presenca:registar` | Regista presenças em lote `[{matriculaId, estado, observacao?}]` (upsert). |

#### Aulas práticas e avaliações

| Método | Rota | Permissão | Descrição |
|---|---|---|---|
| GET | `/aulas-praticas` | `aula_pratica:ver` | Agenda (filtros: `de`, `ate`, `instrutorId`, `veiculoId`, `matriculaId`, `estudanteId`, `estado`, `filialId`). (*paginado*) |
| GET | `/aulas-praticas/disponibilidade` | `aula_pratica:agendar` | Horários livres para `data`, `categoriaId` e (opcional) `instrutorId`/`veiculoId`. |
| POST | `/aulas-praticas` | `aula_pratica:agendar` | Agenda aula `{matriculaId, instrutorId, veiculoId, inicio, fim?}`. Conflito → 409. |
| GET | `/aulas-praticas/:id` | `aula_pratica:ver` | Detalhe. |
| PATCH | `/aulas-praticas/:id/remarcar` | `aula_pratica:remarcar` | Remarca `{inicio, fim?, instrutorId?, veiculoId?}`. |
| POST | `/aulas-praticas/:id/cancelar` | `aula_pratica:cancelar` | Cancela `{motivo}` (obrigatório se tardio). |
| POST | `/aulas-praticas/:id/concluir` | `aula_pratica:registar` | AGENDADA → REALIZADA `{kmInicial?, kmFinal?, observacoes?}`. |
| POST | `/aulas-praticas/:id/falta` | `aula_pratica:registar` | Marca FALTOU (estudante não compareceu). |
| GET | `/avaliacoes` | `avaliacao:ver` | Lista avaliações (filtros: `matriculaId`, `estudanteId`, `tipo`, `de`, `ate`). (*paginado*) |
| POST | `/avaliacoes` | `avaliacao:registar` | Regista avaliação. |
| PATCH | `/avaliacoes/:id` | `avaliacao:registar` | Edita avaliação. |

#### Exames e processo da carta

| Método | Rota | Permissão | Descrição |
|---|---|---|---|
| GET | `/exames/sessoes` | `exame:ver` | Lista sessões de exame (filtros: `tipo`, `categoriaId`, `de`, `ate`, `estado`). (*paginado*) |
| POST | `/exames/sessoes` | `exame:gerir` | Cria sessão. |
| GET | `/exames/sessoes/:id` | `exame:ver` | Detalhe com ocupação. |
| PATCH | `/exames/sessoes/:id` | `exame:gerir` | Edita sessão. |
| POST | `/exames/sessoes/:id/cancelar` | `exame:gerir` | Cancela sessão `{motivo}`. |
| GET | `/exames/sessoes/:id/inscricoes` | `exame:ver` | Inscrições da sessão. (*paginado*) |
| POST | `/exames/sessoes/:id/inscricoes` | `exame_inscricao:criar` | Inscreve `{matriculaId, forcar?}`. Valida elegibilidade; `forcar` exige `exame:gerir`. |
| PATCH | `/exames/inscricoes/:id/resultado` | `exame_inscricao:resultado` | Regista `{estado, resultado, nota?}`. |
| POST | `/exames/inscricoes/:id/cancelar` | `exame_inscricao:criar` | Cancela inscrição. |

#### Financeiro

| Método | Rota | Permissão | Descrição |
|---|---|---|---|
| GET | `/cobrancas` | `cobranca:ver` | Lista cobranças (de `vw_cobranca`). Filtros: `estudanteId`, `matriculaId`, `situacao`, `tipo`, `vencimentoDe`, `vencimentoAte`, `filialId`. (*paginado*) |
| POST | `/cobrancas` | `cobranca:criar` | Cria cobrança avulsa ou lista de prestações. |
| GET | `/cobrancas/:id` | `cobranca:ver` | Detalhe com pagamentos alocados. |
| POST | `/cobrancas/:id/anular` | `cobranca:anular` | Anula `{motivo}` (bloqueado se tiver pagamentos). |
| GET | `/pagamentos` | `pagamento:ver` | Lista pagamentos (filtros: `estudanteId`, `metodo`, `estado`, `de`, `ate`, `filialId`). (*paginado*) |
| POST | `/pagamentos` | `pagamento:registar` | Regista pagamento com alocações. Header `Idempotency-Key` recomendado. |
| GET | `/pagamentos/:id` | `pagamento:ver` | Detalhe com alocações. |
| POST | `/pagamentos/:id/comprovativo` | `pagamento:registar` | Upload multipart do comprovativo. |
| POST | `/pagamentos/:id/anular` | `pagamento:anular` | Anula `{motivo}`; anula também o recibo emitido. |
| GET | `/despesas` | `despesa:ver` | Lista despesas (filtros: `categoria`, `de`, `ate`, `veiculoId`, `filialId`). (*paginado*) |
| POST | `/despesas` | `despesa:registar` | Regista despesa. |
| PATCH | `/despesas/:id` | `despesa:registar` | Edita despesa. |
| DELETE | `/despesas/:id` | `despesa:eliminar` | Elimina (soft delete). |
| GET | `/financeiro/resumo` | `financeiro_relatorio:ver` | Receitas, em aberto, em atraso, despesas, saldo e `serieMensal` (`de`, `ate`, `filialId`). |

#### Documentos emitidos, relatórios e painel

| Método | Rota | Permissão | Descrição |
|---|---|---|---|
| POST | `/documentos/gerar` | `documento:gerar` | Gera o documento e devolve o PDF pronto `{tipo, estudanteId?, matriculaId?, pagamentoId?}`. Headers: `X-Documento-Id`, `X-Documento-Numero`. (*binário*) |
| GET | `/documentos/:id/ficheiro` | `documento:ver` | Descarrega novamente um documento emitido. (*binário*) |
| POST | `/documentos/:id/anular` | `documento:anular` | Anula `{motivo}`. |
| GET | `/relatorios` | `relatorio:ver` | Catálogo de relatórios disponíveis ao utilizador (exceção à paginação). |
| GET | `/relatorios/:codigo/exportar` | `relatorio:exportar` | Exporta `formato=pdf\|xlsx` + parâmetros do relatório. Exige também a permissão própria do relatório. (*binário*) |
| GET | `/dashboard` | `dashboard:ver` | Indicadores do painel respeitando o escopo do utilizador. |

#### Notificações

| Método | Rota | Permissão | Descrição |
|---|---|---|---|
| GET | `/notificacoes` | `notificacao:ver` | Notificações do utilizador autenticado (filtro `naoLidas`). (*paginado*) |
| PATCH | `/notificacoes/:id/lida` | `notificacao:ver` | Marca como lida. |
| POST | `/notificacoes/marcar-todas-lidas` | `notificacao:ver` | Marca todas como lidas. |
| POST | `/notificacoes` | `notificacao:enviar` | Envia `{estudanteId\|utilizadorId, canal, assunto?, mensagem, agendadaPara?}`. Só `INTERNA` funciona na primeira entrega. |

Formas JSON especiais (`/auth/me`, resumo e progresso do estudante, disponibilidade, painel, resumo financeiro, corpos de criação) estão em BACKEND_KB.md, secções 10.1 e 10.2.
