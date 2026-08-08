# 📝 Resposta do Laboratório: A Wiki Perdida dos Arquivos Corporativos

## 👤 Identificação

**Nome:** Luis Felipe Ramalho Carvalho  
**Data:** 08/08/2026  
**Link do repositório:** https://github.com/LuisFelipeRC1/desafio-agente-virtual-dio

---

# ✅ Quest 1: O Mapa dos Arquivos Perdidos

## 1.1 Formatos encontrados na pasta `raw/`

Foram encontrados três arquivos e cada um exige uma estratégia própria:

- **`ata_reuniao_vendas_sa.pdf`**: PDF digital de 5 páginas e com camada de texto. Como o conteúdo já nasce pesquisável, não faz sentido aplicar OCR. O texto pode ser extraído diretamente, preservando página e ordem dos parágrafos.
- **`ata_resultados_vendas_novos_dados.png`**: imagem digitalizada de uma página. Como o arquivo é formado por pixels e possui anotações manuscritas, precisa de OCR. O serviço escolhido é o Amazon Textract, que reconhece texto digitado e manuscrito e também fornece posição e confiança da detecção.
- **`vendas_sa_dados_ficticios_laboratorio.csv`**: exportação estruturada do CRM com 240 oportunidades e 19 colunas. O cabeçalho contém campos como `oportunidade_id`, datas de criação/fechamento, cliente, segmento, região, vendedor, origem do lead, produto, campanha, status, probabilidade, valores, desconto, ciclo, motivo de perda, próxima atividade e observação. Ele deve ser tratado como tabela, e não como um documento de texto corrido.

A primeira decisão da arquitetura é, portanto, **não aplicar o mesmo pipeline aos três arquivos**.

---

## 1.2 Principais desafios encontrados

- Todos os arquivos chegam diretamente em `raw/`, sem subpastas que indiquem o tipo ou a área de negócio.
- A classificação não pode depender apenas do nome do arquivo, porque nomes podem mudar ou estar errados.
- A imagem pode conter texto manuscrito, baixa resolução, ruído ou regiões com confiança de OCR reduzida.
- O PDF é digital e passar por OCR seria desperdício de custo, tempo e qualidade.
- Atas possuem informação semiestruturada: uma mesma decisão pode estar distribuída entre contexto, responsável e prazo.
- O CSV possui tipos diferentes de dados e campos vazios; perder o schema transformando tudo em texto prejudicaria filtros e análises.
- O conteúdo do CSV não deve ser dividido em chunks que misturem oportunidades diferentes.
- Metadados extraídos por IA podem conter erro e precisam manter rastreabilidade para o trecho original.
- Informações comerciais podem exigir restrição por perfil e nível de confidencialidade.
- Reprocessamentos não devem duplicar registros ou embeddings.

---

## 1.3 Informações importantes a serem extraídas

### Atas e imagem digitalizada

- data da reunião;
- título e assunto;
- participantes;
- projetos, clientes ou produtos mencionados;
- temas discutidos;
- decisões tomadas;
- responsáveis por cada ação;
- prazos;
- pendências;
- riscos e impedimentos;
- próximos passos;
- página ou região da imagem de onde a informação foi extraída;
- confiança do OCR, quando aplicável.

### CSV do CRM

Além dos metadados gerais do arquivo, cada oportunidade deve preservar:

- `oportunidade_id`;
- `data_criacao` e `data_fechamento`;
- `cliente_ficticio`;
- `segmento` e `regiao`;
- `vendedor_ficticio`;
- `origem_lead`;
- `produto` e `campanha`;
- `status` e `probabilidade_pct`;
- `valor_bruto_brl`, `desconto_pct` e `valor_liquido_brl`;
- `ciclo_dias`;
- `motivo_perda`;
- `proxima_atividade`;
- `observacao`.

Esses campos permitem responder perguntas semânticas e também aplicar filtros por status, região, vendedor, período ou produto.

---

## 1.4 Estratégia de classificação inicial

A classificação inicial será **técnica e determinística**, antes de qualquer IA:

