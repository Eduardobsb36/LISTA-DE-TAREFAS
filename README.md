
package com.example.listadetarefas

import android.os.Bundle
import android.content.Context
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.viewModels
import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Add
import androidx.compose.material.icons.filled.Delete
import androidx.compose.material.icons.filled.Edit
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.unit.dp
import androidx.lifecycle.ViewModel
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.viewModelScope
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.room.*
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch

// ======================================================
// BANCO DE DADOS ROOM
// ======================================================

@Entity(tableName = "tarefas")
data class Task(
    @PrimaryKey(autoGenerate = true)
    val id: Int = 0,
    val titulo: String,
    val descricao: String
)

@Dao
interface TaskDao {

    @Query("SELECT * FROM tarefas ORDER BY id DESC")
    fun listar(): Flow<List<Task>>

    @Insert
    suspend fun inserir(task: Task)

    @Update
    suspend fun atualizar(task: Task)

    @Delete
    suspend fun excluir(task: Task)
}

@Database(
    entities = [Task::class],
    version = 1,
    exportSchema = false
)
abstract class TaskDatabase : RoomDatabase() {

    abstract fun taskDao(): TaskDao

    companion object {

        @Volatile
        private var INSTANCE: TaskDatabase? = null

        fun getInstance(context: Context): TaskDatabase {

            return INSTANCE ?: synchronized(this) {

                val database = Room.databaseBuilder(
                    context.applicationContext,
                    TaskDatabase::class.java,
                    "lista_tarefas.db"
                ).build()

                INSTANCE = database
                database
            }
        }
    }
}

// ======================================================
// REPOSITÓRIO
// ======================================================

class TaskRepository(
    private val dao: TaskDao
) {

    val tarefas = dao.listar()

    suspend fun inserir(task: Task) {
        dao.inserir(task)
    }

    suspend fun atualizar(task: Task) {
        dao.atualizar(task)
    }

    suspend fun excluir(task: Task) {
        dao.excluir(task)
    }
}

// ======================================================
// VIEWMODEL - MVVM
// ======================================================

class TaskViewModel(
    private val repository: TaskRepository
) : ViewModel() {

    val tarefas = repository.tarefas.stateIn(
        viewModelScope,
        SharingStarted.WhileSubscribed(5000),
        emptyList()
    )

    fun salvar(
        id: Int,
        titulo: String,
        descricao: String
    ) {

        if (titulo.isBlank()) return

        viewModelScope.launch {

            val tarefa = Task(
                id = id,
                titulo = titulo.trim(),
                descricao = descricao.trim()
            )

            if (id == 0) {
                repository.inserir(tarefa)
            } else {
                repository.atualizar(tarefa)
            }
        }
    }

    fun excluir(task: Task) {

        viewModelScope.launch {
            repository.excluir(task)
        }
    }
}

// ======================================================
// VIEWMODEL FACTORY
// ======================================================

class TaskViewModelFactory(
    private val repository: TaskRepository
) : ViewModelProvider.Factory {

    @Suppress("UNCHECKED_CAST")
    override fun <T : ViewModel> create(
        modelClass: Class<T>
    ): T {

        if (modelClass.isAssignableFrom(TaskViewModel::class.java)) {
            return TaskViewModel(repository) as T
        }

        throw IllegalArgumentException("ViewModel não encontrado")
    }
}

// ======================================================
// MAIN ACTIVITY
// ======================================================

class MainActivity : ComponentActivity() {

    private val viewModel: TaskViewModel by viewModels {

        val database = TaskDatabase.getInstance(applicationContext)

        val repository = TaskRepository(database.taskDao())

        TaskViewModelFactory(repository)
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setContent {
            MaterialTheme(
                colorScheme = lightColorScheme(
                    primary = Color(0xFF2457A6),
                    secondary = Color(0xFF5479B8),
                    background = Color(0xFFF5F6FA)
                )
            ) {
                Aplicativo(viewModel)
            }
        }
    }
}

// ======================================================
// CONTROLE PRINCIPAL DO APLICATIVO
// ======================================================

@Composable
fun Aplicativo(viewModel: TaskViewModel) {

    val tarefas by viewModel.tarefas.collectAsStateWithLifecycle()

    var telaFormulario by remember {
        mutableStateOf(false)
    }

    var tarefaSelecionada by remember {
        mutableStateOf<Task?>(null)
    }

    if (telaFormulario) {

        TelaFormulario(
            tarefa = tarefaSelecionada,

            onSalvar = { id, titulo, descricao ->

                viewModel.salvar(id, titulo, descricao)

                telaFormulario = false
                tarefaSelecionada = null
            },

            onVoltar = {
                telaFormulario = false
                tarefaSelecionada = null
            }
        )

    } else {

        TelaLista(
            tarefas = tarefas,

            onAdicionar = {
                tarefaSelecionada = null
                telaFormulario = true
            },

            onEditar = { tarefa ->
                tarefaSelecionada = tarefa
                telaFormulario = true
            },

            onExcluir = { tarefa ->
                viewModel.excluir(tarefa)
            }
        )
    }
}

