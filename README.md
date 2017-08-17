# kinfin_manuscript

[![DOI](https://zenodo.org/badge/95892139.svg)](https://zenodo.org/badge/latestdoi/95892139)

Data for the KinFin manuscript

# Commands
```
# $KINFIN/: path to KinFin installation
# $MAFFT/: path to MAFFT installation
# $TRIMAL/: path to trimal installation
# $FASCONCAT/: path to FASconCAT installation
# $RAXML/: path to RAxML installation

# 1. Clone KinFin manuscript repo
git clone https://github.com/DRL/kinfin_manuscript
cd kinfin_manuscript/

# 2. Basic analysis
$KINFIN/kinfin \
    -g supplementary_data_1/Orthogroups.I1_5.txt \
    -c supplementary_data_1/kinfin.config.basic.txt \
    -s supplementary_data_1/kinfin.SequenceIDs.txt \
    -o kinfin.basic 
  
# 2.1 Get single-copy cluster IDs
grep 'true' kinfin.basic.kinfin_results/all/all.all.cluster_1to1s.txt | \
    cut -f1 > single_copy.cluster_ids.txt

# 2.2 Get protein IDs from single-copy cluster IDs
$KINFIN/scripts/get_protein_ids_from_cluster.py \
    -g supplementary_data_1/Orthogroups.I1_5.txt \
    --clusters single_copy.cluster_ids.txt

# 2.3 Extract FASTA sequences based on protein IDs
cat supplementary_data_2/fastas/*.faa > all.proteins.faa
parallel -j8 '\
    grep --no-group-separator -A1 -wFf {} all.proteins.faa > {/.}.faa\
    ' ::: Orthogroup.I1_5.OG*.txt
  
# 2.4 Align FASTA sequences
parallel -j 8 '\
    $MAFFT/bin/einsi {} > {/.}.einsi.faa" ::: Orthogroup*.faa
 
# 2.5 Trim alignments
parallel -j 8 '\
    $TRIMAL/source/trimal -in {} -out {/.}.trimal.faa -automated1 \
    ' ::: *einsi.faa

# 2.6 Sanitise headers for FASconCAT
parallel -j 8 '\
    cut -f1 -d"." {} > {/.}.sane.fas \
    ' ::: *trimal.faa

# 2.7 Concatenate alignments
$FASCONCAT/FASconCAT_v1.0.pl -s -p -n

# 2.8 RAxML
$RAXML/raxmlHPC-PTHREADS-SSE3 -s FcC_smatrix.phy -m PROTGAMMAGTR -T 32 -n ml -N 20 -p 19
$RAXML/raxmlHPC-PTHREADS-SSE3 -m PROTGAMMAGTR -T 48 -n bs -p 19 -b 19 -# 100 -s FcC_smatrix.phy
$RAXML/raxmlHPC-PTHREADS-SSE3 -m GTRCAT -p 19 -f b -t RAxML_bestTree.ml -z RAxML_bootstrap.bs -n final

# 3. Advanced analysis (assuming KinFin executable is in your $PATH)
$KINFIN/kinfin \
    -g supplementary_data_2/Orthogroups.I1_5.txt \
    -c supplementary_data_2/kinfin.config.tree.txt \
    -s supplementary_data_2/kinfin.SequenceIDs.txt \
    -o kinfin.advanced \
    -p supplementary_data_2/kinfin.SpeciesIDs.txt \
    -a supplementary_data_2/fastas/ \
    -t supplementary_data_2/kinfin.tree.nwk \
    -f supplementary_data_2/kinfin.functional_annotation.txt
  
# 4.1 Infer representative functional annotation (all clusters)
$KINFIN/scripts/filter_functional_annotation_of_clusters.py all \
    -f kinfin.advanced.kinfin_results/cluster_metrics_domains.IPR.txt \
    -c kinfin.advanced.kinfin_results/cluster_counts_by_taxon.txt \
    -x 0.75 \
    -p 0.75 \
    -o kinfin.IPR

# 4.2 Infer representative functional annotation (synapomorphic clusters)
$KINFIN/scripts/filter_functional_annotation_of_clusters.py synapo \
    -f kinfin.advanced.kinfin_results/cluster_metrics_domains.IPR.txt \
    -c kinfin.advanced.kinfin_results/cluster_counts_by_taxon.txt \
    -t kinfin.advanced.kinfin_results/tree/tree.cluster_metrics.txt \
    -n 0.75 \
    -x 0.75 \
    -p 0.75 \
    -o kinfin.IPR
```
