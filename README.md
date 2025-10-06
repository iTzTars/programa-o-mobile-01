import androidx.room.*

@Dao
interface ShoppingDao {
    @Query("SELECT * FROM shopping_lists WHERE ownerEmail = :userEmail ORDER BY title ASC")
    suspend fun getAllLists(userEmail: String): List<ShoppingList> // Mudança aqui!

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertList(list: ShoppingList)

    @Delete
    suspend fun deleteList(list: ShoppingList)

    @Query("SELECT * FROM shopping_items WHERE listId = :listId")
    suspend fun getItemsForList(listId: Long): List<ShoppingItem> // Mudança aqui!

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertItem(item: ShoppingItem)

    @Update
    suspend fun updateItem(item: ShoppingItem)

    @Delete
    suspend fun deleteItem(item: ShoppingItem)
}
import android.content.Context
import androidx.room.Room

object DatabaseProvider {
    private var INSTANCE: AppDatabase? = null

    fun getDatabase(context: Context): AppDatabase {
        return INSTANCE ?: synchronized(this) {
            val instance = Room.databaseBuilder(
                context.applicationContext,
                AppDatabase::class.java,
                "shopping_app_db"
            ).build()
            INSTANCE = instance
            instance
        }
    }
}
@Composable
fun LoginScreen(
    onLoginClick: (String) -> Unit, // Apenas passa o email para "logar"
    onNavigateToRegister: () -> Unit
) {
    var emailText by remember { mutableStateOf("") }
    var passwordText by remember { mutableStateOf("") }
    var errorMessage by remember { mutableStateOf<String?>(null) }

    Column(
        modifier = Modifier.fillMaxSize().padding(16.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        // ... Título, etc.

        OutlinedTextField(
            value = emailText,
            onValueChange = { emailText = it },
            label = { Text("Email") }
        )

        OutlinedTextField(
            value = passwordText,
            onValueChange = { passwordText = it },
            label = { Text("Senha") },
            visualTransformation = PasswordVisualTransformation()
        )
        
        // Exibe a mensagem de erro se houver
        errorMessage?.let {
            Text(it, color = MaterialTheme.colorScheme.error, modifier = Modifier.padding(top = 8.dp))
        }

        Spacer(modifier = Modifier.height(24.dp))

        Button(onClick = {
            // Lógica de validação direto aqui!
            if (emailText.isBlank() || passwordText.isBlank()) {
                errorMessage = "Email e senha não podem ser vazios."
            } else if (!android.util.Patterns.EMAIL_ADDRESS.matcher(emailText).matches()) {
                errorMessage = "O formato do email é inválido."
            } else {
                errorMessage = null
                // Sucesso! Chame a função de login.
                onLoginClick(emailText)
            }
        }) {
            Text("Entrar")
        }

        TextButton(onClick = onNavigateToRegister) {
            Text("Não tem uma conta? Cadastre-se")
        }
    }
}
import android.app.Application
import androidx.lifecycle.AndroidViewModel
import androidx.lifecycle.LiveData
import androidx.lifecycle.MutableLiveData
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.launch

// Usamos AndroidViewModel para pegar o contexto e instanciar o DB
class ListViewModel(application: Application) : AndroidViewModel(application) {

    private val dao: ShoppingDao
    
    // Simulação de usuário logado
    private val currentUserEmail = "user@example.com"

    // Usando LiveData
    private val _lists = MutableLiveData<List<ShoppingList>>()
    val lists: LiveData<List<ShoppingList>> = _lists
    
    // Estado para o texto da busca
    private val _searchQuery = MutableLiveData("")

    init {
        // Pega a instância do DAO
        dao = DatabaseProvider.getDatabase(application).shoppingDao()
        // Carrega as listas pela primeira vez
        loadLists()
    }

    fun loadLists() {
        viewModelScope.launch {
            _lists.value = dao.getAllLists(currentUserEmail)
        }
    }
    
    // Função para filtrar as listas na UI
    fun getFilteredLists(): List<ShoppingList> {
        val allLists = _lists.value ?: emptyList()
        val query = _searchQuery.value ?: ""

        return if (query.isBlank()) {
            allLists
        } else {
            allLists.filter { it.title.contains(query, ignoreCase = true) }
        }
    }

    fun onSearchQueryChange(query: String) {
        _searchQuery.value = query
    }

    fun addList(title: String) {
        viewModelScope.launch {
            dao.insertList(ShoppingList(title = title, ownerEmail = currentUserEmail))
            loadLists() // Recarrega a lista após adicionar!
        }
    }

    fun deleteList(list: ShoppingList) {
        viewModelScope.launch {
            dao.deleteList(list)
            loadLists() // Recarrega a lista após deletar!
        }
    }
}
import androidx.compose.runtime.livedata.observeAsState

@Composable
fun ListScreen(viewModel: ListViewModel = viewModel()) {
    // Observa o LiveData e reage às mudanças de busca
    val listsState by viewModel.lists.observeAsState(initial = emptyList())
    val filteredLists = viewModel.getFilteredLists()

    Scaffold(
        // ... TopBar com campo de busca que chama viewModel.onSearchQueryChange()
    ) { padding ->
        LazyColumn(modifier = Modifier.padding(padding)) {
            items(filteredLists) { list ->
                ListItem(
                    headlineContent = { Text(list.title) },
                    supportingContent = { Text("Clique para ver os itens") }
                )
            }
        }
    }
}
import androidx.compose.runtime.livedata.observeAsState

@Composable
fun ItemScreen(viewModel: ItemViewModel = viewModel()) {
    // Pega a lista "crua" do ViewModel
    val allItems by viewModel.items.observeAsState(initial = emptyList())

    LazyColumn(modifier = Modifier.fillMaxSize().padding(16.dp)) {
        
        // --- LÓGICA DE ORGANIZAÇÃO FEITA DIRETAMENTE AQUI ---
        val uncheckedItems = allItems.filter { !it.isChecked }
        val checkedItems = allItems.filter { it.isChecked }

        // Agrupa os itens não marcados por categoria e ordena
        val groupedUnchecked = uncheckedItems
            .groupBy { it.category }
            .toSortedMap()
            .mapValues { entry -> entry.value.sortedBy { it.name } }

        // --- Renderização dos Itens a Comprar ---
        item { Text("A comprar", style = MaterialTheme.typography.titleLarge) }
        
        groupedUnchecked.forEach { (category, itemsInCategory) ->
            item {
                Text(category, style = MaterialTheme.typography.titleMedium, modifier = Modifier.padding(top = 16.dp, bottom = 8.dp))
            }
            items(itemsInCategory) { item ->
                ItemRow(item = item, onCheckedChange = { viewModel.toggleItemChecked(item) })
            }
        }

        // --- Renderização dos Itens Comprados ---
        if (checkedItems.isNotEmpty()) {
            item {
                Divider(modifier = Modifier.padding(vertical = 16.dp))
                Text("Comprados", style = MaterialTheme.typography.titleLarge)
            }
            // Apenas ordena os itens marcados
            val sortedChecked = checkedItems.sortedBy { it.name }

            items(sortedChecked) { item ->
                ItemRow(item = item, onCheckedChange = { viewModel.toggleItemChecked(item) })
            }
        }
    }
}
