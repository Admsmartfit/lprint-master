# Manual de Instalação — LPrint para Elgin L42 Pro Full em Servidor Linux

Este manual instala o **LPrint** (aplicativo de código aberto de Michael R Sweet, baseado na
biblioteca PAPPL) em um servidor Ubuntu/Debian, deixando disponível **apenas** o driver da
sua impressora **Elgin L42 Pro Full**, com um painel web simples acessível pelo Chrome.

## 0. O que foi verificado neste código

- O projeto em `lprint-master/` é o **LPrint** oficial (github.com/michaelrsweet/lprint), um
  spooler de impressão para impressoras de etiqueta/recibo. Ele roda como um único executável
  (`lprint`), que tanto funciona por linha de comando quanto como **servidor** com interface
  web embutida (fornecida pela biblioteca PAPPL).
- A Elgin L42 Pro Full **não tem driver nomeado** no LPrint, mas a ficha técnica da impressora
  confirma reconhecimento automático das linguagens **EPL / ZPL / PPLA / PPLB**. Por isso ela
  funciona perfeitamente com o driver **ZPL** já existente no LPrint (mesmo driver usado para
  impressoras Zebra).
- **Já editei o código-fonte** desta pasta (arquivos [lprint.c](lprint.c) e
  [lprint-zpl.h](lprint-zpl.h)) para remover todos os outros fabricantes/drivers (DYMO,
  ESC/POS, Seiko, TSPL, EPL2, Brother, CPCL). Ao compilar esta pasta, o LPrint passa a
  oferecer **somente duas opções**, ambas para a sua impressora:
  - `zpl_4inch-203dpi-dt` → **Elgin L42 Pro Full (Térmica Direta)** — sem ribbon.
  - `zpl_4inch-203dpi-tt` → **Elgin L42 Pro Full (Transferência Térmica)** — com ribbon.
- A etiqueta de autoteste que você imprimiu mostra que a impressora está hoje configurada em
  **Transferência Térmica** (`MODO DE IMPRESSAO: TRANSF.TERMICA`), com IP fixo
  **192.168.15.90**, máscara `255.255.255.0`, 203 dpi. Vamos usar esses dados para configurar
  a fila de impressão.

> Se sua etiqueta é do tipo térmica direta (sem fita/ribbon, a mesma usada em impressoras de
> cupom fiscal), troque o driver para `zpl_4inch-203dpi-dt` no passo 9.

---

## 1. Pré-requisitos

- Servidor com **Ubuntu Server 22.04/24.04** ou **Debian 12+**, acesso root via `sudo`.
- Servidor e impressora na mesma rede (a impressora está em `192.168.15.90/24`, gateway não
  configurado — ok, pois a impressão é feita direto na rede local, sem precisar de rota).
- Impressora Elgin L42 Pro Full ligada, com o modo de linguagem em **Auto** (padrão de
  fábrica reconhece EPL/ZPL/PPLA/PPLB automaticamente — não precisa mudar nada nela).

---

## 2. Instalar as dependências do sistema

```bash
sudo apt update
sudo apt install -y build-essential git pkg-config autoconf \
    libcups2-dev libpam0g-dev libavahi-client-dev \
    libjpeg-dev libpng-dev libusb-1.0-0-dev zlib1g-dev \
    libgnutls28-dev avahi-daemon ufw
```

> Se o `./configure` do PAPPL (próximo passo) reclamar que não encontrou o CUPS, tente
> substituir `libcups2-dev` por `libcups3-dev` (o nome do pacote varia conforme a versão do
> Ubuntu/Debian). O próprio `./configure` informa exatamente qual dependência está faltando.

---

## 3. Compilar e instalar a biblioteca PAPPL

O LPrint depende da biblioteca PAPPL (também de Michael Sweet), que normalmente não vem
empacotada — compilamos do código-fonte:

```bash
cd ~
git clone https://github.com/michaelrsweet/pappl.git
cd pappl
./configure
make
sudo make install
sudo ldconfig
```

---

