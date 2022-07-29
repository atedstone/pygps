# DGPS Processing with Python

Andrew Tedstone 2012-14 and 2022, based on previous documentation by Andrew Sole and Ian Bartholomew, plus workflow approach by Matt King (Newcastle).

## Summary of workflow

1. Ensure everything is installed.
2. Download orbit (sp3) files for year.
3. Copy base and rover leica files to working directory.
4. Obtain third party data to cover known gaps in our base record.
5. `process_rinex.py`: Convert leica files to windowed rinex files (base and rover)
6. `process_dgps.py`: Do track kinematic processing.
    6a. Process 'temporary' fixes taken during redrilling/flights.
7. `conc_daily_geod.py`: Concatenate daily track GEOD files to year.
8. `calculate_local_origin.py`: Only if this is a new site, to calculate local origin position.
9. `gnss_disp_vel.py`: Transform coordinates, filter data, convert to along/across-track displacements, calculate velocities.

Example using site **lev5**, 2021, days 129 to 242:

```bash
# Prepare
cd /scratch
mkdir gps_2021
cd gps_2021
cp -r /location/gps_config .
gps.py get_orbits 2021 129 242

# Process site RINEX (also do this for the base)
cp /location/Default...m00 .
process_rinex.py lev5 Default...m00 s -start 2021-05-09 -finish 2021-08-30

# Run TRACK
# First get a-priori coordinates, then...
process_dgps.py rusb lev5 129 242 -ap x y z
# ...following instructions.

# Post-process
conc_daily_geod.py rusb lev5 2021 129 242
# Run next line only if no origin.csv file:
calculate_local_origin.py lev5 lev5_rusb_2021_129_242_GEOD.parquet
gnss_disp_vel.py lev5 lev5_rusb_2021_129_242_GEOD.parquet
```


## Strategy for multi-year field campaigns

This workflow treats each batch of data separately. There is no need to recompute velocities for the entire time series each time more data are acquired and processed.

Batches of data cannot span multiple years. So, if collecting data only once a year e.g. in springtime, the processing needs to be split into two batches separated by the change in year.


## Summary of file types

* Orbit files: `igs<doy>.sp3`
* Rinex files:
	- If compressed: `*.<yy>d`
	- If uncompressed: `*.<yy>o`
	- Daily files: 	 `<site>_<doy>0_d.<yy>o`
	- Overlapped/windowed files: `<site>_<doy>0_ol.<yy>o`
	- (File with multiple days: `all_<site>.<yy>o`)
* TRACK command files: `track_<base>.cmd`
* Files generated by TRACK:
	- Log file: `<rover>_<base>_<doy>.out`
	- GEOD results file: `<rover>_<base>_<year>_<doy>_GEOD.dat` or `<rover>_<base>_<doy>GEOD.dat` (old)
	- NEU results file: `<rover>_<base>_<year>_<doy>_GEOD.dat` or `<rover>_<base>_<doy>NEU.dat` (old)
    - Figures: `<rover>_<base>_<year>_<doy>_NEU.png` or `trackpy.NEU.<rover>.LC.<doy>.eps` (old)
	- Processing session log: `gps.track.<rover>.log`
* Post-processed files: `<rover>_<year>_geod.dat` or `<rover>_<start year>_<end year>_geod.dat`
* Post-processing ancillary files:
	- Rotation file: `rotation_<site>.dat`
	- Origin file: `origin_<site>.csv`
	- Exclusions file: `exclusions_<site>.csv`
	- Corrections file: `corrections_<site>.csv` (not currently implemented 2022-07).
* Post-processing outputs:
	- Various PNG plots
	- HDF5 file containing 24-h and 6-h velocities, xyz displacement.
	- CSV file containing 24-h velocities.


## Installation

More about TEQC: http://facility.unavco.org/software/teqc/

More about GLOBK/Gamit: http://chandler.mit.edu/~simon/gtgk/script_help.htm

More about TRACK: http://geoweb.mit.edu/~tah/track_example/

Access password for GLOBK/Gamit/TRACK downloads: check email / ask maintainers for access.

You should check that the Gamit tables are sufficiently up to date for your captured GPS epochs. If they are not, update the GLOBK/Gamit installation following their instructions.

Create a symlink to gamit, e.g. as follows:
	
	ln -s /usr/geos/gamit_globk/10.4 ~/gg 

