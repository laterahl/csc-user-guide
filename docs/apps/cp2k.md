# CP2K

Versatile ab initio and classical molecular dynamics. CP2K is suited for large parallel quantum chemistry calculations, in
particular for AIMD.

[TOC]

## Available

* Puhti: 6.1
* Mahti: 8.2, 9.1

## License

CP2K is freely available under the GPL license.

## Usage

Check which versions can be loaded directly:

    module avail cp2k

You can find all installed versions with:

    module spider cp2k

With each new project make sure that your job can efficiently
utilize all the cores you request in the batch script.

### Example batch script for Puhti using MPI-only parallelization

```bash
#!/bin/bash
#SBATCH --time=00:10:00
#SBATCH --ntasks-per-node=40
#SBATCH --nodes=2
#SBATCH --mem-per-cpu=2GB
#SBATCH --partition=large
#SBATCH --account=<project>

module load cp2k

srun cp2k.popt H2O-32.inp > H2O-32.out
```

### Example batch script for Mahti using mixed MPI-OpenMP parallelization

```bash
#!/bin/bash
#SBATCH --time=00:05:00
#SBATCH --ntasks-per-node=32  # 2 - 128
#SBATCH --cpus-per-task=4     # 128 / ntasks-per-node
#SBATCH --nodes=2
#SBATCH --partition=test
#SBATCH --account=<project>

module purge
module load gcc/9.4.0 openmpi/4.1.2
module load cp2k/9.1

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
export OMP_PLACES=cores

srun cp2k.psmp H2O-64.inp > H2O-64.out
```

### Performance notes

**Mahti:**

The following table shows the total execution time [s] for the [H2O-64 benchmark](https://github.com/cp2k/cp2k/blob/master/benchmarks/QS/H2O-64.inp)
in Mahti using cp2k/9.1. The column headers show how many omp-threads were used per mpi-task.

Nodes|d1|d2|d4|d8
-|--|--|--|--
1|25.77|18.88|20.58|22.95
2|17.66|15.25|*13.34*|17.10

* For 64 water molecules, the best performance is obtained with 2 full nodes, 32 mpi-tasks, and 4 OMP-threads per task (like the [Mahti example](#example-batch-script-for-mahti-using-mixed-mpi-openmp-parallelization)). For this system the performance does not scale beyond 2 nodes.
* Mixed parallization is efficient: choose tasks and threads so that they add up to 128 (physical) cores available per node (or up to 40 on Puhti).
* Test for optimal run parameters for your model system and method.
* Using the ELPA library may be faster with (metallic) systems that require large matrix diagonalizations for SCF, but there has recently been concerns about the reliability of ELPA, [see issue on GitHub](https://github.com/cp2k/cp2k/issues/1444). Also, recent versions (2021) of ELPA do not benefit from OpenMP.

### High-throughput computing with CP2K

High-throughput computations can be run conveniently with CP2K using the built-in `FARMING` program. This is an excellent option for use cases where the aim is to run a large amount of independent computations, such as when generating data for AI/ML pipelines. All subjobs are run in parallel within a single Slurm allocation, thus avoiding excess calls of `srun` or `sbatch` which decreases the load on the batch queue system.

Running `FARMING` jobs requires an additional input file in which the details of the workflow, such as the input directories and number of parallel task groups, are specified. Example `farming.inp` and `farming.sh` input and batch script files are provided below. Note that `RUN_TYPE` is set to `NONE` in the `&GLOBAL` section.

```
&GLOBAL
  PROJECT my-farming-job
  PROGRAM FARMING
  RUN_TYPE NONE
&END GLOBAL
&FARMING
  NGROUPS 8
  &JOB
    DIRECTORY run1
    INPUT_FILE_NAME nacl.inp
  &END JOB
  &JOB
    DIRECTORY run2
    INPUT_FILE_NAME nacl.inp
  &END JOB
  &JOB
    DIRECTORY run3
    INPUT_FILE_NAME nacl.inp
  &END JOB
  &JOB
    DIRECTORY run4
    INPUT_FILE_NAME nacl.inp
  &END JOB
  &JOB
    DIRECTORY run5
    INPUT_FILE_NAME nacl.inp
  &END JOB
  &JOB
    DIRECTORY run6
    INPUT_FILE_NAME nacl.inp
  &END JOB
  &JOB
    DIRECTORY run7
    INPUT_FILE_NAME nacl.inp
  &END JOB
  &JOB
    DIRECTORY run8
    INPUT_FILE_NAME nacl.inp
  &END JOB
&END FARMING
```

```bash
#!/bin/bash -l
#SBATCH --time=00:30:00
#SBATCH --partition=medium
#SBATCH --ntasks-per-node=128
#SBATCH --nodes=1
#SBATCH --account=<project>

module purge
module load gcc/9.4.0 openmpi/4.1.2 cp2k/9.1

srun cp2k.psmp farming.inp >> farming.out
```

In this particular example, one full Mahti node is requested for running 8 single-point calculations of a NaCl crystal with different lattice constants. In addition to the `farming.inp` input, each subjob requires its own ordinary input file, which are here organized into separate subdirectories named `run*`. Issuing `sbatch farming.sh` in the parent directory launches all calculations in parallel, allocating 16 cores to each subjob.

Note that dependencies can also be specified between subjobs using the `DEPENDENCIES` and `JOB_ID` keywords under the `&JOB` section. This enables the definition of complex workflows. For further details, see the [CP2K manual](https://manual.cp2k.org/trunk/CP2K_INPUT/FARMING.html) and [regtest files for example inputs](https://github.com/cp2k/cp2k/tree/master/tests/FARMING).

## References

CP2K prints out a list of relevant publications towards the end of the
log file. Choose and cite the ones relevant to the methods you've used.

## More Information

* CP2K online manual: <http://manual.cp2k.org/>
* CP2K home page: <http://www.cp2k.org/>
    * Contains [tutorials](https://www.cp2k.org/howto) and links to useful [tools](https://www.cp2k.org/tools)
* [Regtest inputs](https://github.com/cp2k/cp2k/tree/master/tests) can be used as examples on how to use the different features in CP2K. Note that the convergence criteria can be quite loose and should be separately tested for production simulations.
