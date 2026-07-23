# Configurar controles de permanência no grupo de colaboradores

Este guia explica como configurar o grupo
`example_group@dominio.com` para que:

- os colaboradores sejam adicionados como membros;
- cada mensagem aceita e distribuída pelo grupo seja solicitada como e-mail
  individual para o membro;
- a saída voluntária seja restringida e testada no ambiente da organização;
- usuários sem acesso ao Google Groups não consigam alterar a assinatura para
  **No email**.

O texto foi escrito para quem ainda não conhece bem o Google Workspace. Os nomes
dos menus são apresentados em inglês, como aparecem no Google Admin Console.

> **Importante:** faça primeiro em um grupo de teste equivalente e use uma conta
> de teste. Alterar `NONE_CAN_LEAVE` afeta o grupo inteiro; uma conta de teste
> dentro do grupo de produção não isola o risco. A documentação oficial do
> Google é contraditória sobre a garantia de impedir a auto-remoção de membros
> individuais. A configuração existe na Groups Settings API, mas precisa ser
> validada no ambiente da organização e complementada por reconciliação periódica.

## 1. Resultado esperado

Ao terminar:

1. O grupo continuará sendo
   `example_group@dominio.com`.
2. Novos colaboradores serão adicionados como `MEMBER`.
3. A preferência de entrega dos membros será configurada como `ALL_MAIL`.
4. O grupo terá `whoCanLeaveGroup` definido como `NONE_CAN_LEAVE`.
5. O teste no ambiente confirmará se o membro comum não consegue remover a
   própria associação pelos caminhos disponíveis.
6. Se o Google Groups estiver desativado para esses usuários, eles não terão a
   tela que permite mudar a assinatura para **No email**.

## 2. O que cada configuração significa

O Google trata associação, entrega e leitura como coisas diferentes:

| Configuração | O que controla |
|---|---|
| `MEMBER` | A pessoa pertence ao grupo. |
| `NONE_CAN_LEAVE` | Propriedade da API destinada a restringir quem pode sair; o comportamento precisa ser testado no tenant. |
| `ALL_MAIL` | Solicita o envio individual das mensagens aceitas e distribuídas pelo grupo; não garante caixa de entrada nem leitura. |
| **Groups for Business: OFF** | Os grupos internos da organização ficam indisponíveis para o usuário no aplicativo Google Groups; confirme sempre o `Service status`. |

Mesmo com tudo configurado, não é possível obrigar alguém a ler uma mensagem.
O usuário ainda pode apagar o e-mail ou criar um filtro no Gmail.

## 3. Limitação importante do Google

A documentação oficial atual não descreve uma configuração por grupo que impeça
o próprio membro de escolher **No email**.

Para `example_group@dominio.com`:

- impedir a saída é uma configuração específica do grupo;
- configurar `ALL_MAIL` é uma configuração específica de cada membro;
- retirar a tela de assinatura exige desativar **Groups for Business** para a
  unidade organizacional do usuário;
- desativar **Groups for Business** afeta o acesso pelo aplicativo aos grupos
  internos da organização, não apenas o grupo de colaboradores.

Um **access group** não pode ser usado para desligar o serviço. Ele só pode
ligá-lo para seus membros. Se a unidade organizacional estiver como **OFF**, mas
o usuário estiver em um access group que habilita o serviço, o acesso pode
continuar.

Se os colaboradores já não conseguem acessar os grupos internos em
`groups.google.com`, o serviço pode estar desativado para eles. Confirme pelo
**Service status** antes de mudar qualquer coisa.

### Termos usados neste guia

