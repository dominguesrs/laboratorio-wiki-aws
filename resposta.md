# 📝 Resposta do Laboratório: A Wiki Perdida dos Arquivos Corporativos

## 👤 Identificação

**Nome:** Manoel Domingues

**Data:** 29/09/2026

**Link do repositório:**
https://github.com/dominguesrs/laboratorio-wiki-aws

---

# ✅ Quest 1: O Mapa dos Arquivos Perdidos

## 1.1 Formatos encontrados na pasta `raw/`

A pasta `raw/` contém três arquivos com características diferentes:

| Arquivo                                     | Formato | Característica                                                         | Estratégia                        |
| ------------------------------------------- | ------- | ---------------------------------------------------------------------- | --------------------------------- |
| `ata_reuniao_vendas_sa.pdf`                 | PDF     | Documento digital com camada de texto                                  | Extração direta do texto, sem OCR |
| `ata_resultados_vendas_novos_dados.png`     | PNG     | Documento digitalizado, formado por pixels e com anotações manuscritas | Amazon Textract para OCR          |
| `vendas_sa_dados_ficticios_laboratorio.csv` | CSV     | Dados estruturados em tabela, com 240 oportunidades e 19 colunas       | Parsing e normalização dos dados  |

O principal ponto da análise é que os três arquivos **não devem seguir exatamente o mesmo pipeline**.

O PDF já possui texto digital e, portanto, não precisa passar por OCR.

A imagem não possui uma camada de texto e precisa de reconhecimento óptico para transformar os pixels em informação pesquisável.

O CSV representa dados estruturados e deve ser tratado como dataset, preservando suas colunas e relações em vez de transformá-lo simplesmente em texto corrido.

---

## 1.2 Principais desafios encontrados

Os principais desafios são:

* Arquivos de formatos diferentes armazenados no mesmo diretório;
* Ausência de subpastas para classificação;
* Necessidade de preservar os arquivos originais;
* PDF com texto digital que não precisa de OCR;
* Imagem escaneada que precisa de OCR;
* Anotações manuscritas que podem apresentar menor precisão de reconhecimento;
* CSV com grande quantidade de registros e múltiplas colunas;
* Necessidade de transformar dados heterogêneos em um formato comum para consulta;
* Necessidade de manter rastreabilidade entre uma resposta da IA e o documento original;
* Possíveis erros de OCR;
* Possíveis informações incompletas ou ambíguas;
* Necessidade de controlar acesso a informações corporativas.

---

## 1.3 Informações importantes a serem extraídas

Para as atas, eu extrairia:

* Data da reunião;
* Título;
* Participantes;
* Área ou departamento;
* Projetos mencionados;
* Clientes mencionados;
* Temas discutidos;
* Decisões tomadas;
* Responsáveis;
* Prazos;
* Próximos passos;
* Pendências;
* Riscos;
* Observações;
* Número da página de origem.

Para o CSV:

* ID da oportunidade;
* Data de criação;
* Data de fechamento;
* Cliente;
* Segmento;
* Região;
* Vendedor;
* Origem do lead;
* Produto;
* Campanha;
* Status;
* Probabilidade;
* Valor bruto;
* Desconto;
* Valor líquido;
* Ciclo;
* Motivo da perda;
* Próxima atividade;
* Observação.

Além desses campos, todos os documentos receberiam metadados técnicos, como:

* ID único do documento;
* Nome original;
* Tipo de documento;
* Extensão;
* Data de ingestão;
* Caminho no Amazon S3;
* Hash do arquivo;
* Status do processamento;
* Nível de confidencialidade.

---

## 1.4 Estratégia de classificação inicial

Como os arquivos estão todos diretamente dentro de `raw/`, eu não dependeria de subpastas.

A classificação seria feita durante a ingestão utilizando:

1. Extensão do arquivo;
2. MIME type;
3. Metadados do objeto no Amazon S3;
4. Regras de processamento;
5. Validação do conteúdo quando necessário.

Uma função AWS Lambda acionada após o upload no S3 poderia identificar:

```text
.pdf → verificar se possui camada de texto
.png → documento de imagem → OCR
.csv → dataset estruturado
```

Para o caso do PDF, a aplicação pode verificar se existe texto extraível. Caso não exista, o documento poderia ser encaminhado para Textract como exceção.

Dessa forma, a classificação não depende da existência de subpastas.

