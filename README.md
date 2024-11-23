# Migração e Configuração de Instância do n8n para Modo Fila

Este guia tem dois objetivos principais:

1. Migrar uma instância existente do n8n em modo normal para o modo fila.
2. Configurar uma nova instância do n8n diretamente no modo fila.

A configuração contempla:

- Banco de dados: **PostgreSQL**
- **Redis** para gerenciamento de filas.
- 3 instâncias separadas:
  - **Editor** (interface gráfica do n8n)
  - **Webhook** (responsável pelos gatilhos)
  - **Worker** (processamento das fluxos)
- Configuração de concorrência: **5**.

---

## Parte 1: Migração de uma Instância Existente

### Passo a Passo

1. **Clone o repositório com os arquivos de configuração**

```bash
git clone https://github.com/AlexSzefezuk/n8n-config.git
```

2. **Recupere a encryption key da instância antiga**

   - A chave está no arquivo `config` dentro da pasta `.n8n` localizada no container do n8n.
   - Geralmente você terá uma volume mapeado, basta buscar pelo buscar no volume pelo arquivo.

3. **Aponte um domínio provisório para a sua VPS.**

4. **Atualize todas as variáveis de ambiente no arquivo `.env`.**
   ```bash
   cp .env.example .env
   ```
5. **Exporte os workflows da instância antiga:**

   - Entre no container antigo:
     ```bash
     docker ps # pegue o id do seu container antigo
     ```
     ```bash
     docker exec -it <ID_DO_CONTAINER> /bin/sh
     ```
   - Execute o comando de exportação de workflows:
     ```bash
     n8n export:workflow --all --output=/home/node/.n8n/workflows.json
     ```
     **Nota:** Altere o caminho do output para o volume configurado no seu ambiente.

6. **Exporte as credenciais da instância antiga:**

   - Ainda dentro do container:
     ```bash
     n8n export:credentials --all --output=/home/node/.n8n/credentials.json
     ```
     **Nota:** Altere o caminho do output para o volume configurado no seu ambiente.

7. **Crie os volumes necessários:**

   ```bash
   docker volume create n8n_queue_data && \
   docker volume create n8n_db_storage && \
   docker volume create n8n_redis_storage
   ```

8. **Suba o novo container:**

- No arquivo `docker-compose.yml` deixei a versão do n8n de forma fixa, sugiro que faça o mesmo, assim você pode controlar sempre que quiser fazer a atualização.
- Para ver qual a versão mais recente estável na sua data basta acessar o [Docker Hub do n8n](https://hub.docker.com/r/n8nio/n8n) ou o [GitHub do n8n](https://github.com/n8n-io/n8n).
- Quando for mudar de versão, lembre-se de alterar nas 3 instancias dentro do `docker-compose.yml`

  - `n8n_editor`
  - `n8n_webhook`
  - `n8n_worker`

- Comando para subir a os containers

```bash
docker compose up -d
```

9. **Mova os arquivos de workflows e credenciais para o volume mapeado.**

- No caso da configuração deste repo, basta mover os arquivos `workflows.json` e `credentials.json` que você criou nos passos `5` e `6` para dentro da pasta projects na raiz onde esta o seu `docker-compose.yml` pois o volume ja esta mapeado
- Crie a pasta se necessário
  ```bash
  mkdir projects
  ```

10. **Importe os workflows e credenciais no novo container:**

    - Workflows:
      ```bash
      n8n import:workflow --input=/home/projects/workflows.json
      ```
    - Credenciais:
      ```bash
      n8n import:credentials --input=/home/projects/credentials.json
      ```

11. **Configure o NGINX para apontar o domínio provisório para o novo container:**

    - Crie um arquivo de configuração, por exemplo, `n8nprovisorio.conf`:
    - Você também pode fazer uma cópia do arquivo `n8n.conf` para dentro da sua pasta de configuração do nginx

    ```nginx
    server {
      server_name n8n-provisorio.seudominio.com.br;

      location ~* /webhook/ {
        proxy_pass http://localhost:56782;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
      }

      location / {
        proxy_pass http://localhost:56781;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
      }
    }
    ```

    **Ative a configuração e gere um SSL:**

```bash
sudo ln -s /etc/nginx/sites-available/n8nprovisorio.conf /etc/nginx/sites-enabled/
```

```bash
nginx -t # Verifique se tudo esta correto
```

```bash
certbot --nginx # Gera o certificado SSL
```

```bash
sudo systemctl reload nginx # Recarrega o nginx
```

12. **Ative os workflows necessários:**

    - Se houver workflows de gatilhos como o do Facebook Lead Ads Trigger, desative-os na instância antiga antes de ativá-los na nova.

13. **Finalize a migração:**
    - Atualize a variável de domínio no arquivo `.env` para o domínio correto.
    - Configure o `NGINX` para apontar para o novo container, conforme o arquivo `n8n.conf`.
    - Desative a configuração do domínio provisório no `NGINX`, basta apagar o arquivo provisório e rodar o comando que recarrega o `NGINX`.
    - Derrube o container da sua instância antiga.
      - Sugiro que não apague ela até passar alguns dias com sua instancia nova rodando sem erros

---

## Parte 2: Configuração de uma Nova Instância no Modo Fila

Caso esteja criando uma nova instância do zero no modo fila, siga os passos abaixo:

1. **Clone o repositório com os arquivos de configuração:**

   ```bash
   git clone https://github.com/AlexSzefezuk/n8n-config.git
   ```

2. **Atualize o arquivo `.env` com as variáveis de ambiente necessárias.**

   ```bash
   cp .env.example .env
   ```

3. **Crie os volumes necessários:**

   ```bash
   docker volume create n8n_queue_data && \
   docker volume create n8n_db_storage && \
   docker volume create n8n_redis_storage
   ```

4. **Suba o container:**

   ```bash
   docker compose up -d
   ```

5. **Configure o NGINX para apontar o domínio para o novo container:**

   ```bash
   cp n8n.conf /etc/nginx/sites-available/n8n.conf
   ```

   - Não esqueça de atualizar com o seu domínio
   - Essa configuração nao precisa de um subdomínio separado para o webhook, ambos ficam no mesmo domínio

   **Ative a configuração:**

```bash
sudo ln -s /etc/nginx/sites-available/n8n.conf /etc/nginx/sites-enabled/
```

```bash
nginx -t # Verifique se tudo esta correto
```

```bash
certbot --nginx # Gera o certificado SSL
```

```bash
sudo systemctl reload nginx # Recarrega o nginx
```

6. **Acesse a interface do n8n no navegador e comece a configurar seus workflows.**

---

## Notas Finais

- Certifique-se de que os containers estão rodando corretamente com:

  ```bash
  docker ps
  ```

- Monitore os logs para verificar possíveis erros:

  ```bash
  docker logs <ID_DO_CONTAINER>
  ```

  ou dentro da pasta onde esta seu compose

  ```bash
  docker-compose logs
  ```

- A configuração foi projetada para separar as funções de edição, webhooks e processamento, garantindo maior desempenho e escalabilidade.
