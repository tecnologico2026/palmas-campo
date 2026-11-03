# Palmas Campo — APK de polinización

APK **Android aparte** (`com.palmas.campo.dev`) que abre la plataforma Palmas en un WebView y le
suma lo que un navegador no puede dar: **GPS que sigue grabando con la pantalla apagada**.

Se instala junto a la app anterior sin reemplazarla — usa otro `applicationId` a propósito.

## Servidor

Apunta a **producción** de fábrica: el auxiliar instala, abre y trabaja, sin escribir ninguna
dirección. Para probar contra un Django local, **deja pulsado el ícono de la app → «Servidor»**,
escribe la dirección (ej. `192.168.1.50:8000`) y guarda; la app se reinicia sola. El botón
«Volver a producción» deshace el cambio.

Ese atajo no aparece en el cajón de aplicaciones: en campo nadie se lo topa por accidente.

> La URL se inyecta como `server.url` desde `MainActivity`, no navegando desde una página local.
> Es deliberado: en Android los plugins nativos **solo** funcionan con `server.url`; si se llega
> al servidor con `window.location`, el GPS de fondo deja de responder
> ([capacitor#4164](https://github.com/ionic-team/capacitor/issues/4164)).

## App de recorrido standalone (queda incluida)

`www/index.html` + `www/app.js` siguen en el APK y son el respaldo si la configuración del
servidor falla. Graban el recorrido y lo exportan a GPX sin tocar la plataforma:

- Campo **Código** (quién lleva el celular) + lista **cada cuánto toma un punto** (5/10/15/20 s).
- Botones **Iniciar mi día** / **Terminar mi día**.
- Graba el recorrido con el servicio de fondo de Android y lo dibuja sobre el mapa satélite,
  **sin rayas falsas** (bota fixes de mala precisión y corta la línea en los huecos).
- **Compartir recorrido**: exporta el día como archivo GPX etiquetado con el código.

## Instalar (una sola vez, en el celular)

1. Abre en el celular el link de descarga del APK (Releases → el último).
2. Android dirá que no se pueden instalar apps de esta fuente → **Permitir/Configurar** → activa
   "Permitir de esta fuente".
3. Instala y abre **Palmas Campo**.
4. La primera vez pedirá el **permiso de ubicación** → elige **"Permitir todo el tiempo"**
   (imprescindible para que grabe con la pantalla apagada).
5. **Con wifi, antes de salir al lote**, abre la app una vez: ahí es cuando guarda las pantallas
   para poder trabajar sin señal. Si se salta este paso, en campo aparece "No se pudo abrir la
   plataforma".

## Cómo está hecho

- [Capacitor 6](https://capacitorjs.com) envuelve el WebView y expone los plugins nativos.
- GPS de fondo: [`@capacitor-community/background-geolocation`](https://github.com/capacitor-community/background-geolocation)
  (servicio en primer plano con notificación fija).
- Mapa (app standalone): Leaflet + imágenes satélite de Esri.
- Offline de los formularios: lo aporta el Service Worker de la plataforma, no el APK.

## Compilar el APK

**A mano, en local** — el CI está escrito pero inactivo: el workflow vive en `build.yml.pending`
en la raíz porque el token de `gh` no tiene el scope `workflow`.

El proyecto compila **dos apps** (product flavors en `android/app/build.gradle`):

| Flavor | App | Id | Abre |
|---|---|---|---|
| `campo` | Palmas Campo | `com.palmas.campo.dev` | `/labores/m/polinizacion/menu/` |
| `recorridoAuxiliar` | Recorrido Auxiliar | `com.palmas.recorridoauxiliar` | `/labores/m/polinizacion/recorrido-auxiliar/` |

Las dos comparten el código (WebView + GPS de fondo + pantalla «Servidor»); cambian id, nombre,
dirección por defecto y marca del User-Agent (`PalmasCampoAPK` / `PalmasRecorridoAuxiliarAPK`).
Lo propio de Recorrido Auxiliar vive en `android/app/src/recorridoAuxiliar/`. Ids distintos:
conviven en el mismo celular.

```bash
npx cap sync android
cd android && ./gradlew assembleCampoDebug assembleRecorridoAuxiliarDebug
cp app/build/outputs/apk/campo/debug/app-campo-debug.apk ../palmas-campo.apk
cp app/build/outputs/apk/recorridoAuxiliar/debug/app-recorridoAuxiliar-debug.apk ../recorrido-auxiliar.apk
```

Para repartirlas en campo hay que firmarlas con la llave de release de la empresa
(`assembleCampoRelease` / `assembleRecorridoAuxiliarRelease` + `signingConfigs`).

Para activar el CI: `gh auth refresh -h github.com -s workflow`, luego mover
`build.yml.pending` a `.github/workflows/build.yml` y hacer push.
