# Ensamblaje De novo de transcriptomas.
		$ ssh bioinfo1@genoma.med.uchile.cl
		bioinfo12016

El ensamblaje de novo para análisis de RNA-Seq, nos permite estudiar transcriptomas sin la necesidad de una secuencia genómica de referencia. En este protocolo, se describe el uso de la plataforma Trinity para el ensamblaje de transcriptoma a partir de datos de RNA-Seq.

<img src="https://user-images.githubusercontent.com/37847170/56938631-fc4a9980-6ad0-11e9-86a1-0f2c019bded4.png" width=500 />

<img src="https://user-images.githubusercontent.com/37847170/56938736-bfcb6d80-6ad1-11e9-8350-4d8890e24fc6.png" width=500/>

A continuación realizaremos los siguientes análisis. 

* Generación de un ensamble De novo utilizando Trinity
* Inspeción del ensamble con respecto al genoma de referencia.
* Mapeo de lecturas y transcritos (obtenidos por Trinity) a un genoma de referencia.
* Visualización de las lecturas y contigs alineados.

        $ cd /shared/bioinfo1/Ensamble_Transcriptoma_Tutorial/RNASeq_Trinity
        $ Trinity
        
Para más información, visitar: <http://trinityrnaseq.github.io>

### Datos.

Los datos de RNA-Seq de este práctico corresponden a la levadura Schizosaccharomyces pombe (fission yeast). Se utilizó una librería paired-end a partir de cDNA de 4 muestras:  Sp_log (logarithmic growth), Sp_plat (plateau phase), Sp_hs (heat shock), and Sp_ds (diauxic shift). 

