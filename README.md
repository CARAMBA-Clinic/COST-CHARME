# Metabolomics data analysis using Pachyderm
In this page we introduce an metabolomics preprocessing workflow that you can run using [Pachyderm](https://github.com/pachyderm/pachyderm), a distributed data-processing tool built on software containers that enables scalable and reproducible pipelines.

## Requirements
You need to install Vagrant and Virtualbox.

In Windows:

- Install Chocolate: https://chocolatey.org/install
- Install Vagrant: https://www.vagrantup.com/downloads.html
- Install VirtualBox: https://www.virtualbox.org/wiki/Downloads

- Open PowerShell and run: 
```bash
> choco install vagrant
```
In Mac & linux: 

You should be able to download the installation files for your operating system:

- Install Vagrant: https://www.vagrantup.com/downloads.html
- Install VirtualBox: https://www.virtualbox.org/wiki/Downloads

## Set up the vagrant box
The box will be distributed through USB sticks. It is also available to download from: https://drive.google.com/drive/folders/0B-UIZY1NU7VzaUpCTlJpclpjbUE

Using PowerShell or Terminal, navigate to the folder containing the box file and run:
```bash
> vagrant up --provision
```
SSH into the machine 
```bash
> vagrant ssh
```
Check jupyter notebook and Rstudio:
Open a web browser and navigate to [localhost:8888](localhost:8888) for jupyter notebook and [localhost:8787](localhost:8787) for Rstudio.

If you need to destroy your machine use:
```bash
> vagrant destroy
```

## Hands-on with Pachyderm

### Useful information
The most common way to interact with Pachyderm is by using the Pachyderm Client (pachctl). You can explore the different commands available by using:
```bash
> pachctl  --help
```
And if you need more information about a particular command please use:
```bash
> pachctl <name of the command> --help
```

> **Tip:** To do this exercise you can either use the Jupyter notebook terminal (go to New -> Terminal) on localhost:8888 or you can SSH into the machine to interact with the system.


### Pushing data into Pachyderm's Data Repository
A repo is the highest level data primitive in Pachyderm. They should be dedicated to a single source of data such as the input from a particular tool. Examples include training data for an ML model or genome annotation data.
Here we will create a single repo which will serve as input for the first step of the workflow:
```bash
> pachctl create-repo mrpo
```
You can push data into this repository using the put-file command. This will create a new commit, add data, and finish the commit. Explore further on how commits work. The mass spectrometry raw data files are located in the MSData folder. First navigate to MSData: 
```bash
> cd ./MSData
```
Now push the data into the repository you created in the previous step:

```bash
> pachctl put-file <name of the repo> <name of the branch> -c -r -p <number of files to upload in parallel> -f .
```
### Running a Pachyderm pipeline
Once your data is in the repository, you are ready to start a bunch of pipelines cranking through data in a distributed fashion. Pipelines are the core processing primitive in Pachyderm and they’re specified with a JSON encoding. Explore the pipelines folder and find out which of the pipelines is the first step of the pre-processing workflow. You can find it by discovering which pipeline has the previously created repository as an input. Have a look at the input section in the JSON files in the pipelines folder:
```JSON
 "input": {
    "atom": {
      "repo": "",
      "glob": ""
    }
  }
```
Which one reads from the original repository and not from other tools ? Figure it out and then run it using:
```bash
> pachctl create-pipeline -f <JSON file>
```
What happens after you create a pipeline? Creating a pipeline tells Pachyderm to run your code on every finished commit in a repo as well as all future commits that happen after the pipeline is created. Our repo already had a commit, so Pachyderm automatically launched a job (Kubernetes pod) to process that data. This first time it might take some extra time since it needs to download the image from a container image registry. You can view the pipeline status and its corresponding jobs using:
```bash
> pachctl list-job
```
and 

```bash
> pachctl list-pipeline
```
And explore the different worker pods in your Kubernetes cluster via:
```bash
> kubectl get pods -o wide
```
Try changing some parameters such as the parallelism specification, resource specification and glob pattern. What is happening? How many pods are scheduled? Play with the parameters and see the differences. You can learn about the different settings here: http://docs.pachyderm.io/en/v1.5.0/reference/pipeline_spec.html

You can re-run the pipeline with a new pipeline definition (new parameters etc) like this:
```bash
> pachctl update-pipeline -f <JSON file>
```
Three more pipelines compose the pre-processing workflow. Find your way and run the rest of the pipeline (TextExporter is the last step).   After you run the entire workflow, the resulting CSV file generated by the TextExporter in OpenMS will be saved in the TextExporter repository. You can download the file simply by using:
```bash
> pachctl get-file TextExporter <commit-id> <path-to-file-in pachd> > <custom-name-of-file>
```
The <commit-id> is easily obtainable by checking the most recently made commit in the TextExporter repository using:
```bash
> pachctl list-commit TextExporter
```
Also, the <path-to-file> can be obtained by checking the list of files outputted to the TextExporter repository at a specific branch. To which branch does Pachyderm make commits by default?
```bash
> pachctl list-file <name-of-repo> <branch-name>
```
### Data versioning in Pachyderm
Pachyderm uses a Data Repository within its File System. This means that it will keep track of different file versions over time, like Git. Effectively, it enables the ability to track the provenance of results: results can be traced back to their origins at any time point.

Pipelines automatically process the data as new commits are finished. Think of pipelines as being subscribed to any new commits on their input repositories. Similarly to Git, commits have a parental structure that tracks which files have changed. In this case we are going add some more metabolite data files.

Let’s create a new commit in a parental structure. To do this we will simply do two more put-file commands with -c and by specifying master as the branch, it will automatically parent our commits onto each other. Branch names are just references to a particular HEAD commit.
```bash
> cd ./MSDataNew
```
```bash
> pachctl put-file <name of the repo> <name of the branch> -c -r -p <number of files to upload in parallel> -f .
```
Did any new job get triggered? What data is being processed now? All available data or just new data? Explore which new commits have been made as a result of the new input data. 
```bash
> pachctl list-commit <repo-name>
```
## Analyze your data in RStudio
After exporting the TextExporter csv file, you are ready to do some downstream analysis.
On your host machine, open a browser tab and go to localhost:8787 (password and username is "vagrant"). There you will find the common interface of RStudio, where we have placed the following code to get you started. 
```R
# required R packages
library(R.utils)
library(ggplot2)
library(ropls)
.
.
inputFile = ""
outputFile = ""
.
.
ggsave(outputFile, plot = plot.plsda, width = 10, height = 10)
```

Essentially, this code will take the output from the TextExporter as an input, and generate a partial least squeare discriminant analysis (PLS-DA) score plot as an output. Features with a coverage higher than 75% across all samples will be considered and missing values will simply be imputed by zeros.

You need to change "inputFile" to the path of the file you exported from TextExporter and pick an arbitrary name for your output.
Now select the whole code inside the code editor and run using <kbd>Ctrl+Enter</kbd> or press source in the top right corner. 
You can explore your dataset using RStudio functionalities such as view(data_parsed) or by simply clicking the the variables in the environment window. Or how about doing a boxplot to see the distribution of the samples?
```R
boxplot(log2(data_parsed[,-c(1,2)]))
```
## Optional: How to develop a simple R-based microservice with Docker

In this section we show you how to wrap your R-script in a Docker image and integrate it into your workflow using Pachyderm. 
This is a slightly modified version of the R script that you just used. The modification accounts for the need to be able to feed user specified input and output filenames through command line. Assuming that teh following R script has been saved as *plsda.r*, it ca be run using `plsda.r input output`. Where *input* is the path to the result from the TextExporter and *output* is the desired name of your plsda image.

```R
#!/usr/bin/env Rscript

options(stringAsfactors = FALSE, useFancyQuotes = FALSE)

# required R packages
library(R.utils)
library(ggplot2)
library(ropls)

source('/usr/local/bin/Functions.r')

# Get arguments
args <- commandArgs(trailingOnly = TRUE)

inputFile = args[1]
outputFile = args[2]

# Read the TextExporter output
nL <- countLines(inputFile)
nL = read.csv(inputFile, 
                  skip=nL-1,
                  fill=TRUE,
                  sep="")
data = read.csv(inputFile, 
                       header = FALSE, comment.char="#",
                       stringsAsFactors = FALSE, 
                       fill=TRUE, 
                       col.names= paste0("V",seq_len(length(nL))), 
                       sep="")

# Parse the data
data_parsed <- Parse(data)
cat("Size of the matrix:", dim(data_parsed), "\n")

# Rename and transform values from string to numeric
names(data_parsed) <- gsub(".featureXML","",names(data_parsed))
data_parsed[data_parsed=="NaN"]=NA
data_parsed <- data.frame(apply(data_parsed,2,as.numeric))

# Remove features with coverage less than 75%
data_cov <- data_parsed[coverage(data_parsed,0.75),]
data_cov[is.na(data_cov)] <- 0

# Perform PLSDA
Y <- substr(names(data_cov),0,3)[-c(1,2)]
X <- data_cov[,-c(1,2)]

data.plsda <- opls(t(X), Y, predI = 2, scaleC='standard')

plotdata <- data.frame(data.plsda@scoreMN, Y)

# Generate plot
plot.plsda <- ggplot(plotdata, aes(x=p1, y=p2,shape=Y, color=Y)) + geom_point(size=4) +
  xlab("t1") + ylab("t2")  + theme_bw() 

ggsave(outputFile, plot = plot.plsda, width = 10, height = 10)
```

In order to create a Docker container we follow the scheme recommended by Phenomenal (for guidance see the dockerized xcms R package: https://github.com/phnmnl/container-xcms). First, the required linux and R packages must be installed. The provided R scripts (see scripts folder) should be placed in a separate folder which is added to the appropriate folder inside the container and granted execution permission. Now, all you need to do in order to wrap your R script in a Docker image is to write a [Dockerfile](https://docs.docker.com/engine/reference/builder/). In order to do that, please have a look at the Dockerfile in the example.


> **Tip:** To do this exercise you can either use the Jupyter notebook terminal on localhost:8888 or you can SSH into the machine to navigate and manage your structure.


```Docker
FROM r-base
MAINTAINER Stephanie Herman, stephanie.herman@medsci.uu.se

# Fill out the lines by looking at the given example
RUN #### install the needed linux and R packages
ADD #### add all scripts to container
RUN #### give execution permission to the R scripts
```

In the Dockerfile you first specify a base image that you want to start **FROM**. If you are working to an R-based service, like we are doing, the base image *r-base* is a good choice, as it includes all of the dependencies you need to run your script. Then, you provide the **MAINTAINER**, that is typically your name and a contact.

The last two lines in our simple Docker file are the most important. The **ADD** instruction serves to add a file in the build context to a directory in your Docker image. In fact, we use it to add our *plsda.r* script in the root directory. Finally, the **RUN** instruction, specifies which command to execute and commits the results, when the container will be started. Of course, we use it to run our script.

When you are done with the Dockerfile, you need to build the image. The `docker build` command does the job. 

```
$ docker build -t plsda .
```

In the previous command we build the image, naming it *plsda*, specifying the current directory as the build context. To successfully run this command, it is very important that the build context, the current directory, contains both the *Dockerfile* and the *scripts* folder. If everything works fine it will say that the image was successfully built.

To verify that your image works correctly and as excpected, you can use the `docker run` command, which serves to run a service that has been previously built.

```
$ docker run -v /host/directory/data:/data plsda /data/textexporter.csv /data/plsda.png
```

In the previous command we use the `-v` argument to specify a directory on our host machine, that will be mount on the Docker container (note that the full path needs to be provided). This directory is supposed to contain the output of TextExporter. Then we specify the name of the container that we aim to run (*plsda*), and the arguments that will be passed to the **RUN** command. We mounted the host direcory under */data* in the Docker container, hence we use the arguments to instruct the R script to read/write the input from/to it.    

You can read more on how to develop Docker images on the Docker [documentation](https://docs.docker.com/). 

### Integrate your new container to the pipeline
If you managed to successfully build the plsda Docker image, it should now be part of kubernetes. You can check this by the command `docker images`. Now try to connect your final downstream analysis node to the pipeline used previously.

> **Tip:** Have a look at the JSON files located in the *pipelines* folder. You will need to create a new one that reads the output from the TextExporter and use the plsda Docker image to perform the last step.

Good luck!
