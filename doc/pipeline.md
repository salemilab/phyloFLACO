
# RETRIEVE REFERENCE LINEAGES USED BY PANGOLIN


1. Initiate FLACO environment

	```
	source /blue/salemi/share/COVIDseq/bin/FLACO #Note: made alias "flaco"
	```
	
2. Download most up-to-date gisaid alignment (currently folder set up in /blue/salemi/brittany.rife/cov/gisaid_msa). 

	```
	cd <new/path>
	wget --user <username> --password <password> https://www.epicov.org/epi3/	msa_1111.tar.xvf
	tar -xvf msa*
	```


3. Create a file with dates for later referencing and a variable ("today") from the dates file to help with naming:
	
	```
	echo $(date +"%Y-%m-%d") > dates.txt
	today=$( tail -n 1 dates.txt )
	```

4. Clone git repository
	
	```
	mkdir ref_lineages
	cd ref_lineages
	git clone https://github.com/cov-lineages/pango-designation.git
	```

5. Extract sequence names only

	```
	awk -F"," `{print $1}`	pango-designation/lineages.csv | tail -n +2 > ref_lineages_${today}.txt
	```

6. Retrieve sequences from most up-to-date gisiad alignment

	```
	seq_cleaner.py -P ref_lineages_${today}.txt <path/to/gisaid/msa> > ref_lineages_${today}.fa
	```
	
7. Gap strip for alignment

	```
	seq_cleaner.py -g ref_lineages_${today}.fa > ref_lineages_${today}_gapstripped.fa
	```
	
8. Align using viralmsa

	```
	file=$(ls *gapstripped.fa)
	ml viralmsa
	ViralMSA.py -s $file -t 4 -e brittany.rife@ufl.edu -o ${file%_gapstripped.fa} -r MN908947 
	
	```
	
9. Download most recent masked sites vcf:

	```
	wget - O ../problematic_sites_sarsCov2.vcf https://raw.githubusercontent.com/W-L/ProblematicSites_SARS-CoV2/master/problematic_sites_sarsCov2.vcf
	```

10. Mask uncertain sites

	```
	ml python
	python ../mask_aln_using_vcf.py -i ref_lineages_${today}/*.aln -o ref_lineages_${today}_masked.aln -v ../problematic_sites_sarsCov2.vcf
	```

11. Create R list of lineage-specific alignments for spike

	```
	sbatch makeRefSpikeList.sh -r ref_lineages_${today}_masked.aln	
	```
	 	
		
**Note:** If this already exists, just need to get sequences not already included in alignment above.


# ALIGNMENT OF INITAL DATA ##############################################################

1. In the meantime, can align sequences using viralmsa:

	```
	cd initial_data
	seq_cleaner.py -g ${today}.fa > ref_lineages_${today}_gapstripped.fa
	ml viralmsa  
	ViralMSA.py -s <initial>.fasta -t 4 -e 	brittany.rife@ufl.edu -o <initial> -r MN908947
	```
	 
2. Check for extensive gaps in sequences	(may need to reduce sed command above)
	
	```
	grep -o "-" <initial>/<initial>.aln | wc -l | awk '$1>300{c++} END{print c+0}'
	```

