Documentation of Command-line tools used for CAG-170 Analysis 
=========================
Commands were run on the University of Cambridge HPC. Paths and account identifiers are shown as placeholders:
<username>       --> your HPC login (CRSid)
<project-id>     --> your RDS project storage identifier
<slurm-account>  --> your SLURM billing account

Genome completeness and contamination metrics (CheckM v1.0.11) were pre-computed and sourced from genome catalogues and research papers, not run locally.


================================================================================
1. Quality Control
================================================================================

1.1. GUNC v1.0.3 (ProGenomes 2.1 database) - chimeric genome detection
----------------------------------------------
Run separately on each genome subset with identical parameters.
Results were subsequently combined.

gunc run -d gtdb_genomes -e .fa -t 16 -o gunc_out/gtdb -r ~/rds/<project-id>/rfs_data/databases/gunc/gunc_db_progenomes2.1.dmnd

gunc run -d mgnify_genomes_A -e .fa -t 16 -o gunc_out/mgnify_A -r ~/rds/<project-id>/rfs_data/databases/gunc/gunc_db_progenomes2.1.dmnd

gunc run -d mgnify_genomes_B -e .fa -t 16 -o gunc_out/mgnify_B -r ~/rds/<project-id>/rfs_data/databases/gunc/gunc_db_progenomes2.1.dmnd

gunc run -d mgnify_genomes_B -e .fa -t 16 -o gunc_out/pigs -r ~/rds/<project-id>/rfs_data/databases/gunc/gunc_db_progenomes2.1.dmnd

gunc run -i GCA_916988195.3_POC01_20211130.fa -o gunc/gunc_out/isolate -r ~/rds/r<project-id>/rfs_data/databases/gunc/gunc_db_progenomes2.1.dmnd


================================================================================
2. Dereplication
================================================================================

2.1. dRep v3.4.3 - genome dereplication
-----------------------------------------

