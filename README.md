# Lista de Tarefas (To-Do List)

Aplicativo Android de lista de tarefas construído em **Kotlin** com **Jetpack Compose**.
O projeto partiu de uma base fornecida em aula (camada de dados com Room já pronta) e foi
evoluído com a camada de apresentação, a integração via **Repository/ViewModel** e a
**navegação** entre as telas. Com o app é possível **listar, cadastrar, editar, concluir/reabrir
e excluir** tarefas, mantendo tudo salvo localmente no dispositivo.

Atividade individual — FIAP.

## Objetivo da aplicação

Oferecer um gerenciador simples de tarefas do dia a dia. Cada tarefa tem um título, uma
descrição opcional e um estado (concluída ou pendente). A lista é reativa: qualquer alteração
feita no banco é refletida na interface automaticamente, sem precisar recarregar a tela.

## Tecnologias utilizadas

- **Kotlin** — linguagem do projeto.
- **Jetpack Compose** — UI declarativa (telas, componentes e `@Preview`).
- **Room** — persistência local em SQLite (`Tarefa`, `TarefaDao`, `TarefaDatabase`).
- **Coroutines / Flow** — acesso assíncrono ao banco e observação reativa da lista.
- **ViewModel** — mantém o estado da UI e sobrevive a mudanças de configuração (ex.: rotação).
- **Navigation Compose** — troca entre a tela de lista e a tela de formulário.

## Arquitetura

O fluxo de dados segue a direção **UI → ViewModel → Repository → DAO → Room**, e o estado
volta em sentido contrário como um fluxo reativo:

```
        ações do usuário                       Flow<List<Tarefa>>
UI  ───────────────────────►  ViewModel  ──►  Repository  ──►  TarefaDao  ──►  Room (tarefas.db)
(Compose)  ◄───────────────  (StateFlow)  ◄──────────────────  observação reativa
```

Cada tela de Compose foi dividida em duas partes: um composable "de tela" que fala com a
`TarefaViewModel` e um composable "de conteúdo" que recebe apenas dados e callbacks. Isso
mantém a lógica separada da apresentação e permite gerar os `@Preview` sem depender da ViewModel.

### `TarefaRepository`

Arquivo: `repository/TarefaRepository.kt`.

É a camada que isola o acesso aos dados. Ela recebe um `TarefaDao` e expõe para o restante do
app uma interface enxuta:

- `tarefas: Flow<List<Tarefa>>` — todas as tarefas, ordenadas por data de criação (mais recentes primeiro).
- `inserir(tarefa)`, `atualizar(tarefa)` e `deletar(tarefa)` — operações de escrita, todas `suspend`.

O Repository não conhece Compose nem o ciclo de vida do Android. Assim, se um dia a origem dos
dados mudar (por exemplo, uma API remota), só essa camada precisaria ser alterada.

### `TarefaViewModel`

Arquivo: `viewmodel/TarefaViewModel.kt`.

Conecta o Repository à interface e guarda o estado observável da tela:

- Converte o `Flow` do Repository em um `StateFlow<List<Tarefa>>` usando `stateIn(...)` com
  `SharingStarted.WhileSubscribed(5_000)` — a coleta é ligada quando há tela ativa e desligada
  pouco depois de a tela sair, economizando recursos.
- Expõe `inserir`, `atualizar` e `deletar`, que rodam dentro de `viewModelScope.launch` para não
  travar a thread principal.
- Traz uma `factory` (no `companion object`) que monta a dependência: cria/recupera o
  `TarefaDatabase`, pega o `TarefaDao` e entrega um `TarefaRepository` pronto para a ViewModel.
  Com isso a `MainActivity` não precisa saber como o banco é construído.

### `ListaTarefasScreen`

Arquivo: `ui/ListaTarefasScreen.kt`.

Tela inicial. Observa o estado com `collectAsStateWithLifecycle()` — quando a lista muda no
banco, a tela recompõe sozinha. Ela repassa os dados para `ListaTarefasContent`, que:

- Mostra as tarefas em uma `LazyColumn`, cada uma em um `Card`.
- Traz um `Checkbox` para concluir/reabrir a tarefa (título fica riscado com `LineThrough` quando concluída).
- Permite excluir pelo ícone de lixeira e editar ao tocar no card (envia o `id` da tarefa).
- Tem um `FloatingActionButton` que abre o formulário de nova tarefa.
- Exibe uma mensagem quando ainda não há nenhuma tarefa.

