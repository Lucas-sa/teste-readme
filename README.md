# Resumo Expandido das Configurações de Rede e PABX para Tronco E1 SIP Vivo

## 📋 Configuração de Rede para Tronco E1 SIP

### Dados Essenciais que a Operadora Deve Fornecer

Ao solicitar um tronco E1 SIP ou similar a uma operadora, você sempre precisará fornecer as seguintes informações:

1.  **Dados da Rede (para a Placa de Rede):**
    * **IP do PABX na Rede da Operadora:** (Ex: `10.12.116.130`)
    * **Máscara de Sub-rede:** (Ex: `255.255.255.248`).
    * **IP do Gateway da Operadora:** (Ex: `10.12.116.129`)
    * **IPs dos Servidores DNS da Operadora:** (Ex: `10.255.240.111`)

2.  **Dados do SIP/Mídia (para o Asterisk):**
    * **IP do(s) Balanceador(es) SIP / SIP Proxy:** (Ex: `10.255.240.111` para Massivo, `10.255.245.1` para Rajada). Este é o `host` principal do seu tronco.
    * **IP do Servidor de Mídia (RTP):** (Ex: `10.255.240.112`).
    * **Porta SIP Utilizada:** (Geralmente `5060` UDP).
    * **Codecs Permitidos e Priorizados:** (Ex: `G.711-Alaw(PCMA)`, `G.729`).
    * **Range de Portas RTP:** (Ex: `11000 a 19999` UDP).
    * **Método de DTMF:** (Geralmente `RFC 2833` ou `RFC 4733`).
    * **Identificação:**
        * Número Piloto: (Ex: `1131464000`).
        * Range de Ramais/DIDs: (Ex: `1131464001 ao 1131464029`).
    * **Quantidade de Canais:** (Ex: `30`). Para configurar `call-limit`.

    **E, se for um tronco com AUTENTICAÇÃO E REGISTRO, eles deverão fornecer ADICIONALMENTE:**
    * **Nome de Usuário (Username):** (Ex: `1131464000` ou `TRUNK_ABC_ID`).
    * **Senha (Secret):** (Ex: `senha_segura_da_vivo`).
    * **Informação de Registro (se aplicável):** Se o registro precisar ser em um domínio/IP diferente do proxy, eles especificarão (Ex: `register => user:pass@registro.vivo.com.br/piloto`). Caso contrário, o IP/domínio do balanceador é usado.
    * **Tipo de Tronco (Peer-to-peer por IP vs. Requer Registro/Autenticação):** Esta é a pergunta chave a ser feita à operadora.

### I. Configuração da Placa de Rede no Linux (`/etc/network/interfaces`)

**Conceito Importante:** O Linux permite apenas um gateway padrão por sistema. Para usar múltiplas interfaces de rede (eth0 para internet, eth1 para telefonia), utilizamos **Policy Routing** com tabelas de roteamento específicas.

**Objetivo:** Garantir que todo tráfego da operadora (eth1) use suas próprias rotas sem interferir na navegação de internet (eth0).

#### Entendendo os Componentes da Configuração

**1. Tabelas de Roteamento:**
- **Tabela Principal (main):** Contém rotas da eth0 (internet)
- **Tabela 'vivo':** Criada automaticamente para rotas da eth1 (telefonia)

**2. Por que não usar gateway na eth1:**
- Dois gateways padrão causariam conflito
- O sistema não saberia qual usar para cada destino
- Policy Routing resolve isso

**3. Policy Routing:**
- **ip rule:** Define QUANDO usar uma tabela específica
- **ip route:** Define COMO rotear dentro da tabela
- **priority:** Ordem de prioridade das regras (menor número = maior prioridade)

**4. Fluxo de Funcionamento:**
1. Pacote chega ao sistema
2. Linux verifica as regras (`ip rule`) por ordem de prioridade
3. Se o pacote corresponde a uma regra, usa a tabela especificada
4. Dentro da tabela, segue as rotas definidas (`ip route`)