Los datos 'left.fq' y 'right.fq' son archivos FASTQ pareados (Illlumina) para las 4 muestras.  Estos archivos de RNA-Seq, se encuentran en el siguiente subdirectorio. **/RNASeq_Trinity/RNASEQ_data/**.

Tambien se incluye el archivo 'genome.fa', corresponiente al las secuencias del genoma, y los archivos anotación de los genes de referencia ('genes.bed' o 'genes.gff3'). Estos, se encuentran en el siguiente subdirectorio **/RNASeq_Trinity/GENOME_data/**.  


## Ensamble De novo utilizando Trinity

Para generar un ensamble de referencia que luego puede ser utilizado para análisis de expresión diferencial, combinaremos los reads de Schizosaccharomyces pombe de las 4 muestras, en una sola muestras para el ensamble con Trinity. Realizaremos esto con una lista de todos los fastq de inputs pareados delimitados por coma, como en siguiente ejemplo:

       $ Trinity --seqType fq --SS_lib_type RF --left RNASEQ_data/Sp_log.left.fq.gz,RNASEQ_data/Sp_hs.left.fq.gz,RNASEQ_data/Sp_ds.left.fq.gz,RNASEQ_data/Sp_plat.left.fq.gz --right RNASEQ_data/Sp_log.right.fq.gz,RNASEQ_data/Sp_hs.right.fq.gz,RNASEQ_data/Sp_ds.right.fq.gz,RNASEQ_data/Sp_plat.right.fq.gz --CPU 10 --max_memory 1G --no_salmon

La ejecución de Trinity en este conjunto de datos puede tardar entre 10 y 15 minutos. Lo verás progresar a través de varias etapas, comenzando con Jellyfish para generar el catálogo k-mer, luego seguido por Inchworm, Chrysalis, y finalmente Butterfly. La ejecución de un trabajo típico de Trinity requiere ~ 1 hora y ~ 1G RAM por ~ 1 millón de lecturas de PE.

El ensamble de transcriptoma se encuentra en el siguiente archivo de salida. ‘trinity_out_dir/Trinity.fasta’.

Para ver las primeras lineas del archivo de salida, puedes utilizar el siguiente comando:

     $   head trinity_out_dir/Trinity.fasta

    >TRINITY_DN0_c0_g1_i1 len=1894 path=[1872:0-1893] [-1, 1872, -2]
    GTATACTGAGGTTTATTGCCTGTAACGGGCAAACTCGAGCAGTTCAATCACGAGGAGATT
    ACCAGAAAACACTCGCTATTGCTTTGAAAAAGTTTAGCCTTGAAGATGCTTCAAAATTCA
    TTGTATGCGTTTCACAGAGTAGTCGAATTAAGCTGATTACTGAAGAAGAATTTAAACAAA
    TTTGTTTTAATTCATCTTCACCGGAACGCGACAGGTTAATTATTGTGCCAAAAGAAAAGC
    CTTGTCCATCGTTTGAAGACCTCCGCCGTTCTTGGGAGATTGAATTGGCTCAACCGGCAG
    CATTATCATCACAGTCCTCCCTTTCTCCTAAACTTTCCTCTGTTCTTCCCACGAGCACTC
    AGAAACGAAGTGTCCGCTCAAATAATGCGAAACCATTTGAATCCTACCAGCGACCTCCTA
    GCGAGCTTATTAATTCTAGAATTTCCGATTTTTTCCCCGATCATCAACCAAAGCTACTGG
    AAAAAACAATATCAAACTCTCTTCGTAGAAACCTTAGCATACGTACGTCTCAAGGGCACA


## Examinar las estadísticas del ensamble

Algunas estadísticas básicas del ensamble de Trinity:

     $ /opt/anaconda3/bin/TrinityStats.pl trinity_out_dir/Trinity.fasta

Se observan los siguentes datos. Tengan en cuenta que sus números pueden variar ligeramente, ya que los resultados del ensamble no son deterministas.

    ################################
    ## Counts of transcripts, etc.
    ################################
    Total trinity 'genes':	377
    Total trinity transcripts:	384
    Percent GC: 38.66
    
    ########################################
    Stats based on ALL transcript contigs:
    ########################################
     
    Contig N10: 3373
    Contig N20: 2605
    Contig N30: 2219
    Contig N40: 1936
    Contig N50: 1703
    
    Median contig length: 772 
    Average contig: 1047.80
    Total assembled bases: 402355
   
    
    #####################################################
    ## Stats based on ONLY LONGEST ISOFORM per 'GENE':
    #####################################################
    
    Contig N10: 3373
    Contig N20: 2605
    Contig N30: 2216
    Contig N40: 1936
    Contig N50: 1695
    
    Median contig length: 772
    Average contig: 1041.98
    Total assembled bases: 392826



## Compare el transcriptoma reconstruido de novo con las anotaciones de referencia 

### a. Alinear los transcritos al genoma utilizando GMAP

Primero, prepare la región genómica para la alineación mediante GMAP de la siguiente manera:

    $ gmap_build -d genome -D . -k 13 GENOME_data/genome.fa

Ahora, debemos alinear los transcriptos de Trinity con el genoma, obtenemos como resultado un archivo en formato SAM, lo que simplificará la visualización de los datos.

    $ gmap -n 0 -D . -d genome trinity_out_dir/Trinity.fasta -f samse > trinity_gmap.sam

Tenga en cuenta que es probable que encuentre mensajes de advertencia como "No se encontraron rutas para comp42_c0_seq1", lo que significa que GMAP no pudo encontrar una alineación de alta puntuación de ese transcrito con las secuencias del genoma.

Convierta a un formato BAM (sam binario) ordenado por coordenadas de la siguiente manera:

    $ samtools view -Sb trinity_gmap.sam > trinity_gmap.bam

Ordene los reads del archivo bam por coordenadas:

    $ samtools sort -o trinity_gmap -O BAM trinity_gmap.bam

Ahora indexe el archivo bam para poder visualizar los datos en IGV.

    $ samtools index trinity_gmap

### b.  Alinear los reads de RNA-seq al genoma utilizando Tophat

A continuación, se alinea el conjunto de lecturas combinadas con el genoma, para que podamos ver cómo los datos de entrada coinciden con los contigs ensamblados por Trinity.

Preparamos el genoma para ejecutar tophat2

    $ bowtie2-build /GENOME_data/genome.fa genome 

Ahora, ejecutamos tophat con el siguiente comando.

	  $ cd /shared/bioinfo1/Ensamble_Transcriptoma_Tutorial/RNASeq_Trinity/
     $ tophat2 -I 300 -i 20 genome \
         /RNASEQ_data/Sp_log.left.fq.gz,/RNASEQ_data/Sp_hs.left.fq.gz,/RNASEQ_data/Sp_ds.left.fq.gz,/RNASEQ_data/Sp_plat.left.fq.gz \
         /RNASEQ_data/Sp_log.right.fq.gz,/RNASEQ_data/Sp_hs.right.fq.gz,/RNASEQ_data/Sp_ds.right.fq.gz,/RNASEQ_data/Sp_plat.right.fq.gz

Indexe el archivo bam obtenido por tophat para la vizualización en IGV:

    $ samtools index tophat_out/accepted_hits.bam

### c. Visualice todos los datos obtenidos utilizando IGV.

    % igv.sh -g `pwd`/GENOME_data/genome.fa `pwd`/GENOME_data/genes.bed,`pwd`/tophat_out/accepted_hits.bam,`pwd`/trinity_gmap.bam

<img src="https://raw.githubusercontent.com/wiki/trinityrnaseq/RNASeq_Trinity_Tuxedo_Workshop/images/TrinityWorkshop/IGV_trinity_and_reads.png" width=450 />

¿Trinity reconstruye total o parcialmente el transcriptoma de referencia de Schizosaccharomyces pombe, se producen estructuras correctas según la alineación con el genoma?

¿Hay ejemplos en los que el ensamblaje de novo resuelva los intrones que no se resolvieron de manera similar mediante las alineaciones de las lecturas cortas y viceversa?
