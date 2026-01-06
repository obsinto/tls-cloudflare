# Guia: Endpoint TLS para Cloudflare Managed Networks

## O que é isso?

Este guia explica como criar um **endpoint TLS** (HTTPS) no seu servidor Pop!_OS para usar com o **Cloudflare Managed Networks** (anteriormente conhecido como WARP).

### Para que serve?

O **Cloudflare Managed Networks** permite que você:
- Conecte dispositivos de forma segura à sua rede privada através do Cloudflare Zero Trust
- Crie túneis seguros sem precisar de VPN tradicional
- Proteja aplicações internas atrás da rede do Cloudflare

Para isso funcionar, você precisa de um **endpoint TLS** que:
1. Escuta conexões HTTPS na porta 443
2. Possui um certificado SSL (pode ser auto-assinado)
3. Responde às verificações de saúde (health checks) do Cloudflare

---

## Arquitetura

```
Internet → Cloudflare Network → Seu Endpoint TLS (192.168.31.228:443)
                                        ↓
                                   Nginx Container
                                        ↓
                                  Aplicação Interna
```

---

## Pré-requisitos

- Pop!_OS (ou qualquer Linux com Docker)
- Docker e Docker Compose instalados
- Acesso à porta 443
- IP do servidor: `192.168.31.228` (ajuste conforme sua rede)

---

## Passo 1: Criar Diretório do Projeto

```bash
mkdir -p ~/Documents/nginx
cd ~/Documents/nginx
```

---

## Passo 2: Gerar Certificado SSL Auto-Assinado

### Por que certificado auto-assinado?

O Cloudflare **não valida a autoridade certificadora** (CA), mas sim o **fingerprint SHA256** do certificado. Isso significa que:
- ✅ Você pode usar certificados auto-assinados (gratuito)
- ✅ A conexão continua criptografada com TLS
- ✅ O Cloudflare garante autenticidade através do fingerprint

### Comando para criar o certificado:

```bash
openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout example.key \
  -out example.pem \
  -days 3650 \
  -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,DNS:192.168.31.228,IP:192.168.31.228,IP:127.0.0.1"
```

**Explicação dos parâmetros:**
- `-x509`: Cria certificado auto-assinado
- `-newkey rsa:2048`: Gera chave RSA de 2048 bits
- `-nodes`: Não criptografa a chave privada (sem senha)
- `-keyout example.key`: Nome do arquivo da chave privada
- `-out example.pem`: Nome do arquivo do certificado
- `-days 3650`: Válido por 10 anos
- `-subj "/CN=localhost"`: Nome comum do certificado
- `-addext "subjectAltName=..."`: Permite múltiplos nomes/IPs

**Arquivos gerados:**
- `example.key` - Chave privada (mantenha seguro!)
- `example.pem` - Certificado público

### Obter o Fingerprint SHA256:

```bash
openssl x509 -noout -fingerprint -sha256 -inform pem -in example.pem | tr -d : | cut -d= -f2
```

**Exemplo de output:**
```
AD628CAFE95DF9EF6986C349A5A5EF8B48A98F5B882C05AED3D92FE136E9A95D
```

> ⚠️ **IMPORTANTE**: Guarde este fingerprint! Você precisará dele no painel do Cloudflare.

---

## Passo 3: Criar Configuração do Nginx

Crie o arquivo `nginx.conf`:

```bash
cat > nginx.conf << 'EOF'
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    client_max_body_size 20M;

    # HTTPS Server - Managed Network Endpoint
    server {
        listen 443 ssl;
        listen [::]:443 ssl;
        http2 on;
        server_name _;

        ssl_certificate /etc/nginx/certs/cert.pem;
        ssl_certificate_key /etc/nginx/certs/key.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;

        # Health check endpoint para Cloudflare
        location /health {
            access_log off;
            return 200 "OK\n";
            add_header Content-Type text/plain;
        }

        # Root simples
        location / {
            return 200 "Cloudflare Managed Network Endpoint\n";
            add_header Content-Type text/plain;
        }
    }

    # HTTP redirect (opcional, para debug)
    server {
        listen 80;
        listen [::]:80;
        server_name _;

        location / {
            return 301 https://$host$request_uri;
        }
    }
}
EOF
```

**O que esta configuração faz:**
- Escuta na porta 443 com SSL/TLS
- Suporta HTTP/2
- Endpoint `/health` para verificações do Cloudflare
- Endpoint `/` com mensagem de confirmação
- Redireciona HTTP (porta 80) para HTTPS

---

## Passo 4: Criar Docker Compose

Crie o arquivo `compose.yml`:

```bash
cat > compose.yml << 'EOF'
version: '3.8'

services:
  managed-network-endpoint:
    image: nginx:latest
    container_name: cloudflare-managed-network
    network_mode: host
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./example.pem:/etc/nginx/certs/cert.pem:ro
      - ./example.key:/etc/nginx/certs/key.key:ro
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "-k", "https://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
EOF
```

**Por que `network_mode: host`?**
- Elimina camada extra de NAT do Docker
- Melhor desempenho de rede
- Necessário para alguns cenários de Managed Networks
- O container usa diretamente a rede do host

