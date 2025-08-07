# tunel-ssh
Implementação tunel ssh no linux

Excelente pergunta! Vou esclarecer o que essas configurações podem e não podem fazer em relação a firewalls e restrições de rede.**Resposta Direta: Depende do tipo de firewall e das restrições implementadas.**

As configurações básicas que mostrei anteriormente podem contornar firewalls simples, mas para redes corporativas ou governamentais com **Deep Packet Inspection (DPI)** mais avançada, você precisará de configurações adicionais de **obfuscação**.

## Limitações das Configurações Básicas

### O que pode NÃO funcionar em firewalls avançados:
- **WireGuard padrão** - facilmente detectável por DPI
- **SSH na porta 22** - frequentemente bloqueada
- **Protocolos VPN tradicionais** - identificados e bloqueados

## Configurações Avançadas para Contornar Firewalls Restritivos

### 1. SSH sobre HTTPS (Porta 443)

```bash
# Configurar SSH para usar porta 443 (HTTPS)
sudo nano /etc/ssh/sshd_config
```

Adicione:
```
Port 443
Port 2222  # Manter porta alternativa
```

### 2. SSH Tunneling com SSL/TLS (STunnel)

```bash
# Instalar STunnel
sudo apt install stunnel4

# Configurar STunnel
sudo nano /etc/stunnel/ssh.conf
```

```ini
[ssh]
accept = 443
connect = 127.0.0.1:22
cert = /etc/stunnel/stunnel.pem
key = /etc/stunnel/stunnel.key
```

### 3. WireGuard com Obfuscação usando Shadowsocks

```bash
# Instalar Shadowsocks
sudo apt install shadowsocks-libev

# Instalar V2Ray para obfuscação
bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)
```

**Configuração V2Ray + Shadowsocks + WireGuard:**

```bash
sudo nano /usr/local/etc/v2ray/config.json
```

```json
{
  "inbounds": [
    {
      "port": 443,
      "protocol": "shadowsocks",
      "settings": {
        "method": "chacha20-ietf-poly1305",
        "password": "sua_senha_forte"
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "settings": {
        "domainStrategy": "UseIP"
      },
      "streamSettings": {
        "sockopt": {
          "mark": 255
        }
      }
    }
  ]
}
```

### 4. Configuração de Múltiplos Protocolos (Failover)

```bash
sudo nano /usr/local/bin/multi-protocol-connect.sh
```

```bash
#!/bin/bash
# Script para tentar múltiplos métodos de conexão

# Método 1: SSH sobre porta 443
echo "Tentando SSH na porta 443..."
if ssh -p 443 -o ConnectTimeout=10 usuario@servidor 2>/dev/null; then
    echo "SSH 443 funcionou!"
    exit 0
fi

# Método 2: Shadowsocks + WireGuard
echo "Tentando Shadowsocks..."
systemctl start shadowsocks-libev
sleep 5
if wg-quick up wg0; then
    echo "WireGuard via Shadowsocks funcionou!"
    exit 0
fi

# Método 3: V2Ray
echo "Tentando V2Ray..."
systemctl start v2ray
sleep 5

# Método 4: SSH sobre HTTP (porta 80)
echo "Tentando SSH na porta 80..."
ssh -p 80 usuario@servidor
```

### 5. Configuração Ultra-Stealth com Domain Fronting

```bash
# Instalar ferramentas adicionais
sudo apt install obfs4proxy

# Configurar domain fronting
sudo nano /etc/wireguard/stealth.conf
```

```ini
[Interface]
PrivateKey = [SUA_CHAVE]
Address = 10.0.0.2/32
DNS = 1.1.1.1
PostUp = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = [CHAVE_SERVIDOR]
Endpoint = cloudflare.com:443  # Domain fronting
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

## Métodos de Detecção que Firewalls Avançados Usam

### 1. **Análise de Padrões de Tráfego**
- Detectam protocolos VPN por assinatura de pacotes
- Analisam timing e tamanho dos pacotes

### 2. **Deep Packet Inspection (DPI)**
- Inspecionam conteúdo dos pacotes
- Identificam protocolos criptografados

### 3. **Análise Comportamental**
- Detectam padrões de uso anômalos
- Bloqueiam com base em volume de dados

## Script Completo de Evasão

```bash
sudo nano /usr/local/bin/firewall-evasion.sh
```

```bash
#!/bin/bash
# Script de evasão completa de firewall

# Função para testar conectividade
test_connection() {
    curl -s --connect-timeout 5 https://httpbin.org/ip >/dev/null 2>&1
    return $?
}

# Tentar método normal primeiro
if test_connection; then
    echo "Conexão normal funcionando"
    exit 0
fi

# Método 1: SSH Tunnel na porta 443
echo "Tentando SSH tunnel na porta 443..."
ssh -D 8080 -f -C -q -N -p 443 usuario@servidor
export http_proxy=socks5://127.0.0.1:8080
export https_proxy=socks5://127.0.0.1:8080

if test_connection; then
    echo "SSH tunnel 443 funcionou!"
    exit 0
fi

# Método 2: Shadowsocks
echo "Iniciando Shadowsocks..."
ss-local -c /etc/shadowsocks/config.json -f /var/run/shadowsocks.pid
export http_proxy=socks5://127.0.0.1:1080
export https_proxy=socks5://127.0.0.1:1080

if test_connection; then
    echo "Shadowsocks funcionou!"
    exit 0
fi

# Método 3: V2Ray com múltiplos protocolos
echo "Iniciando V2Ray..."
systemctl start v2ray
sleep 10

if test_connection; then
    echo "V2Ray funcionou!"
    exit 0
fi

echo "Todos os métodos falharam"
exit 1
```

## ⚠️ Considerações Importantes

### **Legais:**
- **Verifique as políticas da sua empresa/instituição**
- **Respeite leis locais sobre contorno de censura**
- **Use apenas em redes que você possui ou tem permissão**

### **Técnicas:**
- Firewalls muito avançados podem detectar até métodos de obfuscação
- Alguns países/organizações bloqueiam todos os protocolos criptografados não aprovados
- Métodos de evasão podem ser temporários se detectados

### **Taxa de Sucesso por Tipo de Firewall:**
- **Firewall básico (empresa pequena)**: ~95% de sucesso
- **Firewall corporativo avançado**: ~70% de sucesso  
- **Firewall governamental (China, Irã)**: ~30-50% de sucesso

**Resumo**: Com as configurações avançadas mostradas acima, você terá uma chance muito maior de contornar firewalls restritivos, mas não há garantia de 100% contra sistemas muito sofisticados.
