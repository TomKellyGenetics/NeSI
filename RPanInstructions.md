# How to get your R work done on NeSI's Pan cluster

1.  Prerequisites: Work through the companion document [OtagoPanInstructions](https://rawgit.com/dannybaillie/NeSI/master/OtagoPanInstructions.html) first.
2.  Type `nano simplefunction.R` and Copy and Paste the following (it makes a random matrix and displays the result of a calculation), write out and exit:

    ```
    #!/usr/bin/env Rscript
    args = commandArgs(trailingOnly=TRUE)

    simplefunction <- function(n){
    n <- as.numeric(n)
    a <- matrix(data = rnorm(n^2), nrow=n, ncol = n)
    save(a, file="simple.RData)
    return(2^10)
    }
    
    x <- simplefunction(args$1)
    print(x)

    ```

3.  Type `nano simpleR.sl` and Copy and Paste the following (you need to replace `projectIDhere` with your project ID from [OtagoPanInstructions](https://rawgit.com/dannybaillie/NeSI/master/OtagoPanInstructions.html)), write out and exit:

    ```

    #!/bin/bash
    #SBATCH -J yourjobName
    #SBATCH -A projectIDhere
    #SBATCH --time=1:00:00
    #SBATCH --mem-per-cpu=2G
    #SBATCH --output=simpleRout.txt
    #SBATCH --error=simpleRerr.txt
    module load R/3.1.1-goolf-1.5.14
    srun Rscript simplefunction.R '20'

    ```

4.  Submit the Slurm script you created in step 2 by typing: `sbatch simpleR.sl`.
5.  When your job is finished, you should find three new files:
    *   `simple.RData`: an RData file with a 20 by 20 random matrix in it
    *   `simpleRout.txt`: including the text "1024" from calculating 2^10
    *   `simpleRerr.txt`: empty file
6.  You copy the file `simple.RData` to your local machine (see [OtagoPanInstructions](https://rawgit.com/dannybaillie/NeSI/master/OtagoPanInstructions.html)) or look at it on the build node using:

    ```
    ssh build-sb 
    module load R/3.1.1-goolf-1.5.14
    R
    load("simple.RData")
    print(a)

    ```

7.  Changing your code to work on Pan
    *   Inputs: The code you want to run will be an R script or function. If it is a script then it has no direct inputs (perhaps it will load a .mat file, or all parameters will be set in the script). If it is a function then inputs will be received by R as text strings. If you need numeric inputs, you'll need to use e.g. `as.numeric` (as we did in step 2 above).
    *   Outputs: It is not useful for your function to return explicit outputs: either use the text output (what you normally see on the screen which goes into the file you specify with `#SBATCH --output=`) or (much more likely) save results, e.g. to a `.mat` file (as we did in step 1).
    *   To use more than one core, add the following to your `.sl` script (change the 8 to the number of CPUs your code can use):

        ```

        #SBATCH --cpus-per-task=8
        #SBATCH --ntasks=1

        ```

9.  The script you use will also need to be modified to run R on multiple cores. We recommend using the R module demonstrated on Pan, which uses the `snow` packages and it's `Rmpi` dependancy to take advantage of the the MPI (message passing interface supported by NeSI). A larger list of modules is avaialable of on the [NeSI Wiki](https://github.com/nesi/applications/wiki/R) or you can contact the [NeSI Support Team by email](mailto:support@nesi.org.nz). For instance, with the `simplefunction.R` script, make the following changes to load the `snow` package and run the function on every core:

First modify the process to divide independent steps so they can be allocated to separate cores. For example:

a) Generate each column of the matrix as a random vector (data intensive steps are particularly well suited to this).

b) Run multiple calculations at the same time, e.g., 2^i below

    ```
    #!/usr/bin/env Rscript
    args = commandArgs(trailingOnly=TRUE)

    simplefunction <- function(n){
    n <- as.numeric(n)
    a <- lapply(as.list(1:n), function(i) matrix(data = rnorm(n), nrow=1, ncol = n)) # generate each column on a separate core
    b <- do.call(rbind, a) #combine output vectors into a matrix 
    save(b, file="simple.RData)
    c <- sapply(1:n, function(ii) 2^ii) #do separate calculations
    return(c)
    }
    
    x <- simplefunction(args$1)
    print(x)
    ```

In both of these cases we have used the vectorisation of R to pass a function to multiple arguments in a loop. These "apply" functions have parallel counterparts in the `snow` package.

    ```
    #!/usr/bin/env Rscript
    args = commandArgs(trailingOnly=TRUE)
    library("snow") #load package

    simplefunction <- function(n){
    n <- as.numeric(n)
    
    cl<-makeMPIcluster(outfile="") #First create a cluster to send functions to (without specifying cores)
    clusterExport(cl=cl, list=ls()) #Send environment variables to end node
    
    a <- parLapply(cl, as.list(1:n), function(i) matrix(data = rnorm(n), nrow=1, ncol = n)) #invoke the parallel version of lapply() and run the function on separate inputs simultaneously, the output here returns to the master node environment
    b <- do.call(rbind, a)
    save(b, file="simple.RData)
    c <- parSapply(cl, 1:n, function(ii) 2^ii) #we can use this cluster again with sapply in a similar manner
    stopCluster(cl) #remember to stop a cluster in R once you are done
    return(c)
    }
    
    x <- simplefunction(args$1)
    print(x)
    ```

Note that you **do not** have to specify how many cores to use when creating the cluster in R, this defaults to the maximum number of cores available to your NeSI job (i.e., the "cpus-per-task" you requested in the slurm file).

##Installing R packages on NeSI

Many R packages are already installed in a module on NeSI and can be loaded with the `library()` command in an Rscript (such as `snow` and `Rmpi` as we have already used). However if the package you wish to use is not installed on NeSI, you can run the `install.R` script here with `sbatch install.sl` with the package you wish to use in place of "package_name" in `install.sl` (edit with `nano install.sl` and upload to NeSI with `scp install.sl pan:path_to_project_dir`). This should work for any R package hosted by CRAN.

Once installed (you should only need to do this once per package), future jobs will be able to call on this package with `library` as you would on a local machine. The `nz_repo` variable in `install.R` could be changed for a bioconductor, github, or sourceforge package but if this doesn't work (or your package requires other dependancies, e.g, `apt-get install package_name`, then it's advisable to can contact the [NeSI Support Team by email](mailto:support@nesi.org.nz) for further assistance. Many of the large domain-specific resources (such as NCBI BLAST databases) are hosted on NeSI already.

See this guide for scripts and more information to install CRAN, Bioconductor, and GitHub R packages via slurm:

* https://github.com/TomKellyGenetics/install.nesi 
