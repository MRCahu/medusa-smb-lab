# Laboratório controlado: auditoria de credenciais SMB com Medusa

> **Uso exclusivamente educacional e autorizado.** Este projeto foi desenhado para uma rede virtual isolada contendo apenas uma VM Kali Linux e uma VM Metasploitable 2. Não use os comandos contra endereços públicos, redes corporativas, contas reais ou sistemas sem autorização explícita. Não conecte a VM vulnerável à Internet ou à rede de produção.

## Objetivo

Demonstrar, em ambiente de treinamento, como credenciais fracas podem ser identificadas em um serviço SMB, como interpretar um resultado de ferramenta de auditoria e como reduzir o risco. O foco é um teste pequeno e delimitado — não uma campanha de força bruta.

## Topologia do laboratório

```text
[Kali Linux: cliente de auditoria]  ---- rede interna/host-only ----  [Metasploitable 2: alvo vulnerável]
          sem bridge / sem Internet                                      snapshot descartável
```

1. Em VirtualBox, VMware ou hypervisor equivalente, crie uma **rede interna ou host-only** e conecte ambas as VMs somente a ela. Evite modo *bridged* e desligue adaptadores extras da VM vulnerável.
2. Faça snapshot das VMs. Use uma cópia descartável do Metasploitable 2; ele é intencionalmente vulnerável.
3. Inicie as VMs e obtenha os endereços na própria rede isolada (`ip -br addr` no Kali; `ip addr` no alvo). Confirme que ambos pertencem à mesma sub-rede. Use o IP real do alvo no lugar de `IP_DO_ALVO` abaixo.
4. Confirme conectividade somente entre as VMs (`ping -c 2 IP_DO_ALVO`). Se a conectividade falhar, confira a configuração da rede virtual; não mude para bridge.
5. No alvo, confirme que o serviço SMB está ativo. A exposição do serviço deve permanecer restrita à rede de laboratório.

## Preparação no Kali

Verifique as ferramentas instaladas:

```bash
medusa -h
smbclient --version
```

Se estiver ausente, instale pelos repositórios oficiais da distribuição (`sudo apt update && sudo apt install medusa smbclient`). Não reutilize credenciais pessoais. O projeto traz listas didáticas pequenas em `wordlists/`.

## Execução didática (lista pequena)

Primeiro descubra se o serviço SMB responde, sem varrer outras máquinas:

```bash
nmap -Pn -p 139,445 IP_DO_ALVO
```

Em seguida execute uma única combinação de listas contra **o IP da VM Metasploitable 2**. O módulo `smbnt` é o módulo SMB do Medusa. `-t 2` limita a concorrência a duas tentativas simultâneas; `-T 1` limita o escopo a um host. Não aumente listas ou paralelismo neste exercício.

```bash
medusa -h IP_DO_ALVO -U wordlists/usuarios.txt -P wordlists/senhas-lab.txt -M smbnt -t 2 -T 1 -f -v 4
```

Opções principais: `-h` alvo único; `-U` arquivo de usuários; `-P` arquivo de senhas; `-M smbnt` módulo SMB; `-t 2` tentativas concorrentes limitadas; `-T 1` um host; `-f` interrompe após uma credencial válida; `-v 4` aumenta a informação apresentada. A saída pode variar conforme versão do Medusa e configuração do Samba.

As listas são deliberadamente curtas e contêm apenas pares fictícios para demonstração. O Metasploitable 2 costuma vir com a conta didática `msfadmin` / `msfadmin`; a lista de usuários inclui esse nome, mas a lista de senhas **não** inclui a senha padrão do alvo. Para verificar o caminho positivo sem ampliar a força bruta, acrescente conscientemente essa senha fictícia a uma cópia temporária da lista e execute apenas no laboratório:

```bash
printf '%s\n' 'msfadmin' >> /tmp/senhas-lab-local.txt
medusa -h IP_DO_ALVO -U wordlists/usuarios.txt -P /tmp/senhas-lab-local.txt -M smbnt -t 2 -T 1 -f -v 4
rm -f /tmp/senhas-lab-local.txt
```

Não interprete uma mensagem isolada como prova de acesso: falhas de rede, bloqueio, incompatibilidade de protocolo, conta inexistente ou política do servidor podem produzir resultados diferentes. Registre a versão da ferramenta e o contexto.

## Validação com smbclient

Use a credencial de laboratório encontrada (ou a conhecida do alvo didático) para listar compartilhamentos:

```bash
smbclient -L //IP_DO_ALVO -U msfadmin
```

Digite a senha no prompt; evite passá-la na linha de comando para não deixá-la no histórico/process list. Para verificar um compartilhamento que o servidor realmente anunciou:

```bash
smbclient //IP_DO_ALVO/NOME_DO_COMPARTILHAMENTO -U msfadmin
```

Dentro do cliente, use `ls` para listar e `exit` para sair. Não leia, copie, altere ou apague dados. Permissão negada ou ausência de compartilhamentos acessíveis não invalida o achado de credencial: autenticação e autorização são controles distintos.

## Interpretação e falsos positivos

