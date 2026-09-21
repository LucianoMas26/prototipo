# Monumento a la Bandera — visor 3D

Prototipo en un único archivo HTML que reconstruye en 3D la zona del Monumento
Nacional a la Bandera (Rosario, Argentina) combinando elevación real con datos
vectoriales de OpenStreetMap, y le superpone una capa de deterioro
post-apocalíptico.

```bash
python -m http.server 8080      # o: npx serve .
# http://localhost:8080/monumento-bandera-3d.html
```

Hace falta un servidor: los `import` de Three.js no funcionan bajo `file://`.
No hay que instalar nada — Three.js se carga por CDN vía import map.

## Qué hace

**Terreno.** Descarga teselas de relieve Mapzen Terrarium, decodifica la altura
con la fórmula oficial `(R * 256 + G + B / 256) - 32768` y desplaza los vértices
de un `PlaneGeometry` de 512×512. El DEM se trata en dos escalas (un radio corto
que quita el ruido de restitución y uno largo que define el relieve de gran
escala) porque SRTM es un modelo de *superficie*: incluye los techos de los
edificios, que en una ciudad plana como Rosario aparecen como terreno bacheado.

**Datos OSM.** Consulta Overpass y, si se cae, la API principal de OSM como
respaldo sobre infraestructura distinta. Extrae edificios, `building:part`,
parques, agua, muelles, estacionamientos y toda la red de circulación.

**Volúmenes.** Extrusión de cada huella según `height` o `building:levels`,
respetando la especificación *Simple 3D Buildings* (si un edificio tiene partes,
su contorno no se dibuja) y el color real cuando OSM trae `building:colour`.

**Superficies.** Parques, agua y predios se teselan y se drapean sobre el
terreno. Las vías se dibujan como cintas con *casing*, en capas escalonadas:
calzada, senda, vereda.

**Terrazas.** Cada `highway=steps` es evidencia de que las dos zonas que conecta
están a distinta altura. Se resuelve como reconstrucción de gradiente sobre una
grilla: a las aristas que cruzan el frente de una terraza se les pide el salto,
al resto diferencia cero, y se relaja el campo. El resultado son mesetas, no
lomas locales.

**Nivelación de calzadas.** El terreno se nivela bajo la red de circulación, en
vez de apoyar las calles encima. Cada vía tiene sección transversal horizontal y
rasante suavizada, y el terreno se lleva a esa rasante en tres pasadas.

**Deterioro.** Sedimento acumulado según manchones de ruido, pendiente (en un
talud el material no se queda) y proximidad a muros. Donde la capa supera el
espesor del solado, el suelo se traga la calle; donde no llega, la calle asoma.
Encima, maleza instanciada sembrada solo donde hay sedimento, con vaivén
resuelto en el vertex shader.

## Controles

| | |
|---|---|
| Terrazas / Alcance | desnivel entre zonas conectadas por escaleras |
| Maleza / Altura | densidad y porte de las matas |
| Sedimento | espesor de la capa que tapa el solado |
| Aplanado | cuánto detalle del DEM sobrevive |
| Exag. vertical | multiplicador de alturas |

`R` reencuadra · `W` alterna la malla del terreno.

## Límites conocidos

- **El Monumento no existe en 3D en OSM.** La relación `6804977` solo tiene
  `historic=monument`, `tourism=attraction` y `wikidata`; ni `building`, ni
  `height`, ni `building:part`. Lo que se ve alrededor son otros edificios.
- **Las escaleras no traen altura.** De las 38 del bbox, ninguna tiene `height`
  ni `ele`, y ningún nodo tiene cota. Lo único aprovechable es `step_count`
  (31 de 38) e `incline` (37 de 38): la altura se deduce multiplicando el
  conteo por una contrahuella supuesta.
- **El río Paraná no se dibuja.** Su relación tiene 120 miembros y 119 caen
  fuera del bbox, así que llega como una cadena abierta, no como un polígono.
  Dibujarlo requiere recortar contra el bbox.
- Los escalones no se modelan en 3D: se dibujan como una senda con marcas
  transversales. Una huella mide ~0,30 m y la malla tiene un vértice cada
  2,5 m — no alcanza para representarlos.

## Datos

Mapzen Terrarium (AWS) y OpenStreetMap, © colaboradores de OSM, ODbL.
