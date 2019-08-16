# MediastreamMetricsSDKiOS
La presente es una guía para vincular el uso de players externos a Mediastream al uso de nuestro sistema de analíticas.

## Consideraciones:
Para el correcto funcionamiento de las métricas es mediastream, es necesario realizar ciertas acciones por el lado de la app a conectarse con Mediastream.

Primero definiremos algunos términos:

| Nombre | Definición |
| --- | --- |
| id | Se considera `id` cualquier cadena de caracteres alfanuméricos que permita obtener data de platform, éstos pueden pertenecer a `MEDIA` (AUDIO | VIDEO), `LIVE` (AUDIO | VIDEO), `EPISODIO` (AUDIO)|
| EMBED_HOST | Corresponde al host donde se encuentra alojado el contenido, estos pueden ser `http://develop.mdstrm.com`(DEV) o `https://mdstrm.com` (PROD) |
| type | El tipo de contenido a reproducirse. `video` `live-stream` `episode` |
| MDSTRMUID | ID de usuario único |
| MDSTRMSID | ID de sesión único |
| MDSTRMPID | ID de playback único |

## Consumo
Para poder consumir contenido de nuestra plataforma es necesario generar primero una url de consulta, la cuál se arma con la siguiente estructura:

`{EMBED_HOST}/{type}/{id}.json`

Si la url resulta correcta, por ejemplo `https://mdstrm.com/video/5d543e1b4fd7920fbadb1b35.json` el request tipo GET debería responder algo como ésto:

```javascript
{
  "src": {
    "hls": "https://mdstrm.com/video/5d543e1b4fd7920fbadb1b35.m3u8",
    "f4m": "http://mdstrm.com/video/5d543e1b4fd7920fbadb1b35.f4m",
    "hss": "https://mdstrm.com/video/5d543e1b4fd7920fbadb1b35.hss",
    "mp4": "https://mdstrm.com/video/5d543e1b4fd7920fbadb1b3d.mp4",
    "mpd": "http://mdstrm.com/video/5d543e1b4fd7920fbadb1b35.mpd",
    "p": {}
  },
  "show_title": false,
  "show_status": true,
  "title": "VOD - SDK (No Borrar)",
  "poster": "https://mdstrm.com/thumbs/55117baedc01616019533551/thumb_5d543e1b4fd7920fbadb1b35_5d543e1b4fd7920fbadb1b42_606s.jpg",
  "subtitles": [],
  "account": "55117baedc01616019533551",
  "ads": {},
  "MDSTRMUID": "pudepEd2wuckKVJKEd8f1ZjhaE2ezJRI",
  "MDSTRMSID": "OPxuZ7NAJ5QD9EfAT5Pa4zjeztXXeW2r",
  "MDSTRMPID": "LJkvtLmbZXwWL6UqQcbIsMi3mXGUEKOs"
}
```

## Analizando JSON de respuesta
El JSON de respuesta puede variar dependiendo de la configuración que tenga el contenido en ese momento. Mucha información es de uso exclusivo para Mediastream por ende vamos a centrarnos en las más importantes y realmente útiles para la integración.

| Nombre | Definición |
| --- | --- |
| src | Aquí se puede encontrar las diferentes urls de consumo de contenido, dependerá del player en cuestión el optar por la mejor opción |
| account | Corresponde a la cuenta a la que pertenece el media, live o episodio en cuestión |
| ads | Si la media tuviese configurado algún contenido publicitario aquí llegaría toda la información de los tags |
| MDSTRMUID | ID de usuario generado por platform. Se debe de hacer uso de `cookie jar` para que este id sea consistente, caso contrario va a cambiar cada que se solicite el contenido, aunque el dispositivo sea el mismo.  |
| MDSTRMSID | ID de sessión de usuario, más adelante se explicará con detalle. |
| MDSTRMPID | ID de reproducción de contenido, más adelante se explicará con detalle. |

Muchas veces el contenido está restringido ya sea por DRM, GeoBloqueo u otro, en ese caso el JSON de respuesta vendrá acompañado de un error y su respectivo mensaje.

## Consumiendo un source
Después de decidir que formato es compatible con el player, es necesario añadir a la url ciertos query strings necesarios para obtener una mayor riqueza en la información obtenida. Éstos query strings son:

| Nombre | Valor | Mandatorio |
| --- | --- | --- |
| uid | MDSTRMUID | SI |
| pid | MDSTRMPID | SI |
| sid | MDSTRMSID | SI |
| an | Nombre de la Applicación | NO |
| av | Versión de la Applicación | NO |
| c | Id del cliente (No de la cuenta) | NO |
| access_token | Access token en caso de contenido protegido | NO |
| max_profile | Máxima rendición a reproducir por el player | NO |
| ds | Id del distribuidor (Orientado más a Podcast) | NO |
| res | Resolución del dispositivo | NO |
| at | Tipo de Aplicación (android-app | ios-app) | SI |
| dnt | true | SI |

Al final se debería de obtener un source de consumo similar al siguiente:

