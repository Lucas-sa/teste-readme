# Resumo Expandido das Configurações de Rede e PABX para Tronco SIP Vivo

### Dados Essenciais que a Operadora Deve Fornecer

Ao solicitar um tronco SIP E1 ou similar a uma operadora, você sempre precisará fornecer as seguintes informações:

1.  **Dados da Rede (para a Placa de Rede):**
    * **IP do PABX na Rede da Operadora:** (Ex: `10.12.116.130`)
    * **Máscara de Sub-rede:** (Ex: `255.255.255.248` ou `/29`)
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

Existem duas abordagens para configurar a rede no Linux, especialmente com múltiplas interfaces:

#### 1. Configuração SEM Policy Routing (Abordagem Mais Simples, pode ter Conflitos de Gateway)

Esta abordagem é mais direta. Funciona se você tiver um gateway padrão principal (`eth0`) e precisar de rotas específicas para os IPs da operadora (`eth1`).

**Conteúdo do `/etc/network/interfaces`:**

```text
auto lo
iface lo inet loopback

# A interface de rede principal (para internet/rede local padrão)
auto eth0
allow-hotplug eth0
iface eth0 inet static
        address 192.168.0.132
        netmask 255.255.255.0
        gateway 192.168.0.1  # Este será o gateway padrão do sistema para tráfego geral

# E1 SIP VIVO - Configuração da interface dedicada à operadora
auto eth1
allow-hotplug eth1
iface eth1 inet static
        address 10.12.116.130
        netmask 255.255.255.248
        # O gateway principal já está em eth0. Aqui, usamos o gateway da Vivo SOMENTE para rotas específicas.
        # NÃO COLOQUE "gateway 10.12.116.129" AQUI para evitar conflito de rota padrão.

        # Rotas explícitas para os IPs da Vivo (Balanceador e Mídia)
        post-up route add 10.255.240.111 via 10.12.116.129 dev eth1
        post-up route add 10.255.240.112 via 10.12.116.129 dev eth1

        # Rota para o Balanceador RAJADA (descomente se for usar)
        # post-up route add 10.255.245.1 via 10.12.116.129 dev eth1

        # Servidores DNS para a interface Vivo
        dns-nameservers 10.255.240.111
```

**Quando usar:** Quando o PABX tem um IP fixo na rede da operadora e a operadora confia no IP de origem para sinalização e mídia, e você quer manter a rota padrão da `eth0` para a internet.

**Como aplicar:**
```bash
sudo systemctl restart networking
```
**Verificação:**
* `ip a show`
* `ip route show` (Verifique que a rota padrão é via `eth0` e as rotas específicas para Vivo via `eth1` existem).
* `ping 10.255.240.111`, `ping 10.255.240.112`, `ping google.com`

---

#### 2. Configuração COM Policy Routing (Abordagem Avançada e Robusta para Caso de Conflitos de Gateway)

Esta abordagem é ideal para garantir que todo o tráfego que se origina ou se destina à interface `eth1` (rede da Vivo) use o gateway e as regras de roteamento específicas da Vivo, sem interferir com as outras interfaces.

**Conteúdo do `/etc/network/interfaces`:**

```text
auto lo
iface lo inet loopback

# A interface de rede principal (para internet/rede local padrão)
auto eth0
allow-hotplug eth0
iface eth0 inet static
        address 192.168.0.132
        netmask 255.255.255.0
        gateway 192.168.0.1  # Este será o gateway padrão do sistema para tráfego geral

# E1 SIP VIVO - Configuração da interface dedicada à operadora com Policy Routing
auto eth1
allow-hotplug eth1
iface eth1 inet static
        address 10.12.116.130
        netmask 255.255.255.248
        # O gateway padrão é definido NA TABELA DE ROTEAMENTO SEPARADA (table 100)
        # NÃO COLOQUE "gateway 10.12.116.129" aqui!

        # Definindo a nova tabela de roteamento (table 100) para a interface eth1
        post-up ip route add 10.12.116.128/29 dev eth1 src 10.12.116.130 table 100
        post-up ip route add default via 10.12.116.129 dev eth1 table 100

        # Adicionando regras de roteamento para usar a tabela 100
        # Prioridades (números menores são avaliados primeiro) garantem a ordem
        post-up ip rule add from 10.12.116.130 table 100 priority 1000  # Tráfego originado da eth1
        post-up ip rule add to 10.255.240.111 table 100 priority 1001    # Tráfego destinado ao Balanceador MASSIVO
        post-up ip rule add to 10.255.240.112 table 100 priority 1002    # Tráfego destinado ao IP de Mídia
        # post-up ip rule add to 10.255.245.1 table 100 priority 1003    # Tráfego destinado ao Balanceador RAJADA (se usar)

        # Limpeza das regras e rotas ao derrubar a interface (muito importante!)
        pre-down ip rule del from 10.12.116.130 table 100
        pre-down ip rule del to 10.255.240.111 table 100
        pre-down ip rule del to 10.255.240.112 table 100
        # pre-down ip rule del to 10.255.245.1 table 100 # Tráfego destinado ao Balanceador RAJADA (se usar)

        pre-down ip route flush table 100 # Limpa todas as rotas da tabela 100

        # Servidores DNS para a interface Vivo
        dns-nameservers 10.255.240.111
```

