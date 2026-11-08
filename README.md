# Projeto - DevOps com Vagrant e Ansible

Instituto Federal da Paraíba - Campus João Pessoa
Disciplina: Administração de Sistemas Abertos
Professor: Leonidas Lima
Período: 2026.2

Equipe:

- Samuel Santos Menezes - matrícula 20251380002
- Vinicius Ribeiro Maia - matrícula 20251380032

## Sobre o projeto

Neste projeto usamos o Vagrant para criar quatro máquinas virtuais Debian 12 (bookworm) no VirtualBox e o Ansible para configurar o sistema e os serviços de cada uma delas, sem nenhuma configuração manual. O próprio `vagrant up` cria cada VM e executa em seguida o playbook dela.

| VM  | Função                | Hostname                   | IP                |
|-----|-----------------------|----------------------------|-------------------|
| arq | DHCP, DNS, LVM e NFS  | arq.samuel.vinicius.devops | 192.168.56.102    |
| db  | MariaDB e autofs      | db.samuel.vinicius.devops  | 192.168.56.132    |
| app | Apache2 e autofs      | app.samuel.vinicius.devops | 192.168.56.122    |
| cli | Firefox, X11 e autofs | cli.samuel.vinicius.devops | DHCP (.50 a .100) |

A rede é a 192.168.56.0/24, com gateway 192.168.56.1 e domínio samuel.vinicius.devops.

## Pré-requisitos (máquina hospedeira)

- VirtualBox
- Vagrant
- Ansible

## Estrutura do repositório

```
.
├── Vagrantfile
├── ansible.cfg
├── hosts.ini
├── README.md
└── playbooks
    ├── comum.yml
    ├── autofs.yml
    ├── arq.yml
    ├── db.yml
    ├── app.yml
    ├── cli.yml
    └── arquivos
        ├── dhcpd.conf
        ├── named.conf.options
        ├── named.conf.local
        ├── db.samuel.vinicius.devops
        ├── db.192.168.56
        ├── index.html
        ├── samuel.pub
        └── vinicius.pub
```

## Vagrantfile

Todas as VMs usam o provider VirtualBox, a box debian/bookworm64, clones vinculados (linked_clone), 512 MB de RAM, geração de chaves SSH desativada e verificação de guest additions desabilitada. A única diferença é o cli, que tem 1024 MB.

O arq tem IP fixo e três discos adicionais de 10 GB. O db e o app pegam IP por DHCP com o MAC fixo. O cli pega IP por DHCP sem reserva.

Há um gatilho que roda antes do `vagrant up` e desabilita o servidor DHCP do VirtualBox, para que só o DHCP do arq responda na rede.

**Provisionamento.** Cada VM tem um provisionador `ansible` que executa o playbook dela (`playbooks/<vm>.yml`). O Vagrant gera o próprio inventário e conecta em cada VM pela interface NAT, então o provisionamento não depende de a VM já ter recebido IP na rede privada. O grupo `servidores`, usado pelos playbooks, é declarado no próprio Vagrantfile.

As VMs são criadas e provisionadas na ordem em que aparecem no Vagrantfile: arq, db, app e cli. O arq vem primeiro porque o DHCP e o DNS dele atendem as demais.

## Playbooks

Há um playbook por VM. As tarefas repetidas ficam em dois arquivos compartilhados, importados com `import_playbook`.

| Arquivo      | Alvo           | Conteúdo                                              |
|--------------|----------------|-------------------------------------------------------|
| comum.yml    | servidores     | Configuração básica de todas as máquinas              |
| autofs.yml   | db, app e cli  | Montagem automática do NFS do arq em /var/nfs         |
| arq.yml      | arq            | comum.yml + DHCP, DNS, LVM e NFS                      |
| db.yml       | db             | comum.yml + MariaDB + autofs.yml                      |
| app.yml      | app            | comum.yml + Apache2 e index.html + autofs.yml         |
| cli.yml      | cli            | comum.yml + Firefox, X11 + autofs.yml                 |

