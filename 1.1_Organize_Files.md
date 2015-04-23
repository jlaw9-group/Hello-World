Protocol 1.1 Organize Sequencing Output Files
=============================================
Jeff Law – jlaw@childhooddiseases.org - 11/22/2014

##Table of Contents
  * [Protocol 1.1 Organize Sequencing Output Files](#protocol-11-organize-sequencing-output-files)
      * [Goal](#goal)
      * [Script Overview](#script-overview)
      * [Push the Files](#push-the-files)
  * [Appendix I : Password-less SSH-KEYs](#appendix-i--password-less-ssh-keys)

###Goal
Organize and de-convolute the important files output from a sequencing run by copying them to another server dedicated to analysis. Eventually the goal is to make a plugin to be part of the Torrent Suite (TS) software for running these scripts.

###Script Overview
The script [_start\_Push.py_](https://github.com/jlaw9/Scripts/blob/master/Push/start_Push.py) takes as input a CSV (Comma Separated Value file) which has all of the run or sample information necessary to push or copy sequencing files of a sample from a Torrent Suite sequencing PGM or Proton to a specific project/sample/run directory on another server. 
> **Important note:** This is simply a push script meaning only files where the script is 
> located and run can be copied to another location.   

**Script location** : On Bonnie's PGMs and on pluto and mercury, the push scripts are found in the directory _~/jeff/Push\_Data_.

Here is an example excel document from the Wales project used to push the data from the Yale PGM to TRI's triton:

| **ID** | **Plate Name** | **Plate Cell** | **Case/Control** | **Run ID** | **Barcode No** | **Status** |
| --- | --- | --- | --- | --- | --- | --- |
| 50890-3 | case1 | A01 | 1 | 291 | IonXpress\_001 | Pass |
| 50930-3 | case1 | A02 | 1 | 327 | IonXpress\_002 | Pass |
| 50201-3 | case1 | A03 | 1 | 18 | IonXpress\_003 | Pass |

Here is that same file after exporting it as a CSV from excel:

ID ,Plate Name,Plate Cell ,Case/Control ,ROT ID ,Barcode No,Status

50890-3,Case1,A01,1,291,IonXpress\_001,Pass

50930-3,Case1,A02,1,327,IonXpress\_002,Pass

50201-3,Case1,A03,1,18,IonXpress\_003,Pass

This is the format of the input to _start\_Push.py_. If no barcodes are provided, then the script will adapt accordingly.

>**Important Note:**  If using a mac, excel will add a funny character to the end of the line on a CSV. This will cause linux to read the file incorrectly. To prep the CSV for linux, open the CSV in [_vim_](file:///tmp/d20150412-3-m06vk1/Bioinformatics_Glossary.docx#Vim) and type the following two commands:

	:set fileformat=unix
	:%s/\r/\r/g

**Server Information:** For the IP addresses of the servers, see [Appendix I](file:///tmp/d20150412-3-m06vk1/1_0_Protocols_Overview.docx#AppendixI) of the protocol 1.0 _Protocols Overview_. If you are pushing a large number of runs from one server to another, you do not want to have to type in the password for every file that is copied. See Appendix I for instructions on how to store an ssh-key from one server to another to bypass having to enter a password.

###Push the Files:
**Step 1:** Run _start\_Push.py_ giving the new server IP address, the destination path where the files should be placed on the new server, the CSV, and the log file. _start\_Push.py_ will run all of the next steps 

	USAGE: bash start_Push.sh <ionadmin@ipaddress> </destination/path> <projectFile.csv> <push_results.csv>

	Example: bash start_Push.sh   ionadmin@192.168.200.131   	/mnt/Despina/projects/Einstein   Einstein_Runs.csv   Einstein_Runs_Push_Results.csv

**Step 2:** Parse the input file line by line and will store the info of each column separated by a comma. _start\_Push.sh _will then call push_Data.sh for each line of the CSV to actually transfer the files.

	bash push_Data.sh
		--user_server ionadmin@ipaddress
		--project_path /dest/path/on/new/server
		--sample E0001
		--run_id 468 (the run ID from the CSV)
		--run Run1
		--proton_name PLU (either PLU or MER)
		--backup_path $BACKUP_PATH
		>> push_results.csv </dev/null &


**Step 3:** Generate a JSON file containing this run's info using _writeJson.py_. This JSON file will be used throughout the pipeline to store important run metrics and metadata. See the protocol [_Automated\_Scripts.docx_](file:///tmp/d20150412-3-m06vk1/Automated_Scripts.docx)file for an explanation of how the JSON file is used. Each of the metrics passed into _writeJson.py_ either from the CSV used as input or from the settings used to call _start\_push.py_ will be added to the json file.

	python writeJson.py
		--bam E0001.bam
		--run_name Run1
		--sample E0001
		--proj_path /dest/path/on/new/server
		--orig_path /path/where/bam/file/was/found
		--proton PLU or MER
		--ip_address 192.168.200.42 (IP address of Triton)
		--ts_version 4.2 (TS version used to generate the BAM file. This information is found in the version.txt file)
		--json E0001_Run1.json (JSON file to be made)


An important note here is that if you are pushing this data to a new server, the settings found in [_writeJson.py_](https://github.com/jlaw9/Scripts/blob/master/Push/writeJson.py) from line 38-45 will need to be edited to represent the correct new paths to each file found on the new server.

**Step 4:** Find the following files and prepare them to be copied. If the rawlib.bam is found in the non-archived directory, the following files will be copied from there. Otherwise, they will be copied from the archived location on the server (for pluto: /mnt/Charon/archivedReports for mercury: /mnt/Triton/archivedReports).

- rawlib.bam
- rawlib.bam.bai
- E0001\_Run1.json
- E0001.amplicon.cov.xls (This will only be copied if Coverage Analysis was run on the browser)
- TSVC\_variants.vcf (This will only be copied if TVC was run on the browser)
- report.pdf (Contains the run metrics such as % polyclonality. Will be parsed later in the process.)

**Step 5:** Copy (push) the files to the new server using [rsync](file:///tmp/d20150412-3-m06vk1/Bioinformatics_Glossary.docx#rsync). Files are copied one at a time to the new server. If the file already exists on the new server, the file will not be re-copied. If the copy is successful, the script prints a successful message to the log file. If the file transfer is unsuccessful either because of a lost connection or some other reason, the script will try again in 30 seconds up to 10 times. After that, if the copy is still unsuccessful, the script will print an error message to the log file and continue copying the other files listed in the CSV.

	rsync -avz --progress E0001.bam $Triton:/path/to/E0001.bam

**Step 6:** Check the log _push\_results.csv _file to ensure that all of the files were copied correctly. If any files are not copied properly, they might have to be manually copied using [_rsync_](file:///tmp/d20150412-3-m06vk1/Bioinformatics_Glossary.docx#rsync) or [_scp_](file:///tmp/d20150412-3-m06vk1/Bioinformatics_Glossary.docx#scp).



##Appendix I : Password-less SSH-KEYs.

SSH stands for Secure Shell. In order to connect from one server to another server over a network, authentication is required. Normally this authentication would be a password, but a one time ssh-key "password" can be generated on your machine and stored on the remote server to give you "password-less" authenticated access from your machine to the server. To generate and store this ssh-key, follow the steps below:

**Step 1:** Generate the ssh-key on your machine (On a mac server or laptop. I'm not sure how to set up ssh-key's on windows machines.):

    ssh-keygen -t dsa

This will ask you for a passphrase that you later given to ssh-add.  It will create in your _~/.ssh_ directory:

- id_dsa - this contains your private key, to be kept secret
- id_dsa.pub - this contains your public key that you give to remote machines

You should only do the above step once, i.e. you use the same public key for all remote machines.

**Step 2:** Copy your private key from your workstation into the _~/.ssh_ folder of the remote machine:


	scp   ~/.ssh/id_dsa.pub   
	username@ip_address:~/.ssh/id_dsa.pub.W

Note the .W at the end of the scp command. If you do not append the .W to the end of the file name, you could overwrite that server's private ssh-key.

**Step 3:** Login to the remote server and append your copied public ssh-key to that server's authorized\_keys file:

	ssh username@ip_address
	cat   ~/.ssh/id_dsa.pub.W   >>   ~/.ssh/authorized\_keys
	rm   ~/.ssh/id\_dsa.pub.W