- **Credencial encontrada:** confirme que o host é a VM esperada, que a combinação foi aceita pelo serviço SMB e que o teste ocorreu no intervalo autorizado. Valide com `smbclient`; não conclua que há acesso a arquivos apenas pelo resultado do Medusa.
- **Nenhuma credencial encontrada:** isso não prova que as senhas são fortes. Pode haver usuário errado, serviço indisponível, limitação de tentativas, protocolo incompatível ou lista incompleta.
- **Falso positivo/ambiguidade:** descarte mensagens de timeout, erro de conexão e respostas de protocolo; repita uma única validação manual, com a conta de laboratório, e correlacione com logs do servidor. Não conte mensagens de erro como credenciais válidas.
- **Paralelismo:** acelera tentativas, mas aumenta carga e ruído nos logs. Neste projeto o paralelismo fica limitado a 2; não é recomendação para testar sistemas reais.

## Discussão: relevância, IA e MFA

Ferramentas clássicas como Medusa continuam úteis para compreender autenticação de rede, auditoria autorizada e sinais registrados por controles defensivos. IA pode ajudar a priorizar riscos e resumir logs, mas não elimina credenciais reutilizadas, protocolos legados ou erros de configuração; também não substitui escopo, autorização e validação humana.

MFA reduz o impacto de uma senha comprometida, mas não é proteção absoluta: fatores podem ser contornados por phishing, roubo de sessão, fadiga de aprovação ou recuperação de conta fraca. Prefira MFA resistente a phishing (passkeys/FIDO2), bloqueio de autenticação SMB obsoleta quando possível, segmentação de rede e monitoramento. Não use força bruta em interfaces Web modernas: mecanismos de login variam, têm controles anti-automação, tokens e limites, e testar esses fluxos fora de escopo é arriscado. A auditoria Web deve usar metodologia e autorização específicas, preferencialmente ambiente de staging.

## Mitigações recomendadas

1. Remover contas e senhas padrão; usar senhas longas, únicas e armazenadas em gerenciador.
2. Aplicar política de bloqueio/limitação progressiva de tentativas e alertas para falhas repetidas, considerando risco de negação de serviço por bloqueio abusivo.
3. Restringir SMB por firewall e segmentação; expor portas 139/445 somente a redes e ativos que necessitam.
4. Desativar SMBv1 e protocolos legados; preferir SMB moderno com assinatura e criptografia conforme suporte e necessidade.
5. Habilitar MFA para acesso remoto e administrativo, com fator resistente a phishing quando disponível.
6. Monitorar logs de autenticação e revisar contas, privilégios e compartilhamentos; testar controles em janela autorizada.
7. Manter sistemas atualizados e aplicar princípio do menor privilégio.

## Evidências e relatório

Preencha `RELATORIO.md` com versões, endereços privados do laboratório, comando, resultado resumido, validação e mitigação. Não inclua senhas em capturas, commits ou relatório. A pasta `evidencias/` contém apenas instruções; adicione capturas locais com dados sensíveis ocultados se sua atividade exigir evidências visuais.

## Limpeza

Desligue a VM vulnerável, reverta o snapshot e remova arquivos temporários. Mantenha o adaptador da VM vulnerável desconectado quando não estiver em uso. Não publique capturas que revelem IPs, credenciais ou dados do ambiente.

## Checklist de conclusão

- [ ] Rede virtual isolada (sem bridge/Internet para o alvo)
- [ ] Snapshot criado e escopo documentado
- [ ] Serviço SMB confirmado no IP da VM-alvo
- [ ] Teste limitado a uma VM e listas pequenas
- [ ] Resultado interpretado e autenticado com `smbclient`
- [ ] Relatório preenchido sem segredos
- [ ] Mitigações discutidas e laboratório revertido


## Adequação ao desafio da DIO e estado da execução

A página oficial “Entendendo o Desafio” propõe cenários em FTP, formulário Web com DVWA e SMB, mas declara que o desafio é flexível e permite adaptar o cenário ou documentar detalhadamente o aprendizado. Este repositório escolhe o cenário SMB com Medusa e `smbclient`, alinhado ao foco deste projeto. O guia documenta o ambiente isolado, listas curtas, comandos, validação e mitigação.

**Estado:** este pacote é uma proposta reproduzível de laboratório; não foi executado contra uma VM nesta sessão, pois não havia Kali/Metasploitable disponíveis. `RELATORIO.md` é um modelo em branco, não evidência de execução. Preencha-o depois de realizar o teste em sua própria rede isolada. A página da DIO também exige assistir a todas as videoaulas; o player não disponibilizou o conteúdo na sessão, portanto este projeto não declara que elas foram assistidas. Consulte `TRILHA.md` para o mapa dos títulos visíveis e os requisitos oficiais lidos.

A DIO solicita um repositório público no GitHub com README detalhado e arquivos adicionais relevantes, e o envio do link acompanhado de breve descrição pelo botão “Entregar Projeto”. Este pacote ainda não foi publicado nem submetido. Não inclui capturas de tela ou resultados fictícios.
