# Herramienta de Creacion de Mundos Procedurales
![wrld](https://github.com/javixxu/Procedural-Worlds/assets/82498887/a1f22ca9-65ca-492e-8819-30e4577fa9c4)

## Introduccion
La creacion de un mundo procedural completamente parametrizada con sistema de capas de terreno y creacion de objectos a lo largo de este.Ademas dicho terreno
generado ha de estar optimizado para que se pueda llegar a generar una gram superfice de terreno.

Como la creacion de un mundo completamente "aleatorio" puede resultar algo compleja se ha decidido establecer los siguientes parametros para que el usuario
pueda modificar a su antojo para crear el mundo con el terreno y objectos que desee):
  - int mapSize: Tamaño del Mapa
  - float NoiseScale: El factor de escala del ruido generado. Un valor mayor producirá un ruido con detalles más finos
  - int Octaves:  El número de octavas utilizadas en el algoritmo de ruido.Cada octava es una capa de ruido que se suma al resultado final.
    A medida que se agregan más octavas, el ruido generado se vuelve más detallado
  - float Persistance: Controla la amplitud de cada octava.Un valor más bajo reducirá el efecto de las octavas posteriores de las octavas posteriores
  - float Lacunarity: Un multiplicador que determina qué tan rápido aumenta la frecuencia para cada octava sucesiva en una función de ruido de Perlin
  - int Seed: Numero aleatorio utilizado para generar el ruido
  - Vector2 Offset: La posición inicial del ruido generado
  - TerrainType[]regions: Layers del terreno que se pueden generar
  - ObjectInMap[]objects: Objectos que se pueden generar a lo largo del mapa
  - bool useFallOff: Generar un suavizado en el ruido generado de tal forma que el mapa tenga forma de isla
  - bool autoUpdate: Cuando se realize un cambio des de el editor, auto actualizar el mapa
  - bool autoRegenerate: Cuando se inicilize este componente autoregenerar el terreno


Establecidos todos estos parametros tendremos varias posibilidades, tipos de configuracion para la generacion. Es posible que nosotros queramos previsualizar el mapa que estemos generando en 2D antes que 3D para ello,
tendremos varios modos de creacion:
  - NoiseMap: Generacion de un Mapa de Con los layers de terreno establecidos(Solo visual 2D)
  - ColorMap: Generacion de un Mapa de Ruido con  los bordes del terreno suavizados(Solo visual 2D)
  - FallOff: Generacion de un Mapa de Con los layers de terreno establecidos(Solo visual 3D)
  - NoObjects: Generacion de un Mapa de Con los layers de terreno establecidos y los Objectos puestos para generar(Solo visual 3D)
  - Objects: Generacion de un Mapa de Con los layers de terreno establecidos y los Objectos puestos para generar(Solo visual 3D)
  - NobjectsWithDisplay: Configuracion de ColorMap y NoObjects     
  - All: Configuracion de ColorMap y Objects
En el inspector de unity, existe un boton para generar el mapa.


## Diseño 
La Herramienta posee varios "elementos basicos" que facilitan la creacion del mundo:

  - Class Chunk: Un chunk es una porcion 50x50 del mapa generado, esto se debe a que unity no deja crear mas de 65,534 por ello es necesario dividir la malla del mapa en otras mas pequeñas que que todas estas juntas         conformen el mapa.
  
      - Un GameObject Padre con el nombre de Chunk + posMap que contiene a la malla del suelo, bordes y un gameobject que contiene todos los objectos generados en ese chunk.
      -PosMap representa la posicion del chunk en el mapa 

  - Struct TerrainType: Es una capa de terreno que se puede generar. Esta contiene:
  
      - string Layer: Nombre Capa De Terreno
      - height: Altura a la que se puede generar como maximo
      - color: color que va a tener las casillas si se genera un punto de ruido que no supere la altura maxima
  - class ObjectInMap: Son objectsoq ue se pueden generar a lo largo del mapa. Esta Contiene:
  
      - GameObject prefab
      - float Density: Densidad del objecto
      - float NoiseScale: El factor de escala del ruido generado
      - string GenerationLayer: Capa en la que se puede generar el Objecto
  - class Cell: Casilla del Mapa que guarda toda la informacion generada por el ruido de perlin, posteriormente evaluada con las Layers del terreno y 
    los objectos que pueden generarse en ella. Esta contiene:
    
      - TerrainType type: Tipo de terreno
      - float Noise: Ruido genrado en esa casilla
      - float Height: Altura de esa casilla
      - GameObject objectGenerated: Objecto q se ha podido generar encima de la casilla

