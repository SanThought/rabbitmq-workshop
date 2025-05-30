# Diagrama de Arquitectura (con RabbitMQ)

Este diagrama ilustra la arquitectura de la aplicación, mostrando cómo los servicios interactúan a través de Traefik y RabbitMQ.

```mermaid
graph TD
    subgraph "Navegador del Usuario"
        U["Usuario"] --> T{"Traefik Reverse Proxy"}
    end

    subgraph "Red Docker (web)"
        T -- "/cliente/uno" --> CU["cliente-uno (Express.js)"]
        T -- "/cliente/dos" --> CD["cliente-dos (Express.js)"]
        T -- "/panel" --> P["panel (Express.js)"]
        T -- "/reporte" --> AR["api-reporte (Express.js)"]
        T -- "/reporte/recent" --> AR
        T -- "/logs" --> LC["logger-central (Express.js)"]
        T -- ":8080" --> TD["Traefik Dashboard"]
        CU -- "Publica evento (JSON)" --> R((RabbitMQ))
        CD -- "Publica evento (JSON)" --> R
        R -- "Consume evento" --> AR
        P -- "GET /reporte (Auth)" --> AR
        P -- "GET /reporte/recent (Auth)" --> AR
        R -.-> RMUI["RabbitMQ UI (:15672)"]
        LC --- Note1["Log a consola"]
        style Note1 fill:#fff,stroke:#fff,stroke-width:0px
    end

    U -- ":15672" --> RMUI
```


## Flujo de Datos:

1.  **Acceso Cliente:** El usuario accede a `http://localhost/cliente/uno` o `http://localhost/cliente/dos`.
    *   Traefik (`T`) recibe la solicitud y la redirige al contenedor `cliente-uno` (`CU`) o `cliente-dos` (`CD`) correspondiente, eliminando el prefijo `/cliente/uno` o `/cliente/dos`.
    *   Cada servicio cliente (`CU`, `CD`) responde al navegador y, periódicamente (y al inicio), publica un mensaje de evento en el exchange `events_exchange` de RabbitMQ (`R`) con la clave de enrutamiento `event.analytics`.
2.  **Procesamiento de Eventos:**
    *   RabbitMQ (`R`) enruta los mensajes desde `events_exchange` a la cola `analytics_queue`.
    *   El servicio `api-reporte` (`AR`), que está suscrito a `analytics_queue`, recibe los mensajes.
    *   `api-reporte` procesa cada mensaje, actualiza un contador en memoria para el `serviceId` correspondiente y almacena los últimos 5 mensajes.
3.  **Visualización del Panel:**
    *   El usuario accede a `http://localhost/panel`.
    *   Traefik (`T`) redirige la solicitud al servicio `panel` (`P`), eliminando el prefijo `/panel`.
    *   El servicio `panel` (`P`) periódicamente hace solicitudes HTTP GET a `http://api-reporte:3000/reporte` y `http://api-reporte:3000/reporte/recent` (internamente dentro de la red Docker).
    *   Estas solicitudes pasan por Traefik (`T`), que aplica la autenticación básica (usuario: `admin`, contraseña: `password`) y las redirige a `api-reporte` (`AR`).
    *   `api-reporte` (`AR`) responde con el reporte de conteos (en texto plano) y los mensajes recientes (en JSON).
    *   El servicio `panel` (`P`) renderiza una página HTML que muestra el reporte agregado y la tabla de eventos recientes, y la envía al navegador del usuario. La página se auto-refresca cada 5 segundos.
4.  **Registro Centralizado:**
    *   (Opcional) Si un servicio enviara un POST a `/logs`, Traefik (`T`) lo redirigiría a `logger-central` (`LC`).
    *   `logger-central` (`LC`) imprimiría el log en su consola y respondería "registro recibido".
5.  **Herramientas de Gestión:**
    *   El Dashboard de Traefik es accesible en `http://localhost:8080` (`TD`).
    *   La UI de gestión de RabbitMQ es accesible en `http://localhost:15672` (`RMUI`).

## Componentes:

*   **Traefik:** Actúa como reverse proxy y balanceador de carga, enrutando las peticiones externas a los servicios internos y gestionando la autenticación básica para `/reporte`.
*   **cliente-uno / cliente-dos:** Servicios Node.js/Express que simulan clientes. Ahora publican eventos en RabbitMQ en lugar de hacer llamadas HTTP directas a `api-reporte`.
*   **api-reporte:** Servicio Node.js/Express que actúa como consumidor de RabbitMQ. Agrega datos de los eventos recibidos y los expone a través de una API HTTP (`/reporte`, `/reporte/recent`) protegida.
*   **panel:** Servicio Node.js/Express dinámico que consulta periódicamente a `api-reporte` y muestra los datos en una página HTML auto-refrescante.
*   **logger-central:** Servicio Node.js/Express simple para recibir logs (actualmente no utilizado activamente por otros servicios).
*   **RabbitMQ:** Broker de mensajes que desacopla los servicios cliente de `api-reporte`, permitiendo una comunicación asíncrona y más robusta. 