**Quando usar:** Quando é essencial que o tráfego da operadora use **apenas** a interface dedicada e seu próprio gateway, sem interferir com outras rotas padrão no sistema. Ideal para cenários complexos com múltiplas interfaces e requisitos de roteamento específicos.

**Como aplicar:**
```bash
sudo systemctl restart networking
```
**Verificação:**
* `ip a show`
* `ip route show` (Apenas a rota padrão da `eth0` deve aparecer aqui).
* `ip rule show` (Verifique as regras `from` e `to` para a `table 100`).
* `ip route show table 100` (Verifique as rotas específicas da Vivo aqui, incluindo o `default via 10.12.116.129`).
* `ping 10.255.240.111`, `ping 10.255.240.112`, `ping google.com`

---

### II. Configuração do PABX Asterisk (`sip.conf` e `extensions.conf`)

Esta configuração assume que você está usando `chan_sip`.

#### Cenário A: Tronco Baseado em IP (Peer-to-Peer - Modelo da Vivo Fornecido)

**Característica:** Operadora confia no IP de origem do PABX. Não há necessidade de registro SIP explícito com usuário/senha. Geralmente para IPs fixos/dedicados.

#### 1. Arquivo `/etc/asterisk/sip.conf` (ou `sip_custom.conf`)

```ini
;--------------------------------------------------------------------------------
; Configurações Gerais do SIP (chan_sip)
; Estas linhas DEVEM ir na seção [general]
;--------------------------------------------------------------------------------
[general]
externip=10.12.116.130    ; O IP do seu PABX na rede da Vivo que a Vivo deve ver
localnet=10.12.116.128/29 ; Sua sub-rede da eth1. Importante para o Asterisk saber onde está sua rede local.
bindport=5060             ; Porta padrão para SIP (Vivo usa 5060)
bindaddr=0.0.0.0          ; Asterisk ouvirá em todas as interfaces
disallow=all              ; Desabilita todos os codecs primeiro
allow=alaw                ; Habilita G.711-Alaw (PCMA)
allow=g729                ; Habilita G.729 (verifique licença se necessário)

rtpstart=11000            ; Início do range de portas RTP (Vivo: 11000 a 19999)
rtpend=19999              ; Fim do range de portas RTP

qualifyfreq=60            ; Verifica a cada 60 segundos se o tronco está ativo
canreinvite=no            ; Recomendado 'no' para troncos de operadora
nat=no                    ; Não há NAT envolvido, pois você está em IP direto da Vivo
dtmfmode=rfc2833          ; Método DTMF

; Sem linha 'register =>' no modelo baseado em peer-to-peer.

;--------------------------------------------------------------------------------
; Tronco SIP Vivo (E1 SIP VIVO) - Type=peer para comunicação baseada em IP
;--------------------------------------------------------------------------------
[vivo_trunk]
type=peer                 ; Comunicação peer-to-peer, confiança baseada no IP
host=10.255.240.111       ; IP do Balanceador VIVO (MASSIVO)
port=5060                 ; Porta SIP da Vivo (padrão)

fromuser=1131464000       ; Seu número Piloto para identificação no SIP (no cabeçalho From)
; fromdomain=10.255.240.111 ; Opcional: domínio da Vivo, se eles exigirem

insecure=port,invite      ; Permite que o Asterisk aceite chamadas do IP e invites SIP mesmo sem autenticação estrita
qualify=yes               ; Monitora ativamente a disponibilidade do tronco
directmedia=no            ; Mantenha 'no' para evitar problemas de RTP/NAT
dtmfmode=rfc2833
t38pt_udptl=yes           ; Se usar T.38 para fax
context=from-vivo         ; Contexto no extensions.conf para chamadas de entrada
call-limit=30             ; Limite de Canais: 30
```

