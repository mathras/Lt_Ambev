# Desafio TechLead - Sistema de Geração de Relatórios (XLSX e PDF)

Este projeto descreve a arquitetura de um sistema escalável e resiliente para geração de relatórios em formato XLSX e PDF, utilizando mensageria para processamento assíncrono e garantindo idempotência.

---

## Arquitetura do Sistema

O sistema é composto pelos seguintes componentes principais:

1. **API Gateway**: Recebe as requisições dos clientes e as encaminha para o sistema de mensageria.
2. **Kafka**: Gerencia as filas de mensagens e garante o processamento assíncrono das requisições.
3. **Worker Nodes**: Processam as mensagens da fila, geram os relatórios (XLSX e PDF) e atualizam o status do processo.
4. **Database (PostgreSQL)**: Armazena o status de cada requisição e os metadados dos relatórios gerados.
5. **Storage AWS(S3) ou Bucket(GCP)**: Armazena os arquivos gerados e disponibiliza links para download.
6. **Cache (Redis)**: Otimiza consultas de status, reduzindo a carga no banco de dados.

---

## Fluxo do Sistema

1. O cliente faz uma requisição POST `/generate` para gerar um relatório.
2. A API Gateway recebe a requisição e a publica em um tópico do Kafka.
3. Um Worker Node consome a mensagem do Kafka e inicia o processamento.
4. O Worker Node gera o relatório (XLSX ou PDF) e o armazena no S3.
5. O status do processo é atualizado no PostgreSQL e no Redis.
6. O cliente pode consultar o status do processo através do endpoint GET `/status/{id}`.
7. O cliente recebe um link para download do relatório gerado.

---

## Justificativa Técnica

### 1. Escolha da Tecnologia de Mensageria (Kafka)

O **Kafka** foi escolhido como sistema de mensageria devido às seguintes razões:
- **Alta escalabilidade**: Capaz de lidar com grandes volumes de mensagens.
- **Tolerância a falhas**: Mensagens são replicadas em múltiplos nós, garantindo alta disponibilidade.
- **Reprocessamento**: Permite reprocessar mensagens antigas, o que é útil em cenários de falhas.
- **Desempenho**: Projetado para alta throughput e baixa latência.

### 2. Estratégia de Escalabilidade

O sistema foi projetado para escalar **horizontalmente**:
- **Worker Nodes**: Podem ser escalados de forma independente para lidar com picos de demanda.
- **Load Balancer**: Distribui as requisições entre os Workers, garantindo processamento equilibrado.
- **Kafka**: Escalável horizontalmente, permitindo adicionar mais brokers conforme necessário.

### 3. Idempotência e Evitar Reprocessamento

Para garantir idempotência:
- Cada requisição é identificada por um **UUID único**.
- Antes de processar uma mensagem, o Worker Node verifica no PostgreSQL se a requisição já foi processada.
- Isso evita que requisições duplicadas sejam processadas mais de uma vez.

### 4. Lidando com Falhas e Retries

O sistema utiliza as seguintes estratégias para lidar com falhas:
- **Retentativas automáticas**: O Kafka permite configurar políticas de retentativa em caso de falhas no processamento.
- **Dead Letter Queue (DLQ)**: Mensagens que falham repetidamente são enviadas para uma DLQ, onde podem ser analisadas e reprocessadas manualmente.
- **Atualização de status**: Em caso de falha, o status da requisição é atualizado para "Error" no PostgreSQL e no Redis.

### 5. Uso de Banco de Dados e Cache

- **PostgreSQL**: Utilizado para armazenar o status das requisições e metadados dos relatórios. Escolhido por sua confiabilidade e suporte a transações ACID.
- **Redis**: Utilizado como cache para consultas de status, reduzindo a carga no PostgreSQL e melhorando o tempo de resposta para o cliente.

---

## Gerenciamento de Status

O status de cada requisição é gerenciado através de um sistema de estados:

1. **Queued**: A requisição foi recebida e está na fila de processamento.
2. **Processing**: A requisição está sendo processada por um Worker Node.
3. **Done**: O relatório foi gerado com sucesso e está disponível para download.
4. **Error**: Ocorreu um erro durante o processamento.

O status é armazenado no **PostgreSQL** (para persistência) e no **Redis** (para consultas rápidas).

---

## Ferramentas Utilizadas

- **Kafka**: Para mensageria e processamento assíncrono.
- **PostgreSQL**: Para armazenamento de status e metadados.
- **Redis**: Para cache de consultas de status.
- **S3**: Para armazenamento de arquivos gerados.
- **Docker/Kubernetes**: Para orquestração de containers e escalabilidade dos Worker Nodes.

---

## Conclusão

Este sistema foi projetado para ser altamente escalável, resiliente e capaz de lidar com grandes volumes de requisições. A escolha do Kafka como sistema de mensageria, combinada com a escalabilidade horizontal dos Worker Nodes e o uso de PostgreSQL e Redis para gerenciamento de status, garante que o sistema seja robusto e eficiente.