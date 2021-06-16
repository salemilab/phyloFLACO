# ALIGNMENT OF INITAL DATA ##############################################################

1. Align sequences using viralmsa:
	```
	ml viralmsa;
	ViralMSA.py -s <initial>.fasta -t 4 -e brittany.rife@ufl.edu -o <output_dir> --outfile <initial>.aln -r MN908947
	```
	
2. Check for extensive gaps in sequences	(may need to reduce sed command above)
	```
	grep -o "-" <initial>.aln | wc -l | awk '$1>300{c++} END{print c+0}'
	```

3. Check for gaps in reference sequence (shouldn't be there). If found, need to either use mafft or report in metadata file:
	```
	head -n 2 <initial>.aln | tail -n 1 | grep -o "-" | wc -l
	```
	
	(If mafft necessary):
	```
	ml mafft;
	mafft --thread -1 <Run#>_florida_gisaid.fasta > <Run#>_florida_gisaid2.fasta;
	mafft --retree 3 --maxiterate 10 --thread -1 --nomemsave --op 10 seqsCausingInsertionsInRef.fasta > seqsCausingInsertionsInRef_aligned.fasta;
	mafft --thread 1 --quiet --keeplength --add sequencesNotCausingInsertionsInRef.fa seqsCausingInsertionsInRef_aligned.fasta > <Run#>_florida_gisaid.aln
	```

4. Change first sequence header to just MN908947.3:
	```
	var=">MN908947.3";
	sed -i "1s/.*/$var/" <initial>_masked.aln 
	```
	
5. Create cov_reference folder and copy cov_reference.fasta
	```
	mkdir cov_reference;
	head -n 2 <initial>.aln > ./cov_reference/cov_reference.fasta
	```

6. Mask known sequence error sites (with 'N') using mask_aln_using_vcf.py
	```
	ml python;
	python mask_aln_using_vcf.py -i <initial>.aln -o <initial>_masked.aln -v problematic_sites_sarsCov2.vcf
	```

		
# INITIAL TREE RECONSTRUCTION AND DYNAMITE CLUSTER IDENTIFICATION ####################################

7.  Create ML tree using IQ-TREE job script (which will automatically run on a fasta file):
	```
	sbatch iqtree.sh
	```

### While IQ-TREE running...
8. Create metadata.csv BEFORE next step (pulls information from sequence headers):
	```
	ml R;
	Rscript <path/to>metadata.R
	```

9. Create a file with dates for later referencing and a variable ("today") from the dates file to help with naming:
	```
	echo $(date +"%Y-%m-%d") > dates.txt;
	today=$( tail -n 1 dates.txt )
	```

			
7. When tree finished, run DYNAMITE to identify clusters using tree and metadata:
	```
	sbatch dynamite.sh
	```

# SUBTREE/ALN PROCESSING USING USHER #######################################################
	
30. Copy FaToVcf to working directory (required for UShER):
		```
		rsync -aP rsync://hgdownload.soe.ucsc.edu/genome/admin/exe/linux.x86_64/faToVcf .;
		chmod +x faToVcf
		```
		
33. Perform subtree pre-processing 
	```
	ml usher;	
	for i in ./*.tree; do usher -v ${i%.tree}.vcf -t ${i} -T 4 -c -u -d ./${i%.tree} -o ${i%.tree}.pb; done &
	```
		
32. Transform fasta sub-alignments to vcf
	```
	for i in ./*.fasta ./faToVcf -ref=MN908947.3 ${i} ${i%.fasta}.vcf	&
	```

31. Move initial alignment and tree data to new folder:
	```
	mkdir initial_tree_data;
	mv <initial aln> initial_tree_data;
	mv dynamite_*.tree initial_tree_data;
	mv initial_${today}* initial_tree_data
	```

# FLACO-BLAST ON NEW SEQUENCE DATA  ##########################################################################
8. Load FLACO environment
	```
	source /blue/salemi/share/COVIDseq/bin/FLACO #Note: made alias "flaco"
	```

### The following are performed by Alberto Riva, so need script information: ##############
9. Metadata information is provided in the form of an Excel sheet and is uploaded into an informal database structure that can be queried.

10. Upload sequences to GISAID

11. Load basement module:
    ```
    ml basemount
    ```

12. Create folder to copy files and basemount (must be done in home directory and not in blue):
	```
	mkdir COVID_FL_${today};
	basemount COVID_FL_${today};
	cd COVID_FL_${today};
	cd Projects/<BaseSpace project name>/AppResults
	```

13. Copy consensus fasta files from Basespace:
	```
	cp ./*/Files/consensus/*.fasta <hipergator dir>
	```
	
14. Identify Pangolin lineage for each sequence (if able) and add to separate database with sequence run information.

15. Filter sequences based on 1) high N content (currently allowing 30%), 2) full date information (YYMMDD), and 3) coverage (>50x across all sites)
	
