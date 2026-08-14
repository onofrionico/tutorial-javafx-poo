# Tutorial: Interfaces gráficas con JavaFX

Guía de acompañamiento de la clase. Cubre los fundamentos de JavaFX, el patrón MVC aplicado a una migración real de Swing, y Scene Builder.

---

## Bloque 1 — Fundamentos sin Scene Builder

La idea de este bloque es armar una ventana simple **100% a mano en código**, sin herramientas visuales. Esto obliga a entender qué es cada pieza antes de que Scene Builder oculte la lógica detrás de un drag-and-drop.

### 1.1 — El ciclo de vida de una app JavaFX

En Swing alcanzaba con un `main()` que creara un `JFrame`. JavaFX tiene un ciclo de vida más estructurado, a través de la clase `javafx.application.Application`:

```java
public class Prueba extends Application {

    @Override
    public void start(Stage primaryStage) {
        // acá se construye la interfaz y se muestran las ventanas
    }

    public static void main(String[] args) {
        // launch() es la única forma correcta de arrancar una app JavaFX.
        // Internamente crea la instancia de Prueba y llama a start().
        // Nunca se debe instanciar Application con "new Prueba()".
        launch(args);
    }
}
```

Pasos del ciclo de vida:

1. `main()` llama a `launch()`, que inicializa el toolkit gráfico.
2. `init()` — opcional, inicialización sin UI.
3. `start(Stage)` — acá se construye y se muestra la interfaz. Es el único obligatorio.
4. `stop()` — opcional, limpieza al cerrar la app.

JavaFX crea automáticamente el primer `Stage` y lo pasa como parámetro a `start()`. Para ventanas adicionales se usa `new Stage()`.

### 1.2 — Stage, Scene y el Scene Graph

| Concepto | Qué es | Equivalente en Swing |
|---|---|---|
| `Stage` | La ventana del sistema operativo (barra de título, minimizar/maximizar/cerrar) | `JFrame` |
| `Scene` | El área de contenido dentro del `Stage`. Contiene el *scene graph*: un árbol de `Node`s | `JPanel` raíz |
| `Node` | Cualquier elemento visual: `Label`, `Button`, `TextField`, layouts como `GridPane`, etc. | `JComponent` |

Relación: un `Stage` tiene una `Scene`; una `Scene` tiene un `Node` raíz.

```java
Scene scene = new Scene(nodoRaiz, ancho, alto);
stage.setScene(scene);
```

### 1.3 — Layouts

| Layout JavaFX | Organiza | Equivalente en Swing |
|---|---|---|
| `GridPane` | Filas y columnas | `MigLayout` / `GridLayout` |
| `HBox` | Apila horizontalmente (fila) | `FlowLayout` |
| `VBox` | Apila verticalmente (columna) | `BoxLayout` |
| `BorderPane` | 5 zonas: top / center / right / bottom / left | `BorderLayout` |

Ejemplo de `GridPane` (ventana de login):

```java
GridPane grid = new GridPane();
grid.setPadding(new Insets(12));  // margen interno
grid.setHgap(8);                  // espacio horizontal entre celdas
grid.setVgap(8);                  // espacio vertical entre celdas

ColumnConstraints col1 = new ColumnConstraints();
col1.setHgrow(Priority.ALWAYS);   // esta columna crece para ocupar espacio libre
grid.getColumnConstraints().addAll(new ColumnConstraints(), col1);

// add(nodo, columna, fila)
grid.add(new Label("Usuario:"), 0, 0);
grid.add(new TextField(),       1, 0);
grid.add(new Button("Iniciar"), 1, 1);
```

Ejemplo de `BorderPane` + `HBox` + `VBox` (ventana principal del chat):

```
   ┌──────────────────────────┐
   │          TOP             │
   ├─────────────┬────────────┤
   │             │            │
   │   CENTER    │   RIGHT    │
   │  (crece)    │            │
   ├─────────────┴────────────┤
   │          BOTTOM          │
   └──────────────────────────┘
```

```java
BorderPane root = new BorderPane();
root.setCenter(centroBox);   // HBox con el chat + lista de usuarios
root.setBottom(barraInferior); // HBox con el campo de texto + botón enviar
```

`HBox.setHgrow(nodo, Priority.ALWAYS)` y `VBox.setVgrow(nodo, Priority.ALWAYS)` hacen que un nodo se estire para ocupar el espacio disponible — es el equivalente a los pesos de `MigLayout`.

### 1.4 — Controles básicos

| JavaFX | Swing | Notas |
|---|---|---|
| `Label` | `JLabel` | |
| `TextField` | `JTextField` | mismo nombre, paquete distinto |
| `Button` | `JButton` | |
| `TextArea` | `JTextPane` | `setEditable(false)` para solo lectura |
| `ListView<T>` | `JList` + `AbstractListModel` | tipado con generics, se alimenta con `ObservableList` — no hace falta escribir un modelo propio |

Ejemplo de `ListView`:

```java
ListView<String> listUsuarios = new ListView<>();
listUsuarios.setItems(FXCollections.observableArrayList(nombresDeUsuarios));
```

### 1.5 — Eventos con lambdas

En Swing se implementaba `ActionListener`. En JavaFX se usa `EventHandler<T>`, casi siempre como lambda:

```java
btnIniciar.setOnAction(e -> {
    // lógica al hacer click
});
```

Atajo útil: capturar la tecla ENTER en una `Scene` para que dispare el botón por defecto (equivalente a `setDefaultButton` en Swing):

```java
scene.setOnKeyPressed(e -> {
    if (e.getCode() == KeyCode.ENTER) {
        btnIniciar.fire();
    }
});
```

Cierre de ventana (equivalente a `WindowAdapter.windowClosing()`):

```java
stage.setOnCloseRequest(handler); // handler: EventHandler<WindowEvent>
```
