# Guia: Como Obter o SHA-256 de um Certificado TLS

## Objetivo
Extrair o fingerprint SHA-256 de um certificado TLS no formato correto para uso em configurações de rede gerenciada.

## Arquivos Encontrados
No diretório `/home/deyvid/Documents/nginx`:
- `example.pem` - Certificado TLS
- `example.key` - Chave privada
- `nginx.conf` - Configuração do nginx

## Passo 1: Extrair o Fingerprint SHA-256

Utilize o comando `openssl` para extrair o fingerprint do certificado:

```bash
openssl x509 -in example.pem -noout -fingerprint -sha256
```

### Resultado:
```
sha256 Fingerprint=F1:AE:31:06:29:FC:91:C2:83:76:C5:D6:19:51:70:40:EA:44:C7:29:96:84:9A:0A:AA:14:34:51:D3:17:51:81
```

## Passo 2: Formatar para Uso em Configurações

O formato padrão do OpenSSL inclui dois pontos (`:`) entre cada par de caracteres hexadecimais. No entanto, muitas configurações de rede requerem o hash sem separadores, contendo exatamente **64 caracteres hexadecimais**.

### Formato com dois pontos (OpenSSL padrão):
```
F1:AE:31:06:29:FC:91:C2:83:76:C5:D6:19:51:70:40:EA:44:C7:29:96:84:9A:0A:AA:14:34:51:D3:17:51:81
```

### Formato sem dois pontos (64 hex chars - CORRETO para redes gerenciadas):
```
F1AE310629FC91C28376C5D6195170400EA44C7296849A0AAA143451D3175181
```

## Solução do Erro

### Erro Original:
```
Error creating managed network: invalid managed network request: expected optional sha256 to have length of 64 hex characters
```

### Solução:
Remova os dois pontos do fingerprint, mantendo apenas os caracteres hexadecimais. O resultado deve ter exatamente 64 caracteres.

## SHA-256 Final do Certificado example.pem

```
F1AE310629FC91C28376C5D6195170400EA44C7296849A0AAA143451D3175181
```

## Comandos Úteis Adicionais

### Ver informações completas do certificado:
```bash
openssl x509 -in example.pem -text -noout
```

### Ver apenas a validade do certificado:
```bash
openssl x509 -in example.pem -noout -dates
```

### Verificar se certificado e chave correspondem:
```bash
openssl x509 -noout -modulus -in example.pem | openssl md5
openssl rsa -noout -modulus -in example.key | openssl md5
```
(Os hashes MD5 devem ser idênticos)
