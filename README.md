# Projeto - DevOps com Vagrant e Ansible

Instituto Federal da Paraíba - Campus João Pessoa
Disciplina: Administração de Sistemas Abertos
Professor: Leonidas Lima
Período: 2026.2

Equipe:
- Samuel Santos Menezes - matrícula 20251380002
- Vinicius Ribeiro Maia - matrícula 20251380032

## Sobre o projeto

Neste projeto usamos o Vagrant para criar quatro máquinas virtuais Debian 12 (bookworm) no VirtualBox e o Ansible para configurar o sistema e os serviços de cada uma delas, sem nenhuma configuração manual.

| VM  | Função                              | Hostname                   | IP             |
|-----|-------------------------------------|----------------------------|----------------|
| arq | DHCP, DNS, LVM e NFS                | arq.samuel.vinicius.devops | 192.168.56.102 |
| db  | MariaDB e autofs                    | db.samuel.vinicius.devops  | 192.168.56.132 |
| app | Apache2 e autofs                    | app.samuel.vinicius.devops | 192.168.56.122 |
| cli | Firefox, X11 e autofs               | cli.samuel.vinicius.devops | DHCP (.50 a .100) |

A rede é a 192.168.56.0/24, com gateway 192.168.56.1 e domínio samuel.vinicius.devops.

Os IPs seguem a regra do enunciado: o arq usa o final 02 da matrícula do Samuel, o db usa o final 32 da matrícula do Vinicius e o app usa o último dígito de cada matrícula (2 e 2). O db e o app recebem o IP por DHCP, com reserva pelo endereço MAC definido no Vagrantfile.

## Estrutura do repositório

```
.
├── Vagrantfile
├── ansible.cfg
├── hosts.ini
├── README.md
└── playbooks
    ├── main.yml
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

As chaves privadas dos usuários ficam só na máquina local e não são enviadas ao repositório.

## Vagrantfile

Todas as VMs usam o provider VirtualBox, a box debian/bookworm64, clones vinculados (linked_clone), 512 MB de RAM, geração de chaves SSH desativada e verificação de guest additions desabilitada. A única diferença é o cli, que tem 1024 MB.

O arq tem IP fixo e três discos adicionais de 10 GB. O db e o app pegam IP por DHCP com o MAC fixo. O cli pega IP por DHCP sem reserva.

Também há um gatilho que roda antes do vagrant up e desabilita o servidor DHCP do VirtualBox, para que só o DHCP do arq responda na rede.

## Playbook

O playbooks/main.yml tem um play para cada função.

**Todas as máquinas**
- Update e upgrade do sistema.
- Chrony configurado para usar o pool.ntp.br e fuso America/Recife.
- Grupo ifpb e usuários samuel e vinicius dentro dele, com sudo liberado para o grupo.
- Chaves públicas dos dois usuários instaladas.
- SSH somente com chave pública, root bloqueado, acesso só para os grupos vagrant e ifpb e banner "Acesso apenas para pessoas com autorização expressa!!!".
- Cliente NFS instalado.

**arq**
- DHCP autoritativo com domínio samuel.vinicius.devops, DNS apontando para o próprio arq, lease padrão de 180 s e máximo de 3600 s, faixa de .50 a .100, gateway 192.168.56.1 e reservas fixas para o db e o app.
- DNS autoritativo (bind9) aceitando só a rede 192.168.56.0/24, com forwarders 1.1.1.1 e 8.8.8.8 e zonas direta e reversa para arq, db e app.
- LVM com os três discos formando o volume group dados e o logical volume ifpb de 15 GB, em ext4, montado automaticamente em /dados.
- NFS exportando /dados/nfs para a rede 192.168.56.0/24. O usuário nfs-ifpb não tem shell e é o único com escrita no diretório. As opções all_squash, anonuid e anongid mapeiam todos os usuários remotos para ele, e o sync força a gravação imediata em disco.

**db**
- Instalação do mariadb-server.

**app**
- Instalação do Apache2 e troca da página padrão pelo index.html, que tem a descrição do projeto e o nome e a matrícula da equipe.

**cli**
- Instalação do firefox-esr e do xauth e X11Forwarding habilitado no SSH.

**db, app e cli**
- autofs configurado para montar o /dados/nfs do arq em /var/nfs quando o diretório é acessado.

## Como executar

A ordem importa. Como o DHCP do VirtualBox fica desligado, o db, o app e o cli só recebem IP na rede privada depois que o DHCP do arq estiver funcionando.

1. Subir as VMs com o vagrant up.
2. Conferir se o servidor DHCP do VirtualBox ficou desabilitado e, se não ficou, desabilitar a mão.
3. Rodar o playbook só no arq (--limit arq).
4. Esperar uns 3 minutos para o db, o app e o cli pegarem IP do DHCP do arq. O db deve ficar com .132 e o app com .122. Se o cli pegar um IP diferente de .50, é só ajustar no hosts.ini.
5. Testar a conexão com todas as máquinas usando o ansible all -m ping.
6. Rodar o playbook completo. Rodando uma segunda vez, não deve aparecer nenhuma alteração.

O Ansible acessa as VMs com o usuário vagrant e a chave insecure_private_key do Vagrant, configurados no hosts.ini.


