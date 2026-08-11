# RotaHub — Design System

Referência do padrão visual usado nos 3 frontends (`rotahub-web`, `rotahub-customer-web`,
`rotahub-web-shell`). Não existe um pacote compartilhado de UI — cada repo implementa isso
independentemente com Tailwind CSS, de propósito (ver decisão de polyrepo em
`docs/contracts.md`). Este documento é a fonte de verdade de qual padrão seguir ao adicionar algo
novo, pra manter os 3 visualmente consistentes sem acoplar o código.

## Paleta

| Papel | Classe Tailwind | Uso |
|---|---|---|
| Marca / ação primária | `indigo-600` (hover `indigo-700`) | Botões primários, aba ativa, logo |
| Fundo da página | `slate-50` | `<body>` / container raiz |
| Fundo de card | `white` | Cards, tabelas, painéis |
| Texto principal | `slate-900` | Títulos |
| Texto secundário | `slate-700` | Corpo, valores |
| Texto terciário | `slate-500` | Labels, legendas, uppercase headers |
| Texto desabilitado/placeholder | `slate-400` | Placeholders, estados vazios |
| Borda padrão | `slate-200` | Bordas de card, inputs |
| Borda sutil | `slate-100` | Divisores internos, linhas de tabela |

Neutros usam **slate**, não `gray` puro — o leve viés azulado combina com o indigo da marca em
vez de brigar com ele.

### Cores semânticas de status

Usadas em pills e pontos de timeline (`StatusPill`, `TrackingPanel`, `RoutePlanner`). Nunca usar a
cor de marca (indigo) pra status — indigo é reservado pra ações/navegação.

| Status | Fundo | Texto | Ponto |
|---|---|---|---|
| Neutro (`CREATED`, `AWAITING_PICKUP`, `PLANNED`) | `slate-100` | `slate-700` | `slate-400` |
| Em progresso (`PICKED_UP`, `IN_TRANSIT`, `OUT_FOR_DELIVERY`) | `amber-100` | `amber-800` | `amber-500` |
| Sucesso (`DELIVERED`) | `emerald-100` | `emerald-800` | `emerald-500` |
| Erro/parado (`CANCELLED`, `FAILED_ATTEMPT`) | `rose-100` | `rose-700` | `rose-500` |

## Tipografia

Pilha de fontes do sistema (`-apple-system, "Segoe UI", Roboto, ui-sans-serif, system-ui,
sans-serif`) — deliberado, não default esquecido: carrega instantâneo, sem FOUT/FOIT, e as fontes
nativas de cada SO já são de alta qualidade. Nenhum webfont é importado.

- Números que alinham em coluna (códigos de rastreio, datas, distâncias) sempre levam
  `tabular-nums`
- Labels em maiúsculas (`Novo pedido`, `Pedidos`) usam `text-xs font-semibold tracking-wide
  uppercase text-slate-500`
- Códigos de rastreio usam `font-mono`

## Componentes recorrentes

**Card**: `rounded-xl border border-slate-200 bg-white shadow-sm` — usado em formulários,
tabelas, painéis de resultado.

**Pill de status** (`StatusPill` em `rotahub-web`; padrão replicado manualmente nos outros dois):
pílula arredondada com ponto colorido à esquerda + texto, usando a tabela de cores semânticas
acima.

**Timeline**: lista vertical de eventos, cada item com um ponto colorido (mesma paleta semântica)
conectado por uma linha (`w-px bg-slate-200`) ao próximo item — usada no histórico de rastreio
(`TrackingPanel`, `rotahub-customer-web`) e na rota otimizada (`RoutePlanner`, com números em vez
de cor de status).

**Campo de formulário**: label pequeno (`text-xs font-medium text-slate-500`) acima do input,
input com `rounded-md border border-slate-200 px-3 py-2 text-sm`, foco em
`focus:border-indigo-500 focus:ring-2 focus:ring-indigo-500/20`.

## Onde isso vive

Cada repo implementa isso em `src/index.css` (import do Tailwind + `@theme` com a pilha de
fontes) e nas classes utilitárias direto nos componentes. Se este documento e o código
divergirem, o código é a fonte de verdade — atualize este arquivo para bater.