**comum.yml (todas as máquinas)**

- Update e upgrade do sistema.
- Chrony configurado para usar o pool.ntp.br e fuso America/Recife.
- Grupo ifpb e usuários samuel e vinicius dentro dele, com sudo liberado para o grupo.
- Chaves públicas dos dois usuários instaladas.
- SSH somente com chave pública, root bloqueado, acesso só para os grupos vagrant e ifpb e banner "Acesso apenas para pessoas com autorização expressa!!!".
- Cliente NFS instalado.

**arq.yml**

- DHCP autoritativo com domínio samuel.vinicius.devops, DNS apontando para o próprio arq, lease padrão de 180 s e máximo de 3600 s, faixa de .50 a .100, gateway 192.168.56.1 e reservas fixas para o db e o app.
- DNS autoritativo (bind9) aceitando só a rede 192.168.56.0/24, com forwarders 1.1.1.1 e 8.8.8.8 e zonas direta e reversa para arq, db e app.
- LVM com os três discos formando o volume group dados e o logical volume ifpb de 15 GB, em ext4, montado automaticamente em /dados.
- NFS exportando /dados/nfs para a rede 192.168.56.0/24. O usuário nfs-ifpb não tem shell e é o único com escrita no diretório. As opções all_squash, anonuid e anongid mapeiam todos os usuários remotos para ele, e o sync força a gravação imediata em disco.

**db.yml**

- Instalação do mariadb-server.

**app.yml**

- Instalação do Apache2 e troca da página padrão pelo index.html, que tem a descrição do projeto e o nome e a matrícula da equipe.

**cli.yml**

- Instalação do firefox-esr e do xauth e X11Forwarding habilitado no SSH.

**autofs.yml (db, app e cli)**

- autofs configurado para montar o /dados/nfs do arq em /var/nfs quando o diretório é acessado.

## Como executar

1. Na raiz do projeto, rodar `vagrant up`. Não use `--parallel`: o arq precisa estar pronto antes das outras VMs. O Vagrant sobe cada VM e já roda o playbook dela, na ordem arq, db, app e cli.
2. Conferir se o servidor DHCP do VirtualBox ficou desabilitado e, se não ficou, desabilitar à mão.
3. Conferir os IPs: o db deve ficar com .132 e o app com .122. O cli recebe um IP entre .50 e .100.
4. Para reaplicar a configuração depois que as VMs já existem, usar `vagrant provision` (ou `vagrant provision <vm>`). Rodando uma segunda vez, não deve aparecer nenhuma alteração.

### Execução manual (opcional)

O `hosts.ini` e o `ansible.cfg` permitem rodar os playbooks sem passar pelo Vagrant. O Ansible acessa as VMs com o usuário vagrant e a chave insecure_private_key do Vagrant. Use sempre `--limit`, porque o comum.yml mira o grupo `servidores` inteiro.

```
ansible-playbook playbooks/arq.yml --limit arq
ansible all -m ping
ansible-playbook playbooks/db.yml  --limit db
ansible-playbook playbooks/app.yml --limit app
ansible-playbook playbooks/cli.yml --limit cli
```

Nesse modo o db, o app e o cli precisam já ter recebido IP do DHCP do arq. Se o cli pegar um IP diferente de .50, ajuste o hosts.ini.

### Verificações sugeridas

- Resolução de nomes a partir das VMs: `vagrant ssh db -c "ping -c 1 arq.samuel.vinicius.devops"` (o mesmo vale para app e cli).
- Resolução a partir do hospedeiro: `dig @192.168.56.102 app.samuel.vinicius.devops`.
- NFS: `vagrant ssh app -c "ls /var/nfs"` (o autofs monta o diretório no primeiro acesso).
- Apache: abrir http://192.168.56.122 no navegador do hospedeiro.