16. Rename sequences so that format similar to GISAID 

17. Append fastas to single fasta file for all sequences thus far 

### End Alberto Riva ######################################################################

18. Update (assuming existing) dates.txt file with new date and assign current ("today") and previous ("previous") date variables:
	```
	echo $(date +"%Y-%m-%d") >> dates.txt;
	today=$(tail -n 1 dates.txt);
	previous=$(sed 'x;$!d' <dates.txt)
	```

2. Make new directory for new round of samples:
	```
	mkdir ./<Run#>_${today};
	cd ./<Run#>_${today}
	```


19. Querying sequences according to run can be performed using the flacoblastdb query structure:
	 ```
	 flacodb.py query -r <Run#> -x Haiti,Haiti2,MD +PangoLineage > <Run#>.txt;
	 awk ‘{print $1}’ <Run#>.txt | tail -n +2 > <Run#>_names.txt;
	 seq_cleaner.py -P <Run#>_names.txt /blue/salemi/share/COVIDseq/all-groups/ALL-consensus.good.dates.fa > <Run#>.fasta &   
	 ```
	   
20. Meanwhile, extract Floridian sequences from database that correspond to the relevant time frame:
	```
	gisaidfilt.py msa_<date>.fasta -n USA/FL -a 2020-11-25_15 -b 2021-01-31+15  -o florida_${today}.fasta 
	```

21. Combine in-house sequences with GISAID Floridian sequences:
	```
	cat <Run#>.fasta florida_${today}.fasta > <Run#>_florida_${today}.fasta
	```

22. Remove gaps for BLAST (works better)
	```
	seq_cleaner.py -g <Run#>_florida_${today}.fasta > <Run#>_florida_${today}_nogaps.fasta &
	```

23. Meanwhile, remove Floridian sequences from GISAID database:
	```
	cd ../flaco-blast;
	flaco_blast.sh makedb ../gisaid_msa/msa_<date>.fasta USA/FL
	```