If you haven't already got it, clone the pygps repo and put on your python path:

 - Make sure you can see hidden files and folders.
 - Open `~/<USER>/.bashrc` for editing.
 - Add the following (e.g.):

```bash 
PYTHONPATH="${PYTHONPATH}:~/<USER>/pygps/"
export PYTHONPATH
```

 - Save and close.

This means that it won't matter where you then do your main working - Python will automatically find the scripts. 

Ensure that `gps.py`, `process_rinex.py` and `process_dgps.py` are executable - e.g.:

```bash
chmod u+x <file_name>
```

Set up a working directory in your scratch space, e.g. `/scratch/<USER>/gps_workspace_<YEAR>/`. 

Clean up as you go, otherwise you'll end up with loads of files, e.g. once the leica files have been converted to rinex, delete the leica files, and so on. The scripts do not clean up.
 
In this working directory, clone the GPS configuration files, ensuring that the files land up in a subdirectory called `gps_config`, i.e. (via SSH protocol):

```bash
git clone ssh://git@bitbucket.org/atedstone/lev_gps_config.git gps_config
```

(Alternatively, if setting up processing for a new field campaign in a different location, you may wish to clone this project elsewhere and use the files as templates for your own parameter files to create your own gps_config folder, which you can then version-control if you wish.)
	
pygps will look for TRACK cmd files in this location. If you modify these files, particularly the rover-specific ones, be sure to commit your changes back to the repository once you are done.

Files from Kellyville were sometimes in the compressed rinex format.
If they are, you will also need RNXCMP, available at http://terras.gsi.go.jp/ja/crx2rnx.html.
You need to install this to your home directory.
At the time of writing, fleet requires the Linux 32 bit version.
 1. Download the tar.gz file.
 2. uncompress the download: `tar -zxvf <filename>`
 3. Put the files on your unix path. Either:
     a. Copy the files from `<RNXCMPDirectory>/bin/` directly into `/home/<USER>/bin/`.
	 b. Add the `<RNXCMPdirectory>/bin/` to your unix path.

	 
## Note on gps.py functionality
This file contains two classes used for Python-based GPS processing:

```python
gps.RinexConvert
gps.Kinematic
```

To get full information on the module, at the command line run:

```bash
pydoc gps   #(this displays documentation on the command line)
```

 -or-

```bash
pydoc -w gps   #(this saves documentation to gps.html, to be viewed in a web browser)
```

RinexConvert has been developed with Leica 530 and 1200 receivers in mind. Some functionality works with other receivers, e.g. window_overlap works with Trimble dat files (but not .T00/.T01).

Some critical functions have been enabled to take advantage of python's command line functionality, i.e.

```bash
python -m gps <function name> <arguments>
```

To find out what's available in this way, just do:

```bash	
python -m gps
```

See also http://docs.python.org/using/cmdline.html for more information.


## Preparations

Make sure you are putting all these following files into the scratch/working space you set up above.

Use a terminal window (e.g. PuTTY) to do the following....

### Get the orbit files

```bash
gps.py get_orbits <year> <start doy> <end doy>
```
	 
N.b. don't attempt this over a change in year, e.g. start of 360 and end of 4. Instead do two calls.
The only problem will then be that days 365 and 1 don't contain the next and previous days data respectively.
Just copy and paste from the relevant files into the next, or on unix command line use cat, e.g:

```bash
cat igs364.sp3 igs365.sp3 igs001.sp3 > igs365.sp3
```

Also remember that the .sp3 naming scheme only contains the day of year as this is what Track understands - so if you're not careful when downloading from > 1 year you'll end up overwriting files. Best to only deal with one year's worth of data in scratch space at a time.
   

### Optional: Get 3rd party base RINEX files.

If required, download rinex files from another site to cover the gaps.

```bash
sh_get_rinex -archive sopac -yr 2011 -doy 0 -ndays 250 -sites kely
```

If files have `*.<yy>o` suffix you're all set, otherwise, if they are zipped, unzip the compressed rinex files using 7zip or whatever. Then convert to normal rinex i.e. from `*.10d` to `*.10o`:

```bash	
gps.py crx2rnx <suffix, e.g. 10d>
```	    
 
Overlap/window the kellyville rinex files: see 'Convert leica files to Rinex', but choose appropriate option to deal with rinex files at command line.

### Update `obs_file`

