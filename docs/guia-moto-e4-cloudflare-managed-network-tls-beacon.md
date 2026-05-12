# Guia completo — Moto E4 como TLS Beacon da Cloudflare Managed Network

Este guia configura o **Moto E4 com LineageOS + Termux** como um **endpoint TLS local** para o recurso **Managed Networks** do Cloudflare Zero Trust.

A função do Moto E4 será simples e importante: servir como uma “prova física” de que você está dentro da sua rede de casa.

Quando o WARP conseguir alcançar o beacon TLS no Moto E4, ele entende que você está em casa e aplica o perfil correto. Quando não conseguir alcançar, ele entende que você está fora e aplica outro perfil.

---

## 1. Problema que este guia resolve

O problema atual é que o PC principal roda dois papéis ao mesmo tempo:

```text
cloudflared
= fornece/anuncia a rede local para a Cloudflare

WARP/Zero Trust
= consome/acessa redes privadas pela Cloudflare
```

Isso cria um conflito quando você está em casa.

Dentro de casa, o PC já está fisicamente na rede:

```text
192.168.31.0/24
```

Então ele deveria acessar dispositivos locais diretamente pela LAN, por exemplo:

```text
Moto E4: 192.168.31.150
```

Mas com WARP ativo, o tráfego pode ir para a interface:

```text
CloudflareWARP
```

em vez de ir direto pela Wi-Fi/LAN.

O resultado é erro como:

```text
ssh: connect to host 192.168.31.150 port 8022: No route to host
```

O Managed Network resolve isso aplicando perfis diferentes conforme o dispositivo esteja dentro ou fora da rede de casa.

---

## 2. Arquitetura desejada

Rede local:

```text
LAN: 192.168.31.0/24
Moto E4: 192.168.31.150
Porta TLS do beacon: 8443
Porta SSH do Termux: 8022
```

Fluxo desejado:

```text
Se WARP alcança 192.168.31.150:8443
→ estou em casa
→ aplica perfil Casa
→ 192.168.31.0/24 fica fora do WARP
→ acesso local vai direto pela LAN

Se WARP não alcança 192.168.31.150:8443
→ estou fora de casa
→ aplica perfil Fora
→ 192.168.31.0/24 passa pelo WARP/Tunnel
→ acesso remoto à LAN funciona via Cloudflare
```

Este guia **não** configura:

```text
AdGuard
DNS
DHCP
MQTT
Uptime Kuma
cloudflared no Moto E4
```

O foco é somente deixar o Moto E4 funcional como **TLS Beacon** para Cloudflare Managed Network.

---

## 3. Requisitos

No Moto E4:

```text
LineageOS instalado
Termux instalado pelo F-Droid
Termux:Boot instalado pelo F-Droid
Termux:API instalado pelo F-Droid
IP reservado no roteador
```

No Termux:

```text
python
openssl
openssh
termux-api
nano
```

No roteador, reserve o IP do Moto E4:

```text
192.168.31.150
```

Essa reserva é importante. Se o IP do Moto E4 mudar, o Cloudflare não encontrará mais o beacon.

---

## 4. Entendendo Termux, Termux:Boot e Termux:API

### Termux

É o terminal Linux dentro do Android.

É nele que vamos instalar pacotes, criar o servidor Python e rodar o SSH.

### Termux:Boot

É o app que executa scripts automaticamente quando o Android inicia.

Ele procura scripts em:

```bash
~/.termux/boot/
```

Tudo que estiver ali com permissão de execução pode rodar no boot.

### Termux:API

É a ponte entre o Termux e recursos do Android.

Para comandos como:

```bash
termux-wake-lock
termux-notification
termux-battery-status
```

funcionarem, você precisa de duas coisas:

```text
1. App Termux:API instalado pelo F-Droid
2. Pacote termux-api instalado dentro do Termux
```

O app sozinho não basta. O pacote sozinho também não basta.

---

## 5. Preparar o Android/LineageOS

No Moto E4:

```text
1. Abra o Termux pelo menos uma vez
2. Abra o Termux:Boot pelo menos uma vez
3. Desative otimização de bateria para Termux
4. Desative otimização de bateria para Termux:Boot
5. Desative otimização de bateria para Termux:API
6. Deixe o Wi-Fi ativo durante suspensão, se essa opção existir
7. Deixe o aparelho carregando
8. Reserve o IP 192.168.31.150 no roteador
```

Esses ajustes reduzem o risco do Android matar os processos em segundo plano.

---

## 6. Instalar pacotes no Termux

No Moto E4, abra o Termux ou acesse por SSH e rode:

```bash
pkg update && pkg upgrade
pkg install python openssl termux-api openssh nano
```

Confirme:

```bash
python --version
openssl version
sshd -h 2>&1 | head
```

O comando `sshd -h` pode mostrar ajuda ou erro de uso; isso é normal. O importante é o comando existir.

---

## 7. Configurar SSH no Termux

Defina uma senha:

```bash
passwd
```

Inicie o SSH manualmente para testar:

```bash
sshd
```

Descubra o usuário:

```bash
whoami
```

No nosso caso:

```text
u0_a66
```

Descubra o IP:

```bash
ip addr show wlan0
```

No PC, teste:

```bash
ssh -p 8022 u0_a66@192.168.31.150
```

Se você já configurou alias no PC, o arquivo `~/.ssh/config` pode ter:

```sshconfig
Host motoe4
    HostName 192.168.31.150
    User u0_a66
    Port 8022
```

Então o acesso fica:

```bash
ssh motoe4
```

---

## 8. Criar pasta do beacon

No Termux:

```bash
mkdir -p ~/warp-beacon
cd ~/warp-beacon
```

---

## 9. Gerar certificado TLS autoassinado