1. o upload no Amazon S3 gera um evento;
2. uma AWS Lambda lê os primeiros bytes do objeto, extensão, MIME type e estrutura básica;
3. se for CSV, a função valida o cabeçalho e o schema esperado;
4. se for imagem, define `needs_ocr=true`;
5. se for PDF, verifica se existe camada textual; se houver, `needs_ocr=false`; caso contrário, encaminha para OCR;
6. a Lambda grava tags/metadados como `source_kind`, `mime_type`, `needs_ocr`, `document_id` e `processing_status`;
7. depois da extração, o Amazon Bedrock pode acrescentar uma classificação semântica, como `ata`, `relatorio_comercial`, `crm`, tema e confidencialidade.

Assim, a solução não depende de subpastas nem do nome do arquivo para decidir o pipeline.

---

# ✅ Quest 2: O Portal de Entrada na AWS

## 2.1 Armazenamento dos arquivos brutos

Os arquivos são enviados para um bucket Amazon S3 com uma zona lógica `raw/`. O bucket terá:

- **S3 Block Public Access** habilitado;
- criptografia server-side com chave do **AWS KMS**;
- **S3 Versioning** para preservar versões;
- política IAM de menor privilégio;
- tags para tipo, origem, classificação e status;
- Lifecycle Policy para mover originais antigos para uma classe mais barata quando apropriado.

Apesar de a chave lógica poder conter `raw/`, a classificação funcional não depende dessa organização. O objeto continua sendo identificado por metadados e pelo `document_id`.

---

## 2.2 Preservação dos arquivos originais

O arquivo original nunca é sobrescrito. Para cada entrada:

1. o S3 preserva o objeto recebido e suas versões;
2. é calculado um checksum SHA-256 para detectar alterações e evitar ingestão duplicada;
3. o `document_id`, checksum, URI do S3, versão e data de ingestão são registrados no Amazon DynamoDB;
4. derivados — texto, JSON do Textract, metadados e versões normalizadas — são gravados em uma zona separada `processed/`;
5. em cenários regulatórios, o S3 Object Lock pode ser habilitado para retenção WORM.

Toda resposta da Wiki mantém um caminho de volta para o objeto original e para a versão que originou o chunk indexado.

---

## 2.3 Extração de texto dos documentos

O Amazon S3 inicia o fluxo, e uma **AWS Step Functions** coordena as rotas de processamento.

### Rota A — PDF digital

`ata_reuniao_vendas_sa.pdf` já possui camada textual. Uma Lambda executa a extração dessa camada diretamente, sem OCR, preservando marcadores de página. O resultado é normalizado em UTF-8 e salvo em S3 como Markdown/JSON estruturado.

**Por que não usar Textract aqui?** Porque OCR seria desnecessário e poderia introduzir erros em um conteúdo que já possui texto digital.

### Rota B — imagem digitalizada

`ata_resultados_vendas_novos_dados.png` segue para **Amazon Textract**. Como é uma página, pode ser processada de forma síncrona quando estiver dentro dos limites do serviço. O retorno do Textract é preservado em JSON e inclui:

- linhas e palavras;
- coordenadas do texto;
- tipo de texto quando identificado;
- confiança da detecção.

Depois, uma Lambda reconstrói a ordem de leitura e gera um texto normalizado. Trechos abaixo de um limite de confiança, por exemplo 85%, recebem `review_required=true`.

### Rota C — CSV do CRM

O CSV é validado como tabela. Para o volume atual, uma Lambda é suficiente; para volumes maiores, a transformação pode migrar para **AWS Glue**.

O processamento:

1. valida as 19 colunas e tipos esperados;
2. normaliza datas, números e campos vazios;
3. registra o schema no **AWS Glue Data Catalog**;
4. mantém uma representação estruturada para filtros e análises;
5. cria uma projeção textual por oportunidade, por exemplo:

```text
Oportunidade OPP-20260002 — Nexo Tecnologia 002, segmento Logística, região Sudeste.
Status: Proposta enviada. Probabilidade: 55%. Produto: Analytics de Receita.
Valor líquido: R$ 103.400,00. Próxima atividade: 24/08/2026.
Observação: Proposta em revisão jurídica.
```

Cada oportunidade vira uma unidade lógica separada, evitando mistura de registros durante a indexação.

### Saída comum

Todos os caminhos convergem para S3 `processed/`, com:

- texto limpo;
- documento estruturado em JSON;
- metadados;
- referência ao original;
- versão do schema de processamento.

---

## 2.4 Tratamento de falhas

