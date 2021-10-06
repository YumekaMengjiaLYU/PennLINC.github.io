---
layout: default
title: Fetching Your Data
parent: The Way
nav_order: 1
has_toc: true
---

# Fetching Your Data
{: .no_toc }

There are a number of places you may have to fetch data from to get them onto a cluster filesystem. We will briefly cover best practices for the methods we've used before. Additions will be made as we gain more experience.


## Between Clusters Or Local Disks

The best option for moving a large amount of data between clusters is to use the `scp` command. Remember that this process must remain open and running in your terminal, so it might be useful to do this in a fresh terminal window or use `&` at the end of your command. You could also use [`screen`](https://www.geeksforgeeks.org/screen-command-in-linux-with-examples/) to set up a non-terminating terminal.

As mentioned in our general [PMACS documentation](/docs/pmacs), you should scp *into* a node called `transfer`. That would look like this:

```
## my username is ttapera
scp -r path/to/your/data ttapera@transfer.pmacs.upenn.edu:/path/on/pmacs
```

An alternative to `scp` is `rsync`, but that tends to have [more happening under the hood](https://stackoverflow.com/questions/20244585/how-does-scp-differ-from-rsync).

## Flywheel

On Flywheel, your data may already be in BIDS. In this case we recommend using Flywheel's export function `fw export bids`, or the export function provided by [`fw-heudiconv`](https://fw-heudiconv.readthedocs.io/en/latest/). We built the export function into `fw-heudiconv` because we wanted to have more flexibility in what BIDS data we could grab, including data that's not yet supported in the official BIDS spec. Admittedly though, downloading all of `fw-heudiconv` a lot of overhead for just the export function.

```
# with fw export bids
fw export bids <DESTINATION_DIRECTORY> --project <PROJECT_NAME> --subject <SUBJECT_FILTER>

# with fw-heudiconv
fw-heudiconv-export --project <PROJECT_NAME> --subject <SUBJECTS_FILTER> --session <SESSION_FILTER> --folders <LIST_OF_BIDS_FOLDERS>
```

Try `fw-heudiconv-export -h` for more info.

## Globus

[Globus](https://www.globus.org/) is a research data management platform whose best feature is data transfer and sharing. It's surprisingly easy to use and gets the job done with minimal setup. The data sharing concept revolves around setting virtual *endpoints* that data can be shared to and from. Endpoints can be thought of conceptually as mounts, where you can give outbound network access to a certain directory on your machine or cluster, and by sharing the URL of your endpoint, someone can access your directory through the internet or network cluster.

Currently, the best way to use Globus is either through your local disk or on PMACs (recommended). We're still awaiting CUBIC authorization. The general docs for globus are located [here](https://docs.globus.org/how-to/), but for posterity, here are the best instructions:

On a local disk:

1. Log in to Globus with your UPenn organization account -- [https://docs.globus.org/how-to/get-started/](https://docs.globus.org/how-to/get-started/) -- and try out the tutorial for sharing between two test endpoints on Globus' system
2. Download and install [Globus Connect Personal](https://www.globus.org/globus-connect-personal); this service will manage the endpoint on your local machine
3. Download and install the [CLI](https://docs.globus.org/cli/) with pip -- remember to use conda environments! This service will allow you to manage the Globus session when it's running
4. [Login with the CLI](https://docs.globus.org/cli/quickstart/) and transfer your data either through the CLI commands or by visiting the file manager (which you saw in step 1). If someone has shared a Globus endpoint with your account, you'll have access to it in "Endpoints".

On PMACs:

0. Make sure you have access to the PULSE Secure VPN -- [remote.pmacs.upenn.edu](remote.pmacs.upenn.edu)

1. Log in to PMACs' dedicated node for Globus functionality:

```
# first ssh into sciget for network access
ssh -y ttapera@sciget.pmacs.upenn.edu

# then from sciget, log onto the globus node
ssh -y ttapera@sciglobus.pmacs.upenn.edu
```

2. Globus Connect Personal should be available. As above, use it to initialize an endpoint on a directory of your choice on PMACs. Specifically, you should run it as below so that it opens a GUI for logging in with an auto-generated token:

```
# this command will return a URL you can open in any browser and a token you can use to sign in
globusconnect -start &
```

3. Using a new or existing conda environment (see [here](https://pennlinc.github.io/docs/pmacs#logging-in-to-pmacs-lpc) for how to activate conda on PMACs), install the [CLI](https://docs.globus.org/cli/) using `pip` and login with `globus login`.

4. Visit [https://docs.globus.org/how-to/get-started/](https://docs.globus.org/how-to/get-started/) to access the File Manager, as in the Local Disk instructions, to start transferring data.

## Datalad addurls from AWS S3
Now that AWS S3 has become increasingly popular for public data storage, you might want to fetch data from a set of S3 links. `datalad addurls` is a useful command that can retrieve files from web sources and register their location automatically.

For any dataset that comes with a manifest file, the steps are as below:

1. Prepare a well-organized CSV or TSV or JSON file from the given manifest file
The file should contain columns that specifies, for each entry, the URL and the relative path of the destination file to which the URL’s content will be downloaded. 
An example file for HCP-D is `hcpd_s3.csv` that contains the following:

```
submission_id,associated_file,filename
33171,s3://NDAR_Central_4/submission_33171/HCD0008117_V1_MR/unprocessed/mbPCASLhr/HCD0008117_V1_MR_PCASLhr_SpinEchoFieldMap_PA.json,HCD0008117_V1_MR/unprocessed/mbPCASLhr/HCD0008117_V1_MR_PCASLhr_SpinEchoFieldMap_PA.json
33171,s3://NDAR_Central_4/submission_33171/manifests/HCD0008117_V1_MR_UnprocPcasl_manifest.json,manifests/HCD0008117_V1_MR_UnprocPcasl_manifest.json
33171,s3://NDAR_Central_4/submission_33171/HCD0008117_V1_MR/unprocessed/mbPCASLhr/HCD0008117_V1_MR_mbPCASLhr_PA.json,HCD0008117_V1_MR/unprocessed/mbPCASLhr/HCD0008117_V1_MR_mbPCASLhr_PA.json
```
In this case, for each entry `associated_file` column holds the s3 links/URL and `filename` column holds the relative path of the destination file to which the URL’s content will be downloaded. 
2. Obtain relevant AWS S3 credentials
For security purposes, AWS credentials are needed to access the S3 Objects. For datasets held in [NDA](https://nda.nih.gov/), the web service provides temporary credentials in three parts:

   + an access key, 
   + a secret key, 
   + and a session token


All three parts are needed in order to authenticate properly with S3 and retrieve data. AWS credentials for [NDA](https://nda.nih.gov/) can be obtained according to the following steps:
  
  
  2a. Download a copy
  ```
  wget https://ndar.nih.gov/jnlps/download_manager_client/downloadmanager.zip
  ```
  2b. Unzip
  ```
  unzip downloadmanager.zip
  ```
  2c. Use the command-line download manager to generate temporary Amazon S3 security credentials
  ```
  java -jar downloadmanager.jar -g awskeys.txt
  ```
  You will be prompted for your ndar username and password, after which you will be able to find your keys in the file awskeys.txt
  ```
  cat awskeys.txt
  ```
  You will see something like below from your awskeys.txt file.
  ```
  accessKey=ASIAJ3GPA2W73EXAMPLE
secretKey=0i8oIpzWbbVDaybWxmK2vsZsiaSPSdeXEXAMPLE
sessionToken=AQoDYXdzEPX//////////wEaoAKf5O7+2FhbYIqed/oh69l6FuVuaxpanNbA2yCR/1iYB4cjqQ415FUhDVIN4E4fXF9j8FzV4cTE6vY0dLzOWcUq7dNLvFzJux3oh0bu4bqbZ9EwBAxKb4bNf1pSbUWjQ+Sgrnjz38Uf63jSpxWAUM66mFVOPJhyaHh5lnUREZMNJrwzrkoUn6SR4fTEjXBuQRh9n4idllP+GW7i5XncDqZz+LutYgYMSGjb3x2j1hO1jCyRQ0dtFltFtaq77onMrCnk8k5YCmWyEFgfECtmu0fFE5hpy2NDLg2cFz1aVGN0K2B9vkOPEhG1LIm5+TY8U3MhWQsBnGvGCe0dO/4EOSJfJDhZZe+LsUhVhLJJWnQPRUcqpfNRWU8VnTHxadPLEXAMPLE=
expirationDate=Tue Nov 18 03:14:16 EST 2014

  ```


3. Export the AWS S3 credentials
Navigate to `~/.aws/credentials` and paste in the information as below:
```
[NDAR]
aws_access_key_id = ASIAJ3GPA2W73EXAMPLE
aws_secret_access_key = 0i8oIpzWbbVDaybWxmK2vsZsiaSPSdeXEXAMPLE
aws_session_token = AQoDYXdzEPX//////////wEaoAKf5O7+2FhbYIqed/oh69l6FuVuaxpanNbA2yCR/1iYB4cjqQ415FUhDVIN4E4fXF9j8FzV4cTE6vY0dLzOWcUq7dNLvFzJux3oh0bu4bqbZ9EwBAxKb4bNf1pSbUWjQ+Sgrnjz38Uf63jSpxWAUM66mFVOPJhyaHh5lnUREZMNJrwzrkoUn6SR4fTEjXBuQRh9n4idllP+GW7i5XncDqZz+LutYgYMSGjb3x2j1hO1jCyRQ0dtFltFtaq77onMrCnk8k5YCmWyEFgfECtmu0fFE5hpy2NDLg2cFz1aVGN0K2B9vkOPEhG1LIm5+TY8U3MhWQsBnGvGCe0dO/4EOSJfJDhZZe+LsUhVhLJJWnQPRUcqpfNRWU8VnTHxadPLEXAMPLE=

```


4. Check whether you have the datalad special remote enabled
```
# this command will return a hyphen-separated string if a remote is enabled
git annex info | grep datalad
```
If not enabled (that is, no string returned), run the `git annex initremote` command. 
Don't forget to give the remote a name (it is `datalad` here). The following command will configure datalad as a special remote for the annexed contents in the dataset.
```
# this command will return a hyphen-separated string if a remote is enabled
git annex initremote datalad type=external externaltype=datalad encryption=none

```
5. Create your datalad dataset in the desired directory
```
# mydataset is the name you want to give to the datalad dataset
datalad create mydataset
cd mydataset
```
Take HCP-D as an example, you will run:
```
datalad create HCP-D
cd HCP-D
```

6. Run datalad addurls command
It would be helpful to first go over the [documentation](http://docs.datalad.org/en/stable/generated/man/datalad-addurls.html) to have the command tailored to your needs. 
Take HCP-D as an example, you will run:
```
datalad addurls hcpd_3.csv '{associated_file}' '{filename}'
```
where `{associated_file}` refers to the `associated_file` column in `hcpd_s3.csv` and `{filename}` refers to the `filename` column in `hcpd_s3.csv`. 
