# Evolução Arquitetural: De SPA Monolítica a Micro Frontends

> **Contexto:** Startup de e-commerce em crescimento, produto atual é uma SPA React pura (monólito frontend), empresa expandindo para 3 times independentes.

---

## Seção 1 — Diagnóstico do Monólito

### Problema 1 — Deploy Acoplado (Gargalo de Entrega)

**Sintoma:** Qualquer alteração, por menor que seja, exige um deploy completo da aplicação. Um bug corrigido no checkout bloqueia o lançamento de novas funcionalidades no catálogo.

**Causa Técnica:** Em uma SPA pura, todo o código é empacotado em um único bundle pelo Webpack/Vite. Não existe isolamento de módulos em tempo de build ou em runtime — o artefato é indivisível. Com 3 times trabalhando no mesmo repositório, surgem conflitos de merge frequentes, filas de PR e dependências implícitas entre features.

**Impacto no Negócio:** Cadência de deploy reduzida. Segundo o relatório *Accelerate State of DevOps* (DORA, 2023), times de alta performance fazem múltiplos deploys por dia. Um monólito compartilhado entre 3 times tende a reduzir essa frequência drasticamente, atrasando entregas de valor ao cliente e aumentando o risco de cada release (quanto maior o lote, maior a chance de falha).

---

### Problema 2 — Performance e SEO Prejudicados (FCP / LCP Ruins)

**Sintoma:** O usuário vê uma tela em branco por vários segundos antes do conteúdo aparecer. O Google não indexa corretamente as páginas de produto.

**Causa Técnica:** SPAs puras dependem de JavaScript no cliente para renderizar o conteúdo (Client-Side Rendering — CSR). O First Contentful Paint (FCP) só acontece após o browser: (1) baixar o HTML mínimo, (2) baixar e parsear o bundle JS, (3) executar o React, (4) buscar dados via API e (5) renderizar o DOM. Em e-commerces com catálogos grandes, bundles frequentemente ultrapassam 1MB.

**Impacto no Negócio:** O Google utiliza Core Web Vitals (LCP, FID, CLS) como fator de ranqueamento desde 2021. Uma SPA não otimizada perde posições orgânicas para concorrentes com SSR/SSG. Estudos da Vodafone e Rakuten documentados pelo Google mostram que melhorar o LCP em 31% aumentou as conversões em 8% — diretamente ligado a receita.

---

### Problema 3 — Acoplamento de Código e Dívida Técnica Crescente

**Sintoma:** Alterar o componente de `ProductCard` quebra o carousel da home. Mudanças no contexto de autenticação afetam o checkout. Ninguém sabe ao certo o que pode ser alterado com segurança.

**Causa Técnica:** Em uma SPA monolítica, componentes compartilham o mesmo estado global (Redux, Context API), as mesmas dependências de pacotes e os mesmos tipos TypeScript. Sem fronteiras explícitas, o acoplamento cresce organicamente. A Lei de Conway afirma que sistemas de software tendem a espelhar a estrutura de comunicação das organizações que os constroem — 3 times em 1 repositório = caos arquitetural.

**Impacto no Negócio:** Aumento do lead time para mudanças, aumento de bugs em regressão e maior custo de onboarding para novos engenheiros. O custo de manutenção de software aumenta exponencialmente com o acoplamento, conforme documentado em *Software Engineering at Google* (Winters et al., 2020).

---

### Problema 4 — Autonomia de Times Comprometida

**Sintoma:** O Time de Checkout precisa esperar o Time de Catálogo terminar sua feature para poder fazer o merge e subir para produção. Há reuniões de alinhamento constantes para decidir qual time "dona" cada componente compartilhado.

**Causa Técnica:** Sem fronteiras técnicas (módulos isolados, contratos de API, ownership de pastas), todos os times operam no mesmo espaço de trabalho. Decisões de tecnologia (versão do React, biblioteca de UI, estratégia de estado) são globais e afetam todos.

**Impacto no Negócio:** Redução da velocidade de desenvolvimento, aumento de conflitos interpessoais e dificuldade em escalar a engenharia. O modelo de times de *Team Topologies* (Skelton & Pais, 2019) demonstra que times com alto acoplamento inter-equipe têm throughput significativamente menor que times com fronteiras bem definidas.