# Removing identical genomes (ANI >= 1.000) - 562 genomes
dRep dereplicate drep/drep_100 --SkipSecondary -pa 1.0 -g drep_genomes/*fa --ignoreGenomeQuality

# Strain-level analysis (ANI >= 0.995) - 508 genomes
dRep dereplicate drep/drep_995 --SkipSecondary -pa 0.995 -g drep_genomes/*fa --ignoreGenomeQuality


================================================================================
3. Phylogenetic Analysis
================================================================================

3.1. GTDB-Tk v2.3.2 (GTDB release R10-RS226 database) - marker gene identification
--------------------------------------------------
Run separately on each genome subset due to dataset size.
Identify outputs were merged manually before alignment.

gtdbtk identify --genome_dir mgnify_isolate_pigs --out_dir gtdbtk/gtdbtk_identify/mgnify_isolate_pigs -x fa --cpus 8 --force --tmpdir gtdbtk/tmp

gtdbtk identify --genome_dir gtdb_genomes --out_dir ggtdbtk/gtdbtk_identify/gtdb_genomes -x fa --cpus 8 --force --tmpdir gtdbtk/tmp


3.2. GTDB-Tk v2.3.2 - marker gene alignment
---------------------------------------------
Run on merged identify outputs (1,490 genomes).

gtdbtk align --identify_dir gtdbtk/gtdbtk_identify/merged --out_dir gtdbtk/gtdbtk_align --cpus 8 --skip_gtdb_refs --tmpdir gtdbtk/tmp


3.3. FastTree v2.1.11 - phylogenetic tree inference (Bac120)
-------------------------------------------------------------

# Whole dataset (1,490 genomes)
FastTree -lg gtdbtk/gtdbtk_align/align/gtdbtk.bac120.user_msa.fasta > gtdbtk/cag170_1490_tree.nwk

# Filtered (high quality, dereplicated) subset (562 genomes)
FastTree -lg gtdbtk/gtdbtk_align/align/gtdbtk.bac120.user_msa.filtered562.fasta > gtdbtk/cag170_562_tree.nwk


================================================================================
4. Genome Annotation
================================================================================

4.1. Prokka v1.14.6 - gene prediction and annotation
------------------------------------------------------
Run as SLURM array on filtered subset.

Submission command:
sbatch --array=1-562%50 prokka_array.sh prokka_input_drep100.txt

Array script (prokka_array.sh):

#!/bin/bash
#SBATCH -p icelake
#SBATCH -J prokka_array
#SBATCH -o /rds/project/<project-id>/<username>/logs/prokka_%A_%a.log
#SBATCH -c 1
#SBATCH --mem=5G
#SBATCH -A <slurm-account>
#SBATCH --time=01:00:00

file_list=$1
file=$(awk "NR==$SLURM_ARRAY_TASK_ID" "$file_list")
base=$(basename "$file" .fa)

prokka --cpus 1 ${file} --outdir prokka/${base} --prefix ${base} --force --locustag ${base} --rfam


================================================================================
5. Pangenome Analysis
================================================================================

5.1. Panaroo v1.3.3 - Filtered subset pangenome construction 
---------------------------------------------------------------

panaroo -t 8 -i panaroo_gffs.txt -o 5panaroo/all562 --clean-mode strict --remove-invalid-genes -c 0.9 -f 0.5 --merge_paralogs --core_threshold 0.9 -a core


5.2. Panaroo v1.3.3 - species-level pangenome construction
------------------------------------------------

# sp002404795 (n=117)
panaroo -t 8 -i sp002404795_gff_list.txt -o 5panaroo/panaroo_sp002404795 --clean-mode strict --remove-invalid-genes -c 0.9 -f 0.5 --merge_paralogs --core_threshold 0.9 -a core


================================================================================
6. Genome-Wide Association Study
================================================================================

6.1. pyseer v1.3.11 - kinship matrix generation
-------------------------------------------------
With --llm (similarity matrix)
python phylogeny_distance.py --lmm tree_filtered.nwk > phylogeny_K.tsv

(Distance matrix)
python phylogeny_distance.py tree_filtered.nwk > phylogeny_distances_562.tsv

6.2. pyseer v1.3.11 - GWAS
----------------------------
# Human vs non-human, no species covariate
pyseer --lmm --phenotypes phenotype_human.tsv --pres gene_presence_absence_filtered.Rtab --similarity phylogeny_K.tsv --cpu 4 > pyseer_human_results.tsv

# Human vs non-human, species covariate
pyseer --lmm --phenotypes phenotype_human.tsv --pres gene_presence_absence_filtered.Rtab --similarity phylogeny_K.tsv --covariates covariates_species.tsv --use-covariates 2 --cpu 4 > pyseer_human_covar_results.tsv

# sp002404795 - human vs non-human
pyseer --lmm --phenotypes phenotype_human.tsv --pres 5panaroo/panaroo_sp002404795/gene_presence_absence_sp002404795.Rtab --similarity phylogeny_K.tsv --cpu 4 > pyseer_sp002404795_human_results.tsv

================================================================================
7. Functional annotation
================================================================================

7.1 eggNOG-mapper v2.1.3 (eggNOG v5.0.2 database) - eggNOG annotation

# For 562 pangenome
mamba run -n genofan emapper.py --cpu 8 -i 5panaroo/panaroo_all562/pan_genome_reference.faa -m diamond -o eggnog_filtered --output_dir 5panaroo/panaroo_all562/eggnog --temp_dir 5panaroo/panaroo_all562/eggnog --data_dir /rds/project/<project-id>/rfs_data/databases/eggnog --override

# For sp002404795 pangenome
mamba run -n genofan emapper.py --cpu 8 -i 5panaroo/panaroo_sp002404795/pan_genome_reference.faa -m diamond -o eggnog_sp002404795 --output_dir 5panaroo/panaroo_sp002404795/eggnog --temp_dir 5panaroo/panaroo_sp002404795/eggnog --data_dir /rds/project/<project-id>/rfs_data/databases/eggnog --override

