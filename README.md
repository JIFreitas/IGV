# Dashboard Interativo de Criptoativos — Documentação Completa

Este repositório contém uma aplicação web que apresenta um painel (dashboard) interativo para exploração de dados de criptoativos. O objectivo é permitir a comparação rápida de indicadores temporais entre moedas, personalização da disposição dos cartões e uma experiência responsiva e robusta face a limites da API.

Este ficheiro reúne toda a informação relevante sobre o que a aplicação faz, como a executar, como funciona internamente e sugestões para evoluções futuras.

## Sumário das funcionalidades
A aplicação oferece as seguintes capacidades principais:

- Painel composto por cartões (cards), cada um com um gráfico D3 e controlos próprios.
- Lista de KPIs (mínimo 6) que podem ser arrastados (drag & drop) para o painel. Ao largar um KPI, é criado o cartão correspondente.
- Reordenação dos cartões por arrastar e largar.
- Remoção individual de cartões e possibilidade de os voltar a adicionar.
- Seleção de moedas para cada cartão; alguns KPIs aceitam múltiplas moedas para comparação.
- Períodos temporais ajustáveis: 1 dia, 1 mês, 1 ano.
- Persistência da disposição do painel (`localStorage`) e restauração ao recarregar.
- Botão "Atualizar" que limpa cache e reobtém dados de mercado para todos os cartões.
- Indicação de estados (loading, erro) por cartão, com *toasts* informativos.

## KPIs e tipos de gráfico
Os KPIs implementados são:

| KPI | Tipo de gráfico | O que mostra | Múltiplas moedas |
|-----|-----------------|--------------|-----------------|
| Evolução do preço | Linha (`line`) | Preço ao longo do período selecionado | ✅ Sim |
| Retorno acumulado (%) | Linha (`line`) | Ganho/Perda em % desde o início do período | ✅ Sim |
| Variação diária | Barras (`bar`) | Variação % dia a dia (cores para alta/baixa) | ❌ Não |
| Distribuição de preços | Histograma | Frequência de preços em 10 faixas | ❌ Não |
| Dias subida vs descida | Donut (`pie`/`donut`) | Proporção de dias em alta vs baixa | ❌ Não |
| Comparação normalizada (%) | Linha (`line`) | Normaliza 2+ moedas a 100% para comparar performance | ✅ Sim |

> Nota: A opção "Máximo" foi removida dos períodos por decisão de consistência e disponibilidade de dados.

## Fonte de dados e comunicação com a API

- API utilizada: CoinCap Pro (v3).
- Endpoints principais:
  - `GET /assets?limit=50` — lista de moedas para os seletores.
  - `GET /assets/{id}/history?interval=d1&start={ms}&end={ms}` — histórico diário de preços.
- Autenticação: Bearer token no cabeçalho `Authorization: Bearer <TOKEN>` (token privado do utilizador).

Restrições e estratégias de robustez:

- A API pode impor limites de taxa; a aplicação implementa:
  - Espaçamento entre pedidos (≈350 ms) para reduzir picos de requisições.
  - Retry com backoff exponencial para respostas `429` (e uso de `Retry-After` quando disponível).
  - Cache em memória indexado por `(coinId, período)` para evitar pedidos redundantes.

## Persistência e formato do estado

- O estado do dashboard é guardado em `localStorage` sob a chave `dashboardState`.
- Estrutura geral do estado:

```json
{
  "cards": [
    { "id": "card-1", "kpi": "price", "coins": ["bitcoin"], "period": "1M", ... }
  ],
  "layout": { ... }
}
```

## Interface e experiência do utilizador

- Grid de cartões com dimensão mínima configurada para evitar cortes na renderização (ex.: 700×550 px por cartão).
- Cada cartão mostra um cabeçalho com título, selectores (moeda, período) e botões (refresh, delete).
- Para KPIs que suportam múltiplas moedas, a interface mostra um seletor multi-valor.
- Toasters informam sobre sucesso/erro (guardado, limitada API, atualizações).

## Estrutura do código

Principais ficheiros e papéis:

- `index.html` — estrutura base da página.
- `src/main.js` — bootstrap do Svelte.
- `src/App.svelte` — lógica principal (estado, API, D3, drag-drop). Atualmente monolítico (~1400 linhas).
- `src/styles.css` — estilos e layout.
- `package.json` — dependências: `svelte`, `vite`, `d3`, `sortablejs`, etc.

Organização lógica dentro de `src/App.svelte`:

1. Importações (D3, SortableJS, Svelte helpers).
2. Configurações (URLs, tokens, constantes de retry e cache).
3. `KPI_CONFIG` definindo construção/transformação/render para cada KPI.
4. Estado reativo do dashboard e listas de moedas.
5. Utilitários de formatação e cache.
6. `fetch` com backoff e tratamento de 429.
7. Funções `build<KPI>` que transformam dados da API.
8. Funções `render<KPI>` que desenham os gráficos com D3.
9. Handlers UI: addChart, removeChart, changePeriod, toggleCoinSelection.
10. Lifecycle: `onMount` inicializa SortableJS e carrega estado salvo.

## Execução e desenvolvimento

Requisitos: Node.js e npm (para ambiente de desenvolvimento com Vite). Para servir estático, um servidor HTTP simples é suficiente.

Instalar dependências e correr em modo dev:



Ou servir os ficheiros estáticos com `python -m http.server` a partir da pasta `dist`.

## Como testar funcionalmente (checklist rápido)

1. Arrastar um KPI para o painel e confirmar que o gráfico aparece.
2. Seleccionar várias moedas num KPI que suporte multi-coin e verificar a comparação.
3. Trocar períodos (1D/1M/1A) e validar actualização dos dados e gráficos.
4. Reordenar cartões e gravar; recarregar a página e confirmar restauração do estado.
5. Clicar em "Atualizar" e observar toasts de feedback e timestamp de atualização.

## Limitações conhecidas

- API CoinCap não fornece volume histórico, por isso os KPIs relacionados com volume foram removidos ou redesenhados.
- `src/App.svelte` está grande; leitura e manutenção são mais difíceis que num design componentizado.
- O projecto assume um token Bearer para CoinCap; a gestão segura desse token não está implementada (por ex., backend middleware).

## Possíveis melhorias (prioritárias)

- Refactor: dividir `src/App.svelte` em componentes (`ChartCard.svelte`, `CoinSelector.svelte`, `PeriodSelector.svelte`) para melhorar legibilidade e testes.
- adicionar mais kpis e melhorar os kpis existentes.
- Export: adicionar funcionalidade para exportar imagens/PDF dos gráficos.


## Contacto
João Freitas — joao.freitas@ipvc.pt

---

Trabalho prático desenvolvido no âmbito do Mestrado em Engenharia Informática do IPVC — unidade curricular de Informação Geográfica e Visualização.