A Step Functions implementa estados de `Retry` e `Catch`. O processamento é idempotente usando `document_id + checksum`.

Em caso de erro:

- o status no DynamoDB muda para `FAILED`;
- detalhes técnicos vão para Amazon CloudWatch Logs;
- uma métrica customizada contabiliza falhas por tipo;
- alarmes do CloudWatch avisam quando a taxa de erro ultrapassa o limite;
- eventos não recuperáveis podem ser enviados para uma fila Amazon SQS de dead letter;
- o objeto original permanece intacto no S3;
- o documento pode ser reprocessado sem gerar duplicidade.

Também são armazenados `error_code`, `stage`, `attempt_count` e `last_attempt_at` para facilitar diagnóstico.

---

# ✅ Quest 3: A Relíquia dos Metadados

## 3.1 Padronização dos textos processados

A normalização ocorre antes da indexação:

- UTF-8 como codificação padrão;
- remoção de caracteres de controle e espaços duplicados;
- normalização de quebras de linha;
- preservação de títulos, listas, tabelas e marcadores de página;
- correção apenas de ruídos claramente mecânicos, sem reescrever o conteúdo original;
- deduplicação por checksum e por `document_id`;
- datas convertidas para ISO 8601;
- números mantidos também em formato numérico no JSON;
- campos ausentes representados como `null`, nunca inventados;
- cada trecho recebe ponteiro para arquivo, versão, página/linha ou `oportunidade_id`.

O S3 mantém dois artefatos: uma versão humana em Markdown e uma versão estruturada em JSON.

---

## 3.2 Metadados propostos

| Metadado | Por que ele é importante? |
|---|---|
| `document_id` | Identificador estável entre ingestão, metadados, chunks e respostas |
| Nome do documento | Facilita identificação humana da fonte |
| Tipo do documento | Permite roteamento e filtros por ata, imagem ou CRM |
| Data identificada | Habilita buscas temporais |
| Tema principal | Melhora filtros e recuperação semântica |
| Participantes | Permite perguntas por pessoas envolvidas |
| Decisões tomadas | Destaca conhecimento acionável das atas |
| Responsáveis | Permite localizar donos de ações |
| Próximos passos | Ajuda a identificar pendências |
| Riscos | Permite consultas sobre impedimentos e alertas |
| Projeto/cliente/produto | Dá contexto de negócio |
| Nível de confidencialidade | Apoia autorização e filtragem |
| Caminho do arquivo original | Mantém rastreabilidade |
| Versão S3 | Garante que a citação aponte para a versão correta |
| Checksum | Detecta alteração e duplicidade |
| Página/linha/registro | Permite localizar a evidência dentro da fonte |
| `ocr_confidence` | Informa a confiabilidade da extração de imagem |
| `processing_status` | Permite acompanhar o pipeline |
| `schema_version` | Facilita evolução do formato processado |

Para o CSV, os campos de negócio da oportunidade também são metadados estruturados.

---

## 3.3 Uso de IA para enriquecimento dos documentos

Depois da extração determinística, o **Amazon Bedrock** é utilizado para enriquecer o conteúdo.

Um prompt instrui o modelo a retornar um JSON seguindo schema fixo, contendo, quando houver evidência:

- tema;
- resumo;
- participantes;
- decisões;
- responsáveis;
- prazos;
- riscos;
- pendências;
- projetos/clientes/produtos;
- confidencialidade sugerida;
- referências aos trechos que sustentam cada campo.

A temperatura é mantida baixa e o prompt determina que valores ausentes sejam `null`. Uma Lambda valida o JSON antes de salvá-lo. Informações inferidas sem evidência suficiente não são aceitas como fato.

Para a imagem, o texto enviado ao modelo inclui também a confiança do Textract. Se a informação crítica vier apenas de OCR com baixa confiança, o metadado recebe `review_required=true`.

---

## 3.4 Armazenamento dos metadados

A proposta usa três níveis:

1. **Amazon S3** — mantém os arquivos processados e arquivos sidecar de metadados associados à fonte;
2. **Amazon DynamoDB** — guarda metadados ricos e operacionais, status, checksum, versão, erros e campos de negócio;
3. **AWS Glue Data Catalog** — mantém o schema e a descoberta do conjunto tabular do CRM.

