# Criador de Carrossel

Ferramenta local (Svelte + Vite) para montar carrosséis de Instagram com tipografia e tokens de design padronizados. Nesta v1 você define identidade, escreve o conteúdo de cada um dos 10 slides (com Markdown simples) e exporta tudo como PNG pronto para publicar.

## Rodando localmente

```bash
npm install
npm run dev
```

- A aplicação roda em `http://localhost:5173`.
- Para testar a exportação de PNGs é preciso abrir no navegador (não funciona via SSR).

## Workflow sugerido

1. **Identidade** – configure handle (por padrão `@datamundo.cientista`), nome (opcional), cores (preto/branco/cinza), pilha de fontes, texto do CTA e ajuste as escalas de título (até 0,5x), corpo e elementos auxiliares pelos sliders.  
2. **Conteúdo** – cada um dos 10 slides possui um campo de texto. A primeira linha vira o título (hero) e o restante é renderizado como corpo com Markdown simples (`**negrito**`, `*itálico*`, listas com `-`). Há um toggle por slide para ocultar o título quando quiser “remover o hook”.  
3. **Preview** – a coluna da direita mostra os 10 frames 4:5 em layout minimalista (header com handle e contador no canto direito). Clique em “Ver preview” ou no próprio frame para focar.  
4. **Exportação** – clique em _Exportar selecionado_ para baixar apenas o slide ativo ou _Exportar tudo_ para gerar uma sequência `carrossel-01.png`, `carrossel-02.png`, etc. Cada arquivo é rasterizado no navegador com `html2canvas` em ~1080x1350 px (proporção 4:5).

## Próximos passos possíveis

- Adicionar presets de templates (ex.: lista numerada, comparação lado a lado, matriz).  
- Persistir estados localmente (LocalStorage) para continuar edições.  
- Suporte a múltiplos formatos de exportação (SVG/PDF) ou integração headless via Playwright para gerar carrosséis em lote.