---

## Seção 2 — Proposta de Arquitetura

### Visão Geral: Shell + 3 Micro Frontends

A proposta adota **Module Federation (Webpack 5)** como mecanismo de composição, com **Next.js** como framework base para cada micro frontend. Um **App Shell** central orquestra o roteamento e carrega os MFEs dinamicamente.

```
┌─────────────────────────────────────────────────────────┐
│                      CDN / Edge                         │
│              (Cloudfront / Vercel Edge)                 │
└────────────┬──────────────────────────┬─────────────────┘
             │                          │
             ▼                          ▼
┌────────────────────┐      ┌───────────────────────────┐
│    App Shell       │      │     Design System          │
│  (Next.js SSR)     │      │  (Shared Component Lib)   │
│  - Routing global  │      │  - Tokens, UI primitivos  │
│  - Auth/Session    │      │  - Publicado como npm pkg  │
│  - Header/Footer   │      └───────────────────────────┘
└────┬──────┬────────┘
     │      │
     │  Module Federation (runtime)
     │      │
     ├──────▼──────────────┐
     │                     │
┌────▼───────┐  ┌──────────▼──┐  ┌────────────────────┐
│  MFE-1     │  │   MFE-2     │  │      MFE-3          │
│  Catálogo  │  │  Checkout   │  │  Conta do Usuário   │
│  (Next.js) │  │  (Next.js)  │  │    (Next.js)        │
│  SSG + ISR │  │    SSR      │  │      CSR            │
└────┬───────┘  └──────┬──────┘  └──────────┬──────────┘
     │                 │                     │
     ▼                 ▼                     ▼
┌─────────┐    ┌──────────────┐    ┌─────────────────┐
│Catalog  │    │ Orders API   │    │  Auth API /     │
│  API    │    │ Payment GW   │    │  User Profile   │
└─────────┘    └──────────────┘    └─────────────────┘
```

---

### MFE-1 — Catálogo de Produtos

**Domínio de negócio:** Listagem de produtos, página de detalhe (PDP), busca, filtros e categorias.

**Estratégia de Renderização: SSG + ISR (Incremental Static Regeneration)**

O catálogo tem conteúdo semi-estático: produtos raramente mudam a cada segundo, mas precisam estar atualizados. O ISR do Next.js permite gerar páginas estáticas em build time e revalidá-las em background com um intervalo configurável (ex: `revalidate: 60`). Isso entrega:
- FCP/LCP excelentes (HTML pré-gerado servido pelo CDN)
- Indexabilidade perfeita para SEO (conteúdo no HTML inicial)
- Escalabilidade sem pressionar os servidores de origem

**Module Federation:**
- **Expõe:** `ProductCard`, `ProductGrid`, `SearchBar` (consumidos pelo Shell na Home)
- **Consome:** Tokens do Design System

**Comunicação com outros MFEs:**
- Ao adicionar ao carrinho: dispara evento customizado via `window.CustomEvent` (`cart:item-added`) que o Shell escuta e repassa ao MFE de Checkout
- Não há chamada direta entre MFEs — comunicação via eventos desacoplados

---

### MFE-2 — Checkout

**Domínio de negócio:** Carrinho de compras, endereço de entrega, seleção de frete, pagamento e confirmação de pedido.

**Estratégia de Renderização: SSR (Server-Side Rendering)**

O checkout é altamente dinâmico e personalizado: depende do usuário logado, do carrinho atual, de regras de frete em tempo real e de cálculos de desconto. SSR garante que o HTML inicial já contenha os dados corretos do usuário, evitando flickers de conteúdo (CLS). Além disso, o checkout **não deve ser cacheado** de forma agressiva — cada request deve gerar uma resposta fresca, o que torna SSG/ISR inadequados.

Segurança também é um fator: com SSR, tokens de sessão e dados sensíveis podem ser validados no servidor antes de renderizar qualquer coisa.

**Module Federation:**
- **Expõe:** `CartWidget` (mini-carrinho flutuante consumido pelo Shell/Header)
- **Consome:** `ProductCard` do MFE de Catálogo (para exibir itens no carrinho)

**Comunicação com outros MFEs:**
- Escuta eventos `cart:item-added` do MFE de Catálogo
- Ao finalizar pedido, emite `order:completed` para o Shell atualizar o estado de navegação

