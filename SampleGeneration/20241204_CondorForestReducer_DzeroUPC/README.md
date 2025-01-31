# Requirements

* CMSSW_14_1_4_patch5
* ROOT v6.30
* g++ 12.3
* Enterprise Linux 9 (el9) with x86_64 architecture


# Setup

You will need an account for 
[MIT SubMIT](https://submit.mit.edu/submit-users-guide/index.html) and a 
[CERN/CMS certificate](https://uscms.org/uscms_at_work/computing/getstarted/get_grid_cert.shtml). 
If you need a SubMIT account you can [request one here](https://submit.mit.edu) 
through the SubMIT Portal. You can only use an ssh key to autheticate!
Passwords do not work.

Connect to Submit:
```bash
ssh <user>@submit.mit.edu
```

Navigate to your working directory and install CMSSW:
```bash
source /cvmfs/cms.cern.ch/cmsset_default.sh
cmsrel CMSSW_14_1_4_patch5
cd CMSSW_14_1_4_patch5/src/
cmsenv
git cms-merge-topic CmsHI:forest_CMSSW_14_1_X
cd -
```

Clone the MITHIGAnalysis2024 repository:
```bash
git clone --recursive git@github.com:ginnocen/MITHIGAnalysis2024.git
cd MITHIGAnalysis2024/
source SetupAnalysis.sh
cd SampleGeneration/20241204_CondorForestReducer_DzeroUPC/
source clean.sh
```

You should be ready. If you make changes to `ReduceForest.cpp`, `makefile`, or 
change anything under the `include/` folder or `MITHIGAnalysis2024/CommonCode/`,
then you will need to update the slimmed repo files on `T2_US_MIT`. Edit the 
paths in `CopyToT2.sh` and run with:
```bash
bash CopyToT2.sh
```

# Skimming with Condor

## Running

Refresh your VOMS proxy and run `RunSkimCondor.sh`:
```bash
voms-proxy-init -rfc -voms cms -valid 72:00
cp /tmp/x509up_u'$(id -u)' ~/

bash RunSkimCondor.sh
```

Check the status of condor jobs with:
```bash
condor_q
```

Kill or remove bad jobs with:
```bash
condor_rm <job_id>
```

Once jobs are complete (whether successful or failed), they will no longer
appear on the `condor_q` list.

Log files are saved to `condorConfigs/<YYYYMMDD>_<RUN>_<PD>/`:
```bash
# log of job and server status
<RUN>_HIForward<PD>_log_<job_id>_0.txt
# log of errors printed by the job scripts (some outputs will print here too)
<RUN>_HIForward<PD>_err_<job_id>_0.txt
# log of print statements from job scripts
<RUN>_HIForward<PD>_out_<job_id>_0.txt
```


## Making Changes

Edit `RunSkimCondor.sh` to select the runs and PD range to skim over. You can 
also edit `MakeSkimFileList.sh` and `MakeSkimCondor.sh` to change the Condor 
configuration and job script template without having to copy changed files 
over to T2.

> [!WARNING]
> _Only_ use non-numbered xrootd servers to transfer files, such as
> `root://xrootd.cmsaf.mit.edu/`! Specifying servers with numbers (e.g. 
> `xrootd10`) will not speed up transfers, and may damage the servers if used 
> for file transfers!


## Important Files

**RunSkimCondor.sh**

Loops over the configured list of runs and PDs and submits one job for each
run + PD combination. You can also edit this to change the source and output 
locations on `T2_US_MIT`.

**MakeSkimFileList.sh**

Makes a master list of file paths on `T2_US_MIT` that will be filtered and 
processed in jobs.

**MakeSkimCondor.sh**

Creates the Condor submission file and the bash script that is executed on
the Condor servers. The job script starts after `cat > $SCRIPT <<EOF1` and ends
at the line `EOF1`. The Condor config starts after `cat > $CONFIG <<EOF2` and
ends at `EOF2`.

**CopyToT2.sh**

Copies essential files from the local `MITHIGAnalysis2024` repo to `T2_US_MIT`,
where they are copied to every server at the start of a new job. This should
only be needed if `ReduceForest.cpp`, `makefile`, `include/` or 
`MITHIGAnalysis2024/CommonCode/` are changed.

# Useful Commands

## Condor

```bash
condor_submit <condor_config_file>
condor_q
condor_rm <job_id>
```


## xrootd

**CERN eos:** `root://eoscms.cern.ch/` `/store/group/phys_heavyions/<user>/`
**MIT T2:** `root://xrootd.cmsaf.mit.edu/` `/store/user/<user>/`

```bash
# Recursive list
xrdfs <server> ls -R -l <path>
# Make directory
xrdfs <server> mkdir -p <path/new_dir>

# Copy
xrdcp <server//path/new_dir> <local/path>
# Recursive copy for directory
xrdcp -r <server//path/dir> <local/path>
# Forced copy (overwrite)
xrdcp -f <server//path/file> <local/path>

# Delete file
xrdfs <server> rm <path/file.ext>
# Delete directory
xrdfs <server> rmdir <path/dir>
```


## VOMS

```bash
voms-proxy-init --rfc --voms cms -valid 72:00
voms-proxy-info
```

To make it easier to initiate proxies, add this to `~/.bashrc`:
```bash
alias proxy='voms-proxy-init -rfc -voms cms -valid 72:00 ; cp /tmp/x509up_u'$(id -u)' ~/ ;'
export PROXYFILE=~/x509up_u$(id -u)
```
Reset your terminal:
```bash
hash -r
source ~/.bashrc
```
From now on, you can initiate a new proxy with the command `proxy`!