Para o Bedrock Knowledge Bases, apenas metadados úteis para recuperação e filtragem são enviados ao índice, como `document_id`, tipo, data, projeto/cliente, região, status, confidencialidade e identificador da origem.

Como o S3 Vectors possui limites de metadados por vetor, o índice recebe apenas o conjunto enxuto necessário à busca. Os metadados completos permanecem no DynamoDB/S3.

---

# ✅ Quest 4: O Oráculo da Wiki Inteligente

## 4.1 Estratégia de indexação

O chunking é adaptado ao tipo de fonte.

### Atas

A divisão prioriza limites semânticos: título, seção, pauta, decisão e bloco de ações. Como fallback, podem ser usados chunks de aproximadamente 400 a 800 tokens, com pequena sobreposição para não cortar contexto.

Cada chunk preserva:

- `document_id`;
- página;
- seção;
- data;
- tema;
- confidencialidade;
- URI/versionamento da fonte.

### Imagem

Como é uma única página, o conteúdo pode permanecer em um ou poucos chunks. Se houver blocos visuais independentes, as coordenadas retornadas pelo Textract ajudam a separar regiões sem perder contexto.

### CSV

Uma oportunidade é uma unidade semântica. Um chunk nunca mistura duas oportunidades. Os campos estruturados ficam em metadados e a projeção textual serve para embedding.

Essa estratégia reduz falsos relacionamentos e melhora a precisão da recuperação.

---

## 4.2 Busca semântica e base vetorial

A solução usa **Amazon Bedrock Knowledge Bases** com um modelo de embeddings do Amazon Bedrock e **Amazon S3 Vectors** como vector store.

Fluxo:

1. os documentos processados em S3 são sincronizados com a Knowledge Base;
2. o Bedrock divide o conteúdo conforme a estratégia definida;
3. cada chunk é convertido em embedding;
4. os vetores e metadados são armazenados no S3 Vectors;
5. uma pergunta do usuário é transformada em embedding;
6. a busca retorna os chunks mais similares;
7. filtros de metadados podem limitar por data, tipo, região, projeto, status ou confidencialidade.

O S3 Vectors foi escolhido por ser serverless e integrado ao fluxo de RAG do Bedrock Knowledge Bases. Se no futuro houver necessidade de recursos avançados de busca híbrida ou escala/latência específicas, Amazon OpenSearch Serverless pode ser avaliado.

---

## 4.3 Geração de respostas com IA

A consulta segue o padrão RAG — Retrieval-Augmented Generation.

1. o usuário envia a pergunta pela interface;
2. Amazon API Gateway encaminha a requisição para uma Lambda;
3. a Lambda identifica o usuário e aplica os filtros permitidos;
4. `RetrieveAndGenerate` do Bedrock Knowledge Bases recupera os chunks relevantes;
5. somente esse contexto é enviado ao modelo de geração no Amazon Bedrock;
6. o prompt instrui o modelo a responder apenas com base nas evidências recuperadas;
7. a resposta inclui citações/referências aos chunks de origem;
8. a aplicação converte essas referências em nome do documento, página ou oportunidade do CRM.

Se não existir evidência suficiente, a resposta deve declarar algo como:

> Não encontrei informação suficiente na base autorizada para responder com segurança.

Isso é preferível a completar a resposta com conhecimento externo ou inferência não sustentada.

---

## 4.4 Interface de consulta

A interface proposta é uma aplicação web simples:

- **AWS Amplify** para hospedagem do frontend;
- **Amazon Cognito** para login e grupos de usuários;
- **Amazon API Gateway** para expor a API;
- **AWS Lambda** para autenticação contextual, filtros, chamadas ao Bedrock e montagem da resposta;
- **Amazon Bedrock Knowledge Bases** para recuperação;
- **Amazon Bedrock** para geração.

A tela principal possui:

- campo de pergunta em linguagem natural;
- resposta resumida;
- lista de fontes utilizadas;
- botão para abrir o documento original autorizado;
- filtros por período, tipo, projeto/cliente, região e status;
- indicação de baixa confiança quando a resposta depende de OCR duvidoso.

---

## 4.5 Segurança, auditoria e monitoramento

### Segurança