#### Exemplo Prático - Configuração da Interface eth1 (Vivo)

**Arquivo: `/etc/network/interfaces`**

```text
# ==================== INTERFACE DE INTERNET (ETH0) ====================
auto eth0
allow-hotplug eth0
iface eth0 inet static
    address 192.168.0.132
    netmask 255.255.255.0
    gateway 192.168.0.1     # Gateway padrão do sistema para internet

# ==================== INTERFACE DE TELEFONIA (ETH1) ====================
auto eth1                   # Ativa automaticamente a interface no boot
allow-hotplug eth1          # Permite ativação automática quando cabo é conectado
iface eth1 inet static      # Define configuração estática (não DHCP)
    address 10.12.116.130      # IP do PABX fornecido pela operadora
    netmask 255.255.255.248    # Máscara /29 (8 IPs: .128 até .135)
    
    # NÃO colocamos "gateway" aqui para evitar conflito com eth0
    
    # ===== CRIAÇÃO DE TABELA DE ROTEAMENTO ESPECÍFICA =====
    # Comando executado APÓS a interface subir (post-up)
    
    # Regra que define que todo tráfego destinado à sub-rede 10.12.116.128/29 sera direcionado para eth1 e tera como IP de origem 10.12.116.130
    post-up ip route add 10.12.116.128/29 dev eth1 src 10.12.116.130 table vivo

    # A sub-rede 10.12.116.128/29 foi definida com base na máscara 255.255.255.248, usando o calculo de bits.
    # Em ultimo caso pode ser usado o 10.12.116.129 no lugar da sub-rede 10.12.116.128/29

    # Regra que define que a tabela vivo tara como rota padrão o IP 10.12.116.129 (gateway da Vivo) e sera direcionado para eth1
    post-up ip route add default via 10.12.116.129 dev eth1 table vivo
    
    # ===== REGRAS DE POLÍTICA DE ROTEAMENTO =====
    # Determinam QUANDO usar a tabela 'vivo'

    # Esta regra define que todo tráfego originado do IP 10.12.116.130 da eth1 que será direcionado para tabela 'vivo'
    post-up ip rule add from 10.12.116.130 table vivo priority 1000

    # Esta regra define que todo Pacote destinados ao Balanceador SIP Massivo (10.255.240.111) sera direcionado para tabela 'vivo'
    post-up ip rule add to 10.255.240.111 table vivo priority 1001

    # Esta regra define que todo Pacote de Mídia e RTP direcionados ao destino 10.255.240.112 sera direcionado para tabela 'vivo'
    post-up ip rule add to 10.255.240.112 table vivo priority 1002

    # Esta regra define que todo Pacote destinado ao Balanceador SIP Rajada (10.255.245.1) sera direcionado para tabela 'vivo'
    # Usado para servidores com alta demanda de chamadas
    post-up ip rule add to 10.255.245.1 table vivo priority 1003
    
    # ===== LIMPEZA AUTOMÁTICA AO DESATIVAR A INTERFACE =====
    # Comandos executados ANTES da interface ser desativada (pre-down)
    
    # Remove todas as regras de política para evitar conflitos quando a interface for reativada
    pre-down ip rule del from 10.12.116.130 table vivo
    pre-down ip rule del to 10.255.240.111 table vivo
    pre-down ip rule del to 10.255.240.112 table vivo
    pre-down ip rule del to 10.255.245.1 table vivo
    
    # Limpa completamente a tabela 'vivo', removendo todas as rotas específicas
    pre-down ip route flush table vivo
    
    # Explicação: Define o servidor DNS da operadora para esta interface para resolução de dns
    dns-nameservers 10.255.240.111
    
```

#### Aplicação e Verificação da Configuração

**Como aplicar as mudanças:**
```bash
sudo systemctl restart networking
# ou
sudo ifdown eth1 && sudo ifup eth1
```

**Comandos de verificação essenciais:**

