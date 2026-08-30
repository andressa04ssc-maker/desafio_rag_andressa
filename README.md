# Mini-RAG sobre documentação HTTPX

## Identificação

- Nome do aluno: Andressa S Santana
- Formato da solução: Colab
- Link do vídeo: [PREENCHER]
- Link do Colab, se aplicável: https://colab.research.google.com/drive/1ZELDtllyeuP-JsqjIcmA7U0eZyge34Vn?usp=sharing

## Objetivo

O sistema constrói o núcleo de recuperação de um RAG (Retrieval-Augmented Generation) sobre a documentação do projeto HTTPX. Ele recebe uma pergunta em linguagem natural (em português ou inglês) e retorna os trechos mais relevantes da documentação, com a fonte exata de onde cada trecho veio. A geração de resposta em linguagem natural (etapa opcional do RAG completo) não foi implementada — a entrega cobre o núcleo de recuperação.

## Arquitetura resumida

```text
repositório HTTPX → arquivos Markdown → chunks + metadados → embeddings → busca por similaridade → resultados com fontes
```

Este projeto cobre até a etapa de busca por similaridade. A geração de resposta em linguagem natural (etapa opcional do fluxo completo de um RAG) não foi implementada — veja detalhes em "Objetivo".

## Como executar do zero

1. **Python**: usa-se o ambiente padrão do Google Colab (Python 3, já vem instalado — não é necessário instalar Python manualmente).
2. **Dependências**: rodar a primeira célula do notebook, que instala as bibliotecas:
   ```python
   !pip install -q sentence-transformers scikit-learn
   ```
3. **Obter a base HTTPX**: rodar a célula seguinte, que clona o repositório e fixa a versão exata usada na prova:
   ```python
   !git clone https://github.com/encode/httpx.git
   %cd httpx
   !git checkout b5addb64f0161ff6bfe94c124ef76f6a1fba5254
   ```
4. **Executar todas as células em ordem**: no menu do Colab, `Ambiente de execução` → `Executar tudo`. Isso roda, em sequência: instalação → clonagem → leitura e chunking dos arquivos `.md` → geração dos embeddings → definição da função de busca.
5. **Fazer uma pergunta**: em uma célula ao final do notebook, chamar a função de busca passando o texto da pergunta entre aspas:
   ```python
   resultados = buscar("sua pergunta aqui", top_k=3)
   exibir_resultados(resultados)
   ```

## Decisões técnicas

### Chunking

- **Estratégia**: divisão em duas etapas. Primeiro o texto de cada arquivo `.md` é separado por seções, usando os cabeçalhos Markdown (`#`, `##`, `###`) como pontos de corte, para não separar um título da sua explicação. Depois, se uma seção ainda estiver grande, ela é subdividida em blocos menores.
- **Tamanho aproximado**: blocos de ~80 palavras.
- **Overlap**: 15 palavras de sobreposição entre blocos consecutivos, para não perder contexto na fronteira entre um chunk e o próximo.
- **Justificativa**: o modelo de embeddings escolhido aceita no máximo 128 tokens de entrada. Blocos de ~80 palavras ficam com folga dentro desse limite (já que trechos com código consomem mais tokens por palavra), e o corte por seção evita misturar assuntos diferentes dentro do mesmo chunk.

### Embeddings e busca

- **Modelo**: `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2`, público, multilíngue e executado localmente (sem API key, sem custo).
- **Forma de cálculo da similaridade**: os embeddings são normalizados no momento da geração (`normalize_embeddings=True`), o que permite calcular a similaridade de cosseno como um simples produto escalar entre a matriz de embeddings dos chunks e o embedding da pergunta.
- **Valor de `top_k`**: 3 (dentro da faixa de 3 a 5 pedida no enunciado).
- **Justificativa**: o modelo multilíngue foi escolhido porque as perguntas de teste são em português e a documentação está em inglês — um modelo monolíngue em inglês não capturaria bem essa correspondência. O produto escalar sobre vetores normalizados foi escolhido por ser rápido (calcula a similaridade com todos os chunks de uma vez, sem loop) e por ser matematicamente equivalente à similaridade de cosseno.

### Metadados e fontes

