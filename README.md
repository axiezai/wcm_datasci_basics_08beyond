Datalad, Containers, and Beyond
---

 - Lecture slides:  `datalad_and_beyond.ipynb`, need jupyter's `rise` extension installed to render as slides in browser.

## A datalad dataset
The best way to get started with datalad is to work through their excellent [handbook](http://handbook.datalad.org/en/latest/).

Their full documentation is [here](http://docs.datalad.org/en/stable/index.html).

Here we will go through a brief example, and we begin by creating a new datalad dataset:

```bash
$ datalad create -c text2git example_dataset
```

This is similar to `git init`, where we created a new directory with version control back-end. But in this case, it's `datalad` on top of `git`. We can confirm this by checking the status of the dataset we just created:

```bash
$ cd example_dataset
$ datalad status
```

## Saving a dataset:
Saving a dataset is the equivalent of a `commit` in `git`. Let's say we begin to collect some data and they were saved in our `example_dataset` directory:

```bash
$ scp -r ~/sample-data .
$ datalad status
untracked: sample-data (directory)
```

Here we see that `datalad` recognized an untracked new directory, we can save this new directory as part of this dataset with `data save -m "added new sample data"`. (Just like `git commit -m`)

## Nested dataset:
Imagine we wrote some code in this dataset, but we want the code to be version controlled separately from the data itself. We can make use of datalad's subdataset capabilities:

```bash
$ datalad create -c text2git -d . analysis
[INFO   ] Creating a new annex repo at /demo8/example_dataset/analysis
[INFO   ] Scanning for unlocked files (this may take some time)
```

Now, datalad will recognize another untracked directory named `analysis` in your dataset. However, if you `cd analysis` and then repeat `datalad status`, you will find that it's a subdataset nested within your main dataset. 

We can make modifications to this subdataset, let's say we created an analysis with a python environment:

```bash
$ ls
environment.yml
plot_data.ipynb

$ datalad status
untracked: environment.yml (file)
untracked: plot_data.ipynb (file)
```

When saving these new files inside of your subdataset, the history will be separated from the main dataset:

```bash
$ datalad save -m "visualize analysis results"     add(ok): /demo8/example_dataset/analysis/environment.yml (file)
add(ok): /demo8/example_dataset/analysis/plot_data.ipynb (file)
save(ok): /demo8/example_dataset/analysis (dataset)
action summary:
  add (ok: 2)
  save (ok: 1)

$ git log --oneline
983c02b (HEAD -> master) visualize analysis results

# in our main dataset:
$ cd ..
$ datalad status
untracked: analysis (directory)

$ git log --oneline
d128354 (HEAD -> master) added sample MEG data
```

We see that the main dataset has detected a new directory, but the `save` we performed inside the subdataset did not affect our main dataset. We can tell the main dataset what we did for this data so far:

```bash
# in main dataset
$ datalad save -m "created analysis subdataset"
```

Now what if we want to share this dataset? Or clone this dataset for ourselves on a high performance computing resource?

## Sharing a large dataset through GitHub with datalad and OSF
Github provides an excellent interface for sharing code, but large data files is a different story. Github recommends repositories to remain small for clone and maintanence speed. Individual files in a repository cannot exceed the `100MB` size limit, and the entire repository is recommendeed to be smaller than `1GB`. 

You can see the full guidelines [here](https://docs.github.com/en/github/managing-large-files/what-is-my-disk-quota).

But we want to share some data with collaborators and with ourselves for when we need to work remotely. Additionally, we want to use GitHub's interface to visualize our commits and modifications. This might seem impossible with large datasets, but with the help of datalad and OSF, this is easily achievable!

There are many ways to host your data online, through DorpBox, Google Drive, Amazon S3 Buckets. And datalad can interact with all these services with `git annex` special remotes. But for this example we will utilize [osf.io](https://osf.io/). Open Science Framework (OSF) is a collaboration tool created by the Center for Open Science, providing reliable repositories for projects and data. 

Instead of `remotes`, datalad decentralizes data management with `siblings`:

```bash
$ datalad siblings
.: here(+) [git]
```

So far, our dataset resides only in one place (here). But we can create several remote siblings. If we had a secure shell (ssh) remote destination we like to work with, we can create a remote sibling with: 

```bash
$ datalad create-sibling -s pothos ssh://user@host://data/sample_data
```

The `-s` flag provides a name for the remote.

We will first create an `osf` sibling, and push our dataset so that it's hosted on OSF (This assuming you've created an OSF account and did the authentication steps as illustrated [here](http://docs.datalad.org/projects/osf/en/latest/tutorial/authentication.html), you will have to do the same authentications for other special remotes as well!):

```bash
# within a dataset mydataset
$ datalad create-sibling-osf --title <my-new-osf-project-title> -s <my-sibling-name> \
 --category data \
 --public
 create-sibling-osf(ok): https://osf.io/ujpe5/
[INFO   ] Configure additional publication dependency on "osf-storage"
configure-sibling(ok): /demo8/example_dataset (sibling)

$ datalad siblings
.: here(+) [git]
.: osf-storage(+) [osf]
.: osf(-) [osf://ujpe5 (git)]
```
After this sibling is created, all that's left to do to share if `datalad push` to publish the dataset to OSF:

```bash
$ datalad push --to osf
```
For this example, we'd like to use GitHub to display and share our dataset. Datalad provides specific sibling types for commonly used hosts (you will have to authenticate again for GitHub!):

```bash
$ datalad create-sibling-github sample_data -r --github-login {your github account email} --github-organization {provide optional organization name} --access-protocol ssh --publish-depends osf-storage
```

Note the `--publish-depends` input indicates that our GitHub sibling will depend on `osf-storage`. This means every push will be pushed to `osf` first, and then GitHub will be updated accordingly. Additionally, all files are stored on `osf-storage`, GitHub will simply handle URLs and links. After creating this GitHub sibling, we will see new repositories pop-up on GitHub under our account. We will have to copy the URLs of these new remotes and replace them in the `url` field of `.gitmodules` in our main dataset. You will also recognize that our subdataset is considered a submodule by `git`. Once this is finished, we can simply push to GitHub, and then clone the repository as you would with a GitHub project. 

```bash
datalad push --to github
```

## Creating sharable containers easily with [neurodocker](https://github.com/ReproNim/neurodocker)

Often times, it is sufficient to provide an environment or requirements file with your project. However, there might be times when your analysis requires more than python packages. You may need to share a compiled software that cannot be installed by `conda` or `pip`. Or you are working on a high performance computing resource and you must build a container because the shared resource doesn't have anything installed. [Docker](https://docs.docker.com/engine/reference/builder/) and [Singularity](https://sylabs.io/guides/3.7/user-guide/definition_files.html) are powerful tools to help scientists containerize their softwares, but how do we create these containers with ease? 

[Neurodocker](https://github.com/ReproNim/neurodocker) is a command-line program that generates custom Dockerfiles and Singularity recipes for neuroimaging. It is easily installed with `pip`, and the usage follows a simple template:

```bash
neurodocker generate singularity|docker [options]
```

You can see the options and what neuroimaging softwares are supported by neurodocker on their GitHub repo. While the software coverage is limited, the tool is extremely useful for you to begin customizing Dockerfiles and Singularity recipes from the template that it creates. For example, You can tell `neurodocker` to generate a nearly empty recipe for you to fill in according to your needs, OR replace the existing links for neuroimaging packaged zip files with another link that fits your needs.

To further simplify your command-line usage, you can provide all your installation needs in the form of a `JSON` file, which is a dictionary data format parsed by Python. For example the following `JSON` file can be used with `neurodocker` as: 

```bash
$cat example_specs.json

{
  "pkg_manager": "apt",
  "instructions": [
    ["base", "debian"],
    [
      "ants",
      {
        "version": "2.2.0"
      }
    ],
    ["install", ["git"]],
    [
      "miniconda",
      {
        "create_env": "neuro",
        "conda_install": ["numpy", "traits"],
        "pip_install": ["nipype"]
      }
    ]
  ]
}

$ neurodocker generate docker example_spec.json
$ neurodocker generate singularity example_spec.json
```

A full example gallery can be found [here](https://github.com/ReproNim/neurodocker/tree/master/examples).

