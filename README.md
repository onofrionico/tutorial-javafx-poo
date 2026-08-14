# Tutorial JavaFX para POO

Material didáctico para introducir interfaces gráficas con JavaFX en la cátedra de POO, usando como caso real la migración de una app de chat de Swing a JavaFX.

## Contenido

- [`tutorial.md`](./tutorial.md) — guía escrita completa, para que los alumnos la lean a su ritmo después de la clase.
- [`slides.pptx`](./slides.pptx) — presentación para proyectar en clase.

## Estructura de la clase

1. **Fundamentos sin Scene Builder** — Stage, Scene, layouts, controles, eventos, todo escrito a mano.
2. **MVC y la migración del chat** — por qué separar en capas, recorrido del código real, demo en vivo del chat funcionando.
3. **Scene Builder** — instalación, diseño visual arrastrando componentes, FXML, patrón `@FXML` Controller. Esto es lo que se va a usar más adelante para el proyecto del juego.

## Repos de código usados como ejemplo

- [ejemplo-chat-mvc-fx](https://github.com/onofrionico/ejemplo-chat-mvc-fx) — chat simple, un solo proceso.
- [ejemplo-chat-rmimvc-fx](https://github.com/onofrionico/ejemplo-chat-rmimvc-fx) — misma app con cliente/servidor vía RMI.

Ambos repos tienen el historial completo de la migración: desde el commit original en Swing hasta el commit final en JavaFX, commit por commit.

## Para los profesores — comparación Swing → JavaFX

Estos links usan la vista nativa de comparación de GitHub entre el primer commit (Swing) y el último (JavaFX) de cada repo. No hace falta clonar nada, se ve línea por línea directo en el navegador. (No es material para los alumnos, es para que la cátedra vea el cambio tecnológico de un vistazo.)

- [Compare ejemplo-chat-mvc-fx: Swing → JavaFX](https://github.com/onofrionico/ejemplo-chat-mvc-fx/compare/9b5c01a...b3917cf)
- [Compare ejemplo-chat-rmimvc-fx: Swing → JavaFX](https://github.com/onofrionico/ejemplo-chat-rmimvc-fx/compare/ba94433...61c792f)

## Licencia de Scene Builder

Scene Builder es gratuito y open source (licencia BSD), mantenido por Gluon. No tiene costo para los alumnos. Ver [gluonhq.com/products/scene-builder](https://gluonhq.com/products/scene-builder/).
