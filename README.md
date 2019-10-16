`Locator` is a supervised machine learning method for predicting geographic location from
genotype or sequencing data. 

# Installation 

requires python3, gnuplot, and the following packages:
```
allel, re, os, keras, matplotlib, sys, zarr, time, subprocess, copy
numpy, pandas, tensorflow, scipy, tqdm, argparse, gnuplotlib
```

Gnuplot (http://www.gnuplot.info/) can be installed with your favorite package manager, e.g. 
```
conda install -c bioconda gnuplot #for conda users
brew install gnuplot #mac 
sudo apt-get install gnuplot #linux
```

To install the python dependencies, download the repository and run the setup script: 
```
git clone https://github.com/cjbattey/locator.git
cd locator
python setup.py install
```
 
For large datasets or bootstrap uncertainty estimation we recommend 
running on a CUDA-enabled GPU (https://www.tensorflow.org/install/gpu).

# Examples

This command should fit a model to a simulated test dataset of 
~10,000 SNPs and 450 individuals and predict the locations of 50 validation samples. 

```
locator.py --vcf data/test_genotypes.vcf.gz --sample_data data/test_sample_data.txt --out out/test
```

It will produce 4 files: 

test_predlocs.txt -- predicted locations  
test_history.txt -- training history  
test_weights.hdf5 -- model weights   
test_fitplot.pdf -- plot of training history   

See all parameters with `python scripts/locator_dev.py --h`

# Uncertainty
For whole genome or dense SNP data, we recommend running locator in windows across the genome. 
We did this by subsetting VCFs with Tabix:

```
step=2000000
for chr in {2L,2R,3L,3R,X}
do
	echo "starting chromosome $chr"
	#get chromosome length
	header=`tabix -H /home/data_share/ag1000/phase1/ag1000g.phase1.ar3.pass.biallelic.$chr\.vcf.gz | grep "##contig=<ID=$chr,length="`
	length=`echo $header | awk '{sub(/.*=/,"");sub(/>/,"");print}'` 
	
	#subset vcf by region and run locator
	endwindow=$step
	for startwindow in `seq 1 $step $length`
	do 
		echo "processing $startwindow to $endwindow"
		tabix -h /home/data_share/ag1000/phase1/ag1000g.phase1.ar3.pass.biallelic.$chr\.vcf.gz \
		$chr\:$startwindow\-$endwindow > data/ag1000g/tmp.vcf
		
		python scripts/locator.py \
		--vcf data/ag1000g/tmp.vcf \
		--sample_data data/ag1000g/ag1000g.phase1.samples.locsplit.txt \
		--out out/ag1000g/$chr\_$startwindow\_$endwindow
		
		endwindow=$((endwindow+step))
		rm data/ag1000g/tmp.vcf
	done
done
```

You can also train replicate models on bootstrap samples of the full VCF (sampling SNPs with replacement) with the 
`--bootstrap` argument. To fit 5 bootstrap replicates, run:
```
mkdir out/bootstrap
locator.py --vcf data/test_genotypes.vcf.gz --sample_data data/test_sample_data.txt --out out/bootstrap/test --bootstrap True --nboots 5 --keep_weights False
```
This is slow (you're fitting new models to each replicate), but should give a good idea of uncertainty in predicted locations. A quicker and probably worse estimate can also be generated by the `--jacknife` option. This uses a single trained model and generates predictions while treating 5% of sites as missing data. You can run that with:

```
mkdir out/jacknife
locator.py --vcf data/test_genotypes.vcf.gz --sample_data data/test_sample_data.txt --out out/jacknife/test --jacknife True --nboots 20 --keep_weights False
```

# Plotting
plot_locator.R is a command line script that plots maps of locator output. Install the required packages by running 
```Rscript scripts/install_R_packages.R```

Cross-validation results and predicted locations can be plotted with 
```
Rscript scripts/plot_locator.R --infile out/jacknife --sample_data data/test_sample_data.txt --out out/jacknife/test --map F

```
This will plot predictions and uncertainties for 9 randomly selected individuals to `/out/jacknife/test_windows.png.` For lat/long coordinates you can also calculate and plot error estimates by using the `--error` option. See all parameters with 
```
Rscript scripts/plot_locator.R --help
```


# License

This software is available free for all non-commercial use under the non-profit open software license v 3.0 (see LICENSE.txt). Please contact cjbattey@gmail.com to license for commercial use.