24. BLAST combined sequences against new GISAID global database (now won't retrieve identical sequences):
	```
	flaco_blast.sh run ../<Run#>_${today}/<Run#>_florida_${today}_nogaps.fasta msa_<date>.clean.fa
	``` 
	**Note**: that always needs to be run in the same folder as the folder containing the dbs from the previous step

25. Combine target sequences with original query sequences
	```
	cd <Run#>_${today};
	cat <Run#>_florida_${today}_nogaps.fasta ../flaco-blast/msa_<date>.cleand.out.fa > <Run#>_florida_gisaid_${today}.fasta
	```
25. Re-run pangolin
	```
	ml pangolin 	
	pangolin <Run#>_florida_gisaid_${today}.fasta --outfile <Run#>_florida_gisaid_${today}_lineages.csv
	```	
# ALIGNMENT OF NEW SEQUENCE DATA ########################################################

25. Align sequences using viralmsa:
	```
	ml viralmsa;
	ViralMSA.py -s <Run#>_florida_gisaid_${today}.fasta -t 4 -e brittany.rife@ufl.edu -o <Run#>_florida_gisaid_${today} --outfile <Run#>_florida_gisaid_${today}.aln -r MN908947
	```
	
26. Check for extensive gaps in sequences	(may need to reduce sed command above)
	```
	grep -o "-" <Run#>_florida_gisaid_${today}/<Run#>_florida_gisaid_${today}.aln | wc -l | awk '$1>300{c++} END{print c+0}'
	```

27. Check for gaps in reference sequence (shouldn't be there). If found, need to either use mafft or report in metadata file:
	```
	head -n 2 <Run#>_florida_gisaid_${today}/<Run#>_florida_gisaid_${today}.aln | tail -n 1 | grep -o "-" | wc -l
	```

If mafft necessary, see above.

28. Mask known sequence error sites (with 'N') using mask_aln_using_vcf.py
	```
	python mask_aln_using_vcf.py -i <Run#>_florida_gisaid_${today}/<Run#>_florida_gisaid_${today}.aln -o <Run#>_florida_gisaid_${today}_masked.aln -v problematic_sites_sarsCov2.vcf
	```

29. Change first sequence header to just MN908947.3:
	```
	var=">MN908947.3";
	sed -i "1s/.*/$var/" <Run#>_florida_gisaid_${today}_masked.aln
	``` 
		
# TREE/ALN PROCESSING USING USHER #######################################################
		
	
32. Transform fasta sub-alignments to vcf
	```
	../faToVcf -ref=MN908947.3 <Run#>_florida_gisaid_${today}_masked.aln $<Run#>_florida_gisaid_${today}_masked.vcf	&
	```		
16. Officially add samples to all previous trees using usher (creating new pb files for today):
	```
	ml usher;
	for i in */.tree; do usher -i ${i%_${today}*}_${previous}.pb -v $$<Run#>_florida_gisaid_${today}/$<Run#>_florida_gisaid_${today}_masked.vcf -u -o ${i%_updated}.pb; done	
	```

9. Compute parsimony score for new samples (vcf) assigned to each annotated tree (.pb files):
	```
	for i in ./*_${previous}.pb; do 
	mkdir ${i%_${previous}*}_${today};
	usher -i ${i} -v samples_${today}.vcf -p -d ${i%_${previous}*}_${today};
	done
	```
	
10. Previous step will generate parsimony scores in tsv files, which we need to rename and copy to single folder for R analysis:
	```
	mkdir "BPS_${today}";
	for i in ./*_${today}; do cp ${i}/parsimony-scores.tsv BPS_${today}/${i}_parsimony-scores.tsv; done
	```
	
11. Use R script to evaluate parsimony scores from tsv files and output new fasta files for sequences needing to be placed on trees:
	```
	cd BPS_${today};
	ml R;
	Rscript ../branch_support_eval.R
	```
	
12. Add reference sequence again to each of the newly generated fasta files:
	```
	for i in ./*.fasta; do cat ../cov_reference/cov_reference.fasta >> ${i}; done
	```

13. Place updated fasta in new folders:
	```
	for i in ./*.fasta; do mkdir .${i%.fasta}; cp ${i} .${i%.fasta}; done;
	cd ../
	``
	
14. Replace old folders in source.txt file with updated folders:
	```
	find . -type d -name "*${today}_updated" > source.txt
	```

	
16. Officially add samples to all old trees using usher (creating new pb files for today):
	```
	ml usher;
	for i in $(cat source.txt); do usher -i ${i%_${today}*}_${previous}.pb -v ${i%_updated}.vcf -u -d ${i} -o ${i%_updated}.pb; done
	```	
	
17. Not all trees (.pb) are going to be updated with sequences, so need to find all trees without today's date and make a copy of previous pb with today's date:
	```
	for i in ./*${previous}.pb; do mv -vn ${i} ${i%_${previous}*}_${today}.pb; done
	```

18. Move new unannotated tree (.nh) files generated from the previous step into one folder (as well as original combined sample fasta) for characterization:
	```
	mkdir "trees_${today}";
	for i in $(cat source.txt); do cp ${i}/*final-tree.nh trees_${today}/${i}.tree; done;
	cp ./samples_${today}/*.fasta trees_${today}
	```
	
19. Run modified DYNAMITE to search for (and prune) clusters within background tree:
	```
	cd ./background_${today}
	```
	# Need to run on trees_${today}/background.tree though!
	
	# Identify clusters that contain new sequence data
	# Move clusters to main folder (need to come up with new cluster name so not copied, like "_bg")
	# Prune clusters from tree
	# Save tree in original folder (and update metadata file?) for next step.
	

20. Run R script to characterize added sequences (add new folder to save discard results):	# This needs to be modified so that when discard tree exists, it will create just a fasta that will need to be processed
	mkdir "./discard_${today}"
	cd trees_${today}
	ml R
	Rscript ../fitness_calc.R
	
19. If discard tree not present, create one. If so, process fasta from previous step and add to existing annotated tree, and also update source.txt file
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

20. If first condition met above, then need to pre-process tree and fasta (but somehow need to make sure iqtree is finished:
	
	#Find a way to determine if iqtree job was submitted and if job is finished. Once job is finished (if needed),
	    
	cat ./cov_reference/cov_reference.fasta >> ./discard_${today}_updated/*.fasta
	ml python
	python Fasta2UShER.py -inpath discard_${today}_updated -output discard_${today}.vcf -reference ./cov_reference/cov_reference.fasta	
	ml usher
	usher -v discard_${today}.vcf -i ./discard_${previous}.pb -T 4 -c -u -d ./discard_${today}_updated -o discard_${today}.pb


21. While discard tree being reconstructed, place pruned background sequences onto previous annotated tree:
	cd ../background_${today}_updated
	rm -v !(*.fasta)
	cd ../
	ml python
	python Fasta2UShER.py -inpath ./background_${today}_updated -output ./background_${today}.vcf -reference ./cov_reference/cov_reference.fasta
	ml usher
	usher -v background_${today}.vcf -i ./background_${previous}.pb -T 4 -c -u -d ./background_${today}_updated -o background_${today}.pb
	

22. Concatenate previous fasta with updated fasta for tree optimization (while removing all instances of refseq):
	for i in $(cat source.txt); do 
	head -n -2 ${i}/*.fasta > ${i}/full.fasta;
	head -n -2 ${i%_${today}*}_${previous}/*.fasta >> ${i}/full.fasta
	done

23. Force bifurcating tree in R:
	ml R
	for i in $(cat source.txt); do Rscript ./bifurcate.R -t ${i}/uncondensed-final-tree.nh; done

 
24. Re-optimize trees (only those that were updated) using FastTree: 
	cat ./cov_reference/cov_reference.fasta >> ./background_${today}/full.fasta # This is because refseq should actually be included in the background
	module load fasttree/2.1.7
	for i in $(cat source.txt); do FastTreeMP -nt -gamma -nni 0 -spr 2 -sprlength 1000 -boot 100 -log fasttree.log -intree ${i}/*.nh.tree ${i}/full.fasta > ${i}/reop.tree; done
	

25. Re-annotate updated fasttree trees (replace old ones):
	ml usher
	for i in ${cat source.txt); do
  	usher -v ${i%_updated}.vcf -t ${i}/reop.tree -T 4 -c -u -d ${i} -o ${i%_updated}.pb;
  	done

26. Test background and discard trees for clusters (separate R script using actual branchwise algorithm). R script will write new results for discard pile to a new folder, but will write results from background to same folder.
	IF no clusters are found, results are not written, but downstream steps require that the new discard folder exist, so need to copy over.

	mkdir ./discard_${today}_updated
	cd ./discard_${today}
	Rscript ../branchwise.R -t *.treefile -p ../discard_${today}_updated
	
	cd ../background_${today}_updated
	Rscript ../branchwise.R -t reop.tree -p ./
	cd ../
		
27. Process fasta and tree for background (will require determining if new files added as a result of R script in step above)

	cat ./cov_reference/cov_reference.fasta >> ./background_${today}_updated/*.fasta
	find . -type f -name "background_${today}_updated.tree" -empty -exec cp ./background_${today}/reop.tree ./background_${today}_updated.tree
	ml python
	python Fasta2UShER.py -inpath background_${today}_updated -output background_${today}.vcf -reference ./cov_reference/cov_reference.fasta	
	ml usher
	usher -v background_${today}.vcf -t ./background_${today}_updated.tree -T 4 -c -u -d ./background_${today}_updated -o background_${today}.pb
	
28. Add reference sequence to new cluster fastas and place in new directories:
	for i in ./*.fasta; do  cat ./cov_reference/cov_reference.fasta >> ${i}; cp ${i} .${i%.fasta}; done
	
29. Update source.txt file with new clusters and process fastas:
	
	for i in ./*.fasta; do mkdir .${i%.fasta}; echo ${i%.fasta} > source.txt; done	
	ml python
	for i in $(cat source.txt); do python Fasta2UShER.py -inpath ${i} -output ${i}.vcf -reference ./cov_reference/cov_reference.fasta; done
	
30. Perform subtree pre-processing for new clusters:
	ml usher/0.1.4	
	for i in $(cat source.txt); do usher -v ${i}.vcf -t ${i}.tree -T 4 -c -u -d ./${i} -o ${i}.pb; done




	
	
	
	

			
			