- S3 Block Public Access;
- criptografia com AWS KMS no S3, DynamoDB e recursos compatíveis;
- IAM com menor privilégio;
- Cognito Groups para perfis como `comercial`, `gestao` e `administrador`;
- nível de confidencialidade utilizado como filtro obrigatório antes da recuperação;
- URLs temporárias/pre-assinadas somente para documentos que o usuário pode acessar;
- Amazon Bedrock Guardrails pode ser usado para políticas de conteúdo e proteção adicional na camada generativa.

### Auditoria

- AWS CloudTrail registra chamadas e alterações relevantes na conta;
- consultas podem gerar um registro de auditoria com `user_id`, horário, filtros, documentos recuperados e identificador da resposta;
- nenhuma informação sensível precisa ser gravada integralmente nos logs.

### Monitoramento

Amazon CloudWatch acompanha:

- duração do pipeline;
- erros por etapa;
- documentos pendentes;
- chamadas ao Textract e Bedrock;
- latência da busca;
- taxa de respostas sem evidência;
- OCR abaixo do limite de confiança.

Amazon Macie pode ajudar a identificar dados sensíveis armazenados no S3. AWS Cost Explorer e Budgets podem acompanhar e alertar sobre custo.

---

# 🧩 Arquitetura Final da Solução

## 1. Visão geral

A arquitetura separa o problema em cinco camadas: **ingestão, extração, normalização/enriquecimento, indexação e consulta**. Cada formato recebe o tratamento correto antes de convergir para um conjunto padronizado no S3. A partir daí, Bedrock Knowledge Bases e S3 Vectors implementam o RAG, enquanto Cognito, IAM, KMS, CloudTrail e CloudWatch fornecem controles corporativos.

---

## 2. Serviços AWS utilizados

| Serviço AWS | Papel na solução |
|---|---|
| Amazon S3 | Originais, processados, sidecars de metadados e fonte da Knowledge Base |
| Amazon Textract | OCR da imagem e reconhecimento de texto manuscrito |
| Amazon Bedrock | Enriquecimento e geração de respostas |
| Amazon Bedrock Knowledge Bases | Ingestão, chunking, embeddings, recuperação e RAG com citações |
| Amazon S3 Vectors | Armazenamento e consulta dos vetores |
| AWS Lambda | Classificação, extração digital, normalização, validação e API |
| AWS Step Functions | Orquestração, retries e controle do pipeline |
| Amazon DynamoDB | Estado de processamento e metadados ricos |
| AWS Glue Data Catalog | Catálogo do schema do CSV |
| Amazon Cognito | Autenticação e grupos de acesso |
| Amazon API Gateway | API segura para a Wiki |
| AWS Amplify | Hospedagem da interface web |
| Amazon CloudWatch | Logs, métricas, alarmes e dashboards |
| AWS CloudTrail | Auditoria de ações na AWS |
| AWS IAM | Autorização de serviços pelo menor privilégio |
| AWS KMS | Criptografia e gestão de chaves |
| Amazon Macie | Descoberta de dados sensíveis no S3 |
| Amazon SQS | Dead-letter queue para falhas não recuperáveis |

---

## 3. Fluxo de dados de ponta a ponta

1. Os arquivos entram no Amazon S3 e os originais são preservados.
2. Um evento aciona a Lambda de classificação técnica.
3. A Step Functions escolhe a rota de processamento.
4. O PDF digital tem sua camada de texto extraída sem OCR.
5. A imagem passa pelo Amazon Textract.
6. O CSV é validado como tabela, catalogado e convertido em unidades por oportunidade.
7. Os resultados são normalizados e gravados em S3 `processed/`.
8. Amazon Bedrock enriquece os documentos com metadados estruturados.
9. DynamoDB recebe o estado e os metadados ricos; Glue Data Catalog mantém o schema tabular.
10. Bedrock Knowledge Bases sincroniza os dados processados.
11. Um modelo de embeddings gera os vetores e o S3 Vectors os armazena.
12. O usuário autentica pelo Cognito e pergunta pela interface no Amplify.
13. API Gateway + Lambda aplicam filtros de acesso e consultam a Knowledge Base.
14. Os trechos mais relevantes são recuperados.
15. Bedrock produz uma resposta fundamentada.
16. A interface exibe resposta e fontes.
17. CloudWatch e CloudTrail registram saúde operacional e auditoria.

---