Ainda dentro de `~/warp-beacon`:

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

O certificado será usado pelo servidor HTTPS local.

A Cloudflare vai validar o certificado pelo fingerprint SHA-256.

---

## 10. Criar servidor HTTPS em Python

Crie o arquivo:

```bash
nano ~/warp-beacon/server.py
```

Cole este conteúdo completo:

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

## 11. Testar manualmente o beacon

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

Teste também se a porta está aberta:

```bash
nc -vz 192.168.31.150 8443
```

Se respondeu, o beacon está funcionando.

Pare o servidor no Moto E4 com:

```text
Ctrl + C
```

---

## 12. Obter fingerprint SHA-256 do certificado

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

Exemplo:

```text
DD4F4806C57A5BBAF1AA5B080F0541DA75DB468D0A1FE731310149500CCD8662
```

Também é possível obter pelo PC:

```bash
openssl s_client -connect 192.168.31.150:8443 < /dev/null 2>/dev/null \
  | openssl x509 -noout -fingerprint -sha256 | tr -d :
```

---

## 13. Criar script de boot automático

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

# Entra na pasta do beacon
cd /data/data/com.termux/files/home/warp-beacon || exit 1

# Evita iniciar servidor duplicado
pgrep -f "python server.py" >/dev/null && exit 0

# Inicia beacon TLS em background
nohup python server.py > beacon.log 2>&1 &
```

Dê permissão de execução:

```bash
chmod +x ~/.termux/boot/start-beacon.sh
```

Abra o app **Termux:Boot** pelo menos uma vez no Android.

---

## 14. Testar o script sem reiniciar

No Moto E4:

```bash
~/.termux/boot/start-beacon.sh
```

Verifique se os processos subiram:

```bash
ps aux | grep -E "server.py|sshd"
```

Verifique o log:

```bash
cat ~/warp-beacon/beacon.log
```

No PC:

```bash
curl -k https://192.168.31.150:8443
ssh motoe4
```

Também teste portas:

```bash
nc -vz 192.168.31.150 8443
nc -vz 192.168.31.150 8022
```

Se `curl` e `ssh` funcionarem, o script está correto.

---

## 15. Teste real após reboot

Reinicie o Moto E4.

Aguarde de 30 a 60 segundos.

Sem abrir o Termux manualmente, no PC:

```bash
curl -k https://192.168.31.150:8443
ssh motoe4
```

Se ambos funcionarem, o Moto E4 está configurado corretamente para:

```text
iniciar SSH automaticamente
iniciar beacon TLS automaticamente
servir como Managed Network endpoint
```

---

## 16. Diagnóstico no Moto E4

Ver se o servidor está rodando:

```bash
ps aux | grep server.py
```

Ver se o SSH está rodando:

```bash
ps aux | grep sshd
```

Ver log do beacon:

```bash
cat ~/warp-beacon/beacon.log
```

Ver IP do Wi-Fi:

```bash
ip addr show wlan0
```

Testar comandos da Termux:API:

```bash
termux-battery-status
termux-wifi-connectioninfo
```

Se `termux-wake-lock` não funcionar, revise:

```text
App Termux:API instalado pelo F-Droid
Pacote termux-api instalado no Termux
Permissões do Android
Otimização de bateria
```

---

## 17. Configurar Managed Network no Cloudflare Zero Trust

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
TLS Cert SHA-256: fingerprint gerado no passo 12
```

Use o host/porta sem `https://`:

```text
192.168.31.150:8443
```

---

## 18. Criar perfis WARP

A lógica é ter dois perfis.

---

### Perfil Casa

Aplicado quando a Managed Network `Home LAN` for detectada.

Configuração:

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

Resultado esperado:

```text
192.168.31.150 dev wlp... src 192.168.31.xxx
```

Não deve aparecer:

```text
dev CloudflareWARP
```

---

### Perfil Fora

Aplicado quando a Managed Network `Home LAN` não for detectada.

Configuração:

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

## 19. Checklist final no Android

No Moto E4:

```text
[ ] Termux instalado pelo F-Droid
[ ] Termux:Boot instalado pelo F-Droid
[ ] Termux:API instalado pelo F-Droid
[ ] Termux aberto pelo menos uma vez
[ ] Termux:Boot aberto pelo menos uma vez
[ ] Pacote termux-api instalado dentro do Termux
[ ] SSH configurado
[ ] Senha definida com passwd
[ ] IP reservado no roteador
[ ] Otimização de bateria desativada para Termux
[ ] Otimização de bateria desativada para Termux:Boot
[ ] Otimização de bateria desativada para Termux:API
[ ] Beacon respondendo em https://192.168.31.150:8443
[ ] SSH respondendo em 192.168.31.150:8022
```

---

## 20. Checklist final no PC

No PC:

```bash
curl -k https://192.168.31.150:8443
ssh motoe4
nc -vz 192.168.31.150 8443
nc -vz 192.168.31.150 8022
ip route get 192.168.31.150
```

Dentro de casa, a rota esperada é pela interface local:

```text
dev wlp...
```

Não pela interface:

```text
CloudflareWARP
```

---

## 21. O que este setup resolve

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

## 22. Limitações

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

## 23. Resumo rápido

No Moto E4:

```bash
pkg update && pkg upgrade
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

Criar `server.py`:

```bash
nano ~/warp-beacon/server.py
```

Testar:

```bash
cd ~/warp-beacon
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

Termux:Boot:

```bash
mkdir -p ~/.termux/boot
nano ~/.termux/boot/start-beacon.sh
chmod +x ~/.termux/boot/start-beacon.sh
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

Teste final:

```bash
curl -k https://192.168.31.150:8443
ssh motoe4
ip route get 192.168.31.150
```
