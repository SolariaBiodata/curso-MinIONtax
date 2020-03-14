---
permalink: /index.html
---
![alt text](https://solariabiodata.com.mx/images/solaria_banner.png "Soluciones de Siguiente Generación")
# Secuenciación Masiva y Análisis Taxonómico - Sesión de Análisis Bioinformático

## Sesión Práctica

### Descripción
En esta sesión aprenderemos lo necesario para evaluar la calidad de genotecas preparadas con la plataforma de Oxford Nanopore, teniendo a consideración los formatos de archivos y cómo poder generar nuestras propias asignaciones taxonómicas.

### Requisitos

Para poder realizar este ejercicio, necesitaremos:

1. Datos de secuencias:
    - Puedes usar tus propias secuencias, en formato FASTQ o FASTQ.GZ
2. Sofware Recomendable para esta sesión:
    - Terminal (Mac o Linux)
    - Putty (Windows)
    - WinSCP (Windows)

## Ejercicio 01:
### Descripción
A continuación, observaremos la estructura de un archivo de lecturas en formato FASTQ, el cual es utilizado de manera estándar en múltiples plataformas de secuenciación masiva.

### Instrucciones (Parte 01)
1. Para descomprimir los archivos en extensión FASTQ.GZ, usamos:
    ~~~
    gzip -d archivo.fastq.gz
    ~~~
    Este paso no es necesario en muchos procesamientos, sin embargo ayuda a familiarizarse con el contenido del archivo.
### Instrucciones (Parte 02)
1. Abre las primeras 16 líneas de tu archivo FASTQ favorito:
    ~~~
    head -16 archivo.fastq
    ~~~
2. Determina la lectura con el nucleótido con la calidad más pobre
3. Encuentra los patrones de nombre en los descriptores
4. Encuentra los nombres de los 10 primeros descriptores utilizando:
    ~~~
    grep "^@" archivo.fastq | head -10
    ~~~
5. Encuentra los nombres de los 10 primeros descriptores de calidad utilizando:
    ~~~
    grep "^+" archivo.fastq | head -10
    ~~~
6. Para saber el número de secuencias en el archivo de secuenciación:
    ~~~
    grep -c "^@" archivo.fastq
    ~~~

## Ejercicio 02: Ejecución de FASTQC
### Descripción
A pesar de que los equipos actuales de secuenciación prometen o despliegan secuencias que parecen de alta calidad, existen algunos factores que afectan la calidad del llamado de bases.

### Instrucciones
1. Si ejecutan fastqc sin argumentos se abre la versión gráfica
    ~~~
    fastqc reads.fastq
    ~~~

## Ejercicio 03: Asignación Taxonómica
### Descripción
Realizaremos un análisis taxonómico utilizando el software de Qiime2

### Instrucciones (Parte 01)
1. Activación de Qiime2 en la instancia
    ~~~
    conda activate qiime2
    ~~~
2. Crearemos una tabla de metadatos, con información sobre nuestras bibliotecas
    ~~~
    nano metadata.tsv
    ~~~
    La tabla puede ser parecido a esto:

      | #SampleID | BarcodeSequence | Treatment   | Group       | Year    | Concentration |
      |-----------|-----------------|-------------|-------------|---------|---------------|
      | #q2:types | categorical     | categorical | categorical | numeric | numeric       |
      | SAMPLE_A  | CAGTGTCA        | Cont_A      | Control     | 2010    | 20.0          |
      | SAMPLE_B  | CATTGTCA        | Cont_A      | Experiment  | 2012    | 25.0          |

3. Validación del archivo de metadatos:
    ~~~
    qiime tools inspect-metadata metadata.tsv
    ~~~
### Instrucciones (Parte 02)
1. Crearemos un archivo manifiesto de carga:
    ~~~
    nano manifest.csv
    ~~~
2. Una vez generado el archivo, cargaremos las secuencias con:
    ~~~
    qiime tools import --type 'SampleData[SequencesWithQuality]' \
    --input-path manifest.csv \
    --input-format SingleEndFastqManifestPhred33 \
    --output-path raw-seqs.qza
    ~~~
3. Desreplicación
    ~~~
    qiime vsearch dereplicate-sequences \
    --i-sequences raw-seqs.qza \
    --output-dir derep
    ~~~
4. Búsqueda de Quimeras
    ~~~
    qiime vsearch uchime-denovo \
    --i-table derep/dereplicated_table.qza \
    --i-sequences derep/dereplicated_sequences.qza \
    --output-dir chimeras
    ~~~
5. Remoción: Ahora deben conservarse únicamente las secuencias no-quiméricas
    ~~~
    qiime feature-table filter-features \
    --i-table derep/dereplicated_table.qza \
    --m-metadata-file chimeras/nonchimeras.qza \
    --o-filtered-table filtered-table.qza
    ~~~
    ~~~
    qiime feature-table filter-seqs \
    --i-table derep/dereplicated_sequences.qza \
    --m-metadata-file chimeras/nonchimeras.qza \
    --o-filtered-table filtered-seqs.qza
    ~~~
6. Clusterización: El proceso implica agrupar tantas secuencias se parezcan entre sí.
    ~~~
    qiime tools export --input-path chimeras/chimeras.qza \
    --output-path chimeras/
    ~~~
    ~~~
    qiime vsearch cluster-features-open-reference \
    --i-table filtered-table.qza \
    --i-sequences filtered-seqs.qza \
    --i-reference-sequences chimeras/chimeras.qza \
    --p-perc-identity 0.80 \
    --output-dir clustered
    ~~~
7. Asignación Taxonómica: Descargar la base de referencia
    ~~~
    wget https://data.qiime2.org/2019.1/common/gg-13-8-99-nb-classifier.qza
    ~~~
    ~~~
    mv gg-13-8-99-nb-classifier.qza gg13_16S.qza
    ~~~
8. Utilizaremos este comando para realizar la Asignación taxonómica
    ~~~
    qiime feature-classifier classify-sklearn \
    --i-classifier gg13_16S.qza \
    --i-reads clustered/clustered_sequences.qza \
    --o-classification taxonomy.qza
    ~~~
9. Utilizaremos el comando para generar un archivo conocido como feature table
    ~~~
    qiime taxa collapse \
    --i-table clustered/clustered-table.qza \
    --i-taxonomy taxonomy.qza \
    --p-level 6 \
    --output-dir results
    ~~~
10. Exportamos la tabla de asignación a formato BIOM
    ~~~
    qiime tools export --input-path results/collapsed_table.qza \
    --output-path export
    ~~~
11. Y el BIOM lo podemos exportar a tabla en texto plano
    ~~~
    biom convert -i results/feature-table.biom --to-tsv \
    -o results/feature-table.txt
    ~~~
### Instrucciones (Parte 03)
1. Exportamos a un formato de visualización
    ~~~
    qiime metadata tabulate \
    --m-input-file taxonomy.qza \
    --o-visualization vis/taxonomy.qzv
    ~~~
2. Exportamos una gráfica de barras
    ~~~
    qiime taxa barplot \
    --i-table clustered/clustered_table.qza \
    --i-taxonomy taxonomy.qza \
    --m-metadata-file metadata.tsv \
    --o-visualization vis/taxa-barplot.qzv
    ~~~
3. Exportamos el feature-table a el visualizador de Krona
    ~~~
    SolariaFeatureExporter results/feature-table.txt
    ~~~
4. Y subimos para ver al navegador Web
    ~~~
    sbcp muestra.html
    ~~~


## Ejercicio 04: Índices de Diversidad
### Descripción
La diversidad de especies de una comunidad se define como el número de especies en la comunidad, y en metagenómica, a menudo se estima utilizando el número de unidades taxonómicas operativas (OTU).   
### Instrucciones (Parte 01)
1. El esfuerzo de muestreo es posible observarse, utilizando rarefacciones (o subsets aleatorios) del total.
    ~~~
    qiime diversity alpha-rarefaction \
    --i-table clustered/clustered_table.qza \
    --p-max-depth 9998 --output-dir alpha-rarefaction
    ~~~
2. Para hacer el cálculo de diversidad individual
    ~~~
    mkdir alpha
    ~~~
    ~~~
    qiime diversity alpha \
    --i-table clustered/clustered_table.qza
    --p-metric shannon
    --o-alpha-diversity alpha/shannon.qza
    ~~~
3. Recuerden que los datos se pueden exportar a texto plano
    ~~~
    qiime tools export --input-path [artefacto].qza --output-path .
    ~~~
### Instrucciones (Parte 02)
4. El archivo de visualización resultante ayuda a explicar el índice entre el set de datos respecto a múltiples rarefacciones
    ~~~
    qiime diversity beta-rarefaction --p-metric braycurtis --p-clustering-method upgma --i-table clustered/clustered_table.qza --o-visualization beta-rarefaction.qzv
    ~~~
5. Para hacer el cálculo de diversidad beta individual
   ~~~
   mkdir beta
   ~~~
   ~~~
   qiime diversity beta \
   --i-table clustered/clustered_table.qza
   --p-metric euclidean
   --o-alpha-diversity beta/euclidean.qza
   ~~~
