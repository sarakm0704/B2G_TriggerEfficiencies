
# B2G_TriggerEfficiencies

This repository contains the code to calculate trigger efficiencies that is used by the B2G trigger contacts.
It is using the orthogonal dataset method and mainly targets hadronic triggers, like our jet mass triggers (HLT_AK8PFJet400_MassSD30).

## Installation

The python code used in this framework requires a few packages (and their dependencies), which can be installed using pip or conda. The needed packages are the following:

- uproot
- coffea
- awkward
- numpy

Additionally, the `xrootd` package is required if files from the GRID are to be used.

If you want to execute the code on lxplus, a pre-existing environment can be used. However, a few requirements are not yet included in any stable release (afaik), primarily the `numba` version 0.55.2, that coffea needs, is not available. I was able to run the code using the latest development release, so for the moment one should be able to just do

`source /cvmfs/sft-nightlies.cern.ch/lcg/views/dev3/latest/x86_64-centos7-gcc11-opt/setup.sh`

and be able to run the module. This README will be updated once a working stable release is available. The only other required environment setting then is to activate your GRID proxy (if files should be accessed using xrootd):

`voms-proxy-init -voms cms -rfc`

Finally, to use the GRID proxy inside condor jobs, it needs to be copied over to your home directory, like

`cp /tmp/x509up_u103872 .`

This last step is not needed for local testing.

## Usage

### Locally

A single main python file is implemented that executes the desired code: runTriggerEfficiencies.py

This file can be called as `python runTriggerEfficiencies.py`, and requires several arguments afterwards:

- -i or --input: a text file containing a list of input files to run on (one file per line)
- -r or --refTriggers: a text file containing the reference triggers (one trigger per line)
- -t or --testTriggers: a text file containing the triggers to be measured (one trigger per line)
- -e or --era: the data-taking era (a string, for example "22RunD"). Might be auto-detected later

Two additional, optional flags can be set:
- --doJECs: apply jet energy corrections according to the era
- --useGoldenJSON: apply a goldenJSON selection

A complete example command thus could be
`python runTriggerEfficiencies.py -i example/inputfiles.txt -r example/refTriggers.txt -t example/testTriggers.txt -e 22RunD --doJECs --useGoldenJSON`

The output, called `output.root` , can then be plotted by using

`python plotEfficiencies.py output.root`

### Condor

To run the trigger efficiencies on more files, the HTCondor batch system is used. Example scripts are given in the repository, and an exemplary workflow is shown here. Note that this is written to work on CERN lxplus, and might require changes to run on other systems.

First, prepare the required input files: two text files with reference and test triggers (as explained above), and the input file list. If the input sample is on DAS, this can be done automatically, for example using

`./createFileList.sh "/Muon/Run2022C-PromptNanoAODv10_v1-v1/NANOAOD" example/condor/filelist.txt`

which will produce a single file with all input files corresponding to the passed DAS query. Note that no redirector is added yet, this can be done using

`python add_xrootd_redirector.py example/condor/filelist.txt "root://xrootd-cms.infn.it/"`.

Finally, you most likely want to split this list of input files in multiple jobs. Create a directory where the split files should be located, and execute

`python split_filelist.py example/condor/filelist.txt 10 example/condor/split_files/`

where in this example, we'll create 10 condor jobs.
Now, all input files are ready, so (copy and) adjust `condor.sub` with the required information (this step might be automated later). Then, create a `log` directory to store job logs, and run your jobs using

`condor_submit condor.sub`

When the jobs have run through, you'll have several files named `output_*.root` in the directory where you executed the submit command. Combine these using

`hadd output_total.root output_*.root`.

Then, you can run before/after histograms, as well as efficiency plots, using

`python plotEfficiencies.py output_total.root`

## Corrections / goldenJSON
Jet energy corrections are applied according to the files in the `data` directory, based on the files obtained from the [JERC group recommendations](https://cms-jerc.web.cern.ch/Recommendations/). In case new ones need to be added, these can be added to the `data/corrections` directory. There, a helper script ([ConvertCorrectionFiles.py](https://github.com/finnlabe/B2G_TriggerEfficiencies/blob/master/data/corrections/ConvertCorrectionFiles.py "ConvertCorrectionFiles.py")) is given to obtain the correct file naming.

The golden JSON selection is made based on the file in `data/goldenJSON`, where new ones can be added.