1. **Verificar interfaces ativas:**
   ```bash
   ip a show
   ```
   *Deve mostrar eth0 e eth1 com IPs corretos*

2. **Verificar tabela de roteamento principal:**
   ```bash
   ip route show
   ```
   *Deve mostrar apenas o gateway da eth0 como padrão*

3. **Verificar regras de política:**
   ```bash
   ip rule show
   ```
   *Deve mostrar as regras com prioridades 1000-1003 para tabela vivo*

4. **Verificar tabela específica da Vivo:**
   ```bash
   ip route show table vivo
   ```
   *Deve mostrar as rotas específicas incluindo default via 10.12.116.129*

5. **Testar conectividade:**
   ```bash
   ping 10.255.240.111  # Balanceador SIP
   ping 10.255.240.112  # Servidor de Mídia
   ping 10.255.245.1    # Balanceador SIP Rajada
   ping google.com      # Internet via eth0
   ```

#### Troubleshooting Comum

**Problema:** Ping para servidores da Vivo não funciona
- **Verificar:** `ip rule show` e `ip route show table vivo`
- **Solução:** Certificar que as regras foram aplicadas corretamente

**Problema:** Internet para de funcionar após configurar eth1
- **Verificar:** `ip route show` (deve ter apenas gateway da eth0)
- **Solução:** Não colocar gateway na configuração da eth1

**Problema:** Asterisk não consegue se comunicar com a Vivo
- **Verificar:** Se o Asterisk está fazendo bind na interface correta
- **Verificar:** Firewall não está bloqueando portas 5060 e 11000-19999

---

### II. Configuração do PABX Asterisk 

#### Estrutura Modular de Configuração

Para melhor organização, as configurações do Asterisk podem ser divididas em arquivos específicos:

- **`sip_nat.conf`** - Configurações de rede e NAT
- **`rtp.conf`** - Configurações de mídia RTP  
- **`sip.conf`** - Configurações principais do SIP e troncos

```ini
; === CONFIGURAÇÕES DEO sip_nat.conf ===
externip=10.12.116.130    
localnet=10.12.116.128/29 
localnet=192.168.100.0/255.255.255.0

; === CONFIGURAÇÕES DO rtp.conf ===  
rtpstart=11000
rtpend=19999

```

#### Configuração do Arquivo principal: `sip.conf`

**Cenário A: Tronco Baseado em IP (Peer-to-Peer)**

**Característica:** Operadora confia no IP de origem do PABX. Não há necessidade de registro SIP explícito com usuário/senha.

```ini
;================================================================================
; CONFIGURAÇÕES GERAIS DO SIP
;================================================================================
[general]
; === INCLUIR ARQUIVOS DE CONFIGURAÇÃO MODULAR ===
#include "sip_nat.conf"    ; Configurações de rede (externip, localnet)
#include "rtp.conf"        ; Configurações RTP (rtpstart, rtpend)

; === CONFIGURAÇÕES BÁSICAS ===
bindport=5060              ; Porta padrão para SIP (Vivo usa 5060)
bindaddr=0.0.0.0           ; Asterisk ouvirá em todas as interfaces

; === CODECS ===
disallow=all               ; Desabilita todos os codecs primeiro
allow=alaw                 ; Habilita G.711-Alaw (PCMA) - PRIORITÁRIO
allow=g729                 ; Habilita G.729 (verificar licença se necessário)

; === CONFIGURAÇÕES DE TRONCO ===
qualifyfreq=60             ; Verifica tronco a cada 60 segundos
canreinvite=no             ; Recomendado 'no' para troncos de operadora
nat=no                     ; Sem NAT - IP direto da Vivo
dtmfmode=rfc2833           ; Método DTMF padrão

; === SEM REGISTRO (modelo peer-to-peer) ===
; register => NÃO usar neste cenário

;================================================================================
; TRONCO SIP VIVO - COMUNICAÇÃO PEER-TO-PEER
;================================================================================
[vivo_trunk]
type=peer                  ; Comunicação baseada em IP (sem autenticação)
host=10.255.240.111        ; IP do Balanceador SIP VIVO (MASSIVO)
port=5060                  ; Porta SIP da Vivo

; === IDENTIFICAÇÃO ===
fromuser=1131464000        ; Número Piloto (aparece no cabeçalho From)
; fromdomain=10.255.240.111 ; Opcional: apenas se Vivo exigir

; === CONFIGURAÇÕES DE SEGURANÇA ===
insecure=port,invite       ; Aceita chamadas sem autenticação estrita
qualify=yes                ; Monitora disponibilidade do tronco

; === CONFIGURAÇÕES DE MÍDIA ===
directmedia=no             ; Evita problemas de RTP/NAT
dtmfmode=rfc2833          ; Método DTMF
t38pt_udptl=yes           ; Suporte T.38 para fax

; === ROTEAMENTO ===
context=from-vivo          ; Contexto para chamadas de entrada
call-limit=30              ; Limite de canais simultâneos
```

