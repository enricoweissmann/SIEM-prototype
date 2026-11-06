🛡️ SIEM Caseiro com Wazuh no Bazzite

Um laboratório de monitoramento de segurança (SIEM) rodando em um desktop com Bazzite (Linux imutável), usando Wazuh em containers Podman. Pensado para começar em uma rede doméstica e servir de base de estudo para redes maiores.

💡 A ideia do projeto

Quase todo tutorial de SIEM assume Ubuntu ou uma VM e instala tudo direto no sistema. Este projeto parte de outra pergunta:

É possível montar um SIEM completo em um sistema operacional imutável (Bazzite), sem instalar nada na base do sistema, usando só containers?

A resposta é sim, mas com várias armadilhas específicas (SELinux, containers sem privilégio de administrador, registros de imagens, ferramentas de compose). Este repositório documenta o caminho que realmente funcionou, incluindo os erros que apareceram pelo caminho e como corrigi-los, para que você não precise descobrir tudo sozinho.

Objetivos:

🏠 Monitorar uma rede doméstica: computadores, servidores, roteador e tráfego de rede.
📚 Aprender conceitos de SOC / blue team: coleta de logs, alertas, correlação, MITRE ATT&CK.
🧱 Servir de base para escalar para redes maiores (cluster, relays de syslog, grupos de agentes).
📝 Deixar tudo documentado e reproduzível.

🧩 O que é o Wazuh?

Wazuh é uma plataforma open source de SIEM/XDR. Quatro peças trabalham juntas:

Componente	Função
Agent	Instalado nos computadores monitorados. Coleta logs e verifica integridade de arquivos, vulnerabilidades e configuração.
Manager (server)	Recebe os eventos, aplica regras e gera alertas.
Indexer	Banco de dados (baseado em OpenSearch) onde os alertas ficam guardados.
Dashboard	Painel web para buscar e visualizar alertas.

 Status do projeto

✅ Preparar o Bazzite (parâmetro de memória, firewall)
✅ Subir Wazuh single-node em containers rootless com Podman
✅ Acessar o dashboard em https://localhost:8443
❌ Conectar o primeiro agente (Windows/Linux)
❌ Trocar as senhas padrão
❌ Receber syslog do roteador/firewall
❌ Integrar IDS de rede (Suricata)
❌ Criar regras e alertas personalizados
❌ Documentar a escalada para redes grandes (cluster)

Este quadro é atualizado conforme o laboratório avança.

🖥️ Ambiente testado
Item	              | Valor
Sistema	            | Bazzite (Fedora Atomic)
Motor de containers |	Podman (rootless)
Compose	            | podman-compose (instalado via pip --user)
Wazuh	              | 4.14.6 (stack single-node do wazuh-docker)

Requisitos mínimos do host segundo a documentação oficial: 4 núcleos, 8 GB de RAM, 50 GB de disco.

🚀 Instalação rápida (caminho que funcionou)

Rode os comandos um bloco por vez, no terminal do Bazzite.

1. Ajustar a memória do host (obrigatório)
bash
echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-wazuh.conf
sudo sysctl --system

2. Instalar o podman-compose
   
bash
pip install --user podman-compose
export PATH="$HOME/.local/bin:$PATH"
podman-compose --version

3. Baixar o projeto do Wazuh
 
bash
cd ~
git clone https://github.com/wazuh/wazuh-docker.git -b v4.14.6
cd wazuh-docker/single-node

4. Liberar a pasta para os containers (SELinux)
bash
chcon -R -t container_file_t .

5. Garantir que as imagens venham do docker.io
   
bash
mkdir -p ~/.config/containers/registries.conf.d
cat > ~/.config/containers/registries.conf.d/wazuh.conf << 'EOF'
[aliases]
"wazuh/wazuh-manager"="docker.io/wazuh/wazuh-manager"
"wazuh/wazuh-indexer"="docker.io/wazuh/wazuh-indexer"
"wazuh/wazuh-dashboard"="docker.io/wazuh/wazuh-dashboard"
EOF

