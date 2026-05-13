---
name: proposta-dgenny
description: Cria e publica uma proposta comercial dgenny® no WordPress (proposta.dgenny.com.br) para um cliente em fase avançada de negociação. Segue exatamente o padrão visual da proposta Moinho — dark theme, canvas em branco, seções de diagnóstico, ROI, plano e CTA. TRIGGER quando usuário pedir para criar proposta comercial para um cliente.
---

# Skill: Proposta Comercial dgenny®

## O que esta skill faz

Gera e publica automaticamente uma página de proposta comercial personalizada no WordPress `proposta.dgenny.com.br`, seguindo o padrão visual e estrutural da proposta de referência (Moinho — ID 522).

## Credenciais e referências

- **WordPress:** credenciais em memória em `reference_wordpress.md` (URL: `https://proposta.dgenny.com.br`, usuário: `tierre`)
- **Página de referência visual:** ID 522 (`/proposta-moinho/`) — buscar via API para reutilizar os SVGs do logo
- **Template obrigatório:** `elementor_canvas` (canvas em branco, sem header/footer do tema)
- **Meta obrigatório:** `_elementor_edit_mode: ""` — desativa Elementor para não quebrar o HTML
- **Botão CTA:** sempre linkar para `https://www.dgenny.com.br/registro-cliente` com `target="_blank"`

## Argumentos esperados

O usuário pode passar:
- **URL de Google Doc** com notas da reunião → buscar com WebFetch (tentar export txt, seguir redirect se necessário)
- **Dados da proposta** diretamente no argumento:
  - Nome do cliente
  - Volume de compras/mês (R$)
  - Plano (semestral, anual, etc.)
  - Valor mensal (R$)
  - Número de assistentes de IA
  - Valor de implantação (R$)
  - Usuário(s) gestor(es) sem assistente vinculado
  - ERP utilizado e status de integração
  - Dores/desafios identificados na reunião
  - Validade da proposta
  - Qualquer outro dado relevante

Se faltar algum dado essencial (como volume de compras), use as notas da reunião para inferir. Se não houver notas e o dado for crítico para o ROI, pergunte antes de continuar.

## Fluxo de execução

### 1. Coletar dados
- Se houver URL de Google Doc: buscar via `WebFetch` com export `?format=txt`, seguir redirect se retornar 307
- Extrair todas as informações relevantes: contexto do cliente, dores, dados financeiros, termos negociados

### 2. Buscar SVGs do logo na página Moinho
- Via API: `GET /wp-json/wp/v2/pages/522?context=edit`
- Extrair o SVG principal (logo com `width="160"`) e o SVG do footer (com `width="100"`)
- Usar esses SVGs exatos no HTML da proposta

### 3. Calcular ROI
Com base no volume de compras/mês informado, calcular 3 cenários de saving:
- **Conservador:** 5% de saving
- **Realista:** 8% de saving  
- **Projetado:** 12% de saving

Para cada cenário calcular:
- Economia/mês = volume × percentual
- Economia no período contratado = economia/mês × duração em meses
- Investimento total = (mensalidade × meses) + implantação
- Ganho líquido = economia período − investimento total
- ROI (%) = (ganho líquido / investimento total) × 100 — arredondar para inteiro
- Payback = investimento total / (economia/mês) → converter para dias (× 30)

Exibir ROI realista nas hero metrics.

### 4. Montar o HTML

Seguir **exatamente** a estrutura CSS e HTML da proposta Moinho. Reusar todo o CSS — apenas alterar conteúdo textual. Estrutura de seções:

1. **HERO** — logo SVG, badge com tipo de plano, h1, hero-sub com nome do cliente, hero-metrics (4 métricas: volume/mês, ROI realista, payback realista, nº de assistentes)
2. `<div class="divider">`
3. **Situação Atual** — métricas do cliente (volume, equipe, % tempo em burocracia, ERP)
4. `<div class="divider">`
5. **Diagnóstico / Desafios identificados** — grid de 6 cards `.ci` com as dores da reunião
6. `<div class="divider">`
7. **Solução Proposta** — plan-card com features dos assistentes (adaptar ao contexto)
8. **ROI** — roi-section dark com 3 colunas: conservador, realista, projetado
9. **Impacto Operacional** — prod-grid com 4 cards (24/7, 5×, 80%, ERP)
10. `<div class="divider">`
11. **Investimento** — pricing card `.pc` com classe do tipo de plano, lista de itens, total, validity-box com validade
12. `<div class="divider">`
13. **Próximos Passos** — timeline com 5 etapas (adaptar se ERP pendente → incluir como etapa 5)
14. **CTA** — cta-section roxo com h2, parágrafo e botão linkando para WhatsApp da Tierre
15. **Footer** — logo SVG menor + tagline

**Regras CSS críticas:**
- `body { background: #121325 }` — fundo escuro em todo o documento
- Não alterar nenhuma classe CSS — apenas o conteúdo textual
- O HTML deve começar com `<!-- wp:html -->` e terminar com `<!-- /wp:html -->`

### 5. Publicar no WordPress

Usar `COMPOSIO_REMOTE_WORKBENCH` para fazer a chamada via Python/requests:

```python
payload = {
    "title": f"Proposta Comercial — {nome_cliente}",
    "content": html_completo,
    "status": "publish",
    "slug": f"proposta-{slug_cliente}",
    "template": "elementor_canvas",
    "meta": {
        "_elementor_edit_mode": "",
        "_elementor_template_type": "",
        "_elementor_data": "",
        "_elementor_page_settings": None
    }
}
# POST para /wp-json/wp/v2/pages
```

Se já existir uma página com o mesmo slug (retorno 400/duplicate), deletar a anterior primeiro com `DELETE /wp-json/wp/v2/pages/{id}?force=true`.

### 6. Confirmar e entregar

- Retornar a URL pública da página publicada
- Formato: `https://proposta.dgenny.com.br/proposta-{slug}/`

## Padrão de qualidade

Antes de publicar, verificar mentalmente:
- [ ] SVGs do logo estão presentes (não vazios)
- [ ] Template é `elementor_canvas`
- [ ] Meta `_elementor_edit_mode` está vazio
- [ ] Botão CTA aponta para WhatsApp da Tierre
- [ ] ROI calculado com os dados reais do cliente
- [ ] Nenhum nome de outro cliente aparece no conteúdo
- [ ] Validade da proposta está correta
- [ ] ERP do cliente mencionado corretamente (não misturar CIGAM, Sienge, Koper, etc.)
- [ ] Slug é único e identifica o cliente (`proposta-{cliente}`)

## Observações

- O conteúdo de cada seção deve ser **personalizado** com os dados reais da reunião — não usar textos genéricos
- As dores devem ser específicas ao cliente, não copiar as da Lagotela ou Moinho
- O número de assistentes, tipo de plano e valor podem variar — adaptar o pricing card accordingly
- Se o cliente tiver ERP pendente de integração, sempre tratar como etapa 5 do timeline e incluir nota no pricing card
