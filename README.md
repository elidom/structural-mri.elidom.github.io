## About this page

In this page I intend to summarise the usage of structural MRI workflows to estimate gray matter density (GMD) and cortical thickness (CT) from T1-weighted MRI volumes and carry out analyses to answer interesting research questions. My main interest here is to provide a detailed tutorial on CIVET (Montreal Neurological Institute) and RMINC (https://github.com/Mouse-Imaging-Centre/RMINC) to estimate vertex-based individual cortical thickness and later carry out group comparisons and correlations. However,I will also try to describe -- with lesser detail -- the usage of FSL (https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/) for Voxel-Based Morphometry (VBM) analyses and FreeSurfer (https://surfer.nmr.mgh.harvard.edu/) for CT analyses, especially with an aim towards doing custum region of interest (ROI) analysis.

## Dataset

For the examples we will be using the Empathic Response in Psychotherapists dataset, generated by [Olalde-Mathieu et al. (2020)](https://www.biorxiv.org/content/10.1101/2020.07.01.182998v2) and the structural analysis of which were carried out in [Domínguez-Arriola et al. (2021)](https://www.biorxiv.org/content/10.1101/2021.01.02.425096v2). You can copy the NIFTI volumes as well as the behavioral and demographic information from the directory: /misc/charcot2/dominguezma/tutorial

## CIVET

[CIVET](http://www.bic.mni.mcgill.ca/ServicesSoftware/CIVET-2-1-0-Introduction) is an sMRI processing pipeline developed by The McConnell Brain Imaging Centre to more reliably extract individual cortical surfaces at the vertex-level and estimate cortical thickness in milimeters at each point of the brain cortex. Several definitions of cortical thickness are available to use in the pipeline, and users will be available to choose the one they think is most appropritate. However, there is evidence that the linked-distance definition is more accurate and reliable than other geometric cortical-thickness definitions (Lerch & Evans, 2005). 

### Preprocessing

Users could simply feed the raw volumes to the pipeline; however, I recommend customizing the preprocessing of the volumes to ensure the best quality and most accurate results. This will consist in:

* A qualitative quality control of the volumes.
* [N4 Bias Correction](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC3071855/), and formatting file names.
* Generation of individual brain masks using [volBrain 1.0](https://volbrain.upv.es/).

#### Quality Control 

For this quality control I recommend the one described in [Backhausen et al. (2016)](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC5138230/). Please follow the steps reported in the paper. It basically consists in evaluating each volume in four different criteria, average their score, and decide -- on the basis of the individually asigned score -- whether to preserve or drop the volume for the rest of the workflow. It is important to be rigurous here because the presence of artifacts and overall bad quality can seriously bias the subsequent tissue segmentation and surface extraction.  

#### N4 Bias Field Correction and formatting file names

First, in order to run the *N4BiasFieldCorrection* algorithm ([ANTS](http://stnava.github.io/ANTs/) must be installed) you have to go to the folder where your NIFTI files are, unzip the volumes and, if you wanted to preprocess one file, run for instance: 

```bash
gunzip sub-1000_ses-1_T1w.nii.gz
N4BiasFieldCorrection -i sub-1000_ses-1_T1w.nii -o sub-1000_n4.nii
```
where -i specifies the input and -o the output name. Of course, we would ideally not want to process the volumes one by one, and would like to have the resulting volume in a dedicated directory. We may use a for loop for the former:

```bash
mkdir n4_corrected_output

for nii in *.nii; do
        id=`echo $nii | cut -d "_" -f 1` #extract subject ID
        N4BiasFieldCorrection -d 3 -i $nii -o n4_corrected_output/${id}_n4.nii #Perform correction
done
```

Finally, for the T1 volumes to be ready for the CIVET pipeline, they need to be transformed into MINC files and have a specific pattern in their names; so each file should look something like this: *PREFIX_ID_t1.mnc*, where __PREFIX__ is the study prefix (whichever we want it to be as long as it is consistently used throughout the whole workflow), __ID__ is the indivual volume's identifier, __t1__ tells CIVET that this is a T1-weighted MRI volume that needs processing, and __.mnc__ because it is a MINC file (not NIFTI anymore). For instance, here I would like to use the prefix **TA** and, so, my subject **sub-1000** should look something like this: *TA_1000_t1.mnc*. 

Let's do this in code. Suppose we are in the same directory as before -- i.e where the NIFTI files are, and where we now have a *n4_corrrected_output* folder filled with volumes. For the sake of tidyness I will move all the old NIFTI files to a new directory (since they are no longer useful; but we want them at hand as a backup) and I will create a new directory for the propperly formatted MINC volumes (note that you need to have the [MINC Toolkit](https://bic-mni.github.io/) installed -- or a virtual machine with it): 

```bash
mkdir NIFTI
mv *.nii NIFTI

mkdir mnc_files

for nii in $(ls n4_corrected_output); do
        id=`echo $nii | cut -d "_" -f 1` #extract subject ID
        nii2mnc $nii mnc_files/TA_${id}_t1.mnc
done
```

You can download or copy my script for these two last steps <a id="raw-url" href="https://github.com/elidom/structural-mri/blob/main/N4_formatting.sh" download>HERE</a>. 

#### Generation of brain masks with [volBrain 1.0](https://volbrain.upv.es/)

Even though the CIVET has implemented a brain extraction step (i.e. generation of a binary mask to be multiplied by the original image and strip out the skull, etc.), it might be safer to generate these masks ourselves with a very precise tool: [volBrain 1.0](https://volbrain.upv.es/). First, create a volBrain account (it is free!). Then, upload your NIFTI images one by one (note: they have to be compressed -- i.e. .gz termination) to the volBrain pipeline. Wait until its processing is finished (about 15 minutes; you should receive an email notification). Download the Native Space (NAT) zip file, extract the content, identify the __mask__ file (for instance: _native_mask_n_mmni_fjob293223.nii_)and save it somewhere separate. Be sure to change its name so that you can correctly associate it to its corresponding subject; in the end it will have to be similarly named to the CIVET pattern (PREFIX_ID_mask.mnc), so you might as well save it as, for instance, TA_1000_mask.nii. Before converting the masks into MINC files, carefully inspect that every mask fits the brain as perfectly as possible, so that in future steps only the cephalic mass is segmented from the volume. You can use __fsleyes__ for this, and [manually correct](https://users.fmrib.ox.ac.uk/~paulmc/fsleyes/userdoc/latest/editing_images.html) wherever needed. This quality control of the masks may take time, but it is absolutely necessary; otherwise, the reliability of the rest of the workflow would be compromised.

Finally, the mask files need to be converted into the MINC format. I will suppose that all the mask files are in one dedicated directory and are named like in the example above (e.g. TA_1000_mask.nii). Go to the directory where the masks are stored. You could transform them one by one:

```bash
nii2mnc TA_1000_mask.nii TA_1000_mask.mnc
nii2mnc TA_1001_mask.nii TA_1001_mask.mnc
```
...and so on; but of course, we always prefer to automatize the process:

```bash
for file in $(ls); do
        basename=`echo $file | cut -d "." -f 1`
        nii2mnc $file ${basename}.mnc
done

mkdir nii_masks mnc_masks
mv *.nii nii_masks
mv *.mnc mnc_masks
```
Now all the mask files should be exactly as CIVET asks them to be: PREFIX_ID_mask.mnc (e.g. TA_1001_mask.mnc). You can see the masks I generated in /misc/charcot2/dominguezma/tutorial/masks