// ======================================================
// TELA DE LISTAGEM
// ======================================================

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun TelaLista(
    tarefas: List<Task>,
    onAdicionar: () -> Unit,
    onEditar: (Task) -> Unit,
    onExcluir: (Task) -> Unit
) {

    var tarefaExcluir by remember {
        mutableStateOf<Task?>(null)
    }

    if (tarefaExcluir != null) {

        AlertDialog(
            onDismissRequest = {
                tarefaExcluir = null
            },

            title = {
                Text("Excluir tarefa")
            },

            text = {
                Text("Tem certeza que deseja excluir esta tarefa?")
            },

            confirmButton = {
                TextButton(
                    onClick = {
                        tarefaExcluir?.let(onExcluir)
                        tarefaExcluir = null
                    }
                ) {
                    Text("Excluir")
                }
            },

            dismissButton = {
                TextButton(
                    onClick = {
                        tarefaExcluir = null
                    }
                ) {
                    Text("Cancelar")
                }
            }
        )
    }

    Scaffold(

        topBar = {
            TopAppBar(
                title = {
                    Text("Minhas Tarefas")
                }
            )
        },

        floatingActionButton = {
            FloatingActionButton(
                onClick = onAdicionar
            ) {
                Icon(
                    Icons.Default.Add,
                    contentDescription = "Adicionar"
                )
            }
        }

    ) { padding ->

        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(padding)
                .padding(16.dp)
        ) {

            Text(
                text = "Total de tarefas: ${tarefas.size}",
                style = MaterialTheme.typography.titleMedium
            )

            Spacer(modifier = Modifier.height(16.dp))

            if (tarefas.isEmpty()) {

                Box(
                    modifier = Modifier.fillMaxSize(),
                    contentAlignment = Alignment.Center
                ) {
                    Text(
                        "Nenhuma tarefa cadastrada.\nClique em + para adicionar."
                    )
                }

            } else {

                LazyColumn(
                    verticalArrangement = Arrangement.spacedBy(12.dp)
                ) {

                    items(
                        tarefas,
                        key = { it.id }
                    ) { tarefa ->

                        Card(
                            modifier = Modifier.fillMaxWidth(),
                            shape = RoundedCornerShape(14.dp),
                            elevation = CardDefaults.cardElevation(3.dp)
                        ) {

                            Row(
                                modifier = Modifier
                                    .fillMaxWidth()
                                    .padding(16.dp),

                                verticalAlignment = Alignment.CenterVertically
                            ) {

                                Column(
                                    modifier = Modifier
                                        .weight(1f)
                                        .clickable {
                                            onEditar(tarefa)
                                        }
                                ) {

                                    Text(
                                        tarefa.titulo,
                                        style = MaterialTheme.typography.titleMedium
                                    )

                                    Spacer(
                                        modifier = Modifier.height(6.dp)
                                    )

                                    Text(
                                        tarefa.descricao.ifBlank {
                                            "Sem descrição"
                                        }
                                    )
                                }

                                IconButton(
                                    onClick = {
                                        onEditar(tarefa)
                                    }
                                ) {
                                    Icon(
                                        Icons.Default.Edit,
                                        contentDescription = "Editar"
                                    )
                                }

                                IconButton(
                                    onClick = {
                                        tarefaExcluir = tarefa
                                    }
                                ) {
                                    Icon(
                                        Icons.Default.Delete,
                                        contentDescription = "Excluir",
                                        tint = MaterialTheme.colorScheme.error
                                    )
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}

// ======================================================
// TELA DE CADASTRO E EDIÇÃO
// ======================================================

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun TelaFormulario(
    tarefa: Task?,
    onSalvar: (Int, String, String) -> Unit,
    onVoltar: () -> Unit
) {

    var titulo by remember(tarefa?.id) {
        mutableStateOf(tarefa?.titulo ?: "")
    }

    var descricao by remember(tarefa?.id) {
        mutableStateOf(tarefa?.descricao ?: "")
    }

    var erro by remember {
        mutableStateOf(false)
    }

    Scaffold(

        topBar = {

            TopAppBar(
                title = {
                    Text(
                        if (tarefa == null) {
                            "Nova Tarefa"
                        } else {
                            "Editar Tarefa"
                        }
                    )
                },

                navigationIcon = {
                    TextButton(onClick = onVoltar) {
                        Text("Voltar")
                    }
                }
            )
        }

    ) { padding ->

        Column(

            modifier = Modifier
                .fillMaxSize()
                .padding(padding)
                .padding(20.dp),

            verticalArrangement = Arrangement.spacedBy(16.dp)

        ) {

            Text(
                "Informações da tarefa",
                style = MaterialTheme.typography.titleLarge
            )

            OutlinedTextField(

                value = titulo,

                onValueChange = {
                    titulo = it
                    erro = false
                },

                label = {
                    Text("Título")
                },

                placeholder = {
                    Text("Digite o título")
                },

                modifier = Modifier.fillMaxWidth(),

                singleLine = true,

                isError = erro
            )

            OutlinedTextField(

                value = descricao,

                onValueChange = {
                    descricao = it
                },

                label = {
                    Text("Descrição")
                },

                placeholder = {
                    Text("Digite a descrição")
                },

                modifier = Modifier
                    .fillMaxWidth()
                    .height(160.dp)
            )

            if (erro) {
                Text(
                    "O título é obrigatório.",
                    color = MaterialTheme.colorScheme.error
                )
            }

            Button(

                onClick = {

                    if (titulo.isBlank()) {
                        erro = true
                    } else {
                        onSalvar(
                            tarefa?.id ?: 0,
                            titulo,
                            descricao
                        )
                    }
                },

                modifier = Modifier.fillMaxWidth()

            ) {
                Text("Salvar Tarefa")
            }

            OutlinedButton(
                onClick = onVoltar,
                modifier = Modifier.fillMaxWidth()
            ) {
                Text("Cancelar")
            }
        }
    }
}
