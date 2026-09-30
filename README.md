# TripTogether

App para organizar un viaje con amigos: ver dónde está cada uno en un mapa, sacar fotos del viaje y armar un calendario de actividades compartido.

Proyecto de la materia **Desarrollo de Apps**, Escuela Secundaria PROA, Corral de Bustos - Ifflinger.

**Autores:** Joaquín Trejo, Ian Acosta y Facundo Rodríguez.

## Qué hace

La app tiene tres pantallas:

### 🗺️ Mapa
Muestra tu ubicación real usando la **Geolocation API** del navegador, sobre un mapa de **OpenStreetMap** (vía Leaflet). Desde ahí podés agregar amigos al grupo; como no hay un servidor que reciba la ubicación de otros dispositivos en tiempo real, sus posiciones se simulan cerca de la tuya a modo de demostración.

### 📸 Fotos
Abre la cámara del dispositivo con la **MediaDevices API** (`getUserMedia`) y permite sacar fotos que quedan guardadas en una galería dentro de la app.

### 📅 Calendario
Actividades del viaje guardadas con una **API de almacenamiento compartido**: si otra persona abre la misma app, ve el mismo calendario. Así se simula cómo una app se conecta a datos que viven afuera de ella.

## Por qué usamos APIs

Esta app es, ante todo, un ejercicio sobre el uso de APIs:

| Función | API usada | Tipo |
|---|---|---|
| Ver tu ubicación | Geolocation API | Hardware del dispositivo |
| Mostrar el mapa | Leaflet + OpenStreetMap | Servicio externo |
| Sacar fotos | MediaDevices API | Hardware del dispositivo |
| Guardar el calendario | API de almacenamiento | Servicio externo |

En vez de programar desde cero cómo acceder al GPS, a la cámara o a un mapa, la app le pide esa función a una API ya existente y confiable.

## Tecnología

- React
- [Leaflet](https://leafletjs.com/) + OpenStreetMap
- [lucide-react](https://lucide.dev/) (íconos)

## Estado del proyecto

Prototipo funcional con fines educativos. La ubicación de amigos y la sincronización multiusuario del calendario son simulaciones pensadas para mostrar el concepto, no una implementación con backend propio.
