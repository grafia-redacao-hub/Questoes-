# Auditoria técnica — Grafia Questões v1.0.1

## Escopo e método

Foi feita uma revisão estática dos arquivos HTML, CSS, JavaScript, manifesto PWA, Service Worker, importador, banco IndexedDB, utilitário ZIP e documentação do pacote v1.0.0 enviado. Também foram executadas verificações de sintaxe JavaScript, leitura/validação do manifesto e do modelo JSON, unicidade dos IDs HTML e testes unitários pontuais para mesclagem de metadados de imagens, normalização de texto e escrita/leitura ZIP.

Esta auditoria não equivale a uma bateria completa de testes de uso real. A tentativa de abrir a aplicação em Chromium headless não concluiu neste ambiente; a migração IndexedDB e os fluxos completos de importação/restauração ainda precisam ser exercitados em um navegador real, idealmente no mesmo Android em que o aplicativo será usado.

## Problemas identificados e corrigidos nesta versão

### 1. Indicador decorativo vazio em “Acompanhamento” — corrigido

O cabeçalho do cartão de desempenho continha um símbolo decorativo sem função. Ele foi removido do cartão na página inicial. Os símbolos das configurações foram preservados, pois são elementos diferentes e ainda têm conteúdo.

### 2. Respostas de atividades podiam ser lidas de uma cópia desatualizada — corrigido

As respostas eram gravadas no IndexedDB, mas a retomada e a finalização da atividade podiam consultar `state.data.answers`, uma cópia mantida para renderização. Ela nem sempre continha a gravação mais recente. O carregamento agora consulta o armazenamento persistente e atualiza o estado de tela; as alterações de resposta, confiança e marcação para revisão sincronizam as duas representações.

### 3. Migração de imagens antigas não era disparada corretamente — corrigido

A migração consultava `request.oldVersion`, mas a versão anterior pertence ao evento `upgradeneeded`. Em consequência, imagens guardadas junto dos metadados em versões antigas poderiam não ser movidas para o armazenamento binário separado. A migração agora usa `event.oldVersion` e a transação de atualização do IndexedDB. O banco foi elevado para a versão 3, preservando os registros existentes.

### 4. Backup incremental podia falhar quando o checkpoint era zero — corrigido

O código tratava `0` como se não houvesse backup completo anterior. Um backup válido feito antes de qualquer registro no diário podia, portanto, impedir o primeiro incremental. A verificação agora distingue checkpoint ausente de checkpoint igual a zero, tanto na criação como na aplicação de incrementais.

### 5. Restauração incremental podia deslocar a sequência do diário — corrigido

A reprodução de alterações importadas criava novas entradas no diário local, em vez de preservar o ponto de sequência da origem. Isso podia comprometer a comparação dos próximos incrementais. A restauração agora aplica os registros sem duplicar as alterações e grava um marcador de checkpoint na sequência de origem. Também bloqueia a aplicação quando há alterações locais posteriores ao último checkpoint, evitando mesclar silenciosamente duas linhas de histórico divergentes.

### 6. Atividades duplicadas podiam ser iniciadas em alguns fluxos secundários — mitigado

A criação de atividades a partir de revisão, coleção ou importação agora verifica se já existe uma atividade em andamento e solicita que ela seja retomada antes de criar outra. Os IDs recebidos também são deduplicados. Isso protege os fluxos tratados por essa função; não constitui um bloqueio entre duas abas abertas simultaneamente.

### 7. Texto-base com o mesmo ID podia ser sobrescrito durante importação — corrigido

A importação agora compara título e conteúdo dos textos-base existentes. Se o mesmo ID vier com conteúdo diferente, a importação é interrompida com mensagem explícita; se o conteúdo for equivalente, o registro já existente é preservado.

### 8. Associação de imagem por nome de arquivo podia escolher a imagem errada — corrigido parcialmente

Quando o caminho exato não existia no ZIP, o importador escolhia a primeira entrada com o mesmo nome final. ZIPs com arquivos homônimos em pastas diferentes podiam, assim, associar a imagem errada. Agora a associação por nome final ou prefixo de ID só ocorre quando há uma correspondência única. Em caso ambíguo, a imagem é apontada como ausente para revisão, em vez de ser selecionada silenciosamente.

### 9. Metadados de imagem podiam ser apagados por dados vazios — corrigido

Ao enriquecer uma questão, campos vazios da importação podiam substituir texto alternativo, legenda ou nome de arquivo já preenchidos no banco. A mesclagem agora preserva os valores existentes quando os novos campos estão vazios.

### 10. Filtro “não incluir questões resolvidas” podia excluir questões apenas parcialmente respondidas — corrigido

Antes, qualquer registro de resposta podia fazer uma questão ser tratada como resolvida, mesmo sem uma atividade concluída. O filtro agora considera apenas respostas completas, corrigidas e associadas a atividades concluídas.

### 11. Finalização duplicada e duração do cronômetro — corrigidos

Foi incluída uma trava contra duplo envio da mesma atividade. O tempo configurado no cronômetro também é limitado a um intervalo válido de 1 a 600 minutos, evitando valores negativos ou zero.