---

### MFE-3 — Conta do Usuário

**Domínio de negócio:** Perfil, histórico de pedidos, endereços salvos, lista de favoritos e configurações de notificação.

**Estratégia de Renderização: CSR (Client-Side Rendering)**

A área de conta do usuário é **100% autenticada** — não há nenhum benefício de SEO, pois motores de busca não indexam conteúdo atrás de login. Todo o conteúdo é profundamente personalizado e muda frequentemente (ex: "meu último pedido"). CSR com React puro é adequado aqui, reduzindo a carga nos servidores. O Next.js permite CSR seletivo com `dynamic(() => import(...), { ssr: false })`.

**Module Federation:**
- **Expõe:** `UserAvatar`, `AuthStatus` (consumidos pelo Shell no Header)
- **Consome:** Nada dos outros MFEs diretamente

**Comunicação com outros MFEs:**
- Expõe o estado de autenticação via `window.__AUTH__` (objeto controlado pelo Shell) para que outros MFEs possam verificar se o usuário está logado sem chamadas diretas

---

## Seção 3 — Componentes e Boas Práticas

### MFE escolhido: Catálogo de Produtos

O Catálogo foi escolhido por apresentar a maior riqueza de casos de uso: mistura de Server e Client Components, uso estratégico de Suspense e a necessidade de balancear performance com interatividade.

---

### Árvore de Componentes

```
app/
├── layout.tsx                          [SERVER] — Shell do MFE, providers globais
│
├── (catalog)/
│   ├── page.tsx                        [SERVER] — Listagem principal, busca SSG
│   ├── loading.tsx                     [SERVER] — Streaming Suspense fallback
│   │
│   ├── components/
│   │   ├── ProductGrid.tsx             [SERVER] — Renderiza grid com dados do servidor
│   │   │   └── ProductCard.tsx         [SERVER] — Card estático com dados do produto
│   │   │       └── AddToCartButton.tsx [CLIENT] — Interação, precisa de onClick/estado
│   │   │
│   │   ├── SearchBar.tsx               [CLIENT] — Input controlado, debounce, estado local
│   │   │
│   │   ├── FilterPanel.tsx             [CLIENT] — Accordion com estado de aberto/fechado
│   │   │   ├── PriceRangeSlider.tsx    [CLIENT] — Slider interativo
│   │   │   └── CategoryCheckbox.tsx    [CLIENT] — Checkbox com estado
│   │   │
│   │   └── SortDropdown.tsx            [CLIENT] — Select controlado
│   │
├── [slug]/
│   ├── page.tsx                        [SERVER] — PDP, gerada via ISR
│   │
│   └── components/
│       ├── ProductImages.tsx           [CLIENT] — Galeria com zoom e swipe
│       ├── ProductInfo.tsx             [SERVER] — Nome, preço, descrição (estático)
│       ├── VariantSelector.tsx         [CLIENT] — Seleção de cor/tamanho (estado)
│       ├── AddToCartForm.tsx           [CLIENT] — Formulário com quantidade
│       └── RelatedProducts.tsx         [SERVER] — Grid de produtos relacionados
│           └── <Suspense>              [SERVER] — Streaming independente
```

**Regra de decisão Server vs Client:**

| Critério | Server Component | Client Component |
|---|---|---|
| Precisa de `useState` / `useEffect` | ❌ | ✅ |
| Precisa de event handlers (`onClick`) | ❌ | ✅ |
| Acessa banco/API diretamente | ✅ | ❌ |
| Conteúdo estático ou semi-estático | ✅ | ❌ |
| Usa APIs do browser (`localStorage`) | ❌ | ✅ |
| Contribui para o bundle JS do cliente | ❌ (zero!) | ✅ |

---

### Uso de Suspense

Suspense é utilizado em três situações críticas no Catálogo:

**1. Streaming de dados lentos na listagem**

```tsx
// app/(catalog)/page.tsx — SERVER COMPONENT
export default function CatalogPage() {
  return (
    <main>
      <SearchBar />  {/* Client: renderiza imediatamente */}

      <Suspense fallback={<ProductGridSkeleton count={12} />}>
        {/* ProductGrid busca dados do servidor — streameia quando pronto */}
        <ProductGrid />
      </Suspense>
    </main>
  );
}
```

