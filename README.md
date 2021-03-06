# HSC LSS analyses

We have processed the HSC data for clustering analyses following a number of steps:
1. Download the relevant data from the DR1 database. This was done using the scripts in the directory sql_utils. The script submit_job.py starts all the necessary jobs in your HSC space.
   All the different fields are then downloaded using the script dwl.py with data from urls.txt.
   The full dataset is currently stored (and available) at `/global/cscratch1/sd/damonge/HSC/HSC_*.fits`. This includes both the forced photometry catalog and the metadata.
   
   We have also downloaded the COSMOS 30-band photometry data (Laigle et al. 2016). These data can be downloaded with `COSMOS30band/get_COSMOS_photoz.py`, and is currently stored and available at `/global/cscratch1/sd/damonge/HSC/COSMOS2015_Laigle+_v1.1.fits\`.
2. Reduce the metadata. This implies:
   - Removing all the unnecessary clutter from the raw files
   - Filter out all unnecessary columns (see line 50 of `process_metadata.py` for the columns we actually keep).
   - Writing the reduced tables into new files.

   Additionally, we have downloaded the photo-z pdfs, which are not directly in the HSC database. The data was downloaded with photoz_binning/get_pdfs.sh.

   All this is done with the script `process_metadata.py`. Run `python process_metadata.py -h` to see all available command-line options. The reduced files are stored (and available) at `/global/cscratch1/sd/damonge/HSC/HSC_processed/HSC_*_frames_proc.fits`.
3. Reduce the catalog data. This implies:
   - Removing all the unnecessary clutter from the raw files
   - Applying sanity cuts (duplicates, etc. see description in 1705.06745).
   - Constructing maps of all relevant catalog-based systematics (X-sigma depth, extinction, stars, B.O. mask).
   - Applying a magnitude cut and the star-galaxy separator.
   - Writing the reduced catalog into new files.
   
   All this is done with the script `process.py`. Run `python process.py -h` to see all available command-line options.
   The reduced files are stored (and available) at `/global/cscratch1/sd/damonge/HSC/HSC_processed/` in per-field sub-directories.
   We match the photo-z pdf data to the clean catalogs to produce reduced pdf files. These are currently stored at in the same directories described above (with hopefully self-explanatory file names). The pdf data is stored as a FITS file containig three data tables. The first table contains the object IDs in one column and the pdf array for each object in the second column. The second table contains a single array with the redshift binning of the pdfs (the same for all pdfs).
4. Use the metadata to generate maps of the per-frame systematics (i.e. observing conditions) in each field. This implies:
   - Selecting the frames that fall within the field region.
   - Computing the map pixels each frame intersects and their corresponding areas (this is the most time-consuming part).
   - For each map pixel, build a histogram of the values of each relevant systematic in each exposure touching that pixel.
   - Compress those histograms into maps of summary statistics (currently only computing the mean of each quantity).
   - Maps are generated per-band.
   - The currently mapped quantities are given in line 13 of `map_obscond.py`.
   
   All this is done with the script `map_obscond.py`. Run `python map_obscond.py -h` to see all available command-line options. The maps are stored (and available) at `/global/cscratch1/sd/damonge/HSC/HSC_processed/` in the sub-directories created above for each field.
5. Create galaxy count maps in a number of redshift bins for each field. This implies:
   - Reading in the processed catalogs.
   - Binning them in terms of a given photo-z marker (e.g. ML, mean etc.) for a particular photo-z code.
   - Generating a map of the number of objects in a given bin found per pixel.
   - Generating an estimate of the redshift distribution for objects in a given bin. We currently do this as a histogram of the MC redshift value stored for each object.
   - Save all maps and redshift distributions to file. These are currently collected into a single FITS file that alternates image HDUs (containing the maps) and table HDUs (containing the binned N(z)).
   
   All this is done with the script `cat_sampler.py`. Run `python cat_sampler.py -h` to see all possible command-line options. 
6. Generate diagonstic plots and data for the impact of different systematics. For each field we basically estimate the average galaxy density in bins of the local value of a given systematic. All this is done with the script `check_sys.py`. Run `python check_sys.py -h` to see all possible command-line options. 

   The diagnostic plots and data can be found in e.g. `/global/cscratch1/sd/damonge/HSC/HSC_processed/WIDE_GAMA09H/WIDE_GAMA09H_eab_best_pzb4bins_systematics/` (for the GAMA09H field).
7. Compute tomographic cross-power spectrum and covariance matrix. This implies, for each field:
   - Reading in the maps generated in the previous stage.
   - Reading in masks and systematics maps.
   - Computing all possible power spectra, including systematics deprojection, using NaMaster
   - Estimating the covariance matrix. Currently only theoretical Gaussian estimate is implemented.
   - Saving all power spectra into a SACC file
   
   All this is done with the script `power_specter.py`. Run `python power_specter.py -h` to see all possible command-line options.
8. It's worth noting that we currently use WCS to create flat-sky maps of different quantities (depth, mask, dust etc) when processing each field. The maps are generated using gnomonic projection, with the median coordinates of all sources in each field as the tangent point.
9. The bash script `run_process_all.sh` runs 2, 3, 4 and 5 above for all the WIDE fields.
10. Once all fields have been processed, we compute maps of the galaxy distribution and their corresponding power spectra for each field. The script `study_power_spectra.py` does this for all the WIDE fields. The power spectra are contaminant-deprojected for all contaminants studied in the previous step. We expect to extend this study to a larger list of systematics.
11. Our studies currently use a magnitude limit i<24.5. This is based on a study of the 10-sigma depth maps on all the different fields, and corresponds to a conservative estimate of the magnitude limit of the sample. Note that the quality of the photo-zs degrades significantly for fainter sources (according to the HSC papers).

The scripts described above make use of some dependencies and python modules written explicitly for this work. The most relevant ones are:
- `check_sys.py` and `twoPtCorr.py` are currently orphan files used in the development of the real-space pipeline.
- `flatmaps.py`: contains routines to construct and manipulate flat-sky maps (mimicking as much as possible the functionality implemented in HEALPix for curved skies).
- `createMaps.py`: contains routines to generate maps based on information defined on a discrete set of points.
- `estDepth.py`: contains routines to create depth maps using 3 different methods. All routines are wrapped into a single one called `get_depth`.
- `flatMask.py`: describes the method used to generate the bright-object mask from the catalog data.
- `obscond.py`: defines an "ObservingCondition" class to handle the generation of observing condition maps from the metadata. Used by `map_obscond.py`.
- `NaMaster` (https://github.com/LSSTDESC/NaMaster): a python module to compute arbitrary-spin power spectra of masked fields both in flat and curved skies. The current state of the pipeline requires the use of the `validation` branch instead of `master`.
- `SACC` (https://github.com/LSSTDESC/sacc): a file format to store generic two-point functions.

The repo currently also hosts a number of ipython notebooks that were used to carry out the first analyses on the data. These also illustrate the use that has been made of the database information.
