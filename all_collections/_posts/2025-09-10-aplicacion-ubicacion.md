---
layout: post
title: Crea tu propia apliación Android para falsificar la localización GPS
date: 2025-09-10 
categories: [Android]
---

¿Alguna vez te has preguntado si es posible crear una aplicación para Android que sea capaz de falsificar la ubicación del móvil?. Yo si. En cuanto me enteré de que esto era posible, decidí crear mi propia aplicación (bastante sencilla, eso sí) sin ningún tipo de experiencia previa en el desarrollo de aplicaciones Android.

Hoy aprenderás a hacerlo tu también.

Mi aplicación
-------------

Puedes encontrar mi aplicación en este [repositorio](https://github.com/3z-p0wn/gps-spoofer) de GitHub.

![Interfaz aplicación](/assets/images/2025-01-19-aplicacion-ubicacion/inicio.png)

El funcionamiento es muy sencillo, simplemente tienes que introducir las coordenadas de la localización que quieres simular (latitud y longitud) y posteriormente presionar el botón **Spoof!**.

> Para que la aplicación funcione, debes otorgarle permisos de ubicación y seleccionarla como _Aplicación para localización ficticia_ en las opciones de desarrollador_._

Funcionamiento
--------------

En este punto, quizá te preguntes cómo logra la aplicación falsificar la ubicación del dispositivo

Resulta que Android, desde hace mucho tiempo cuenta con una API conocida como **Mock Location API,** que permite a los desarrolladores crear ubicaciones ficticias para facilitar el desarrollo de aplicaciones basadas en geolocalización.

Vamos a ver un resumen del funcionamiento de cada parte de la aplicación (Recuerda que el código completo está disponible en [_GitHub_](https://github.com/3z-p0wn/gps-spoofer)):

### 1. Estructura de la aplicación

![estructura aplicación](/assets/images/2025-01-19-aplicacion-ubicacion/estructura.png)

La aplicación integra dos componentes principales: **MainActivity.kt (**Donde está definida la interfaz de la aplicación**)** y **MockLocationService (**Servicio que se encarga de generar las ubicaciones ficticias**).**

### 2. MockLocationService

Cuando presionas el botón de **Spoof!** en la aplicación, se inicia el servicio **MockLocationService,** que se encarga de actualizar la localización falsa cada 200 milisegundos para que sea más realista. Para más información sobre como crear servicios en Android ver este [artículo](https://developer.android.com/develop/background-work/services)

Este servicio solo se detiene cuando presionas el botón **Stop** o cuando se cierra la apicación.

El primer paso es crear un _test provider,_ con el que posteriormente generaremos las localizaciones:

```kotlin
locationManager = getSystemService(Context.LOCATION_SERVICE) as LocationManager
locationManager.addTestProvider(
                LocationManager.GPS_PROVIDER,
                true,
                false,
                false,
                false,
                false,
                false,
                false,
                ProviderProperties.POWER_USAGE_HIGH,
                ProviderProperties.ACCURACY_FINE
            )
locationManager.setTestProviderEnabled(LocationManager.GPS_PROVIDER, true)
```

Ahora, desde la función **mockLoop** inyectamos las coordenadas que introducimos en la aplicación cada 200 milisegundos (más tarde profundizaré más en el flujo de ejecución de las diferentes funciones de la aplicación):

```
while (isMocking) {
            val mockLocation = Location(LocationManager.GPS_PROVIDER).apply {
                latitude = lat
                longitude = lon
                altitude = 12.5
                time = System.currentTimeMillis()
                accuracy = 2f
                elapsedRealtimeNanos = SystemClock.elapsedRealtimeNanos()
            }
            locationManager.setTestProviderLocation(
                LocationManager.GPS_PROVIDER,
                mockLocation
            )
            Log.d(TAG, "Ubicación: $lat, $lon")
            delay(200L)
        }
```

### 3. MainActivity

En desarrollo Android, una Actvidad es la que se encarga de manejar la interfaz de usuario y las interacciones con la pantalla. **MainActivity,** como su propio nombre indica, es la actividad principal de la aplicación.

El formulario en el que el usuario introduce las coordenadas está definido en la función **LocationForm:**

```kotlin
@Composable
fun LocationForm(modifier: Modifier = Modifier, activity: MainActivity, onStart: (Double, Double) -> Unit, onStop: () -> Unit) {
    var longitud by remember { mutableStateOf("") }
    var latitud by remember { mutableStateOf("") }
    Column (
        modifier = Modifier.padding(25.dp).fillMaxWidth(),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(
            text = "Coordenadas:",
            modifier = modifier
        )
        
        OutlinedTextField(
            value = latitud,
            onValueChange = {latitud = it},
            label = { Text("Latitud") }
        )
        OutlinedTextField(
            value = longitud,
            onValueChange = {longitud = it},
            label = { Text("Longitud") }
        )
        Row {
            OutlinedButton(onClick = {
                val lat = latitud.toDoubleOrNull()
                val lon = longitud.toDoubleOrNull()
                if (lat != null && lon != null) {
                    // Iniciamos mock
                    onStart(lat, lon)
                }
            }) {
                Text("Spoof!")
            }
            Spacer(Modifier.width(20.dp))
            OutlinedButton(onClick = { onStop() }) { // Parar mock
                Text("Stop")
            }
        }
    }
```

Las coordenadas que introducimos en los campos de texto, se almacenan en las variables **latitud** y **longitud**, que posteriormente son pasadas como argumento a la función **onStart().**

Esta función llama a **startMocking()** para que inicie el servicio.

### 4. AndroidManifest.xml

Cualquier aplicación Android, requiere unos permisos concretos. Estos permisos los tenemos que contemplar en el archivo **AndroidManifest.xml:**

```
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />
<uses-permission
  android:name="android.permission.ACCESS_MOCK_LOCATION"
  tools:ignore="MockLocation,ProtectedPermissions" />
```

### 5. Flujo de ejecución

Aunque aquí solo he mencionado las partes más importantes de la aplicación, realmente intervienen muchas más funciones.

Cuando pulsamos el botón **Spoof!:**

1.  Se ejecuta la función lambda **onStart()** que a su vez ejecuta la función **startMocking()**
2.  **startMocking()** recibe como parámetros la latitud y la longitud, y se los pasa a la función **startMockingLocation()** del servicio.
3.  **startMockingLocation()** ejecuta **mockLocation()**
4.  Es aquí donde se crea el _test provider_ con **addTestProvider()** y posteriormente se inicia el bucle while para falsificar las ubicaciones
