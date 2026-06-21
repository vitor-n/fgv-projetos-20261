## `(A2) Task 1`

### Estrutura do Projeto

```text
.
├── scripts/
│   ├── init_watermark.py              # Inicializa a tabela de watermark no RDS   (A2 - Task 1)
│   ├── simulate_new_orders.py         # Simula a criação de novos pedidos no RDS  (A2 - Task 1)
│   └── validate_incremental_source.py # Valida a origem incremental e o watermark (A2 - Task 1)
├── src/
│   ├── loader.py        # Carga inicial do RDS
│   ├── validate.py      # Validação da integridade do banco de origem
│   ├── glue_etl.py      # Script PySpark executado pelo AWS Glue
│   └── validate_etl.py  # Script de validação dos critérios (4.6)
├── terraform/
│   ├── main.tf          # Definição de recursos AWS
│   ├── terraform.tf     # Configuração de providers
│   └── terraform.tfvars # Variáveis sensíveis
├── dashboard.ipynb      # Notebook com consultas Athena e Dashboard Interativo
└── requirements.txt     # Dependências Python
```

### Como Executar

#### 1. Preparação do Ambiente

Instale as dependências:
```bash
pip install -r requirements.txt
```

#### 2. Infraestrutura (Terraform)

Configure suas credenciais AWS e inicialize o Terraform:
```bash
cd terraform
terraform init
terraform plan
terraform apply
```
*Obs.: O Security Group está configurado para permitir acesso apenas ao seu IP atual automaticamente. O Terraform também criará o Glue Catalog Database e o Glue Crawler necessários para a Task 3.*

#### 3. Carga do Sistema de Origem

Configure o arquivo `.env` com os dados de acesso gerados pelo Terraform e carregue os dados:
```bash
python src/loader.py
python src/validate.py  # Valida se o RDS foi populado corretamente
```

#### 4. Inicialização do Watermark (Task 1 - Assignment 2)

Inicialize a tabela de controle `etl_watermark` no banco de dados (define a data inicial `last_processed_order_date` com o `MAX(orders.orderDate)` da carga histórica):
```bash
python scripts/init_watermark.py
```

#### 5. Validação Inicial da Origem

Rode o script de validação para checar o baseline inicial da origem (deve passar com "Sem pedidos pendentes"):
```bash
python scripts/validate_incremental_source.py
```

#### 6. Simulação de Carga Incremental

Simule novos pedidos fictícios no banco RDS com datas estritamente posteriores ao watermark (aceita `--count` e `--seed`):
```bash
python scripts/simulate_new_orders.py --count 5 --seed 42
```

#### 7. Validação Final de Origem Incremental

Rode a validação novamente para garantir que novos pedidos foram detectados e a estrutura está consistente (deve retornar exit code 0 e apontar que há novos dados para ETL):
```bash
python scripts/validate_incremental_source.py
```

---

## `Task 3 - Consultas e Dashboard`

#### 1. Catalogar as Tabelas no Glue Catalog
Após o término do Job do AWS Glue (Task 2), execute o Glue Crawler para descobrir os schemas e criar as tabelas no banco de dados do Glue Catalog:
```bash
aws glue start-crawler --name star-schema-crawler --region us-east-1
```
*Aguarde cerca de 1 a 2 minutos até que o Crawler retorne ao estado `READY` no console da AWS ou no CLI.*

#### 2. Abrir o Dashboard Interativo
Abra o Jupyter Lab ou VS Code na raiz do projeto:
```bash
jupyter lab
```
E abra o arquivo `dashboard.ipynb`. Execute todas as células para rodar as consultas SQL exploratórias diretamente no Athena (usando `awswrangler`) e interagir com o painel de vendas interativo criado com `ipywidgets` e `seaborn`.

---

## `Task 2 - Detalhes do Pipeline Incremental e Agendamento`

### 1. Tipo de Dados do Watermark
A coluna `last_processed_order_date` na tabela `etl_watermark` é do tipo **`DATE`** do MySQL. A comparação no filtro do Spark SQL é feita convertendo essa data para string e comparando diretamente com a coluna `orderDate` da tabela `orders` (`orderDate > last_processed_order_date`).

### 2. Agendamento e Permissões IAM (EventBridge → AWS Glue)
- **Role do EventBridge**: Para acionar o Job do AWS Glue (`rds-to-s3-star-schema`), o AWS EventBridge utiliza a role **`LabRole`** (especificada como `role_arn` no target da regra do CloudWatch Event).
- **Permissões Associadas**: No ambiente de laboratório (AWS Academy), a `LabRole` já possui a política gerenciada que contém a permissão necessária `glue:StartJobRun`. Além disso, a sua política de confiança (trust policy) permite que o serviço `events.amazonaws.com` assuma a role para executar o agendamento.