As ações do usuário sobem como callbacks até `ListaTarefasScreen`, que as traduz em chamadas à
`TarefaViewModel`. Há `@Preview` para a lista com itens, a lista vazia e o item concluído.

### `FormularioTarefaScreen`

Arquivo: `ui/FormularioTarefaScreen.kt`.

Atende **cadastro e edição na mesma tela**, decidindo o modo pelo parâmetro `tarefaId`:

- `tarefaId == 0` → **cadastro**: campos vazios; ao salvar chama `viewModel.inserir(...)`.
- `tarefaId != 0` → **edição**: localiza a tarefa na lista observada (`tarefas.find { it.id == tarefaId }`),
  pré-preenche título e descrição e, ao salvar, chama `viewModel.atualizar(...)` preservando o `id`.

O botão "Salvar" só fica habilitado quando há título. Ao salvar ou tocar em voltar na `TopAppBar`,
a tela retorna para a lista. Há `@Preview` para os modos de cadastro e de edição.

### `AppNavigation`

Arquivo: `navigation/AppNavigation.kt`.

Define o grafo de navegação com `NavHost` e duas rotas:

| Rota | Tela | Descrição |
|------|------|-----------|
| `lista` (inicial) | `ListaTarefasScreen` | Abre `formulario/0` para nova tarefa ou `formulario/{id}` para editar |
| `formulario/{tarefaId}` | `FormularioTarefaScreen` | Lê o argumento `tarefaId` da rota; `0` = cadastro, demais valores = edição |

O **ID é passado pela própria rota** (`formulario/$id`). Na tela de formulário o argumento é lido
de `backStackEntry.arguments` e convertido para `Int`, e é ele que faz a tela decidir entre criar
ou editar. Voltar é feito com `navController.popBackStack()`, sem encerrar o app.

### `MainActivity`

Arquivo: `MainActivity.kt`.

É o ponto de entrada. Dentro de `setContent`, envolvido pelo tema `TodolistTheme`:

1. Cria a `TarefaViewModel` com `viewModel(factory = TarefaViewModel.factory(applicationContext))`,
   garantindo que ela receba o Repository já ligado ao Room.
2. Chama `AppNavigation(viewModel = viewModel)` como conteúdo raiz — o template de exemplo do
   Android Studio deixa de ser a tela principal.

## Como executar

1. Abrir a pasta do projeto no **Android Studio** (compatível com o AGP e o Kotlin definidos em `gradle/libs.versions.toml`).
2. Aguardar o **Gradle Sync** baixar as dependências (Room, Navigation Compose, Compose BOM, etc.).
3. Selecionar um emulador ou dispositivo físico com **API 24 (Android 7.0) ou superior**.
4. Executar com **Run 'app' (▶)**. Na primeira abertura a lista aparece vazia; use o botão **+**
   para cadastrar a primeira tarefa.

> Observação: `local.properties` não é versionado (ele guarda o caminho do SDK da sua máquina) —
> o Android Studio o recria automaticamente ao abrir o projeto.

## Funcionalidades

- Listar tarefas (lista reativa via `Flow`/`StateFlow`).
- Cadastrar nova tarefa.
- Editar tarefa existente.
- Marcar/desmarcar como concluída.
- Excluir tarefa.
- Navegar entre a lista e o formulário sem fechar o app.
- Persistência local com Room.

## Evidências

As imagens de execução do aplicativo estão na pasta [`docs/evidencias`](docs/evidencias).

### Tela inicial com a lista de tarefas
![Tela inicial](docs/evidencias/01-lista-inicial.png)

### Cadastro de uma nova tarefa
![Cadastro](docs/evidencias/02-cadastro.png)

### Tarefa cadastrada aparecendo na lista
![Lista com tarefa](docs/evidencias/03-lista-com-tarefa.png)

### Edição de uma tarefa existente
![Edição](docs/evidencias/04-edicao.png)

### Tarefa marcada como concluída
![Concluída](docs/evidencias/05-concluida.png)

### Exclusão de uma tarefa
![Exclusão](docs/evidencias/06-exclusao.png)

### Navegação entre a lista e o formulário
![Navegação](docs/evidencias/07-navegacao.png)

### Build/execução do projeto sem erros
![Build](docs/evidencias/08-build.png)