## 4. Baixar o código-fonte do LPrint no servidor

O código (já com as edições que deixam só a Elgin) está no seu repositório GitHub. No
servidor Linux:

```bash
cd ~
git clone https://github.com/Admsmartfit/lprint-master.git
```

> Se você fizer alterações no código depois, basta rodar `git pull` dentro da pasta
> `~/lprint-master` e repetir os passos 5 e 6 (recompilar e reiniciar o serviço).

---

## 5. Compilar e instalar o LPrint

Já no servidor Linux:

```bash
cd ~/lprint-master
./configure
make
sudo make install
```

Isso instala o binário em `/usr/local/bin/lprint`, as páginas de manual, e o serviço
systemd (`lprint.service`), ainda **desativado**.

Confirme que restou só o driver da Elgin:

```bash
lprint drivers
```

Saída esperada (apenas estas duas linhas):

```
zpl_4inch-203dpi-dt        Elgin L42 Pro Full (Termica Direta)
zpl_4inch-203dpi-tt        Elgin L42 Pro Full (Transferencia Termica)
```

---

## 6. Configurar o serviço (systemd + avahi)

O serviço do LPrint exige o `avahi-daemon` rodando (para descoberta automática de impressora
tipo AirPrint na rede):

```bash
sudo systemctl enable --now avahi-daemon
```

Crie o arquivo de configuração `/etc/lprint.conf` para fixar a porta e liberar o acesso pela
rede (em vez da porta aleatória padrão):

```bash
sudo tee /etc/lprint.conf > /dev/null <<'EOF'
server-name=lprint-server
server-port=8050
listen-hostname=*
server-options=multi-queue,web-interface,web-log,web-network
log-file=/var/log/lprint.log
log-level=info
spool-directory=/var/spool/lprint
EOF

sudo mkdir -p /var/spool/lprint
```

> **Sobre a linha `server-options`:** por padrão o LPrint usaria
> `multi-queue,web-interface,web-log,web-security,web-tls`. Ao definir `server-options=`
> explicitamente acima, isso é **substituído** — de propósito removi `web-security` (sem
> pedir login, painel "simples" como pedido) e `web-tls` (sem HTTPS autoassinado, evita aviso
> de certificado no Chrome). Isso é aceitável em uma rede local confiável, mas **qualquer
> pessoa na rede poderá ver o status e cancelar trabalhos**. Se quiser exigir login, veja a
> seção 12.

Ative e inicie o serviço:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now lprint.service
sudo systemctl status lprint.service
```

---

## 7. Liberar a porta no firewall

```bash
sudo ufw allow 8050/tcp comment "LPrint web"
sudo ufw allow 5353/udp comment "mDNS/Avahi"
sudo ufw status
```

(Pule este passo se o `ufw` não estiver ativo no servidor — confira com `sudo ufw status`.)

---

## 8. Remover filas de impressora antigas (se existirem)

Se você já tinha testado o LPrint antes e cadastrado outras impressoras, liste e remova:

```bash
lprint printers
sudo lprint delete -d NOME-DA-FILA-ANTIGA
```

---

## 9. Adicionar a impressora Elgin L42 Pro Full

Use o IP fixo mostrado na etiqueta de autoteste (`192.168.15.90`). Escolha o driver conforme
o tipo de etiqueta que você usa:

```bash
# Se usa RIBBON (transferência térmica) — modo atual da impressora:
sudo lprint add -d ElginL42Pro \
  -v socket://192.168.15.90 \
  -m zpl_4inch-203dpi-tt \
  -o media-ready=na_index-4x6_4x6in

# OU, se usa etiqueta térmica direta (sem ribbon):
# sudo lprint add -d ElginL42Pro \
#   -v socket://192.168.15.90 \
#   -m zpl_4inch-203dpi-dt \
#   -o media-ready=na_index-4x6_4x6in