6. Gerar os certificados

bash
podman-compose -f generate-indexer-certs.yml run --rm generator
ls config/wazuh_indexer_ssl_certs/

Devem aparecer vários arquivos .pem.

7. Trocar as portas para funcionar sem privilégio de administrador

Containers rootless não abrem portas abaixo de 1024, então movemos o dashboard de 443 para 8443 e o syslog de 514 para 5514:

bash
sed -i -E 's/^(\s*-\s*"?)443:5601/\18443:5601/' docker-compose.yml
sed -i -E 's#^(\s*-\s*"?)514:514/udp#\15514:514/udp#' docker-compose.yml
grep -n -E '8443|5514' docker-compose.yml
podman-compose config > /dev/null && echo "ARQUIVO OK"

8. Subir a stack
bash
podman-compose up -d
podman ps

Aguarde cerca de 2 minutos e abra https://localhost:8443 (aceite o aviso do certificado autoassinado).

Em versões 4.x, o login padrão costuma ser admin / SecretPassword. Confirme em docker-compose.yml e troque imediatamente.

9. Abrir o firewall para os agentes
    
bash
sudo firewall-cmd --permanent --add-port=1514/tcp
sudo firewall-cmd --permanent --add-port=1515/tcp
sudo firewall-cmd --permanent --add-port=5514/udp
sudo firewall-cmd --permanent --add-port=8443/tcp
sudo firewall-cmd --reload

🧯 Problemas encontrados (e como foram resolvidos)

Sintoma	Causa	Solução
podman compose falha com dial unix ... podman.sock: no such file	O provider docker-compose precisa do socket do Podman, que não estava no caminho esperado	Usar o podman-compose (com hífen), que não depende de socket
cp: cannot stat '/config/certs.yml': Permission denied	SELinux bloqueando o acesso da pasta pelo container	chcon -R -t container_file_t . dentro de single-node
Menu "Please select an image" e erro Repo not found	Imagem escolhida em registro que não tem o Wazuh (redhat/fedora)	Usar sempre docker.io/wazuh/... (alias do passo 5)
yaml.parser.ParserError ou expected at least one soft or hard limit	docker-compose.yml corrompido por edição manual (indentação quebrada) e porta virando 88443	Restaurar com git checkout -- docker-compose.yml e trocar as portas com o sed do passo 7
Connection refused ao abrir o painel	Containers não estavam rodando	podman ps -a e podman logs NOME para ver o motivo

💡 Lição principal: não edite o docker-compose.yml à mão em editor de terminal. Espaços no começo das linhas fazem parte da sintaxe do YAML, e um desalinhamento quebra tudo. Prefira sed e valide com podman-compose config.

📂 Estrutura do repositório
.
├── README.md            # esta página: a ideia e o caminho rápido
├── GUIA-COMPLETO.md     # guia detalhado: agentes, syslog, Suricata, redes grandes, referências
└── LICENSE


🔐 Segurança e ética
Monitore somente redes e dispositivos seus ou com autorização por escrito.
Logs contêm dados pessoais (usuários, IPs, arquivos). Em ambiente de empresa, a LGPD pode se aplicar.
Nunca publique senhas, certificados ou dados reais neste repositório.

Troque as senhas padrão antes de expor o painel além do seu computador.

📚 Referências principais
Wazuh: Quickstart
Wazuh: Implantação em Docker
Repositório wazuh-docker
Bazzite: Containers
Canal oficial do Wazuh no YouTube
podman-compose

🤝 Contribuindo

Encontrou um erro ou um jeito melhor de rodar isso no Bazzite? Abra uma issue ou um pull request. Relatos de outros hardwares e versões do Bazzite são muito bem-vindos.

📄 Licença

Documentação sob licença MIT. Wazuh é marca de Wazuh, Inc. e Bazzite é um projeto da comunidade Universal Blue; este repositório não tem afiliação oficial com nenhum deles.

Feito por wico: laboratório de estudos em segurança defensiva.
