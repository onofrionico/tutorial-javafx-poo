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

---

## Bloque 2 — MVC y la migración del chat

### 2.1 — Por qué separar en capas

El proyecto original de chat en Swing ya estaba armado con MVC bien separado: hay una interfaz `IVista`, y `Controlador`/`Modelo` no saben nada de Swing. Gracias a esto, migrar a JavaFX se puede acotar **solo** al paquete `vista.grafica` (3 clases), sin tocar `Controlador`, `Modelo`, ni la lógica de red (en la versión RMI).

```java
public interface IVista {
    void iniciar();
    // ... métodos que el Controlador necesita llamar sobre la vista
}
```

La clave para que la migración sea transparente: mantener **exactamente los mismos nombres y firmas de métodos** que ya usaba la vista Swing (`onClickEnviar`, `actualizarChat`, `getTextoMensaje`, `addWindowListener`, etc.). Así la clase que conecta todo (`VistaGrafica`) queda intacta desde el punto de vista del `Controlador`.

### 2.2 — Recorrido del código real

Tres clases forman la vista gráfica:

- **`VentanaInicioSesion`** — pantalla de login. `Stage` + `Scene` + `GridPane` con un `Label`, un `TextField` y un `Button`.
- **`VentanaPrincipal`** — pantalla del chat. `BorderPane` con un `HBox` central (el `TextArea` del chat + un `VBox` con la lista de usuarios) y un `HBox` inferior (campo de mensaje + botón enviar).
- **`VistaGrafica`** — coordina las dos ventanas anteriores e implementa `IVista`.

Método de actualización del chat (se llama cada vez que el modelo cambia):

```java
public void actualizarChat(IMensaje[] mensajes) {
    StringBuilder sb = new StringBuilder();
    for (IMensaje m : mensajes) {
        sb.append(m.getFecha().format(FORMATO));
        if (m.isMensajeDelSistema()) {
            sb.append("AVISO: ");
        } else {
            sb.append(m.getUsuario().getNombre()).append(": ");
        }
        sb.append(m.getTexto()).append("\n");
    }
    textChat.setText(sb.toString());
    textChat.setScrollTop(Double.MAX_VALUE);  // scroll automático al final
}
```

### 2.3 — Tabla comparativa Swing vs JavaFX

| Concepto | Swing | JavaFX |
|---|---|---|
| Ventana | `JFrame` | `Stage` |
| Contenedor raíz | `JPanel` | `Scene` |
| Layout en grilla | `MigLayout` / `GridLayout` | `GridPane` |
| Layout en fila | `FlowLayout` | `HBox` |
| Layout en columna | `BoxLayout` | `VBox` |
| Layout con zonas | `BorderLayout` | `BorderPane` |
| Texto editable | `JTextField` | `TextField` |
| Texto multilínea | `JTextPane` | `TextArea` |
| Lista | `JList` + `AbstractListModel` | `ListView<T>` + `ObservableList` |
| Botón | `JButton` | `Button` |
| Etiqueta | `JLabel` | `Label` |
| Evento de botón | `ActionListener` | `EventHandler<ActionEvent>` (lambda) |
| Evento de cierre | `WindowAdapter.windowClosing()` | `stage.setOnCloseRequest()` |
| Hilo de UI | `SwingUtilities.invokeLater()` | `Platform.runLater()` |
| Estilos | Propiedades Java (`.setBackground()`) | CSS con propiedades `-fx-` |

### 2.4 — `Platform.runLater()` y los hilos

En la versión con RMI (`ejemplo-chat-rmimvc-fx`), las actualizaciones del chat pueden llegar por un callback de red en **otro hilo**, no el hilo de UI. JavaFX (igual que Swing) exige que todo lo que toca componentes visuales corra en el hilo de UI:

```java
// Swing:  SwingUtilities.invokeLater(() -> actualizarChat(mensajes));
// JavaFX: Platform.runLater(() -> actualizarChat(mensajes));
```

No hace falta explicar threading a fondo en esta clase — alcanza con mostrar que existe el mismo problema en ambos frameworks y la misma solución con distinto nombre de método.

### 2.5 — Demo en vivo

Correr el proyecto con Maven:

```bash
mvn javafx:run
```

Se abren **dos ventanas de login** — dos "clientes" que comparten el mismo modelo `Chat`, simulando dos usuarios chateando desde la misma máquina. Iniciar sesión con un nombre distinto en cada una y mandar mensajes cruzados para mostrar que se sincronizan.