---

## Passo 5: Iniciar o Container

```bash
docker compose up -d
```

**Verificar se está rodando:**

```bash
docker compose ps
```

**Saída esperada:**
```
NAME                         IMAGE          STATUS                    PORTS
cloudflare-managed-network   nginx:latest   Up 30 seconds (healthy)
```

---

## Passo 6: Testar o Endpoint

### Teste local:

```bash
# Testar endpoint raiz
curl -k https://localhost/

# Testar health check
curl -k https://localhost/health

# Testar com o IP do servidor
curl -k https://192.168.31.228/
```

**Respostas esperadas:**
- `/` → `Cloudflare Managed Network Endpoint`
- `/health` → `OK`

### No navegador:

Acesse: `https://192.168.31.228`

Você verá um aviso de segurança (**isso é normal!**):
```
Your connection is not private
net::ERR_CERT_AUTHORITY_INVALID
```

- Clique em "Advanced" → "Proceed to 192.168.31.228"
- Você verá: "Cloudflare Managed Network Endpoint"

---

## Passo 7: Configurar no Cloudflare

### No painel do Cloudflare Zero Trust:

1. Acesse: Settings → Network → Managed Networks
2. Clique em "Add a network"
3. Preencha:
   - **Name**: Nome da sua rede
   - **Endpoint**: `https://192.168.31.228:443`
   - **SHA256 Fingerprint**: `AD628CAFE95DF9EF6986C349A5A5EF8B48A98F5B882C05AED3D92FE136E9A95D`
4. Salve

### Como obter o fingerprint novamente:

```bash
openssl x509 -noout -fingerprint -sha256 -inform pem -in example.pem | tr -d : | cut -d= -f2
```

---

## Comandos Úteis

### Ver logs em tempo real:
```bash
docker compose logs -f
```

### Parar o container:
```bash
docker compose stop
```

### Iniciar o container:
```bash
docker compose start
```

### Reiniciar o container:
```bash
docker compose restart
```

### Parar e remover:
```bash
docker compose down
```

### Verificar status:
```bash
docker compose ps
docker inspect cloudflare-managed-network
```

---

## Estrutura de Arquivos

```
~/Documents/nginx/
├── compose.yml              # Configuração do Docker Compose
├── nginx.conf               # Configuração do Nginx
├── example.pem              # Certificado SSL público
├── example.key              # Chave privada SSL
└── GUIA-ENDPOINT-TLS.md     # Este guia
```

---

## Troubleshooting

### Porta 443 já está em uso:

```bash
# Verificar o que está usando a porta 443
sudo lsof -i :443

# Parar outro serviço se necessário
```

### Certificado expirado:

```bash
# Verificar data de expiração
openssl x509 -noout -dates -in example.pem

# Recriar certificado (passos 2 e 7)
```

### Container não inicia:

```bash
# Ver logs de erro
docker compose logs

# Testar configuração do nginx
docker run --rm -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf nginx nginx -t
```

### Conexão recusada:

```bash
# Verificar se o container está rodando
docker compose ps

# Verificar se a porta está escutando
ss -tlnp | grep :443

# Reiniciar container
docker compose restart
```

---

## Segurança

### Boas práticas:

1. **Proteja a chave privada:**
   ```bash
   chmod 600 example.key
   ```

2. **Use firewall para limitar acesso:**
   ```bash
   # Permitir apenas Cloudflare IPs (exemplo)
   sudo ufw allow from 173.245.48.0/20 to any port 443
   ```

3. **Monitore os logs:**
   ```bash
   docker compose logs -f | grep -E "error|warn"
   ```

4. **Atualize regularmente:**
   ```bash
   docker compose pull
   docker compose up -d
   ```

---

## Conceitos Importantes

### TLS/SSL:
- **TLS** (Transport Layer Security) criptografa a comunicação
- Garante privacidade e integridade dos dados
- Certificado auto-assinado é válido para este caso de uso

### Fingerprint SHA256:
- Hash único do certificado
- Como uma "impressão digital" do certificado
- Cloudflare usa isso para validar autenticidade
- Mesmo que o certificado seja auto-assinado, o fingerprint garante que é o certificado correto

### Network Mode Host:
- Container usa a rede do host diretamente
- Sem NAT/port mapping do Docker
- Melhor desempenho
- Necessário para alguns casos de uso do Cloudflare

---

## Próximos Passos

Depois de configurar o endpoint:

1. Configure políticas de acesso no Cloudflare Zero Trust
2. Instale o WARP client nos dispositivos
3. Configure rotas de rede
4. Teste a conectividade

---

## Recursos Adicionais

- [Cloudflare Zero Trust Documentation](https://developers.cloudflare.com/cloudflare-one/)
- [Nginx SSL Configuration](https://nginx.org/en/docs/http/configuring_https_servers.html)
- [OpenSSL Documentation](https://www.openssl.org/docs/)

---

## Licença

Este guia é de domínio público. Use livremente!

---

**Criado em:** 2025-12-27
**Versão:** 1.0