`https://mdstrm.com/video/5d543e1b4fd7920fbadb1b35.m3u8?uid=DA0cIrzAriJq8zbwYVN1rdwGOe1WrUbq&pid=fQCHHoJoJvNSD9Od3GPdgcF4b1LYGXfj&sid=5ApAARc33zEvZ20HDyF0z20vfVbVri8u&res=1080x1776&sdk=true&at=android-app&dnt=true`

# Metrics
Ya que logramos consumir contenido de Platform es necesaria la integración con nuestro sistema de analytics. Vamos a definir algunos detalles:

## Definiciones y Conceptos
Existe una Sesión (session) que es única y permanente por Browser o Cliente HTTP que requiera el contenido. Las sesiones pueden tener asociado multiples eventos los cuales definen el comportamiento del usuario y están asociados a una acción explicita o implícita de los usuarios. Los eventos pertenecen a diferentes categorías entre ellas Eventos Generales, Eventos de Usuario, Eventos de Playback, entre otros. Cada evento se identifica por su nombre, timestamp y la data asociada al mismo y puede ser generado desde distintas fuentes externas cómo Tracker, Proceso de Logs, Platform, entre otros.

## Session
Cuando un usuario genera una primera acción y esta genera un evento, se le asigna un identificador de sesión (sessionId) en caso que no tenga uno previamente asignado. Esta sesión es persistente a través de cookies o queryString y se renueva por 30 min extras en cada evento que el usuario genere, es decir, si el usuario para de generar eventos por 30 minutos la sesión expirará y todos los eventos que genere posterior a eso serán asociado a otra sesión y otro sessionId.
La session se puede generar en el sistema de balanceo o en la generación del primer evento que el usuario haya generado. La sesión es persistente a nivel de browser.

## Eventos
Los eventos permiten trazar el comportamiento explícito e implícito de un usuario. Todos los eventos deben contener timestamp server-side del momento que se generó el evento. Aunque existen muchos eventos para nuestra integración nos vamos a centrar en los eventos de tipo `PLAYBACK`

### Eventos Playback:
Son todos los eventos generados debido a la reproducción de un contenido audiovisual ya sea por acción directa del usuario o por consecuencia de las mismas. Algunos ejemplos de eventos de esta categoría son: playback_start, playing, buffering, entre otros.

Algunos eventos de playback son generados automáticamente por platform al momento de que un media es requerido por ejemplo: playback_start.

Para almacenar éstos eventos existe data requerida y opcional, a continuación el detalle:

| Event Data Name | Data Category | Event Data Description | Ejemplo |
| --- | --- | --- | --- |
| session_id | general | Identificador único de la sesión que ha sido iniciada. | VA6nBImZXmbeH4bQNrA1CO2bo4qjtKvA |
| playback_id | general | Identificador único de la reproducción que se está llevando a cabo por un dispositivo. Este ID es generado por el ente que da inicio al playback. | vJKHMht0o69kN3SkM0VYbKWSYQSeEGPZ |
| unique_id | general | Identificador único de usuario. | 2XoiIwQL8VMTfiFwut4PIXHIJmvT2qS0 |
| content_id | platform | ID del contenido que se solicita referente a Platform u otro identificador de contenido. | 529a628d2808dd092a00001a |
| player_loading_time | general | Tiempo que transcurrió entre que comenzó a cargar el player hasta que estuvo listo para operar en milisegundos. (ms) | 800 |
| position | playback | Position absoluta en la que se genera el evento referente al contenido. Ejemplo: Si se genera un evento al minuto 01:20 de reproducción de un video se debe informar 80000.(ms) | 80000 |
| duration | playback | Duración total del contenido referente a la reproducción, en milisegundos. En caso de contenido en demanda se debe informar la duración total del contenido. En caso de contenido Live se debe informar la duración del schedule en caso que exista schedule para ese periodo. En caso de un contenido Live que no posee un schedule, no debe informar este valor. (ms) | 200000 |
| buffering_time | playback | Representa el tiempo que el player ha estado haciendo buffer. Es relativo a la última vez que se informó a Metrics. Por ejemplo si el player entró a modo buffering a las 15:30:00 y se informa a Metrics a las 15:30:10 el valor será de 10000, si continúa el buffer y sale antes del siguiente track a metrics por ejemplo 15:30:15 y el siguiente track es 15:30:20 el valor de buffering time será de 5000. Siempre es con relación al último valor trackeado, no es acumulativo. (ms) | 3000 |
| from_position | playback | Se usa cuando se informa un evento tipo seek, corresponde a la posición anterior del player antes de su nueva posición. (ms) | 5000 |
| relative_streamed_time | playback | Duración en milisegundossegundos de cuantos milisegundos ha hecho streaming (visto o escuchado) el usuario. Es relativa dado que se debe informar a contar desde la última vez que se informo y no el total reproducido. (ms) | 10000 |
| content_audio_bitrate | playback | Bitrate en Kbps del audio del contenido que está siendo reproducido en el momento que se informa el evento. | 96 |
| content_video_bitrate | playback | Bitrate en Kbps del video del contenido que está siendo reproducido en el momento que se informa el evento. | 1000 |
| content_resolution | playback | Resolución de video del contenido que se encuentra reproduciendo el usuario en el momento que se genera el evento, en pixeles. Solo aplica para contenido de video. | 854x480 |