---

# ✅ Quest 2: O Portal de Entrada na AWS

## 2.1 Armazenamento dos arquivos brutos

O Amazon S3 será o ponto central de armazenamento.

Eu criaria um bucket privado, por exemplo:

```text
s3://wiki-corporativa-documentos/
```

Com uma organização lógica por prefixos:

```text
raw/
processed/
metadata/
failed/
```

O conteúdo original seria colocado em:

```text
raw/
```

Mesmo que os arquivos originalmente estejam misturados, o pipeline pode criar uma organização lógica posteriormente sem modificar os arquivos de origem.

O bucket seria configurado com:

* Bloqueio de acesso público;
* IAM com princípio do menor privilégio;
* Criptografia com AWS KMS;
* Versionamento;
* Lifecycle para controlar custos;
* Logs e auditoria;
* Políticas de bucket restritivas.

---

## 2.2 Preservação dos arquivos originais

Os arquivos da camada `raw/` seriam considerados a **fonte de verdade**.

Eles nunca seriam sobrescritos pelo processamento.

A solução manteria:

```text
raw/
    arquivo-original
```

e criaria resultados separados:

```text
processed/
    documento-processado
```

Também seria armazenado um identificador único e, quando aplicável, um hash do arquivo.

Assim é possível responder:

> "De qual arquivo veio esta informação?"

por meio de:

```text
document_id
    ↓
s3://wiki-corporativa-documentos/raw/arquivo-original
```

O versionamento do S3 permite manter versões caso um objeto seja substituído acidentalmente.

---

## 2.3 Extração de texto dos documentos

### PDF digital

Para:

```text
ata_reuniao_vendas_sa.pdf
```

a prioridade seria extrair diretamente a camada de texto existente.

Não faz sentido executar OCR em um documento que já possui texto digital, pois isso adicionaria uma etapa desnecessária e poderia introduzir erros.

O texto extraído seria enviado para:

```text
s3://wiki-corporativa-documentos/processed/
```

---

### Imagem digitalizada

Para:

```text
ata_resultados_vendas_novos_dados.png
```

seria utilizado o **Amazon Textract**.

O Textract transforma o conteúdo visual em dados textuais e também pode identificar estruturas de documentos.

Como existe a possibilidade de anotações manuscritas, a qualidade do OCR seria monitorada.

Quando a confiança de determinada informação for baixa, o registro poderia receber:

```text
ocr_confidence = low
```

para posterior validação humana.

---

### CSV

Para:

```text
vendas_sa_dados_ficticios_laboratorio.csv
```

não utilizaria Textract.

O arquivo contém dados estruturados e deve ser interpretado como tabela.

Uma AWS Lambda faria:

1. Leitura do CSV;
2. Validação do cabeçalho;
3. Validação do número de colunas;
4. Conversão dos tipos de dados;
5. Identificação de campos obrigatórios;
6. Normalização;
7. Geração de registros estruturados;
8. Armazenamento do resultado no S3.

Uma representação lógica poderia ser:

```json
{
  "oportunidade_id": "OPP-20260001",
  "cliente": "Orion Digital 001",
  "segmento": "Servicos empresariais",
  "regiao": "Sudeste",
  "produto": "CRM Profissional",
  "status": "Qualificacao",
  "probabilidade_pct": 25,
  "valor_liquido_brl": 100045.00
}
```

Além do formato estruturado, os registros poderiam receber uma representação textual controlada para permitir consultas em linguagem natural.

Por exemplo:

```text
A oportunidade OPP-20260001 pertence ao cliente Orion Digital 001,
do segmento Serviços empresariais, região Sudeste.
O produto é CRM Profissional.
O status atual é Qualificação, com probabilidade de 25%.
O valor líquido é R$ 100.045,00.
```

Isso permite combinar busca semântica com os dados estruturados.

---

## 2.4 Tratamento de falhas

O processamento seria orquestrado pelo AWS Step Functions.

Fluxo simplificado:

```text
S3
 ↓
Lambda - Classificação
 ↓
Step Functions
 ├── PDF digital → Extração de texto
 ├── PNG → Textract
 └── CSV → Parser/normalização
 ↓
Validação
 ↓
Processado
```

Em caso de erro:

```text
Processamento
      ↓
    Falha
      ↓
CloudWatch Logs
      ↓
failed/
```

Cada execução teria:

* `document_id`;
* nome do arquivo;
* etapa que falhou;
* timestamp;
* mensagem de erro;
* status.

O CloudWatch seria utilizado para logs, métricas e alarmes.

---

# ✅ Quest 3: A Relíquia dos Metadados

## 3.1 Padronização dos textos processados

Após a extração, todos os documentos seriam convertidos para um modelo comum.

Exemplo:

```json
{
  "document_id": "DOC-000001",
  "source_file": "ata_reuniao_vendas_sa.pdf",
  "document_type": "meeting_minutes",
  "source_uri": "s3://wiki-corporativa-documentos/raw/ata_reuniao_vendas_sa.pdf",
  "content": "...",
  "ingestion_date": "2026-09-29",
  "processing_status": "processed"
}
```

O conteúdo seria normalizado para:

* Remover espaços duplicados;
* Normalizar quebras de linha;
* Corrigir fragmentações provocadas pela extração;
* Preservar títulos;
* Preservar páginas;
* Remover conteúdos duplicados;
* Separar seções importantes;
* Manter a referência ao documento original.

Para OCR, o texto também receberia informações de confiança quando disponíveis.

---

## 3.2 Metadados propostos

| Metadado          | Por que ele é importante?                 |
| ----------------- | ----------------------------------------- |
| `document_id`     | Identificador único e estável             |
| Nome do documento | Identificação humana                      |
| Tipo do documento | Diferencia ata, imagem e dataset          |
| Formato           | Define características técnicas           |
| Data identificada | Permite consultas temporais               |
| Data de ingestão  | Controle operacional                      |
| Tema principal    | Facilita filtros e busca                  |
| Participantes     | Permite consultas por pessoa              |
| Projetos          | Permite localizar informações por projeto |
| Clientes          | Relaciona documentos a clientes           |
| Decisões tomadas  | Permite encontrar decisões                |
| Responsáveis      | Identifica responsáveis por ações         |
| Próximos passos   | Facilita acompanhamento                   |
| Riscos            | Permite localizar riscos                  |
| Pendências        | Facilita gestão de tarefas                |
| Confidencialidade | Controle de acesso                        |
| Fonte original    | Rastreabilidade                           |
| Página/origem     | Localização precisa                       |
| OCR confidence    | Identifica possíveis erros                |
| Hash              | Integridade do documento                  |

---

## 3.3 Uso de IA para enriquecimento dos documentos

O Amazon Bedrock seria utilizado na etapa de enriquecimento sem alterar o documento original.

O modelo receberia o conteúdo processado e uma instrução estruturada para identificar:

```text
- resumo
- temas
- projetos
- decisões
- responsáveis
- prazos
- pendências
- riscos
```

A saída poderia seguir um JSON padronizado.

Exemplo:

```json
{
  "resumo": "...",
  "temas": ["vendas", "expansão"],
  "decisoes": [
    "Aprovar a próxima etapa do projeto"
  ],
  "responsaveis": [
    "Responsável identificado no documento"
  ],
  "pendencias": [
    "Enviar proposta revisada"
  ]
}
```

Uma regra importante seria instruir o modelo a:

> Não inventar informações que não estejam presentes no documento.

Quando uma informação não existir, o campo deverá ser `null` ou uma lista vazia.

---

## 3.4 Armazenamento dos metadados

O conteúdo processado ficaria no Amazon S3.

Para metadados operacionais e consultas estruturadas, utilizaria Amazon DynamoDB.

Exemplo:

```text
DynamoDB
    document_id
        ↓
    source_uri
        ↓
    metadata
        ↓
    processing_status
```

O S3 continuaria sendo a fonte dos documentos, enquanto o DynamoDB facilitaria consultas rápidas por atributos.

O relacionamento seria:

```text
DynamoDB
    ↓
document_id
    ↓
S3 original
```

O Amazon Bedrock Knowledge Bases seria responsável pela camada de conhecimento e recuperação semântica.

---

# ✅ Quest 4: O Oráculo da Wiki Inteligente

## 4.1 Estratégia de indexação

Documentos longos não devem ser enviados como um único bloco para a busca semântica.

O conteúdo seria dividido em chunks.

Exemplo:

```text
Documento
   ↓
Seção
   ↓
Chunk
   ↓
Embedding
```

Cada chunk manteria seus metadados:

```json
{
  "document_id": "DOC-000001",
  "source_file": "ata_reuniao_vendas_sa.pdf",
  "page": 3,
  "document_type": "meeting_minutes",
  "content": "..."
}
```

O tamanho dos chunks seria definido de acordo com o tipo de documento.

Em atas, eu procuraria preservar a unidade semântica das seções para evitar separar uma decisão do contexto que a explica.

No CSV, cada oportunidade ou conjunto lógico de registros poderia ser tratado como unidade de conhecimento.

---

## 4.2 Busca semântica e base vetorial

A solução utilizaria **Amazon Bedrock Knowledge Bases**.

A Knowledge Base poderia utilizar:

* Amazon S3 como fonte dos documentos processados;
* Modelo de embeddings disponível no Amazon Bedrock;
* Um armazenamento vetorial compatível, como Amazon OpenSearch Serverless.

Fluxo:

```text
Documento
   ↓
Chunks
   ↓
Modelo de embeddings
   ↓
Vetores
   ↓
Base vetorial
```

Quando o usuário fizer uma pergunta, ela também será transformada em representação vetorial.

Isso permite localizar documentos pelo significado da pergunta e não somente pela existência exata das palavras.

Por exemplo:

```text
"qual foi a decisão sobre o projeto X?"
```

poderia encontrar um trecho que utiliza:

```text
"ficou aprovado o início da próxima fase do projeto X"
```

mesmo que as palavras da pergunta e do documento não sejam idênticas.

---

## 4.3 Geração de respostas com IA

A solução utilizaria o padrão **RAG — Retrieval-Augmented Generation**.

Fluxo:

```text
Pergunta do usuário
        ↓
Busca semântica
        ↓
Chunks relevantes
        ↓
Amazon Bedrock
        ↓
Resposta fundamentada
```

Exemplo:

```text
Usuário:
"Quais foram as decisões sobre o projeto de expansão comercial?"
```

A Knowledge Base recuperaria os trechos mais relevantes.

O Amazon Bedrock receberia:

```text
Pergunta
+
Contexto recuperado
+
Regras de resposta
```

E geraria uma resposta baseada somente nesse contexto.

A resposta deverá apresentar as fontes utilizadas, por exemplo:

```text
Resposta:
A reunião registrou a aprovação da próxima etapa do projeto.

Fonte:
- ata_reuniao_vendas_sa.pdf
- Página 3
```

Isso é fundamental para evitar uma Wiki que apenas "parece inteligente".

A empresa precisa conseguir voltar ao documento original e verificar a informação.

---

## 4.4 Interface de consulta

Para uma solução corporativa, uma possibilidade é utilizar o **Amazon Q Business** como interface de perguntas e respostas sobre conteúdo empresarial.

Outra alternativa seria construir uma interface própria:

```text
Amazon Cognito
       ↓
Interface web
       ↓
API Gateway
       ↓
AWS Lambda
       ↓
Bedrock Knowledge Bases
       ↓
Amazon Bedrock
```

O Cognito seria responsável pela autenticação dos usuários.

A solução própria permitiria implementar filtros como:

```text
Tipo de documento
Período
Projeto
Área
Cliente
Confidencialidade
```

---

## 4.5 Segurança, auditoria e monitoramento

### IAM

IAM seria utilizado para definir permissões de:

* Ingestão;
* Processamento;
* Consulta;
* Administração.

Cada função teria somente as permissões necessárias.

---

### KMS

Os documentos armazenados no S3 seriam protegidos com criptografia utilizando AWS KMS.

Isso reduz o risco de exposição de documentos corporativos.

---

### Cognito

Para a interface web, o Amazon Cognito controlaria:

* Login;
* Identidade;
* Grupos;
* Permissões.

Exemplo:

```text
Administradores
    ↓
Acesso completo

Comercial
    ↓
Documentos comerciais

Financeiro
    ↓
Documentos financeiros
```

---

### CloudTrail

O AWS CloudTrail registraria atividades relacionadas à conta e aos recursos AWS.

Isso permite investigar:

* Quem acessou recursos;
* Alterações de configuração;
* Eventos administrativos;
* Atividades relacionadas aos serviços.

---

### CloudWatch

O Amazon CloudWatch monitoraria:

* Falhas de Lambda;
* Execuções do Step Functions;
* Processamentos do Textract;
* Latência;
* Erros;
* Volume de documentos;
* Indicadores operacionais.

