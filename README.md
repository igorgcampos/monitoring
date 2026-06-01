# Stack de Monitoramento — Prometheus + Grafana

Stack de observabilidade containerizada com Prometheus para coleta de métricas e Grafana para visualização.

## Requisitos

- Docker Engine 24+ ou Docker Desktop 4.20+
- Docker Compose v2.20+ (plugin integrado ao Docker)
- Git

---

## Instalação do Docker

### Linux (Ubuntu/Debian)

```bash
# Remova versões antigas, se houver
sudo apt-get remove docker docker-engine docker.io containerd runc

# Instale dependências
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release

# Adicione a chave GPG oficial do Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Adicione o repositório
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instale Docker Engine e o plugin Compose
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Adicione seu usuário ao grupo docker (evita usar sudo)
sudo usermod -aG docker $USER
newgrp docker
```

### WSL2 (Windows)

Instale o **Docker Desktop** no Windows e habilite a integração com WSL2 em:
`Settings → Resources → WSL Integration → habilite sua distro`

O Docker e o Compose ficam disponíveis automaticamente no terminal WSL.

### Verificação da instalação

```bash
docker --version
docker compose version
```

---

## Variáveis de Ambiente

O stack utiliza variáveis de ambiente para evitar que credenciais sejam expostas no código-fonte.

### Variáveis obrigatórias

| Variável               | Descrição                          | Exemplo            |
|------------------------|------------------------------------|--------------------|
| `GRAFANA_ADMIN_PASSWORD` | Senha do usuário `admin` no Grafana | `MinhaS3nh@Forte!` |

### Como definir as variáveis

Crie um arquivo `.env` na raiz do projeto (mesmo diretório do `docker-compose.yml`):

```bash
cp .env.example .env
```

Edite o arquivo `.env` com seus valores:

```dotenv
GRAFANA_ADMIN_PASSWORD=MinhaS3nh@Forte!
```

> O arquivo `.env` já está no `.gitignore` e **nunca deve ser commitado**.

### Boas práticas para a senha

- Mínimo de 12 caracteres
- Combine letras maiúsculas, minúsculas, números e símbolos
- Não reutilize senhas de outros serviços
- Em produção, use um gerenciador de segredos (HashiCorp Vault, AWS Secrets Manager, etc.)

---

## Estrutura do Projeto

```
monitoring/
├── docker-compose.yml                        # Orquestração dos serviços
├── prometheus.yml                            # Configuração e targets do Prometheus
├── grafana.ini                               # Configurações de segurança do Grafana
├── provisioning/
│   ├── datasource.yml                        # Datasource do Prometheus (provisionado automaticamente)
│   └── dashboards/
│       ├── provider.yml                      # Configuração do provedor de dashboards
│       └── infograficos.json                 # Dashboard CloudFront + S3 + EC2
├── .env                                      # Variáveis de ambiente (não versionado)
├── .env.example                              # Modelo de variáveis (versionado)
└── .gitignore
```

---

## Como Executar

### 1. Clone o repositório

```bash
git clone <url-do-repositorio>
cd monitoring
```

### 2. Configure as variáveis de ambiente

```bash
cp .env.example .env
# Edite .env com sua senha
```

### 3. Suba o stack

```bash
docker compose up -d
```

O Grafana aguarda o Prometheus estar saudável antes de iniciar (`depends_on: condition: service_healthy`).

### 4. Verifique o status

```bash
docker compose ps
docker compose logs -f
```

### 5. Acesse as interfaces

| Serviço    | URL                      | Credenciais             |
|------------|--------------------------|-------------------------|
| Grafana    | http://localhost:3000    | admin / `<sua senha>`   |
| Prometheus | http://localhost:9090    | — (sem autenticação)    |

---

## Comandos Úteis

```bash
# Parar o stack (preserva dados)
docker compose stop

# Remover containers (preserva volumes)
docker compose down

# Remover containers e apagar todos os dados
docker compose down -v

# Recarregar configuração do Prometheus sem restart
curl -X POST http://localhost:9090/-/reload

# Ver uso de recursos
docker stats
```

---

## Adicionando Targets ao Prometheus

Edite `prometheus.yml` e adicione seus serviços em `scrape_configs`:

```yaml
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'minha-aplicacao'
    static_configs:
      - targets: ['host.docker.internal:8080']
```

### Acessando aplicações rodando no host (EC2 / Linux)

O hostname `host.docker.internal` **não é resolvido automaticamente no Docker Engine no Linux**. Ele funciona apenas no Docker Desktop (Mac/Windows).

O `docker-compose.yml` já inclui o mapeamento necessário via `extra_hosts` no serviço do Prometheus:

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

Isso faz com que `host.docker.internal` aponte para o IP do host (`172.17.0.1` por padrão), permitindo que o Prometheus alcance serviços rodando fora do Docker.

Após editar o `prometheus.yml`, recarregue sem derrubar o container:

```bash
curl -X POST http://localhost:9090/-/reload
```

---

## Datasource CloudWatch — Acesso às Métricas AWS