sudo lprint default -d ElginL42Pro
```

Ajuste `media-ready` para o tamanho real da sua etiqueta se não for 4x6 polegadas (veja
`lprint options -d ElginL42Pro` para ver os tamanhos suportados).

---

## 10. Testar

```bash
lprint status -d ElginL42Pro
```

Confirme que o status aparece como "Idle"/"Ocioso". O jeito mais simples de testar a
impressão é pelo botão **"Print Test Page"** no painel web (veja o próximo passo). Se
preferir por linha de comando, envie qualquer arquivo PNG ou ZPL que você tenha:

```bash
lprint submit -d ElginL42Pro caminho/para/etiqueta.png
```

Se uma etiqueta sair impressa, está tudo funcionando.

---

## 11. Acessar o painel (frontend) pelo Chrome

No Chrome, de **qualquer computador da mesma rede**, acesse:

```
http://IP-DO-SERVIDOR:8050
```

Nesse painel você consegue:

- Ver o status da impressora Elgin e da fila de trabalhos.
- Cancelar/reimprimir trabalhos.
- Ver o log do servidor.
- Ver/alterar mídia (tamanho de etiqueta) configurada.

**Para efetivamente imprimir documentos do Chrome** (páginas web, PDFs, etc. via `Ctrl+P`),
o computador cliente precisa "enxergar" a Elgin como uma impressora de rede — isso é feito
pelo protocolo IPP Everywhere que o LPrint já expõe, não pelo painel web:

- **Windows:** Configurações → Impressoras e scanners → Adicionar dispositivo. A impressora
  deve aparecer automaticamente (via Bonjour/mDNS) como "ElginL42Pro" — adicione e pronto,
  ela passa a existir no diálogo de impressão do Chrome.
- **Se não aparecer automaticamente:** adicione manualmente usando a URL
  `ipp://IP-DO-SERVIDOR:8050/ipp/print/ElginL42Pro`.
- **Chrome OS / Android / iOS / macOS:** detectam a impressora automaticamente por AirPrint,
  já que o LPrint implementa IPP Everywhere.

---

## 12. Adicionar senha ao painel (opcional, recomendado fora de rede confiável)

Edite `/etc/lprint.conf` (`sudo nano /etc/lprint.conf`) e troque a linha `server-options=`
para incluir `web-security` de volta, além de adicionar as duas linhas de autenticação:

```
server-options=multi-queue,web-interface,web-log,web-network,web-security
auth-service=login
admin-group=sudo
```

Depois reinicie o serviço:

```bash
sudo systemctl restart lprint.service
```

Isso usa o serviço PAM `login` (já existe por padrão em `/etc/pam.d/login`) e passa a exigir
um usuário Linux válido do grupo `sudo` para adicionar, modificar ou remover impressoras pelo
painel.

---

## 13. Solução de problemas

| Sintoma | Causa provável | Solução |
|---|---|---|
| `lprint add` trava/erro de conexão | IP errado ou impressora desligada | Reimprima o autoteste (Feed + Power) e confira o IP |
| Serviço não inicia | `avahi-daemon` não está rodando | `sudo systemctl status avahi-daemon` |
| Painel não abre no Chrome | Porta bloqueada no firewall | `sudo ufw allow 8050/tcp` |
| Etiqueta sai deslocada/cortada | Tamanho de mídia errado | Ajuste `media-ready` com `lprint modify` |
| Etiqueta sai em branco | Driver errado (`dt` vs `tt`) para o tipo de mídia usado | Troque o driver com `lprint modify -d ElginL42Pro -m zpl_4inch-203dpi-dt` (ou `-tt`) |
| `./configure` não acha o CUPS/PAPPL | Pacote `-dev` com nome diferente | Rode `pkg-config --list-all \| grep -i cups` para achar o nome certo |

---

## Referência rápida de comandos

```bash
lprint printers                 # lista filas cadastradas
lprint status -d ElginL42Pro    # status da impressora
lprint jobs -d ElginL42Pro      # trabalhos na fila
lprint submit -d ElginL42Pro arquivo.png   # imprime um arquivo
sudo systemctl restart lprint.service      # reinicia o servidor após mudar /etc/lprint.conf
```