#### Cenário B: Tronco Baseado em Registro e Autenticação

**Característica:** PABX precisa se registrar no servidor da operadora usando usuário e senha. Comum para IPs dinâmicos ou maior segurança.

#### 1. Arquivo `/etc/asterisk/sip.conf` (ou `sip_custom.conf`)

```ini
;--------------------------------------------------------------------------------
; Configurações Gerais do SIP (chan_sip)
; As configurações de externip, localnet, rtpstart/rtpend, codecs etc. permanecem as mesmas
;--------------------------------------------------------------------------------
[general]
externip=10.12.116.130    ; O IP do seu PABX na rede da Vivo que a Vivo deve ver
localnet=10.12.116.128/29 ; Sua sub-rede da eth1.
bindport=5060
bindaddr=0.0.0.0
disallow=all
allow=alaw
allow=g729
rtpstart=11000
rtpend=19999
qualifyfreq=60
canreinvite=no
nat=no                    ; Se seu IP externo for fixo na rede da Vivo, continua 'no'.
                          ; Se o Asterisk precisar lidar com NAT (ex: PABX atrás de outro firewall), pode ser 'yes'.
dtmfmode=rfc2833

; O Asterisk se registrará no servidor da operadora usando usuário e senha.
; Formato: register => USUARIO:SENHA@HOST_DE_REGISTRO:PORTA_DE_REGISTRO/CONTEXTO_ENTRADA_OU_USUARIO_PARA_INVITES_DE_ENTRADA
register => SEU_USUARIO_SIP:SUA_SENHA_SIP@10.255.240.111:5060/1131464000
```
; Explicação:
; SEU_USUARIO_SIP: O usuário fornecido pela Vivo (ex: 1131464000)
; SUA_SENHA_SIP: A senha fornecida pela Vivo
; 10.255.240.111: O IP do Balanceador VIVO (ou o domínio que eles indicarem para registro)
; 5060: A porta de registro (geralmente 5060)
; 1131464000: Este é o "exten" ou "fromuser" que o Asterisk usará para identificar chamadas de entrada
;             que vêm deste registro (geralmente o número piloto ou o usuário de registro).

```ini
;--------------------------------------------------------------------------------
; Tronco SIP Vivo (E1 SIP VIVO) - Configurado para usar autenticação
;--------------------------------------------------------------------------------
[vivo_trunk]
type=friend               ; 'friend' permite enviar e receber chamadas, lidando com autenticação.
host=10.255.240.111       ; Ainda apontamos para o IP do Balanceador Vivo
port=5060                 ; Porta SIP da Vivo

; **PARA AUTENTICAÇÃO:**
username=SEU_USUARIO_SIP  ; O nome de usuário para autenticar no servidor da Vivo
secret=SUA_SENHA_SIP      ; A senha para autenticar no servidor da Vivo
fromuser=1131464000       ; O número Piloto que será usado no campo From da requisição SIP (CALLERID)
fromdomain=10.255.240.111 ; O domínio SIP para o tronco (geralmente o IP do balanceador ou domínio fornecido)

insecure=port,invite      ; Ainda útil, mas a autenticação é o principal meio de confiança
qualify=yes               ; Monitora a disponibilidade do tronco
directmedia=no            ; Mantenha 'no' para evitar problemas de RTP/NAT
dtmfmode=rfc2833
t38pt_udptl=yes
context=from-vivo         ; Contexto no extensions.conf
call-limit=30             ; Limite de Canais
```

**Como aplicar:**
```bash
sudo asterisk -rvvv # Acessar o CLI
sip reload
dialplan reload
```
**Verificação (para cenário de Registro/Autenticação):**
* **`sip show registry`**: Se usar autenticação por `Registered`.
* `sip show peers` (Verifique status do `vivo_trunk`)
* Faça chamadas de teste de saída e entrada.