3. Check for gaps in reference sequence (shouldn't be there). If found, need to either use mafft or report in metadata file:
	
	```
	head -n 2 <initial>/<initial>.aln | tail -n 1 | grep -o "-" | wc -l
	```
	
	(If mafft necessary):
	
	```
	ml mafft
	mafft --thread -1 <initial>.fasta > <initial>.aln
	mafft --retree 3 --maxiterate 10 --thread -1 --nomemsave --op 10 seqsCausingInsertionsInRef.fasta > seqsCausingInsertionsInRef_aligned.fasta
	mafft --thread 1 --quiet --keeplength --add sequencesNotCausingInsertionsInRef.fa seqsCausingInsertionsInRef_aligned.fasta > <Run#>_florida_gisaid.aln
	```

4. Mask uncertain sites

	```
	ml python
	python ../mask_aln_using_vcf.py -i ref_lineages_${today}/*.aln -o ref_lineages_${today}_masked.aln -v ../problematic_sites_sarsCov2.vcf
	```
	
5. Change first sequence header to just MN908947.3 (spaces in original name wreak havoc downstream):
	
	```
	var=">MN908947.3"
	sed -i "1s/.*/$var/" <initial>/<initial>.aln 
	```
	
		
# INITIAL TREE RECONSTRUCTION AND DYNAMITE CLUSTER IDENTIFICATION ####################################

1.  Create ML tree using IQ-TREE job script (which will automatically run on a fasta file):
	
	```
	cd <initial>
	file=$(ls *masked.aln)
	ml iq-tree
	iqtree2 -s $file -bb 1000 -nt AUTO -m GTR+I+G
	```
2. Create metadata.txt file containing minimal information used for DYNAMITE:
	
	```
	ml R
	Rscript metadata.R
	```
3. run DYNAMITE to identify clusters using tree and metadata (will output individual trees and fasta files for clusters and background):
	
	```
	sbatch dynamite.sh
	```

# SUBTREE/ALN PROCESSING USING USHER #######################################################
	

7. Create folder for reference sequence
	
	```
	cd ../
	mkdir cov_reference
	head -n 2 *.aln > cov_reference.fasta 
	```

8. Add reference sequence again to each of the newly generated fasta files:
	
	```
	for i in ./*.fasta; do cat ../cov_reference/cov_reference.fasta >> ${i}; done
	```
	
1. Copy FaToVcf to working directory (required for UShER):
		
	```
	rsync -aP rsync://hgdownload.soe.ucsc.edu/genome/admin/exe/linux.x86_64/faToVcf .
	chmod +x faToVcf
	```
				
2. Transform fasta sub-alignments to vcf
	
	```
	for i in ./*.fasta; do ./faToVcf -ref=MN908947.3 ${i} ${i%.fasta}.vcf; done
	```
	
3. Perform subtree pre-processing (need to modify to only select cluster and background trees and to exclude original, full tree) 
	
	```
	ml usher
	for i in ./*${today}.tree; do usher -v ${i%.tree}.vcf -t ${i} -T 4 -o ${i%.tree}.pb; done &
	```


# FLACO-BLAST ON NEW SEQUENCE DATA  ##########################################################################

### The following are performed by Alberto Riva, so need script information: ##############
1. Metadata information is provided in the form of an Excel sheet and is uploaded into an informal database structure that can be queried.

2. Upload sequences to GISAID

3. Load basement module:
    
    ```
    ml basemount
    ```

4. Create folder to copy files and basemount (must be done in home directory and not in blue):
	
	```
	mkdir COVID_FL_${today};
	basemount COVID_FL_${today};
	cd COVID_FL_${today};
	cd Projects/<BaseSpace project name>/AppResults
	```

5. Copy consensus fasta files from Basespace:
	
	```
	cp ./*/Files/consensus/*.fasta <hipergator dir>
	```
	
6. Identify Pangolin lineage for each sequence (if able) and add to separate database with sequence run information.

7. Filter sequences based on 1) high N content (currently allowing 30%), 2) full date information (YYMMDD), and 3) coverage (>50x across all sites)
	
8. Rename sequences so that format similar to GISAID 

9. Append fastas to single fasta file for all sequences thus far 

### End Alberto Riva ######################################################################

10. Update (assuming existing) dates.txt file with new date and assign current ("today") and previous ("previous") date variables:
	
	```
	echo $(date +"%Y-%m-%d") >> dates.txt
	today=$(tail -n 1 dates.txt)
	previous=$(sed 'x;$!d' <dates.txt)
	```

11. Make new directory for new round of samples:
	
	```
	mkdir ./<Run#>_${today}
	cd ./<Run#>_${today}
	```


12. Querying sequences according to run can be performed using the flacoblastdb query structure:
	 
	 ```
	 flacodb.py query -r <Run#> -x Haiti,Haiti2,Haiti3,Haiti4,Haiti5,Haiti6,Haiti7,MD +PangoLineage > <Run#>.txt;
	 awk ‘{print $1}’ <Run#>.txt | tail -n +2 > <Run#>_names.txt
	 seq_cleaner.py -P <Run#>_names.txt /blue/salemi/share/COVIDseq/all-groups/ALL-consensus.good.dates.fa > <Run#>.fasta &   
	 ```
	   