**Por quê:** Sem Suspense, a página inteira aguardaria o fetch de produtos para começar a renderizar. Com Suspense + streaming, o usuário vê o esqueleto imediatamente e o grid aparece quando os dados chegam — sem JavaScript extra no cliente.

**2. Produtos relacionados na PDP (carregamento independente)**

```tsx
// app/[slug]/page.tsx — SERVER COMPONENT
export default function ProductDetailPage({ params }) {
  return (
    <>
      <ProductInfo slug={params.slug} />

      {/* RelatedProducts é lento (recomendação por ML) — não bloqueia a PDP */}
      <Suspense fallback={<RelatedProductsSkeleton />}>
        <RelatedProducts slug={params.slug} />
      </Suspense>
    </>
  );
}
```

**Por quê:** O algoritmo de recomendação de produtos relacionados pode ter latência variável. Isolar em Suspense garante que a informação principal do produto (preço, descrição, imagens) chegue ao usuário o mais rápido possível, enquanto as recomendações carregam de forma não-bloqueante.

**3. Lazy loading de Client Components pesados**

```tsx
// Galeria de imagens com biblioteca de zoom — carregada apenas na PDP
const ProductImages = dynamic(
  () => import('./components/ProductImages'),
  {
    loading: () => <ImagePlaceholder />,
    ssr: false  // Biblioteca usa APIs do browser
  }
);
```

**Por quê:** Bibliotecas de galeria com zoom/swipe (ex: `react-image-gallery`, `yet-another-react-lightbox`) adicionam ~80-150KB ao bundle. Com `next/dynamic`, esse peso só é baixado quando o usuário acessa a PDP, não na listagem.

---

## Seção 4 — Trade-offs e Decisão Final

### Por que Micro Frontends com Next.js e Module Federation superam a SPA monolítica?

A principal justificativa para a transição arquitetural é a **correspondência entre estrutura técnica e estrutura organizacional**. Com três times independentes, manter um monólito React gera fricção de alto custo: deploys acoplados, conflitos de merge constantes e ausência de ownership claro. A proposta com Module Federation permite que cada time implante seu MFE de forma completamente autônoma, sem coordenação com outros times — alinhando-se ao princípio de *Conway's Law* invertido, onde a arquitetura guia a colaboração. A escolha do Next.js resolve simultaneamente o problema de SEO e performance: o MFE de Catálogo com ISR entrega HTML indexável ao Googlebot sem abrir mão da dinamicidade, enquanto o SSR no Checkout garante segurança e consistência de dados sensíveis. O principal trade-off dessa abordagem é a **complexidade operacional adicional**: múltiplos pipelines de CI/CD, gestão de versões entre MFEs e o overhead de configuração do Module Federation exigem maturidade de engenharia. Adicionalmente, há risco de duplicação de dependências no bundle final caso o Module Federation não seja configurado corretamente para compartilhar React e bibliotecas core. No entanto, conforme documentado por Luca Mezzalira em *Building Micro-Frontends* (O'Reilly, 2021), esses custos operacionais são compensados pelo ganho exponencial em autonomia de times e velocidade de entrega à medida que a organização cresce. Para a startup neste momento de escala, o investimento em micro frontends é justificado — e o Module Federation com Next.js representa a abordagem mais pragmática por reaproveitar o conhecimento React existente nos times enquanto introduz fronteiras técnicas reais.

---

## Referências

- DORA. *Accelerate State of DevOps Report*, 2023. https://dora.dev/research/
- Winters, T.; Manshreck, T.; Wright, H. *Software Engineering at Google*. O'Reilly Media, 2020.
- Skelton, M.; Pais, M. *Team Topologies*. IT Revolution Press, 2019.
- Mezzalira, L. *Building Micro-Frontends*. O'Reilly Media, 2021.
- Next.js Documentation — Rendering Strategies. https://nextjs.org/docs/app/building-your-application/rendering
- Webpack Module Federation. https://webpack.js.org/concepts/module-federation/
- Google. *Core Web Vitals and SEO*, 2023. https://developers.google.com/search/docs/appearance/core-web-vitals
- Rakuten Case Study — Web Performance. https://web.dev/case-studies/rakuten