O Grafana acessa o CloudWatch diretamente via plugin, sem passar pelo Prometheus. Para isso precisa de permissão para chamar a API da AWS.

```
Grafana ──── Prometheus  (métricas locais)
     └────── CloudWatch  (CloudFront, S3, EC2)
                  │
                  └──► AWS CloudWatch API
```

### Autenticação via IAM Instance Profile (recomendado)

Se o Grafana roda em uma EC2, o container herda as credenciais da instância automaticamente via endpoint de metadados (`169.254.169.254`). Nenhuma chave é necessária no código.

#### 1. Criar a Policy IAM

No **AWS Console → IAM → Policies → Create policy → JSON**, cole:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CloudWatchLeitura",
      "Effect": "Allow",
      "Action": [
        "cloudwatch:GetMetricData",
        "cloudwatch:GetMetricStatistics",
        "cloudwatch:ListMetrics",
        "cloudwatch:DescribeAlarmsForMetric"
      ],
      "Resource": "*"
    },
    {
      "Sid": "TagsEInstancias",
      "Effect": "Allow",
      "Action": [
        "tag:GetResources",
        "ec2:DescribeInstances",
        "ec2:DescribeTags",
        "ec2:DescribeRegions"
      ],
      "Resource": "*"
    }
  ]
}
```

Nome sugerido: `GrafanaCloudWatchReadOnly`

> `tag:GetResources` e `ec2:Describe*` são necessários para as queries EC2 por tag e para os dropdowns de variáveis do dashboard popularem corretamente.

#### 2. Criar a Role

**IAM → Roles → Create role**

- **Trusted entity type:** `AWS service`
- **Use case:** `EC2`
- Selecionar a policy `GrafanaCloudWatchReadOnly`

Nome sugerido: `GrafanaMonitoringRole`

#### 3. Associar a Role à instância EC2

**EC2 → Instances → selecionar a instância → Actions → Security → Modify IAM role**

Selecionar `GrafanaMonitoringRole` → **Update IAM role**

Não é necessário reiniciar a instância.

#### 4. Configurar o datasource no Grafana

**Connections → Add new connection → CloudWatch**

| Campo | Valor |
|---|---|
| Authentication Provider | `AWS SDK Default` |
| Default Region | sua região (ex: `us-east-1`) |

Deixe Access Key e Secret Key em branco. O Grafana buscará as credenciais automaticamente via instance profile.

---

### Autenticação via Access Key (desenvolvimento local / WSL)

Se estiver rodando fora de uma EC2, crie um **IAM User** com acesso programático, anexe a mesma policy `GrafanaCloudWatchReadOnly` e adicione as chaves ao `.env`:

```dotenv
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
AWS_DEFAULT_REGION=us-east-1
```

No datasource do Grafana, selecione **Authentication Provider: Access & secret key** e preencha com os valores acima.

> Nunca commite o `.env` — ele já está no `.gitignore`.

---

### Dashboard provisionado

O dashboard **Infográficos — CloudFront + S3 + EC2** é carregado automaticamente ao subir o stack. Ele aparece em **Dashboards → Infograficos** com variáveis dinâmicas para selecionar região, distribuição CloudFront e bucket S3.

> **Métricas de requisição S3** (latência, erros, contagem de requests) precisam ser habilitadas por bucket em: **S3 → seu bucket → Properties → Request metrics → Create filter**.
>
> **Métricas de disco EC2** (`DiskReadBytes`, `DiskWriteBytes`) aparecem apenas em instâncias com **instance store**. Para instâncias EBS-backed, use o namespace `AWS/EBS`.

---

## Habilitando HTTPS (Produção)

1. Obtenha um certificado TLS (Let's Encrypt, certificado interno, etc.)
2. Coloque os arquivos em `./certs/grafana.crt` e `./certs/grafana.key`
3. Monte o diretório no container e descomente as linhas em `grafana.ini`:

```ini
[server]
protocol = https
cert_file = /etc/grafana/certs/grafana.crt
cert_key  = /etc/grafana/certs/grafana.key
```

4. Habilite `cookie_secure = true` em `grafana.ini` — obrigatório com HTTPS.
5. Ajuste a porta mapeada no `docker-compose.yml` para `443:3000` se necessário.

---

## Segurança

- Imagens fixadas em versões específicas (sem `latest`) para builds reproduzíveis
- Containers rodam com usuário não-root (`user: "1000:1000"` / `"1001:1001"`)
- `no-new-privileges:true` bloqueia escalada de privilégios
- Limites de CPU e memória configurados
- Sign-up público desabilitado
- Autenticação anônima desabilitada
- Embedding desabilitado (prevenção de clickjacking)
- Arquivos de configuração montados como somente leitura (`:ro`)
- Prometheus exposto apenas localmente — considere remover o mapeamento de porta `9090` em produção

---

## Atualização das Imagens

Para atualizar para uma nova versão, edite as tags em `docker-compose.yml`:

```yaml
image: prom/prometheus:v2.54.0   # nova versão
image: grafana/grafana:11.2.0    # nova versão
```

Em seguida:

```bash
docker compose pull
docker compose up -d
```