13. Meanwhile, extract Floridian sequences from database that correspond to the relevant time frame:
	
	```
	gisaidfilt.py ../../gisaid_msa/msa_<date>/msa_<date>.fasta -n USA/FL -a <date>_15 -b <date>+15  -o florida_${today}.fasta & 
	```

14. Meanwhile, remove Floridian sequences from GISAID database:
	
	```
	cd ../flaco-blast;
	flaco_blast.sh makedb ../gisaid_msa/msa_<date>/msa_<date>.fasta USA/FL &
	```
	
15. Combine in-house sequences with GISAID Floridian sequences:
	
	```
	cat <Run#>.fasta florida_${today}.fasta > <Run#>_florida_${today}.fasta
	```

16. Remove gaps for BLAST (works better)
	
	```
	seq_cleaner.py -g <Run#>_florida_${today}.fasta > <Run#>_florida_${today}_gapstripped.fa &
	```


17. BLAST combined sequences against new GISAID global database:
	
	```
	cd ../flaco_blast
	flaco_blast.sh run <Run#>_florida_${today}_gapstripped.fa ../flaco_blast/msa_<date>.clean.fa
	``` 
	**Note**: that always needs to be run in the same folder as the folder containing the dbs from the previous step

18. Combine target sequences with original query sequences
	
	```
	cd ../<Run#>_${today}
	cat <Run#>_florida_${today}_gapstripped.fa ../flaco-blast/msa_<date>.clean.out.fa > <Run#>_florida_gisaid_${today}_gapstripped.fa
	```
19. Run pangolin on new sequences (updated daily)
	
	```
	ml pangolin 	
	pangolin <Run#>_florida_gisaid_${today}_gapstripped.fa --outfile <Run#>_florida_gisaid_${today}_lineages.csv
	```	
# ALIGNMENT OF NEW SEQUENCE DATA ########################################################

1. Align sequences using viralmsa:
	
	```
	sbatch ../viralmsa.sh
	```

If mafft necessary, see above.

2. Mask uncertain sites

	```
	ml python
	python ../mask_aln_using_vcf.py -i <Run#>_florida_gisaid_${today}/*.aln -o <Run#>_florida_gisaid_${today}_masked.aln -v ../problematic_sites_sarsCov2.vcf
	```
	
3. Change first sequence header to just MN908947.3 (spaces in original name wreak havoc downstream):
	
	```
	var=">MN908947.3"
	sed -i "1s/.*/$var/" *.aln 
	```
	 
		
# SUBTREE/ALN PROCESSING USING USHER #######################################################
		
	
1. Transform fasta to vcf
	
	```
	../faToVcf -ref=MN908947.3 ./<Run#>_florida_gisaid_${today}_masked.aln <Run#>_florida_gisaid_${today}_masked.vcf	&
	```		

2. Officially add samples to all previous trees using usher (creating new pb files for today):
	
	```
	cd ../
	ml usher
	for i in $(ls | grep "${previous}.pb"); do usher -i ${i} -v <Run#>_florida_gisaid_${today}/<Run#>_florida_gisaid_${today}_masked.vcf -o -p ${i%_${previous}.pb}_${today}.pb; done	
	```

3. Compute parsimony score for new samples (vcf) assigned to each annotated tree (.pb files):
	
	```
	for i in $(ls | grep "${previous}.pb"); do 
	mkdir ${i%_${previous}.pb}_${today};
	usher -i ${i} -v <Run#>_florida_gisaid_${today}/<Run#>_florida_gisaid_${today}_masked.vcf -p -d ${i%_${previous}.pb}_${today};
	done
	```
	
4. Previous step will generate parsimony scores in tsv files, which we need to rename and copy to single folder for R analysis:
	
	```
	mkdir BPS_${today}
	for i in ./*_${today}; do cp ${i}/parsimony-scores.tsv BPS_${today}/${i}_parsimony-scores.tsv; done
	```
	
5. Use R script to evaluate parsimony scores from tsv files and output new fasta files for sequences needing to be placed on trees:
	
	```
	cd BPS_${today}
	ml R
	Rscript ../branch_support_eval.R
	```
6. ## Need to add lines to evaluate if same sequence found in multiple subtrees and choose using usher rules.
	


