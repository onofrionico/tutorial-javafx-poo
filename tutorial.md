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

---

## Bloque 3 — Scene Builder

Todo lo que armamos a mano en el Bloque 1 se puede construir también arrastrando y soltando componentes con **Scene Builder**. Vale la pena mostrarlo recién ahora: entender el código primero ayuda a leer lo que Scene Builder genera automáticamente.

### 3.1 — Licencia

Scene Builder es **gratuito y open source** (licencia BSD), mantenido por [Gluon](https://gluonhq.com/products/scene-builder/). No tiene costo para los alumnos, ni para uso comercial.

### 3.2 — Instalación

1. Descargar el instalador desde [gluonhq.com/products/scene-builder](https://gluonhq.com/products/scene-builder/) (Windows/macOS/Linux).
2. En IntelliJ IDEA: `File → Settings → Languages & Frameworks → JavaFX` y apuntar el campo "Path to Scene Builder" al ejecutable instalado. Esto permite hacer doble click en un `.fxml` y que abra directo en Scene Builder.

### 3.3 — Demo en vivo: recrear la ventana de login

Objetivo: recrear visualmente la misma ventana `VentanaInicioSesion` que se armó a mano en el Bloque 1 (Label + TextField + Button en un GridPane), pero esta vez arrastrando componentes desde la paleta de Scene Builder al canvas.

Pasos de la demo:

1. Abrir Scene Builder. Dos formas de hacerlo:
   - **Standalone:** ejecutar directo el `.exe`/`.app` instalado en el paso anterior (en Windows, por defecto queda en `%LOCALAPPDATA%\SceneBuilder\SceneBuilder.exe`, dentro de una carpeta oculta — hay que pegar la ruta a mano si el explorador no la muestra).
   - **Desde IntelliJ** (una vez configurado el "Path to Scene Builder" en el paso anterior): clic derecho sobre la carpeta `src/main/resources` → `New → Scene Builder File`, o doble click sobre un `.fxml` ya existente — IntelliJ lo abre directo en Scene Builder sin salir del IDE. Esta es la forma recomendada para la demo en clase.
2. Scene Builder → nuevo documento → elegir `GridPane` como contenedor raíz.
3. Arrastrar un `Label`, un `TextField` y un `Button` a la grilla.
4. En el panel de la derecha (Inspector), configurar texto y propiedades igual que en el código del Bloque 1 (`"Usuario:"`, `promptText`, `"Iniciar"`).
5. Guardar el archivo dentro de `src/main/resources/` (por ejemplo como `login.fxml`) — esa es la carpeta que Maven copia al classpath, así después se puede cargar con `getClass().getResource("/login.fxml")`.

### 3.4 — El FXML generado

Scene Builder guarda el diseño como XML declarativo. Comparar con el código imperativo del Bloque 1: mismo resultado, dos formas de expresarlo.

```xml
<GridPane hgap="8.0" vgap="8.0" xmlns="http://javafx.com/javafx"
          xmlns:fx="http://javafx.com/fxml" fx:controller="ar.edu.unlu.chatmvc.LoginController">
    <Label text="Usuario:" GridPane.columnIndex="0" GridPane.rowIndex="0" />
    <TextField fx:id="textUsuario" GridPane.columnIndex="1" GridPane.rowIndex="0" />
    <Button fx:id="btnIniciar" text="Iniciar" onAction="#onIniciar"
            GridPane.columnIndex="1" GridPane.rowIndex="1" />
</GridPane>
```

Puntos a resaltar:

- `fx:controller` — la clase Java que va a manejar los eventos de este FXML.
- `fx:id` — el nombre de campo que va a recibir la inyección automática en el controller.
- `onAction="#onIniciar"` — conecta el botón directamente a un método del controller, sin escribir el `setOnAction(...)` a mano.

### 3.5 — El patrón `@FXML` Controller

```java
public class LoginController {

    @FXML
    private TextField textUsuario;

    @FXML
    private Button btnIniciar;

    @FXML
    private void onIniciar(ActionEvent event) {
        String nombre = textUsuario.getText();
        // lógica al iniciar sesión
    }
}
```

JavaFX inyecta automáticamente los campos anotados con `@FXML` que coincidan con un `fx:id` del FXML, y conecta los métodos `@FXML` referenciados por `onAction`. No hace falta buscar los nodos a mano ni registrar los listeners con código.

### 3.6 — Cargar el FXML

```java
@Override
public void start(Stage primaryStage) throws IOException {
    Parent root = FXMLLoader.load(getClass().getResource("/login.fxml"));
    primaryStage.setScene(new Scene(root));
    primaryStage.show();
}
```

`FXMLLoader` lee el archivo, instancia el `fx:controller`, inyecta los campos `@FXML` y arma el `Node` raíz — todo en una línea.

### 3.7 — Por qué esto importa para el proyecto del juego

El patrón FXML + Controller es el que se va a usar para las pantallas del proyecto de juego más adelante: diseño visual rápido en Scene Builder, lógica separada en una clase Controller. Lo que se vio en este bloque — instalación, arrastrar componentes, `fx:id`, `@FXML`, `FXMLLoader` — es exactamente lo que van a reutilizar ahí.
