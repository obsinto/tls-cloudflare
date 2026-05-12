# Guia — Moto E4 como TLS Beacon da Cloudflare Managed Network

Este guia transforma o Moto E4 com LineageOS + Termux em um **endpoint TLS local** para o Cloudflare Zero Trust Managed Network.

O objetivo é resolver o conflito entre:

```text
PC em casa
= deve acessar 192.168.31.0/24 direto pela LAN

PC/celular fora de casa
= deve acessar 192.168.31.0/24 pelo WARP/Tunnel
```

O Moto E4 será usado como uma “prova física” de que você está dentro da rede de casa.

---

## 1. Arquitetura

Rede local:

```text
LAN: 192.168.31.0/24
Moto E4: 192.168.31.150
Porta TLS: 8443
```

Fluxo desejado:

```text
Se WARP alcança https://192.168.31.150:8443
→ estou em casa
→ aplica perfil Casa
→ 192.168.31.0/24 fica fora do WARP

Se WARP não alcança https://192.168.31.150:8443
→ estou fora de casa
→ aplica perfil Fora
→ 192.168.31.0/24 passa pelo WARP/Tunnel
```

Este guia não configura AdGuard, DNS, DHCP, MQTT ou cloudflared no Moto E4. O foco é somente o **TLS beacon** da Managed Network.

---

## 2. Requisitos

No Moto E4:

```text
LineageOS instalado
Termux instalado
Termux:Boot instalado
Termux:API instalado
IP reservado no roteador: 192.168.31.150
```

No roteador, reserve o IP do Moto E4 para ele não mudar:

```text
192.168.31.150
```

---

## 3. Instalar pacotes no Termux

No Moto E4, abra o Termux ou acesse via SSH:

```bash
pkg update && pkg upgrade
pkg install python openssl termux-api openssh nano
```

---

## 4. Criar pasta do beacon

```bash
mkdir -p ~/warp-beacon
cd ~/warp-beacon
```

---

## 5. Gerar certificado TLS autoassinado

```bash
openssl req -x509 -newkey rsa:2048 \
  -keyout key.pem \
  -out cert.pem \
  -days 3650 \
  -nodes \
  -subj "/CN=motoe4-managed-network"
```

Arquivos criados:

```text
cert.pem = certificado público
key.pem  = chave privada
```

---

## 6. Criar servidor HTTPS em Python

Crie o arquivo:

```bash
nano ~/warp-beacon/server.py
```

Cole:

```python
from http.server import HTTPServer, BaseHTTPRequestHandler
import ssl
import json
from datetime import datetime

HOST = "0.0.0.0"
PORT = 8443

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        data = {
            "status": "ok",
            "service": "motoe4-cloudflare-managed-network-beacon",
            "time": datetime.utcnow().isoformat() + "Z",
        }

        body = json.dumps(data, indent=2).encode("utf-8")

        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

    def log_message(self, format, *args):
        return

httpd = HTTPServer((HOST, PORT), Handler)

context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
context.load_cert_chain(certfile="cert.pem", keyfile="key.pem")

httpd.socket = context.wrap_socket(httpd.socket, server_side=True)

print(f"WARP Beacon ativo em https://{HOST}:{PORT}")
httpd.serve_forever()
```

Salve:

```text
Ctrl + O
Enter
Ctrl + X
```

---

## 7. Testar manualmente

No Moto E4:

```bash
cd ~/warp-beacon
python server.py
```

Deve aparecer:

```text
WARP Beacon ativo em https://0.0.0.0:8443
```

No PC:

```bash
curl -k https://192.168.31.150:8443
```

Resposta esperada:

```json
{
  "status": "ok",
  "service": "motoe4-cloudflare-managed-network-beacon",
  "time": "..."
}
```

Se respondeu, o servidor TLS está funcionando.

Pare o servidor no Moto E4 com:

```text
Ctrl + C
```

---

## 8. Obter fingerprint SHA-256 do certificado

No Moto E4:

```bash
cd ~/warp-beacon

openssl x509 -noout -fingerprint -sha256 -inform pem -in cert.pem | tr -d :
```

Saída parecida:

```text
SHA256 Fingerprint=DD4F4806C57A5BBAF1AA5B080F0541DA75DB468D0A1FE731310149500CCD8662
```

Use no Cloudflare somente o valor depois de:

```text
SHA256 Fingerprint=
```

Também é possível obter pelo PC:

```bash
openssl s_client -connect 192.168.31.150:8443 < /dev/null 2>/dev/null \
  | openssl x509 -noout -fingerprint -sha256 | tr -d :
```

---

## 9. Criar script de boot automático

Crie a pasta do Termux:Boot:

```bash
mkdir -p ~/.termux/boot
```

Crie o script:

```bash
nano ~/.termux/boot/start-beacon.sh
```

Cole:

```bash
#!/data/data/com.termux/files/usr/bin/sh

# Mantém CPU acordada
termux-wake-lock

# Inicia SSH se ainda não estiver rodando
pgrep -f "sshd" >/dev/null || sshd

# Inicia o TLS beacon se ainda não estiver rodando
cd /data/data/com.termux/files/home/warp-beacon || exit 1

pgrep -f "python server.py" >/dev/null && exit 0

nohup python server.py > beacon.log 2>&1 &
```

Dê permissão:

```bash
chmod +x ~/.termux/boot/start-beacon.sh
```

Abra o app **Termux:Boot** pelo menos uma vez.

---

## 10. Testar sem reiniciar

No Moto E4:

```bash
~/.termux/boot/start-beacon.sh
```

Verifique:

```bash
ps aux | grep -E "server.py|sshd"
cat ~/warp-beacon/beacon.log
```

No PC:

```bash
curl -k https://192.168.31.150:8443
ssh motoe4
```

---

## 11. Teste real após reboot

Reinicie o Moto E4.

Aguarde 30 a 60 segundos.

Sem abrir o Termux manualmente, teste do PC:

```bash
curl -k https://192.168.31.150:8443
ssh motoe4
```

Se responder, o Moto E4 está funcional como TLS beacon e SSH node.

---

## 12. Configurar no Cloudflare Zero Trust

No painel Cloudflare Zero Trust:

```text
Settings
WARP Client
Managed Networks
Add managed network
```

Crie:

```text
Name: Home LAN
Type: TLS
Host/Port: 192.168.31.150:8443
TLS Cert SHA-256: fingerprint gerado no passo 8
```

Use host/porta sem `https://`:

```text
192.168.31.150:8443
```

---

## 13. Criar perfis WARP

A lógica é ter dois perfis.

### Perfil Casa

Aplicado quando a Managed Network `Home LAN` for detectada.

```text
Split Tunnel mode: Exclude

Exclude:
192.168.31.0/24
```

Resultado:

```text
Dentro de casa, seu PC acessa a LAN direto.
```

Teste no PC:

```bash
ip route get 192.168.31.150
```

Esperado:

```text
192.168.31.150 dev wlp... src 192.168.31.xxx
```

Não deve aparecer:

```text
dev CloudflareWARP
```

---

### Perfil Fora

Aplicado quando a Managed Network não for detectada.

```text
Split Tunnel mode: Include

Include:
192.168.31.0/24
```

Resultado:

```text
Fora de casa, sua LAN passa pelo WARP/Tunnel.
```

Fora de casa, o teste:

```bash
ip route get 192.168.31.150
```

deve mostrar:

```text
dev CloudflareWARP
```

---

## 14. Ajustes no Android/Lineage

No Moto E4:

```text
desativar otimização de bateria para Termux
desativar otimização de bateria para Termux:Boot
manter Wi-Fi ativo durante suspensão
deixar carregando
reservar IP no roteador
abrir Termux:Boot ao menos uma vez
```

No Termux:

```bash
termux-wake-lock
```

---

## 15. Diagnóstico

Ver servidor:

```bash
ps aux | grep server.py
```

Ver SSH:

```bash
ps aux | grep sshd
```

Ver log:

```bash
cat ~/warp-beacon/beacon.log
```

Ver IP:

```bash
ip addr show wlan0
```

Testar do PC:

```bash
curl -k https://192.168.31.150:8443
ssh motoe4
```

Ver rota no PC:

```bash
ip route get 192.168.31.150
```

---

## 16. O que este setup resolve

Antes:

```text
PC principal roda cloudflared e WARP
PC tenta acessar LAN pela Cloudflare mesmo estando em casa
rota vai para CloudflareWARP
SSH local quebra
```

Depois:

```text
Moto E4 serve como prova física de que você está em casa
WARP aplica perfil Casa automaticamente
LAN fica fora do túnel quando você está em casa
fora de casa, LAN volta a passar pelo WARP/Tunnel
```

---

## 17. Limitações

O Moto E4 precisa ficar:

```text
ligado
carregando
na mesma rede
com IP fixo
com Termux vivo
com Wi-Fi estável
```

Se o celular dormir, cair do Wi-Fi ou o Android matar o Termux, a Managed Network pode não ser detectada.

---

## 18. Resumo rápido

No Moto E4:

```bash
pkg install python openssl termux-api openssh nano

mkdir -p ~/warp-beacon
cd ~/warp-beacon

openssl req -x509 -newkey rsa:2048 \
  -keyout key.pem \
  -out cert.pem \
  -days 3650 \
  -nodes \
  -subj "/CN=motoe4-managed-network"
```

Criar `server.py`, testar:

```bash
python server.py
```

No PC:

```bash
curl -k https://192.168.31.150:8443
```

Fingerprint:

```bash
openssl x509 -noout -fingerprint -sha256 -inform pem -in cert.pem | tr -d :
```

Cloudflare:

```text
Managed Network:
192.168.31.150:8443
SHA-256 fingerprint do cert.pem

Perfil Casa:
Exclude 192.168.31.0/24

Perfil Fora:
Include 192.168.31.0/24
```
