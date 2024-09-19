## Modules

Load the following modules
```
module load gcc/11.3.0
module load cmake/3.26.3/gcc/11.3.0/zen2
module load openmpi/gcc/11.3.0/zen2/4.1.5
module load python/gcc/11.3.0/linux-rhel8-zen2/3.10.10
module load intel-mkl/gcc/11.3.0/openmpi/4.1.5/zen2/2023.0.0
module save pysf
```
where the last line saves these module settings. Later you can quickly load all the modules above by
```
module restore pyscf
```

## Compile PySCF

Once appropriate modules are loaded, compiling PySCF from source should be straightforward by following instrutions from the [manual](https://pyscf.org/install.html#build-from-source), which are reproduced here.
1. Clone the PySCF repository:
```shell
git clone https://github.com/pyscf/pyscf.git
cd pyscf
```
2. Compile (change `ncore` if needed):
```shell
ncore=4
rm -rf build
mkdir -p build
cmake ../
make -j$ncore
```

You can check if PySCF is appropriately installed by
```shell
export PYTHONPATH=[your_path_to_pyscf]:$PYTHONPATH
python -c "import pyscf"
```
You should see no errors.


## Submitting jobs

Let's do a HF calculation for a single water molecule in cc-pVDZ basis set. Save the following to `hf_water.py`:
```python
from pyscf import gto, scf

atom = '''
O          0.00000        0.00000        0.11779
H          0.00000        0.75545       -0.47116
H          0.00000       -0.75545       -0.47116
'''
basis = 'cc-pvdz'

mol = gto.M(atom=atom, basis=basis).set(verbose=4)
mf = scf.RHF(mol).run()
```
Save the following to `submit.sh`:
```shell
#!/bin/bash

#SBATCH -e sbatch.err
#SBATCH -o sbatch.log
#SBATCH -n 1
#SBATCH -c 4
#SBATCH -t 1:00:00
#SBATCH --mem-per-cpu=4gb
#SBATCH --mail-type=BEGIN,END
#SBATCH --mail-user=[youremailaddress]

module restore pyscf

echo "after module load"
echo $PYTHONPATH
echo

export PYTHONPATH=[your_path_to_pyscf]:${PYTHONPATH}

export OMP_NUM_THREADS=4
export MKL_NUM_THREADS=1

python hf_water.py 2> err > stdout
```
Then
```shell
sbatch -J hf_water submit.sh
```
You will see a message
```
Submitted batch job 7897676
```
To check the status of your job, do
```shell
squeue -u [your_user_name]
```
which gives
```
JOBID    PARTITION         NAME     USER ST       TIME  NODES NODELIST(REASON)
7897676  standard      hf_water     hzye PD       0:00      1 (Priority)
```
where `PD` indicates the job is pending. After some time the status will change to
```
JOBID    PARTITION         NAME     USER ST       TIME  NODES NODELIST(REASON)
7897676  standard      hf_water     hzye  R       0:05      1 compute-a5-10
```
where `R` indicates the job is running.


## Installing LNO

Clone the lno repository and checkout the `ccsd_t_clean` branch
```shell
git clone https://github.com/hongzhouye/lno.git
cd lno
git checkout -b ccsd_t_clean origin/ccsd_t_clean
```
To use LNO-CCSD(T), follow the instruction in `lno/cc/pyscflib/ccsd_t.c` to update the PySCF `ccsd_t.c` code and then recompile PySCF. Note that you do not need to remove the `build` directory. Simply do
```shell
ncore=4
cd build
cmake ../
make -j$ncore
```