## 4. Diagrama textual da arquitetura

```text
raw/
  ↓
Amazon S3 (originais, versionamento, KMS)
  ↓ ObjectCreated
AWS Lambda (classificação)
  ↓
AWS Step Functions
  ├─ PDF digital ─→ Lambda: extrair camada textual ─┐
  ├─ PNG scan ────→ Amazon Textract ───────────────┤
  └─ CSV CRM ─────→ Lambda / Glue ─────────────────┤
                                                   ↓
                                          S3 processed/
                                                   ↓
                                Amazon Bedrock (enriquecimento)
                                     ↓                 ↓
                                  DynamoDB       metadados S3
                                                   ↓
                                  Bedrock Knowledge Bases
                                                   ↓
                                      Amazon S3 Vectors
                                                   ↑
Usuário → Cognito → Amplify → API Gateway → Lambda ─┘
                                      ↓
                             Amazon Bedrock (RAG)
                                      ↓
                         Resposta + fontes/citações

Observabilidade: CloudWatch + CloudTrail
Segurança: IAM + KMS + Cognito + Macie
```

---

## 5. Riscos e limitações

- Texto manuscrito pode ter OCR com baixa confiança.
- Um PDF futuro pode parecer digital, mas conter páginas escaneadas; a classificação deve ser feita por conteúdo e não só extensão.
- Metadados gerados por IA podem apresentar erro e precisam de validação.
- RAG não garante respostas corretas se a recuperação selecionar contexto inadequado.
- Consultas numéricas exatas sobre o CRM, como somas e contagens complexas, são melhor atendidas por consulta estruturada do que por busca vetorial.
- Os metadados enviados ao índice vetorial precisam permanecer compactos.
- Controle de acesso mal aplicado pode causar vazamento de conteúdo na etapa de recuperação.
- Custos de Textract, embeddings e modelos generativos crescem com volume e frequência de reprocessamento.
- Mudanças de versão do documento precisam invalidar ou atualizar chunks antigos.

Mitigações incluem confiança mínima de OCR, validação de schema, filtros de autorização antes da recuperação, versionamento, testes de qualidade de RAG e monitoramento de custo.

---

## 6. Melhorias futuras

- ingestão automática para qualquer novo objeto no S3;
- painel de decisões, responsáveis e pendências;
- revisão humana para OCR de baixa confiança;
- Amazon Athena para perguntas analíticas exatas sobre o CRM;
- Amazon Bedrock Agent para rotear perguntas entre Knowledge Base e consultas estruturadas;
- classificação automática de confidencialidade;
- Bedrock Guardrails e políticas contra prompt injection;
- avaliação automática da qualidade das respostas e conjunto de perguntas de teste;
- feedback do usuário sobre relevância das fontes;
- notificações para prazos e ações em aberto;
- políticas de retenção e descarte por categoria documental;
- cache para consultas frequentes quando fizer sentido.

---

# 🧠 Checklist Final

- [x] Como transformar documentos escaneados em texto?
- [x] Como lidar com diferentes formatos dentro da mesma pasta `raw/`?
- [x] Como armazenar os documentos originais?
- [x] Como preservar a rastreabilidade entre resposta e documento fonte?
- [x] Como organizar metadados?
- [x] Como criar busca semântica?
- [x] Como usar Amazon Bedrock na solução?
- [x] Como proteger documentos sensíveis?
- [x] Como monitorar falhas?
- [x] Como a empresa usaria essa Wiki no dia a dia?

---

# 🏁 Conclusão

A solução proposta evita o erro de tratar todo o acervo como se fosse um único tipo de documento. O PDF digital, a imagem escaneada e o CSV seguem rotas específicas, mas convergem para um modelo padronizado, rastreável e seguro. A combinação de Amazon S3, Textract, Lambda, Step Functions, Bedrock Knowledge Bases e S3 Vectors cria uma base RAG serverless capaz de responder perguntas em linguagem natural e apontar as fontes utilizadas.

Para uso corporativo, o ponto mais importante não é apenas gerar uma boa resposta: é conseguir demonstrar **de qual documento ela veio, qual versão foi usada, quem tinha autorização para consultá-la e qual evidência sustenta a conclusão**. Essa rastreabilidade transforma a proposta de um simples chatbot em uma Wiki Corporativa confiável.