Check the `obs_file` file extensions in the track cmd files (in gps_config) - they need to be set to the correct year suffix, otherwise you'll get 'olding' errors from track.
		
	
### Convert leica files to RINEX

Copy the raw leica files for the rover and base into the scratch space.

We convert to RINEX files which are windowed to 28hrs duration: from 22:00 the previous day to 02:00 the next day.

In your terminal window, make sure you're in your scratch gps directory.

There's two possibilities, depending on whether the data were saved to daily files or one big lump. Assuming single:

```bash
process_rinex.py 
```
	
Follow on-screen instructions. Be aware that if you're processing from one big leica file you'll be asked to specify the start and end dates. The processing will then commence and will tell you what it is doing. You can wander off, it shouldn't need your attention.

Make sure you create RINEX files for each site (i.e. run the script for each site).


## Do the kinematic processing

Pre-requisites:
	- Firstly, make sure you have appropriate `.cmd` files for your base station site. These are already available for the leverett transect (`track_levb.cmd` and `track_kely.cmd`, in `lev_gps_config`). My initial recommendation for shorter baselines with 10-15sec sampling would be to use `track_levb.cmd` as a template. For longer baselines with sparser sampling, use `track_kely.cmd` as a template.
	- A number of additional arguments are hard-set in the cmd files created for each base station, (e.g. levb, kely), e.g. site_stats and bf_set. These probably don't need to be modified from their default values...
	- Secondly, open `pygps/process_dgps.py` and ensure that the default parameters for processing with your base station are set up - follow the format in the file. Again, set initial values based on my recommendation above, unless you've more specific information to go on... 


### A-priori coordinates

For first day for each site use a priori coordinates of each site derived from online 
 http://apps.gdgps.net/apps_file_upload.php (lev_apr_coords.txt)
 Also, webapp.geod.nrcan.gc.ca/geod/tools-outils/ppp.php
Upload a rinex file and specify the day from which to return a priori coordinates.
Subsequently select no. The program takes the APR coordinates from the previous days results.

In 2012 I had problems with coordinates from the above website. However, teqc estimates the daily positions and puts them in the top of daily rinex files - these seem to do the trick, so I used these instead. You might also find that the site_stats (see later) initially have to be loosened to deal with the APR coordinates. (AJT, September 2012).


### Processing

Open a terminal window at the scratch location. 
**If using PuTTY make sure you also have Xming working - for figure windows.**

Run:

```bash
process_dgps.py <base> <rover> <start DOY> <end DOY> -ap <X> <Y> <Z>
```
	
You don't have to process an entire site in one session. Enter the start day and guesstimate an appropriate end day. If you get fed up before the end day is reached, do `CTRL+C` to break out/halt the process (preferably between processing two days of data, rather than during).