Alarmes poderiam ser configurados para falhas recorrentes.

---

### Controle de custos

O AWS Cost Explorer seria utilizado para acompanhar os custos.

Os principais componentes a serem monitorados seriam:

* Amazon S3;
* Amazon Textract;
* Amazon Bedrock;
* Base vetorial;
* Lambda;
* Step Functions;
* CloudWatch.

---

# 🧩 Arquitetura Final da Solução

## 1. Visão geral

A arquitetura proposta utiliza o Amazon S3 como camada central de armazenamento e separa o processamento de acordo com o tipo de documento.

O PDF digital é tratado por extração de texto, a imagem digitalizada passa pelo Amazon Textract e o CSV é processado como dado estruturado.

Após a normalização, os conteúdos recebem metadados e são disponibilizados no Amazon Bedrock Knowledge Bases.

O usuário realiza uma pergunta em linguagem natural e a solução utiliza busca semântica para recuperar os trechos relevantes antes de gerar a resposta com Amazon Bedrock.

A resposta mantém a referência ao documento original, garantindo rastreabilidade.

---

## 2. Serviços AWS utilizados

| Serviço AWS                    | Papel na solução                                     |
| ------------------------------ | ---------------------------------------------------- |
| Amazon S3                      | Armazenamento dos documentos originais e processados |
| Amazon Textract                | OCR da imagem digitalizada                           |
| Amazon Bedrock                 | IA para enriquecimento e geração de respostas        |
| Amazon Bedrock Knowledge Bases | Orquestração da base de conhecimento e RAG           |
| AWS Lambda                     | Classificação, parsing, normalização e processamento |
| AWS Step Functions             | Orquestração do workflow                             |
| Amazon CloudWatch              | Logs, métricas e alarmes                             |
| AWS IAM                        | Controle de permissões                               |
| AWS KMS                        | Criptografia                                         |
| Amazon DynamoDB                | Metadados e informações operacionais                 |
| Amazon OpenSearch Serverless   | Armazenamento vetorial                               |
| Amazon Cognito                 | Autenticação dos usuários                            |
| Amazon API Gateway             | API da aplicação                                     |
| AWS CloudTrail                 | Auditoria                                            |
| AWS Cost Explorer              | Acompanhamento de custos                             |

---

## 3. Fluxo de dados de ponta a ponta

```text
1. Arquivos são disponibilizados na pasta raw/

2. Os arquivos são enviados para o Amazon S3

3. O S3 preserva os arquivos originais

4. Um evento de upload aciona o pipeline

5. AWS Lambda identifica o tipo do arquivo

6. AWS Step Functions direciona o processamento

7. PDF digital:
      Extração direta do texto

8. PNG:
      Amazon Textract → OCR

9. CSV:
      Lambda → validação → normalização → registros estruturados

10. Conteúdo processado é armazenado no S3

11. Metadados são gerados

12. Amazon Bedrock pode enriquecer o conteúdo com:
      - resumo
      - temas
      - decisões
      - responsáveis
      - pendências
      - riscos

13. Documentos são divididos em chunks

14. Amazon Bedrock gera embeddings

15. Embeddings são armazenados na base vetorial

16. Knowledge Base disponibiliza busca semântica

17. Usuário realiza uma pergunta

18. A pergunta é utilizada para recuperar os trechos relevantes

19. Amazon Bedrock gera a resposta utilizando o contexto recuperado

20. A resposta apresenta as fontes utilizadas

21. CloudWatch e CloudTrail monitoram a operação
```

---

## 4. Diagrama textual da arquitetura

```text
                    ┌──────────────────┐
                    │   Arquivos raw/  │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │    Amazon S3     │
                    │  Original/Raw    │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │   AWS Lambda     │
                    │ Classificação    │
                    └────────┬─────────┘
                             │
                             ▼
                  ┌───────────────────────┐
                  │    Step Functions     │
                  └───────────┬───────────┘
                              │
             ┌────────────────┼─────────────────┐
             │                │                 │
             ▼                ▼                 ▼
      ┌────────────┐   ┌────────────┐   ┌──────────────┐
      │ PDF digital│   │ PNG/Image  │   │ CSV          │
      │ Texto      │   │ Textract   │   │ Parser       │
      └─────┬──────┘   └─────┬──────┘   └──────┬───────┘
            │                │                  │
            └────────────────┼──────────────────┘
                             ▼
                    ┌──────────────────┐
                    │ S3 Processed     │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ Bedrock          │
                    │ Enriquecimento   │
                    └────────┬─────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │ Bedrock Knowledge    │
                  │ Bases                │
                  └──────────┬───────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ Base Vetorial    │
                    │ OpenSearch       │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ Usuário / Wiki   │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ Amazon Bedrock   │
                    │ Resposta + Fonte │
                    └──────────────────┘


IAM + KMS + CloudTrail + CloudWatch
                │
                └── Segurança, auditoria e monitoramento
```