8. Add reference sequence again to each of the newly generated fasta files:
	
	```
	for i in ./*.fasta; do cat ../cov_reference/cov_reference.fasta >> ${i}; done
	```

9. Place updated fasta in new folders:
	
	```
	for i in ./*.fasta; do mkdir .${i%.fasta}; cp ${i} .${i%.fasta}; done;
	cd ../
	``
	
10. Replace old folders in source.txt file with updated folders:
	
	```
	find . -type d -name "*${today}_updated" > source.txt
	```

	
11. Convert to vcf as above (this needs some work to make sure naming is correct)
	
	 ```
	for i in $(cat source.txt); do faTovcf -ref=MN908947.3 ${i}/*.fasta ${i}/*.vcf; done	
	```

12. Officially add samples to all old trees using usher (creating new pb files for today):
	
	```
	ml usher
	for i in $(cat source.txt); do usher -i ${i%_${today}*}_${previous}.pb -v ${i}/*.vcf -u -d -o ${i%_updated}.pb; done
	```	
	
13. Not all trees (.pb) are going to be updated with sequences, so need to find all trees without today's date and make a copy of previous pb with today's date:
	
	```
	for i in ./*${previous}.pb; do mv -vn ${i} ${i%_${previous}*}_${today}.pb; done
	```

14. Move new unannotated tree (.nh) files generated from the previous step into one folder (as well as original combined sample fasta) for characterization:
	
	```
	mkdir "trees_${today}";
	for i in $(cat source.txt); do cp ${i}/*final-tree.nh trees_${today}/${i}.tree; done;
	cp ?????? trees_${today}
	```
	
15. Run modified DYNAMITE to search for (and prune) clusters within background tree:
	
	```
	cd ./background_${today}
	sbatch ../dynamite.sh
	```

*Note* Need to determine if cluster name already given in main folder. If so, rename (new number doesn't matter). 
*Note* Need better way to update previously considered background nodes in metadata file.
	

16. Run R script to characterize added sequences (add new folder to save discard results):	# This needs to be modified so that when discard tree exists, it will create just a fasta that will need to be processed
	
	```
	mkdir "./discard_${today}"
	cd trees_${today}
	ml R
	Rscript ../fitness_calc.R
	```
	
17. If discard tree not present, create one. If so, process fasta from previous step and add to existing annotated tree, and also update source.txt file
	
	```
	if [ ! -e "discard_${previous}.vcf" ]; then
       cd 	../discard_${today}; sbatch ../iqtree.sh;
      else
    	cat ./cov_reference/cov_reference.fasta >> ./discard_${today}_updated/*.fasta
		ml python
		python Fasta2UShER.py -inpath discard_${today}_updated -output discard_${today}.vcf -reference ./cov_reference/cov_reference.fasta	
		ml usher
		usher -v discard_${today}.vcf -t ./discard_${today}_updated.tree -T 4 -c -u -d ./discard_${today}_updated -o discard_${today}.pb
		echo "discard_${today}_updated" >> source.txt
	fi 
	```

18. If first condition met above, then need to pre-process tree and fasta (but somehow need to make sure iqtree is finished:
	    
	
	```
	cat ./cov_reference/cov_reference.fasta >> ./discard_${today}_updated/*.fasta
	ml python
	python Fasta2UShER.py -inpath discard_${today}_updated -output discard_${today}.vcf -reference ./cov_reference/cov_reference.fasta	
	ml usher
	usher -v discard_${today}.vcf -i ./discard_${previous}.pb -T 4 -c -u -d ./discard_${today}_updated -o discard_${today}.pb
	```

*Note* Is this step still necessary??
21. While discard tree being reconstructed, place pruned background sequences onto previous annotated tree:
	
	```
	cd ../background_${today}_updated
	rm -v !(*.fasta)
	cd ../
	ml python
	python Fasta2UShER.py -inpath ./background_${today}_updated -output ./background_${today}.vcf -reference ./cov_reference/cov_reference.fasta
	ml usher
	usher -v background_${today}.vcf -i ./background_${previous}.pb -T 4 -c -u -d ./background_${today}_updated -o background_${today}.pb
	```
	

22. Concatenate previous fasta with updated fasta for tree optimization :
	
	```
	for i in $(cat source.txt); do 
*.fasta >> ${i}/full.fasta
	done
	```
23. Remove duplicated sequences:
	
	```
	for i in $(cat source.txt); cat ${i}/full.fasta | seqkit rmdup -n -o ${i}/clean.aln
	```

23. Force bifurcating tree in R:
	
	```
	ml R
	for i in $(cat source.txt); do Rscript ./bifurcate.R -t ${i}/uncondensed-final-tree.nh; done
	```

 
24. Re-optimize trees (only those that were updated) using FastTree: 
	
	```
	module load fasttree/2.1.7
	for i in $(cat source.txt); do FastTreeMP -nt -gamma -nni 0 -spr 2 -sprlength 1000 -boot 100 -log fasttree.log -intree ${i}/*.nh.tree ${i}/clean.aln > ${i}/reop.tree; done
	```
	

25. Re-annotate updated fasttree trees (replace old ones):
	
	```
	ml usher
	for i in ${cat source.txt); do
  	usher -v ${i%_updated}.vcf -t ${i}/reop.tree -T 4 -c -u -d ${i} -o ${i%_updated}.pb;
  	done
  	```

26. Test background and discard trees for clusters (separate R script using actual branchwise algorithm). R script will write new results for discard pile to a new folder, but will write results from background to same folder.
	IF no clusters are found, results are not written, but downstream steps require that the new discard folder exist, so need to copy over.

	```
	mkdir ./discard_${today}_updated
	cd ./discard_${today}
	Rscript ../branchwise.R -t *.treefile -p ../discard_${today}_updated
	
	cd ../background_${today}_updated
	Rscript ../branchwise.R -t reop.tree -p ./
	cd ../
	```
		
27. Process fasta and tree for background (will require determining if new files added as a result of R script in step above)

	```
	cat ./cov_reference/cov_reference.fasta >> ./background_${today}_updated/*.fasta
	find . -type f -name "background_${today}_updated.tree" -empty -exec cp ./background_${today}/reop.tree ./background_${today}_updated.tree
	ml python
	python Fasta2UShER.py -inpath background_${today}_updated -output background_${today}.vcf -reference ./cov_reference/cov_reference.fasta	
	ml usher
	usher -v background_${today}.vcf -t ./background_${today}_updated.tree -T 4 -c -u -d ./background_${today}_updated -o background_${today}.pb
	```
	
28. Add reference sequence to new cluster fastas and place in new directories:
	
	```
	for i in ./*.fasta; do  cat ./cov_reference/cov_reference.fasta >> ${i}; cp ${i} .${i%.fasta}; done
	```
	
29. Update source.txt file with new clusters and process fastas:
	
	```
	for i in ./*.fasta; do mkdir .${i%.fasta}; echo ${i%.fasta} > source.txt; done	
	ml python
	for i in $(cat source.txt); do python Fasta2UShER.py -inpath ${i} -output ${i}.vcf -reference ./cov_reference/cov_reference.fasta; done
	```
	
30. Perform subtree pre-processing for new clusters:
	
	```
	ml usher/0.1.4	
	for i in $(cat source.txt); do usher -v ${i}.vcf -t ${i}.tree -T 4 -c -u -d ./${i} -o ${i}.pb; done
	```




	
### Obtaining additional metadata
	
	8. Pull metadata from FLACODB:
	
	```
	cd ../
	sqlite3 -header -separator $'\t' $FLACO_DB \
"select SampleName, SamplingDate, DaysAfterBaseline, Location, City, County, ZipCode, Sex, Age, DiseaseOutcome,CoMorbidityFactors, ViralLoad \
  from Samples \
  where SampleName like 'Shands-%' or \
	SampleName like 'Path-%' or \
	SampleName like 'STP-%'or \
	SampleName like 'BayCare-%'or \
	SampleName like 'Miami-%' or \
	SampleName like 'FDOH-%' or \
	SampleName like 'FtMyers-%';" > metadata_${today}.txt &
	flacodb.py query -x Haiti,Haiti2,Haiti3,MD +PangoLineage +HiDepthFrac50 > lineages_${today}.txt
	```
**Note** groups are constantly changing, so need to make sure only grabbing Florida sequences.


			
			