If providing a-priori coordinates (-ap): TRACK will not transfer negative values to the cmd file. Instead, put them positive here and add a negative sign (-) to the relevant field of the cmd file. Once the day using the a-priori coordinates has finished, remove the -ve sign. (Tech note: this is because convertc which is used to convert lat/lon to ECEF for the next day's a-priori coordinates always outputs positive values - hence need to remove -ve sign.)

`process_dgps.py` has defaults for `ion_stats`, `MW_WL` and `LG` set. These vary depending on the specified base station: at the time of writing, levb and kely are both supported.

By default, track is set up to accept the day's results automatically, using an RMS approach. (A Spearman correlation approach is also available). The thresholds are set in the arguments to gps.Kinematic.track().

Track can also be set up to require the user to accept every day manually - provide use_auto_qa=False as an argument to the call to gps.Kinematic.track in process_dgps. 

Initially, track will try to use the default ion_stats, MW_WL and LG parameters as specified in process_dgps.py.

If you choose to reject track's initial results based on the default parameters, you can enter your own:

- `ion_stats`. This parameter sets ion_stats in the track cmd file. It seems to do something - not sure what - even though the track documentation suggests that many more arguments for ion_stats are required. 	Does it just set `<jump>`? (String number: `<S06>`)
- MW_WL weighting. Weight to be given to deviation of MW-WL from zero. Default is 1 (ie., equal weight with LC residuals). Setting the value smaller will downweight the contribution of the MW-WL. For noisy or systematic range data (can be tested with a P1,P2 or PC solution), the WL_fact may be reduced.
	+ MW WL stands for Melbourne-Wubbena (MW) wide lane (this is the combination of range and phase that gives the difference between the L1 and L2 biases and it independent of the ionospheric delay and the geometry - track help file line 266). The parameter is taken by the float_type command, setting `@FLOAT_TYPE <WL_Fact>` (string number: `<S04>`).
- LG combination weighting. Weight to be given to deviation of the Ionospheric delay from zero.  Default is 1 (i.e., ionospheric delay is assumed to be zero and given unit weighting in deterimining how well a set of integer ambiquities fit the data.  On long baselines, value should be reduced.
	+ For long baselines (>20 km) the `<Ion_fact>` should be reduced to give less weight to the ionospheric delay constraint.  For 100 km baselines, 0.1 seems to work well.  With very good range data (ie., WL ambiquities all near integer values), this factor can be reduced. Sets `<Ion_fact>` argument for @FLOAT_TYPE (string number `<S05>`).
		
The results should eventually pop up in a figure window, and the RMS values should appear in the terminal window. Particularly if using kely, processing times of >15 mins per day are not unusual.

The figure window will block input to the terminal window until it is closed. Unfortunately there's no way around this with the version of python currently on burn.

You'll be asked to input some information for logging - a quality identifier, and an optional longer comment. These are appended the the log file.

Quality identifiers:

* G: Good
* O: Ok
* B: Bad
* A: Accepted Automatically (by Spearman test)

If RMS high and results not good, try (in the following order):

1. Increase ion_stats, up to maybe 3 or 4. Choose the combination that gives the lowest rms and produces the closest to a straight line when plotted.
2. Reduce MW_WL weighting from default=1 to say 0.5 if data appear noisy.
3. Possibly try reducing LG weighting - especially for sites further away where the ionosphere has an increasingly large effect.

We used to suggest that `site_stats` could be increased if results looked to be of poor quality. However, now we think we understand this more... the defaults set in the cmd file reflect our predictions that sites are unlikely to be > 10 metres away from a priori position (ish), and should not have moved more than 2 metres compared to the last epoch. So, increasing these values shouldn't really help... (but it does sometimes!)

Possible errors:

- `SP3 Interpolation Errors`. This may mean that there are problems with one or more satellites in the SP3 orbit files. Open the .out file for the day which just failed and go to the bottom of it. Check the line(s) which say "Interpolation error PRN" - the number directly after this is the satellite. Open the cmd file for the relevant base station and add: `exclude_svs <satellite_number>`
- (e.g.) `Track IOSTAT error:  IOSTAT error      2 occurred opening gps_config/track_kely.cmd in MAKEXK` - this probably means you're trying to process while not in your working directory.
- `!!APR .LC file does not exist! Terminating processing.` If you are expecting a .LC file to exist this may again mean that you are trying to process while not in your working directory.
- `STOP FATAL Error: Stop from report_stat.` Have you remembered to update the RINEX file suffix in the track CMD files to reflect the year you are processing?
- Something about fortran files: you probably haven't loaded gamit module - see start of this doc.

Old ideas on trying to improve results - sometimes work but we're even less certain why!

- Increase site_stats in the .cmd (track_kely.cmd/track_levb.cmd) file (df=10 10 10 1 1 1, try 100 100 100 10 10 10 first, then gradually increase)
- Increase bf_set in the .cmd file (bf= 2 40, try 5 100 and then increase gradually)
- Increase or decrease ion_stats in the input parameters (df=1, try 0.3 first then 2,3 if that dosnt work)
	
	
If a day or more accidentally or otherwise get missed during processing and you need to go back to re-process them:

- Find the NEU and GEOD files for the day prior that on which you need to re-start processing.
- Make sure they are in your workspace directory.
- Rename to `track.NEU.<site>.LC` and `track.GEOD.<site>.LC`
- This ensures that track uses the previous day's coordinates for a priori estimate positioning.
- If you haven't finished processing the rest of the season, be sure to change filenames back to the appropriate LC files.


### Dealing with temporary positions

E.g. those taken when GPS has powered down so we need to get seasonal/annual displacement from one short survey.

There are likely to be multiple temporary positions in one raw leica file, corresponding to a number of rover locations.

It's probably easiest to do this in a completely separate gps processing folder, also with the gps_config folder pulled down from the SVN repository, in order to avoid filename clashes. Also copy over the relevant base station and orbit files.

First convert the leica file to rinex file(s) using process_rinex. Then window this rinex file into separate rinex files for each rover. Flight timings are very helpful here. You can first check the contents of the rinex file:

```bash
teqc +qc rinex_file_name
```
  
Check satellite status for each site - if a site is only seeing 4 satellites for a period, or there are significant tracking problems, consider windowing out the poor data.  

To do the windowing:

```bash
teqc -st hhmm00 +dm mm input_rinex_file > output_rinex_file
```
  
N.b. -st by default assumes everything is in seconds, hence you have to specify the number of seconds to force it to understand hours and minutes. So e.g. to extract Lev6 temporary position from 2012 autumn file, begining at 12pm and continuing for 28 minutes:

```bash
teqc -st 120000 +dm 28 levr_2430_ol_12o > lev6_<doy>0_ol.12o
```

(Use the same filename format as proper rinex files so that track knows what to look for.)
  
Get the APR coordinates to give to track by uploading the rinex file to:
http://www.geod.nrcan.gc.ca/online_data_e.php
(The other PPP service linked to above doesn't seem to work for these small rinex files. Also, the estimate in the rinex header is also generally incorrect after windowing to each site.)

N.b. PPP services don't necessarily appear to give locations accurate enough by themselves - you do have to process the measurements kinematically through track/process_dgps.
 
Now run `process_dgps.py` for each rover site you wish to process.

Copy the GEOD results files into your main GPS processing directory. You'll then be able to concatenate the output GEOD file to the main rover dataset when you run concatenate_geod. 


## Concatenate the track files

The GEOD track output files need to be combined together into one big file, removing the overlapping hours.


## Correcting pole leans

This can be attempted by fitting functions to remove the lean and leave the residual.
It's best to write your own script to do this on a case by case basis.
Load in the GEOD file, do alterations on the NEU data in it, and re-save the GEOD file.
 
  
## Converting to displacements and velocities

### Site origin

If older data for this site has already been post-processed then a file, `origin_<site>.csv` will have been generated. Place this file in the working directory.

If this is the first occasion of processing for this site,  run 

```bash
calculate_local_origin.py <site> <Parquet file>
```

### Displacements and velocities of batch

Use `gnss_disp_vel.py`.

If older data for this site have already been post-processed then a file `rotation_<site>.dat` will exist, defining the coefficients to rotate the coordinates into along/across-track displacement. Place this file in the working directory.

Some hints on a workable processing strategy:

* Use the script "iteratively" to identify periods which should be excluded.
* Add exclusion periods to a file named `exclusions_<site>.csv`, located in your working directory (or specify elsewhere with the `-optpath` option of `gnss_disp_vel.py`).
* Re-run the script.
* If corrections due to pole re-drilling are needed then this functionality will first need to be implemented, as it was not (yet) needed for the high-elevation ice velocities campaign!


### Joining batches together

This could be considered an analysis task. TO-DO: create script to estimate seasonal and annual velocities.
	

## Other information

### Trimble files

Net RS files are in T00 format. You'll need to runpkr00 utility to convert them to dat files first. gps.RinexConvert.window_overlap can then process the dat file, e.g.:

```python
import gps
rx = gps.RinexConvert()
rx.window_overlap("file.dat","22:00:00",28)
```
	

### Problems with noisy pseudo-range data (lev7)

Email exchange between AJT and MAK, December 2013-January 2014.
"The other thing that can cause this sort of numerical instability is that the site motion is too tightly constrained. It turns out that is what it looks like was resulting in the NaNs. After fiddling for a couple of days it seems likely there are some other issues as well, and I think the receiver is suspect. I've copied below a .cmd file that seemed to produce sensible results. I think the main problem here is that the pseudorange data are very noisy, and as a result too many cycle slips are flagged and incorrect ambiguities are being fixed. To overcome this I changed the noise for the pseudoranges to 10m and modified some thresholds for detecting slips and fixing them to integers. 

Note: it may well be this works fine for this day but not so well for any other. Please only use it for lev7 during this season. I'd recommend this receiver is serviced before further use. 

A couple of other notes: 

I don't think you're including the details of the different antennas (ANTMOD_FILE, ANTE_OFF). I've used here what is in the rinex headers but I guess these may vary from what actually was installed. This reduces some high frequency noise. Using a DCB_FILE is useful for ambiguity fixing. It's in the gamit tables directory. "

See also the lev7-specific .cmd file in gps_config.



