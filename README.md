# Metabolomics data analysis using Pachyderm
In this page we introduce an metabolomics preprocessing workflow that you can run using [Pachyderm](https://github.com/pachyderm/pachyderm), a distributed data-processing tool built on software containers that enables scalable and reproducible pipelines.

## Requirements
You need to install Vagrant and Virtualbox.

In Windows:

- Install Chocolate: https://chocolatey.org/install

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
Using PowerShell or Terminal, navigate to the folder containing the box file and run:
```bash
> vagrant up
```
SSH into the machine 
```bash
> vagrant ssh
```
Check jupyter notebook and Rstudio:
Open a web browser and navigate to [localhost:8888](localhost:8888) for jupyter notebook and [localhost:8787](localhost:8787) for Rstudio.

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
### Pushing data into Pachyderm's Data Repository
A repo is the highest level data primitive in Pachyderm. They should be dedicated to a single source of data such as the input from a particular tool. Examples include training data for an ML model or genome annotation data.
Here we will create a single repo which will serve as input for the first step of the workflow:
```bash
> pachctl create-repo mrpo
```
You can push data into this repository using the put-file command. This will create a new commit, add data, and finish the commit. Explore further on how commits work.
```bash
> cd ./MSData
```
```bash
> pachctl put-file <name of the repo> <name of the branch> -c -r -p <number of files to upload in parallel> -f .
```
### Running a Pachyderm pipeline
Once your data is in the repository, you are ready to start a bunch of pipelines cranking through data in a distributed fashion. Pipelines are the core processing primitive in Pachyderm and they’re specified with a JSON encoding. Explore the pipelines folder and find out which of the pipelines is the first step of the pre-processing workflow. You can find it by discovering which pipeline has the previously created repository as an input. Then run it using:
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
Three more pipelines compose the pre-processing workflow. After you run the entire workflow, the resulting CSV file generated by the TextExporter in OpenMS will be saved in the TextExporter repository. You can download the file simply by using:
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