### 12. Atualização do PWA e documentação — corrigido

O nome do cache do Service Worker foi atualizado para `grafia-questoes-v1.0.1`, para que os arquivos novos não fiquem presos ao cache anterior. A documentação foi corrigida para manter o formato de pacote (`schemaVersion: "1.0.0"`) separado da versão do aplicativo (`1.0.1`).

## Pontos que ainda merecem correção ou teste dedicado

1. **Duplicatas dentro do mesmo pacote:** a detecção compara cada questão importada com o banco atual, mas ainda não consolida de forma completa duas questões semanticamente idênticas que apareçam no próprio arquivo recebido.
2. **Revisão de duplicata depois de editar uma questão importada:** a relação de duplicidade é calculada na análise inicial. Se o usuário alterar o enunciado ou as alternativas na tela de revisão, a relação pode ficar desatualizada. É recomendável reanalisar o pacote após alterações estruturais, ou recalcular as duplicatas no próprio formulário de edição.
3. **Arquivos extras no ZIP:** o importador valida imagens referenciadas, mas ainda não apresenta uma lista explícita de imagens extras que não foram referenciadas pelo JSON.
4. **Importação em lotes:** se uma falha acontecer no meio da gravação, alguns lotes podem já estar no banco. A retomada é baseada em IDs e deduplicação, não em um cursor formal de transação. Deve-se reanalisar o mesmo par JSON/ZIP e conferir a contagem antes de repetir operações manualmente.
5. **Duas abas simultâneas:** não existe coordenação de atividade em andamento entre abas. Duas abas do mesmo site podem iniciar fluxos concorrentes antes de uma delas atualizar o estado. O bloqueio adicionado cobre o estado visto por uma única instância do aplicativo.
6. **Falha de gravação durante o uso:** os caminhos de escrita usam transações IndexedDB, mas nem todos os eventos individuais exibem uma mensagem específica caso o navegador recuse a gravação. Um tratamento unificado de erros de armazenamento melhoraria o feedback ao usuário.
7. **Escala do diário de alterações:** o diário incremental cresce com o uso e algumas operações leem o histórico inteiro para calcular checkpoints. Em uso prolongado, vale implementar compactação segura do diário e consultas por cursor.
8. **Métrica da porcentagem:** o desempenho geral calcula acertos sobre respostas efetivamente dadas; a porcentagem individual de cada atividade calcula acertos sobre o total de questões, incluindo as em branco. Ambas podem ser métricas válidas, mas a diferença de denominador deveria ser explicitada na interface para evitar comparação equivocada.
9. **Instalação após renomear o repositório:** o manifesto usa caminhos relativos e o ID do PWA é derivado do novo caminho. Uma instalação antiga do caminho anterior pode continuar registrada no Chrome. A atualização do código não remove automaticamente uma instalação antiga; o fluxo de desinstalação/reinstalação deve ser testado depois da publicação.
10. **Conflito com Grafia Study:** este código usa o banco `grafia-questoes-db` e cache `grafia-questoes-*`. Isso reduz colisões com outros projetos, mas não foi possível comparar com o código atual do Study porque ele não foi enviado nesta conversa. A hipótese de conflito com o Study não está confirmada.

## Preservação dos dados existentes

A aplicação continua usando o nome de banco IndexedDB `grafia-questoes-db`. A atualização não chama a rotina de apagar o banco nem altera o identificador das questões. A versão do esquema foi elevada para 3 apenas para executar a migração de imagens antigas. Portanto, atualizar os arquivos no GitHub Pages não deve apagar o banco local existente. Não limpe os dados do site nem use a opção de apagar o banco para tentar resolver a instalação do Chrome.

O IndexedDB é associado à origem do site (domínio), não ao nome do repositório no caminho da URL. Como o caminho do repositório foi renomeado, é importante testar se o Chrome está abrindo a instalação nova ou um atalho antigo. O conteúdo local não é incluído no ZIP do código-fonte; os 40 registros exibidos na captura pertencem ao armazenamento local do navegador.

## Verificações executadas

- `node --check` nos arquivos JavaScript do aplicativo, importador, banco, ZIP, worker e Service Worker: aprovado.
- `manifest.webmanifest` e `MODELO_PACOTE.json`: JSON válido.
- Modelo de pacote: `schemaVersion: "1.0.0"` e lista `questions` vazia.
- IDs do HTML: 85 IDs verificados, sem duplicatas.
- Testes pontuais: mesclagem preservando metadados de imagens, normalização textual e escrita/leitura de ZIP: aprovados.
- ZIP original: integridade estrutural verificada sem erros.
- Teste de interface em navegador headless: não concluído por timeout do processo Chromium neste ambiente.
- Migração IndexedDB, importação/restauração completas, instalação PWA e uso em Android: ainda precisam de teste funcional real.

## Resultado

A versão 1.0.1 contém correções de integridade e sincronização importantes em relação ao pacote 1.0.0, além da remoção do elemento decorativo indicado na captura. Ela está sintaticamente validada e passou nos testes pontuais descritos acima, mas não deve ser considerada livre de bugs até que os fluxos reais de atividade, importação, backup, restauração e instalação sejam testados no Chrome Android.