## Detalle de Eventos
A continuación el detalle de la estructura de cada evento a informarse:

| Nombre de Evento | Descripción | Estructura |
| --- | --- | --- |
| content_end | Se genera cuando el player llegó al final del contenido. Se deben enviar los eventos generados hasta este momento. | timestamp* playback_id* position* |
| playing | Se genera cuando el player se encuentra reproduciendo contenido (no publicidad). | timestamp* playback_id* position* duration* relative_streamed_time* content_resolution* bandwith* content_video_bitrate* content_audio_bitrate* |
| play | Se genera cuando 1) el usuario da click en play 2) el player hace autoplay 3) se utiliza el método de play a través de API. No se debe informar este evento cuando el player hace seek. | timestamp* playback_id* position* |
| pause | Se genera al detener la reproducción. Se deben enviar los eventos generados hasta este momento. | timestamp* playback_id* position* |
| buffering | Se genera cuando el player que se encuentra reproduciendo el contenido está en proceso de buffering. | timestamp* playback_id* position* buffering_time* |
| seek | Se genera cuando el player ya sea por acción del usuario o por acción programática (API) el player adelanta o retrocede el contenido. | timestamp* playback_id* position* from_position* unique_id session_id |
| player_loaded | Se genera cuando el player (web o sdk) ha terminado de cargar todos sus elementos para el correcto funcionamiento en una propiedad (página web, app, otro). | timestamp* session_id* playback_id player_loading_time* |

Todos los eventos deben contener en su estructura: content_id* playback_id* unique_id* session_id*

# MediastreamMetricsSDK
## Integración

Añada en su archivo de librerías de android la siguiente dependencia:

```java
pod 'MediastreamTrackeriOS'
```

## Clase MediastreamCollectorConfig
### Atributos

| Nombre | Tipo | Mandatorio | Descripción |
| --- | --- | --- | --- |
| debug | Boolean | NO | Sirve para revisar logs de track |
| account | String | NO | ID correspondiente a la cuenta de los medias a informar |
| environment | MediastreamCollectorConfig.Environment | No | Ambiente en que se va a instanciar collector, `PRODUCTION` o `DEV`. Default: `PRODUCTION` |

### Construtor
MediastreamCollectorConfig()

## Classe MediastreamCollector
MediastreamCollector crea una instancia de collector, para enviar datos a platform.

### Métodos

| Retorno | Método | Descripción |
| --- | --- | --- |
| void | addPlayerLoadEvent(JSONObject: data) | Recibe data con estructura de datos player_loaded. |
| void | addPlayEvent(JSONObject: data) | Recibe data con estructura de datos play. |
| void | addPauseEvent(JSONObject: data) | Recibe data con estructura de datos pause. |
| void | addContentEndEvent(JSONObject: data) | Recibe data con estructura de datos content_end. |
| void | addSeekEvent(JSONObject: data) | Recibe data con estructura de datos seek. |
| void | addBufferingEvent(JSONObject: data) | Recibe data con estructura de datos buffering. |
| void | addPlayingEvent(JSONObject: data) | Recibe data con estructura de datos playing. |
| void | startCollector() | Inicia el ping de collector, éste va a intentar trackear la info agregada cada 10seg |
| void | stopCollector() | Detiene el ping de collector |
| void | track() | Informa a metrics lo que tenga registrado en el momento. |

### Construtor

MediastreamCollector(config: MediastreamCollectorConfig, Context: context)

## Ejemplo de Uso

```swift
import UIKit
import MediastreamTrackeriOS

class ViewController: UIViewController {
    var collector: MediastreamCollector?

    override func viewDidLoad() {
        super.viewDidLoad()
        let mediastreamCollectorConfig = MediastreamCollectorConfig()
        mediastreamCollectorConfig.environment = MediastreamCollectorConfig.Environment.DEV
        mediastreamCollectorConfig.debug = true;
        mediastreamCollectorConfig.account = "5bd3720e9a2d860c55c8b766"
        self.collector = MediastreamCollector(config: mediastreamCollectorConfig)
    }
}

```

La clase está diseñada para que se inicie un timer que se va a ejecutar cada 10 segundos e intentar informar los eventos registrados hasta el momento. También tiene el método TRACK() en caso de que se desee implementar un propio timer. Lo único que se recomienda es que respete los 10 segundos de track original, de lo contrario Mediastream no se responsabiliza por una información no tan clara o segmentada.

Es recomendable, utilizar éste SDK con configuración DEV hasta corroborar con el equipo de Mediastream, que la información está llegando de manera correcta para evitar ensuciar así la data de su cliente en producción.