Cada chunk é armazenado junto com um dicionário de metadados contendo: caminho do arquivo de origem (`arquivo`), título da seção (`secao`) e um identificador sequencial (`chunk_id`). As listas de chunks, embeddings e metadados são construídas na mesma iteração do laço de repetição, garantindo que o índice `i` se refira sempre ao mesmo chunk nas três estruturas. Na hora de exibir um resultado, o índice do chunk mais similar é usado para buscar diretamente o arquivo e a seção correspondentes em `metadados`.

## Perguntas de teste

### 1. Pergunta com resposta clara

- Pergunta: "como configurar timeout nas requisições"
- Resultado esperado: trechos do arquivo sobre configuração de timeout no HTTPX.
- O resultado foi relevante? Sim. Os três resultados vieram de `docs/advanced/timeouts.md`, com scores altos (0.708, 0.676 e 0.669), cobrindo timeout por requisição individual, por instância de cliente e timeout padrão.

### 2. Pergunta ampla ou ambígua

- Pergunta: "como o httpx funciona"
- Resultado esperado: trechos de diferentes partes da documentação, sem um único ponto de resposta.
- O resultado foi relevante? Sim, de forma parcial e esperada. Os três resultados vieram de arquivos diferentes (`docs/advanced/proxies.md`, `docs/third_party_packages.md` e `docs/compatibility.md`), cada um cobrindo um aspecto distinto do HTTPX (proxies, pacotes de terceiros e a camada de rede usada internamente). Os scores (0.674, 0.647 e 0.631) ficaram altos e próximos entre si — mais altos do que o esperado inicialmente para uma pergunta ampla — o que mostra que a frase "como o httpx funciona" tem afinidade temática com vários pontos diferentes da documentação, mesmo sem apontar para uma única seção específica.

### 3. Pergunta fora do escopo

- Pergunta: "qual a receita de um bolo de chocolate"
- Como o sistema reagiu: retornou 3 chunks mesmo sem nenhum ser realmente relevante — o núcleo de recuperação não tem mecanismo para recusar ou dizer "não sei". O score do melhor resultado (0.419) foi bem mais baixo que o das perguntas dentro do escopo (~0.7), e puxado por uma coincidência lexical: um exemplo de código sobre cookies HTTP que usa o valor `chocolate=chip`.
- Como essa reação poderia melhorar: definindo um limiar mínimo de score (por exemplo, recusar resultados abaixo de 0.5) ou adicionando a etapa opcional de geração, instruindo o modelo gerador a admitir quando o contexto recuperado não contém uma resposta real à pergunta.

## Limitações conhecidas

- O sistema sempre retorna `top_k` resultados, mesmo quando nenhum chunk é realmente relevante para a pergunta — não há um mecanismo de rejeição por score baixo.
- A busca por embeddings pode se confundir com sobreposição lexical superficial (mesma palavra usada em contextos de significado completamente diferente), como observado no teste da pergunta fora do escopo.
- Blocos que começam logo após um separador Markdown (`---`) ou a abertura de um bloco de código (` ``` `) podem aparecer cortados de forma pouco legível no início do trecho exibido.
- Não há etapa de geração de linguagem natural: a saída é sempre o trecho bruto da documentação, não uma resposta redigida.

## Uso de ferramentas de IA

- Ferramentas utilizadas: Claude (Anthropic).
- Tarefas em que ajudaram: planejamento do fluxo do projeto, explicação conceitual de cada etapa (chunking, embeddings, similaridade de cosseno), geração e revisão do código Python, diagnóstico de um erro de execução (`NameError: name 'buscar' is not defined`, causado por uma célula não executada em sequência), e apoio na organização deste README.
- Exemplo representativo de prompt ou orientação: "me guie passo a passo na construção do RAG, considerando que meu conhecimento em Python é básico e não sei usar o Google Colab".
- O que foi testado, modificado ou validado por você: execução de cada célula no Colab, conferência da contagem de arquivos `.md` (23) e do formato da matriz de embeddings (333, 384), validação manual de que os resultados retornados pelas três perguntas de teste faziam sentido com o conteúdo da documentação.

## Referências e código externo

- Repositório da base documental: <https://github.com/encode/httpx>
- Guia oficial de busca semântica do Sentence Transformers: <https://www.sbert.net/examples/sentence_transformer/applications/semantic-search/README.html>
- Modelo de embeddings utilizado: <https://huggingface.co/sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2>
- Documentação de similaridade cosseno do scikit-learn: <https://scikit-learn.org/stable/modules/generated/sklearn.metrics.pairwise.cosine_similarity.html>

## Segurança

- [x] Minha solução não usa API key.