#### Cenário B: Tronco Baseado em Registro e Autenticação

**Característica:** PABX precisa se registrar no servidor da operadora usando usuário e senha.

```ini
;================================================================================
; CONFIGURAÇÕES GERAIS DO SIP
;================================================================================
[general]
; === INCLUIR ARQUIVOS DE CONFIGURAÇÃO MODULAR ===
#include "sip_nat.conf"    ; Configurações de rede (externip, localnet)  
#include "rtp.conf"        ; Configurações RTP (rtpstart, rtpend)

; === CONFIGURAÇÕES BÁSICAS ===
bindport=5060
bindaddr=0.0.0.0
disallow=all
allow=alaw
allow=g729
qualifyfreq=60
canreinvite=no
nat=no                     ; Se IP fixo na rede Vivo = 'no'
                          ; Se PABX atrás de firewall = 'yes'
dtmfmode=rfc2833

; === REGISTRO COM AUTENTICAÇÃO ===
; Formato: register => USUARIO:SENHA@HOST:PORTA/NUMERO_PILOTO
register => SEU_USUARIO_SIP:SUA_SENHA_SIP@10.255.240.111:5060/1131464000

; Explicação do registro:
; SEU_USUARIO_SIP: Nome de usuário fornecido pela Vivo
; SUA_SENHA_SIP: Senha fornecida pela Vivo  
; 10.255.240.111: IP do servidor de registro da Vivo
; 5060: Porta de registro (padrão)
; 1131464000: Número piloto para identificar chamadas de entrada

;================================================================================
; TRONCO SIP VIVO - COM AUTENTICAÇÃO
;================================================================================
[vivo_trunk]
type=friend                ; Permite enviar/receber com autenticação
host=10.255.240.111        ; IP do Balanceador Vivo
port=5060                  ; Porta SIP

; === CREDENCIAIS DE AUTENTICAÇÃO ===
username=SEU_USUARIO_SIP   ; Usuário para autenticação
secret=SUA_SENHA_SIP       ; Senha para autenticação
fromuser=1131464000        ; Número Piloto (cabeçalho From)
fromdomain=10.255.240.111  ; Domínio SIP (IP do balanceador)

; === CONFIGURAÇÕES DE SEGURANÇA ===
insecure=port,invite       ; Ainda útil mesmo com autenticação
qualify=yes                ; Monitora disponibilidade

; === CONFIGURAÇÕES DE MÍDIA ===
directmedia=no
dtmfmode=rfc2833
t38pt_udptl=yes

; === ROTEAMENTO ===
context=from-vivo
call-limit=30
```

#### Aplicação e Verificação das Configurações do Asterisk

**Como aplicar as mudanças:**
```bash
# Acessar o CLI do Asterisk
sudo asterisk -rvvvvvvcgi

# No CLI do Asterisk:
sip reload          # Recarrega configurações SIP
dialplan reload     # Recarrega dialplan (se alterou extensions.conf)
```