La Herramienta constara de varios Generadores:

  - Noise: Es el generador de Ruidos de Perlin.Constara de dos Grandes metodos: 
  
      - GenerateNoiseMap(int size,int seed,float noiseScale,int octaves, float persistance, float lacunarity,Vector2 offset) que generara un ruido a partir de los parametros de entrada, 
      devolviendo una matriz de floats con el valor de ruido generado en cada casilla.
      
      - GenerateFalloffMap(int size): Genera un mapa de fallOff para suavizar los bordes del terreno usando esta ecuacion para ello 
  ![image](https://github.com/IAV23-G07/IAV23-G07-ProyectoFinal/assets/82498887/efada9ad-700b-494a-8676-2d9b27243b7f) 

        Siendo b=2.2 y a=3.De esta forma simulamos una especie de isla combinando la matriz que obtenemos del falloff y la matriz de ruido obtenida en el metodo GenerateNoiseMap
  - TextureGenerator: Es una clase para la creacion de una textura, que servira para la representacion del mapa generado. Constara de dos metodos:
       - TextureFromColorMap(Color[] colorMap,int width,int height): crea una textura y le aplica a cada texel de la texura el valor del array color, creando asi una textura a color.
       - TextureFromNoiseMap(float[,] noiseMap): a su misma vez que el metodo anteior este crea una textura para la representacion del mapa creado pero esta vez en blanco y negro interpolando dichos colores con el               noiseMap que contiene el noise de cada celda
  - MeshGenerator: Es el generador de la malla en 3D, como queremos que la malla que estemos creado sea lo mas eficiente posible solo crearemos las caras visibles, pues el mapa generado esta pensado para que no se le          puedan quitar bloques,trozos, etc. Para ello se ha divido en dos partes, el suelo y los bordes.En ambos hay que definir los vertices, triangulos y coordenadas de textura. Como se ha explicado anteriormente se crean 
     mallas para cada chunk para no exceder el limite de vertices establecido por unity. Constara de tres metodos:
     
      - GenerateTerrainMeshChunk(Cell[,] mapaCells, Vector2 pos, GameObject chunkObject, float sizePerBlock, int chunkSize): Genera el suelo de un Chunk
      - DrawEdgesChunk(Cell[,] mapaCells, Vector2 pos, GameObject edges,float sizePerBlock,int chunkSize): Genera los bordes de un Chunk
      - DrawTextureChunk(Cell[,] mapaCells, Vector2 pos, MeshRenderer renderer, int chunkSize): Crea una textura estableciendo el color de cada texel correspondiente a el color de cada tipo de terreno que hay en cada
        Cell.
  -ObjectsGenerator: Esta clase se encarga de generar proceduralmete objectos por el mapa.
  Esta clase contiene el metodo   GenerateObjects(int mapSize, int chunkSize,float heightPerBlock, Cell[,] cellMap, Dictionary<Vector2,Chunk> chunks, ObjectInMap[] objectsToGenerate), el cual para cada casilla si puede     generar un objecto y recorre el array objectsToGenerate por orden de densidad intentando generar objectos para esa casilla.
  
 El script MapDisplay es el encargado de dibujar el map2D si ha asi ha querido el usuario
 
 Por ultimo Tendremos la clase MapGenerator, que sera la encarga de "Hacer Todo".A su vez hay un script MapGeneratorEditor para facilitar el uso al usuario. Esta tendra todos los parametros indicados en la propuesta y     sera la encargada de llamar a los distintos metods explicados anteriormente     para la generacion del mapa. Esta contendra los siguientes metodos:
 
  - GenerateMap: Genera el mapa con los parametros establecidos
  - GenerateMapByChunks: Es llamado por el metodo anterior y es elencargado de crear los chunks o modificarlos segun sea necesario
  
 #### Metricas:
 
  - A: El mapa generado se puede representar en 2D {A color, el ruido de perlin y el falloff}
  - B: El mapa 3D generado, esta completamente optimizado de manera que la mmala creada es como un quad y solo se renderiza por la superficie, ademas se que solo se crean los vertices unicos y necesarios para la                representacion del mapa. Cuenta con un sistema de chunks que permite identificar entranbdo al gameobject del chunk todo lo que hay en su superficie.Crear un suavizado de bordes para que el terreno generado tenga          forma de isla.
  - C: El mapa generado se puede representar en 3D{Con objectos, Sin Objectos}
  - D : El mapa es completamente parametrizable,el ususario es capaz de determinar: el tamaño del mapa, el numero de octavas , el noisecale, la amplitud de cada octava("persistance"), la freciencia de cada                       octava("lacunarity"), mundos completamente unicos con un entero que el usiario determina("seed"), el offset del ruido y los objectos que quieres generar en el mapa.
  - E: El sistema cuenta con 3 booleandos, uno para regenrar automaticamente el mapa cuando se cambien parametros desde el editor("AutoUpdate"), otro para autoregenerar el mapa cuando entras en ejecucion y por ultimo
       un boleano para ver si kieres aplicar el suavizado de bordes("fallOff").

## Referencias
Los recursos de terceros utilizados son de uso público:
- Arboles y rocas: https://assetstore.unity.com/packages/3d/environments/landscapes/low-poly-simple-nature-pack-162153
- API Unity
- Libro de Milinton
