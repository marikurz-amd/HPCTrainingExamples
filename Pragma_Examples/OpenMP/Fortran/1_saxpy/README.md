
## First Fortran OpenMP offload: Porting saxpy step by step and explore the discrete GPU and APU programming models:

README.md from `HPCTrainingExamples/Pragma_Examples/OpenMP/Fortran/1_saxpy` in the Training Examples repository

This exercise will show in a step by step solution how to port your first kernels. 
This simple example will not use a Makefile to practice how to compile for the GPU or APU. 
All following exercises will use a Makefile.


> [!TIP]
> There are 7 different enumerated folders.
> Use this command to see the differences between them:
> ```
> vimdiff saxpy.f90 ../<X_saxpy_version>/saxpy.f90
> ```

First, prepare the environment (load modules, set environment variables), if you didn't do so before.

### Part 1: Porting with unified shared memory enabled
For now, set
```
export HSA_XNACK=1
```
and load the amdflang-new compiler module
```
module load amdflang-new
```
to make use of the APU programming model (unified memory).
#### 0) The serial CPU code.
```
cd 0_saxpy_serial_portyourself
```
Try to port this example yourself. If you are stuck, use the step by step solution in folders 1-6 and read the instructions for those exercises below. Recommendation for your first port: use ```!$omp requires unified_shared memory``` (in the code after ```implicit none``` in each module) and ```export HSA_XNACK=1``` (before running) that you do not have to worry about map clauses for now. Steps 1-3 of the solution assume unified shared memory. Later steps will introduce map clauses and investigate the behaviour of ```export HSA_XNACK=0``` or ```=1```.

- Compile the serial version. Note that ```-fopenmp``` is required as omp_get_wtime is used to time the loop execution:
```
amdflang -fopenmp saxpy.F90 -o saxpy
```
- Run the serial version:
```
./saxpy
```
You can now try to port the serial CPU version to the GPU or follow the
step by step solution:

#### 1) Move the computation to the device
```
cd ../1_saxpy_omptarget
vi saxpy.f90
```
add ```!$omp target``` to move the loop in the saxpy subroutine to the device.
- Compile this first GPU version. Make sure you add ```--offload-arch=gfx942``` (on MI300A, find out what your system's gfx... is with ```rocminfo```)
on aac6 or aac7 with amdflang-new:
```
amdflang -fopenmp --offload-arch=gfx942 saxpy.F90 -o saxpy
```

To compile it on aac7 with ```ftn```, make sure the correct module is loaded and offload is enabled before you compile with
```
ftn -fopenmp saxpy.F90 -o saxpy
```
- Run
```
./saxpy
```
The observed time is much larger than for the CPU version. More parallelism is required!

#### 2) Add parallelism
```
cd ../2_saxpy_teamsdistribute
vi saxpy.f90
```
add the ```teams distribute``` directive and
- compile again
- run again
The observed time is a bit better than in case 1 but still not the full parallelism is used.

#### 3) Add multi-level parallelism
```
cd ../3_saxpy_paralleldosimd
vi saxpy.f90
``` 
add ```parallel do``` for even more parellelism and
- compile again
- run again
The observed time is much better than all previous versions.
Note that the initialization kernel is a warm-up kernel here. If we do not have a warm-up kernel, the observed performance would be significantly worse. Hence the benefit of the accelerator is usually seen only after the first kernel. You can try this by commenting out the ```!$omp target...``` in the initialize subroutine, then the measured kernel is the first which touches the arrays used in the kernel.

### Part 2: Explore the impact of unified shared memory
#### 4) Explore the impact of unified memory
```
cd ../4_saxpy_nousm
vi saxpy.f90
```
The ```!$omp requires...``` line is removed.
- compile again
- run again
So far, we worked with unfied shared memory and the APU programming model. This allows good performance on MI300A, but not on discrete GPUs. In case you will work on discrete GPUs or want to write portable code for both discrete GPUs and APUs, you have to focus on data management, too. Use
```
export HSA_XNACK=0
```
to get behaviour similar to discrete GPUs (with memory copies).
Compiling and running this version without any map clauses will result in much worse performance than with unified shared memory and ```HSA_XNACK=1``` (no memory copies on MI300A).

### Part 3: Using map clauses
Set
```
export HSA_XNACK=0
```
for map clauses to have an effect on MI300A.

#### 5) Introduce map clauses for each kernel.
```
cd ../5_saxpy_map 
vi saxpy.f90
```
see where the map clasues where added. The x vector only has to be maped ```to```.
- compile again
- run again
The performance is not much better than version 4.

#### 6) Optimize data transfers
With the ```enter data``` and ```exit data``` clauses the memory is only moved once at the beginning. Thus, the time to solution should be roughly in the order of magnitude of the unified shared memory version, but still slightly slower as the memory is copied like on discrete GPUs. Test yourself:
```
cd ../6_saxpy_targetdata
vi saxpy.f90
```
- compile again
- run again

**Additional exercise**: What happens to the result, if you comment the ```!$omp target update``` (in line 29)? 
```
vi saxpy.f90
```
- Don't forget to recompile after commenting it.

The results will be wrong! This shows, that proper validation of results is crutial when porting! Before you port a large app, think about your validation strategy before you start. Incremental testing is essential to capture such errors like missing data movement.

#### 7) Experiment with ```num_teams```
```
cd ../7_saxpy_numteams
vi saxpy.f90
```
specify ```num_teams(...)``` and choose a number of teams you want to test 
- compile again
- run again
Investigating different numbers of teams you will find that the compiler default (without setting this) was already leading to good performance. saxpy is a very simple kernel, this finding may differ for very complex kernels.

After finishing this introductory exercise, go to the next exercise in the Fortran folder:
```
cd ../..
```