| Termo | Explicação simples |
|---|---|
| API | Forma controlada de um programa pedir uma ação ao Google. |
| APIs Explorer | Tela oficial do Google para testar uma chamada de API sem criar um programa completo. |
| JSON | Formato de texto usado para informar configurações à API. |
| Request body | Campo onde o JSON da alteração é colocado. |
| OAuth scope | Permissão específica concedida a um aplicativo. |
| Service Account | Identidade técnica usada pelo script, diferente de uma conta de pessoa. |
| Domain-Wide Delegation | Autorização para a Service Account agir em nome de um usuário do domínio; neste projeto, o usuário representado precisa dos privilégios administrativos exigidos pelas operações. |
| Client ID | Número que identifica a Service Account na delegação. |
| Organizational Unit | Unidade organizacional usada para aplicar políticas a um conjunto de usuários. |
| Access group | Grupo que pode ligar um serviço para seus membros, mesmo quando a unidade organizacional está como `OFF`. |
| Inherit | Herdar a configuração da unidade organizacional superior. |
| Override | Substituir a configuração herdada apenas naquele nível. |
| Propagation | Tempo necessário para uma alteração aparecer em todos os serviços do Google. |
| `groupUniqueId` ou `groupKey` | Identificador do grupo; neste guia, é o e-mail principal do grupo. |
| `memberKey` | Identificador do membro; neste guia, normalmente é o e-mail principal do usuário. |
| Admin SDK | Conjunto de APIs administrativas do Google Workspace. |
| `SCOPES` | Lista de permissões OAuth solicitadas pelo código. |
| `with_subject(...)` | Parte do código que escolhe o usuário do domínio em nome de quem a Service Account atuará. |
| `OWNER`, `MANAGER`, `MEMBER` | Papéis do grupo, do mais privilegiado ao membro comum. |
| Tenant | Organização Google Workspace que está sendo administrada. |

## 4. Antes de começar

Separe:

- uma conta administrativa autorizada;
- o endereço principal exato do grupo:
  `example_group@dominio.com`;
- um grupo de teste com configurações equivalentes;
- uma conta comum para teste;
- o nome da **Organizational Unit** dos colaboradores;
- os access groups que podem ligar o Google Groups para esses usuários;
- autorização interna para alterar a política do grupo.

Registre também, antes de qualquer mudança:

- o valor atual de `whoCanLeaveGroup`;
- o estado efetivo de **Groups for Business**: `ON` ou `OFF`;
- se a política está **Inherited** ou **Overridden**;
- a unidade organizacional ou o grupo de acesso selecionado;
- o estado de `delivery_settings` da conta de teste;
- data, horário e administrador responsável.

Também confirme que o grupo existe:

1. Entre em [Google Admin Console](https://admin.google.com).
2. Abra **Menu > Directory > Groups**.
3. Pesquise por `example_group@dominio.com`.
4. Abra o grupo.
5. Confirme que esse é o endereço principal, e não apenas um alias.

Não altere membros nem configurações enquanto estiver apenas conferindo. Aplique
e valide primeiro no grupo de teste; só depois repita no grupo de produção.

### Quem executa cada parte

| Parte | Identidade recomendada |
|---|---|
| Conferir e alterar o grupo | Administrador com privilégio para ler e gerenciar configurações e membros de grupos. |
| Preparar Domain-Wide Delegation | Super Admin, somente durante a preparação. |
| Habilitar APIs no Google Cloud | Administrador autorizado no projeto correto do Google Cloud. |
| Executar o projeto diariamente | `workspace-automation-executor` com papel personalizado que cubra somente criação de usuários e gerenciamento de membros. |
| Validar o comportamento | Conta comum em um grupo de teste equivalente. |

Evite usar Super Admin para executar o script diariamente.

## 5. Visão geral do procedimento

```text
Confirmar o grupo
        |
        v
Consultar a configuração atual
        |
        v
Definir NONE_CAN_LEAVE
        |
        v
Garantir ALL_MAIL nos membros
        |
        v
Verificar acesso ao Google Groups
        |
        v
Testar com uma conta comum
```

## 6. Impedir que os membros saiam do grupo

Essa alteração é feita pela **Groups Settings API**. Não existe uma caixa no
Admin Console onde o JSON é colado.

O Google documenta `NONE_CAN_LEAVE` na referência da API. Ao mesmo tempo, a FAQ
administrativa informa que não há garantia geral de impedir a auto-remoção de
um membro individual, com exceções como associação de toda a organização ou
grupos mantidos por sincronização. Por isso, este guia trata
`NONE_CAN_LEAVE` como um controle a ser testado, e não como garantia absoluta.

Para uma configuração feita apenas uma vez, o caminho mais simples é usar o
**APIs Explorer** com uma conta administrativa. Assim, não é necessário
adicionar um novo escopo permanentemente ao script principal.

### 6.1 Consultar a configuração atual

Antes de alterar, registre o valor atual:

1. Entre no navegador com a conta administrativa.
2. Abra a página oficial
   [Groups: get](https://developers.google.com/workspace/admin/groups-settings/v1/reference/groups/get).
3. Clique em **Try it!** ou **Try this method**.
4. No campo `groupUniqueId`, informe:

   ```text
   example_group@dominio.com
   ```

5. Abra **Show scopes**.
6. Mantenha apenas:

   ```text
   https://www.googleapis.com/auth/apps.groups.settings
   ```

   A Groups Settings API não oferece um escopo somente leitura separado para
   essa consulta.
7. Clique em **Execute**.
8. Autorize a solicitação, se o Google pedir.
9. Procure por `whoCanLeaveGroup` na resposta.
10. Salve o valor encontrado para poder reverter depois.

Use somente o valor retornado pela consulta. Os valores documentados incluem
`ALL_MEMBERS_CAN_LEAVE`, `ALL_MANAGERS_CAN_LEAVE` e `NONE_CAN_LEAVE`.

### 6.2 Aplicar `NONE_CAN_LEAVE`

1. Abra a página oficial
   [Groups: patch](https://developers.google.com/workspace/admin/groups-settings/v1/reference/groups/patch).
2. Clique em **Try it!** ou **Try this method**.
3. Em `groupUniqueId`, informe:

   ```text
   example_group@dominio.com
   ```

4. No corpo da solicitação, use:

   ```json
   {
     "whoCanLeaveGroup": "NONE_CAN_LEAVE"
   }
   ```

5. Abra **Show scopes** e mantenha somente:

   ```text
   https://www.googleapis.com/auth/apps.groups.settings
   ```

6. Clique em **Execute**.
7. Confirme a autorização solicitada.
8. Aguarde uma resposta de sucesso.
9. Confirme que a resposta contém:

   ```json
   {
     "whoCanLeaveGroup": "NONE_CAN_LEAVE"
   }
   ```

Essa propriedade é aplicada ao grupo inteiro. Um administrador com privilégios
para gerenciar grupos ainda poderá remover membros. O teste da seção 11 é
obrigatório para confirmar o efeito no ambiente da organização.

### 6.3 Confirmar que a alteração foi salva

Repita a consulta da etapa 6.1. O resultado precisa mostrar:

```json
{
  "whoCanLeaveGroup": "NONE_CAN_LEAVE"
}
```

Se outro valor aparecer, não continue para produção até resolver o problema.

## 7. Garantir que os membros recebam cada e-mail

Impedir a saída não configura automaticamente a entrega. Cada membro precisa
estar com:

```text
delivery_settings = ALL_MAIL
```

### 7.1 Para novos colaboradores adicionados pelo script

No arquivo `app.py`, na função `adicionar_membro`, localize o corpo usado por
`servico.members().insert(...)` e inclua:

```python
body={
    "email": email_usuario,
    "role": "MEMBER",
    "delivery_settings": "ALL_MAIL",
}
```

No projeto atual, isso pertence à função que adiciona o usuário ao grupo.

Este guia não altera o código automaticamente. A mudança precisa ser
implementada e testada antes de ser usada com contas reais.

O escopo já utilizado pelo projeto permite gerenciar a associação:

```text
https://www.googleapis.com/auth/admin.directory.group.member
```

### 7.2 Para um membro que já existe

Faça primeiro com a conta de teste:

1. Abra a referência
   [Members: get](https://developers.google.com/workspace/admin/directory/reference/rest/v1/members/get).
2. Clique em **Try it!**.
3. Abra **Show scopes** e mantenha somente:

   ```text
   https://www.googleapis.com/auth/admin.directory.group.member.readonly
   ```

4. Preencha:

   ```text
   groupKey: example_group@dominio.com
   memberKey: usuario_teste@dominio.com
   ```

5. Clique em **Execute**.
6. Anote `email`, `role` e `delivery_settings`.
7. Abra
   [Members: update](https://developers.google.com/workspace/admin/directory/reference/rest/v1/members/update).
8. Preencha o mesmo `groupKey` e `memberKey`.
9. No corpo, preserve exatamente o e-mail e o papel retornados pelo `GET`, e
   altere somente `delivery_settings`.

   ```json
   {
     "email": "usuario_teste@dominio.com",
     "role": "VALOR_EXATO_RETORNADO_PELO_GET",
     "delivery_settings": "ALL_MAIL"
   }
   ```

   Por exemplo, use `"MEMBER"` somente se o `GET` retornar `MEMBER`. Não troque
   `OWNER` ou `MANAGER` por `MEMBER`, pois isso rebaixaria o papel do usuário.

10. Abra **Show scopes** e mantenha somente:

   ```text
   https://www.googleapis.com/auth/admin.directory.group.member
   ```

11. Clique em **Execute**.
12. Consulte novamente com **Members: get**, usando o escopo somente leitura.
13. Confirme que `delivery_settings` retornou `ALL_MAIL`.

Depois do teste, o mesmo procedimento pode ser aplicado aos demais membros. Para
muitos usuários, é melhor automatizar e registrar o resultado de cada conta.

Não faça uma atualização em massa sem:

1. listar previamente os membros que serão afetados;
2. registrar o valor atual de cada associação;
3. testar com uma única conta;
4. limitar a primeira execução a um lote pequeno;
5. registrar sucessos e falhas por endereço;
6. ter uma forma de retomar apenas os itens que falharam.

Alterar todos para `ALL_MAIL` pode aumentar bastante o volume de mensagens
recebidas.

### 7.3 Atualizar vários membros com segurança

Para muitos membros, não repita o APIs Explorer manualmente sem controle. O
procedimento técnico deve:

1. usar `members.list` para obter os membros do grupo;
2. para cada membro, chamar `members.get`, porque `delivery_settings` não é
   suportado pelo método `list`;
3. salvar e-mail, ID, papel e `delivery_settings` anteriores em um registro de
   rollback;
4. ignorar `OWNER` e `MANAGER`, ou preservar exatamente seus papéis;
5. atualizar apenas contas planejadas para `ALL_MAIL`;
6. começar com um lote pequeno, por exemplo dez contas;
7. chamar `members.get` novamente para verificar cada resultado;
8. registrar falhas sem gravar credenciais ou senhas;
9. interromper o lote se aparecer um padrão de erro;
10. restaurar os valores anteriores a partir do registro, se a mudança precisar
    ser revertida.

Essa automação é uma mudança separada do guia de configuração e deve receber
testes próprios antes de operar no grupo real.

## 8. Verificar se os colaboradores acessam o Google Groups

Use uma conta comum de colaborador:

1. Entre em [Google Groups](https://groups.google.com).
2. Observe o resultado.

Existem dois cenários:

### Cenário A — o serviço está indisponível

Se o **Service status** no Admin Console estiver como `OFF` para a unidade
organizacional e nenhum access group religar o serviço, os grupos internos da
organização ficam indisponíveis no aplicativo Google Groups para esse usuário.
Ele não consegue abrir **My membership settings** para mudar a assinatura para
**No email**.

A mensagem exibida ao usuário pode variar; não dependa do texto literal
“Service not allowed”.

Não é necessário desativar novamente.

### Cenário B — o Google Groups abre normalmente

O usuário poderá abrir o grupo e acessar:

```text
colaboradores > My membership settings > Subscription
```

Nessa tela, o Google permite escolher **No email**. A documentação oficial atual
não descreve um bloqueio específico apenas para
`example_group@dominio.com`.

## 9. Desativar o Google Groups para os colaboradores

Execute esta etapa somente se os colaboradores não precisarem da interface web
do Google Groups para outros grupos internos da organização.

### 9.1 Entender o impacto

Ao desativar **Groups for Business**:

- os grupos existentes não são excluídos;
- mensagens continuam funcionando por e-mail;
- administradores continuam gerenciando grupos pelo Admin Console;
- os grupos internos da organização ficam temporariamente indisponíveis para os
  usuários afetados no aplicativo Google Groups;
- histórico e configurações deixam de ficar acessíveis pela interface, mas o
  histórico não é apagado e volta a ser acessível quando o serviço é reativado.

Essa ação não é específica do grupo de colaboradores.

### 9.2 Passo a passo no Admin Console

1. Entre em [Google Admin Console](https://admin.google.com).
2. Abra **Menu > Apps > Google Workspace > Groups for Business**.
3. Clique em **Service status**.
4. À esquerda, selecione a **Organizational Unit** correta.
5. Registre o estado efetivo atual, **ON** ou **OFF**.
6. Registre se a política está **Inherited** ou **Overridden**.
7. Confira se algum **access group** está ligando o serviço para esses usuários.
8. Confira quantos usuários serão afetados.
9. Selecione **OFF**.
10. Clique em **Override** ou **Save**, conforme aparecer na tela.
11. Se um access group ainda ligar o serviço, obtenha autorização antes de
    remover os colaboradores dele ou alterar essa política.
12. Aguarde a propagação da configuração.
13. Teste novamente `groups.google.com` com uma conta comum.

Não aplique **OFF for everyone** sem uma decisão administrativa explícita. Se
outros departamentos usam o Google Groups pela web, selecione somente a unidade
organizacional adequada e verifique os access groups que podem religar o
serviço.

## 10. Configuração opcional para automatizar pelo script

Esta seção é opcional. Use somente se a política do grupo precisar ser aplicada
ou verificada automaticamente pelo projeto.

### 10.1 Habilitar as APIs necessárias

No projeto correto do Google Cloud:

1. Entre em [Google Cloud Console](https://console.cloud.google.com).
2. Confirme o projeto no seletor superior.
3. Abra **APIs & Services > Library**.
4. Pesquise por **Groups Settings API**.
5. Abra o resultado e clique em **Enable**.
6. Volte para **APIs & Services > Library**.
7. Pesquise por **Admin SDK API**.
8. Confirme que ela também está como **Enabled**.

### 10.2 Autorizar o novo escopo

O escopo necessário é:

```text
https://www.googleapis.com/auth/apps.groups.settings
```

Para autorizar a Service Account:

1. Entre no Google Admin Console como **Super Admin**.
2. Abra
   **Menu > Security > Access and data control > API controls**.
3. Clique em **Manage Domain Wide Delegation**.
4. Localize o cliente pelo **Client ID numérico** da Service Account.
5. Clique em **Edit**.
6. Preserve os escopos existentes.
7. Acrescente:

   ```text
   https://www.googleapis.com/auth/apps.groups.settings
   ```

8. Salve.

Não informe o endereço de e-mail da Service Account no campo **Client ID**. O
Google exige o identificador numérico.

Os escopos totais do projeto atual ficariam:

```text
https://www.googleapis.com/auth/admin.directory.user
https://www.googleapis.com/auth/admin.directory.group.member
https://www.googleapis.com/auth/apps.groups.settings
```

O escopo `admin.directory.user` não é necessário para configurar o grupo; ele
continua na lista porque o projeto também cria contas de usuário. Para a
operação diária, `workspace-automation-executor` não deve usar Super Admin. Como o projeto cria
usuários e gerencia membros de grupos, use um papel personalizado com os
privilégios mínimos dessas duas operações, ou separe as identidades
administrativas por responsabilidade. O papel Groups Admin sozinho não cobre a
criação de usuários. A conta Super Admin é usada somente para preparar a
Domain-Wide Delegation.

No código, a Groups Settings API usa um cliente separado do Admin SDK Directory:

```python
servico_configuracoes = build(
    "groupssettings",
    "v1",
    credentials=credenciais_delegadas,
    cache_discovery=False,
)

resultado = (
    servico_configuracoes.groups()
    .patch(
        groupUniqueId=GROUP_EMAIL,
        body={"whoCanLeaveGroup": "NONE_CAN_LEAVE"},
    )
    .execute()
)
```

Antes de executar essa alteração automaticamente, o código deve consultar o
valor atual, registrar o resultado e evitar repetir mudanças desnecessárias.

### 10.3 Princípio de menor privilégio

Se a política será configurada apenas uma vez pelo APIs Explorer, não acrescente
o novo escopo ao script. Mantenha `apps.groups.settings` somente quando o projeto
realmente precisar consultar ou corrigir essa configuração.

### 10.4 Revogar o acesso do APIs Explorer depois da configuração

O APIs Explorer trabalha com dados reais e pode solicitar vários escopos da API.
Depois de concluir e verificar a configuração:

1. Abra [Google Account connections](https://myaccount.google.com/connections).
2. Localize **Google APIs Explorer**.
3. Revogue a conexão.
4. Se precisar usar a ferramenta novamente, autorize apenas os escopos
   necessários em **Show scopes**.

## 11. Teste completo com uma conta comum

Não use inicialmente uma conta importante. Utilize um usuário de teste.

### Teste 1 — associação

1. Adicione a conta ao grupo.
2. No Admin Console, abra
   **Menu > Directory > Groups > colaboradores > Members**.
3. Confirme que a conta aparece como `MEMBER`.

### Teste 2 — recebimento

1. Envie uma mensagem simples para:

   ```text
   example_group@dominio.com
   ```

2. Confirme que ela chegou à caixa de entrada da conta de teste.
3. Se o remetente receber uma rejeição, confira também quem tem permissão para
   postar no grupo; isso é diferente de um problema de entrega.
4. Se não chegar, verifique spam, filtros e o estado de entrega do membro.

### Teste 3 — tentativa de saída

1. Com a conta de teste, tente sair pela interface web, se ela estiver
   disponível.
2. Envie também uma mensagem para:

   ```text
   example_group+unsubscribe@dominio.com
   ```

3. Volte ao Admin Console.
4. Confirme se o usuário continua em:
   **Directory > Groups > colaboradores > Members**.

O Google pode enviar uma confirmação ou uma mensagem de erro para o comando
`+unsubscribe`. Registre o resultado real. Se a associação for removida, a
configuração não atende sozinha ao requisito da organização e será necessário usar
reconciliação automática.

### Teste 4 — segunda entrega

1. Envie uma segunda mensagem para o grupo.
2. Confirme que a conta ainda recebe o e-mail.
3. Consulte o membro pela API.
4. Confirme:

   ```json
   {
     "role": "MEMBER",
     "delivery_settings": "ALL_MAIL"
   }
   ```

Somente depois dos quatro testes avalie aplicar a configuração aos colaboradores
reais. Se qualquer caminho remover a conta, não declare o grupo como obrigatório
sem implantar uma rotina de reconciliação.

Mudanças de **Service status** e uma nova autorização de Domain-Wide Delegation
podem levar até 24 horas para se propagar, embora normalmente aconteçam antes.
Durante esse período, aguarde e repita o teste antes de diagnosticar uma falha.

## 12. Como reverter

### Permitir que membros saiam novamente

Use **Groups: patch** e restaure o valor registrado antes da alteração. Se o
valor original era `ALL_MEMBERS_CAN_LEAVE`, use:

```json
{
  "whoCanLeaveGroup": "ALL_MEMBERS_CAN_LEAVE"
}
```

### Reativar o Google Groups para os usuários

No Admin Console:

```text
Apps > Google Workspace > Groups for Business > Service status
```

Restaure o estado efetivo anterior, **ON** ou **OFF**. Se a política anterior era
herdada, use a ação **Inherit** para voltar à herança; se era substituída,
restaure o valor e o modo **Override** registrados.

### Reverter um membro específico

Não remova o membro apenas para testar a reversão. Se necessário, restaure o
`delivery_settings` registrado antes da mudança.

### Desfazer a automação opcional

Se a Groups Settings API não será mais usada pelo projeto:

1. remova `apps.groups.settings` da lista `SCOPES` do código;
2. remova as chamadas que dependem desse escopo;
3. implante e teste a nova versão sem esse acesso;
4. somente depois remova o escopo da entrada correspondente em
   **Manage Domain Wide Delegation**;
5. opcionalmente, desative a **Groups Settings API** no projeto do Google Cloud
   se nenhum outro sistema a utilizar.
6. revogue o acesso do **Google APIs Explorer**, caso ele tenha sido usado.

Se a unidade organizacional errada foi alterada, restaure imediatamente o estado
anterior nela e repita os testes com uma conta de cada unidade afetada.

## 13. Solução de problemas

### Erro `403: Forbidden`

Possíveis causas:

- conta autenticada sem privilégio para gerenciar o grupo;
- escopo OAuth não autorizado;
- Groups Settings API não habilitada;
- Service Account usando o e-mail no lugar do Client ID numérico;
- `with_subject(...)` apontando para uma conta errada, inativa ou sem o
  privilégio administrativo necessário;
- Client ID ou escopos não autorizados em Domain-Wide Delegation;
- domínio ou organização incorretos;
- política OAuth da organização bloqueando o APIs Explorer.

### Erro `404: Not Found`

Confirme:

- o endereço principal do grupo;
- a grafia de `example_group@dominio.com`;
- se a conta administrativa pertence ao mesmo Workspace;
- se foi usado um alias no lugar do endereço principal.

### O usuário permanece no grupo, mas não recebe mensagens

Verifique:

1. `delivery_settings` deve ser `ALL_MAIL`;
2. a conta não pode estar suspensa;
3. o endereço precisa estar correto;
4. o Gmail não pode estar rejeitando ou desviando a mensagem;
5. filtros e spam da conta;
6. **Reporting > Email Log Search** no Admin Console, se a edição for
   compatível e a conta administrativa tiver o privilégio **Gmail Settings**;
7. tempo de propagação após criar a conta ou mudar a associação.

### O usuário ainda acessa o Google Groups

Confirme:

- a unidade organizacional selecionada em **Service status**;
- se o estado efetivo está **ON** ou **OFF**;
- se a política está **Inherited** ou **Overridden**;
- se existe um grupo de acesso substituindo a configuração da unidade;
- se o teste foi feito com a conta corporativa correta.

### O usuário consegue selecionar “No email”

Isso significa que ele ainda tem acesso às configurações de assinatura do Google
Groups. A documentação oficial atual não descreve um bloqueio de **No email**
por grupo. As opções são:

1. desativar **Groups for Business** para aquele conjunto de usuários;
2. auditar e restaurar periodicamente `ALL_MAIL`;
3. usar outro processo para comunicações obrigatórias.

## 14. Operação recomendada

Para o ambiente da organização:

1. Aplique `NONE_CAN_LEAVE` após o piloto e audite periodicamente se o valor
   continua configurado.
2. Adicione novos colaboradores com `ALL_MAIL`.
3. Verifique se **Groups for Business** já está desativado para eles.
4. Não desative o serviço globalmente sem avaliar outros departamentos.
5. Faça uma auditoria periódica de associação e entrega.
6. Registre quem aplicou a política, quando e qual era o valor anterior.
7. Para comunicados críticos, use confirmação explícita de ciência.

### Reconciliação recomendada

Se o teste mostrar que algum caminho ainda permite sair ou selecionar
**No email**, a configuração não é suficiente sozinha. Implemente uma rotina de
reconciliação:

1. use o cadastro corporativo como fonte da lista esperada de colaboradores;
2. consulte os membros reais do grupo;
3. recoloque membros ausentes;
4. consulte individualmente `delivery_settings`;
5. restaure `ALL_MAIL` quando a política corporativa exigir;
6. confira se `whoCanLeaveGroup` continua com o valor aprovado;
7. registre data, responsável, correções e falhas;
8. envie alerta quando uma correção falhar;
9. execute diariamente e também após cadastro ou desligamento de colaboradores.

Reconciliação corrige divergências depois que elas acontecem; não elimina o
intervalo entre a saída e a próxima execução. Para comunicação crítica, mantenha
também um processo de confirmação de ciência.

## 15. Riscos que precisam ser avaliados

- **Unidade errada:** desativar Groups for Business na unidade incorreta pode
  retirar a interface web de usuários que dependem dela.
- **Volume de mensagens:** aplicar `ALL_MAIL` em massa pode gerar muitos e-mails.
- **Falha parcial:** uma atualização em lote pode funcionar para alguns membros
  e falhar para outros.
- **Membros externos:** usuários externos podem seguir políticas diferentes e
  precisam de teste separado.
- **Contas suspensas:** permanecem associadas, mas não recebem mensagens enquanto
  estiverem suspensas.
- **Aliases:** um alias não deve ser confundido com o endereço principal do
  usuário ou do grupo.
- **Privilégio excessivo:** manter escopos administrativos desnecessários aumenta
  o impacto de um vazamento de credenciais.
- **Ausência de garantia de leitura:** entrega não comprova que a mensagem foi
  aberta ou compreendida.
- **Mudança posterior:** se Groups for Business continuar ativo, o usuário ainda
  poderá alterar a assinatura, exigindo auditoria periódica.
- **Documentação contraditória:** `NONE_CAN_LEAVE` existe na API, mas a FAQ
  administrativa não promete bloqueio geral de auto-remoção individual. O teste
  no tenant e a reconciliação são necessários.

## 16. Checklist final

- [ ] O grupo foi confirmado pelo endereço principal.
- [ ] O valor anterior de `whoCanLeaveGroup` foi registrado.
- [ ] `whoCanLeaveGroup` está como `NONE_CAN_LEAVE`.
- [ ] A conta de teste está como `MEMBER`.
- [ ] A conta de teste está com `delivery_settings = ALL_MAIL`.
- [ ] Foi verificado se os usuários acessam `groups.google.com`.
- [ ] O impacto de **Groups for Business** foi avaliado.
- [ ] A conta de teste recebeu duas mensagens.
- [ ] A tentativa de saída não removeu a conta.
- [ ] Se a saída foi possível, existe uma rotina de reconciliação aprovada.
- [ ] O acesso concedido ao Google APIs Explorer foi revogado.
- [ ] O procedimento de reversão foi registrado.

## 17. Referências oficiais

- [Groups Settings API — propriedade `whoCanLeaveGroup`](https://developers.google.com/workspace/admin/groups-settings/v1/reference/groups)
- [Groups Settings API — consultar configurações](https://developers.google.com/workspace/admin/groups-settings/v1/reference/groups/get)
- [Groups Settings API — alterar configurações](https://developers.google.com/workspace/admin/groups-settings/v1/reference/groups/patch)
- [Directory API — propriedades de membros e `ALL_MAIL`](https://developers.google.com/workspace/admin/directory/reference/rest/v1/members)
- [Directory API — atualizar um membro](https://developers.google.com/workspace/admin/directory/reference/rest/v1/members/update)
- [Google Groups — opções de assinatura](https://support.google.com/groups/answer/2464926)
- [Google Workspace — limitar acesso e atividade em grupos](https://support.google.com/a/answer/167093)
- [Google Workspace — ativar ou desativar Groups for Business](https://support.google.com/a/answer/167096?hl=en)
- [Google Workspace — FAQ para administradores de grupos](https://knowledge.workspace.google.com/admin/groups/groups-administrator-faq)
- [Google APIs Explorer — autorização e escopos](https://developers.google.com/explorer-help/authorization-and-authentication)
- [Google OAuth — Domain-Wide Delegation](https://developers.google.com/identity/protocols/oauth2/service-account#delegatingauthority)