---

## 5. Riscos e limitações

Os principais riscos são:

* Documentos escaneados podem possuir baixa qualidade;
* OCR pode interpretar incorretamente textos manuscritos;
* Informações manuscritas podem exigir validação humana;
* Modelos de IA podem gerar informações incorretas se não forem devidamente limitados ao contexto recuperado;
* Metadados inferidos por IA devem ser tratados como dados enriquecidos e não como substitutos do documento original;
* Consultas podem não encontrar informação quando o documento não estiver na base;
* Custos podem aumentar conforme o volume de documentos e consultas;
* A base vetorial precisa ser monitorada e atualizada;
* Documentos confidenciais exigem políticas de acesso adequadas;
* Dados do CSV podem conter informações que não deveriam estar disponíveis para todos os usuários.

### Política para informação inexistente

A Wiki não deve inventar uma resposta.

Quando não houver evidência suficiente nos documentos recuperados, a resposta deverá informar:

```text
"Não encontrei informação suficiente nos documentos disponíveis
para responder a essa pergunta."
```

Quando possível, também deverá informar quais documentos foram consultados.

---

## 6. Melhorias futuras

A solução poderia evoluir para:

* Ingestão automática de novos documentos;
* Classificação automática por tipo;
* Classificação por departamento;
* Classificação por nível de confidencialidade;
* Controle de acesso por grupo;
* Interface web corporativa;
* Integração com Amazon Q Business;
* Dashboard de decisões;
* Dashboard de pendências;
* Alertas para prazos próximos;
* Detecção de documentos duplicados;
* Validação humana de OCR com baixa confiança;
* Monitoramento de qualidade das respostas;
* Avaliação automática de respostas RAG;
* Controle de custos por departamento;
* Lifecycle automático de documentos;
* Retenção e governança conforme políticas corporativas.

---

# 🧠 Checklist Final

* [x] Como transformar documentos escaneados em texto?
* [x] Como lidar com diferentes formatos dentro da mesma pasta `raw/`?
* [x] Como armazenar os documentos originais?
* [x] Como preservar a rastreabilidade entre resposta e documento fonte?
* [x] Como organizar metadados?
* [x] Como criar busca semântica?
* [x] Como usar Amazon Bedrock na solução?
* [x] Como proteger documentos sensíveis?
* [x] Como monitorar falhas?
* [x] Como a empresa usaria essa Wiki no dia a dia?

---

# 🏁 Conclusão

A proposta transforma um conjunto de documentos heterogêneos em uma plataforma de conhecimento corporativo utilizando exclusivamente serviços AWS.

O ponto central da arquitetura é **não tratar todos os arquivos da mesma maneira**. O PDF digital deve aproveitar sua camada de texto, a imagem deve passar por OCR utilizando Amazon Textract e o CSV deve permanecer como dado estruturado.

O Amazon S3 preserva os documentos originais, enquanto Lambda e Step Functions organizam o processamento. O Amazon Bedrock pode enriquecer os documentos com informações como temas, decisões e pendências. O Amazon Bedrock Knowledge Bases disponibiliza os conteúdos para busca semântica e RAG.

Dessa forma, uma pergunta como:

```text
"Quais foram as decisões sobre o projeto de expansão comercial?"
```

pode ser transformada em uma busca semântica, recuperar os trechos relevantes e gerar uma resposta utilizando o Amazon Bedrock, mantendo a referência ao documento de origem.

O resultado é uma arquitetura escalável, auditável e preparada para evolução, na qual a IA não substitui os documentos originais: ela facilita o acesso ao conhecimento existente neles.

A principal garantia da solução é a **rastreabilidade**: toda informação utilizada pela Wiki deve continuar vinculada ao documento original armazenado no Amazon S3.
